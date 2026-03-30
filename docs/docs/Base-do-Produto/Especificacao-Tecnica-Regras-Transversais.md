# ⚙️ Especificação Técnica — Regras Transversais
> **Audiência:** Engenheiros, Tech Leads
> 

> **Objetivo:** Definir **como** cada regra transversal é tecnicamente enforced — camada de enforcement, ponto de falha e observabilidade.
> 

---

## 1. Mapa de Enforcement por Regra

| Regra | Camada de Enforcement | Mecanismo |
| --- | --- | --- |
| R01 — Agendamento exige auth | Pedestal Interceptor | `auth-interceptor` bloqueia qualquer rota de agendamento sem JWT válido |
| R02 — Dual role proibido | JWT payload + Interceptor | Campo `type` no token; interceptor de rota valida que `type` bate com contexto |
| R03 — Dados sigilosos exigem sessão | Pedestal Interceptor + DB query scoping | Interceptor valida token; queries de laudo incluem `WHERE patient_id = :user_id OR professional_id = :user_id` |
| R04 — CPF e email únicos | PostgreSQL UNIQUE constraint | `UNIQUE INDEX` em `users.email` e `patients.cpf`, `professionals.cpf` |
| R05 — Profissional sem aprovação invisível | DB query filter | Toda query de busca inclui `WHERE approval_status = 'APPROVED'` |
| R06 — Suspensão imediata | DB status + cache invalidation | Update de status + flush de cache Redis com `KEYS professional:*` |
| R07 — Modalidade obrigatória | DB constraint + query filter | Busca filtra `is_profile_complete = true` |
| R08 — Confirmação bilateral | State machine em `appointments` | Status só vai para `COMPLETED` quando ambos os flags `patient_confirmed` e `professional_confirmed` forem `true` |
| R09 — Avaliação após `COMPLETED` | FK constraint + service guard | `reviews.appointment_id` FK + check no service: `appointment.status == :completed` |
| R10 — 1 avaliação por consulta | PostgreSQL UNIQUE constraint | `UNIQUE (appointment_id)` em `reviews` |
| R11 — Laudo restrito às partes | Row-level scoping | Query sempre inclui cláusula de ownership; presigned URL do S3 é gerada apenas para partes autorizadas |
| R12 — Laudo isolado por consulta | FK + escopo de query | `documents.appointment_id` não pode ser nulo; nunca retornar documentos de outros appointments |
| R13 — Pagamento inside platform | Sem endpoint de pagamento externo | Não existe endpoint público para registrar pagamento manual; toda transação passa pelo gateway |
| R14 — Repasse após `COMPLETED` | State machine + Kafka event | Evento `appointment.completed` dispara job de repasse; sem evento, sem repasse |
| R15 — Reembolso em cancelamento pelo profissional | Service logic | Quando `cancelled_by == :professional`, gateway é chamado para reversal automático |
| R16 — Audit log imutável | Pedestal Interceptor (after) + append-only table | Interceptor de audit grava em `audit_logs` — tabela sem UPDATE/DELETE permitido por role de DB |
| R17 — Arquivamento de inativos | Cron job agendado | Job semanal verifica `last_active_at < NOW() - INTERVAL '12 months'` e marca `status = ARCHIVED` |
| R18 — Soft delete | DB convention | Todas as entidades críticas têm coluna `deleted_at TIMESTAMP NULL`; nenhuma query de leitura omite o filtro `WHERE deleted_at IS NULL` |

---

## 2. Interceptor Stack (Pedestal) — Ordem de Execução

```
Request in
    │
    ▼
[1] rate-limit-interceptor       ← R01 suporte: bloqueia abuso antes de validar token
    │
    ▼
[2] auth-interceptor             ← R01, R02, R03: valida JWT, extrai :user/id e :user/type
    │
    ▼
[3] authorization-interceptor    ← R02: valida que :user/type bate com rota acessada
    │
    ▼
[4] audit-interceptor (enter)    ← R16: registra início da requisição
    │
    ▼
[5] route-handler (business)     ← R08, R09, R13, R14, R15: logic de domínio
    │
    ▼
[6] audit-interceptor (leave)    ← R16: registra resultado e persiste no audit_log
    │
    ▼
Response out
```

---

## 3. Tabela `audit_logs` — Eventos Mandatoriamente Registrados

| Evento (`action`) | Trigger | Dados no `metadata` |
| --- | --- | --- |
| `USER_LOGIN` | Qualquer login bem-sucedido | ip, user_agent, timestamp |
| `USER_LOGIN_FAILED` | Falha de login | ip, tentativa número |
| `APPOINTMENT_CREATED` | Paciente cria agendamento | appointment_id, professional_id, slot |
| `APPOINTMENT_CONFIRMED` | Parte confirma consulta | quem confirmou, timestamp |
| `APPOINTMENT_COMPLETED` | Status vai para COMPLETED | confirmed_by ambos |
| `APPOINTMENT_CANCELLED` | Qualquer cancelamento | cancelled_by, motivo, dentro/fora do prazo |
| `PAYMENT_PROCESSED` | Pagamento aprovado | gateway_tx_id, amount, method |
| `PAYMENT_REFUNDED` | Reembolso emitido | motivo, amount |
| `DOCUMENT_ACCESSED` | Acesso a laudo/documento de saúde | accessor_id, document_id, appointment_id |
| `PROFESSIONAL_APPROVED` | Admin aprova profissional | reviewed_by, professional_id |
| `PROFESSIONAL_SUSPENDED` | Admin suspende profissional | motivo, professional_id |
| `REVIEW_PUBLISHED` | Avaliação publicada | review_id, rating |
| `USER_DATA_EXPORTED` | Solicitação de portabilidade LGPD | requester_id, scope |

---

## 4. Política de Soft Delete

```clojure
;; Toda query de leitura DEVE incluir este filtro
;; Padrão a ser seguido em todos os repositórios

(defn base-query [table]
  {:select [:*]
   :from   [table]
   :where  [:is :deleted_at nil]})

;; Soft delete padronizado
(defn soft-delete! [db table id actor-id]
  (db/execute! db
    {:update table
     :set    {:deleted_at (Instant/now)
              :deleted_by actor-id}
     :where  [:= :id id]}))

;; NUNCA executar DELETE em entidades com deleted_at
;; DB role de aplicação não tem permissão de DELETE nas tabelas críticas
```

---

## 5. State Machine Global de Appointments

```
PENDING
    │
    ├── profissional aceita ──► CONFIRMED
    │                              │
    ├── profissional recusa ──► REJECTED   ├── paciente confirma realizacão
    │                                          │   E profissional confirma ──► COMPLETED
    └── paciente cancela ────► CANCELLED    │
                                               ├── paciente cancela ───► CANCELLED
                                               │
                                               ├── profissional cancela ─► CANCELLED_BY_PROFESSIONAL
                                               │
                                               └── ninguém comparece ──► NO_SHOW

COMPLETED → libera avaliação (R09) + dispara evento repasse (R14)
CANCELLED_BY_PROFESSIONAL → dispara reembolso automático (R15)
```

---

## 6. Kafka Events das Regras Transversais

| Evento | Tópico Kafka | Consumidores |
| --- | --- | --- |
| `AppointmentCompleted` | `appointment.completed` | Payment (repasse), Review (libera avaliação), Notification |
| `AppointmentCancelledByProfessional` | `appointment.cancelled` | Payment (reversal), Notification |
| `ProfessionalApproved` | `professional.approved` | Search (indexa no catálogo), Notification |
| `ProfessionalSuspended` | `professional.suspended` | Search (remove do catálogo), Notification |
| `PaymentProcessed` | `payment.processed` | Audit, Notification |
| `PaymentRefunded` | `payment.refunded` | Audit, Notification |

---

## 7. Observabilidade

- Alerta: qualquer endpoint retornando dado de saúde sem `user_id` no contexto
- Métrica: `audit_log.missing_events` — diff entre eventos esperados e registrados
- Alerta: `appointment.completed` sem subsequente `payment.transfer_initiated` em > 1h
- Dashboard: taxa de soft deletes por entidade (detectar abusos ou bugs de UI)
