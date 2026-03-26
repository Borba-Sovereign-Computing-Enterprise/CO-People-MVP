# Especificação Técnica — Notificações

> **Audiência:** Engenheiros
> 

> **Objetivo:** Schema, pipeline de disparo Kafka-driven, retry com backoff, janela de silêncio, endpoints e observabilidade.
> 

---

## 1. Entidades e Atributos

### 1.1 `notification_preferences` — Preferências por usuário

| Atributo | Tipo | Constraints | Descrição |
| --- | --- | --- | --- |
| `user_id` | UUID | PK, FK [users.id](http://users.id) |  |
| `push_enabled` | BOOLEAN | NOT NULL, DEFAULT true |  |
| `email_enabled` | BOOLEAN | NOT NULL, DEFAULT true |  |
| `sms_enabled` | BOOLEAN | NOT NULL, DEFAULT true |  |
| `scheduling_push` | BOOLEAN | NOT NULL, DEFAULT true |  |
| `scheduling_email` | BOOLEAN | NOT NULL, DEFAULT true |  |
| `chat_push` | BOOLEAN | NOT NULL, DEFAULT true |  |
| `reviews_push` | BOOLEAN | NOT NULL, DEFAULT true |  |
| `marketing_email` | BOOLEAN | NOT NULL, DEFAULT false |  |
| `quiet_hours_start` | TIME | NOT NULL, DEFAULT '22:00' |  |
| `quiet_hours_end` | TIME | NOT NULL, DEFAULT '07:00' |  |
| `timezone` | VARCHAR(50) | NOT NULL, DEFAULT 'America/Sao_Paulo' |  |
| `updated_at` | TIMESTAMP | NOT NULL |  |

### 1.2 `push_tokens`

| Atributo | Tipo | Constraints | Descrição |
| --- | --- | --- | --- |
| `id` | UUID | PK |  |
| `user_id` | UUID | FK [users.id](http://users.id), NOT NULL |  |
| `token` | VARCHAR(500) | NOT NULL | Token FCM/OneSignal |
| `platform` | ENUM | NOT NULL | `web` \ |
| `provider` | ENUM | NOT NULL | `fcm` \ |
| `is_active` | BOOLEAN | NOT NULL, DEFAULT true |  |
| `registered_at` | TIMESTAMP | NOT NULL |  |
| `last_used_at` | TIMESTAMP | NULLABLE |  |
| `deactivated_at` | TIMESTAMP | NULLABLE |  |

### 1.3 `notifications`

| Atributo | Tipo | Constraints | Descrição |
| --- | --- | --- | --- |
| `id` | UUID | PK |  |
| `user_id` | UUID | FK [users.id](http://users.id), NOT NULL |  |
| `type` | VARCHAR(100) | NOT NULL | Ex: `appointment.confirmed` |
| `title` | VARCHAR(255) | NOT NULL | Título do push/email |
| `body` | TEXT | NOT NULL | Conteúdo |
| `channel` | ENUM | NOT NULL | `push` \ |
| `status` | ENUM | NOT NULL, DEFAULT `PENDING` | `PENDING` \ |
| `reference_type` | VARCHAR(50) | NULLABLE | `appointment`, `payment`, etc. |
| `reference_id` | UUID | NULLABLE | ID do objeto relacionado |
| `attempts` | SMALLINT | NOT NULL, DEFAULT 0 |  |
| `last_attempt_at` | TIMESTAMP | NULLABLE |  |
| `next_retry_at` | TIMESTAMP | NULLABLE |  |
| `sent_at` | TIMESTAMP | NULLABLE |  |
| `failure_reason` | TEXT | NULLABLE |  |
| `created_at` | TIMESTAMP | NOT NULL |  |

---

## 2. Pipeline de Disparo — Kafka Consumer

```
Kafka Event
    │ (ex: appointment.confirmed)
    ▼
[notification-consumer]
    │
    ▼
[1] Identifica destinatários (paciente + profissional)
    │
    ▼
[2] Carrega preferências do usuário (notification_preferences)
    │
    ▼
[3] Verifica janela de silêncio (quiet hours + timezone)
    │
    ├── Dentro da janela → schedula para fim da janela (09h)
    │
    ▼
[4] Seleciona canal (push → email → sms → in-app, nessa prioridade)
    │
    ▼
[5] Cria registro em `notifications` (status: PENDING)
    │
    ▼
[6] Envia via provider
    │
    ├── Sucesso → status: SENT
    └── Falha   → status: FAILED + calcula next_retry_at
```

---

## 3. Retry com Exponential Backoff

```clojure
(def retry-delays-minutes [1 5 15])

(defn schedule-retry! [db notification-id attempt]
  (when (< attempt (count retry-delays-minutes))
    (let [delay-min (nth retry-delays-minutes attempt)
          retry-at  (-> (Instant/now)
                        (.plusSeconds (* delay-min 60)))]
      (db/execute! db
        {:update :notifications
         :set    {:status        "PENDING"
                  :next_retry_at retry-at
                  :attempts      (inc attempt)}
         :where  [:= :id notification-id]}))))

;; Job de retry (roda a cada minuto)
(defn process-retries! [db providers]
  (let [due (db/query db
               {:select [:*]
                :from   [:notifications]
                :where  [:and
                         [:= :status "FAILED"]
                         [:< :attempts 3]
                         [:<= :next_retry_at (Instant/now)]]})]
    (doseq [n due]
      (dispatch-notification! db providers n))))
```

---

## 4. Janela de Silêncio

```clojure
(defn in-quiet-hours? [user-prefs]
  (let [tz    (ZoneId/of (:timezone user-prefs))
        now   (.atZone (Instant/now) tz)
        time  (.toLocalTime now)
        start (LocalTime/parse (:quiet_hours_start user-prefs))
        end   (LocalTime/parse (:quiet_hours_end user-prefs))]
    ;; Janela pode cruzar meia-noite (22:00 - 07:00)
    (if (.isAfter start end)
      (or (.isAfter time start) (.isBefore time end))
      (and (.isAfter time start) (.isBefore time end)))))

;; Exceções que ignoram a janela
(def bypass-quiet-hours
  #{"auth.2fa_code"
    "appointment.cancelled_urgent"  ;; cancelamento < 60min
    "payment.failed"
    "security.suspicious_login"})
```

---

## 5. Provedores e Abstração

```clojure
;; Protocol - qualquer provider implementa isso
(defprotocol NotificationProvider
  (send-push!   [this user-id title body data])
  (send-email!  [this to subject html-body])
  (send-sms!    [this phone message]))

;; FCM implementation
(defrecord FCMProvider [api-key]
  NotificationProvider
  (send-push! [this user-id title body data]
    (let [tokens (get-active-tokens user-id)]
      (doseq [token tokens]
        (http/post "https://fcm.googleapis.com/fcm/send"
          {:headers {"Authorization" (str "key=" api-key)}
           :body    (json/encode
                     {:to           (:token token)
                      :notification {:title title :body body}
                      :data         data})})))))

;; Seleciona provider via config EDN
;; {:notification {:push-provider :fcm | :onesignal
;;                 :email-provider :sendgrid | :ses
;;                 :sms-provider :twilio | :zenvia}}
```

---

## 6. Endpoints REST

```
GET /api/v1/me/notifications
    ?page=1&status=unread
    Response: { notifications[], unread_count, pagination }

PATCH /api/v1/me/notifications/:id/read
    Response: 204

PATCH /api/v1/me/notifications/read-all
    Response: 204

GET  /api/v1/me/notification-preferences
    Response: { preferences }

PUT  /api/v1/me/notification-preferences
    Body: { push_enabled, scheduling_push, chat_push,
            reviews_push, marketing_email,
            quiet_hours_start, quiet_hours_end, timezone }
    Response: 200

POST /api/v1/me/push-tokens
    Body: { token, platform, provider }
    Response: 201

DELETE /api/v1/me/push-tokens/:token
    Response: 204
```

---

## 7. Agendamento de Lembretes

```clojure
;; Kafka consumer de appointment.confirmed agenda os lembretes
(defn schedule-reminders! [db appointment]
  (let [scheduled-at (:scheduled-at appointment)
        h24-before   (.minus scheduled-at 24 ChronoUnit/HOURS)
        h1-before    (.minus scheduled-at 1  ChronoUnit/HOURS)]

    ;; Persiste lembretes agendados
    (db/insert-multi! db :scheduled_notifications
      [{:appointment_id (:id appointment)
        :type          "appointment.reminder_24h"
        :send_at       h24-before}
       {:appointment_id (:id appointment)
        :type          "appointment.reminder_1h"
        :send_at       h1-before}])
  ))

;; Job de disparo (roda a cada 5 minutos)
;; Busca scheduled_notifications onde send_at <= NOW
;; e appointment.status = CONFIRMED
```

---

## 8. Observabilidade

- Métrica: `notification.sent_count` por tipo e canal
- Métrica: `notification.failed_rate` por canal e provider
- Métrica: `notification.retry_count`
- Métrica: `push_token.invalid_rate` (tokens expirados/inválidos)
- Alerta: `email.failed_rate > 5%` — provider com problema
- Alerta: `sms.failed_rate > 10%` — pode afetar 2FA
- Alerta: lag de disparo > 5min para eventos críticos (cancelamento, pagamento)
