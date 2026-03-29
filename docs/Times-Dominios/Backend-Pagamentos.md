# 🔧 Backend — Pagamentos

## Microsserviços
| Microsserviço | Responsabilidade |
| --- | --- |
| `payment-service` | Checkout, state machine de payment, idempotência via `idempotency_key` |
| `transfer-service` | Repasse D+2, agendamento, retry, suspensão em disputa |
| `webhook-service` | Recepção HMAC-validada de webhooks do gateway, idempotência via `webhook_logs` |
| `outbox-publisher` | Publica eventos do Outbox table no Kafka — garantia at-least-once |
