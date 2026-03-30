# 📦 Produto & Negócio — Regras Transversais
> **Audiência:** Product Manager, Designer, Stakeholders, QA
> 

> **Objetivo:** Registrar de forma clara e rastreável todas as regras que atravessam módulos — com rationale de negócio para cada uma.
> 

---

## O que são regras transversais?

São regras que **não pertencem a um único módulo** — elas existem em múltiplos contextos e precisam ser respeitadas em toda a plataforma. Se uma feature quebra uma regra transversal, o produto inteiro pode ser comprometido.

> Leia este documento antes de começar qualquer módulo. Se tiver dúvida se sua feature conflita com alguma regra aqui, pergunte antes de implementar.
> 

---

## 1. Identidade & Acesso

| # | Regra | Rationale |
| --- | --- | --- |
| R01 | Todo agendamento exige autenticação ativa do paciente | Rastreabilidade, responsabilização e conformidade LGPD — toda ação sobre dados de saúde precisa de identidade verificada |
| R02 | Um usuário não pode estar logado como paciente e profissional ao mesmo tempo | Evita conflito de permissões e ações indevidas em nome errado |
| R03 | Dados sigilosos (laudos, prontuários, histórico) só são acessíveis com sessão ativa | Proteção de dados sensíveis de saúde — dado nunca exposto a sessão expirada |
| R04 | CPF e e-mail são únicos por conta na plataforma | Evita identidades duplicadas e fraude de cadastro |

---

## 2. Profissional & Aprovação

| # | Regra | Rationale |
| --- | --- | --- |
| R05 | Profissionais não aprovados não aparecem nas buscas nem recebem agendamentos | A plataforma só oferece profissionais com credenciais verificadas — é um pilar de confiança do produto |
| R06 | Profissionais suspensos perdem visibilidade imediatamente, sem período de carência | Risco de dano ao paciente não admite delay — suspensão é instantânea |
| R07 | Profissional deve ter ao menos uma modalidade configurada para aparecer na busca | Sem modalidade, não há como o paciente agendar — evita cadastro incompleto visível |

---

## 3. Consulta & Confirmação

| # | Regra | Rationale |
| --- | --- | --- |
| R08 | Uma consulta só é marcada como "Realizada" após confirmação de **ambas as partes** | Evita unilateralidade — protege o paciente de consultas marcadas como feitas sem ter ocorrido, e o profissional de no-show não registrado |
| R09 | Avaliações só são habilitadas após consulta com status `COMPLETED` | Avaliação sem consulta real é desonesta — protege reputação do profissional e credibilidade do sistema |
| R10 | Cada consulta permite apenas 1 avaliação por paciente | Evita spam de avaliações e manipulação de nota |

---

## 4. Documentos & Privacidade

| # | Regra | Rationale |
| --- | --- | --- |
| R11 | Laudos e documentos de saúde são acessíveis apenas pelas partes da consulta (paciente + profissional) | Dado de saúde é categoria especial pela LGPD (Art. 11) — acesso estritamente restrito |
| R12 | Documentos não são compartilhados entre consultas distintas | Cada consulta tem escopo isolado — laudo de uma consulta não é visível em outra mesmo com o mesmo profissional |

---

## 5. Pagamento & Financeiro

| # | Regra | Rationale |
| --- | --- | --- |
| R13 | Toda negociação financeira ocorre exclusivamente dentro da plataforma | Garante rastreabilidade, proteção ao consumidor e receita da plataforma |
| R14 | O profissional só recebe o repasse após a consulta ser marcada como realizada | Evita fraude de agendamento + recebimento sem prestação de serviço |
| R15 | Cancelamentos pelo profissional resultam em reembolso integral ao paciente | O profissional é responsável por sua disponibilidade — o paciente não arca com custos de falha do lado do prestador |

---

## 6. Auditoria & Ciclo de Vida

| # | Regra | Rationale |
| --- | --- | --- |
| R16 | Toda comunicação e ação relevante gera registro de log imutável para auditoria | LGPD exige rastreabilidade; disputas comerciais precisam de histórico confiável |
| R17 | Perfis inativos por mais de 12 meses podem ser arquivados pelo administrador | Evita dados desatualizados expostos na busca e reduz superfície de ataque |
| R18 | Dados nunca são deletados fisicamente — apenas marcados como soft delete | Rastreabilidade, recuperação de dados e obrigações legais de retenção |

---

## 7. Critérios de Aceite Transversais

- [ ]  Nenhuma feature permite agendamento sem sessão ativa
- [ ]  Profissional pendente/rejeitado/suspenso nunca aparece na busca
- [ ]  Todo evento relevante (login, cancelamento, pagamento, acesso a laudo) gera entry no audit log
- [ ]  Nenhum dado de saúde é retornado em endpoint sem autenticação
- [ ]  Soft delete em todas as entidades críticas

---

## 8. Fluxograma (LucidChart)

> 🔗 **[Inserir link do LucidChart — Mapa de Regras Transversais por Módulo]**
> 

> Cobrir: qual regra afeta qual módulo, dependências e pontos de enforcement
>
