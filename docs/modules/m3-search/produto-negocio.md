# 📦 Produto & Negócio — Busca e Categorias

> **Audiência:** Product Manager, Designer, Stakeholders  
> **Objetivo:** Descrever a experiência de busca do paciente, as categorias disponíveis, os filtros e as regras que governam o que é exibido.

---

## 1. Por que este módulo importa

A busca é o **primeiro contato do paciente com o produto** depois do login. É aqui que a promessa de valor é testada: se o paciente não encontrar o profissional certo rápido, ele sai.  
A busca precisa ser rápida, relevante e confiável — exibindo apenas profissionais verificados e com perfil completo.

---

## 2. Como funciona a busca

### Jornada do paciente

1. Paciente acessa a página de busca  
   → Pode buscar por: especialidade, nome do profissional, cidade  

2. Aplica filtros (opcional)  
   → Modalidade, faixa de preço, avaliação mínima, data disponível  

3. Visualiza lista de resultados  
   → Card com: foto, nome, especialidade, nota, preço e modalidade  

4. Clica em um profissional  
   → Vai para o Perfil do Profissional (M2)  

5. Agenda a consulta  
   → Vai para o Agendamento (M4)  

---

## 3. Categorias de Profissionais

As categorias são gerenciadas pelo administrador da plataforma. No lançamento do MVP, as categorias iniciais são:

| Categoria                | Subespecialidades (exemplos)       | Conselho |
|---------------------------|------------------------------------|----------|
| Clínico Geral             | —                                  | CRM      |
| Cardiologia               | Cardiovascular, Eletrofisiologia   | CRM      |
| Dermatologia              | Estética, Dermatoscopia            | CRM      |
| Ginecologia e Obstetrícia | Pré-natal, Colposcopia             | CRM      |
| Pediatria                 | Neonatologia, Adolescente          | CRM      |
| Psiquiatria               | Adulto, Infantil                   | CRM      |
| Psicologia                | Clínica, Organizacional            | CRP      |
| Odontologia               | Clínico Geral, Ortodontia, Implantes | CRO    |
| Fisioterapia              | Ortopédica, Neurológica, Respiratória | CREFITO |
| Nutrição                  | Clínica, Esportiva                 | CRN      |
| Urologia                  | Próstata, Uroginecologia           | CRM      |
| Geriatria                 | —                                  | CRM      |

⚠️ **Convênios/planos de saúde:** filtro de convênio está **fora do MVP**. Profissional informa manualmente quais aceita, mas validação automática fica para v2.

---

## 4. Filtros Disponíveis no MVP

| Filtro              | Tipo               | Observação |
|---------------------|--------------------|------------|
| **Especialidade**   | Seleção            | Obrigatória ou livre |
| **Cidade / Bairro** | Texto + Geolocalização | Busca por proximidade |
| **Modalidade**      | Checkbox           | Presencial / Domiciliar |
| **Faixa de preço**  | Slider (mín — máx) | Baseado em `consultation_price` do perfil |
| **Avaliação mínima**| Estrelas (1–5)     | Filtra por `average_rating >=` |
| **Disponibilidade** | Seletor de data    | Exibe só profissionais com slots naquele dia |

---

## 5. Regras de Exibição

- Apenas profissionais com `approval_status = APPROVED`
- Apenas profissionais com `is_profile_complete = true`
- Profissionais com `accepts_new_appointments = false` aparecem na busca mas **sem botão de agendar**
- Resultados ordenados por relevância (algoritmo na seção técnica)
- Profissionais suspensos ou rejeitados **não aparecem em nenhuma circunstância**

---

## 6. O que o card de resultado exibe

| Campo                  | Fonte |
|-------------------------|-------|
| Foto de perfil          | `profile_photo_url` |
| Nome completo           | `professionals.full_name` |
| Especialidade primária  | `professional_specialties` (is_primary) |
| Nota média              | `average_rating` + `total_reviews` |
| Menor preço de consulta | Mínimo entre `price_presential` e `price_home_visit` |
| Modalidades             | Ícones de presencial / domiciliar |
| Cidade do consultório   | `office_addresses.city` |
| Próximo horário disponível | Calculado em tempo real (M4 — Agendamento) |

---

## 7. Critérios de Aceite (MVP)

- [ ] Busca retorna resultados em menos de 1 segundo para até 1.000 profissionais  
- [ ] Nenhum profissional pendente, rejeitado ou suspenso aparece nos resultados  
- [ ] Filtros de especialidade, modalidade e localidade funcionam combinados  
- [ ] Card de resultado exibe todos os campos listados na seção 6  
- [ ] Profissional com agenda fechada aparece mas sem botão de agendar  
- [ ] Busca sem filtro retorna lista paginada com 20 resultados por página  
- [ ] Admin consegue adicionar/remover especialidades sem deploy  

---

## 8. Fluxograma (LucidChart)

🔗 **[Inserir link do LucidChart — Fluxo de Busca e Descoberta]**

Cobrir: entrada do paciente → aplica filtros → lista resultados → perfil → agendamento

