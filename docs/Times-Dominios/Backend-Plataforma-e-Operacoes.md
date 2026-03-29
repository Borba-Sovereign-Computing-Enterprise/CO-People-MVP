# 🔧 Backend — Plataforma & Operações

## Microsserviços
| Microsserviço | Responsabilidade |
| --- | --- |
| `admin-service` | Painel, RBAC data-driven via EDN, gestão de users e profissionais, configurações financeiras |
| `audit-service` | Log imutável append-only, consulta com filtros, export, retenção 5 anos |
| `report-service` | Geração assíncrona de relatórios CSV/PDF, controle por perfil |
| `security-service` | Criptografia AES-256-GCM via KMS, anonimização LGPD, fluxo de portabilidade |
