# 🔧 Especificação Técnica — Busca e Categorias

> **Audiência:** Engenheiros  
> **Objetivo:** Schemas de busca, indexação, full-text search, geosearch, endpoints e algoritmo de relevância.

---

## 1. Entidades e Atributos

### 1.1 `specialties` — Catálogo de especialidades (já definido em M2)

| Atributo       | Tipo         | Constraints                  |
|----------------|--------------|------------------------------|
| `id`           | UUID         | PK                           |
| `name`         | VARCHAR(100) | UNIQUE, NOT NULL             |
| `council_type` | ENUM         | `CRM`                        |
| `slug`         | VARCHAR(100) | UNIQUE, NOT NULL             |
| `is_active`    | BOOLEAN      | NOT NULL, DEFAULT true       |
| `display_order`| SMALLINT     | NOT NULL, DEFAULT 0          |
| `icon_url`     | VARCHAR(500) | NULLABLE                     |

---

### 1.2 `search_index` — View materializada para busca

```sql
CREATE MATERIALIZED VIEW professional_search AS
SELECT
  p.id,
  p.full_name,
  pp.profile_photo_url,
  pp.average_rating,
  pp.total_reviews,
  pp.consultation_price_presential,
  pp.consultation_price_home_visit,
  pp.accepts_new_appointments,
  pp.home_visit_radius_km,
  p.service_modalities,
  p.approval_status,
  pp.is_profile_complete,
  ARRAY_AGG(DISTINCT s.id)   AS specialty_ids,
  ARRAY_AGG(DISTINCT s.name) AS specialty_names,
  ARRAY_AGG(DISTINCT s.slug) AS specialty_slugs,
  oa.city,
  oa.state,
  oa.lat,
  oa.lng,
  TO_TSVECTOR('portuguese',
    p.full_name || ' ' ||
    COALESCE(pp.bio, '') || ' ' ||
    STRING_AGG(DISTINCT s.name, ' ')
  ) AS search_vector
FROM professionals p
JOIN professional_profiles pp ON pp.id = p.id
LEFT JOIN professional_specialties ps ON ps.professional_id = p.id
LEFT JOIN specialties s ON s.id = ps.specialty_id
LEFT JOIN office_addresses oa ON oa.professional_id = p.id AND oa.is_primary = true
WHERE p.approval_status = 'APPROVED'
  AND pp.is_profile_complete = true
  AND p.deleted_at IS NULL
GROUP BY p.id, pp.id, oa.city, oa.state, oa.lat, oa.lng;

-- Índices
CREATE INDEX idx_search_specialty_ids  ON professional_search USING GIN (specialty_ids);
CREATE INDEX idx_search_vector         ON professional_search USING GIN (search_vector);
CREATE INDEX idx_search_location       ON professional_search USING GIST (POINT(lng, lat));
CREATE INDEX idx_search_rating         ON professional_search (average_rating DESC);
CREATE INDEX idx_search_price_p        ON professional_search (consultation_price_presential);
CREATE INDEX idx_search_price_hv       ON professional_search (consultation_price_home_visit);

Refresh da view: via Kafka ou job agendado a cada 5 minutos.

(defn refresh-search-index! [db]
  (db/execute! db ["REFRESH MATERIALIZED VIEW CONCURRENTLY professional_search"]))

2. Algoritmo de Relevância

(defn relevance-score
  [{:keys [average-rating total-reviews distance-km accepts-new price-delta]}]
  (let [rating-score   (* 40 (/ average-rating 5.0))
        reviews-score  (* 20 (min 1.0 (/ total-reviews 50)))
        distance-score (* 25 (- 1.0 (min 1.0 (/ distance-km 50))))
        avail-score    (* 10 (if accepts-new 1.0 0.0))
        price-score    (* 5  (- 1.0 (min 1.0 (/ price-delta 500))))]
    (+ rating-score reviews-score distance-score avail-score price-score)))

Pesos (100 pontos):

40 pts: nota média

20 pts: volume de avaliações

25 pts: proximidade geográfica

10 pts: agenda aberta

5 pts: preço relativo
```

3. Endpoints REST
3.1 Busca principal
```
GET /api/v1/search/professionals

Query params:

q string — busca por nome ou especialidade

specialty UUID

city string

lat, lng float

radius_km int (default: 25)

modality enum (presential | home_visit | both)

price_min, price_max decimal

rating_min float

date date (YYYY-MM-DD)

page, per_page (default: 20, max: 50)

sort enum (relevance | price_asc | rating | distance)

Response 200:
{
  "professionals": [
    {
      "id": "...",
      "full_name": "...",
      "profile_photo_url": "...",
      "primary_specialty": "...",
      "specialty_names": [],
      "average_rating": 4.8,
      "total_reviews": 120,
      "service_modalities": ["presential"],
      "consultation_price_presential": 200,
      "consultation_price_home_visit": 250,
      "accepts_new_appointments": true,
      "city": "Recife",
      "distance_km": 3.2,
      "next_available_slot": "2026-03-28T14:00:00"
    }
  ],
  "pagination": { "total": 1000, "page": 1, "per_page": 20, "total_pages": 50 }
}
```
3.2 Catálogo de especialidades
```
GET /api/v1/specialties
Query: ?council_type=CRM&active=true
Response 200: {
  specialties: [
    { id, name, slug, council_type, icon_url }
  ]
}
```
3.3 Admin — gestão de especialidades
```
POST /api/v1/admin/specialties
Body: { name, council_type, slug, icon_url, display_order }
Response 201: { id, name, slug }

PATCH /api/v1/admin/specialties/:id
Body: { name?, icon_url?, is_active?, display_order? }
Response 200: { updated specialty }

DELETE /api/v1/admin/specialties/:id
Note: Soft deactivate (is_active = false), não deleta
```

4. Busca Geográfica — PostGIS
```

-- Busca profissionais dentro de raio com ordenação por distância
SELECT *,
  POINT(lng, lat) <-> POINT(:user_lng, :user_lat) AS distance_km
FROM professional_search
WHERE
  POINT(lng, lat) <-> POINT(:user_lng, :user_lat) <= :radius_km
  AND :modality = ANY(service_modalities)
ORDER BY distance_km;
```
5. Full-Text Search — PostgreSQL
```

-- Busca por texto (nome, bio, especialidade)
SELECT *
FROM professional_search
WHERE search_vector @@ PLAINTO_TSQUERY('portuguese', :query)
ORDER BY TS_RANK(search_vector, PLAINTO_TSQUERY('portuguese', :query)) DESC;
```

6. Query Clojure — Busca Completa
```
(defn build-search-query
  [{:keys [q specialty-id city lat lng radius-km
           modality price-min price-max rating-min
           date page per-page sort-by]}]
  (cond-> {:select [:*]
           :from   [:professional_search]
           :where  [:and
                    [:= :approval_status "APPROVED"]
                    [:= :is_profile_complete true]]}

    q         (update :where conj
                [:@@ :search_vector
                 [:plainto_tsquery "portuguese" q]])

    specialty-id (update :where conj
                   [:@> :specialty_ids [:array [specialty-id]]])

    modality  (update :where conj
                [:@> :service_modalities [:array [(name modality)]]])

    price-min (update :where conj [:>= :consultation_price_presential price-min])
    price-max (update :where conj [:<= :consultation_price_presential price-max])
    rating-min (update :where conj [:>= :average_rating rating-min])

    (and lat lng radius-km)
    (update :where conj
      [:<= [:distance [:point :lng :lat] [:point lng lat]] radius-km])

    :always
    (assoc :limit per-page
           :offset (* (dec page) per-page))))
```

7. Cache de Busca (Redis)
```

;; Cache de resultados de busca por hash dos parâmetros
;; TTL: 2 minutos (busca muda pouco em curto prazo)

(def search-cache-ttl 120)

(defn search-with-cache [params db redis]
  (let [cache-key (str "search:" (hash params))
        cached    (redis/get redis cache-key)]
    (if cached
      (json/decode cached)
      (let [result (db/execute! db (build-search-query params))]
        (redis/setex redis cache-key search-cache-ttl (json/encode result))
        result))))

;; Invalidação: quando a MV é refreshada, flush do namespace search:*
(defn invalidate-search-cache! [redis]
  (redis/del-pattern redis "search:*"))
```

8. Observabilidade
```
Métrica: search.query.duration (p50/p95/p99)

Métrica: search.results.count — média de resultados por busca

Métrica: search.cache.hit_rate

Alerta: busca retornando 0
