# SQLviz Architecture Handbook

## Capítulo 14 — Metric Engine

Versión: 0.1

---

# 1. Introducción

La mayoría de herramientas BI tratan una métrica como una columna.

Ejemplo:

```sql
SUM(revenue)
```

↓

```text
Revenue
```

---

Pero un analista no piensa así.

Piensa:

```text
Revenue
↓
Growth
↓
Profit
↓
Margin
↓
Conversion
↓
Customer Lifetime Value
```

---

Es decir:

Una métrica es parte de una red de conocimiento.

---

# 2. Objetivo

Transformar:

```text
SUM(revenue)
```

en:

```text
Revenue
├── Revenue Growth
├── Revenue Forecast
├── Revenue YoY
├── Revenue MoM
├── Revenue Share
├── Revenue Contribution
└── Revenue Trend
```

---

# 3. Filosofía

Las métricas no son números.

Las métricas son conceptos de negocio.

---

SQLviz debe conocer:

```text
Qué mide
Cómo se calcula
Con qué se relaciona
Qué análisis habilita
```

---

# 4. Arquitectura

```text
SQL
 │
 ▼
Semantic Engine
 │
 ▼
Metric Detection
 │
 ▼
Metric Registry
 │
 ▼
Metric Graph
 │
 ▼
Metric Intelligence
```

---

# 5. Metric Registry

DuckDB.

```sql
CREATE TABLE metric_registry (

    metric_id VARCHAR,

    metric_name VARCHAR,

    semantic_type VARCHAR,

    aggregation_type VARCHAR,

    confidence DOUBLE
);
```

---

# 6. Metric Types

SQLviz define:

```text
Base Metric
Derived Metric
Ratio Metric
Growth Metric
Forecast Metric
KPI Metric
```

---

# 7. Base Metric

Detectadas desde SQL.

Ejemplo:

```sql
SUM(revenue)
```

↓

```text
Revenue
```

---

```sql
COUNT(order_id)
```

↓

```text
Orders
```

---

# 8. Derived Metric

Calculadas desde otras.

Ejemplo:

```text
Revenue
Cost
```

↓

```text
Profit
```

---

# 9. Ratio Metrics

Ejemplo:

```text
Revenue / Customers
```

↓

```text
ARPU
```

---

```text
Orders / Visits
```

↓

```text
Conversion Rate
```

---

# 10. Growth Metrics

Ejemplo:

```text
Revenue
```

genera:

```text
Revenue Growth %
Revenue MoM
Revenue YoY
Revenue CAGR
```

---

# 11. Metric Families

Ejemplo.

```text
Revenue
├── Revenue Growth
├── Revenue Forecast
├── Revenue YoY
├── Revenue Share
└── Revenue Contribution
```

---

# 12. Metric Graph

Representación.

```text
Revenue
 │
 ├── Growth
 │
 ├── Forecast
 │
 ├── Geography
 │
 └── Product
```

---

# 13. Metric Relationships

DuckDB.

```sql
CREATE TABLE metric_edges (

    parent_metric VARCHAR,

    child_metric VARCHAR,

    relationship_type VARCHAR,

    score DOUBLE
);
```

---

# 14. KPI Detection

Problema:

¿Cómo detectar KPIs automáticamente?

---

Señales:

```text
SUM()
AVG()
COUNT()
Window Functions
Frecuencia de uso
Alias semántico
```

---

Ejemplo.

```sql
SUM(revenue)
```

↓

```text
High KPI Probability
```

---

# 15. KPI Score

```python
kpi_score =
(
 semantic_importance * 0.4
 +
 usage_frequency * 0.3
 +
 aggregation_strength * 0.3
)
```

---

# 16. KPI Registry

```sql
CREATE TABLE kpi_registry (

    metric_name VARCHAR,

    kpi_score DOUBLE
);
```

---

# 17. Growth Detection

Consulta:

```sql
GROUP BY month
```

*

```sql
SUM(revenue)
```

↓

```text
Growth Candidate
```

---

# 18. Growth SQL Generation

```sql
SELECT

month,

revenue,

(
 revenue
 -
 LAG(revenue)
 OVER(ORDER BY month)
)
/ LAG(revenue)
 OVER(ORDER BY month)

AS growth_pct
```

---

# 19. YoY Detection

Si existe:

```text
Time Granularity >= Month
```

↓

Generar.

```text
YoY
```

---

# 20. Contribution Analysis

Revenue por producto.

↓

Generar automáticamente.

```sql
revenue
/
SUM(revenue)
OVER()
```

↓

```text
Contribution %
```

---

# 21. Pareto Detection

Revenue por producto.

↓

Ordenar.

↓

Acumulado.

↓

Detectar.

```text
Top 20%
Contribuye 80%
```

---

# 22. Metric Tree

Ejemplo.

```text
Revenue
├── Gross Revenue
├── Net Revenue
├── Growth
├── Share
└── Forecast
```

---

# 23. Business KPI Packs

Plantillas.

---

Sales Pack

```text
Revenue
Orders
Customers
Margin
```

---

Product Pack

```text
DAU
MAU
Retention
Churn
```

---

Finance Pack

```text
Revenue
Cost
Profit
Cashflow
```

---

# 24. KPI Archetypes

DuckDB.

```sql
CREATE TABLE kpi_archetypes (

    archetype_name VARCHAR,

    metrics JSON
);
```

---

# 25. Metric Similarity

```python
metric_similarity(
    "revenue",
    "sales"
)
```

↓

```text
0.95
```

---

# 26. Metric Centrality

En el Dashboard Graph.

---

Ejemplo.

```text
Revenue
```

aparece en:

```text
Trend
Country
Product
Forecast
```

↓

Alta centralidad.

---

# 27. Metric Importance

```python
importance =
(
 centrality * 0.4
 +
 kpi_score * 0.4
 +
 usage_score * 0.2
)
```

---

# 28. Metric Learning

Registrar.

```text
KPI aceptado

KPI eliminado

KPI agregado
```

---

# 29. Metric Feedback

```sql
CREATE TABLE metric_feedback (

    metric_name VARCHAR,

    accepted BOOLEAN
);
```

---

# 30. Explainability

Ejemplo.

```json
{
  "metric":"Revenue",
  "reason":"high semantic importance",
  "kpi_score":0.96
}
```

---

# 31. Dashboard Composer Integration

Metric Engine alimenta:

```text
Intent Engine
Dashboard Composer
Layout Engine
Query Rewrite Engine
```

---

# 32. Insight Generation

Revenue detectado.

↓

Composer sabe generar:

```text
Revenue KPI
Revenue Growth
Revenue Forecast
Revenue Share
```

---

# 33. La Idea Más Importante

La mayoría de herramientas modelan:

```text
Metric = Aggregation
```

---

SQLviz debe modelar:

```text
Metric = Knowledge Graph Node
```

---

# 34. Principio Fundamental

El Metric Engine transforma agregaciones SQL en conceptos analíticos reutilizables.

Es la pieza que permite que SQLviz construya KPI trees, métricas derivadas y análisis automáticos sin configuración manual.

Sin él, SQLviz visualiza datos.

Con él, SQLviz empieza a comprender negocio.