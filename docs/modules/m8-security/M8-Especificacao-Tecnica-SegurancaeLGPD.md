# Especificação Técnica — Segurança e LGPD

> **Audiência:** Engenheiros, Tech Leads
> 

> **Objetivo:** RBAC, criptografia por campo, schemas de consentimento, fluxos de exclusão/portabilidade e enforcement de auditoria.
> 

---

## 1. RBAC — Papéis e Permissões

```clojure
;; Definido como dado puro em EDN — nunca hardcoded
;; config/rbac.edn

{:roles
  {:patient
    {:can [:appointment/read-own :appointment/create :payment/read-own
           :profile/read-public :profile/update-own
           :chat/read-own :chat/send :document/read-own
           :review/create-own :notification/read-own]}

   :professional
    {:can [:appointment/read-own :appointment/manage-own
           :profile/update-own :profile/read-public
           :chat/read-own :chat/send :document/read-consultation
           :review/reply-own :availability/manage
           :payment/read-own :notification/read-own]}

   :admin
    {:can [:user/read-all :user/suspend :user/delete
           :professional/approve :professional/reject :professional/suspend
           :review/moderate :payment/configure
           :specialty/manage :audit-log/read
           :report/read]}

   :super-admin
    {:can :all  ;; inclui tudo acima + config crítica
     :requires-2fa true
     :session-timeout-minutes 30}}}
```

### Interceptor de Autorização

```clojure
(defn authorization-interceptor [required-permission]
  {:name  ::authorization
   :enter (fn [ctx]
            (let [user-type (get-in ctx [:request :identity :type])
                  allowed?  (has-permission? user-type required-permission)]
              (if allowed?
                ctx
                (assoc ctx :response
                  {:status 403
                   :body   {:error "Insufficient permissions"
                            :required required-permission}}))))}) 

;; Uso nas rotas
(defn routes []
  [[["/api/v1"
     ["/admin/professionals/:id/approve"
      :patch [(authorization-interceptor :professional/approve)
              admin-interceptor]
      :route-name ::approve-professional]]
    ["/api/v1/me/profile"
     :put  [(authorization-interceptor :profile/update-own)
             auth-interceptor]
     :route-name ::update-profile]]]))
```

---

## 2. Criptografia por Campo

```clojure
;; Campos criptografados com AES-256-GCM via AWS KMS
;; Chave mestre no KMS, data key derivada por registro

(def encrypted-fields-map
  {:patients      #{:cpf}
   :professionals #{:cpf :bank_account_data :two_factor_secret}
   :messages      #{:content}
   :documents     #{:s3_key}})

;; Middleware de repositório: transparente para o service layer
(defn encrypt-fields [table record kms]
  (let [fields (get encrypted-fields-map table #{})]
    (reduce (fn [acc field]
              (if (contains? acc field)
                (update acc field #(kms/encrypt kms (str %)))
                acc))
            record fields)))

(defn decrypt-fields [table record kms]
  (let [fields (get encrypted-fields-map table #{})]
    (reduce (fn [acc field]
              (if (contains? acc field)
                (update acc field #(kms/decrypt kms %))
                acc))
            record fields)))
```

---

## 3. Schema de Consentimento LGPD

### 3.1 `consent_purposes` — Catálogo de finalidades

| Atributo | Tipo | Descrição |
| --- | --- | --- |
| `id` | UUID | PK |
| `code` | VARCHAR(100) | Ex: `health_data_processing`, `marketing_email` |
| `name` | VARCHAR(255) | Exibido ao usuário |
| `description` | TEXT | Descrição clara e não jurídica |
| `legal_basis` | VARCHAR(100) | Art. da LGPD que fundamenta |
| `is_mandatory` | BOOLEAN | Se recusar bloqueia o uso |
| `version` | SMALLINT | Versão da finalidade |
| `active_from` | DATE |  |

### 3.2 `user_consents` — Consentimento por usuário

| Atributo | Tipo | Constraints | Descrição |
| --- | --- | --- | --- |
| `id` | UUID | PK |  |
| `user_id` | UUID | FK [users.id](http://users.id), NOT NULL |  |
| `purpose_id` | UUID | FK consent_[purposes.id](http://purposes.id), NOT NULL |  |
| `version` | SMALLINT | NOT NULL | Versão do documento aceito |
| `granted` | BOOLEAN | NOT NULL | Aceitou ou recusou |
| `granted_at` | TIMESTAMP | NOT NULL |  |
| `ip_address` | INET | NOT NULL |  |
| `user_agent` | TEXT | NOT NULL |  |
| `revoked_at` | TIMESTAMP | NULLABLE | Quando revogou |

---

## 4. Fluxo de Exclusão e Anonimização

```clojure
;; LGPD Art. 18, IV: exclusão anonimiza — não deleta
;; Dados de saúde retidos pelo prazo legal (CFM: 20 anos para prontuários)

(defn anonymize-user! [db kms user-id actor-id]
  (db/with-transaction [tx db]
    ;; 1. Cancela agendamentos futuros
    (cancel-future-appointments! tx user-id)

    ;; 2. Anonimiza dados pessoais identificáveis
    (db/execute! tx
      {:update :users
       :set    {:email      (str "deleted_" (random-uuid) "@anon.invalid")
                :phone      "000000000"
                :status     "DELETED"
                :deleted_at (Instant/now)}
       :where  [:= :id user-id]})

    ;; 3. Anonimiza dados do perfil (paciente ou profissional)
    (anonymize-profile! tx user-id)

    ;; 4. Revoga tokens ativos
    (db/execute! tx
      {:update :refresh_tokens
       :set    {:revoked_at (Instant/now)}
       :where  [:= :user_id user-id]})

    ;; 5. Registra no audit log
    (db/insert! tx :audit_logs
      {:actor_id    actor-id
       :action      "USER_DATA_ANONYMIZED"
       :target_type "user"
       :target_id   user-id
       :metadata    {:reason "user_request" :timestamp (Instant/now)}})))

;; Dados de saúde não são anonimizados — apenas o vínculo com a identidade
;; appointments, documents, messages ficam com user_id referenciando a conta anonimizada
```

---

## 5. Fluxo de Portabilidade (Art. 18, V)

```clojure
;; Exporta todos os dados do usuário em JSON estruturado
;; Job assíncrono — link de download enviado por email em até 15 dias

(defn export-user-data! [db s3 user-id]
  (let [data {:user          (get-user-data db user-id)
              :profile       (get-profile-data db user-id)
              :appointments  (get-appointments db user-id)
              :payments      (get-payments db user-id)
              :reviews       (get-reviews db user-id)
              :consents      (get-consents db user-id)
              :notifications (get-notification-history db user-id)
              :exported_at   (Instant/now)}
        json-str (json/encode data {:pretty true})
        s3-key   (str "exports/" user-id "/" (random-uuid) ".json")]
    (s3/put! s3 s3-key json-str {:content-type "application/json"})
    (s3/presigned-url s3 s3-key {:expires-hours 72})))
```

---

## 6. Auditoria Imutável — Garantia Técnica

```sql
-- A tabela audit_logs é protegida por role de banco de dados
-- A role da aplicação tem apenas INSERT, nunca UPDATE/DELETE

CREATE ROLE app_write;
GRANT SELECT, INSERT ON audit_logs TO app_write;
-- UPDATE e DELETE não concedidos

-- Apenas a role admin_dba tem UPDATE (somente para operações de backup)
-- e nunca é usada pela aplicação
```

---

## 7. Security Headers e Hardening

```clojure
(def security-headers
  {"Strict-Transport-Security"   "max-age=31536000; includeSubDomains; preload"
   "X-Content-Type-Options"      "nosniff"
   "X-Frame-Options"             "DENY"
   "Content-Security-Policy"     "default-src 'self'; img-src 'self' data: https:"
   "Referrer-Policy"             "strict-origin-when-cross-origin"
   "Permissions-Policy"          "camera=(), microphone=(), geolocation=(self)"
   "X-XSS-Protection"            "1; mode=block"})
```

---

## 8. Configuração de Backup

```clojure
;; config/config.edn
{:backup
  {:schedule          "0 3 * * *"        ;; 3am diário
   :retention-days    90
   :encryption-key    #env BACKUP_KMS_KEY ;; chave separada da produção
   :storage-bucket    "plataforma-saude-backups"
   :rto-hours         4                  ;; Recovery Time Objective
   :rpo-hours         24}}               ;; Recovery Point Objective
```

---

## 9. Observabilidade de Segurança

- Alerta: `auth.login_failed_rate > 20/min` por IP — possível brute force
- Alerta: token JWT rev revogado sendo reutilizado (token theft)
- Alerta: acesso a dados clínicos sem `audit_log` correspondente
- Alerta: 3+ tentativas de acesso negado ao painel admin em 5min
- Métrica: `consent.revocations_per_day` — aumento indica problema de confiança
- Dashboard: mapa de acessos por IP (anomalias geográficas)
