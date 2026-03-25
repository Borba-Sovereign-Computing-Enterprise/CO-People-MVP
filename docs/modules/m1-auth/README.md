# Módulo 1 — Autenticação e Cadastro

> **Plataforma de Saúde** · Documentação Técnica v1.0 · Março 2026 · Confidencial

Este módulo controla o acesso à plataforma para ambos os tipos de usuário: **cliente (paciente)** e **profissional de saúde**. Garante a segurança dos dados e a rastreabilidade de ações dentro do sistema.

---

## 2.1 Cadastro do Cliente

Dados obrigatórios para criação de conta de paciente:

| Campo | Observação |
|---|---|
| Nome completo | — |
| CPF | Validado |
| Data de nascimento | — |
| E-mail | — |
| Número de celular | — |
| Endereço | Necessário para atendimentos domiciliares |
| Senha | Mínimo 8 caracteres, com critérios de segurança |

---

## 2.2 Cadastro do Profissional

Dados obrigatórios para criação de conta de profissional de saúde:

| Campo | Observação |
|---|---|
| Nome completo e CPF | — |
| Número de registro profissional | CRM, CRO, CREFITO, etc. |
| Especialidade(s) de atuação | — |
| Endereço do consultório | Se aplicável |
| Foto de perfil | — |
| Documento comprobatório de habilitação | — |
| Modalidade de atendimento | Presencial / Domiciliar / Ambos |
| Dados bancários | Para repasse de pagamentos |

---

## 2.3 Segurança e Tokens

A autenticação utiliza o padrão **JWT (JSON Web Token)** para garantir que apenas usuários autorizados acessem recursos protegidos.

- Tokens de acesso com expiração curta (ex: 15 minutos)
- Refresh tokens com expiração estendida (ex: 7 dias)
- Revogação de token em logout e troca de senha
- Autenticação em dois fatores (2FA) opcional via SMS ou app autenticador
- Histórico de logins para auditoria

---

## 2.4 Regras de Negócio

- Um usuário **não pode estar logado** como cliente e profissional simultaneamente na mesma sessão
- Profissionais passam por **validação manual de credenciais** antes da aprovação do cadastro
- Dados sigilosos (laudos, prontuários) só são acessíveis mediante **autenticação ativa**

---

*Próximo módulo: [M2 — Perfil do Profissional](../m2-profile/README.md)*
