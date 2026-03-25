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
| RN-AGE-
