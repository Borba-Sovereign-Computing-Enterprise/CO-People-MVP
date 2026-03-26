# Especificação Técnica — Comunicação

> **Audiência:** Engenheiros
> 
> **Objetivo:** Schema de entidades, protocolo de mensageria em tempo real, endpoints REST, Kafka events e política de retenção.
> 

---

## 1. Decisão de Stack — Mensageria em Tempo Real

| Opção | Pros | Contras | Recomendação MVP |
| --- | --- | --- | --- |
| **WebSocket nativo** (via Pedestal) | Controle total, sem dependência externa, data-driven | Gerenciamento de conexões, reconnect, presense | ✅ MVP — implementação simples para monolito modular |
| [**Socket.io**](http://Socket.io) | Fallback automático, reconnect, rooms | Não idiomático em Clojure, overhead de JS | ❌ Overengineering para MVP |
| **Twilio Conversations / Stream** | Gerenciado, escalado, push nativo | Custo, vendor lock-in, latência | ❌ Avaliar em v2 se volume crescer |

**Decisão MVP:** WebSocket nativo com fallback para polling a cada 5s se WS não suportado.

---

## 2. Entidades e Atributos

### 2.1 `conversations`

| Atributo | Tipo | Constraints | Descrição |
| --- | --- | --- | --- |
| `id` | UUID | PK |  |
| `patient_id` | UUID | FK [users.id](http://users.id), NOT NULL |  |
| `professional_id` | UUID | FK [professionals.id](http://professionals.id), NOT NULL |  |
| `appointment_id` | UUID | FK [appointments.id](http://appointments.id), NULLABLE | Conversa vinculada a consulta específica |
| `status` | ENUM | NOT NULL, DEFAULT `ACTIVE` | `ACTIVE` \ |
| `ura_option` | VARCHAR(50) | NOT NULL | Opção escolhida na URA |
| `ura_description` | TEXT | NULLABLE | Descrição livre ("Outros assuntos") |
| `last_message_at` | TIMESTAMP | NULLABLE | Para ordenação na lista |
| `archived_reason` | TEXT | NULLABLE | Motivo do arquivamento |
| `created_at` | TIMESTAMP | NOT NULL |  |
| `updated_at` | TIMESTAMP | NOT NULL |  |

```sql
CREATE UNIQUE INDEX idx_conversation_participants
ON conversations (patient_id, professional_id)
WHERE status = 'ACTIVE';
-- Uma conversa ativa por par paciente-profissional
```
### 2.2 messages

| Atributo | Tipo | Constraints | Descrição |
| --- | --- | --- | --- |
| ``id`` | UUID | PK |  |
| ``conversation_id`` | UUID | FK, NOT NULL |  |
| ``sender_id`` | UUID | FK [users.id](https://users.id), NOT NULL |  |
| ``sender_type`` | ENUM | NOT NULL | ``patient`` \\ |
| ``content`` | TEXT | NULLABLE | Texto da mensagem (criptografado em repouso) |
| ``content_encrypted`` | BOOLEAN | NOT NULL, DEFAULT true |  |
| ``type`` | ENUM | NOT NULL, DEFAULT ``text`` | ``text`` \\ |
| ``file_s3_key`` | VARCHAR(500) | NULLABLE | Para mensagens tipo ``file`` |
| ``file_name`` | VARCHAR(255) | NULLABLE | Nome original do arquivo |
| ``file_mime_type`` | VARCHAR(100) | NULLABLE |  |
| ``file_size_bytes`` | INT | NULLABLE | Max: 10_485_760 (10MB) |
| ``deleted_at`` | TIMESTAMP | NULLABLE | Soft delete pelo remetente |
| ``deleted_by`` | UUID | NULLABLE |  |
| ``read_at`` | TIMESTAMP | NULLABLE | Quando o destinatário leu |
| ``delivered_at`` | TIMESTAMP | NULLABLE | Quando foi entregue |
| ``created_at`` | TIMESTAMP | NOT NULL |  |

### 2.3 message_audit_log — Imutável

| Atributo | Tipo | Descrição |
| --- | --- | --- |
| ``id`` | UUID | PK |
| ``message_id`` | UUID | FK [messages.id](https://messages.id) |
| ``action`` | ENUM | ``SENT`` \\ |
| ``actor_id`` | UUID | Quem agiu |
| ``metadata`` | JSONB | ip, user_agent, scan_result, etc. |
| ``created_at`` | TIMESTAMP | NOT NULL |


### 2.4 websocket_sessions

| Atributo | Tipo | Descrição |
| --- | --- | --- |
| ``id`` | UUID | PK |
| ``user_id`` | UUID | FK [users.id](https://users.id) |
| ``connection_id`` | VARCHAR(255) | ID da conexão WS |
| ``last_seen_at`` | TIMESTAMP | Heartbeat |
| ``connected_at`` | TIMESTAMP |  |
| ``disconnected_at`` | TIMESTAMP | NULLABLE |

## 3. Protocolo WebSocket
```
;; Endpoint: ws://api/v1/ws
;; Auth: Bearer token no header Sec-WebSocket-Protocol

;; Eventos do servidor para o cliente (server → client)
{:type "message.new"
 :payload {:message-id, :conversation-id, :sender-id,
           :content, :type, :created-at}}

{:type "message.delivered"
 :payload {:message-id, :delivered-at}}

{:type "message.read"
 :payload {:message-id, :read-at}}

{:type "conversation.archived"
 :payload {:conversation-id, :reason}}

;; Eventos do cliente para o servidor (client → server)
{:type "message.send"
 :payload {:conversation-id, :content, :type}}

{:type "message.read"
 :payload {:message-id}}

{:type "heartbeat"}
;; Servidor responde com {:type "heartbeat.ack"}
;; Se sem heartbeat por 60s → fechar conexão
```
## 4. Endpoints REST (fallback e operações não-realtime)
```
GET  /api/v1/conversations
     ?status=ACTIVE&page=1
     Response: { conversations[], unread_count_total }

GET  /api/v1/conversations/:id/messages
     ?before=<message_id>&limit=50   // cursor-based pagination
     Response: { messages[], has_more }

POST /api/v1/conversations/start
     Body: { professional_id, ura_option, ura_description? }
     Response 201: { conversation_id }
     Note: valida que existe consulta ativa/histórico com o profissional

POST /api/v1/conversations/:id/messages
     Body: { content, type: "text" }
     Response 201: { message_id, created_at }

POST /api/v1/conversations/:id/messages/file
     Content-Type: multipart/form-data
     File: attachment (PDF/JPEG/PNG, max 10MB)
     Response 201: { message_id, file_url }

DELETE /api/v1/messages/:id
     Auth: apenas o remetente, em até 5min após envio
     Response 200: { message_id, deleted_at }

PATCH /api/v1/messages/:id/read
     Response 200: { message_id, read_at }
```
## 5. Anti-virus Scan em Uploads
```
;; Pipeline de upload de arquivo
;; 1. Arquivo salvo no S3 com flag quarantine=true
;; 2. Job de scan (ClamAV ou AWS Macie)
;; 3. Se limpo: flag quarantine=false, mensagem entregue
;; 4. Se infectado: arquivo deletado, mensagem com erro genérico

(defn process-file-upload! [db s3 scanner file conversation-id sender-id]
  (let [s3-key (upload-to-quarantine! s3 file)
        scan   (scanner/scan s3-key)]
    (if (= :clean (:result scan))
      (do
        (s3/move-from-quarantine! s3 s3-key)
        (db/insert! db :messages
          {:conversation_id conversation-id
           :sender_id       sender-id
           :type            "file"
           :file_s3_key     s3-key
           :file_name       (:original-name file)
           :file_mime_type  (:content-type file)
           :file_size_bytes (:size file)}))
      (do
        (s3/delete! s3 s3-key)
        {:error :file-rejected-malicious}))))
```
## 6. Criptografia de Mensagens

```
;; Mensagens armazenadas com AES-256-GCM
;; Chave derivada do conversation_id + chave mestre do KMS

(defn encrypt-message [plaintext conversation-id kms]
  (let [data-key (kms/generate-data-key conversation-id)
        iv       (crypto/random-bytes 12)
        cipher   (crypto/aes-gcm-encrypt plaintext (:plaintext data-key) iv)]
    {:ciphertext      (base64/encode cipher)
     :encrypted-key   (base64/encode (:ciphertext data-key))
     :iv              (base64/encode iv)}))

;; Ao ler: descriptografar apenas se o user_id for participante da conversa
(defn decrypt-message [encrypted conversation-id user-id db kms]
  (when (participant? db conversation-id user-id)
    (let [data-key (kms/decrypt (:encrypted-key encrypted))
          plain    (crypto/aes-gcm-decrypt
                     (:ciphertext encrypted)

```
## 7. Kafka Events

| Evento | Tópico | Consumidores |
| --- | --- | --- |
| `MessageSent` | `communication.message_sent` | Notification (push), Audit |
| `ConversationArchived` | `communication.conversation_archived` | Notification |
| `FileRejected` | `communication.file_rejected` | Audit |

---

## 8. Job de Alerta de Não-Resposta

```clojure
;; Roda a cada hora
;; Alerta profissional se há mensagem do paciente sem resposta há > 48h

(defn alert-unresponsive-professionals! [db kafka]
  (let [stale (db/query db
                {:select [:c.professional_id
                          :c.id
                          [(sql/call :max :m.created_at) :last_patient_msg]]
                 :from   [[:messages :m]]
                 :join   [[:conversations :c] [:= :m.conversation_id :c.id]]
                 :where  [:and
                          [:= :m.sender_type "patient"]
                          [:is :m.deleted_at nil]
                          [:< :m.created_at (-> (Instant/now)
                                               (.minus 48 ChronoUnit/HOURS))]]
                 :group-by [:c.professional_id :c.id]
                 :having [:is
                          ;; sem mensagem do profissional após a última do paciente
                          (db/subquery
                            {:select [:id]
                             :from   [:messages]
                             :where  [:and
                                      [:= :conversation_id :c.id]
                                      [:= :sender_type "professional"]
                                      [:> :created_at :last_patient_msg]]})
                          nil]})
    (doseq [conv stale]
      (kafka/produce! kafka "communication.message_sent"
        {:type             "unresponded_alert"
         :professional_id  (:professional_id conv)
         :conversation_id  (:id conv)}))))
```

---

## 9. Observabilidade

- Métrica: `chat.messages_per_day` por tipo (text/file)
- Métrica: `chat.websocket.active_connections`
- Métrica: `chat.file.rejected_count` (malício + tamanho + formato)
- Métrica: `chat.unresponded_48h_count` — qualidade de engajamento do profissional
- Alerta: `chat.websocket.error_rate > 5%` por 5min
- Alerta: upload de arquivo falhando > 3x consecutivas (S3 indisponível)
