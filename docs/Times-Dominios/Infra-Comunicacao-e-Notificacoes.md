# ☁️ Infra — Comunicação & Notificações

## Banco de Dados

| Recurso | Tabelas |
| --- | --- |
| **PostgreSQL** | `conversations`, `messages`, `message_audit_log`, `websocket_sessions`, `notifications`, `notification_preferences`, `push_tokens`, `scheduled_notifications` |
| **Redis** | Sessões WebSocket ativas, presense de conexões |

## Object Storage

| Bucket | Conteúdo |
| --- | --- |
| `plataforma-saude-chat-files` | Arquivos enviados no chat (quarentena + scan) |

## Integrações Externas

| Serviço | Uso |
| --- | --- |
| FCM / OneSignal | Push notifications |
| SendGrid / AWS SES | E-mail transacional |
| Twilio / Zenvia | SMS e OTP |

## Observabilidade

| Métrica | Alerta |
| --- | --- |
| `chat.websocket.error_rate` | > 5% por 5min |
| `notification.email.failed_rate` | > 5% |
| `notification.sms.failed_rate` | > 10% (afeta 2FA) |
| `chat.unresponded_48h_count` | Aumento > 20% semana a semana |
