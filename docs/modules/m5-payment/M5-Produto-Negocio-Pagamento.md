# 📦 Produto & Negócio — Pagamento

> **Audiência:** Product Manager, Stakeholders, Designer, QA

> **Objetivo:** Descrever como funciona o pagamento para o paciente, o modelo de repasse ao profissional e a política de reembolso.

---

## 1. Premissa do Módulo

Todo pagamento ocorre **dentro da plataforma** — sem exceção. O paciente nunca paga diretamente ao profissional fora do sistema. Isso garante:

- Rastreabilidade total de transações
- Proteção ao consumidor em caso de cancelamento
- Receita da plataforma via taxa de serviço
- Conformidade com obrigações fiscais

---

## 2. Jornada de Pagamento do Paciente

```
Passo 1: Paciente confirma o agendamento (M4)
  → Redirecionado para tela de pagamento

Passo 2: Escolhe forma de pagamento
  → Pix, Cartão de crédito, Cartão de débito

Passo 3: Realiza o pagamento
  → Pix: QR code ou cópia e cola → confirmação instantânea
  → Cartão: dados inseridos ou salvo anteriormente → aprovação do gateway

Passo 4: Confirmação
  → Pagamento aprovado → agendamento vai para CONFIRMED
  → Recibo enviado por email
  → Pagamento falhou → pode tentar novamente ou usar outra forma

Resultado: Agendamento confirmado, valor retido até realização da consulta
```

---

## 3. Formas de Pagamento — MVP

| Forma | Confirmação | Observações |
|-------|------------|------------|
| **Pix** | Instantânea | Método preferencial; sem taxa adicional ao paciente |
| **Cartão de crédito** | Instantânea | Parcelamento configurável (1x a 12x); taxa de parcelamento pode ser repassada |
| **Cartão de débito** | Instantânea | À vista |
| **Boleto** | 1–3 dias úteis | Somente para agendamentos com > 3 dias de antecedência |

> **Plano de saúde / convênio:** fora do MVP. O profissional informa quais aceita no perfil, mas a validação e cobrança pelo plano é feita diretamente entre as partes no atendimento.

---

## 4. Modelo de Split — Como o Dinheiro é Dividido

```
Paciente paga: R$ 200,00
    │
    ▼
[Gateway de Pagamento]
    │
    ├── Taxa da plataforma (ex: 12%) ─► R$ 24,00 → Conta da Plataforma
    │
    └── Repasse ao profissional (88%) ─► R$ 176,00 → Conta do Profissional
                                            (após consulta COMPLETED)
```

> A taxa de serviço é configurável pelo admin. O valor exibido no perfil e no checkout é sempre o valor total que o paciente paga — a divisão é transparente para o profissional mas **não exibida** ao paciente.

---

## 5. Quando o Profissional Recebe

| Evento | Quando repassa |
|--------|---------------|
| Consulta marcada como `COMPLETED` | D+2 úteis (default; configurável) |
| Cancelamento pelo paciente fora do prazo | Taxa de cancel é repassada após período de disputa (D+7) |
| Cancelamento pelo profissional | Sem repasse; reembolso integral ao paciente |
| Dispute/chargeback aberto | Repasse suspenso até resolução |

---

## 6. Política de Reembolso

| Situação | Reembolso ao Paciente | Prazo |
|---------|----------------------|-------|
| Cancelamento pelo paciente **dentro do prazo** | 100% do valor pago | Até 5 dias úteis |
| Cancelamento pelo paciente **fora do prazo** | Valor pago - taxa de cancelamento | Até 5 dias úteis |
| Cancelamento pelo **profissional** | 100% do valor pago | Até 2 dias úteis |
| Pagamento com erro após débito | 100% automático | Instantâneo |
| Disputa (chargeback) | Analisado pelo admin | Variável |

---

## 7. Extrato e Histórico Financeiro

**Para o paciente:**

- Histórico de pagamentos realizados com recibo PDF
- Status de cada transação (aprovado, reembolsado, pendente)

**Para o profissional:**

- Extrato de repasses (valor bruto, taxa da plataforma, valor líquido)
- Data prevista de repasse por consulta
- Histórico de reembolsos que afetaram o repasse

---

## 8. Disputas e Problemas

- Disputas são gerenciadas pelo **administrador** via painel
- O admin tem acesso ao log completo do agendamento + pagamento
- Em caso de chargeback, o gateway notifica a plataforma via webhook
- A plataforma suspende o repasse ao profissional durante a análise

---

## 9. Critérios de Aceite (MVP)

- [ ] Paciente consegue pagar via Pix e Cartão em menos de 2 minutos
- [ ] Pagamento aprovado muda status do agendamento para `CONFIRMED` em até 10 segundos
- [ ] Timeout de pagamento de 30 minutos — após isso, slot é liberado
- [ ] Reembolso por cancelamento no prazo é iniciado automaticamente, sem ação manual
- [ ] Recibo de pagamento enviado por e-mail em até 5 minutos
- [ ] Profissional visualiza extrato de repasses no painel
- [ ] Webhook do gateway é processado de forma idempotente (sem duplicação)
- [ ] Falha no pagamento não confirma agendamento

---

## 10. Fluxograma (LucidChart)

> 🔗 **[Inserir link do LucidChart — Fluxo de Pagamento e Repasse]**

> Cobrir: checkout • confirmação via webhook • split • repasse após COMPLETED • reembolso em cancelamento • disputa

---

## 📂 Navegação

| | |
|--|--|
| ← Hub | [README — M5 Pagamento](./README.md) |
| → Técnico | [Especificação Técnica — Pagamento](./02-M5-Especificacao-Tecnica-Pagamento.md) |
