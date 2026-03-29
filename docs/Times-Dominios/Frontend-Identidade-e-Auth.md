# 🎨 Frontend — Identidade & Auth
> **Stack:** React 19 + TypeScript | **Arquitetura:** Microfrontend via Module Federation | **Paradigma:** Data-Driven com Zod schemas
> 

---

## Microfrontend App

| App | Rota base | Descrição |
| --- | --- | --- |
| `app-auth` | `/auth/*` | Cadastro, login, recuperação de senha, 2FA, confirmação de email |

---

## Telas e Rotas

| Rota | Tela |
| --- | --- |
| `/auth/register/patient` | Cadastro do paciente |
| `/auth/register/professional` | Cadastro do profissional (multi-step) |
| `/auth/login` | Login com email/senha |
| `/auth/login/2fa` | Verificação de código 2FA |
| `/auth/forgot-password` | Recuperação de senha |
| `/auth/reset-password` | Redefinição de senha |
| `/auth/confirm-email` | Confirmação de email (link) |

---

## Estado Global (Zustand)

| Store | Responsabilidade |
| --- | --- |
| `auth-store` | Token de acesso, tipo de usuário, status de sessão |
| `registration-store` | Estado multi-step do cadastro de profissional |

---

## Validações (Zod)

Schemas Zod espelham os schemas Malli do backend — fonte única de verdade por tipo de formulário: `PatientRegisterSchema`, `ProfessionalRegisterSchema`, `LoginSchema`, `ResetPasswordSchema`.
