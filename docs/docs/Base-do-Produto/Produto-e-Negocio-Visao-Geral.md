
# 📦 Produto & Negócio — Visão Geral
> **Audiência:** Product Manager, Designer, Stakeholders de Negócio
> 

> **Objetivo:** Alinhar proposta de valor, atores, jornadas e escopo do MVP sem ambiguidade.
> 

---

## 1. O Problema que Resolvemos

Encontrar um profissional de saúde confiável no Brasil ainda é fragmentado: o paciente depende de indicações boca-a-boca, busca em planos de saúde desatualizados ou sites com pouca credibilidade. Do lado do profissional, a agenda é gerida por telefone, WhatsApp ou ferramentas genéricas sem integração com pagamento.

Nossa plataforma resolve isso centralizando **busca, agendamento, pagamento e comunicação** em um único lugar — com verificação de credenciais dos profissionais e conformidade LGPD desde o dia um.

---

## 2. Proposta de Valor
| Para quem | Proposta |
| --- | --- |
| **Paciente** | Encontre profissionais verificados, agende e pague em minutos, com histórico e avaliações reais |
| **Profissional** | Gerencie agenda, receba pagamentos automaticamente e construa reputação digital com credibilidade |
| **Administrador** | Garanta conformidade, qualidade dos profissionais e visibilidade operacional da plataforma |

## 3. Atores Principais

### 3.1 Paciente (Cliente)

Pessoa física que busca atendimento de saúde. Pode agendar consultas presenciais ou domiciliares, realizar pagamento pela plataforma, avaliar profissionais e acessar documentos do atendimento.

### 3.2 Profissional de Saúde

Médico, dentista, fisioterapeuta ou qualquer profissional regulamentado por conselho de classe (CRM, CRO, CREFITO, etc.). Responsável por manter perfil atualizado, gerenciar agenda e comunicar-se com pacientes via plataforma.

### 3.3 Administrador da Plataforma

Equipe interna responsável por aprovar novos profissionais, monitorar a saúde operacional da plataforma, gerir conformidade LGPD e moderar avaliações.

---

## 4. Jornadas Principais no MVP

### Jornada do Paciente

1. Acessa a plataforma → cria conta ou faz login
2. Busca profissional por especialidade e localização
3. Visualiza perfil, avaliações e disponibilidade
4. Agenda horário e realiza pagamento
5. Recebe confirmação e lembretes
6. Realiza consulta
7. Avalia o profissional

### Jornada do Profissional

1. Cria conta e submete documentos para validação
2. Aguarda aprovação do administrador
3. Configura perfil, agenda e valores
4. Recebe solicitações de agendamento
5. Confirma ou recusa agendamentos
6. Realiza atendimento
7. Recebe repasse do pagamento

---

## 5. Premissas do MVP

- Todo agendamento exige autenticação ativa — sem guest checkout
- Profissional precisa ser aprovado antes de aparecer nas buscas
- Pagamento ocorre 100% dentro da plataforma — negociação externa é vedada
- Dados de saúde e laudos são criptografados e acessíveis apenas pelas partes da consulta
- LGPD é requisito, não backlog — consentimento e políticas de privacidade desde o cadastro
- Modalidades suportadas no MVP: **presencial** e **domiciliar**
- Teleconsulta fica fora do escopo do MVP(**possivel meeting**)
- **Forma de validar que o medico chamado e ele mesmo PS**

---

## 6. O que está FORA do MVP (Next Releases)

- Teleconsulta (vídeo)(**nao obrigatorio por intermedio da plataforma**)
- Planos de saúde e reembolso de convênio
- Prontuário eletrônico completo
- Sincronização com Google Calendar / Apple Calendar
- App mobile nativo (MVP é web responsivo)
- Multi-idioma

---

## 7. Critérios de Sucesso do MVP
| Métrica | Meta inicial (3 meses pós-lançamento) |
| --- | --- |
| Profissionais aprovados e ativos | ≥ 10 |
| Consultas agendadas e realizadas | ≥ 25 |
| Taxa de conversão busca → agendamento | ≥ 15% |
| NPS dos pacientes | ≥ 40 |
| Taxa de churn de profissionais | ≤ 20% no primeiro mês |

8. Mapa de Módulos do MVP
┌─────────────────────────────────────────────────────────┐
│                   PLATAFORMA DE SAÚDE                   │
├──────────────┬──────────────────┬───────────────────────┤
│  M1 Autent.  │  M2 Perfil Prof. │  M3 Busca & Categ.    │
├──────────────┴──────────────────┴───────────────────────┤
│         M4 Agendamento          │   M5 Pagamento         │
├──────────────────────────────────────────────────────────┤
│  M6 Chat/Comunicação  │  M7 Notificações                │
├──────────────────────────────────────────────────────────┤
│  M8 Segurança & LGPD  │  M9 Painel Admin                │
└──────────────────────────────────────────────────────────┘

## 9. Fluxograma de Jornadas (LucidChart)

> 🔗 **[Inserir link do LucidChart aqui]**
> 

> Fluxograma deve cobrir:
> 

> - Jornada completa do paciente (cadastro → avaliação)
> 

> - Jornada do profissional (onboarding → atendimento)
> 

> - Fluxo de aprovação pelo administrador
> 

---

## 10. Glossário de Domínio

| Termo | Definição |
| --- | --- |
| **Consulta** | Atendimento de saúde agendado e pago via plataforma |
| **Agendamento** | Solicitação de horário feita pelo paciente |
| **Aprovação** | Processo de validação manual de credenciais do profissional pelo admin |
| **Modalidade** | Forma de atendimento: presencial (consultório) ou domiciliar (residência do paciente) |
| **Repasse** | Transferência do valor da consulta ao profissional após taxa da plataforma |
| **Laudo** | Documento médico gerado após atendimento, acessível apenas às partes |

