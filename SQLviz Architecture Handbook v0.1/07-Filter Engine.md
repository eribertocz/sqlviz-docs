# SQLviz Architecture Handbook

## Capítulo 7 — Filter Engine

Versión: 0.1

---

# 1. Introducción

La mayoría de herramientas BI tratan los filtros como elementos manuales.

El usuario debe decidir:

* qué filtrar
* dónde mostrar el filtro
* qué paneles afecta
* qué valores mostrar

SQLviz elimina esta configuración.

---

# 2. Filosofía

El usuario nunca debería escribir:

```yaml
filters:
  - country
  - region
  - product
```

Ni:

```json
{
  "filters": [...]
}
```

Ni:

```python
add_filter(...)
```

---

El usuario únicamente escribe SQL.

SQLviz descubre automáticamente:

```text
Qué filtros existen
Qué filtros son útiles
Qué filtros deben ser globales
Qué filtros deben ser locales
Qué paneles afectan
```

---

# 3. Definición

Un filtro es una dimensión interactiva que permite restringir el contexto analítico.

Ejemplos:

```text
country
region
city
product
category
customer
date
year
month
```

---

# 4. Arquitectura

```text
SQL
 │
 ▼
Feature Extraction
 │
 ▼
Dimension Discovery
 │
 ▼
Filter Candidate Generator
 │
 ▼
Filter Scoring
 │
 ▼
Filter Ranking
 │
 ▼
Filter Selection
```

---

# 5. Problema Fundamental

Supongamos:

```sql
SELECT
    country,
    SUM(revenue)
FROM sales
GROUP BY country
```

Existen múltiples candidatos:

```text
country
region
product
channel
customer
date
```

No todos deben convertirse en filtros.

---

# 6. Filter Candidate Discovery

Fuentes principales:

```text
GROUP BY
WHERE
JOIN
Foreign Keys
Semantic Dictionary
Panel Relationships
```

---

# 7. GROUP BY Discovery

Regla:

Toda dimensión usada en GROUP BY es candidata.

---

Ejemplo:

```sql
GROUP BY country
```

Genera:

```python
{
    "country": candidate
}
```

---

# 8. WHERE Discovery

Ejemplo:

```sql
WHERE region = 'LATAM'
```

Genera:

```python
{
    "region": candidate
}
```

---

Esto suele indicar una dimensión importante.

---

# 9. JOIN Discovery

Ejemplo:

```sql
FROM sales s
JOIN dim_product p
```

Genera:

```text
product
category
brand
```

como candidatos.

---

# 10. Foreign Key Discovery

DuckDB schema:

```text
customer_id
product_id
region_id
```

sugiere dimensiones filtrables.

---

# 11. Semantic Discovery

Diccionario:

```text
country
region
city
```

↓

```text
GEO_DIMENSION
```

---

Estas dimensiones reciben prioridad.

---

# 12. Filter Feature Vector

Cada filtro recibe features.

---

Ejemplo:

```python
FilterFeatures(
    cardinality=15,
    usage_frequency=0.9,
    semantic_score=0.8,
    relationship_score=0.7
)
```

---

# 13. Cardinality Score

Una dimensión con:

```text
country = 10
```

es excelente.

---

Una dimensión con:

```text
customer = 2,000,000
```

es mala para filtro.

---

Fórmula:

```python
cardinality_score =
1 / log(cardinality + 1)
```

---

# 14. Semantic Score

Categorías preferidas:

```text
date
country
region
category
product
channel
```

---

Ejemplo:

```python
semantic_score = 0.95
```

---

# 15. Usage Score

Frecuencia histórica.

---

Si:

```text
country
```

ha sido usado frecuentemente.

Aumentar score.

---

# 16. Relationship Score

Dimensiones compartidas entre paneles.

---

Ejemplo:

Panel A

```sql
GROUP BY country
```

Panel B

```sql
GROUP BY country, product
```

---

Score alto.

---

# 17. Filter Score

Fórmula inicial:

```python
filter_score =
(
    semantic_score * 0.3
    +
    cardinality_score * 0.3
    +
    relationship_score * 0.2
    +
    usage_score * 0.2
)
```

---

# 18. Ranking

Resultado:

```python
[
 ("country",0.94),
 ("region",0.88),
 ("product",0.62),
 ("customer",0.05)
]
```

---

# 19. Global Filters

Afectan múltiples paneles.

---

Ejemplo:

```text
country
```

aparece en:

```text
Revenue
Customers
Orders
Margin
```

---

Convertir en:

```text
Global Filter
```

---

# 20. Local Filters

Afectan un panel.

---

Ejemplo:

```text
salesperson
```

aparece sólo en un panel.

---

Convertir en:

```text
Local Filter
```

---

# 21. Shared Dimension Graph

Construir grafo.

---

Ejemplo:

```text
Revenue ── country ── Customers
     │
     └── country ── Orders
```

---

Esto permite descubrir filtros globales.

---

# 22. DuckDB Schema

```sql
CREATE TABLE filter_candidates (

    fingerprint_id VARCHAR,

    filter_name VARCHAR,

    score DOUBLE,

    cardinality INTEGER,

    semantic_score DOUBLE,

    relationship_score DOUBLE
);
```

---

# 23. Filter Registry

```sql
CREATE TABLE filter_registry (

    filter_name VARCHAR PRIMARY KEY,

    semantic_category VARCHAR,

    preferred_widget VARCHAR
);
```

---

# 24. Widget Inference

Cardinalidad determina widget.

---

## Low Cardinality

```text
2–15
```

Widget:

```text
Dropdown
Radio
Segmented Control
```

---

## Medium Cardinality

```text
15–100
```

Widget:

```text
Searchable Select
```

---

## High Cardinality

```text
100+
```

Widget:

```text
Autocomplete
```

---

# 25. Date Filter Inference

Detectar:

```text
DATE
TIMESTAMP
```

---

Widgets:

```text
Date Picker
Date Range
Month Selector
Year Selector
```

---

# 26. Cross Filtering

Cuando usuario selecciona:

```text
country = Bolivia
```

SQLviz recalcula:

```text
Revenue
Orders
Customers
Margin
```

automáticamente.

---

# 27. Filter Context Engine

Representación interna.

```python
FilterContext(
    country="Bolivia",
    year=2026
)
```

---

# 28. Query Rewriting

Filtro:

```text
country = Bolivia
```

---

Consulta original:

```sql
SELECT
country,
SUM(revenue)
FROM sales
GROUP BY country
```

---

Consulta reescrita:

```sql
SELECT
country,
SUM(revenue)
FROM sales
WHERE country='Bolivia'
GROUP BY country
```

---

# 29. Learning Engine

Registrar:

```text
Filtro usado
Filtro ignorado
Filtro eliminado
Filtro agregado
```

---

# 30. Feedback Table

```sql
CREATE TABLE filter_feedback (

    filter_name VARCHAR,

    accepted BOOLEAN,

    timestamp TIMESTAMP
);
```

---

# 31. Acceptance Rate

```python
acceptance_rate =
accepted
/
(
accepted + rejected
)
```

---

# 32. Auto-Hide

Si un filtro tiene:

```text
acceptance_rate < 5%
```

dejar de sugerirlo.

---

# 33. Auto-Promote

Si un filtro tiene:

```text
acceptance_rate > 90%
```

promover a:

```text
High Confidence Filter
```

---

# 34. Explainability

Mostrar:

```json
{
  "filter":"country",
  "score":0.94,
  "reasons":[
    "shared_dimension",
    "low_cardinality",
    "geo_dimension"
  ]
}
```

---

# 35. Dashboard Intelligence

La verdadera función del Filter Engine no es generar filtros.

Es descubrir el contexto compartido del dashboard.

---

Porque un dashboard no es:

```text
10 gráficos
```

---

Un dashboard es:

```text
10 gráficos
+
Contexto compartido
+
Interacción
```

---

# 36. Principio Fundamental

El Filter Engine convierte dimensiones en contexto.

Y el contexto es el mecanismo que transforma paneles aislados en una experiencia analítica integrada.

Por esta razón el Filter Engine es la primera pieza necesaria para construir dashboards verdaderamente autónomos.
