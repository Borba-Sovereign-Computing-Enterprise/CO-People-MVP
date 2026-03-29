# Backend — Descoberta & Busca

## Microsserviços

| Microsserviço | Responsabilidade |
| --- | --- |
| `search-service` | Full-text search (PostgreSQL tsvector), geosearch, algoritmo de relevância ponderado, cache Redis |
| `catalog-service` | CRUD de especialidades, catálogo, invalidar MV e cache ao mudar dados |
