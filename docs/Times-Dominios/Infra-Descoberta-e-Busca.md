# Infra — Descoberta & Busca

## Banco de Dados

| Recurso | Descrição |
| --- | --- |
| **PostgreSQL** | Materialized View `professional_search` com índices GIN (full-text) e GiST (geosearch) |
| **Redis** | Cache de resultados de busca (TTL 2min), flush via consumer Kafka |

## Kafka

| Papel | Tópicos |
| --- | --- |
| Consumer | `professional.approved`, `professional.suspended`, `professional.profile_updated`, `review.published` |

## Observabilidade

| Métrica | Alerta |
| --- | --- |
| `search.results.zero_rate` | > 30% de buscas retornando 0 resultados |
| `search.query.p99` | > 500ms |
| `search.cache.hit_rate` | < 60% |
