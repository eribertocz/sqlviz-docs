# SQLviz Technical Specification v1

## Capítulo 11 — Metric Engine

Versión 0.1

---

# 1. Objetivo

Identificar automáticamente qué expresiones representan métricas analíticas.

---

Después del Semantic Engine ya conocemos significados.

---

Ahora debemos responder:

```text
¿Qué debe medirse?
```

---

# 2. Filosofía

Las métricas son el corazón de cualquier dashboard.

---

Sin métricas:

```text
No hay KPI

No hay charts

No hay insights
```

---

# 3. Input

```python
context.features

context.semantic_tags

context.ast
```

---

# 4. Output

```python
context.metrics
```

---

# 5. Metric Model

```python
class Metric:

    metric_id: str

    name: str

    expression: str

    aggregation: str | None

    semantic_tag: str

    confidence: float

    explanation: Explanation
```

---

# 6. Ejemplo

SQL.

```sql
SELECT
    SUM(revenue) revenue
FROM sales
```

---

Resultado.

```python
Metric(
    name="Revenue",
    expression="SUM(revenue)",
    aggregation="SUM",
    semantic_tag="Revenue",
    confidence=0.99
)
```

---

# 7. Regla Principal

Toda agregación es candidata a métrica.

---

# 8. Aggregation Candidates

```text
SUM

AVG

COUNT

MIN

MAX

MEDIAN
```

---

# 9. Ejemplo

```sql
SUM(revenue)
```

↓

```text
Metric
```

---

# 10. Ejemplo

```sql
AVG(price)
```

↓

```text
Metric
```

---

# 11. Semantic Candidates

Algunas columnas son métricas incluso sin agregación.

---

# 12. Ejemplos

```text
Revenue

Profit

Cost

Price

Quantity
```

---

# 13. Ejemplo

```sql
SELECT revenue
FROM sales
```

---

Resultado.

```text
Revenue Metric
```

---

# 14. Metric Score

Modelo inicial.

```python
score =
(
 aggregation_score * 0.5
 +
 semantic_score * 0.3
 +
 datatype_score * 0.2
)
```

---

# 15. Aggregation Score

```python
SUM()     = 1.0

AVG()     = 1.0

COUNT()   = 1.0

none      = 0.5
```

---

# 16. Semantic Score

```text
Revenue = 1.0

Profit = 1.0

Country = 0.0

Product = 0.0
```

---

# 17. Datatype Score

```text
Numeric = 1.0

String  = 0.0

Date    = 0.0
```

---

# 18. Metric Categories

V1.

```text
Financial

Operational

Count

Ratio

Derived
```

---

# 19. Financial Metrics

```text
Revenue

Profit

Cost

Margin
```

---

# 20. Operational Metrics

```text
Orders

Units

Inventory

Shipments
```

---

# 21. Count Metrics

```sql
COUNT(*)
```

↓

```text
Record Count
```

---

# 22. Ratio Metrics

Ejemplo.

```sql
profit / revenue
```

↓

```text
Margin
```

---

# 23. Derived Metrics

Construidas por expresiones.

---

# 24. Ejemplo

```sql
revenue - cost
```

↓

```text
Derived Metric
```

---

# 25. Expression Analysis

Analizar.

```text
+

-

*

/

CASE
```

---

# 26. Formula Registry

DuckDB.

```sql
CREATE TABLE metric_patterns (

    pattern_name VARCHAR,

    expression_pattern VARCHAR,

    semantic_tag VARCHAR
);
```

---

# 27. Ejemplo

```text
profit/revenue
```

↓

```text
Margin
```

---

# 28. Multi Metric Detection

Ejemplo.

```sql
SELECT
 revenue,
 profit
```

---

Resultado.

```python
[
 Revenue,
 Profit
]
```

---

# 29. Metric Priority

Ranking.

---

Ejemplo.

```text
Revenue 0.99

Profit 0.96

Quantity 0.82
```

---

# 30. Primary Metric

Siempre identificar.

---

# 31. Why?

Layout Engine.

---

Chart Engine.

---

Insights.

---

# 32. Primary Metric Score

```python
score =
(
 semantic_importance * 0.5
 +
 aggregation_weight * 0.3
 +
 usage_frequency * 0.2
)
```

---

# 33. Semantic Importance

Inicial.

```text
Revenue = 1.0

Profit = 0.95

Cost = 0.90

Quantity = 0.80
```

---

Configurable.

---

# 34. Metric Registry

DuckDB.

```sql
CREATE TABLE metrics (

    metric_id VARCHAR,

    execution_id VARCHAR,

    metric_name VARCHAR,

    metric_type VARCHAR,

    confidence DOUBLE
);
```

---

# 35. Explainability

Ejemplo.

```text
Revenue Metric

Reason:

SUM aggregation detected

Revenue semantic detected

Numeric datatype detected
```

---

# 36. Event

Publicar.

```text
METRICS_INFERRED
```

---

# 37. Payload

```python
{
   "metric_count": 2,
   "primary_metric": "Revenue"
}
```

---

# 38. Cache Strategy

Clave.

```python
column_name

semantic_tag
```

---

# 39. Benchmark Cases

```text
Revenue

Profit

Cost

Quantity

Count

Margin
```

---

# 40. Accuracy Goal

```text
98%+
```

para métricas comunes.

---

# 41. KPI Detection

Caso especial.

---

Ejemplo.

```sql
SELECT
 SUM(revenue)
```

---

Resultado.

```text
Primary KPI
```

---

# 42. KPI Score

```python
score =
(
 metric_score * 0.6
 +
 aggregation_score * 0.4
)
```

---

# 43. Composite Metrics

Detectar.

```sql
SUM(revenue) / COUNT(customer_id)
```

↓

```text
Revenue Per Customer
```

---

# 44. Future

V2.

```text
Metric Catalog

Business Metrics

Custom Metrics
```

---

# 45. Future

V3.

```text
Metric Learning

Metric Lineage

Metric Graph
```

---

# 46. Strategic Insight

Muchos sistemas identifican:

```text
Columnas numéricas
```

---

SQLviz debe identificar:

```text
Métricas de negocio
```

---

# 47. Principio Fundamental

El Metric Engine transforma números en medidas.

No identifica solamente columnas numéricas.

Identifica aquello que el usuario realmente quiere medir.
