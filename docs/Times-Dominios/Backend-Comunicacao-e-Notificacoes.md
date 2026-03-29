# 🔧 Backend — Comunicação & Notificações

## Microsserviços
| Microsserviço | Responsabilidade |
| --- | --- |
| `messaging-service` | Conversões, URA, mensagens, WebSocket server, criptografia AES-256-GCM por conversa |
| `file-service` | Upload de arquivos do chat via S3, scan antivírus |
| `notification-service` | Pipeline Kafka-driven, preferências, janela de silêncio, retry com backoff |
| `push-service` | Abstracão FCM/OneSignal, gestão de tokens, fallback por canal |
| `reminder-jobs` | Cron: lembretes de consulta 24h e 1h antes |
