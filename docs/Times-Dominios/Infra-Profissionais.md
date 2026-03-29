# Infra — Profissionais

---

## Banco de Dados

| Recurso | Tabelas |
| --- | --- |
| **PostgreSQL 16** | `professionals`, `professional_profiles`, `specialties`, `professional_specialties`, `professional_education`, `office_addresses`, `professional_documents`, `reviews`, `moderation_queue`, `professional_review_queue` |

---

## Object Storage

| Bucket | Conteúdo | Acesso |
| --- | --- | --- |
| `plataforma-saude-profiles` | Fotos de perfil | Público via CDN |
| `plataforma-saude-docs-private` | CRM/CRO + docs de habilitação | Presigned URL (1h) |

---

## Kafka

| Papel | Tópicos |
| --- | --- |
| Producer | `professional.approved`, `professional.suspended`, `review.published` |
| Consumer | `appointment.completed` (libera avaliação) |

---

## Observabilidade
| Métrica | Alerta |
| --- | --- |
| `profile.completeness_rate` | < 70% de perfis completos |
| `review.moderation_queue_size` | > 50 itens sem ação por 4h |
| `approval.queue.overdue` | Cadastros pendentes > 72h |
