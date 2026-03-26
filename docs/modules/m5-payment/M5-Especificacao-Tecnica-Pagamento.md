# ⚙️ Especificação Técnica — Pagamento

> **Audiência:** Engenheiros

> **Objetivo:** Schemas, endpoints, Outbox Pattern, split payment, webhook handling idempotente e Kafka events de pagamento.

---

## 1. Entidades e Atributos

### 1.1 `payments` — Transação principal

| Atributo | Tipo | Constraints | Descrição |
|---------|------|------------|----------|
| `id` | UUID | PK | |
| `appointment_id` | UUID | FK, UNIQUE, NOT NULL | 1 pagamento por consulta |
| `patient_id` | UUID | FK users.id, NOT NULL | |
| `professional_id` | UUID | FK professionals.id, NOT NULL | |
| `amount` | DECIMAL(10,2) | NOT NULL | Valor total cobrado do paciente |
| `platform_fee_pct` | DECIMAL(5,2) | NOT NULL | % da plataforma (snapshot) |
| `platform_fee_amount` | DECIMAL(10,2) | NOT NULL | Valor absoluto da taxa |
| `professional_amount` | DECIMAL(10,2) | NOT NULL | Valor a repassar |
| `currency` | CHAR(3) | NOT NULL, DEFAULT 'BRL' | |
| `method` | ENUM | NOT NULL | `pix` \| `credit_card` \| `debit_card` \| `boleto` |
| `status` | ENUM | NOT NULL, DEFAULT `PENDING` | Ver state machine |
| `gateway_provider` | VARCHAR(50) | NOT NULL | `stripe` \| `pagarme` |
| `gateway_transaction_id` | VARCHAR(255) | UNIQUE, NULLABLE | ID externo do gateway |
| `gateway_payment_intent_id` | VARCHAR(255) | NULLABLE | Para Stripe |
| `idempotency_key` | VARCHAR(255) | UNIQUE, NOT NULL | `payment:{appointment_id}` |
| `paid_at` | TIMESTAMP | NULLABLE | |
| `refunded_at` | TIMESTAMP | NULLABLE | |
| `refund_amount` | DECIMAL(10,2) | NULLABLE | |
| `refund_reason` | TEXT | NULLABLE | |
| `created_at` | TIMESTAMP | NOT NULL | |
| `updated_at` | TIMESTAMP | NOT NULL | |

### 1.2 `transfers` — Repasse ao profissional

| Atributo | Tipo | Constraints | Descrição |
|---------|------|------------|----------|
| `id` | UUID | PK | |
| `payment_id` | UUID | FK, NOT NULL | |
| `professional_id` | UUID | FK, NOT NULL | |
| `amount` | DECIMAL(10,2) | NOT NULL | Valor do repasse |
| `status` | ENUM | NOT NULL, DEFAULT `PENDING` | `PENDING` \| `COMPLETED` \| `FAILED` \| `SUSPENDED` |
| `scheduled_for` | TIMESTAMP | NOT NULL | Data prevista (D+2) |
| `gateway_transfer_id` | VARCHAR(255) | NULLABLE | |
| `completed_at` | TIMESTAMP | NULLABLE | |
| `failure_reason` | TEXT | NULLABLE | |
| `created_at` | TIMESTAMP | NOT NULL | |

### 1.3 `payment_events` — Outbox Table

```sql
-- Outbox Pattern: eventos persistidos na mesma transação do pagamento
-- Consumer Kafka lê e publica no broker
CREATE TABLE payment_events (
  id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  payment_id      UUID        NOT NULL REFERENCES payments(id),
  event_type      VARCHAR(100) NOT NULL,   -- 'PaymentProcessed', 'PaymentRefunded', etc.
  payload         JSONB       NOT NULL,
  published_at    TIMESTAMP   NULL,        -- NULL = ainda não publicado
  created_at      TIMESTAMP   NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payment_events_unpublished
ON payment_events (created_at)
WHERE published_at IS NULL;
```

### 1.4 `webhook_logs` — Idempotência de webhooks

| Atributo | Tipo | Descrição |
|---------|------|----------|
| `id` | UUID | PK |
| `provider` | VARCHAR(50) | Gateway que enviou |
| `event_id` | VARCHAR(255) | ID do evento no gateway (UNIQUE) |
| `event_type` | VARCHAR(100) | Ex: `payment_intent.succeeded` |
| `payload` | JSONB | Payload raw |
| `processed_at` | TIMESTAMP | NULL se ainda não processado |
| `status` | ENUM | `RECEIVED` \| `PROCESSED` \| `FAILED` |
| `received_at` | TIMESTAMP | NOT NULL |

---

## 2. State Machine — Payments

```
PENDING
    │
    ├── gateway webhook: success ──► PAID ────────────► TRANSFER_SCHEDULED
    │                               │                      (após appointment.completed)
    ├── gateway webhook: failed ──► FAILED    │
    │                                          └── cancel ─► REFUNDING ─► REFUNDED
    └── timeout 30min ────────► EXPIRED

TRANSFER_SCHEDULED
    │
    ├── transfer executado ─► TRANSFER_COMPLETED
    ├── transfer falhou ───► TRANSFER_FAILED (retry 3x)
    └── disputa aberta ───► TRANSFER_SUSPENDED
```

---

## 3. Outbox Pattern — Garantia de Entrega

```clojure
;; Toda operação de pagamento persiste o evento na mesma transação
;; Garante que o evento não se perde mesmo se o Kafka estiver indisponível

(defn process-payment-success! [db payment-id gateway-tx-id]
  (db/with-transaction [tx db]
    ;; 1. Atualiza status do pagamento
    (db/execute! tx
      {:update :payments
       :set    {:status                  "PAID"
                :gateway_transaction_id  gateway-tx-id
                :paid_at                 (Instant/now)}
       :where  [:= :id payment-id]})

    ;; 2. Confirma o agendamento
    (db/execute! tx
      {:update :appointments
       :set    {:status "CONFIRMED" :updated_at (Instant/now)}
       :where  [:= :payment_id payment-id]})

    ;; 3. Persiste evento no Outbox (mesma transação!)
    (db/execute! tx
      {:insert-into :payment_events
       :values [{:payment_id  payment-id
                 :event_type  "PaymentProcessed"
                 :payload     (json/encode {:gateway_tx_id gateway-tx-id})}]})))

;; Outbox consumer (job separado, roda a cada 5s)
(defn publish-pending-events! [db kafka]
  (let [events (db/query db
                  {:select [:*]
                   :from   [:payment_events]
                   :where  [:is :published_at nil]
                   :limit  100
                   :order-by [[:created_at :asc]]})]
    (doseq [event events]
      (kafka/produce! kafka (event-topic event) event)
      (db/execute! db
        {:update :payment_events
         :set    {:published_at (Instant/now)}
         :where  [:= :id (:id event)]}))))
```

---

## 4. Webhook Handler — Idempotência

```clojure
;; POST /webhooks/payment-gateway
;; Verifica assinatura HMAC antes de processar

(defn handle-webhook! [db request]
  (let [{:keys [headers body]}   request
        provider                  (get-provider headers)
        signature                 (get-signature headers)
        event                     (json/decode body)
        event-id                  (:id event)]

    ;; 1. Valida assinatura HMAC do gateway
    (when-not (valid-signature? provider body signature)
      (throw (ex-info "Invalid signature" {:status 401})))

    ;; 2. Idempotência: verifica se já processamos este evento
    (let [existing (db/find-by db :webhook_logs {:event_id event-id})]
      (when existing
        (return {:status 200 :body "Already processed"})))

    ;; 3. Registra o recebimento
    (db/insert! db :webhook_logs
      {:provider     provider
       :event_id     event-id
       :event_type   (:type event)
       :payload      event
       :status       "RECEIVED"})

    ;; 4. Processa de acordo com o tipo
    (case (:type event)
      "payment_intent.succeeded"  (process-payment-success! db event)
      "payment_intent.failed"     (process-payment-failed! db event)
      "charge.refunded"           (process-refund! db event)
      "transfer.paid"             (process-transfer-completed! db event)
      (log/info "Unhandled event type" (:type event)))

    ;; 5. Marca como processado
    (db/update! db :webhook_logs {:event_id event-id}
      {:status "PROCESSED" :processed_at (Instant/now)})

    {:status 200}))
```

---

## 5. Endpoints REST

### 5.1 Checkout

```
POST /api/v1/payments/checkout
Auth: paciente autenticado
Body: { appointment_id, method, card_token? }
Response 201: {
  payment_id,
  method,
  status: "PENDING",
  pix_qr_code?,         // se PIX
  pix_expiry_at?,
  boleto_url?,          // se boleto
  redirect_url?         // se cartão via hosted checkout
}
```

### 5.2 Status e histórico

```
GET /api/v1/payments/:id
Response 200: { payment details }

GET /api/v1/me/payments?page=1&status=PAID
Response 200: { payments[], pagination }

GET /api/v1/me/transfers?page=1      // apenas profissional
Response 200: { transfers[], pagination }
```

### 5.3 Webhook (gateway → plataforma)

```
POST /webhooks/payment-gateway
Header: X-Gateway-Signature: <hmac>
Body: { gateway event payload }
Response 200  // sempre 200 para evitar reentrega desnecessária
```

### 5.4 Admin

```
GET  /api/v1/admin/payments?status=DISPUTED
POST /api/v1/admin/payments/:id/refund
Body: { amount, reason }
POST /api/v1/admin/transfers/:id/suspend
Body: { reason }
POST /api/v1/admin/transfers/:id/release
```

---

## 6. Kafka Events

| Evento | Tópico | Consumidores |
|--------|--------|-------------|
| `PaymentProcessed` | `payment.processed` | Appointment (confirma), Notification, Audit |
| `PaymentFailed` | `payment.failed` | Appointment (exp. slot), Notification |
| `PaymentRefunded` | `payment.refunded` | Appointment, Notification, Audit |
| `TransferCompleted` | `payment.transfer_completed` | Notification (avisa profissional), Audit |
| `TransferFailed` | `payment.transfer_failed` | Admin alert, Notification |

---

## 7. Configurações de Pagamento (EDN)

```clojure
;; config/config.edn
{:payment
  {:provider          #env PAYMENT_PROVIDER   ;; :stripe | :pagarme
   :api-key           #env PAYMENT_API_KEY
   :webhook-secret    #env PAYMENT_WEBHOOK_SECRET
   :platform-fee-pct  12.0                    ;; % padrão (override por admin)
   :transfer-delay-days 2                     ;; D+2
   :payment-timeout-minutes 30               ;; após isso: EXPIRED
   :max-retry-attempts 3
   :supported-methods [:pix :credit_card :debit_card :boleto]}}
```

---

## 8. Observabilidade

- Métrica: `payment.success_rate` por método e gateway
- Métrica: `payment.refund_rate`
- Métrica: `payment.webhook.processing_time`
- Métrica: `payment_events.unpublished_count` (lag do Outbox)
- Alerta: `payment.success_rate < 95%` por 5min — CRITICAL
- Alerta: `payment_events.unpublished_count > 50` por 10min — WARNING (Outbox lag)
- Alerta: `transfer.failed_count > 0` — imediato, profissional não receberá
- Dashboard: volume financeiro diário, taxa de repasse, disputes em aberto

---

## 📂 Navegação

| | |
|--|--|
| ← Hub | [README — M5 Pagamento](./README.md) |
| ← Produto | [Produto & Negócio — Pagamento](./01-M5-Produto-Negocio-Pagamento.md) |
