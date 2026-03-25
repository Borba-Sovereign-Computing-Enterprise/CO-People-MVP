# 📦 Produto & Negócio — Busca e Categorias

> **Audiência:** Product Manager, Designer, Stakeholders  
> **Objetivo:** Descrever a experiência de busca do paciente, categorias, filtros e regras de exibição.

---

## 1. 🎯 Por que este módulo importa

A busca é o **primeiro contato do paciente com o produto** após o login.

É aqui que a promessa de valor é testada:
- se o paciente não encontra rapidamente → ele abandona
- se encontra com confiança → converte

👉 A busca precisa ser:
- rápida
- relevante
- confiável

Exibindo apenas profissionais:
- ✅ verificados  
- ✅ com perfil completo  

---

## 2. 🔎 Como funciona a busca

### 🧭 Jornada do paciente

```txt
Passo 1: Paciente acessa a página de busca
  → Pode buscar por: especialidade, nome, cidade

Passo 2: Aplica filtros (opcional)
  → Modalidade, preço, avaliação, data

Passo 3: Visualiza resultados
  → Cards com foto, nome, especialidade, nota e preço

Passo 4: Clica em um profissional
  → Vai para Perfil (M2)

Passo 5: Agenda consulta
  → Vai para Agendamento (M4)


3. 🏥 Categorias de Profissionais

Categorias são gerenciadas pelo administrador.

📋 MVP — Categorias iniciais
Categoria	Subespecialidades (exemplos)	Conselho
Clínico Geral	—	CRM
Cardiologia	Cardiovascular, Eletrofisiologia	CRM
Dermatologia	Estética, Dermatoscopia	CRM
Ginecologia e Obstetrícia	Pré-natal, Colposcopia	CRM
Pediatria	Neonatologia, Adolescente	CRM
Psiquiatria	Adulto, Infantil	CRM
Psicologia	Clínica, Organizacional	CRP
Odontologia	Clínico Geral, Ortodontia, Implantodontia	CRO
Fisioterapia	Ortopédica, Neurológica, Respiratória	CREFITO
Nutrição	Clínica, Esportiva	CRN
Urologia	Próstata, Uroginecologia	CRM
Geriatria	—	CRM

⚠️ Convênios/planos de saúde

Fora do MVP
Profissional informa manualmente
Validação automática fica para v2
4. 🎛️ Filtros Disponíveis (MVP)
Filtro	Tipo	Observação
Especialidade	Seleção	Obrigatória ou livre
Cidade / Bairro	Texto + Geolocalização	Busca por proximidade
Modalidade	Checkbox	Presencial / Domiciliar
Faixa de preço	Slider	Baseado em consultation_price
Avaliação mínima	Estrelas (1–5)	average_rating >=
Disponibilidade	Data	Consulta agenda (M4)
5. ⚙️ Regras de Exibição
✅ Apenas approval_status = APPROVED
✅ Apenas is_profile_complete = true
⚠️ accepts_new_appointments = false
aparece, mas sem botão de agendar
🔒 Suspensos ou rejeitados → nunca aparecem
📊 Ordenação → por relevância (ver técnico)
6. 🧾 Card de Resultado
Campo	Fonte
Foto	profile_photo_url
Nome	professionals.full_name
Especialidade	professional_specialties (is_primary)
Avaliação	average_rating, total_reviews
Preço	mínimo entre preços disponíveis
Modalidades	presencial / domiciliar
Cidade	office_addresses.city
Disponibilidade	calculado em tempo real (M4)
7. ✅ Critérios de Aceite (MVP)
 Resposta < 1s (até 1.000 profissionais)
 Nenhum perfil inválido aparece
 Filtros combinados funcionam
 Card exibe todos os dados corretamente
 Profissional sem agenda → sem botão
 Paginação (20 por página)
 Admin gerencia categorias sem deploy
8. 🔄 Fluxo

🔗 Inserir link do LucidChart

Fluxo esperado:

entrada do paciente
aplicação de filtros
listagem de resultados
acesso ao perfil
agendamento

---
