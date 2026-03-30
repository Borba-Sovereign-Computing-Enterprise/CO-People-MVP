# ⚙️ Especificação Técnica — Considerações Técnicas

> **Audiência:** Engenheiros, Tech Leads, Arquitetos
> 

> **Objetivo:** Documentação técnica das preocupações transversais: integrações externas, segurança, observabilidade e configuração de infra que permeiam todos os módulos.
> 

> ℹ️ Para a stack completa, ADRs e bounded contexts, consulte a **⚙️ Especificação Técnica — Visão Geral** (sub-page de Visão Geral do Produto).
> 

---

## 1. Integrações Externas — Especificação

### 1.1 Gateway de Pagamento

| Item | Detalhe |
| --- | --- |
| **Opções** | Stripe (preferência técnica), [Pagar.me](http://Pagar.me), PagSeguro |
| **Modelo** | Split Payment nativo: a plataforma recebe 100% e repassa ao profissional menos a taxa |
| **Webhook** | Gateway notifica a plataforma de pagamentos, reembolsos e falhas via POST em endpoint dedicado |
| **Idempotência** | Toda requisição ao gateway usa `idempotency_key = payment_id` para evitar dupla cobrança |
| **Outbox Pattern** | Eventos de pagamento publicados no Kafka via Outbox — garante at-least-once sem perda |
| **Endpoint interno** | `POST /webhooks/payment-gateway` — autenticado por signature HMAC do gateway |

### 1.2 E-mail Transacional

| Item | Detalhe |
| --- | --- |
| **Opções** | SendGrid (preferência), AWS SES |
| **Uso** | Confirmação de email, recuperação de senha, confirmação de agendamento, lembretes, notificação de aprovação |
| **Template** | Templates gerenciados no provider; apenas variáveis dinâmicas são enviadas pelo backend |
| **Retry** | Máximo 3 tentativas com exponential backoff em caso de falha de entrega |

### 1.3 SMS / 2FA

| Item | Detalhe |
| --- | --- |
| **Opções** | Twilio (preferência), Zenvia |
| **Uso** | Códigos OTP para 2FA, lembretes de consulta via SMS |
| **OTP** | 6 dígitos, expira em 5 minutos, máximo 3 tentativas antes de bloquear |

### 1.4 Object Storage (AWS S3)

| Tipo de arquivo | Bucket | Acesso | TTL da URL |
| --- | --- | --- | --- |
| Fotos de perfil | `plataforma-saude-profiles` | Público via CDN | N/A |
| Documentos de habilitação | `plataforma-saude-docs-private` | Presigned URL | 1 hora |
| Laudos e documentos médicos | `plataforma-saude-health-docs` | Presigned URL | 30 minutos |

Fluxo de upload:

```
1. POST /api/v1/upload/presigned-url { file_type, content_type }
2. Backend gera presigned PUT URL no S3 (expira em 5min)
3. Client faz PUT direto no S3
4. Client confirma: POST /api/v1/upload/confirm { s3_key, document_type }
5. Backend valida e registra no DB
```

### 1.5 Validação de CRM/CRO

| Item | Detalhe |
| --- | --- |
| **MVP** | Manual pelo admin — admin acessa site do CFM/CRO e verifica manualmente |
| **v2** | Job assíncrono que consulta API do CFM via scraping ou API oficial quando disponível |
| **Fallback** | Se API indisponível, mantém status `PENDING_VERIFICATION` e notifica admin |

---

## 2. Configuração por Environment (Aero + EDN)

```clojure
;; config/config.edn
{:db         #profile {:dev  {:url "jdbc:postgresql://localhost/plataforma_dev"}
                       :prod {:url #env DB_URL}}
 :redis      {:url #profile {:dev "redis://localhost:6379"
                             :prod #env REDIS_URL}}
 :jwt        {:secret       #env JWT_PRIVATE_KEY
              :access-ttl   900
              :refresh-ttl  604800}
 :s3         {:region       #env AWS_REGION
              :profile-bucket    "plataforma-saude-profiles"
              :docs-bucket       "plataforma-saude-docs-private"
              :health-docs-bucket "plataforma-saude-health-docs"}
 :kafka      {:bootstrap-servers #env KAFKA_BROKERS
              :topics {:professional-approved  "professional.approved"
                       :appointment-completed  "appointment.completed"
                       :payment-processed      "payment.processed"}}
 :payment    {:provider     #env PAYMENT_PROVIDER  ;; :stripe | :pagarme
              :api-key      #env PAYMENT_API_KEY
              :webhook-secret #env PAYMENT_WEBHOOK_SECRET}
 :email      {:provider     #env EMAIL_PROVIDER    ;; :sendgrid | :ses
              :api-key      #env EMAIL_API_KEY
              :from-address "no-reply@plataforma-saude.com.br"}
 :sms        {:provider     #env SMS_PROVIDER      ;; :twilio | :zenvia
              :api-key      #env SMS_API_KEY}}
```

---

## 3. Segurança Cross-Cutting

### Criptografia em repouso

```clojure
;; Campos criptografados com AES-256-GCM
;; Chave gerenciada pelo AWS KMS
(def encrypted-fields
  {:patients     [:cpf]
   :professionals [:cpf :bank-account-data :two-factor-secret]
   :documents    [:s3-key]}) ;; path do S3 também encriptado
```

### Headers de segurança (Pedestal)

```clojure
(def security-headers-interceptor
  {:name  ::security-headers
   :leave (fn [ctx]
            (update-in ctx [:response :headers] merge
              {"Strict-Transport-Security"  "max-age=31536000; includeSubDomains"
               "X-Content-Type-Options"     "nosniff"
               "X-Frame-Options"            "DENY"
               "Content-Security-Policy"    "default-src 'self'"
               "Referrer-Policy"            "strict-origin-when-cross-origin"}))})
```

### Rate Limiting

| Endpoint | Limite | Janela |
| --- | --- | --- |
| `POST /auth/login` | 5 tentativas | por IP, janela de 15min |
| `POST /auth/register` | 3 cadastros | por IP, janela de 1h |
| `POST /auth/forgot-password` | 3 solicitações | por email, janela de 1h |
| Endpoints gerais | 100 req | por usuário autenticado, janela de 1min |

---

## 4. Observabilidade — Setup Completo

### OpenTelemetry Instrumentation

```clojure
;; Todas as rotas Pedestal são instrumentadas automaticamente
;; via interceptor de tracing

(def tracing-interceptor
  {:name  ::otel-tracing
   :enter (fn [ctx]
            (let [span (tracer/start-span (get-route-name ctx))]
              (assoc ctx :otel/span span)))
   :leave (fn [ctx]
            (tracer/end-span (:otel/span ctx))
            ctx)
   :error (fn [ctx err]
            (tracer/record-error (:otel/span ctx) err)
            ctx)})
```

### Métricas mandatoriamente coletadas

| Métrica | Tipo | Labels |
| --- | --- | --- |
| `http.request.duration` | Histogram | route, method, status_code |
| `auth.login.attempts` | Counter | result (success/failure), type (patient/professional) |
| `appointment.created` | Counter | modality |
| `payment.processed` | Counter | gateway, status |
| `professional.approval.queue.size` | Gauge | — |
| `review.moderation.queue.size` | Gauge | — |

### Alertas críticos (AlertManager)

| Alerta | Condição | Severidade |
| --- | --- | --- |
| Alta taxa de erro | `error_rate > 1%` por 5min | CRITICAL |
| Latency degradada | `p99 > 2s` por 5min | WARNING |
| Falha de pagamento | `payment.failed_rate > 5%` | CRITICAL |
| Fila de aprovação acumulando | `approval.queue > 20 items` por 1h | WARNING |
| Token JWT inválido em alta frequência | `auth.invalid_token > 50/min` | WARNING |

---

## 5. CI/CD Pipeline

```
PR aberto
    │
    ▼
[GitHub Actions]
  1. Lint (clj-kondo)
  2. Testes unitários (clojure.test)
  3. Testes de integração (state-flow)
  4. Build Docker image
  5. Push para ECR (somente main)
    │
    ▼ (merge em main)
[ArgoCD]
  6. Detecta nova imagem no registry
  7. Aplica manifests K8s (GitOps)
  8. Rolling deploy (zero downtime)
  9. Health check automático
```

---

## 6. Decisões Pendentes

| Decisão | Prazo | Responsável |
| --- | --- | --- |
| Gateway de pagamento: Stripe vs [Pagar.me](http://Pagar.me) | Antes do M5 | Tech Lead + PM |
| Malli vs clojure.spec.alpha como padrão | Sprint 1 | Tech Lead |
| Provedor de SMS: Twilio vs Zenvia | Antes do M7 | Tech Lead |
| Plano de CDN para fotos de perfil | Sprint 2 | Infra |
| Estrategia de multi-tenancy futura | Pós-MVP | Arquiteto |
