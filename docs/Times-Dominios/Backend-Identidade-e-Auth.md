# Backend — Identidade & Auth

> **Linguagem:** Clojure 1.12 | **Framework:** Pedestal | **Paradigma:** Data-Driven + Railway Oriented Programming
> 

---

## Microsserviços desta ilha

| Microsserviço | Responsabilidade |
| --- | --- |
| `identity-service` | Cadastro, CPF/email validation, soft delete, ciclo de vida do usuário |
| `auth-service` | Login, JWT (RS256), refresh token rotation, 2FA (TOTP), bloqueio de conta |
| `consent-service` | Consentimento LGPD por finalidade, versões, revogação, portabilidade |

---

## Dependências de infraestrutura

- **PostgreSQL** — tabelas: `users`, `patients`, `professionals`, `refresh_tokens`, `user_consents`, `audit_logs`
- **Redis** — blacklist de tokens revogados, rate limiting de login
- **AWS Secrets Manager** — chaves JWT RSA 2048
- **Kafka** — producer: `user.registered`, `user.approved`, `user.deleted`

---

## Eventos Kafka publicados

| Evento | Tópico |
| --- | --- |
| Usuário cadastrado | `user.registered` |
| Email confirmado | `user.email_confirmed` |
| Login realizado | `user.logged_in` |
| Conta bloqueada | `user.account_locked` |
| Dados anonimizados | `user.anonymized` |

---

## Referências

- Spec técnica completa: **⚙️ Especificação Técnica — Autenticação e Cadastro**
- Regras transversais: **⚙️ Especificação Técnica — Regras Transversais**
