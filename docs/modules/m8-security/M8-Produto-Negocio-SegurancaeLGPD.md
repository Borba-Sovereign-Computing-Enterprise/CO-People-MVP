# Produto & Negócio — Segurança e LGPD

> **Audiência:** Product Manager, Stakeholders, Designer, Juridico
> 

> **Objetivo:** Definir as obrigações com o usuário, os direitos que a plataforma deve garantir e o plano de resposta a incidentes em linguagem não-técnica.
> 

---

## 1. Por que isso é não-negociável

A plataforma lida com **dados sensíveis de saúde** (Art. 11 da LGPD), a categoria mais protegida pela lei. Qualquer vazamento pode gerar danos irreparáveis aos titulares e responsabilidade civil, administrativa e criminal para a empresa. Segurança e conformidade **não são features — são requisitos de existência do produto**.

---

## 2. Dados que tratamos e por quê

| Categoria | O que é | Base legal |
| --- | --- | --- |
| Identificação pessoal | CPF, nome, endereço, telefone, e-mail | Execução de contrato (Art. 7º, V) |
| Dados clínicos | Laudos, histórico, exames, consultas | Tutela da saúde (Art. 11, II, f) |
| Comunicação | Mensagens e arquivos do chat | Contrato + Legítimo interesse |
| Dados financeiros | Dados bancários do profissional, pagamentos | Execução de contrato |
| Dados de acesso | Logs de login, IP, dispositivo | Segurança (Art. 7º, IX) |
| Dados de menores | Qualquer dado de menor de 18 anos | Proteção especial + autorização do responsável (Art. 14) |

---

## 3. Direitos do Usuário — Como a Plataforma Atende

| Direito (LGPD) | Como a plataforma atende | Prazo |
| --- | --- | --- |
| Confirmar se tratamos seus dados (Art. 18, I) | Seção "Meus Dados" no perfil | Imediato |
| Acessar seus dados (Art. 18, II) | Exportação disponível no perfil | Até 15 dias |
| Corrigir dados (Art. 18, III) | Edição no perfil | Imediato |
| Anonimizar ou excluir (Art. 18, IV) | Solicitação via suporte | Até 15 dias |
| Portabilidade (Art. 18, V) | Exportação em JSON ou PDF | Até 15 dias |
| Revogar consentimento (Art. 18, IX) | Opção nas configurações de privacidade | Imediato |
| Saber com quem compartilhamos (Art. 18, VII) | Política de privacidade detalhada | Imediato |

---

## 4. Obrigações da Plataforma

| Obrigação | Status no MVP |
| --- | --- |
| Termo de consentimento explícito por finalidade no cadastro | ⚠️ A definir |
| Política de privacidade acessível em todas as telas | ⚠️ A definir |
| DPO (Encarregado de Proteção de Dados) designado com canal público | ⚠️ A definir |
| Relatório de Impacto (RIPD) elaborado antes do lançamento | ⚠️ A definir |
| Consentimento granular (aceitar/recusar finalidades individualmente) | ✅ Implementar |
| Registro do aceite com timestamp e versão do documento | ✅ Implementar |

---

## 5. Regras de Negócio Chave

| Código | Regra |
| --- | --- |
| RN-SEG-001 | Dados clínicos nunca compartilhados com terceiros sem consentimento explícito |
| RN-SEG-002 | Dados de menores só com autorização do responsável legal |
| RN-SEG-003 | Dados bancários tokenizados — nunca em texto puro |
| RN-SEG-008 | 2FA obrigatório para profissionais de saúde (1º acesso e a cada 30 dias) |
| RN-SEG-015 | Consentimento granular por finalidade |
| RN-SEG-016 | Não é perm. condicionar uso da plataforma a finalidades desnecessárias (Art. 7º, §3º) |
| RN-SEG-017 | Exclusão de conta anonimiza dados do titular — não apenas desativa |
| RN-SEG-020 | Aceite dos termos registrado com timestamp, versão e IP |

---

## 6. Plano de Resposta a Incidentes

### Classificação

| Nível | Descrição | Exemplo |
| --- | --- | --- |
| 🔴 Crítico | Vazamento de dados sensíveis confirmado | Dump do banco exposto |
| 🟠 Alto | Acesso não autorizado a dados de usuários | Conta admin comprometida |
| 🟡 Médio | Tentativa de ataque sem sucesso | Brute force bloqueado |
| ⚪ Baixo | Anomalia sem impacto confirmado | Pico incomum de requisições |

### Fluxo de Resposta

| Etapa | Ação | Prazo |
| --- | --- | --- |
| 1. Detecção | Identificar e isolar o sistema afetado | Imediato |
| 2. Contenção | Revogar acessos, desativar endpoints comprometidos | Até 1 hora |
| 3. Avaliação | Mapear dados afetados e extensão do incidente | Até 24 horas |
| 4. Notificação ANPD | Comunicar à Autoridade Nacional | Até 72h (Art. 48 LGPD) |
| 5. Notificação titulares | Informar usuários afetados com clareza | Até 72h |
| 6. Remediação | Corrigir vulnerabilidade e restaurar serviço | Conforme severidade |
| 7. Post-mortem | Causa raiz, impacto e medidas preventivas | Até 7 dias |

---
## 7. Decisões em Aberto

| Decisão | Contexto |
| --- | --- |
| Designação do DPO | Interno ou terceirizado? |
| Escopo do RIPD | Um consolidado ou um por módulo? |
| RTO e RPO | Quanto tempo de indisponibilidade e perda de dados são aceitáveis? |
| Ferramenta de SIEM | Datadog, Elastic Security ou outro? |
| Política de privacidade | Versão final revisada por advogado especialista em LGPD |
