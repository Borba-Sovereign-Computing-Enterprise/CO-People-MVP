# Backend — Profissionais

> **Linguagem:** Clojure 1.12 | **Framework:** Pedestal | **Paradigma:** Data-Driven + Railway Oriented Programming
> 

---

## Microsserviços

| Microsserviço | Responsabilidade |
| --- | --- |
| `professional-service` | Perfil, especialidades, modalidades, configurações de agenda, ciclo de vida |
| `approval-service` | Fila de aprovação, optimistic lock, motivos padronizados, alertas de SLA 72h |
| `document-service` | Upload via S3 presigned URL, scan antivírus, acesso controlado por partes |
| `review-service` | Avaliações pós-consulta, moderação, cálculo de média, respostas públicas |

---

## Eventos Kafka

| Evento | Tópico |
| --- | --- |
| Profissional aprovado | `professional.approved` |
| Profissional suspenso | `professional.suspended` |
| Perfil atualizado | `professional.profile_updated` |
| Avaliação publicada | `review.published` |
