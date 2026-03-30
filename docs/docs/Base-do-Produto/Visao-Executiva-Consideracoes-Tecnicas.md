# 📦 Visão Executiva — Considerações Técnicas

> **Audiência:** Product Manager, Stakeholders, Designers que precisam entender as escolhas técnicas sem entrar no código
> 

> **Objetivo:** Explicar *por quê* fazemos as escolhas técnicas que fazemos, quais integramos e quais são os limites do que a tecnologia nos permite no MVP.
> 

---

## 1. Filosofia de Engenharia — em linguagem simples

### Tudo como dados (Data-Driven)

A equipe de engenharia optou por descrever o comportamento do sistema como **dados**, não como código "mágico". Isso significa que rotas, configurações e regras de validação vivem em arquivos de configuração legíveis — qualquer membro do time consegue ler e entender o que o sistema faz sem precisar ser especialista em programação.

### Histórico imutável de eventos (Event Sourcing)

O backend não sobrescreve dados — ele **registra eventos**. Quando uma consulta é cancelada, o sistema não apaga o agendamento: ele registra o evento “cancelamento” com data, hora, motivo e quem cancelou. Isso dá rastreabilidade total — algo fundamental para LGPD, disputas financeiras e auditoria.

### Falhas explícitas

O código nunca “engole” erros silenciosamente. Todo erro é tratado explicitamente: ou a operação dá certo, ou retorna um erro claro com causa. Isso melhora a qualidade do produto e facilita a identificação de problemas.

---

## 2. Frontend — Microfrontends

O frontend não é uma única aplicação monolítica. Ele é dividido em **mini-aplicações independentes por domínio** (ex: app de autenticação, app de agendamento, app do profissional). Isso permite que times diferentes trabalhem em partes diferentes sem interferir uns nos outros, e que cada parte seja atualizada de forma independente.

---

## 3. Por que Clojure?

- É uma linguagem funcional que **favorece imutabilidade** — dados não mudam inesperadamente
- Rodar na JVM garante performance e maturidade de ecossistema
- Excelente para sistemas que lidam com eventos e transformações de dados (como o nosso)
- Adotado por empresas como Nubank, Walmart Labs, Cisco e Netflix

---

## 4. Integrações Externas do MVP

| Integração | Para que serve | Status no MVP |
| --- | --- | --- |
| **Gateway de Pagamento** (Stripe / [Pagar.me](http://Pagar.me) / PagSeguro) | Processar cobranças, split e repasses ao profissional | ✅ MVP |
| **E-mail transacional** (SendGrid / AWS SES) | Confirmações, lembretes, recuperação de senha | ✅ MVP |
| **SMS / 2FA** (Twilio / Zenvia) | Códigos de verificação em duas etapas | ✅ MVP |
| **Storage de arquivos** (AWS S3) | Laudos, documentos de habilitação, fotos de perfil | ✅ MVP |
| **Validação de CRM/CRO** | Verificar automaticamente registro profissional | ⚠️ Manual no MVP, automatação em v2 |
| **Google Calendar / Apple Calendar** | Sincronizar agenda do profissional | ❌ Fora do MVP |
| **Teleconsulta (vídeo)** | Atendimento remoto | ❌ Fora do MVP |
| **Planos de saúde / convênios** | Validação de cobertura em tempo real | ❌ Fora do MVP |

---

## 5. O que o MVP Não faz (por decisão técnica deliberada)

- **Sem app mobile nativo** — web responsiva atende o MVP com menor custo de entrega
- **Sem validação automática de CRM/CRO** — APIs dos conselhos são instáveis; admin valida manualmente
- **Sem teleconsulta** — complexidade de vídeo, LGPD em transmissão e CFM ficam para v2

---

## 6. Observabilidade — o que monitoramos

- **Desempenho:** toda requisição é medida; se algo ficar lento, sabemos antes do usuário reclamar
- **Erros:** Sentry captura todas as exceções em tempo real com contexto de onde e por que ocorreram
- **Alertas automáticos:** sistema envia alertas se taxa de erro ultrapassar 1% ou se pagamentos falharem
- **Histórico de auditoria:** toda ação sobre dados de saúde fica registrada de forma imutável

---

## 7. Segurança — o que protegemos e como

| O quê | Como é protegido |
| --- | --- |
| Senhas | Nunca armazenadas em texto puro — apenas hash irreversível |
| CPF e dados bancários | Criptografados no banco de dados |
| Laudos e documentos médicos | Acesso via link temporário único; expiram em 1 hora |
| Comunicação | TLS 1.3 obrigatório em toda a plataforma |
| Credenciais de sistema | Nunca no código — armazenadas em cofre seguro (AWS Secrets Manager) |
