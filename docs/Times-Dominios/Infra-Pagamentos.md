# ☁️ Infra — Pagamentos

## Banco de Dados

| Recurso | Tabelas |
| --- | --- |
| **PostgreSQL** | `payments`, `transfers`, `payment_events` (Outbox), `webhook_logs` |

## Integração Externa

| Serviço | Config |
| --- | --- |
| Gateway (Stripe/[Pagar.me](http://Pagar.me)) | Chaves via AWS Secrets Manager |
| Webhook endpoint | `POST /webhooks/payment-gateway` — HMAC validado |

## Kafka

| Papel | Tópicos |
| --- | --- |
| Producer (via Outbox) | `payment.processed`, `payment.failed`, `payment.refunded`, `payment.transfer_completed` |
| Consumer | `appointment.completed` (agenda repasse) |

## Observabilidade

| Métrica | Alerta |
| --- | --- |
| `payment.success_rate` | < 95% por 5min — CRITICAL |
| `outbox.unpublished_count` | > 50 por 10min |
| `transfer.failed_count` | Qualquer ocorrência > 0 — IMMEDIATE |
