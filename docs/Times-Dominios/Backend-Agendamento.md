# Backend — Agendamento

## Microsserviços

| Microsserviço | Responsabilidade |
| --- | --- |
| `scheduling-service` | CRUD de agendamentos, state machine (8 estados), confirmação bilateral, cancelamento |
| `availability-service` | Configuração de slots recorrentes, bloqueios de agenda, conflict detection via UNIQUE INDEX + SELECT FOR UPDATE |
| `appointment-jobs` | Cron: no-show (48h), lembretes (24h/1h), alerta de fila |
