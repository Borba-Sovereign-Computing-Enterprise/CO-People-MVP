# Especificação Técnica — Painel Administrativo

> **Audiência:** Engenheiros
> 

> **Objetivo:** Schemas de entidades admin, RBAC data-driven, endpoints, optimistic locking e Kafka events do painel.
> 

---

## 1. Entidades e Atributos

### 1.1 `admin_users`

| Atributo | Tipo | Constraints | Descrição |
| --- | --- | --- | --- |
| `id` | UUID | PK, FK [users.id](http://users.id) |  |
| `role` | ENUM | NOT NULL | `super_admin` \ |
| `created_by` | UUID | FK admin_[users.id](http://users.id), NOT NULL | Quem criou este admin |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT true |  |
| `last_login_at` | TIMESTAMP | NULLABLE |  |
| `requires_2fa` | BOOLEAN | NOT NULL, DEFAULT true |  |
| `allowed_ip_ranges` | VARCHAR[] | NULLABLE | IPs permitidos (null = todos) |

### 1.2 `professional_review_queue`

| Atributo | Tipo | Constraints | Descrição |
| --- | --- | --- | --- |
| `id` | UUID | PK |  |
| `professional_id` | UUID | FK [professionals.id](http://professionals.id), NOT NULL |  |
| `status` | ENUM | NOT NULL, DEFAULT `PENDING` | `PENDING` \ |
| `attempt_number` | SMALLINT | NOT NULL, DEFAULT 1 | Máx. 3 tentativas |
| `reviewed_by` | UUID | NULLABLE, FK admin_[users.id](http://users.id) |  |
| `reviewed_at` | TIMESTAMP | NULLABLE |  |
| `rejection_code` | VARCHAR(10) | NULLABLE | REJ-01..REJ-06 |
| `rejection_notes` | TEXT | NULLABLE |  |
| `locked_by` | UUID | NULLABLE | Optimistic lock (quem está revisando) |
| `locked_at` | TIMESTAMP | NULLABLE |  |
| `submitted_at` | TIMESTAMP | NOT NULL |  |
| `alert_sent_at` | TIMESTAMP | NULLABLE | Alerta 72h enviado |

### 1.3 `moderation_queue` — Avaliações denunciadas

| Atributo | Tipo | Constraints | Descrição |
| --- | --- | --- | --- |
| `id` | UUID | PK |  |
| `review_id` | UUID | FK [reviews.id](http://reviews.id), UNIQUE, NOT NULL |  |
| `reported_by` | UUID | FK [users.id](http://users.id), NOT NULL |  |
| `report_reason` | TEXT | NOT NULL |  |
| `status` | ENUM | NOT NULL, DEFAULT `PENDING` | `PENDING` \ |
| `reviewed_by` | UUID | NULLABLE, FK admin_[users.id](http://users.id) |  |
| `reviewed_at` | TIMESTAMP | NULLABLE |  |
| `decision_notes` | TEXT | NULLABLE |  |
| `locked_by` | UUID | NULLABLE | Optimistic lock |
| `locked_at` | TIMESTAMP | NULLABLE |  |
| `created_at` | TIMESTAMP | NOT NULL |  |

### 1.4 `platform_configs` — Configurações financeiras e de catálogo

| Atributo | Tipo | Descrição |  |
| --- | --- | --- | --- |
| `key` | VARCHAR(100) | PK. Ex: `platform_fee_pct`, `transfer_delay_days` |  |
| `value` | JSONB | Valor atual |  |
| `previous_value` | JSONB | Valor anterior (histórico imutável em audit_log) |  |
| `updated_by` | UUID | FK admin_[users.id](http://users.id) |  |
| `updated_at` | TIMESTAMP | NOT NULL |  |
| `effective_from` | TIMESTAMP | NOT NULL | Quando entra em vigor |
| `notes` | TEXT | NULLABLE | Justificativa da alteração |

---

## 2. Optimistic Locking — Dupla Moderação Simultânea

```clojure
;; Evita dois admins moderando o mesmo item ao mesmo tempo

(defn lock-review-item! [db queue-table item-id admin-id]
  (let [now (Instant/now)
        result (db/execute! db
                 {:update queue-table
                  :set    {:locked_by admin-id
                           :locked_at now}
                  :where  [:and
                           [:= :id item-id]
                           ;; Só bloqueia se:
                           ;; 1. Não está bloqueado OU
                           ;; 2. Lock expirou há mais de 10min (admin saiu sem finalizar)
                           [:or
                            [:is :locked_by nil]
                            [:< :locked_at (.minus now 10 ChronoUnit/MINUTES)]]]})]
    (when (zero? (:next.jdbc/update-count result))
      (throw (ex-info "Item already locked by another admin"
                     {:type :conflict :item-id item-id})))))

(defn unlock-review-item! [db queue-table item-id admin-id]
  (db/execute! db
    {:update queue-table
     :set    {:locked_by nil :locked_at nil}
     :where  [:and [:= :id item-id] [:= :locked_by admin-id]]}))
```

---

## 3. RBAC Data-Driven via EDN

```clojure
;; As permissões do painel são verificadas da mesma forma que o app
;; mas com escopo :admin/*

;; config/rbac.edn (extrato do painel)
{:admin-roles
  {:operations
    {:can [:professional/approve :professional/reject
           :user/suspend :user/view
           :review/moderate :specialty/manage]}

   :financial
    {:can [:payment/configure :report/financial
           :transfer/view :dispute/manage]}

   :support
    {:can [:user/view :appointment/view :payment/view
           :chat/view-reported]}

   :data-analyst
    {:can [:report/read-anonymized]}  ;; sem PII

   :super-admin
    {:can :all
     :requires {:2fa true :session-timeout-minutes 30}}}}
```

---

## 4. Endpoints REST — Painel Admin

### Fila de Aprovação

```
GET  /api/v1/admin/professionals/queue
     ?status=PENDING&page=1
     Response: { items[], pagination, overdue_count }

POST /api/v1/admin/professionals/:id/lock
     Response: 200 | 409 (já bloqueado)

PATCH /api/v1/admin/professionals/:id/approve
     Response: 200 { professional_id, status: "APPROVED" }

PATCH /api/v1/admin/professionals/:id/reject
     Body: { rejection_code, rejection_notes? }
     Response: 200 { professional_id, status: "REJECTED" }

PATCH /api/v1/admin/professionals/:id/suspend
     Body: { reason, until? }
     Response: 200
```

### Gerenciamento de Usuários

```
GET  /api/v1/admin/users?type=patient|professional&status=ACTIVE
GET  /api/v1/admin/users/:id

PATCH /api/v1/admin/users/:id/suspend
      Body: { reason, until? }

DELETE /api/v1/admin/users/:id
       Body: { reason }           // Soft delete + anonimização
       Header: X-Approver-Token   // Token do segundo aprovador

POST /api/v1/admin/users/:id/reset-password
     // Envia link para o email do usuário
```

### Moderação de Avaliações

```
GET  /api/v1/admin/moderation/reviews
     ?status=PENDING

POST /api/v1/admin/moderation/reviews/:id/lock

PATCH /api/v1/admin/moderation/reviews/:id/keep
      Body: { notes? }

PATCH /api/v1/admin/moderation/reviews/:id/remove
      Body: { notes }
```

### Configurações

```
GET  /api/v1/admin/configs
PUT  /api/v1/admin/configs/:key
     Body: { value, effective_from, notes }
     Response: 200 | 403 (permissão insuficiente)

GET  /api/v1/admin/configs/:key/history
     Response: { changes[] }  // via audit_log
```

### Relatórios

```
GET /api/v1/admin/reports/revenue?from=2025-01-01&to=2025-01-31
GET /api/v1/admin/reports/appointments?specialty_id=&region=
GET /api/v1/admin/reports/churn?period=monthly

POST /api/v1/admin/reports/:type/export
     Body: { format: "csv" | "pdf", filters }
     Response: 202 { job_id }  // assíncrono

GET /api/v1/admin/reports/export/:job_id
    Response: 200 { download_url } | 202 { status: "processing" }
```

---

## 5. Kafka Events do Painel

| Evento | Tópico | Consumidores |
| --- | --- | --- |
| `ProfessionalApproved` | `professional.approved` | Search (indexa), Notification |
| `ProfessionalRejected` | `professional.rejected` | Notification |
| `ProfessionalSuspended` | `professional.suspended` | Search (remove), Notification |
| `UserSuspended` | `user.suspended` | Notification, Auth (revoga tokens) |
| `PlatformConfigChanged` | `admin.config_changed` | Audit, Payment (recalcula fees) |
| `ReviewRemoved` | `review.removed` | Profile (recalcula média) |

---

## 6. Job de Alerta de Fila

```clojure
;; Alerta quando cadastros estão pendentes > 72h sem revisão
(defn alert-overdue-queue! [db kafka]
  (let [threshold (.minus (Instant/now) 72 ChronoUnit/HOURS)
        overdue   (db/query db
                    {:select [[:count :id] :total]
                     :from   [:professional_review_queue]
                     :where  [:and
                              [:= :status "PENDING"]
                              [:< :submitted_at threshold]
                              [:is :alert_sent_at nil]]})]
    (when (pos? (:total overdue))
      (kafka/produce! kafka "admin.alert"
        {:type    :queue_overdue
         :count   (:total overdue)
         :details "Cadastros pendentes de aprovação > 72h"}))))
```

---

## 7. Observabilidade

- Métrica: `admin.approval_queue.size` (ativo)
- Métrica: `admin.approval.avg_review_time_hours`
- Métrica: `admin.moderation_queue.size`
- Alerta: `approval_queue.overdue_count > 0` por > 1h
- Alerta: alteração de `platform_fee_pct` — notifica time de operações imediatamente
- Alerta: falha no lock de moderação (conflito de concorrência)
- Dashboard: métricas de NPS, receita e churn (acesso restrito por perfil)
