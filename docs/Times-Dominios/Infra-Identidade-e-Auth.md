# ☁️ Infra — Identidade & Auth
## Banco de Dados

| Recurso | Uso |
| --- | --- |
| **PostgreSQL 16** | Tabelas: `users`, `patients`, `professionals`, `refresh_tokens`, `user_consents`, `audit_logs` |
| **Redis** | Blacklist de tokens revogados, rate limiting de login (5 tentativas/15min por IP) |

---

## Secrets & Chaves

| Recurso | Onde |
| --- | --- |
| Chave privada JWT (RSA 2048) | AWS Secrets Manager |
| Chave pública JWT | Config EDN via Aero |
| Secret de criptografia CPF (AES-256-GCM) | AWS KMS |

---

## Kafka

| Papel | Tópicos |
| --- | --- |
| Producer | `user.registered`, `user.email_confirmed`, `user.anonymized` |

---

## Observabilidade

| Métrica | Alerta |
| --- | --- |
| `auth.login_failed_rate` | > 20/min por IP — possível brute force |
| `auth.invalid_token_rate` | > 50/min — possível token theft |
| `auth.registration_rate` | Métrica de crescimento |

---

## Kubernetes

| Recurso | Config |
| --- | --- |
| Deployments | `identity-service`, `auth-service`, `consent-service` |
| HPA | Min 2 réplicas, max 10, target CPU 70% |
| Secrets | `external-secrets-operator` via AWS Secrets Manager |
