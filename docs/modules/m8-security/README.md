# 🔒 M8 — Segurança e LGPD

> Hub central de Segurança e LGPD. Acesse as sub-páginas conforme seu papel.
> 

## Acesso Rápido

| Para quem | Sub-página |
| --- | --- |
| Product Manager / Jurídico / Stakeholder | 📦 Produto & Negócio — dados tratados, direitos, obrigações LGPD, plano de incidentes |
| Engenheiro / Tech Lead | ⚙️ Especificação Técnica — RBAC, criptografia por campo, consentimento, anonimização |

---

## Contexto rápido

Dados de saúde são **categoria especial pela LGPD (Art. 11)**. Segurança é camada transversal de toda a plataforma. RBAC data-driven via EDN. Criptografia AES-256-GCM por campo sensível. Consentimento granular por finalidade. Notificação ANPD em até 72h em caso de incidente.

---

> 👋 **Bem-vindo(a) ao módulo de Segurança e LGPD.** Por tratar dados sensíveis de saúde, a plataforma deve seguir rigorosamente a Lei Geral de Proteção de Dados (LGPD — Lei 13.709/2018) e as melhores práticas de segurança da informação. Este módulo define os requisitos, limites e responsabilidades que atravessam todos os outros módulos.
> 

---

## 👋 Por onde começar

Se você acabou de entrar neste módulo, siga esta ordem:

1. 📌 Leia a **Visão Geral** — entenda por que segurança é transversal a toda a plataforma
2. 🗂️ Consulte os **Dados Sensíveis Tratados** — saiba o que está em jogo
3. 🛡️ Revise as **Medidas de Segurança** técnicas e operacionais
4. ⚖️ Leia a seção de **Conformidade LGPD** — direitos, obrigações e prazos
5. 🚨 Verifique o **Plano de Resposta a Incidentes**

---

## 🎯 Visão Geral

Dados de saúde são classificados pela LGPD como **dados sensíveis** (Art. 11), exigindo nível de proteção superior ao de dados comuns. Qualquer vazamento ou uso indevido pode gerar danos irreparáveis aos titulares e responsabilidade civil, administrativa e criminal para a plataforma.

Segurança não é uma funcionalidade isolada — é uma **camada transversal** que deve ser considerada em todo novo módulo, endpoint, fluxo de dados e decisão arquitetural.

---

## 🗂️ Guia por Perfil

| Perfil | O que este módulo significa para você |
| --- | --- |
| **Paciente** | Seus dados clínicos, financeiros e pessoais são protegidos por lei e por escolhas técnicas da plataforma |
| **Profissional de Saúde** | Seus dados bancários e de cadastro são protegidos; você também é responsável pelo sigilo das informações dos seus pacientes |
| **Engenheiro Backend** | Implementação de criptografia, autenticação, RBAC, logs de auditoria e conformidade técnica com a LGPD |
| **Engenheiro Frontend** | Garantir que dados sensíveis não sejam expostos no cliente, sem logs de console em produção, sem armazenamento indevido no browser |
| **Product Manager / Designer** | Consentimento explícito nos fluxos, política de privacidade acessível, direitos do titular mapeados nas telas |

---

## 9.1 Dados Sensíveis Tratados

Todos os dados abaixo são classificados como **sensíveis** e exigem tratamento diferenciado conforme Art. 11 da LGPD:

| Categoria | Exemplos | Base legal (LGPD) |
| --- | --- | --- |
| Identificação pessoal | CPF, nome completo, endereço, telefone, e-mail | Execução de contrato (Art. 7º, V) |
| Dados clínicos | Laudos, prontuários, resultados de exames, histórico de consultas | Tutela da saúde (Art. 11, II, f) |
| Dados de comunicação | Mensagens trocadas no chat, arquivos enviados | Execução de contrato + Legítimo interesse |
| Dados financeiros | Dados bancários do profissional, histórico de pagamentos | Execução de contrato (Art. 7º, V) |
| Dados de acesso | Logs de login, IP, dispositivo, token de sessão | Legítimo interesse / Segurança (Art. 7º, IX) |
| Dados de menores | Qualquer dado de paciente menor de 18 anos | Proteção especial — requer autorização do responsável (Art. 14) |

### Regras de negócio — Dados

| Regra | Descrição |
| --- | --- |
| RN-SEG-001 | Dados clínicos nunca podem ser compartilhados com terceiros sem consentimento explícito do titular |
| RN-SEG-002 | Dados de menores de 18 anos só podem ser coletados e processados com autorização do responsável legal cadastrado |
| RN-SEG-003 | Dados bancários do profissional devem ser armazenados de forma tokenizada — nunca em texto puro |
| RN-SEG-004 | Logs de auditoria não devem conter dados sensíveis em texto puro — apenas identificadores e referências |
| RN-SEG-005 | Nenhum dado sensível pode ser exposto em URLs, query strings ou logs de aplicação |

---

## 9.2 Medidas de Segurança

### Criptografia

| Requisito | Especificação |
| --- | --- |
| Dados em repouso | AES-256 para dados sensíveis armazenados no banco |
| Dados em trânsito | TLS 1.3 obrigatório em todas as comunicações |
| Dados bancários | Tokenização via provedor de pagamento (nunca armazenados diretamente) |
| Backups | Criptografados com chave separada da chave de produção |

### Autenticação e Autorização

| Requisito | Especificação |
| --- | --- |
| Autenticação | JWT com expiração curta (ex: 15 min) + refresh token com rotação |
| 2FA | Disponível via SMS (obrigatório para profissionais de saúde, opcional para pacientes) |
| RBAC | Controle de acesso baseado em papéis: paciente, profissional, admin, moderador |
| Política de senha | Mín. 8 caracteres, com letras maiúsculas, minúsculas, números e símbolo |
| Bloqueio de conta | Após 5 tentativas de login fracassadas — desbloqueio via e-mail |
| Sessões simultâneas | Limite configurável de sessões ativas por usuário |

### Regras de negócio — Autenticação

| Regra | Descrição |
| --- | --- |
| RN-SEG-006 | Refresh tokens devem ser invalidados imediatamente em caso de logout ou troca de senha |
| RN-SEG-007 | Tokens JWT não devem conter dados sensíveis no payload — apenas identificadores |
| RN-SEG-008 | 2FA é obrigatório para profissionais de saúde no primeiro acesso e a cada 30 dias |
| RN-SEG-009 | Tentativas de login fracassadas devem ser registradas em log com IP e timestamp |
| RN-SEG-010 | Após desbloqueio de conta, o usuário deve redefinir a senha obrigatoriamente |
| RN-SEG-011 | Senhas não podem ser armazenadas em texto puro — uso obrigatório de bcrypt ou Argon2 |

### Auditoria e Monitoramento

| Requisito | Descrição |
| --- | --- |
| Logs de auditoria | Imutáveis para todas as ações críticas: login, acesso a dados clínicos, alterações de cadastro, exclusões |
| Retenção de logs | Mínimo 5 anos conforme orientação do CFM e LGPD |
| Alertas de anomalia | Detecção de padrões suspeitos: múltiplos logins de IPs diferentes, acesso massivo a dados, etc. |
| Acesso privilegiado | Acessos administrativos ao banco de dados devem ser logados e auditados separadamente |

### Regras de negócio — Auditoria

| Regra | Descrição |
| --- | --- |
| RN-SEG-012 | Logs de auditoria não podem ser editados ou excluídos por nenhum usuário, incluindo administradores |
| RN-SEG-013 | Qualquer acesso a dados clínicos de um paciente deve gerar um registro de auditoria vinculado ao profissional que acessou |
| RN-SEG-014 | Em caso de acesso administrativo a dados de produção, deve haver registro com justificativa formal |

---

## 9.3 Conformidade LGPD

### Direitos do Titular

| Direito (LGPD) | Como a plataforma atende | Prazo de resposta |
| --- | --- | --- |
| Confirmação de tratamento (Art. 18, I) | Seção "Meus Dados" no perfil do usuário | Imediato |
| Acesso aos dados (Art. 18, II) | Exportação de dados disponível no perfil | Até 15 dias |
| Correção de dados (Art. 18, III) | Edição disponível no perfil | Imediato |
| Anonimização ou exclusão (Art. 18, IV) | Solicitação via suporte — fluxo definido | Até 15 dias |
| Portabilidade (Art. 18, V) | Exportação em formato estruturado (JSON ou PDF) | Até 15 dias |
| Revogação de consentimento (Art. 18, IX) | Opção disponível nas configurações de privacidade | Imediato |
| Informação sobre compartilhamento (Art. 18, VII) | Política de privacidade detalhada e acessível | Imediato |

### Obrigações Operacionais

| Obrigação | Descrição |
| --- | --- |
| Termo de consentimento | Explícito e específico por finalidade, coletado no momento do cadastro |
| Política de privacidade | Acessível em todas as telas, linguagem clara e não jurídica |
| DPO designado | Encarregado de Proteção de Dados com canal de contato público |
| Relatório de Impacto (RIPD) | Elaborado antes do lançamento e revisado a cada mudança significativa no tratamento |
| Transferência internacional | Só permitida para países com nível adequado de proteção ou com cláusulas contratuais aprovadas pela ANPD |

### Regras de negócio — LGPD

| Regra | Descrição |
| --- | --- |
| RN-SEG-015 | Consentimento deve ser granular — o usuário deve poder aceitar ou recusar finalidades individualmente |
| RN-SEG-016 | Não é permitido condicionar o uso da plataforma à aceitação de finalidades desnecessárias (Art. 7º, §3º) |
| RN-SEG-017 | A exclusão de conta deve anonimizar os dados do titular — não apenas desativar o cadastro |
| RN-SEG-018 | Dados de menores não podem ser usados para fins de marketing ou análise comportamental |
| RN-SEG-019 | Em caso de compartilhamento com terceiros (ex: parceiros de pagamento), o usuário deve ser informado na política de privacidade |
| RN-SEG-020 | O aceite dos termos deve ser registrado com timestamp, versão do documento e IP do usuário |

---

## 🚨 Plano de Resposta a Incidentes

### Classificação de Incidentes

| Nível | Descrição | Exemplo |
| --- | --- | --- |
| 🔴 Crítico | Vazamento de dados sensíveis confirmado | Dump do banco exposto publicamente |
| 🟠 Alto | Acesso não autorizado a dados de usuários | Conta administrativa comprometida |
| 🟡 Médio | Tentativa de ataque detectada sem sucesso | Brute force bloqueado |
| ⚪ Baixo | Anomalia de comportamento sem impacto confirmado | Pico incomum de requisições |

### Fluxo de Resposta

| Etapa | Ação | Prazo |
| --- | --- | --- |
| 1. Detecção | Identificar e isolar o sistema afetado | Imediato |
| 2. Contenção | Revogar acessos, desativar endpoints comprometidos | Até 1 hora |
| 3. Avaliação | Mapear dados afetados e extensão do incidente | Até 24 horas |
| 4. Notificação ANPD | Comunicar à Autoridade Nacional de Proteção de Dados | Até 72 horas (Art. 48 LGPD) |
| 5. Notificação aos titulares | Informar usuários afetados com clareza sobre o ocorrido | Até 72 horas |
| 6. Remediação | Corrigir a vulnerabilidade e restaurar o serviço | Conforme severidade |
| 7. Post-mortem | Documentar causa raiz, impacto e medidas preventivas | Até 7 dias após resolução |

---

## 🚨 Casos Especiais e Edge Cases

| Situação | Comportamento esperado |
| --- | --- |
| Usuário solicita exclusão de conta com consultas futuras | Cancelar consultas, notificar profissionais, então iniciar anonimização |
| Profissional solicita exclusão com histórico clínico de pacientes | Dados clínicos dos pacientes são retidos pelo prazo legal — apenas dados do profissional são anonimizados |
| Usuário menor atinge 18 anos | Sistema deve notificar para atualização do consentimento pelo próprio titular |
| Requisição de dados por autoridade judicial | Seguir orientação jurídica — não fornecer sem ordem judicial formal |
| Token JWT interceptado | Rotação imediata de todos os tokens do usuário afetado + notificação por e-mail |
| Backup corrompido | Acionar plano de recuperação de desastres — RTO e RPO a definir |

---

## 📊 Status do Módulo

| Item | Status |
| --- | --- |
| Especificação de regras de negócio | 🟡 Em andamento |
| Elaboração do RIPD | ⚪ Não iniciado |
| Designação do DPO | ⚪ Não iniciado |
| Implementação de criptografia (repouso + trânsito) | ⚪ Não iniciado |
| Implementação de RBAC | ⚪ Não iniciado |
| Implementação de logs de auditoria imutáveis | ⚪ Não iniciado |
| Fluxo de exclusão e portabilidade de dados | ⚪ Não iniciado |
| Plano de resposta a incidentes operacional | ⚪ Não iniciado |
| Política de privacidade (versão final) | ⚪ Não iniciado |

---

## 🔗 Dependências com outros módulos

| Módulo | Tipo de dependência |
| --- | --- |
| M1 — Autenticação e Cadastro | Coleta de consentimento, política de senha, 2FA e controle de sessão |
| M2 — Perfil do Profissional | Validação de dados bancários tokenizados e dados de identidade profissional |
| M5 — Pagamento | Tokenização de dados financeiros e conformidade PCI-DSS |
| M6 — Comunicação | Criptografia de mensagens e retenção de histórico do chat |
| M7 — Notificações | Log de auditoria para disparos de SMS/e-mail com dados de contato |
| Todos os módulos | RBAC, logs de auditoria e política de acesso a dados sensíveis |

---

## ❓ Decisões em Aberto

| Decisão | Contexto |
| --- | --- |
| Provedor de gerenciamento de chaves | AWS KMS, HashiCorp Vault ou solução própria? |
| Algoritmo de hash de senhas | bcrypt ou Argon2id? |
| RTO e RPO do plano de recuperação | Quanto tempo de indisponibilidade e perda de dados são aceitáveis? |
| Ferramenta de SIEM | Datadog, Elastic Security ou outro para monitoramento de anomalias? |
| Responsável pelo DPO | Interno ou terceirizado? |
| Escopo do RIPD | Elaborar um por módulo ou um consolidado para toda a plataforma? |
- Dados sensíveis tratados
- Medidas de segurança
- Conformidade LGPD

Por tratar de dados sensíveis de saúde, a plataforma deve seguir rigorosamente a Lei Geral de Proteção de Dados (LGPD - Lei 13.709/2018) e boas práticas de segurança da informação.

## 9.1 Dados sensíveis tratados

- Dados de identificação pessoal (CPF, endereço, telefone)
- Histórico de consultas e agendamentos
- Laudos, prontuários, resultados de exames
- Informações financeiras (dados bancários do profissional)
- Mensagens trocadas no chat

## 9.2 Medidas de segurança

**Requisitos técnicos de segurança**

---

- Criptografia em repouso e em trânsito (AES-256 + TLS 1.3)

---

- Autenticação com JWT + refresh token com rotação

---

- Autenticação em dois fatores (2FA) disponível

---

- Controle de acesso baseado em papéis (RBAC)

---

- Logs de auditoria imutáveis para todas as ações críticas

---

- Política de senha forte (mín. 8 caracteres, letras, números e símbolo)

---

- Bloqueio de conta após N tentativas de login fracassadas

---

- Backup criptografado com retenção configurável

---

## 9.3 Conformidade LGPD

- Termo de consentimento explícito no cadastro
- Política de privacidade acessível na plataforma
- Direito de exclusão de dados a pedido do usuário
- Dados de menores de 18 anos só são processados com autorização do responsável
- DPO (Encarregado de Proteção de Dados) designado

[📦 Produto & Negócio — Segurança e LGPD](M8-Produto-Negocio-SegurancaeLGPD.md)

[⚙️ Especificação Técnica — Segurança e LGPD](M8-Especificacao-Tecnica-SegurancaeLGPD.md)
