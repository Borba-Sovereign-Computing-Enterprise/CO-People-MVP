# 📌 Regras de Negócio Transversais

> Regras que se aplicam a **todos os módulos** da plataforma. Leia antes de começar qualquer feature. Se sua implementação conflita com alguma regra aqui, pergunte antes de prosseguir.
> 

---

## Acesso Rápido

| Para quem | Sub-página |
| --- | --- |
| Product Manager / QA / Stakeholder | 📦 Produto & Negócio — visão clara de cada regra e seu rationale |
| Engenheiro / Tech Lead | ⚙️ Especificação Técnica — como cada regra é enforced no código |

---

## Resumo das Regras (18 regras)

| # | Domínio | Regra resumida |
| --- | --- | --- |
| R01 | Acesso | Agendamento exige autenticação ativa |
| R02 | Acesso | Sem dual role simultâneo (paciente + profissional) |
| R03 | Acesso | Dados sigilosos só com sessão ativa |
| R04 | Acesso | CPF e email únicos por conta |
| R05 | Profissional | Sem aprovação = invisível na busca |
| R06 | Profissional | Suspensão é imediata |
| R07 | Profissional | Modalidade obrigatória para aparecer na busca |
| R08 | Consulta | COMPLETED só com confirmação bilateral |
| R09 | Consulta | Avaliação só após COMPLETED |
| R10 | Consulta | 1 avaliação por consulta por paciente |
| R11 | Documento | Laudo restrito às partes da consulta |
| R12 | Documento | Laudo isolado por consulta |
| R13 | Pagamento | Pagamento 100% dentro da plataforma |
| R14 | Pagamento | Repasse só após COMPLETED |
| R15 | Pagamento | Cancelamento pelo profissional = reembolso integral |
| R16 | Auditoria | Todo evento relevante gera audit log |
| R17 | Ciclo de vida | Perfis inativos 12m podem ser arquivados |
| R18 | Ciclo de vida | Soft delete em todas as entidades críticas |

[📦 Produto & Negócio — Regras Transversais](./Produto-e-Negocio-Regras-Transversais.md)

[⚙️ Especificação Técnica — Regras Transversais](./Especificacao-Tecnica-Regras-Transversais.md)
