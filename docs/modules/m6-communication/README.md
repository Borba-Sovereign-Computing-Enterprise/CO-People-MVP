# 💬 M6 — Comunicação

> Hub central de Comunicação. Acesse as sub-páginas conforme seu papel.
> 

## Acesso Rápido

| Para quem | Sub-página |
| --- | --- |
| Product Manager / Designer / QA | 📦 Produto & Negócio — propósito, URA, regras, casos especiais |
| Engenheiro / Tech Lead | ⚙️ Especificação Técnica — WebSocket, schemas, criptografia, Kafka |

---

## Contexto rápido

Chat **não é mensageiro aberto** — só habilitado entre partes com consulta ativa ou histórico. Toda conversa começa com URA obrigatória. Mensagens criptografadas em repouso, retidas por mínimo 5 anos (CFM + LGPD). Decisão de stack de mensageria (WS nativo vs gerenciado) em aberto.

---

> 👋 **Bem-vindo(a) ao módulo de Comunicação.** Este módulo define como pacientes e profissionais de saúde trocam mensagens dentro da plataforma — com foco em segurança, rastreabilidade e contexto clínico.
> 

---

## 👋 Por onde começar

Se você acabou de entrar neste módulo, siga esta ordem:

1. 📌 Leia a seção **Visão Geral** — entenda o propósito e os limites do chat  
2. 📋 Consulte as **Finalidades do Chat** — o que pode e o que não pode ser tratado aqui  
3. 🔁 Entenda o fluxo da **URA** — como o sistema direciona as conversas  
4. ⚙️ Revise as **Funcionalidades** e as **Regras de Negócio** detalhadas  

---

## 🎯 Visão Geral

O chat da plataforma **não é um sistema de bate-papo aberto**. Ele é estruturado para suprir necessidades específicas da relação médico-paciente, mantendo o foco na saúde e na continuidade do atendimento.

O canal de comunicação existe para **complementar a consulta**, não substituí-la. Toda interação deve ser rastreável, segura e vinculada a um contexto clínico.

---

## 🗂️ Guia por Perfil

| Perfil | O que este módulo significa para você |
| --- | --- |
| **Paciente** | Canal para tirar dúvidas pós-consulta, enviar exames e tratar assuntos administrativos com o profissional |
| **Profissional de Saúde** | Ferramenta de continuidade do cuidado — orientações, documentos e gestão de agenda via mensagem |
| **Engenheiro Backend** | Implementação do sistema de mensagens, URA, controle de arquivos e notificações |
| **Engenheiro Frontend** | Interface de chat com suporte a arquivos, status de leitura e notificações push |
| **Product Manager / Designer** | Fluxos de URA, limites de uso, experiência do usuário e regras de moderação |

---

## 7.1 Finalidades do Chat

O chat só deve ser utilizado para as seguintes finalidades:

| Finalidade | Quem inicia | Observação |
| --- | --- | --- |
| Reagendamento e confirmação de consultas | Paciente ou Profissional | Deve refletir no módulo M4 — Agendamento |
| Envio de resultados de exames, imagens e documentos | Paciente | Formatos permitidos: PDF, JPEG, PNG |
| Feedback pós-consulta pelo profissional | Profissional | Só permitido após consulta realizada |
| Orientações rápidas pós-atendimento | Profissional | Não substitui consulta médica — deve haver aviso explícito na interface |
| Solicitação de segunda via de receita ou laudo | Paciente | Sujeito a avaliação do profissional |

### ❌ O que NÃO é permitido via chat

- Diagnósticos médicos por mensagem  
- Prescrição de medicamentos via chat (apenas confirmação de receitas já emitidas)  
- Atendimentos de urgência ou emergência — deve haver redirecionamento para SAMU/UPA  
- Compartilhamento de dados de terceiros sem consentimento  
- Uso do canal para fins comerciais ou publicitários  

---

## 7.2 URA — Triagem Automática Inicial

Ao iniciar uma conversa, o sistema apresenta um **menu estruturado (URA textual)** para direcionar a necessidade do usuário antes de qualquer troca de mensagem livre.

### Regras da URA

- A URA é **obrigatória** na abertura de toda nova conversa  
- O usuário pode selecionar apenas **uma opção por conversa iniciada**  
- A opção selecionada fica **registrada no histórico** e visível para o profissional  
- A opção "Outros assuntos" exige que o usuário escreva uma **descrição mínima de 20 caracteres**  
- Conversas iniciadas via URA com opção de agendamento devem ser **integradas ao M4**  

### Fluxo da URA — Paciente
```
Agendar nova consulta

Reagendar consulta existente

Cancelar consulta

Enviar resultado de exame

Solicitar receita / laudo

Outros assuntos
```
### Fluxo da URA — Profissional
```
Responder mensagem pendente

Enviar orientação pós-consulta

Solicitar exame ou retorno

Informar cancelamento de horário

Outros assuntos

```

### Regras de negócio — URA

| Regra | Descrição |
| --- | --- |
| RN-COM-001 | Toda nova conversa obrigatoriamente passa pela URA antes de habilitar campo de texto livre |
| RN-COM-002 | A opção escolhida na URA é salva como metadado da conversa |
| RN-COM-003 | Opções de agendamento (1, 2, 3 para paciente) disparam integração com M4 |
| RN-COM-004 | "Outros assuntos" requer descrição mínima de 20 caracteres |
| RN-COM-005 | Profissional não pode iniciar conversa com paciente sem consulta prévia ou agendamento ativo |

---

## 7.3 Funcionalidades do Chat

### Recursos disponíveis

| Funcionalidade | Detalhe |
| --- | --- |
| Envio de arquivos | PDF, JPEG, PNG — máx. 10MB por envio |
| Carimbo de data e hora | Exibido em todas as mensagens, fuso horário local |
| Status de leitura | Indicadores: Enviado / Entregue / Lido |
| Histórico de conversas | Acessível por ambas as partes durante toda a relação ativa |
| Notificação push | Disparada ao receber nova mensagem (requer permissão do dispositivo) |
| Restrição de tamanho | Máx. 10MB por arquivo enviado |

### Regras de negócio — Funcionalidades

| Regra | Descrição |
| --- | --- |
| RN-COM-006 | Arquivos com extensões fora de PDF, JPEG e PNG são rejeitados com mensagem de erro clara |
| RN-COM-007 | Arquivos acima de 10MB são bloqueados antes do envio, com feedback imediato ao usuário |
| RN-COM-008 | O status "Lido" só é marcado quando o destinatário abre a conversa e visualiza a mensagem |
| RN-COM-009 | Histórico de conversa é preservado por no mínimo 5 anos (conformidade com CFM e LGPD) |
| RN-COM-010 | Notificações push só são enviadas se o usuário concedeu permissão nas configurações do dispositivo |
| RN-COM-011 | Mensagens não podem ser editadas após o envio — apenas excluídas pelo remetente em até 5 minutos |
| RN-COM-012 | Exclusão de mensagem oculta o conteúdo mas mantém o registro no log interno (auditoria) |

---

## 🔒 Regras de Segurança e Privacidade

| Regra | Descrição |
| --- | --- |
| RN-COM-013 | Todas as mensagens devem ser armazenadas com criptografia em repouso |
| RN-COM-014 | A transmissão de mensagens deve ocorrer via canal seguro (TLS 1.2+) |
| RN-COM-015 | Nenhuma mensagem ou arquivo pode ser acessado por terceiros sem consentimento explícito das duas partes |
| RN-COM-016 | Em caso de denúncia, o conteúdo da conversa pode ser acessado pelo time de moderação mediante registro formal |
| RN-COM-017 | Dados de conversas não podem ser utilizados para fins de marketing ou treinamento de modelos sem consentimento |

---

## 🚨 Casos Especiais e Edge Cases

| Situação | Comportamento esperado |
| --- | --- |
| Paciente envia mensagem fora do horário de atendimento | Sistema exibe aviso de horário e informa prazo esperado de resposta |
| Profissional não responde em até 48h | Sistema envia alerta ao profissional e notifica o paciente sobre o atraso |
| Arquivo enviado contém vírus ou conteúdo malicioso | Arquivo é bloqueado automaticamente; usuário recebe erro genérico sem detalhes técnicos |
| Consulta cancelada enquanto há conversa ativa | Conversa é arquivada automaticamente com registro do motivo |
| Usuário tenta iniciar chat sem consulta prévia ou agendamento | Sistema bloqueia e exibe mensagem orientando a agendar primeiro |
| Paciente menor de idade | Chat só é habilitado com responsável cadastrado vinculado ao perfil |

---

## 📊 Status do Módulo

| Item | Status |
| --- | --- |
| Especificação de regras de negócio | 🟡 Em andamento |
| Definição de stack de mensageria | ⚪ Não iniciado |
| Design das telas de chat | ⚪ Não iniciado |
| Implementação backend | ⚪ Não iniciado |
| Implementação frontend | ⚪ Não iniciado |
| Testes e QA | ⚪ Não iniciado |

---

## Dependências com outros módulos

| Módulo | Tipo de dependência |
| --- | --- |
| M1 — Autenticação e Cadastro | Identidade do usuário para vincular conversas |
| M4 — Agendamento | Integração das opções de URA com ações de agenda |
| M7 — Notificações | Disparo de push ao receber mensagens |
| M8 — Segurança e LGPD | Criptografia, retenção e auditoria de mensagens |

---

## ❓ Decisões em Aberto

| Decisão | Contexto |
| --- | --- |
| Stack de mensageria | WebSocket nativo, Socket.io ou serviço gerenciado (ex: Twilio, Stream)? |
| Canal de notificação | Firebase FCM, OneSignal ou outro? |
| Política de retenção de arquivos | Armazenamento permanente ou expiração após X dias? |
| Moderação de conteúdo | Manual, automática (IA) ou híbrida? |

- Finalidades do chat  
- URA — triagem automática  
- Funcionalidades do chat  

O chat da plataforma não é um sistema de bate-papo aberto. Ele é estruturado para suprir necessidades específicas da relação médico-paciente, mantendo o foco na saúde e na continuidade do atendimento.

---

## 7.1 Finalidades do chat

- Reagendamento e confirmação de consultas  
- Envio de resultados de exames, imagens e documentos  
- Feedback pós-consulta pelo profissional  
- Orientações rápidas pós-atendimento (não substitui consulta médica)  
- Solicitação de segunda via de receita ou laudo  

---

## 7.2 URA — Triagem automática inicial

Ao iniciar uma conversa, o sistema apresenta um menu de opções estruturado (URA — Unidade de Resposta Audível textual) para direcionar a necessidade do usuário:

**Fluxo da URA — Cliente**

- 1. Agendar nova consulta  
    1. Agendar nova consulta  

- 2. Reagendar consulta existente  
    1. Reagendar consulta existente  

- 3. Cancelar consulta  
    1. Cancelar consulta  

- 4. Enviar resultado de exame  
    1. Enviar resultado de exame  

- 5. Solicitar receita / laudo  
    1. Solicitar receita / laudo  

- 6. Outros assuntos  
    1. Outros assuntos  

**Fluxo da URA — Profissional**

- 1. Responder mensagem pendente  
    1. Responder mensagem pendente  

- 2. Enviar orientação pós-consulta  
    1. Enviar orientação pós-consulta  

- 3. Solicitar exame ou retorno  
    1. Solicitar exame ou retorno  

- 4. Informar cancelamento de horário  
    1. Informar cancelamento de horário  

- 5. Outros assuntos  
    1. Outros assuntos  

---

## 7.3 Funcionalidades do chat

- Envio de arquivos: PDF, JPEG, PNG (laudos, receitas, exames)  
- Mensagens com carimbo de data e hora  
- Status de leitura (entregue / lido)  
- Histórico de conversas acessível por ambas as partes  
- Notificação push ao receber nova mensagem  
- Restrição de tamanho de arquivo (ex: máx. 10MB por envio)  

