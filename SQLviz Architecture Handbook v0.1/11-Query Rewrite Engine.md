# SQLviz Architecture Handbook

## Capítulo 11 — Query Rewrite Engine

Versión: 0.1

---

# 1. Introducción

Hasta ahora SQLviz puede:

```text
SQL
 ↓
Features
 ↓
Intent
 ↓
Charts
 ↓
Filters
 ↓
Layout
 ↓
Dashboard Composition
```

Pero existe un problema.

El Dashboard Composer puede inferir:

```text
Revenue KPI
Revenue Trend
Revenue by Country
Revenue by Product
Revenue Forecast
```

Sin embargo aún no existe una forma de generar las consultas necesarias.

---

Aquí aparece Query Rewrite Engine.

---

# 2. Objetivo

Transformar una consulta existente en nuevas consultas relacionadas.

---

Ejemplo:

Consulta original:

```sql
SELECT
    month,
    SUM(revenue) revenue
FROM sales
GROUP BY month
```

---

Consultas derivadas:

```sql
SELECT
    SUM(revenue)
FROM sales
```

---

```sql
SELECT
    country,
    SUM(revenue)
FROM sales
GROUP BY country
```

---

```sql
SELECT
    product,
    SUM(revenue)
FROM sales
GROUP BY product
```

---

```sql
SELECT
    month,
    SUM(revenue),
    LAG(SUM(revenue)) OVER (
        ORDER BY month
    )
FROM sales
GROUP BY month
```

---

# 3. Filosofía

No generar SQL desde cero.

Primero reutilizar SQL existente.

---

Incorrecto:

```text
Prompt
 ↓
LLM
 ↓
SQL
```

---

Correcto:

```text
SQL
 ↓
AST
 ↓
Transformations
 ↓
New SQL
```

---

# 4. Arquitectura

```text
Original SQL
      │
      ▼
SQL Parser
      │
      ▼
AST
      │
      ▼
Rewrite Rules
      │
      ▼
Candidate Queries
      │
      ▼
Scoring
      │
      ▼
Generated Panels
```

---

# 5. SQL Parser

Recomendación:

```text
sqlglot
```

---

Porque permite:

```text
Parse
Transform
Generate
Normalize
```

---

# 6. Canonical AST

Ejemplo:

```sql
SELECT
 month,
 SUM(revenue)
FROM sales
GROUP BY month
```

↓

AST.

---

Toda transformación trabaja sobre AST.

Nunca sobre strings.

---

# 7. Query Fingerprint

Reutilizar.

```text
TIME_SUM
```

---

Ejemplo:

```sql
GROUP BY month
SUM(revenue)
```

↓

```text
TIME_SUM
```

---

# 8. Rewrite Categories

SQLviz define:

```text
Aggregation Rewrite
Dimension Rewrite
Metric Rewrite
Time Rewrite
Ranking Rewrite
Comparison Rewrite
Forecast Rewrite
Anomaly Rewrite
```

---

# 9. KPI Rewrite

Entrada:

```sql
SELECT
 month,
 SUM(revenue)
FROM sales
GROUP BY month
```

---

Salida:

```sql
SELECT
 SUM(revenue) revenue
FROM sales
```

---

Regla:

```python
remove_group_by()
```

---

# 10. Geography Rewrite

Entrada:

```sql
GROUP BY month
```

---

Descubrir:

```text
country
region
city
```

---

Salida:

```sql
GROUP BY country
```

---

# 11. Product Rewrite

Entrada:

```sql
GROUP BY month
```

---

Descubrir:

```text
product
category
brand
```

---

Salida:

```sql
GROUP BY product
```

---

# 12. Ranking Rewrite

Entrada:

```sql
GROUP BY product
```

---

Salida:

```sql
GROUP BY product

ORDER BY revenue DESC

LIMIT 10
```

---

# 13. Growth Rewrite

Entrada:

```sql
Revenue Trend
```

---

Salida:

```sql
SELECT

 month,

 revenue,

 revenue
 /
 lag(revenue)
```

---

Genera:

```text
Growth %
```

---

# 14. Moving Average Rewrite

Entrada:

```sql
Revenue Trend
```

---

Salida:

```sql
AVG(revenue)
OVER (
  ORDER BY month
  ROWS BETWEEN 2 PRECEDING
  AND CURRENT ROW
)
```

---

# 15. Forecast Rewrite

Entrada:

```text
Time + Metric
```

---

Genera:

```text
Forecast Dataset
```

---

Implementación inicial:

```text
Linear Trend
```

---

Posteriormente:

```text
Prophet-like
ARIMA
ETS
```

---

# 16. Anomaly Rewrite

Entrada:

```text
Trend
```

---

Genera:

```sql
SELECT
 month,
 revenue,
 zscore
```

---

Panel:

```text
Anomaly Detection
```

---

# 17. Comparison Rewrite

Entrada:

```sql
GROUP BY country
```

---

Salida:

```sql
GROUP BY country

ORDER BY revenue DESC
```

---

# 18. Dimension Replacement

Algoritmo clave.

---

Consulta:

```sql
GROUP BY month
```

---

Detectar dimensiones disponibles.

```python
[
  "country",
  "region",
  "product",
  "channel"
]
```

---

Generar variantes.

---

# 19. Metric Expansion

Consulta:

```sql
SUM(revenue)
```

---

Buscar métricas relacionadas.

```text
profit
orders
customers
margin
```

---

Generar paneles complementarios.

---

# 20. Rewrite Candidate

Modelo.

```python
class RewriteCandidate:

    sql: str

    source_panel: str

    transformation: str

    score: float
```

---

# 21. Rewrite Registry

DuckDB.

```sql
CREATE TABLE rewrite_registry (

    rewrite_name VARCHAR,

    category VARCHAR,

    enabled BOOLEAN
);
```

---

# 22. Rewrite History

```sql
CREATE TABLE rewrite_history (

    fingerprint_id VARCHAR,

    rewrite_name VARCHAR,

    accepted BOOLEAN,

    timestamp TIMESTAMP
);
```

---

# 23. Rewrite Score

Cada transformación recibe score.

---

Fórmula:

```python
rewrite_score =
(
    relevance * 0.5
    +
    novelty * 0.2
    +
    historical_success * 0.3
)
```

---

# 24. Relevance

Ejemplo:

Revenue Trend

↓

Revenue KPI

```text
0.98
```

---

Revenue Trend

↓

Forecast

```text
0.70
```

---

# 25. Novelty

Evitar duplicados.

---

Si ya existe:

```text
Revenue by Country
```

---

No generar nuevamente.

---

# 26. Historical Success

Aprendizaje.

---

Ejemplo:

```text
Growth Panel
```

aceptado frecuentemente.

↓

Score mayor.

---

# 27. Query Family

Concepto importante.

---

Consulta original:

```sql
Revenue by Month
```

---

Familia:

```text
Revenue KPI

Revenue Trend

Revenue Growth

Revenue Forecast

Revenue Anomalies
```

---

# 28. Query Graph

Representación.

```text
Revenue Trend
      │
      ├── KPI
      │
      ├── Growth
      │
      ├── Forecast
      │
      └── Anomaly
```

---

# 29. Rewrite Templates

Tabla.

```sql
CREATE TABLE rewrite_templates (

    rewrite_name VARCHAR,

    source_pattern VARCHAR,

    target_pattern VARCHAR
);
```

---

# 30. Explainability

Toda transformación debe explicarse.

---

Ejemplo:

```json
{
  "generated_panel":"Revenue KPI",
  "rewrite":"remove_group_by",
  "reason":"metric overview"
}
```

---

# 31. Learning

Usuario elimina panel.

---

Registrar.

```sql
INSERT INTO rewrite_history
```

---

Actualizar score.

---

# 32. Safety Rules

Nunca modificar:

```text
WHERE
JOIN
SECURITY FILTERS
ROW FILTERS
```

sin validación.

---

Mantener:

```text
business meaning
```

---

# 33. Performance

Cachear.

---

Clave:

```text
fingerprint
+
rewrite
```

---

Tabla:

```sql
CREATE TABLE rewrite_cache (

    cache_key VARCHAR,

    generated_sql TEXT
);
```

---

# 34. Roadmap

V1:

```text
Rule-Based Rewrite
```

---

V2:

```text
Graph-Based Rewrite
```

---

V3:

```text
Learning-Based Rewrite
```

---

# 35. Principio Fundamental

El Query Rewrite Engine convierte una consulta en una familia de consultas relacionadas.

No intenta adivinar SQL arbitrario.

No intenta reemplazar al usuario.

Su función es expandir automáticamente el espacio analítico alrededor de la intención detectada.

Por esta razón es el puente entre:

```text
Intent Understanding
```

y

```text
Dashboard Generation
```

y constituye uno de los componentes esenciales para alcanzar la visión de SQLviz como sistema Autonomous BI.
