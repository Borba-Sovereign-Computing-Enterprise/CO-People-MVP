# Especificação Técnica — Visão Geral

> **Audiência:** Engenheiros, Tech Leads, Arquitetos
> 

> **Objetivo:** Definir a arquitetura macro, bounded contexts, stack e decisões de design para o MVP.
> 

---

## 1. Arquitetura Macro

```
┌────────────────────────────────────────────────────────────┐
│                    CLIENTES                              │
│        Web App (React/TypeScript - Responsive)           │
└─────────────────────┬──────────────────────────────────────┘
                     │
              REST / HTTPS
                     │
┌─────────────────────┴──────────────────────────────────────┐
│           API GATEWAY  (Pedestal + Interceptors)         │
│     Auth │ Rate Limit │ Audit Log │ CORS │ Validations   │
└─────┬─────────┬──────────┬─────────┬─────────┬───────┘
          │         │          │         │         │
       ┌──┴──┐  ┌──┴──┐  ┌──┴──┐  ┌──┴──┐  ┌──┴──┐
       │ Auth │  │ Prof │  │ Sched│  │ Pay  │  │Notif │
       │ Svc  │  │ Svc  │  │ Svc  │  │ Svc  │  │ Svc  │
       └──┬──┘  └──┬──┘  └──┬──┘  └──┬──┘  └──┬──┘
          │         │          │         │         │
       ┌──┴─────────┴─────────┴─────────┴─────────┴─┐
       │           PostgreSQL (Primary Store)               │
       └───────────────────────────────────────────────┘
```

---

## 2. Bounded Contexts (Domínios)

| Contexto | Responsabilidade | Dep. Diretas |
| --- | --- | --- |
| **Identity** | Cadastro, autenticação, sessões, 2FA | — |
| **Professional Profile** | Perfil, especialidades, avaliações, modalidades | Identity |
| **Search & Discovery** | Busca por especialidade, localização, filtros | Professional Profile |
| **Scheduling** | Agenda, slots, confirmação, cancelamento | Identity, Professional Profile |
| **Payment** | Cobrança, split, repasse, reembolso | Scheduling, Identity |
| **Communication** | Chat, histórico, auditoria | Identity, Scheduling |
| **Notification** | Email, SMS, push (interno) | Todos |
| **Document** | Laudos, upload, acesso controlado | Identity, Scheduling |
| **Admin** | Painel, aprovação, métricas, LGPD | Todos |

---

## 3. Stack Tecnológica

---

### 3.0 Paradigmas & Filosofia de Engenharia

Antes de ver as tecnologias, é fundamental entender **como** trabalhamos — esses princípios guiam todas as decisões de código e arquitetura.

| Paradigma | O que significa na prática |
| --- | --- |
| **Data-Driven com EDN** | Toda configuração, roteamento, schemas e regras de negócio são representados como dados puros em arquivos `.edn` ou mapas Clojure. Zero "magia" em código — o sistema é descrito por dados, não por comportamento implícito. [O que é EDN →](https://github.com/edn-format/edn) |
| **Event Sourcing (Backend)** | O estado do sistema é derivado de um log imutável de eventos. Nunca sobrescrevemos registros — appendamos eventos (`AppointmentConfirmed`, `PaymentProcessed`, etc.). Isso garante auditabilidade total e rastreabilidade nativa para LGPD. |
| **Railway Oriented Programming** | Fluxos de execução modelados como `Result<Success, Error>`. Toda operação retorna `{:ok value}` ou `{:error reason}`. Sem exceções silenciosas, sem `nil` inesperado. Facilita composição e tratamento explícito de falhas. |
| **Microfrontends (Frontend)** | O frontend é dividido em aplicações independentes por domínio (ex: `app-auth`, `app-scheduling`, `app-professional-profile`). Cada micro-app tem seu próprio ciclo de deploy. Integração via Module Federation (Webpack/Vite). |

---

### 3.1 Backend — Clojure

| Camada | Tecnologia | Para que serve | Doc |
| --- | --- | --- | --- |
| **Linguagem** | Clojure 1.12 | Linguagem funcional na JVM. Imutabilidade por padrão, REPL-driven development, interop Java completo. Base de toda a lógica de negócio. | [clojure.org](http://clojure.org) |
| **HTTP Framework** | Pedestal | Framework para APIs REST em Clojure. Baseado em interceptors (middleware composável), não em funções puras. Define rotas, autentica, loga e valida via cadeia de interceptors. **Usado 100% para todas as rotas REST.** | [pedestal.io](http://pedestal.io) |
| **Config & Lifecycle** | Aero + Integrant | **Aero** lê arquivos `.edn` de configuração com suporte a múltiplos environments (`#profile`). **Integrant** gerencia o ciclo de vida dos componentes (start/stop/reset) de forma declarativa via mapa EDN. | [Aero](https://github.com/juxt/aero) · [Integrant](https://github.com/weavejester/integrant) |
| **Schema & Validação** | Malli *(primário)* | Biblioteca de schemas data-driven para Clojure. Define shapes de dados, valida entradas, gera documentação e specs de forma declarativa. Escolha principal para MVP. *(Ver seção 3.5 — tradeoff com Spec Alpha)* | [Malli](https://github.com/metosin/malli) |
| **DB Access** | next.jdbc (100%) | Wrapper fino e idiomático sobre JDBC. Usado **100%** para todas as operações de banco. Sem ORMs. Queries como dados (mapas/vetores). Controle total sobre SQL sem abstração excessiva. | [next.jdbc](https://github.com/seancorfield/next-jdbc) |
| **Query Builder** | HoneySQL | Constrói queries SQL como estruturas de dados Clojure (vetores/mapas). Composável, testável, sem string concatenation. Usado em conjunto com next.jdbc. | [HoneySQL](https://github.com/seancorfield/honeysql) |
| **Migrations** | Migratus | Gerenciamento de migrations SQL em Clojure. Arquivos SQL versionados aplicados de forma ordenada e idempotente. | [Migratus](https://github.com/yogthos/migratus) |
| **Testes Unitários** | clojure.test | Biblioteca nativa do Clojure. Usada para testar funções puras, transformações de dados, validações e regras de negócio isoladas. | [clojure.test](https://clojure.github.io/clojure/clojure.test-api.html) |
| **Testes de Integração** | state-flow | Framework para testes de fluxo com estado. Permite descrever cenários completos (ex: cadastro → agendamento → pagamento) de forma composável e legível. Ideal para testar bounded contexts de ponta a ponta. | [state-flow](https://github.com/nubank/state-flow) |

---

### 3.2 Messaging & Streaming — Kafka

| Camada | Tecnologia | Para que serve | Doc |
| --- | --- | --- | --- |
| **Message Broker** | Apache Kafka | Backbone de eventos da plataforma. Transporta eventos de domínio entre bounded contexts (`AppointmentScheduled`, `PaymentCompleted`, `ProfessionalApproved`). Garante desacoplamento entre serviços e persistência do log de eventos. | [kafka.apache.org](http://kafka.apache.org) |
| **Schema Registry** | Confluent Schema Registry | Registra e versiona schemas Avro/Protobuf dos eventos. Garante compatibilidade entre producers e consumers sem quebrar contratos. | [Schema Registry](https://docs.confluent.io/platform/current/schema-registry/index.html) |
| **Client Clojure** | clj-kafka / jackdaw | Client Kafka idiomático para Clojure. Jackdaw oferece abstração data-driven para producers, consumers e Kafka Streams. | [Jackdaw](https://github.com/FundingCircle/jackdaw) |
| **Stream Processing** | Kafka Streams | Processamento stateful de eventos em tempo real diretamente no Kafka. Usado para agregações, joins e projeções (ex: calcular nota média do profissional após novo review). | [Kafka Streams](https://kafka.apache.org/documentation/streams/) |

> **Tópicos principais (MVP):** `professional.approved`, `appointment.scheduled`, `appointment.confirmed`, `appointment.completed`, `payment.processed`, `review.published`
> 

---

### 3.3 Banco de Dados

| Tecnologia | Papel | Para que serve | Doc |
| --- | --- | --- | --- |
| **PostgreSQL 16** | Store principal | Armazena todas as entidades transacionais: users, professionals, appointments, payments, reviews. ACID completo, JSONB para dados semi-estruturados, suporte nativo a criptografia de colunas. | [postgresql.org](http://postgresql.org) |
| **Redis** | Cache & sessões | Cache de refresh tokens, rate limiting por IP/usuário, sessões temporárias (ex: token MFA). TTL nativo elimina garbage collection manual. | [redis.io](http://redis.io) |

---

### 3.4 Frontend — Primário: React + TypeScript (Microfrontends)

**Arquitetura:** Microfrontends via **Module Federation**. Cada bounded context de UI é uma aplicação React independente com seu próprio build, deploy e ownership. Comunicação entre micro-apps via eventos de domínio (custom events / shared store).

**Modelo Data-Driven no Frontend:** Assim como no backend, estruturas de dados guiam o comportamento da UI. Formulários, rotas e regras de validação são derivados de schemas (Zod) — não hardcoded em componentes.
| Camada | Tecnologia | Para que serve | Doc |
| --- | --- | --- | --- |
| **Linguagem** | TypeScript 5 | Tipagem estática sobre JavaScript. Types como documentação viva, error catching em compile-time, IntelliSense confiável. | [typescriptlang.org](http://typescriptlang.org) |
| **Framework** | React 19 | Biblioteca de UI baseada em componentes. Server Components para SSR/SSG, Concurrent Mode para UX fluida. Base de todos os micro-apps. | [react.dev](http://react.dev) |
| **Roteamento** | TanStack Router | Roteamento type-safe com inferência de tipos nas rotas e params. Suporte nativo a loaders de dados e search params tipados. Alternativa ao React Router com melhor DX. | [tanstack.com/router](http://tanstack.com/router) |
| **Estado Global** | Zustand | Gerenciamento de estado minimalista e Data-Driven. Stores como objetos imutáveis simples, sem boilerplate. Substituível por atom pattern (inspirado em Clojure). | [zustand docs](https://zustand.docs.pmnd.rs/) |
| **Validação / Schema** | Zod | Schemas de validação TypeScript-first. Define o shape dos dados de formulários e respostas de API. Integra com React Hook Form e gera types automaticamente — **princípio Data-Driven aplicado ao frontend.** | [zod.dev](http://zod.dev) |
| **Estilo** | TailwindCSS | Utility-first CSS. Estilos como dados (classes), sem CSS-in-JS overhead. Design tokens via `tailwind.config`. | [tailwindcss.com](http://tailwindcss.com) |
| **Testes Unitários** | Vitest | Test runner ultrarrápido, compatível com Jest API. Para funções utilitárias, hooks e lógica de estado. | [vitest.dev](http://vitest.dev) |
| **Testes de Componente** | Testing Library | Testa componentes React como o usuário os usa (queries por role/label, não por implementação). Paired com Vitest. | [testing-library.com](http://testing-library.com) |

### 3.5 Schema & Validação — Malli vs clojure.spec.alpha

> ⚠️ **Decisão pendente para o time.** Ambas as bibliotecas são adicionadas ao projeto; a escolha definitiva ocorre após spike técnico na primeira sprint.
| Critério | **Malli** *(atual primário)* | **clojure.spec.alpha** |
| --- | --- | --- |
| **Sintaxe** | Schemas como vetores/mapas EDN (`[:map [:name :string]]`) — totalmente data-driven | Macros (`s/def`, `s/keys`) — mais verboso, menos composável como dado |
| **Composição** | Alta — schemas são dados, podem ser transformados/gerados programaticamente | Média — specs são globais por namespace, mais difícil de compor dinamicamente |
| **Geração de dados** | `malli.generator` nativo com `test.check` | `clojure.spec.gen.alpha` nativo |
| **Performance** | Mais rápido para validação hot-path | Mais lento em validações frequentes |
| **Integração Pedestal** | Excelente via interceptors | Boa, mas requer adaptadores |
| **Filosofia do projeto** | ✅ Mais alinhado ao modelo Data-Driven do projeto | Menos alinhado — specs são efeitos colaterais globais |
| **Quando usar Spec Alpha** | Instrumentação de funções em desenvolvimento (REPL), generative testing de funções internas, interop com libs que consomem specs |  |

**Recomendação atual:** Malli como padrão de validação de inputs/outputs de API. clojure.spec.alpha como ferramenta de desenvolvimento (instrumentação via `stest/instrument`) e onde bibliotecas externas exijam specs.

---

### 3.6 Frontend — Alternativa: ClojureScript + Reagent + Reitit

> 💡 **Alternativa estratégica** caso o time tenha perfil Clojure forte e deseje máxima coesão de paradigma entre frontend e backend.

| Camada | Tecnologia | Para que serve |
| --- | --- | --- |
| **Linguagem** | ClojureScript | Clojure compilado para JavaScript. Mesma filosofia funcional/imutável do backend. Compartilhamento de código (cljc) com backend. |
| **UI Framework** | Reagent | Wrapper React em ClojureScript baseado em Hiccup (HTML como dados). Componentes são funções puras — máxima expressividade Data-Driven na UI. |
| **Roteamento** | Reitit (frontend) | Mesmo roteador usado no backend Clojure, com versão frontend. Rotas como dados EDN, composável e type-safe. |
| **Estado** | re-frame | Arquitetura Elm-like para ClojureScript. Events → Effects → State → View. Totalmente Data-Driven. |

**Trade-offs da alternativa:**

- ✅ Paradigma unificado front + back (Clojure/CLJS), compartilhamento de schemas
- ✅ Imutabilidade nativa, sem precisar de Zustand/Immer
- ❌ Ecossistema menor de componentes UI prontos vs React
- ❌ Curva de aprendizado maior para devs sem experiência em Clojure
- ❌ Ferramental de build mais complexo (shadow-cljs)

**Decisão para o MVP:** React + TypeScript (maior disponibilidade de squad e ecossistema). ClojureScript avaliado para v2 se o time crescer com perfil Clojure.

---

### 3.7 Infra & Plataforma
| Camada | Tecnologia | Para que serve | Doc |
| --- | --- | --- | --- |
| **Container** | Docker | Empacota cada serviço com suas dependências em imagens reproduzíveis. Base para dev local e CI/CD. | [docs.docker.com](http://docs.docker.com) |
| **Orquestração** | Kubernetes (K8s) | Gerencia containers em produção: scaling, self-healing, rolling deploys, HPA (auto-scaling horizontal). | [kubernetes.io](http://kubernetes.io) |
| **CI/CD** | GitHub Actions + ArgoCD | **GitHub Actions** executa build, testes e push de imagem a cada PR/merge. **ArgoCD** aplica o estado declarativo (GitOps) no cluster K8s — o Git é a fonte da verdade do que está em produção. | [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) |
| **Observabilidade** | OpenTelemetry + Grafana + Prometheus | **OTel** instrumenta traces e métricas automaticamente. **Prometheus** coleta e armazena métricas. **Grafana** visualiza dashboards e gerencia alertas. Stack padrão de mercado, vendor-neutral. | [opentelemetry.io](http://opentelemetry.io) |
| **Error Tracking** | Sentry | Captura exceções em runtime com stack trace, contexto de usuário (anonimizado conforme LGPD) e alertas por threshold de erros. | [sentry.io/docs](http://sentry.io/docs) |
| **Secrets** | AWS Secrets Manager | Armazena e rotaciona credenciais sensíveis (DB passwords, JWT keys, API keys de gateways). Integrado ao K8s via External Secrets Operator. | [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/) |
| **Object Storage** | AWS S3 | Armazena laudos, documentos de habilitação profissional e fotos de perfil. Acesso via presigned URLs com TTL. Bucket privado por padrão. | [S3 Docs](https://docs.aws.amazon.com/s3/) |

## 4. Decisões Arquiteturais (ADRs — MVP)

### ADR-001: Monolito Modular no MVP

- **Decisão:** Monolito modular com namespaces isolados por bounded context
- **Razão:** Velocidade de desenvolvimento, menor overhead operacional
- **Consequência:** Separar em microsserviços quando houver friction real (escala ou times distintos)

### ADR-002: PostgreSQL como store principal

- **Decisão:** PostgreSQL único para MVP
- **Razão:** ACID, JSONB para flex, suporte nativo a criptografia, equipe familiar
- **Consequência:** Avaliar DynamoDB para dados de alta escala de leitura (busca/catalog) em v2

### ADR-003: JWT com Refresh Token Rotation

- **Decisão:** Access token de curta duração (15min) + refresh token com rotation (7 dias)
- **Razão:** Balancear UX e segurança sem dependência de session store distribuído
- **Consequência:** Implementar blacklist de tokens invalidados em Redis

### ADR-004: Validação de CRM/CRO assíncrona

- **Decisão:** Profissional entra com status `PENDING_VERIFICATION`, admin aprova manualmente no MVP
- **Razão:** APIs de conselhos profissionais não são estáveis; automatação em v2
- **Consequência:** Gargalo operacional se volume crescer — priorizar automação antes de escalar onboarding

- 5. Entidades Principais do Modelo de Domínio

  ```
users
  id, type (patient|professional|admin),
  email, phone, password_hash,
  status (ACTIVE|INACTIVE|PENDING|SUSPENDED)

professionals
  id, user_id, full_name, cpf,
  registration_type (CRM|CRO|CREFITO|...),
  registration_number, registration_state,
  specialties[], service_modalities[],
  approval_status (PENDING_VERIFICATION|APPROVED|REJECTED|SUSPENDED),
  bank_account_info (encrypted)

patients
  id, user_id, full_name, cpf,
  birth_date, address (for home visits)

appointments
  id, patient_id, professional_id,
  modality, status (PENDING|CONFIRMED|COMPLETED|CANCELLED|NO_SHOW),
  scheduled_at, confirmed_at, completed_at

payments
  id, appointment_id, amount, status,
  gateway_transaction_id, split_config

reviews
  id, appointment_id, patient_id, professional_id,
  rating (1-5), comment, status (PENDING_MODERATION|PUBLISHED|REJECTED)
```
```
6. Fluxograma Técnico — Requisição End-to-End
```
Cliente → [TLS] → Pedestal Router
                          ↓
              [Interceptor: Auth JWT]
                          ↓
              [Interceptor: Rate Limit]
                          ↓
              [Interceptor: Audit Log]
                          ↓
              [Route Handler: ns/handler]
                          ↓
              [Service Layer: ns/service]
                          ↓
              [Repository Layer: ns/db]
                          ↓
                     PostgreSQL

```
🔗 [Inserir link do LucidChart — Diagrama de Arquitetura Completo]
## 7. Observabilidade — Requisitos Mínimos do MVP

- Tracing distribuído com OpenTelemetry em todas as requisições HTTP
- Métricas: latency p50/p95/p99, error rate, throughput por endpoint
- Alertas críticos: taxa de erro > 1%, latency p99 > 2s, falha de pagamento
- Sentry para captura de exceções com contexto de usuário (anonimizado conforme LGPD)
- Audit log imutável para todas as ações sobre dados sensíveis de saúde

