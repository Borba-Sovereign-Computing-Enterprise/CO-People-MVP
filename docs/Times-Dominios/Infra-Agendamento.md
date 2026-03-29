# Infra — Agendamento

## Banco de Dados

| Recurso | Tabelas |
| --- | --- |
| **PostgreSQL** | `appointments`, `availability_slots`, `blocked_periods`, `appointment_history`, `professional_settings` |
| **Redis** | Mutex para lock de conflito em alta concorrência |

## Kafka

| Papel | Tópicos |
| --- | --- |
| Producer | `appointment.created`, `appointment.confirmed`, `appointment.completed`, `appointment.cancelled`, `appointment.no_show` |
| Consumer | `payment.processed` (confirma agendamento) |

## Observabilidade
| Métrica | Alerta |
| --- | --- |
| `appointment.cancellation_rate` | > 20% em sliding window 1h |
| `appointment.double_booking_conflict` | Qualquer ocorrência > 0 |
| `appointment.no_show_rate` | > 15% |
| Job `appointment-jobs` | Não rodou em > 2h |
