# Produto & Negócio — Agendamento

> **Audiência:** Product Manager, Designer, Stakeholders, QA
> 

> **Objetivo:** Descrever como o agendamento funciona do ponto de vista do paciente e do profissional, as regras de cancel e reagendamento, e os critérios de aceite do MVP.
> 

---

## 1. Por que este módulo é o coração do produto

O agendamento é a conversão principal da plataforma — tudo antes dele (cadastro, busca, perfil) existe para chegar aqui. Ele precisa ser **rápido, claro e sem ambiguidade**: o paciente sabe exatamente o que reservou, o profissional sabe exatamente o que confirmou, e ambos têm certeza do que acontece em caso de cancelamento.

---

## 2. Jornada do Paciente — Agendar
```

Passo 1: Paciente acessa o perfil do profissional
→ Visualiza horários disponíveis no calendário

Passo 2: Seleciona data e horário
→ Seleciona também a modalidade: presencial ou domiciliar
→ Se domiciliar: confirma ou informa endereço de atendimento

Passo 3: Revisa o resumo
→ Profissional, data, horário, modalidade, valor e endereço

Passo 4: Realiza o pagamento
→ Escolhe forma de pagamento (M5)
→ Pagamento aprovado → agendamento confirmado

Passo 5: Confirmação
→ Recebe e-mail e SMS de confirmação
→ Consulta aparece no histórico do paciente
→ Profissional recebe notificação

Resultado: Status = CONFIRMED
```
---

## 3. Jornada do Profissional — Gerenciar Agenda
```
Configuração (uma vez ou periodicamente):
→ Define dias e horários de atendimento
→ Define antecêdencia mínima para agendamento (ex: 2h)
→ Define antecêdencia mínima para cancelamento sem multa (ex: 24h)
→ Define limite de consultas por dia (opcional)

No dia a dia:
→ Recebe notificação de novo agendamento
→ Visualiza agenda do dia/semana
→ Pode bloquear horários (férias, indisponibilidade)
→ Pode cancelar consulta (com justificativa)
```

## 4. Estados de uma Consulta

| Status | Significado | Quem pode ver |
| --- | --- | --- |
| `PENDING` | Agendamento criado, aguardando pagamento | Paciente + Sistema |
| `CONFIRMED` | Pagamento aprovado, consulta marcada | Ambos + Admin |
| `COMPLETED` | Ambas as partes confirmaram a realização | Ambos + Admin |
| `CANCELLED` | Cancelado pelo paciente dentro do prazo | Ambos + Admin |
| `CANCELLED_LATE` | Cancelado fora do prazo — taxa pode ser cobrada | Ambos + Admin |
| `CANCELLED_BY_PROFESSIONAL` | Cancelado pelo profissional — reembolso total | Ambos + Admin |
| `NO_SHOW` | Nenhuma das partes confirmou após a consulta | Admin |
| `REJECTED` | Profissional recusou o agendamento (se fluxo permitir) | Ambos |

---

## 5. Regras de Cancelamento e Reagendamento

| Situação | Prazo | Impacto financeiro |
| --- | --- | --- |
| Paciente cancela dentro do prazo | Configurável pelo profissional (ex: 24h antes) | Reembolso integral |
| Paciente cancela fora do prazo | Após prazo configurado | Taxa configurável (ex: 20% do valor) |
| Profissional cancela (qualquer momento) | — | Reembolso integral ao paciente, sem multa |
| Reagendamento pelo paciente | Dentro do prazo de cancel | Sem custo adicional |
| Reagendamento fora do prazo | Após prazo configurado | Tratado como cancelamento + novo agendamento |

> **Regra transversal R08:** Consulta só vai para `COMPLETED` após confirmação de ambas as partes.
> 

---

## 6. Gestão de Disponibilidade

- O profissional define sua grade de horários no painel
- O paciente vê **apenas** horários livres — horários ocupados não são exibidos
- Horários bloqueados (férias, pausa) ficam invisíveis para o paciente
- O sistema evita **double booking**: dois pacientes não podem reservar o mesmo slot
- Antecêdencia mínima: profissional configura (ex: “só aceito agendamentos com 2h de antecêdencia”)

---

## 7. Limites e Configurações

| Regra | Valor | Quem configura |
| --- | --- | --- |
| Antecêdencia mínima para agendamento | 1h a 72h | Profissional |
| Antecêdencia para cancelamento sem multa | 0h a 72h | Profissional |
| Máximo de consultas por dia | 1 a 50 | Profissional |
| Janela máxima de agendamentos futuros | 30 a 180 dias | Admin (global) |
| Máximo de agendamentos ativos por paciente | Configurável | Admin (global) |

---

## 8. Histórico de Ações

Toda alteração em um agendamento é registrada com:

- **Quem agiu** — paciente, profissional ou sistema
- **O que foi feito** — criou, confirmou, cancelou, reagendou
- **Data e hora exata** — determina se estava dentro ou fora do prazo
- **Estado anterior → novo estado**
- **Motivo** — campo livre, obrigatório em cancelamentos
- **Impacto financeiro** — reembolso, taxa ou sem alteração

---

## 9. Notificações do Módulo de Agendamento

| Evento | Quem recebe | Canal |
| --- | --- | --- |
| Agendamento confirmado | Paciente + Profissional | E-mail + SMS |
| Lembrete 24h antes | Paciente + Profissional | E-mail + SMS |
| Lembrete 1h antes | Paciente | SMS |
| Cancelamento | Ambos | E-mail |
| Reagendamento | Ambos | E-mail |
| Consulta concluída — convite p/ avaliação | Paciente | E-mail |

---

## 10. Critérios de Aceite (MVP)

- [ ]  Paciente consegue agendar em menos de 3 cliques após escolher o profissional
- [ ]  Double booking é impossível — sistema bloqueia concorrência
- [ ]  Cancelamento dentro do prazo gera reembolso integral automático
- [ ]  Cancelamento pelo profissional gera reembolso integral sem ação manual
- [ ]  Todo cancelamento é registrado no histórico com timestamp e motivo
- [ ]  Paciente recebe e-mail de confirmação em até 2 minutos
- [ ]  Consulta só vai para COMPLETED com confirmação de ambas as partes
- [ ]  Profissional pode bloquear horários via painel

---

## 11. Fluxograma (LucidChart)

> 🔗 **[Inserir link do LucidChart — Fluxo de Agendamento]**
> 

> Cobrir: agendamento (happy path) • cancelamento por paciente • cancelamento por profissional • confirmação bilateral de realização
