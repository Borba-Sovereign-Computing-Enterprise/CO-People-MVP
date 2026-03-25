# 📅 M4 — Agendamento

> 👋 **Bem-vindo(a) ao módulo de Agendamento.** O agendamento é a conversão principal da plataforma — é aqui que a intenção do paciente se transforma em consulta marcada. Este hub centraliza as referências do módulo e orienta cada perfil para a sub-página mais relevante.

---

## 👋 Por onde começar

Se você acabou de entrar neste módulo, siga esta ordem:

1. 📌 Leia a **Visão Geral** — entenda o papel central do agendamento na plataforma  
2. 🗂️ Consulte o **Guia por Perfil** — e acesse a sub-página certa para o seu papel  
3. 🔁 Revise o **Fluxo de Agendamento** passo a passo  
4. 📆 Leia as regras de **Calendário e Visibilidade**  
5. ⚙️ Verifique os **Limites**, **Cancelamentos** e **Edge Cases**

---

## 🎯 Visão Geral

O agendamento é a **conversão principal da plataforma** — o momento em que o paciente transforma uma busca em uma consulta confirmada.

O módulo envolve:

- **Conflict detection** — prevenção de double booking em tempo real  
- **State machine de 8 estados** — controle preciso do ciclo de vida de cada consulta  
- **Confirmação bilateral** — ambas as partes confirmam a realização  
- **Integração direta com M5** — pagamento é parte do fluxo de agendamento, não uma etapa separada  

---

## 🗂️ Guia por Perfil

| Perfil | Comece por |
| --- | --- |
| **Product Manager / Designer** | 📦 Sub-página Produto & Negócio — jornadas, estados, regras de cancelamento, histórico e critérios de aceite |
| **QA / Tester** | 📦 Sub-página Produto & Negócio — critérios de aceite e edge cases |
| **Engenheiro Backend / Tech Lead** | ⚙️ Sub-página Especificação Técnica — schema, state machine, conflict detection, endpoints e Kafka events |
| **Engenheiro Frontend** | 📦 Produto & Negócio (fluxo e estados) → ⚙️ Especificação Técnica (contratos de API e eventos) |

---

## 📦 Acesso Rápido às Sub-páginas

| Sub-página | Conteúdo |
| --- | --- |
| 📦 Produto & Negócio — Agendamento | Jornadas, estados, regras de cancelamento, histórico e critérios de aceite |
| ⚙️ Especificação Técnica — Agendamento | Schema, state machine, conflict detection, endpoints e Kafka events |

---

## 5.1 Fluxo de Agendamento

Cliente autenticado acessa o perfil do profissional
↓

Visualiza calendário com slots disponíveis
↓

Seleciona data, horário e modalidade (presencial / domiciliar)
↓

Revisa resumo e confirma o agendamento
↓

Pagamento processado (pré-agendamento ou no dia — conforme configuração do profissional)
↓

Ambos recebem notificação de confirmação (M7)
↓

Consulta aparece no calendário de ambos


### Regras de negócio — Fluxo

| Regra | Descrição |
| --- | --- |
| RN-AGE-001 | O paciente deve estar autenticado para agendar — agendamento anônimo não é permitido |
| RN-AGE-002 | O slot selecionado deve ser bloqueado temporariamente assim que o paciente iniciar o processo de confirmação, por até 10 minutos |
| RN-AGE-003 | Se o pagamento não for concluído dentro do tempo de bloqueio, o slot é liberado automaticamente |
| RN-AGE-004 | O agendamento só é considerado confirmado após o pagamento ser processado com sucesso |
| RN-AGE-005 | Ambos — paciente e profissional — devem receber notificação de confirmação imediatamente após o agendamento |
| RN-AGE-006 | O paciente não pode agendar com o mesmo profissional em horários sobrepostos |

---

## 5.2 Estados do Agendamento (State Machine)

Cada consulta percorre um ciclo de vida com 8 estados possíveis:

| Estado | Descrição | Transições possíveis |
| --- | --- | --- |
| `PENDENTE` | Agendamento iniciado, aguardando pagamento | → CONFIRMADO, → EXPIRADO |
| `CONFIRMADO` | Pagamento concluído, consulta agendada | → REAGENDADO, → CANCELADO_CLIENTE, → CANCELADO_PROFISSIONAL, → REALIZADO |
| `REAGENDADO` | Data ou horário alterado por uma das partes | → CONFIRMADO, → CANCELADO_CLIENTE, → CANCELADO_PROFISSIONAL |
| `CANCELADO_CLIENTE` | Cancelado pelo paciente | → (estado final) |
| `CANCELADO_PROFISSIONAL` | Cancelado pelo profissional | → (estado final) |
| `CANCELADO_SISTEMA` | Cancelado automaticamente (ex: inadimplência, fraude) | → (estado final) |
| `REALIZADO` | Consulta marcada como realizada por ambas as partes | → (estado final) |
| `EXPIRADO` | Slot não foi confirmado dentro do prazo de bloqueio | → (estado final) |

### Regras de negócio — Estados

| Regra | Descrição |
| --- | --- |
| RN-AGE-007 | Nenhum estado pode ser revertido — o histórico de transições é imutável |
| RN-AGE-008 | O estado `REALIZADO` só pode ser atingido após confirmação de ambas as partes ou decurso do prazo de confirmação automática |
| RN-AGE-009 | Se o profissional não marcar a consulta como realizada em até 24h após o horário, o sistema confirma automaticamente |
| RN-AGE-010 | Cancelamentos com estado `CANCELADO_SISTEMA` devem gerar alerta interno para o time de operações |

---

## 5.3 Calendário e Visibilidade

| Regra | Descrição |
| --- | --- |
| RN-AGE-011 | O profissional cadastra seus horários disponíveis — o sistema exibe apenas slots livres ao paciente |
| RN-AGE-012 | Slots já ocupados, bloqueados ou fora da janela de antecedência mínima não são exibidos ao paciente |
| RN-AGE-013 | O profissional visualiza todos os seus agendamentos confirmados, pendentes e histórico |
| RN-AGE-014 | O paciente visualiza apenas seus próprios agendamentos — nunca a agenda completa do profissional |
| RN-AGE-015 | Dois pacientes não podem reservar o mesmo slot simultaneamente — conflict detection obrigatório |
| RN-AGE-016 | Integração com Google Calendar / Apple Calendar é planejada para versão futura — não é requisito do MVP |

---

## 5.4 Limites de Agendamento

| Regra | Valor padrão | Quem configura |
| --- | --- | --- |
| Máximo de agendamentos ativos por paciente/mês por especialidade | 5 | Admin (M9) |
| Antecedência mínima para agendamento | 2 horas | Profissional |
| Antecedência mínima para cancelamento sem multa | 24 horas | Profissional |
| Janela máxima de reagendamento | Configurável | Profissional |
| Tempo máximo de bloqueio de slot durante pagamento | 10 minutos | Sistema |

### Regras de negócio — Limites

| Regra | Descrição |
| --- | --- |
| RN-AGE-017 | Paciente que atingir o limite de agendamentos ativos vê os slots disponíveis mas não consegue confirmar — com mensagem explicativa |
| RN-AGE-018 | Profissional pode definir antecedência mínima diferente por tipo de consulta (presencial vs. domiciliar) |
| RN-AGE-019 | Limites configurados pelo admin sobrepõem os do profissional quando forem mais restritivos |
| RN-AGE-020 | Alterações nos limites pelo admin ou profissional se aplicam apenas a novos agendamentos — nunca retroativamente |

---

## 5.5 Cancelamentos e Reagendamentos

### Regras de negócio — Cancelamento pelo paciente

| Regra | Descrição |
| --- | --- |
| RN-AGE-021 | Cancelamento dentro do prazo mínimo configurado pelo profissional não gera cobrança — reembolso integral |
| RN-AGE-022 | Cancelamento fora do prazo pode gerar cobrança de taxa configurável pelo profissional (ex: 30% do valor da consulta) |
| RN-AGE-023 | A taxa de cancelamento tardio deve ser exibida claramente ao paciente no momento do agendamento — sem surpresas |
| RN-AGE-024 | Reembolso deve ser processado automaticamente via M5, no mesmo meio de pagamento utilizado |

### Regras de negócio — Cancelamento pelo profissional

| Regra | Descrição |
| --- | --- |
| RN-AGE-025 | Cancelamento pelo profissional nunca gera cobrança ao paciente — reembolso integral e automático |
| RN-AGE-026 | Profissional deve informar um motivo ao cancelar — campo obrigatório |
| RN-AGE-027 | Paciente deve ser notificado imediatamente via push + SMS ao ter uma consulta cancelada pelo profissional |
| RN-AGE-028 | Profissional que cancelar mais de 3 consultas no mesmo mês recebe alerta interno do time de operações |

### Regras de negócio — Reagendamento

| Regra | Descrição |
| --- | --- |
| RN-AGE-029 | Reagendamento só é permitido dentro da janela configurada pelo profissional |
| RN-AGE-030 | O novo horário proposto deve passar por conflict detection antes de ser confirmado |
| RN-AGE-031 | Ambos devem ser notificados ao confirmar um reagendamento |
| RN-AGE-Reagendamento iniciado pelo profissional não gera cobrança — o paciente pode aceitar ou cancelar com reembolso integral|

### 5.6 Histórico de Alterações
Todo evento no ciclo de vida de um agendamento deve ser registrado com os seguintes dados:

| Campo | Descrição |
| --- | --- |
| **Ator** | Quem fez a ação — paciente, profissional ou sistema automático |
| **Ação** | O que foi feito — cancelamento, reagendamento, confirmação, expiração |
| **Timestamp** | Data e hora exata — determina se estava dentro ou fora do prazo |
| **Estado anterior** | Estado do agendamento antes da ação |
| **Estado posterior** | Estado do agendamento após a ação |
| **Motivo** | Campo opcional para o usuário, obrigatório para ações do sistema |
| **Resultado financeiro** | Se houve reembolso, valor, taxa cobrada e canal de pagamento afetado |

### Regras de negócio — Histórico

| Regra | Descrição |
| --- | --- |
| RN-AGE-033 | O histórico de alterações é imutável — nenhum usuário ou admin pode editá-lo |
| RN-AGE-034 | Paciente e profissional têm acesso ao histórico de seus próprios agendamentos |
| RN-AGE-035 | O time interno acessa o histórico completo via M9, com registro de auditoria do acesso |
| RN-AGE-036 | Histórico deve ser retido por no mínimo 5 anos (conformidade com CFM e LGPD) |

---

## 🚨 Casos Especiais e Edge Cases

| Situação | Comportamento esperado |
| --- | --- |
| Dois pacientes tentam reservar o mesmo slot ao mesmo tempo | Conflict detection: apenas o primeiro que concluir o pagamento confirma — o segundo recebe mensagem de slot indisponível |
| Paciente não conclui o pagamento no prazo de bloqueio | Slot liberado automaticamente após 10 minutos; paciente recebe notificação de expiração |
| Profissional cancela com menos de 1h de antecedência | Reembolso integral + notificação com prioridade máxima (push + SMS) + registro para monitoramento interno |
| Paciente tenta agendar fora da janela de antecedência mínima | Slot não aparece como disponível — sem mensagem de erro, apenas omissão do slot |
| Consulta domiciliar com endereço não informado | Sistema bloqueia a confirmação e solicita o endereço antes de prosseguir |
| Profissional altera a antecedência mínima após agendamentos confirmados | A nova regra se aplica apenas a agendamentos futuros — os já confirmados mantêm a regra original |
| Paciente menor de 18 anos agenda sem responsável | Sistema deve exigir vinculação com responsável cadastrado antes de permitir o agendamento |
| Falha no pagamento durante o bloqueio do slot | Slot permanece bloqueado enquanto o pagamento está sendo reprocessado — libera após esgotamento das tentativas |

---

## 📊 Status do Módulo

| Item | Status |
| --- | --- |
| Especificação de regras de negócio (hub) | 🟡 Em andamento |
| Sub-página Produto & Negócio | ⚪ Não iniciado |
| Sub-página Especificação Técnica | ⚪ Não iniciado |
| Definição da state machine (8 estados) | ⚪ Não iniciado |
| Implementação de conflict detection | ⚪ Não iniciado |
| Implementação backend (endpoints + eventos) | ⚪ Não iniciado |
| Implementação frontend (calendário + fluxo) | ⚪ Não iniciado |
| Integração com M5 — Pagamento | ⚪ Não iniciado |
| Testes e QA | ⚪ Não iniciado |

---

## 🔗 Dependências com outros módulos

| Módulo | Tipo de dependência |
| --- | --- |
| M1 — Autenticação e Cadastro | Identidade do paciente e do profissional para vincular o agendamento |
| M2 — Perfil do Profissional | Dados do profissional exibidos no fluxo + configurações de agenda e antecedência |
| M3 — Busca e Categorias | Filtro de disponibilidade consome a agenda em tempo real |
| M5 — Pagamento | Pagamento é parte obrigatória do fluxo de confirmação |
| M7 — Notificações | Confirmações, lembretes, cancelamentos e reagendamentos |
| M8 — Segurança e LGPD | Histórico imutável, retenção de dados e acesso auditado |
| M9 — Painel Administrativo | Configuração de limites e monitoramento de cancelamentos excessivos |

---

## ❓ Decisões em Aberto

| Decisão | Contexto |
| --- | --- |
| Mecanismo de conflict detection | Pessimistic lock no banco, Redis lock distribuído ou outra abordagem? |
| Confirmação de realização | Ambas as partes confirmam ativamente ou apenas o profissional? |
| Prazo de confirmação automática | 24h é o valor correto ou deve ser configurável por especialidade? |
| Integração com Google / Apple Calendar | Para qual versão está prevista? Quais dados sincronizar? |
| Kafka vs. outros event brokers | Kafka é o escolhido para os eventos de estado? |
| Cancelamento automático por sistema | Quais condições além de expiração de pagamento disparam `CANCELADO_SISTEMA`? |

> Hub central de Agendamento. Acesse as sub-páginas conforme seu papel.

---

## Acesso Rápido

| Para quem | Sub-página |
| --- | --- |
| Product Manager / Designer / QA | 📦 Produto & Negócio — jornadas, estados, regras de cancel, histórico e critérios de aceite |
| Engenheiro / Tech Lead | ⚙️ Especificação Técnica — schema, state machine, conflict detection, endpoints e Kafka events |

---

## Contexto rápido

O agendamento é a **conversão principal** da plataforma. Envolve conflict detection (double booking), state machine de 8 estados, confirmação bilateral de realização e integração direta com M5 (Pagamento).

## 5.1 Fluxo de agendamento

**Etapas do agendamento**

---

- 1. Cliente está autenticado na plataforma  
    1. Cliente está autenticado na plataforma  

---

- 2. Cliente acessa o perfil do profissional desejado  
    1. Cliente acessa o perfil do profissional desejado  

---

- 3. Cliente visualiza o calendário com dias e horários disponíveis  
    1. Cliente visualiza o calendário com dias e horários disponíveis  

---

- 4. Cliente seleciona data, horário e modalidade (presencial/domiciliar)  
    1. Cliente seleciona data, horário e modalidade (presencial/domiciliar)  

---

- 5. Cliente revisa dados e confirma o agendamento  
    1. Cliente revisa dados e confirma o agendamento  

---

- 6. Pagamento é processado (pré-agendamento ou no dia)  
    1. Pagamento é processado (pré-agendamento ou no dia)  

---

- 7. Ambos recebem notificação de confirmação  
    1. Ambos recebem notificação de confirmação  

---

- 8. Consulta aparece no calendário de ambos  
    1. Consulta aparece no calendário de ambos  

---

## 5.2 Calendário e visibilidade

- O profissional cadastra seus horários disponíveis no sistema  
- O cliente visualiza apenas os slots disponíveis (não ocupados)  
- O profissional tem acesso ao calendário do cliente para ver os próximos passos pós-consulta  
- Integração futura com Google Calendar / Apple Calendar (recomendada)  

## 5.3 Limite de agendamentos

Devem ser definidos limites para evitar sobrecarga ou abusos do sistema:

| **Regra** | **Valor sugerido** |
| --- | --- |
| Máximo de agendamentos ativos por cliente/mês | Configurável pelo admin (ex: 5 por especialidade) |
| Antecedência mínima para agendamento | Configurável pelo profissional (ex: 2h) |
| Antecedência mínima para cancelamento sem multa | Configurável pelo profissional (ex: 24h) |
| Janela de reagendamento | Até X horas antes da consulta, configurável |

## 5.4 Cancelamentos e reagendamentos

- Cliente pode cancelar ou reagendar com antecedência mínima definida  
- Cancelamentos tardios podem gerar cobrança de taxa (configurável por profissional)  
- Profissional também pode cancelar; nesse caso, sem cobrança ao cliente e com notificação imediata  
- Histórico completo de alterações é armazenado para auditoria  

**O histórico deve conter:**

- **Quem fez a ação** — cliente, profissional ou sistema automático
- **O que foi feito** — cancelamento, reagendamento ou tentativa não concluída
- **Data e hora exata** — determina se estava dentro ou fora do prazo
- **Estado antes e depois** — agendamento original e para o quê foi alterado (data, horário, modalidade)
- **Motivo informado** — campo opcional para o usuário, mas deve sempre existir no sistema
- **Resultado financeiro** — se houve reembolso, valor, taxa cobrada e canal de pagamento afetado
