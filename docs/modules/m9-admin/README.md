# ⚙️ M9 — Painel Administrativo

> Hub central do Painel Administrativo. Acesse as sub-páginas conforme seu papel.
> 

## Acesso Rápido

| Para quem | Sub-página |
| --- | --- |
| Product Manager / Operações / Stakeholder | 📦 Produto & Negócio — perfis RBAC, funcionalidades, regras e decisões em aberto |
| Engenheiro / Tech Lead | ⚙️ Especificação Técnica — schemas, optimistic locking, endpoints, Kafka events |

---

## Contexto rápido

Centro de operações internas — acessível apenas por admins autenticados. 5 perfis com RBAC data-driven. Aprovação de profissionais sempre humana. Optimistic locking para evitar dupla moderação simultânea. Toda ação gera audit log imútável.

---

> 👋 **Bem-vindo(a) ao módulo do Painel Administrativo.** Este é o centro de controle da plataforma, acessível exclusivamente pelo time interno. Aqui são gerenciados usuários, métricas, configurações de negócio e ações de moderação — tudo com rastreabilidade total.
> 

---

## 👋 Por onde começar

Se você acabou de entrar neste módulo, siga esta ordem:

1. 📌 Leia a **Visão Geral** — entenda o escopo e os limites do painel
2. 🗂️ Consulte o **Guia por Perfil Administrativo** — nem todo admin tem acesso a tudo
3. 🔧 Revise as **Funcionalidades** organizadas por área
4. ⚙️ Leia as **Regras de Negócio** de cada seção
5. 🚨 Verifique os **Edge Cases** e situações sensíveis

---

## 🎯 Visão Geral

O painel administrativo é o **centro de operações internas** da plataforma. Ele não é acessível por pacientes ou profissionais de saúde — apenas pelo time interno com perfil administrativo devidamente autenticado e autorizado.

Toda ação realizada no painel deve ser **rastreável, justificável e auditável**. O poder de alterar configurações que afetam todos os usuários exige controles rígidos de acesso e registro.

---

## 🗂️ Guia por Perfil Administrativo

Nem todo membro do time interno tem acesso às mesmas funcionalidades. O painel adota RBAC com perfis distintos:

| Perfil | Acesso |
| --- | --- |
| **Super Admin** | Acesso total — todas as funcionalidades, incluindo configurações críticas e logs |
| **Operações** | Aprovação de cadastros, gerenciamento de usuários, moderação de avaliações |
| **Financeiro** | Configuração de taxas, repasses, relatórios de receita |
| **Suporte** | Visualização de dados de usuários, histórico de agendamentos, abertura de tickets internos |
| **Analista de Dados** | Acesso somente leitura a relatórios e métricas — sem acesso a dados pessoais identificáveis |

### Regras de negócio — Perfis

| Regra | Descrição |
| --- | --- |
| RN-ADM-001 | Nenhum perfil administrativo pode ser criado sem aprovação de um Super Admin |
| RN-ADM-002 | Toda criação, alteração ou revogação de perfil administrativo deve gerar registro de auditoria |
| RN-ADM-003 | Super Admins devem usar 2FA obrigatoriamente — sem exceção |
| RN-ADM-004 | Sessões administrativas expiram em 30 minutos de inatividade |
| RN-ADM-005 | Acesso ao painel é bloqueado fora da rede corporativa ou VPN, a menos que 2FA esteja ativo |

---

## 10.1 Gestão de Cadastros de Profissionais

### Fluxo de Aprovação

`Profissional envia cadastro
        ↓
Status: Aguardando Revisão
        ↓
Admin analisa documentação (CRM, especialidade, identidade)
        ↓
    ┌───┴───┐
Aprovado   Rejeitado
    ↓          ↓
Perfil      Notificação com
ativado     motivo detalhado`

### Regras de negócio — Aprovação de Cadastros

| Regra | Descrição |
| --- | --- |
| RN-ADM-006 | Todo cadastro de profissional deve ser revisado por um humano — não há aprovação automática |
| RN-ADM-007 | A rejeição deve sempre incluir um motivo selecionado de uma lista padronizada + campo de observação livre |
| RN-ADM-008 | O profissional deve ser notificado por push + e-mail tanto na aprovação quanto na rejeição (ver M7) |
| RN-ADM-009 | Um cadastro rejeitado pode ser resubmetido pelo profissional após correção — limite de 3 tentativas |
| RN-ADM-010 | Após a 3ª rejeição, o caso deve ser escalado para revisão manual pelo Super Admin |
| RN-ADM-011 | Cadastros pendentes há mais de 72h devem gerar alerta interno para o time de operações |

### Motivos de Rejeição Padronizados

| Código | Motivo |
| --- | --- |
| REJ-01 | Documentação incompleta ou ilegível |
| REJ-02 | CRM inválido ou não encontrado |
| REJ-03 | Especialidade não compatível com o CRM informado |
| REJ-04 | Dados de identidade não conferem |
| REJ-05 | Suspeita de fraude ou duplicidade de cadastro |
| REJ-06 | Outros — descrição obrigatória |

---

## 10.2 Gerenciamento de Usuários

### Ações disponíveis por tipo de usuário

| Ação | Paciente | Profissional | Observação |
| --- | --- | --- | --- |
| Visualizar perfil completo | ✅ | ✅ | Requer justificativa para acesso a dados clínicos |
| Editar dados cadastrais | ✅ | ✅ | Apenas em casos de suporte — com registro de auditoria |
| Suspender conta temporariamente | ✅ | ✅ | Requer motivo e prazo definido |
| Encerrar conta permanentemente | ✅ | ✅ | Requer aprovação de Super Admin |
| Redefinir senha | ✅ | ✅ | Apenas via link enviado ao e-mail do usuário — admin não vê a nova senha |
| Visualizar histórico de agendamentos | ✅ | ✅ | Somente leitura |
| Visualizar histórico financeiro | ✅ | ✅ | Perfil Financeiro ou Super Admin |

### Regras de negócio — Gerenciamento de Usuários

| Regra | Descrição |
| --- | --- |
| RN-ADM-012 | Nenhum admin pode alterar dados de usuário sem registrar justificativa no log de auditoria |
| RN-ADM-013 | Encerramento permanente de conta requer dupla aprovação: solicitante + Super Admin |
| RN-ADM-014 | Suspensão temporária notifica o usuário imediatamente com motivo e prazo |
| RN-ADM-015 | Admin não pode visualizar senhas em nenhuma circunstância — senhas são sempre hashed |
| RN-ADM-016 | Acesso a dados clínicos de pacientes pelo painel admin é restrito ao perfil Suporte e Super Admin, com registro obrigatório |

---

## 10.3 Moderação de Avaliações

### Fluxo de Moderação

`Avaliação denunciada pelo profissional ou outro usuário
        ↓
Status: Em análise (avaliação fica oculta publicamente)
        ↓
Admin revisa conteúdo e contexto
        ↓
    ┌───────┴────────┐
Manter avaliação   Remover avaliação
    ↓                    ↓
Reativar            Notificar autor
publicamente        com motivo`

### Regras de negócio — Moderação

| Regra | Descrição |
| --- | --- |
| RN-ADM-017 | Avaliação denunciada fica oculta publicamente enquanto estiver em análise |
| RN-ADM-018 | O prazo máximo para decisão de moderação é de 5 dias úteis |
| RN-ADM-019 | Remoção de avaliação deve incluir motivo registrado e notificação ao autor |
| RN-ADM-020 | O profissional denunciante deve ser notificado do resultado da moderação |
| RN-ADM-021 | Avaliações removidas são arquivadas internamente — não excluídas — para fins de auditoria |
| RN-ADM-022 | Usuário que tiver 3 avaliações removidas por violação de conduta pode ter a conta suspensa automaticamente |

---

## 10.4 Configurações Financeiras

### Parâmetros configuráveis

| Parâmetro | Descrição | Quem pode alterar |
| --- | --- | --- |
| Taxa de serviço da plataforma (%) | Percentual retido de cada consulta | Super Admin + Financeiro |
| Prazo de repasse ao profissional | Quantos dias após a consulta o repasse é processado | Super Admin + Financeiro |
| Valor mínimo de consulta | Piso de preço permitido por especialidade | Super Admin |
| Valor máximo de consulta | Teto de preço por especialidade (se aplicável) | Super Admin |
| Regras de reembolso | Prazo e percentual de devolução por tipo de cancelamento | Super Admin + Financeiro |

### Regras de negócio — Configurações Financeiras

| Regra | Descrição |
| --- | --- |
| RN-ADM-023 | Toda alteração de taxa ou regra financeira deve registrar o valor anterior, o novo valor, o responsável e o timestamp |
| RN-ADM-024 | Alterações financeiras só entram em vigor para novos agendamentos — nunca retroativamente |
| RN-ADM-025 | Redução do prazo de repasse abaixo de 1 dia útil requer aprovação do Super Admin |
| RN-ADM-026 | Qualquer alteração de taxa acima de 5 pontos percentuais requer dupla aprovação |

---

## 10.5 Configurações de Catálogo

### O que pode ser configurado

| Configuração | Descrição |
| --- | --- |
| Especialidades disponíveis | Adicionar, editar ou desativar especialidades na plataforma |
| Categorias de profissionais | Médico, psicólogo, nutricionista, fisioterapeuta, etc. |
| Limites de agendamento por especialidade | Máximo de consultas por dia/semana por profissional |
| Duração padrão de consulta por especialidade | Tempo padrão sugerido ao profissional no cadastro |
| Regiões de atendimento disponíveis | Habilitar ou restringir cidades/estados na plataforma |

### Regras de negócio — Catálogo

| Regra | Descrição |
| --- | --- |
| RN-ADM-027 | Desativar uma especialidade não afeta profissionais já cadastrados nela — apenas impede novos cadastros |
| RN-ADM-028 | Alteração de limite de agendamento entra em vigor no próximo dia útil |
| RN-ADM-029 | Toda alteração de catálogo deve ser registrada com histórico de versões |

---

## 10.6 Relatórios e Métricas

### Relatórios disponíveis

| Relatório | Descrição | Frequência sugerida |
| --- | --- | --- |
| Receita total da plataforma | Valor total transacionado e taxa retida por período | Diária / Mensal |
| Repasses realizados | Total repassado a profissionais por período | Mensal |
| Consultas realizadas | Volume por especialidade, região e período | Semanal |
| Taxa de cancelamento | % de cancelamentos por motivo e perfil | Semanal |
| Churn de usuários | Pacientes e profissionais que deixaram de usar a plataforma | Mensal |
| Novos cadastros | Volume de novos pacientes e profissionais por período | Semanal |
| Avaliações e NPS | Média de avaliações por especialidade e NPS geral | Mensal |
| Tempo médio de resposta no chat | Média de resposta do profissional por período | Semanal |

### Regras de negócio — Relatórios

| Regra | Descrição |
| --- | --- |
| RN-ADM-030 | Relatórios com dados pessoais identificáveis só são acessíveis para Super Admin e perfis autorizados |
| RN-ADM-031 | Analistas de dados acessam apenas relatórios com dados anonimizados ou agregados |
| RN-ADM-032 | Exportação de relatórios deve ser registrada em log com usuário, timestamp e filtros aplicados |
| RN-ADM-033 | Dados de relatórios não podem ser usados para contato direto com usuários sem base legal definida |

---

## 10.7 Logs de Auditoria e Segurança

### Ações que geram log obrigatório no painel

| Ação | Dados registrados |
| --- | --- |
| Login no painel admin | Usuário, IP, dispositivo, timestamp, sucesso/falha |
| Aprovação ou rejeição de cadastro | Admin responsável, profissional afetado, motivo, timestamp |
| Suspensão ou encerramento de conta | Admin responsável, usuário afetado, motivo, prazo |
| Alteração de configuração financeira | Valor anterior, novo valor, admin responsável, timestamp |
| Acesso a dados clínicos de paciente | Admin, paciente acessado, justificativa, timestamp |
| Exportação de relatório | Admin, relatório, filtros, timestamp |
| Remoção de avaliação | Admin, avaliação, motivo, timestamp |

### Regras de negócio — Logs

| Regra | Descrição |
| --- | --- |
| RN-ADM-034 | Logs do painel administrativo são imutáveis — nenhum perfil pode editá-los ou excluí-los |
| RN-ADM-035 | Logs devem ser retidos por no mínimo 5 anos |
| RN-ADM-036 | Tentativas de acesso negado devem gerar alerta automático se ocorrerem mais de 3 vezes seguidas |
| RN-ADM-037 | Logs devem estar disponíveis para consulta dentro do próprio painel, com filtros por usuário, ação e período |

---

## 🚨 Casos Especiais e Edge Cases

| Situação | Comportamento esperado |
| --- | --- |
| Admin tenta aprovar o próprio cadastro de familiar | Sistema deve bloquear conflito de interesse — cadastros não podem ser aprovados por admins com vínculo no sistema |
| Super Admin é desligado da empresa | Conta deve ser revogada imediatamente e suas ações recentes revisadas |
| Relatório exportado contém dado sensível indevido | Registrar o incidente, notificar o DPO e acionar plano de resposta do M8 |
| Profissional contesta rejeição de cadastro | Deve haver canal formal de contestação com prazo de resposta de 5 dias úteis |
| Configuração financeira alterada por engano | Reversão possível apenas pelo Super Admin, com registro do motivo — sem apagar o histórico |
| Dois admins tentam moderar a mesma avaliação simultaneamente | Sistema deve implementar lock otimista — apenas o primeiro que confirmar a ação prevalece |

---

## 📊 Status do Módulo

| Item | Status |
| --- | --- |
| Especificação de regras de negócio | 🟡 Em andamento |
| Definição de perfis administrativos (RBAC) | ⚪ Não iniciado |
| Design das telas do painel | ⚪ Não iniciado |
| Implementação backend | ⚪ Não iniciado |
| Implementação frontend | ⚪ Não iniciado |
| Integração com logs de auditoria (M8) | ⚪ Não iniciado |
| Relatórios e dashboard de métricas | ⚪ Não iniciado |
| Testes e QA | ⚪ Não iniciado |

---

## 🔗 Dependências com outros módulos

| Módulo | Tipo de dependência |
| --- | --- |
| M1 — Autenticação e Cadastro | Autenticação dos admins, 2FA, controle de sessão |
| M2 — Perfil do Profissional | Fluxo de aprovação e rejeição de cadastros |
| M4 — Agendamento | Configuração de limites por especialidade |
| M5 — Pagamento | Configuração de taxas, repasses e reembolsos |
| M6 — Comunicação | Moderação de conteúdo do chat em casos extremos |
| M7 — Notificações | Disparos de notificação para aprovações, rejeições e suspensões |
| M8 — Segurança e LGPD | RBAC, logs de auditoria, resposta a incidentes |

---

## ❓ Decisões em Aberto

| Decisão | Contexto |
| --- | --- |
| Ferramenta de painel | Construir do zero ou usar base como AdminJS, Retool ou similar? |
| Dashboard de métricas | Integrar com Metabase, Grafana ou construir visualizações próprias? |
| Controle de acesso por IP | Restringir painel apenas à rede corporativa/VPN? |
| Dupla aprovação | Quais ações além do encerramento de conta exigem dupla aprovação? |
| Exportação de relatórios | Formatos suportados: CSV, PDF, ambos? |
| Auditoria em tempo real | Alertas imediatos para ações críticas ou revisão periódica? |

Funcionalidades do painel

O painel administrativo é o centro de controle da plataforma, acessível apenas pelo time interno. Permite gerenciar usuários, acompanhar métricas e configurar regras de negócio.

## 10.1 Funcionalidades do painel

- Aprovação e rejeição de cadastros de profissionais
- Gerenciamento de usuários (clientes e profissionais)
- Moderação de avaliações denunciadas
- Configuração de taxas de serviço e regras de repasse
- Configuração de limites de agendamento por especialidade
- Gerenciamento de categorias e especialidades disponíveis
- Relatórios de uso, receita e churn
- Logs de auditoria e segurança

[📦 Produto & Negócio — Painel Administrativo](M9-Produto-Negocio-PainelAdm.md)

[⚙️ Especificação Técnica — Painel Administrativo](M9-Produto-Negocio-PainelAdm.md)
