# 🔔 Produto & Negócio — Notificações

> **Audiência:** Product Manager, Designer, Stakeholders
> 

> **Objetivo:** Definir quando, por qual canal e para quem cada notificação é enviada, e quais o usuário pode controlar.
> 

---

## 1. Filosofia

**Notificar com propósito.** Cada disparo deve ser relevante para o destinatário, no canal certo, no momento certo. Notificações excessivas levam à revogação de permissões e degradam a experiência. O sistema é **configurável, rastreável e respeitoso com a atenção do usuário**.

---

## 2. Canais Disponíveis

| Canal | Uso recomendado | Requer permissão? |
| --- | --- | --- |
| **Push (app/web)** | Ações em tempo real: nova mensagem, lembrete | ✅ Sim — permissão do dispositivo |
| **E-mail** | Confirmações formais: agendamento, recibo, aprovação | ✅ Sim — aceite nos termos |
| **SMS** | Lembretes críticos e 2FA | ✅ Sim — número validado |
| **In-app (central)** | Histórico de todos os eventos | ❌ Não — sempre disponível |

> **Regra:** Se push não autorizado → fallback automático para e-mail. Se e-mail também sem permissão → apenas in-app.
> 

---

## 3. Tabela Completa de Eventos

### Agendamento

| Evento | Paciente | Profissional | Canal |
| --- | --- | --- | --- |
| Agendamento confirmado | ✅ | ✅ | Push + E-mail |
| Lembrete 24h antes | ✅ | ✅ | Push + SMS |
| Lembrete 1h antes | ✅ | ✅ | Push + SMS |
| Cancelamento | ✅ | ✅ | Push + E-mail + SMS |
| Reagendamento | ✅ | ✅ | Push + E-mail |

### Comunicação

| Evento | Paciente | Profissional | Canal |
| --- | --- | --- | --- |
| Nova mensagem no chat | ✅ | ✅ | Push |
| Mensagem não respondida em 48h | ❌ | ✅ | Push + E-mail |
| Arquivo recebido | ✅ | ✅ | Push |

### Cadastro & Perfil

| Evento | Paciente | Profissional | Canal |
| --- | --- | --- | --- |
| Aprovação de cadastro | ❌ | ✅ | Push + E-mail |
| Rejeição de cadastro | ❌ | ✅ | Push + E-mail |
| Alteração de senha | ✅ | ✅ | E-mail |
| Código 2FA | ✅ | ✅ | SMS (não desativável) |

### Financeiro

| Evento | Paciente | Profissional | Canal |
| --- | --- | --- | --- |
| Pagamento confirmado | ✅ | ❌ | Push + E-mail |
| Repasse realizado | ❌ | ✅ | Push + E-mail |
| Falha no pagamento | ✅ | ❌ | Push + E-mail + SMS |
| Reembolso processado | ✅ | ❌ | Push + E-mail |

### Avaliações

| Evento | Paciente | Profissional | Canal |
| --- | --- | --- | --- |
| Convite para avaliar | ✅ | ❌ | Push + E-mail |
| Avaliação recebida | ❌ | ✅ | Push |

---

## 4. Controle do Usuário — Opt-out

| Categoria | Pode desativar? | Mínimo obrigatório |
| --- | --- | --- |
| Lembretes de consulta | ✅ Parcialmente | E-mail |
| Mensagens do chat | ✅ Sim | In-app |
| Eventos financeiros | ❌ Não | Push + E-mail |
| Segurança e 2FA | ❌ Não | E-mail + SMS |
| Avaliações | ✅ Sim | — |
| Marketing / engajamento | ✅ Sim | — |

---

## 5. Regras de Negócio Chave

| Código | Regra |
| --- | --- |
| RN-NOT-001 | Todo envio por canal externo também registra na central in-app |
| RN-NOT-002 | Fallback automático push → e-mail se permissão revogada |
| RN-NOT-003 | SMS só para eventos críticos — nunca marketing |
| RN-NOT-006 | Lembretes só disparam se a consulta ainda não foi cancelada/reagendada |
| RN-NOT-007 | Máx. 3 push por hora para o mesmo usuário |
| RN-NOT-009 | Todo disparo logado com: usuário, evento, canal, timestamp, status |
| RN-NOT-010 | Retry: até 3 tentativas com exponential backoff (1min, 5min, 15min) |
| RN-NOT-011 | Janela de silêncio: 22h–7h (horário local), exceto 2FA e cancelamento de última hora |

---

## 6. Casos Especiais

| Situação | Comportamento |
| --- | --- |
| Sem push habilitado | Fallback para e-mail; se também sem e-mail, só in-app |
| Cancelamento nos últimos 60min | SMS + Push com prioridade máxima, ignora janela de silêncio |
| Múltiplos dispositivos do mesmo usuário | Push enviado para todos os dispositivos com sessão ativa |
| Token de push expirado | Tenta renovar; falha → fallback para e-mail |

---

## 7. Critérios de Aceite (MVP)

- [ ]  Confirmacao de agendamento chega em até 2 minutos
- [ ]  Lembrete 24h é enviado apenas para consultas com status `CONFIRMED`
- [ ]  Cancelamento fora da janela de silêncio dispara SMS + Push
- [ ]  Usuário sem push recebe fallback para e-mail automaticamente
- [ ]  Central in-app sempre atualizada, independente de permissões externas
- [ ]  Falha de envio gera retry até 3 vezes com backoff

---

## 8. Fluxograma (LucidChart)

> 🔗 **[Inserir link — Fluxo de Notificações]**
> 

> Cobrir: evento → verifica preferências → seleciona canal → envia → retry → log
>
