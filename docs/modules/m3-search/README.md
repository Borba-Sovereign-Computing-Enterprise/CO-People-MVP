
# 🔍 M3 — Busca e Categorias

> 👋 **Bem-vindo(a) ao módulo de Busca e Categorias.**  
A busca é o primeiro ponto de conversão do paciente — é aqui que ele encontra o profissional certo.

---

## 👋 Por onde começar

Se você acabou de entrar neste módulo, siga esta ordem:

1. 📌 Leia a **Visão Geral**
2. 🗂️ Consulte o **Guia por Perfil**
3. 📋 Revise as **Categorias de Profissionais**
4. 🔎 Leia os **Filtros disponíveis**
5. ⚙️ Verifique as **Regras Gerais** e **Edge Cases**

---

## 🎯 Visão Geral

A busca é o **primeiro ponto de conversão do paciente** na plataforma.

Exibe apenas:
- ✅ Profissionais aprovados
- ✅ Perfil 100% completo

Filtros disponíveis:
- Especialidade
- Localização
- Modalidade
- Preço
- Avaliação
- Disponibilidade
- Convênio

---

## 🗂️ Guia por Perfil

| Perfil | Comece por |
|--------|-----------|
| Product Manager / Designer | Produto & Negócio |
| Backend / Tech Lead | Especificação Técnica |
| Frontend | Produto & Negócio → Especificação Técnica |
| QA / Tester | Produto & Negócio + Edge Cases |

---

## 📦 Sub-páginas

| Sub-página | Conteúdo |
|-----------|---------|
| Produto & Negócio | Jornada, filtros, regras e critérios de aceite |
| Especificação Técnica | MVP, busca, geosearch, algoritmo e endpoints |

---

## 🏥 4.1 Categorias de Profissionais

### Principais categorias

- Odontologia
- Ginecologia e Obstetrícia
- Urologia
- Cardiologia
- Dermatologia
- Fisioterapia
- Nutrição
- Psicologia e Psiquiatria
- Pediatria
- Geriatria
- Outras especialidades (via admin)

---

### 📏 Regras de negócio — Categorias

| Código | Regra |
|-------|------|
| RN-BUS-001 | Apenas categorias ativas aparecem |
| RN-BUS-002 | Profissional pode ter múltiplas sub-especialidades |
| RN-BUS-003 | Apenas admin cria categorias |
| RN-BUS-004 | Categoria principal define agrupamento |
| RN-BUS-005 | Categoria desativada não remove profissional |

---

## 🔎 4.2 Filtros Disponíveis

| Filtro | Tipo | Comportamento |
|-------|------|--------------|
| Categoria | Multi ou único | Filtra especialidade |
| Localização | Texto + geo | Cidade, bairro ou raio |
| Modalidade | Multi | Presencial / domiciliar / tele |
| Preço | Range | Valor da consulta |
| Avaliação | Seleção | Ex: 4+ estrelas |
| Disponibilidade | Data | Agenda em tempo real |
| Convênio | Multi | Planos aceitos |

---

### 📏 Regras de negócio — Filtros

| Código | Regra |
|-------|------|
| RN-BUS-006 | Filtros são opcionais |
| RN-BUS-007 | Disponibilidade em tempo real |
| RN-BUS-008 | Geolocalização exige consentimento |
| RN-BUS-009 | Sem geo → entrada manual |
| RN-BUS-010 | Teleconsulta ignora localização |
| RN-BUS-011 | Preço respeita limites do sistema |
| RN-BUS-012 | Filtros usam lógica AND |
| RN-BUS-013 | Zero resultados → sugerir ajustes |

---

## ⚙️ Regras Gerais

| Código | Regra |
|-------|------|
| RN-BUS-014 | Apenas profissionais aprovados aparecem |
| RN-BUS-015 | Perfis incompletos são ocultos |
| RN-BUS-016 | Paginação: 20 por página |
| RN-BUS-017 | Busca cobre nome + especialidade |
| RN-BUS-018 | Ordenação por relevância |
| RN-BUS-019 | Ordenação manual disponível |
| RN-BUS-020 | Sem resultados patrocinados |

---

## 🚨 Edge Cases

| Situação | Comportamento |
|---------|--------------|
| Zero resultados | Sugestão de ajuste |
| Novo profissional aprovado | Aparece em até 5 min |
| Agenda suspensa | Aparece sem horários |
| Mesmo nome | Diferenciar por dados |
| Erro de digitação | Fuzzy search |
| Alteração de preço | Atualização imediata |
| Convênio removido | Atualiza em até 5 min |

---

## 📊 Status do Módulo

| Item | Status |
|-----|-------|
| Regras de negócio | 🟡 Em andamento |
| Produto & Negócio | ⚪ Não iniciado |
| Especificação Técnica | ⚪ Não iniciado |
| Algoritmo | ⚪ Não iniciado |
| Backend | ⚪ Não iniciado |
| Frontend | ⚪ Não iniciado |
| QA | ⚪ Não iniciado |

---

## 🔗 Dependências

| Módulo | Uso |
|------|----|
| M1 — Autenticação | Identidade do paciente |
| M2 — Perfil | Dados exibidos |
| M4 — Agendamento | Disponibilidade |
| M9 — Admin | Categorias |

---

## ❓ Decisões em Aberto

- Engine de busca (Postgres, Elasticsearch, Typesense)
- Algoritmo de relevância
- Estratégia de geosearch
- Tempo de reindexação
- Busca sem login
- Histórico de buscas

---

## 📎 Links

- Produto & Negócio  
- Especificação Técnica

---

## 🧠 Resumo

A busca conecta o paciente ao profissional ideal com base em:

- Filtros inteligentes
- Dados confiáveis
- Regras de negócio bem definidas




---


- 📦 [Produto & Negócio — Busca e Categorias](./produto-negocio.md)

- ⚙️ [Especificação Técnica — Busca e Categorias](./especificacao-tecnica.md)

---


