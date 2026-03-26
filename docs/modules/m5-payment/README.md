# 💳 M5 — Pagamento

> 👋 **Bem-vindo(a) ao módulo de Pagamento.** Todo pagamento ocorre dentro da plataforma — sem redirecionamentos externos. Este hub centraliza as referências do módulo e orienta cada perfil para a sub-página mais relevante.

---

## 👋 Por onde começar

Se você acabou de entrar neste módulo, siga esta ordem:

1. 📌 Leia a **Visão Geral** — entenda o modelo financeiro da plataforma
2. 🗂️ Consulte o **Guia por Perfil** — e acesse a sub-página certa para o seu papel
3. 💳 Revise as **Formas de Pagamento** e suas regras
4. 💰 Leia o **Modelo de Repasse** ao profissional
5. 🔁 Verifique as regras de **Cancelamento e Reembolso**
6. 🚨 Confira os **Edge Cases** antes de implementar qualquer fluxo financeiro

---

## 🎯 Visão Geral

Todo pagamento ocorre **dentro da plataforma** — sem redirecionamentos para sites externos. A plataforma atua como intermediária financeira entre paciente e profissional, retendo uma taxa de serviço e repassando o restante ao profissional após a confirmação da consulta.

O módulo foi desenhado com foco em:

- **Confiabilidade** — garantia de entrega via Outbox Pattern, sem perda de eventos financeiros
- **Idempotência** — webhooks do gateway processados de forma segura, sem cobranças duplicadas
- **Rastreabilidade** — todo evento financeiro é registrado e auditável
- **Split automático** — divisão entre plataforma e profissional processada pelo gateway, sem operação manual

> ⚠️ O repasse ao profissional só ocorre após o agendamento atingir o estado `REALIZADO` — nunca antes.

---

## 🗂️ Guia por Perfil

| Perfil | Comece por |
|--------|-----------|
| **Product Manager / Stakeholder** | 📦 Sub-página Produto & Negócio — jornada, formas de pagamento, split, reembolso e critérios de aceite |
| **QA / Tester** | 📦 Sub-página Produto & Negócio (critérios de aceite) + Edge Cases desta página |
| **Engenheiro Backend / Tech Lead** | ⚙️ Sub-página Especificação Técnica — schema, Outbox Pattern, webhook idempotente, Kafka events e endpoints |
| **Engenheiro Frontend** | 📦 Produto & Negócio (jornada e estados) → ⚙️ Especificação Técnica (contratos de API) |
| **Financeiro / Operações** | 📦 Produto & Negócio (modelo de repasse e extrato) + seção de Disputas desta página |

---

## 📦 Acesso Rápido às Sub-páginas

| Sub-página | Conteúdo |
|-----------|---------|
| 📦 [Produto & Negócio — Pagamento](./M5-Produto-Negocio-Pagamento.md) | Jornada, formas de pagamento, split, reembolso e critérios de aceite |
| ⚙️ Especificação Técnica — Pagamento | Schema, Outbox Pattern, webhook idempotente, Kafka events e endpoints |

---

## 6.1 Formas de Pagamento

| Forma de pagamento | Confirmação | Parcelamento | Observações |
|-------------------|------------|-------------|------------|
| **Pix** | Instantânea | Não | Recomendado para pagamentos imediatos; confirmação via webhook do gateway |
| **Cartão de crédito** | Instantânea | Sim — configurável | Integração via gateway; parcelamento máximo definido pelo admin |
| **Cartão de débito** | Instantânea | Não | Pagamento à vista no ato do agendamento |
| **Boleto bancário** | 1–3 dias úteis | Não | Recomendado apenas para consultas com antecedência; slot não bloqueado até compensação |
| **Plano de saúde / convênio** | A definir | Não | Profissional informa quais convênios aceita; validação por API é planejada para versão futura |

### Regras de negócio — Formas de pagamento

| Regra | Descrição |
|-------|----------|
| RN-PAG-001 | O agendamento só é confirmado após a compensação do pagamento — sem exceção por forma de pagamento |
| RN-PAG-002 | Boleto bancário não bloqueia o slot durante o prazo de compensação — o slot só é reservado após a confirmação |
| RN-PAG-003 | Se o boleto não for compensado dentro do prazo, o agendamento é cancelado automaticamente com estado `EXPIRADO` |
| RN-PAG-004 | Parcelamento no cartão de crédito é configurável pelo admin — o profissional recebe o valor integral no repasse, independente do parcelamento do paciente |
| RN-PAG-005 | Plano de saúde e convênio não passam pelo gateway de pagamento — o fluxo financeiro é tratado fora da plataforma até a integração por API estar disponível |
| RN-PAG-006 | Nenhuma forma de pagamento pode armazenar dados de cartão na plataforma — tokenização obrigatória via gateway (conformidade PCI-DSS) |

---

## 6.2 Modelo de Repasse ao Profissional

### Como funciona o split

```
Valor pago pelo paciente
        ↓
Gateway processa o pagamento
        ↓
Split automático:
  ├── Taxa da plataforma (ex: 10–15%) → retida pela plataforma
  └── Valor líquido → reservado para repasse ao profissional
        ↓
Agendamento atinge estado REALIZADO
        ↓
Repasse liberado no prazo configurado (ex: D+2 ou D+7)
        ↓
Profissional visualiza no extrato
```

### Parâmetros do repasse

| Parâmetro | Valor padrão | Quem configura |
|-----------|-------------|---------------|
| Taxa de serviço da plataforma | 10–15% | Admin (M9) |
| Prazo de repasse após `REALIZADO` | D+2 a D+7 | Admin (M9) |
| Valor mínimo para repasse | A definir | Admin (M9) |
| Acúmulo de repasses abaixo do mínimo | Aguarda próximo ciclo | Sistema |

### Regras de negócio — Repasse

| Regra | Descrição |
|-------|----------|
| RN-PAG-007 | O repasse ao profissional só é iniciado após o agendamento atingir o estado `REALIZADO` — nunca antes |
| RN-PAG-008 | Cancelamentos com reembolso ao paciente bloqueiam o repasse automaticamente, independente do prazo |
| RN-PAG-009 | O profissional deve ter conta bancária ou chave Pix validada no cadastro para receber repasses |
| RN-PAG-010 | Repasses não realizados por dados bancários inválidos ficam em estado `PENDENTE` por até 30 dias — após isso, são escalados para o time financeiro |
| RN-PAG-011 | O profissional acessa extrato completo de repasses no painel do próprio perfil |
| RN-PAG-012 | Notas fiscais e recibos são gerados automaticamente pela plataforma a cada repasse realizado |
| RN-PAG-013 | Alterações na taxa de serviço pelo admin se aplicam apenas a novos agendamentos — nunca retroativamente |

---

## 6.3 Cancelamento e Reembolso

### Tabela de cenários de reembolso

| Cenário | Reembolso ao paciente | Taxa cobrada | Repasse ao profissional |
|---------|----------------------|-------------|------------------------|
| Cancelamento dentro do prazo (pelo paciente) | Integral | Nenhuma | Não ocorre |
| Cancelamento fora do prazo (pelo paciente) | Parcial (desconto configurável) | Taxa de cancelamento tardio | Parcial (proporcional à taxa) |
| Cancelamento pelo profissional | Integral | Nenhuma ao paciente | Não ocorre |
| Cancelamento pelo sistema (ex: fraude) | Integral | Nenhuma | Não ocorre |
| No-show do paciente (não compareceu) | Sem reembolso | Taxa total | Repasse ao profissional |
| No-show do profissional (não compareceu) | Integral | Nenhuma | Não ocorre + alerta interno |

### Regras de negócio — Reembolso

| Regra | Descrição |
|-------|----------|
| RN-PAG-014 | Reembolsos devem ser processados automaticamente no mesmo meio de pagamento utilizado pelo paciente |
| RN-PAG-015 | O prazo de estorno no cartão de crédito segue as regras da bandeira — a plataforma deve informar o prazo estimado ao paciente |
| RN-PAG-016 | Reembolsos via Pix devem ser processados em até 1 hora após o cancelamento |
| RN-PAG-017 | A taxa de cancelamento tardio deve ser exibida claramente ao paciente no momento do agendamento — não apenas no cancelamento |
| RN-PAG-018 | No-show do paciente só é registrado pelo profissional — com janela de até 1 hora após o horário da consulta |
| RN-PAG-019 | No-show do profissional gera reembolso integral automático + alerta para o time de operações via M9 |

---

## 6.4 Disputas e Chargebacks

| Situação | Fluxo |
|---------|-------|
| **Paciente contesta cobrança no banco (chargeback)** | Gateway notifica a plataforma → time financeiro analisa logs → contesta ou aceita dentro do prazo da bandeira |
| **Paciente alega não ter recebido o serviço** | Admin verifica logs de agendamento e confirmação de realização → reembolso ou manutenção conforme evidências |
| **Profissional alega não ter recebido o repasse** | Time financeiro verifica extrato e logs do gateway → reprocessa ou escala para o banco |
| **Pagamento duplicado** | Idempotência do webhook previne duplicação — se ocorrer, estorno automático da cobrança duplicada |

### Regras de negócio — Disputas

| Regra | Descrição |
|-------|----------|
| RN-PAG-020 | Toda disputa deve ser registrada com ID único, partes envolvidas, valor e status de resolução |
| RN-PAG-021 | O prazo máximo de resposta a uma disputa aberta pelo paciente é de 5 dias úteis |
| RN-PAG-022 | Webhooks do gateway devem ser processados de forma idempotente — o mesmo evento não pode gerar dois registros financeiros |
| RN-PAG-023 | Em caso de chargeback confirmado, o repasse ao profissional é suspenso até resolução — se já repassado, o valor é descontado do próximo repasse |

---

## 🚨 Casos Especiais e Edge Cases

| Situação | Comportamento esperado |
|---------|----------------------|
| Gateway fora do ar no momento do pagamento | Exibir mensagem de erro amigável + orientar a tentar novamente; não bloquear o slot permanentemente |
| Pix expirado (QR Code venceu) | Gerar novo QR Code automaticamente; slot permanece bloqueado pelo tempo restante |
| Cartão recusado | Permitir tentativa com outro cartão ou outra forma de pagamento sem perder o slot reservado |
| Paciente paga duas vezes por engano | Idempotência garante que apenas um pagamento seja processado; o segundo é estornado automaticamente |
| Profissional muda a taxa da consulta após agendamentos confirmados | O valor original do agendamento é mantido — novos agendamentos usam o novo valor |
| Repasse falha por dados bancários inválidos | Notificar o profissional + manter valor em `PENDENTE` por até 30 dias |
| Consulta realizada mas paciente contesta que não ocorreu | Logs de agendamento, confirmação bilateral e histórico do chat são evidências para resolução |
| Split configurado pelo admin muda durante agendamentos em andamento | Novos splits seguem a nova configuração; agendamentos já confirmados mantêm a taxa original |

---

## 📊 Status do Módulo

| Item | Status |
|------|--------|
| Especificação de regras de negócio (hub) | 🟡 Em andamento |
| Sub-página Produto & Negócio | ⚪ Não iniciado |
| Sub-página Especificação Técnica | ⚪ Não iniciado |
| Escolha do gateway de pagamento | ⚪ Não iniciado |
| Implementação do Outbox Pattern | ⚪ Não iniciado |
| Implementação de webhooks idempotentes | ⚪ Não iniciado |
| Implementação do split automático | ⚪ Não iniciado |
| Extrato e painel do profissional | ⚪ Não iniciado |
| Geração automática de notas fiscais | ⚪ Não iniciado |
| Testes e QA | ⚪ Não iniciado |

---

## 🔗 Dependências com outros módulos

| Módulo | Tipo de dependência |
|--------|-------------------|
| M1 — Autenticação e Cadastro | Identidade do paciente e dados bancários/Pix do profissional validados no cadastro |
| M4 — Agendamento | Pagamento é parte obrigatória do fluxo de confirmação; repasse depende do estado `REALIZADO` |
| M7 — Notificações | Confirmação de pagamento, falha, reembolso e repasse notificam ambas as partes |
| M8 — Segurança e LGPD | Tokenização de dados financeiros, conformidade PCI-DSS e logs de auditoria |
| M9 — Painel Administrativo | Configuração de taxas, prazos de repasse e gestão de disputas |

---

## ❓ Decisões em Aberto

| Decisão | Contexto |
|---------|---------|
| Gateway de pagamento | Stripe, Pagar.me, Adyen ou outro? Qual suporta split nativo no Brasil? |
| Parcelamento máximo no cartão | Até quantas vezes? Quem absorve os juros — paciente ou plataforma? |
| Prazo padrão de repasse | D+2 ou D+7? Varia por especialidade ou é único? |
| Geração de nota fiscal | NFS-e automática via API de prefeitura ou serviço terceirizado (ex: Nota Fiscal Brasil)? |
| Convênios — fase de integração | Qual versão prevê a integração por API com operadoras? |
| Valor mínimo de repasse | Acumular repasses abaixo de R$X ou repassar independente do valor? |

---

> Hub central de Pagamento. Acesse as sub-páginas conforme seu papel.
>
> ## 📦 Acesso Rápido

| Para quem | Sub-página |
|-----------|-----------|
| **Product Manager / Stakeholder / QA** | 📦 Produto & Negócio — jornada, formas de pagamento, split, reembolso e critérios de aceite |
| **Engenheiro / Tech Lead** | ⚙️ Especificação Técnica — schema, Outbox Pattern, webhook idempotente, Kafka events e endpoints |

---

## 🧭 Contexto rápido

Todo pagamento ocorre **dentro da plataforma**. Suporta Pix, cartão de crédito/débito e boleto. Split automático entre plataforma e profissional via gateway. Repasse ao profissional ocorre apenas após `COMPLETED`. Garantia de entrega via **Outbox Pattern**.

---

## 6.1 Formas de pagamento

| Forma de pagamento | Observações |
|-------------------|------------|
| **Pix** | Confirmação instantânea; recomendado para pagamentos imediatos |
| **Cartão de crédito** | Parcelamento configurável; integração via gateway de pagamento |
| **Cartão de débito** | Pagamento à vista no ato do agendamento |
| **Plano de saúde / convênio** | Profissional informa quais convênios aceita; validação futura por API |
| **Boleto bancário** | Prazo de 1–3 dias úteis; recomendado apenas para consultas antecipadas |

---

## 6.2 Modelo de repasse ao profissional

- A plataforma retém uma taxa de serviço sobre cada consulta (ex: 10–15%)
- O restante é repassado ao profissional no prazo combinado (ex: D+2 ou D+7)
- O profissional acessa extrato de repasses no painel
- Notas fiscais e recibos são gerados automaticamente

---

## 6.3 Cancelamento e reembolso

- Cancelamento dentro do prazo: reembolso integral ao cliente
- Cancelamento fora do prazo: desconto de taxa configurável
- Cancelamento pelo profissional: reembolso integral e sem penalidade ao cliente
- Disputas são gerenciadas pelo administrador com análise de logs

  ---


> 📦 [Produto & Negócio — Pagamento](./M5-Produto-Negocio-Pagamento.md)

> ⚙️ [Especificação Técnica — Pagamento](./M5-Especificacao-Tecnica-Pagamento.md)
