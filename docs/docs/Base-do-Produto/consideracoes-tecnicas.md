# 🛠️ Considerações Técnicas
> Hub central de Considerações Técnicas. Acesse as sub-páginas conforme seu papel.
> 

## Acesso Rápido

| Para quem | Sub-página |
| --- | --- |
| Product / Stakeholder | 📦 Visão Executiva — por quê essas escolhas, integrações e limites do MVP |
| Engenheiro / Tech Lead | ⚙️ Especificação Técnica — integrações, segurança, observabilidade e infra detalhados |

> ℹ️ Para stack completa, ADRs e bounded contexts, consulte a **⚙️ Especificação Técnica — Visão Geral** (sub-page de Visão Geral do Produto).
> 

---

## Integrações Externas — MVP

| Integração | Finalidade | Status |
| --- | --- | --- |
| Gateway de pagamento (Stripe / [Pagar.me](http://Pagar.me)) | Processar pagamentos e repasses | ✅ MVP |
| E-mail transacional (SendGrid / AWS SES) | E-mails transacionais | ✅ MVP |
| SMS/2FA (Twilio / Zenvia) | OTP e lembretes SMS | ✅ MVP |
| AWS S3 | Laudos, documentos, fotos | ✅ MVP |
| API validação CRM/CRO | Verificação de registro profissional | ⚠️ Manual no MVP |
| Google Calendar / Apple Calendar | Sincronização de agenda | ❌ Fora do MVP |

---

## Antes existia aqui: 12.3 Integrações externas recomendadas

| **Integração** | **Finalidade** |
| --- | --- |
| Gateway de pagamento (ex: Stripe, PagSeguro, Pagar.me) | Processar pagamentos e repasses |
| Serviço de e-mail (ex: SendGrid, AWS SES) | Envio de e-mails transacionais |
| Serviço de SMS (ex: Twilio, Zenvia) | Envio de SMS e 2FA |
| Google Calendar / Apple Calendar | Sincronização de agenda (futuro) |
| API de validação de CRM/CRO | Validação automática de registros profissionais |

[📦 Visão Executiva — Considerações Técnicas](./Visao-Executiva-Consideracoes-Tecnicas.md)

[⚙️ Especificação Técnica — Considerações Técnicas](./Especificacao-Tecnica-Consideracoes-Tecnicas.md)
