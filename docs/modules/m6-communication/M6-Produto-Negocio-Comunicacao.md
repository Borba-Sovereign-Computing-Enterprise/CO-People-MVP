
# 📦 Produto & Negócio — Comunicação

> **Audiência:** Product Manager, Designer, Stakeholders, QA

> **Objetivo:** Definir o propósito, limites, fluxos e regras do canal de comunicação entre pacientes e profissionais.

---

## 1. O que é e o que não é

O chat da plataforma **não é um mensageiro aberto**. Ele existe para complementar a relação de cuidado, com foco em continuidade clínica e resolução de assuntos administrativos vinculados a consultas.

> **Regra central:** Chat só é habilitado entre paciente e profissional que têm ou tiveram uma consulta ativa juntos. Sem histórico de consulta = sem chat.

---

## 2. O que pode ser tratado via chat

| Finalidade | Quem inicia | Observação |
|-----------|------------|------------|
| Reagendamento ou confirmação de consulta | Paciente ou Profissional | Deve refletir no M4 — Agendamento |
| Envio de exames, imagens e documentos | Paciente | PDF, JPEG, PNG — máx. 10MB |
| Feedback pós-consulta | Profissional | Só após consulta `COMPLETED` |
| Orientações rápidas pós-atendimento | Profissional | Com aviso explícito: não substitui consulta |
| Solicitação de segunda via de receita/laudo | Paciente | Sujeito à avaliação do profissional |

### O que NÃO é permitido

- Diagnósticos médicos por mensagem
- Prescrição de medicamentos
- Atendimentos de urgência/emergência — redirecionar para SAMU/UPA
- Fins comerciais ou publicitários
- Compartilhamento de dados de terceiros sem consentimento

---

## 3. Fluxo URA — Triagem Automática

Toda nova conversa começa obrigatoriamente com a URA antes de liberar texto livre.

**Paciente:**

```
1. Agendar nova consulta      → integra com M4
2. Reagendar consulta         → integra com M4
3. Cancelar consulta          → integra com M4
4. Enviar resultado de exame
5. Solicitar receita / laudo
6. Outros assuntos            → requer descrição mín. 20 caracteres
```

**Profissional:**

```
1. Responder mensagem pendente
2. Enviar orientação pós-consulta
3. Solicitar exame ou retorno
4. Informar cancelamento de horário  → integra com M4
5. Outros assuntos
```

---

## 4. Funcionalidades e Limites

| Funcionalidade | Detalhe |
|---------------|--------|
| Envio de arquivos | PDF, JPEG, PNG — máx. 10MB por envio |
| Timestamp | Exibido em todas as mensagens, fuso horário local |
| Status de leitura | Enviado → Entregue → Lido |
| Histórico | Acessível por ambas as partes durante toda a relação ativa |
| Notificação push | Ao receber nova mensagem (requer permissão) |
| Edição de mensagem | **Não permitida** após envio |
| Exclusão de mensagem | Pelo remetente em até 5 minutos — conteúdo oculto, log mantido |

---

## 5. Regras de Negócio Consolidadas

| Código | Regra |
|--------|-------|
| RN-COM-001 | Toda nova conversa passa pela URA antes do texto livre |
| RN-COM-002 | Opção da URA é salva como metadado da conversa |
| RN-COM-003 | Opções de agendamento (1, 2, 3 do paciente) disparam integração com M4 |
| RN-COM-004 | "Outros assuntos" exige descrição mínima de 20 caracteres |
| RN-COM-005 | Profissional não pode iniciar conversa sem consulta prévia ou agendamento ativo |
| RN-COM-006 | Arquivos fora de PDF/JPEG/PNG são rejeitados com erro claro |
| RN-COM-007 | Arquivos acima de 10MB são bloqueados antes do envio |
| RN-COM-008 | Status "Lido" só é marcado ao abrir e visualizar a mensagem |
| RN-COM-009 | Histórico retido por mínimo 5 anos (CFM + LGPD) |
| RN-COM-010 | Push só enviado com permissão concedida pelo usuário |
| RN-COM-011 | Mensagem não pode ser editada após envio |
| RN-COM-012 | Exclusão oculta conteúdo mas mantém registro no log interno |
| RN-COM-013 | Mensagens armazenadas com criptografia em repouso |
| RN-COM-014 | Transmissão via TLS 1.3+ obrigatório |
| RN-COM-015 | Nenhuma mensagem acessível por terceiros sem consentimento explícito |

---

## 6. Casos Especiais

| Situação | Comportamento |
|---------|--------------|
| Mensagem fora do horário de atendimento | Aviso de horário + prazo esperado de resposta |
| Profissional não responde em 48h | Alerta ao profissional + notificação ao paciente |
| Arquivo com vírus/conteúdo malicioso | Bloqueio automático + erro genérico sem detalhes técnicos |
| Consulta cancelada com conversa ativa | Conversa arquivada automaticamente com registro do motivo |
| Paciente menor de idade | Chat só habilitado com responsável cadastrado vinculado |

---

## 7. Decisões em Aberto

| Decisão | Contexto |
|---------|---------|
| Stack de mensageria | WebSocket nativo, Socket.io ou serviço gerenciado (Twilio, Stream)? |
| Canal de push | Firebase FCM ou OneSignal? |
| Retenção de arquivos | Permanente ou expiração após X dias? |
| Moderação de conteúdo | Manual, automática (IA) ou híbrida? |

---

## 8. Critérios de Aceite (MVP)

- [ ] Chat só abre entre partes com consulta ativa ou histórico
- [ ] URA é obrigatória em toda nova conversa
- [ ] Arquivo inválido ou acima de 10MB é bloqueado com feedback imediato
- [ ] Mensagem deletada oculta o conteúdo mas mantém entrada no log
- [ ] Histórico acessível por ambas as partes durante a relação ativa
- [ ] Integração com M4: opção 1/2/3 da URA abre fluxo de agendamento

---

## 9. Fluxograma (LucidChart)

> 🔗 **[Inserir link — Fluxo de Comunicação]**

> Cobrir: abertura da conversa → URA → texto livre → envio de arquivo → integração M4

---

## 📂 Navegação

| | |
|--|--|
| ← Hub | [README — M6 Comunicação](./README.md) |
| → Técnico | [Especificação Técnica — Comunicação](./M6-Especificacao-Tecnica-Comunicacao.md) |
