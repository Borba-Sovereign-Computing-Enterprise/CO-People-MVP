# 🔔 M7 — Notificações

> Hub central de Notificações. Acesse as sub-páginas conforme seu papel.
> 

## Acesso Rápido

| Para quem | Sub-página |
| --- | --- |
| Product Manager / Designer | 📦 Produto & Negócio — canais, tabela de eventos, opt-out por categoria |
| Engenheiro / Tech Lead | ⚙️ Especificação Técnica — schema, pipeline Kafka, retry, janela de silêncio |

---

## Contexto rápido

Sistema de notificações **Kafka-driven** com 4 canais: push, e-mail, SMS e in-app. Retry automático com backoff exponencial. Janela de silêncio 22h–7h configurável por usuário. 2FA e eventos financeiros não são desativáveis.

---

> 👋 **Bem-vindo(a) ao módulo de Notificações.** Este módulo define como e quando a plataforma comunica eventos relevantes a pacientes e profissionais — garantindo que ninguém perca informações críticas sem gerar ruído desnecessário.
> 

---

## 👋 Por onde começar

Se você acabou de entrar neste módulo, siga esta ordem:

1. 📌 Leia a **Visão Geral** — entenda a filosofia do sistema de notificações
2. 📋 Consulte os **Canais de Notificação** — quando usar push, e-mail ou SMS
3. 🔁 Revise os **Eventos que disparam notificação** e as regras por perfil
4. ⚙️ Verifique as **Regras de Negócio** e os **Edge Cases**

---

## 🎯 Visão Geral

O sistema de notificações mantém pacientes e profissionais informados sobre eventos relevantes da plataforma. O princípio central é **notificar com propósito** — cada disparo deve ter relevância clara para o destinatário, no canal certo, no momento certo.

Notificações excessivas ou mal direcionadas degradam a experiência e levam ao cancelamento de permissões pelo usuário. O sistema deve ser **configurável, rastreável e respeitoso com a atenção do usuário**.

---

## 🗂️ Guia por Perfil

| Perfil | O que este módulo significa para você |
| --- | --- |
| **Paciente** | Ser avisado sobre confirmações, lembretes e atualizações da sua consulta sem ser sobrecarregado |
| **Profissional de Saúde** | Receber alertas sobre agenda, mensagens, avaliações e repasses financeiros |
| **Engenheiro Backend** | Implementação do serviço de disparo, filas de notificação, retry e rastreabilidade |
| **Engenheiro Frontend** | Exibição de notificações in-app, badge de não lidas e central de notificações |
| **Product Manager / Designer** | Definição de frequência, preferências do usuário e política de opt-out |

---

## 8.1 Canais de Notificação

| Canal | Uso recomendado | Requer permissão? |
| --- | --- | --- |
| **Push notification (app)** | Ações em tempo real: nova mensagem, lembrete de consulta | ✅ Sim — permissão do dispositivo |
| **E-mail** | Confirmações formais: agendamento, cancelamento, recibo | ✅ Sim — aceite nos termos |
| **SMS** | Lembretes críticos: consulta em X horas, código 2FA | ✅ Sim — número validado no cadastro |
| **In-app (central de notificações)** | Histórico de todos os eventos — sempre disponível, independente de permissões externas | ❌ Não — nativo da plataforma |

### Regras de negócio — Canais

| Regra | Descrição |
| --- | --- |
| RN-NOT-001 | Toda notificação enviada por canal externo (push, e-mail, SMS) deve também ser registrada na central in-app |
| RN-NOT-002 | Se o usuário revogar permissão de push, o sistema deve fallback para e-mail automaticamente |
| RN-NOT-003 | SMS deve ser usado apenas para eventos críticos — não pode ser usado para marketing ou engajamento |
| RN-NOT-004 | O usuário pode desativar canais individuais nas configurações, exceto a central in-app |
| RN-NOT-005 | Notificações de 2FA via SMS têm prioridade máxima e nunca podem ser desativadas pelo usuário |

---

## 8.2 Eventos que Disparam Notificação

### Eventos de Agendamento

| Evento | Paciente | Profissional | Canal recomendado |
| --- | --- | --- | --- |
| Agendamento confirmado | ✅ | ✅ | Push + E-mail |
| Lembrete 24h antes da consulta | ✅ | ✅ | Push + SMS |
| Lembrete 1h antes da consulta | ✅ | ✅ | Push + SMS |
| Cancelamento de consulta | ✅ | ✅ | Push + E-mail + SMS |
| Reagendamento de consulta | ✅ | ✅ | Push + E-mail |
| Consulta iniciada (profissional entrou na sala) | ✅ | ❌ | Push |

### Eventos de Comunicação

| Evento | Paciente | Profissional | Canal recomendado |
| --- | --- | --- | --- |
| Nova mensagem no chat | ✅ | ✅ | Push |
| Mensagem não respondida em 48h | ❌ | ✅ | Push + E-mail |
| Arquivo recebido no chat | ✅ | ✅ | Push |

### Eventos de Perfil e Cadastro

| Evento | Paciente | Profissional | Canal recomendado |
| --- | --- | --- | --- |
| Aprovação de cadastro | ❌ | ✅ | Push + E-mail |
| Rejeição de cadastro | ❌ | ✅ | Push + E-mail |
| Solicitação de complemento de dados | ❌ | ✅ | Push + E-mail |
| Alteração de senha realizada | ✅ | ✅ | E-mail |
| Código 2FA | ✅ | ✅ | SMS |

### Eventos Financeiros

| Evento | Paciente | Profissional | Canal recomendado |
| --- | --- | --- | --- |
| Pagamento confirmado | ✅ | ❌ | Push + E-mail |
| Repasse de pagamento realizado | ❌ | ✅ | Push + E-mail |
| Falha no pagamento | ✅ | ❌ | Push + E-mail + SMS |
| Reembolso processado | ✅ | ❌ | Push + E-mail |

### Eventos de Avaliação

| Evento | Paciente | Profissional | Canal recomendado |
| --- | --- | --- | --- |
| Solicitação de avaliação pós-consulta | ✅ | ❌ | Push + E-mail |
| Avaliação recebida | ❌ | ✅ | Push |

---

## ⚙️ Regras de Negócio Gerais

| Regra | Descrição |
| --- | --- |
| RN-NOT-006 | Lembretes de consulta (24h e 1h) são enviados apenas se a consulta não foi cancelada ou reagendada |
| RN-NOT-007 | Não devem ser enviadas mais de 3 notificações push em um intervalo de 1 hora para o mesmo usuário |
| RN-NOT-008 | Notificações de e-mail devem seguir o template do Design System da plataforma |
| RN-NOT-009 | Todo disparo de notificação deve ser registrado em log com: usuário, evento, canal, timestamp e status (entregue/falhou) |
| RN-NOT-010 | Em caso de falha no envio, o sistema deve fazer até 3 tentativas com intervalo exponencial (1min, 5min, 15min) |
| RN-NOT-011 | Notificações não devem ser enviadas entre 22h e 7h (horário local do usuário), exceto 2FA e cancelamentos de última hora |
| RN-NOT-012 | O usuário pode configurar preferências de notificação por categoria (agendamento, financeiro, mensagens, etc.) |
| RN-NOT-013 | Ao desinstalar o app, o token de push deve ser invalidado imediatamente no backend |

---

## 🔕 Preferências e Opt-out

| Categoria | Pode desativar? | Canal mínimo obrigatório |
| --- | --- | --- |
| Lembretes de consulta | ✅ Parcialmente | E-mail |
| Mensagens do chat | ✅ Sim | In-app |
| Eventos financeiros | ❌ Não | Push + E-mail |
| Cadastro e segurança | ❌ Não | E-mail + SMS |
| Avaliações | ✅ Sim | — |
| Marketing / engajamento | ✅ Sim | — |

---

## 🚨 Casos Especiais e Edge Cases

| Situação | Comportamento esperado |
| --- | --- |
| Usuário sem push habilitado | Fallback automático para e-mail; se também sem e-mail, registrar apenas in-app |
| Consulta cancelada nos últimos 60 minutos antes do horário | SMS + Push com prioridade máxima, independente do horário |
| Múltiplos dispositivos do mesmo usuário | Push enviado para todos os dispositivos com sessão ativa |
| Usuário muda de fuso horário | Janela de silêncio (22h–7h) recalculada com base no novo fuso detectado |
| Notificação de repasse com valor divergente | Disparar alerta para o profissional com detalhamento e link para suporte |
| Token de push expirado | Sistema tenta renovar token; em caso de falha, faz fallback para e-mail |
| E-mail de confirmação cai em spam | Fora do escopo de controle da plataforma — orientar usuário nas FAQs |

---

## 📊 Status do Módulo

| Item | Status |
| --- | --- |
| Especificação de regras de negócio | 🟡 Em andamento |
| Definição de provedor de push (FCM / OneSignal) | ⚪ Não iniciado |
| Definição de provedor de e-mail (SendGrid / SES) | ⚪ Não iniciado |
| Definição de provedor de SMS (Twilio / Zenvia) | ⚪ Não iniciado |
| Design dos templates de e-mail | ⚪ Não iniciado |
| Implementação backend (serviço de disparo + filas) | ⚪ Não iniciado |
| Implementação frontend (central in-app + badge) | ⚪ Não iniciado |
| Testes e QA | ⚪ Não iniciado |

---

## 🔗 Dependências com outros módulos

| Módulo | Tipo de dependência |
| --- | --- |
| M1 — Autenticação e Cadastro | Dados de contato (e-mail, telefone) e token de push vinculados ao usuário |
| M4 — Agendamento | Eventos de confirmação, lembrete e cancelamento |
| M5 — Pagamento | Eventos de confirmação de pagamento, repasse e reembolso |
| M6 — Comunicação | Disparo de push ao receber nova mensagem no chat |
| M8 — Segurança e LGPD | Consentimento para uso de dados de contato e log de auditoria |

---

## ❓ Decisões em Aberto

| Decisão | Contexto |
| --- | --- |
| Provedor de push | Firebase FCM, OneSignal ou outro? |
| Provedor de e-mail | SendGrid, Amazon SES ou outro? |
| Provedor de SMS | Twilio, Zenvia, Total Voice? |
| Sistema de filas | Bull/BullMQ (Redis), RabbitMQ ou serviço gerenciado? |
| Janela de silêncio configurável? | Usuário pode personalizar o horário de não perturbe? |
| Agrupamento de notificações | Agrupar múltiplos eventos em uma única notificação ou enviar individualmente? |

[📦 Produto & Negócio — Notificações](https://www.notion.so/Produto-Neg-cio-Notifica-es-3278f5aea1ae8179921ad7d1f7a68a19?pvs=21)

[⚙️ Especificação Técnica — Notificações](https://www.notion.so/Especifica-o-T-cnica-Notifica-es-3278f5aea1ae81ea96d2e0dfcc264f28?pvs=21)


