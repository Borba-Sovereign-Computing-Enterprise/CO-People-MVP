# ☁️ Infra — Plataforma & Operações

## Banco de Dados
| Recurso | Tabelas |
| --- | --- |
| **PostgreSQL** | `admin_users`, `audit_logs` (append-only), `professional_review_queue`, `moderation_queue`, `platform_configs`, `consent_purposes` |
| **AWS KMS** | Chaves mestres AES-256-GCM (CPF, dados bancários, mensagens) |

## Object Storage

| Bucket | Conteúdo |
| --- | --- |
| `plataforma-saude-reports` | Relatórios exportados (acesso via presigned URL 72h) |

## Observabilidade & Monitoramento

| Recurso | Uso |
| --- | --- |
| **Grafana** | Dashboards de métricas (latency, error rate, throughput) |
| **Prometheus** | Coleta e armazenamento de métricas (OpenTelemetry) |
| **Sentry** | Captura de exceções com contexto anonimizado |
| **AlertManager** | Alertas críticos por threshold |

## Kubernetes (K8s)

| Recurso | Config |
| --- | --- |
| Deployments | Um deployment por microsserviço |
| ArgoCD (GitOps) | Estado declarativo — Git como fonte de verdade |
| HPA | Auto-scaling baseado em CPU e memória |
| External Secrets Operator | Sincroniza segredos do AWS Secrets Manager |
