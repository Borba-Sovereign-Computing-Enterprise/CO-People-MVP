# Frontend — Profissionais

> **Stack:** React 19 + TypeScript | **Arquitetura:** Microfrontend via Module Federation
> 

---

## Microfrontend Apps

| App | Rota base | Descrição |
| --- | --- | --- |
| `app-professional-profile` | `/professional/*` | Perfil público, edição de perfil, especialidades, formação, avaliações |
| `app-professional-onboarding` | `/onboarding/*` | Fluxo multi-step de cadastro e submissão de documentos |

---

## Telas Principais
| Rota | Tela |
| --- | --- |
| `/professional/:id` | Perfil público (visão do paciente) |
| `/me/profile` | Edição de perfil (profissional autenticado) |
| `/me/profile/specialties` | Gestão de especialidades |
| `/me/profile/education` | Formação acadêmica |
| `/me/profile/addresses` | Endereços do consultório |
| `/me/reviews` | Avaliações recebidas + resposta |
