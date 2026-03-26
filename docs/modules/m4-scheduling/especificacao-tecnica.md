# Especificação Técnica — Agendamento

> **Audiência:** Engenheiros
> 

> **Objetivo:** Schema de entidades, state machine completa, endpoints, conflict detection, Kafka events e regras de concorrência.
> 

---

## 1. Entidades e Atributos

### 1.1 `availability_slots` — Horários configurados pelo profissional

| Atributo | Tipo | Constraints | Descrição |
| --- | --- | --- | --- |
| `id` | UUID | PK |  |
| `professional_id` | UUID | FK, NOT NULL |  |
| `day_of_week` | SMALLINT | NOT NULL, 0-6 | 0=Dom ... 6=Sáb |
| `start_time` | TIME | NOT NULL | Horário de início |
| `end_time` | TIME | NOT NULL | Horário de fim |
| `slot_duration_minutes` | SMALLINT | NOT NULL, DEFAULT 60 | Duração de cada slot |
| `modality` | ENUM | NOT NULL | `presential` \ |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT true |  |

### 1.2 `blocked_periods` — Bloqueios de agenda

| Atributo | Tipo | Constraints | Descrição |
| --- | --- | --- | --- |
| `id` | UUID | PK |  |
| `professional_id` | UUID | FK, NOT NULL |  |
| `starts_at` | TIMESTAMP | NOT NULL |  |
| `ends_at` | TIMESTAMP | NOT NULL |  |
| `reason` | VARCHAR(255) | NULLABLE | Férias, cirurgia, etc. |

### 1.3 `appointments` — Entidade principal

| Atributo | Tipo | Constraints | Descrição |
| --- | --- | --- | --- |
| `id` | UUID | PK |  |
| `patient_id` | UUID | FK [users.id](http://users.id), NOT NULL |  |
| `professional_id` | UUID | FK [professionals.id](http://professionals.id), NOT NULL |  |
| `specialty_id` | UUID | FK [specialties.id](http://specialties.id), NOT NULL |  |
| `modality` | ENUM | NOT NULL | `presential` \ |
| `status` | ENUM | NOT NULL, DEFAULT `PENDING` | Ver state machine |
| `scheduled_at` | TIMESTAMP | NOT NULL | Data/hora do início |
| `ends_at` | TIMESTAMP | NOT NULL | Data/hora do fim |
| `address_snapshot` | JSONB | NULLABLE | Snapshot do endereço no momento do agendamento |
| `price_snapshot` | DECIMAL(10,2) | NOT NULL | Preço no momento do agendamento |
| `min_cancel_hours` | SMALLINT | NOT NULL | Regra do profissional capturada no momento |
| `cancel_fee_pct` | DECIMAL(5,2) | NOT NULL, DEFAULT 0 | % de taxa em cancelamento tardio |
| `patient_confirmed` | BOOLEAN | NOT NULL, DEFAULT false | Confirmação bilateral R08 |
| `professional_confirmed` | BOOLEAN | NOT NULL, DEFAULT false |  |
| `confirmed_at` | TIMESTAMP | NULLABLE | Quando ambos confirmaram |
| `cancelled_by` | ENUM | NULLABLE | `patient` \ |
| `cancellation_reason` | TEXT | NULLABLE |  |
| `cancelled_at` | TIMESTAMP | NULLABLE |  |
| `payment_id` | UUID | NULLABLE, FK [payments.id](http://payments.id) | Vinculação com pagamento |
| `created_at` | TIMESTAMP | NOT NULL |  |
| `updated_at` | TIMESTAMP | NOT NULL |  |
| `deleted_at` | TIMESTAMP | NULLABLE | Soft delete |

**Índice crítico (evita double booking):**

```sql
CREATE UNIQUE INDEX idx_no_double_booking
ON appointments (professional_id, scheduled_at)
WHERE status NOT IN ('CANCELLED', 'CANCELLED_LATE',
                     'CANCELLED_BY_PROFESSIONAL', 'REJECTED');

```
### 1.4 appointment_history — Log de estados

| Atributo | Tipo | Descrição |
| --- | --- | --- |
| ``id`` | UUID | PK |
| ``appointment_id`` | UUID | FK, NOT NULL |
| ``actor_type`` | ENUM | ``patient`` \\ |
| ``actor_id`` | UUID | ID de quem agiu |
| ``from_status`` | VARCHAR(50) | Status anterior |
| ``to_status`` | VARCHAR(50) | Novo status |
| ``reason`` | TEXT | Motivo informado |
| ``financial_impact`` | JSONB | ``{refund_amount, ``fee_charged, ``channel}`` |
| ``created_at`` | TIMESTAMP | NOT NULL |


### 1.5 professional_settings — Configurações de agenda

| Atributo | Tipo | DEFAULT | Descrição |
| --- | --- | --- | --- |
| ``professional_id`` | UUID | PK, FK |  |
| ``min_advance_hours`` | SMALLINT | 2 | Antecêdencia mínima para agendar |
| ``min_cancel_hours`` | SMALLINT | 24 | Prazo de cancelamento sem multa |
| ``cancel_fee_pct`` | DECIMAL(5,2) | 0 | % cobrado em cancelamento tardio |
| ``max_appointments_per_day`` | SMALLINT | NULL | NULL = sem limite |
| ``booking_window_days`` | SMALLINT | 90 | Quantos dias à frente pode agendar |

PENDING (pagamento pendente)
    │
    ├── pagamento aprovado ────────────────► CONFIRMED
    │                                        │
    ├── timeout de pagamento (30min) ─► EXPIRED  ├─ paciente cancela (no prazo) ──► CANCELLED
    │                                        │
    └── paciente cancela ──────► CANCELLED  ├─ paciente cancela (fora prazo) ─► CANCELLED_LATE
                                             │
                                             ├─ profissional cancela ───► CANCELLED_BY_PROFESSIONAL
                                             │
                                             └─ ambos confirmam realização ─► COMPLETED
                                                        │ (também possível se
                                             nenhum confirmar após 48h)
                                             └─ sistema ──────────────► NO_SHOW

## 3. Conflict Detection — Double Booking

;; Verifica se o slot está disponível antes de criar appointment
;; Usa SELECT FOR UPDATE para garantir atomicidade

(defn slot-available? [db professional-id scheduled-at duration-minutes]
  (let [ends-at (-> scheduled-at
                    (.plusMinutes duration-minutes))
        conflicts (db/query db
                    {:select [:id]
                     :from   [:appointments]
                     :where  [:and
                              [:= :professional_id professional-id]
                              [:not-in :status ["CANCELLED" "CANCELLED_LATE"
                                               "CANCELLED_BY_PROFESSIONAL"
                                               "REJECTED" "EXPIRED"]]
                              ;; overlap check
                              [:< :scheduled_at ends-at]
                              [:> :ends_at scheduled-at]]
                     :for    :update})]
    (empty? conflicts)))

;; Em caso de corrida (dois requests simultâneos):
;; O UNIQUE INDEX garante que apenas um INSERT seja aceito
;; O segundo receberá uma constraint violation — retorna 409 Conflict

## 4. Endpoints REST

### 4.1 Horários disponíveis

GET /api/v1/professionals/:id/availability
Query: ?date=2025-06-15&modality=presential
Response 200: {
  date,
  available_slots: [
    { starts_at, ends_at, modality }
  ]
}

### 4.2 Agendamento

POST /api/v1/appointments
Auth: paciente autenticado
Body: {
  professional_id, specialty_id,
  scheduled_at, modality,
  address_id (para home_visit)
}
Response 201: {
  appointment_id, status: "PENDING",
  payment_session_url  // redireciona para M5
}

GET /api/v1/appointments/:id
Response 200: { appointment details + history }

GET /api/v1/me/appointments?status=CONFIRMED&page=1
Response 200: { appointments[], pagination }

### 4.3 Confirmação bilateral

POST /api/v1/appointments/:id/confirm-completion
Auth: paciente ou profissional autenticado
Response 200: {
  appointment_id,
  patient_confirmed, professional_confirmed,
  status  // COMPLETED se ambos confirmaram
}

### 4.4 Cancelamento

DELETE /api/v1/appointments/:id
Auth: paciente ou profissional
Body: { reason: string }
Response 200: {
  appointment_id,
  new_status,           // CANCELLED | CANCELLED_LATE | CANCELLED_BY_PROFESSIONAL
  refund_amount,
  fee_charged
}

### 4.5 Gestão de agenda (profissional)

GET  /api/v1/me/availability
POST /api/v1/me/availability           // cria slot recorrente
DELETE /api/v1/me/availability/:id

GET  /api/v1/me/blocked-periods
POST /api/v1/me/blocked-periods        // bloqueia período

## 5. Kafka Events


| Evento | Tópico | Payload resumido | Consumidores |
| --- | --- | --- | --- |
| `AppointmentCreated` | `appointment.created` | appointment_id, professional_id, scheduled_at | Notification |
| `AppointmentConfirmed` | `appointment.confirmed` | appointment_id, payment_id | Notification |
| `AppointmentCompleted` | `appointment.completed` | appointment_id, professional_id, patient_id | Payment (repasse), Review (libera), Notification |
| `AppointmentCancelled` | `appointment.cancelled` | appointment_id, cancelled_by, refund_amount, fee | Payment (reembolso), Notification |
| `AppointmentNoShow` | `appointment.no_show` | appointment_id | Admin, Notification |

---

## 6. Job de No-Show

```clojure
;; Roda a cada hora
;; Marca como NO_SHOW consultas que passaram 48h sem confirmação

(defn mark-no-shows! [db]
  (let [threshold (-> (Instant/now) (.minus 48 ChronoUnit/HOURS))]
    (db/execute! db
      {:update :appointments
       :set    {:status     "NO_SHOW"
                :updated_at (Instant/now)}
       :where  [:and
                [:= :status "CONFIRMED"]
                [:< :ends_at threshold]
                [:or
                 [:= :patient_confirmed false]
                 [:= :professional_confirmed false]]]})))
```

## 7. Observabilidade
Métrica: appointment.created_per_hour — volume de agendamentos

Métrica: appointment.cancellation_rate por tipo (patient/professional/late)

Métrica: appointment.no_show_rate

Alerta: cancellation_rate > 20% em sliding window de 1h — possível bug de UX

Alerta: double_booking_conflict_rate > 0 — indica falha no lock de concorrência

Alerta: job de no-show não rodou em > 2h
   
