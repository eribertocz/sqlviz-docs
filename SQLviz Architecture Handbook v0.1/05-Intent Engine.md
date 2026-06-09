# SQLviz Architecture Handbook

## Capítulo 5 — Intent Engine

Versión: 0.1

---

# 1. Introducción

El Intent Engine es el verdadero cerebro de SQLviz.

No decide qué gráfico mostrar.

No decide qué filtros crear.

No decide el layout.

Su responsabilidad es más importante:

Determinar qué está intentando hacer el usuario.

---

# 2. El Problema Fundamental

La mayoría de herramientas BI interpretan:

```text
SQL → Chart
```

SQLviz interpreta:

```text
SQL → Intent → Dashboard
```

Esta diferencia cambia completamente la arquitectura.

---

# 3. ¿Qué es una Intención?

Una intención es el objetivo analítico implícito del usuario.

Ejemplo:

```sql
SELECT
    month,
    SUM(revenue)
FROM sales
GROUP BY month
```

La intención no es:

```text
Line Chart
```

La intención es:

```text
Trend Analysis
```

---

# 4. Filosofía

Los gráficos son consecuencias.

Las intenciones son causas.

---

Incorrecto:

```text
SQL
 ↓
Bar Chart
```

Correcto:

```text
SQL
 ↓
Comparison Intent
 ↓
Bar Chart
```

---

# 5. Taxonomía Oficial de Intenciones

SQLviz define inicialmente 12 intenciones.

---

## Trend

Analizar evolución temporal.

Ejemplos:

```sql
GROUP BY month
GROUP BY year
GROUP BY date
```

---

## Comparison

Comparar categorías.

Ejemplo:

```sql
GROUP BY region
```

---

## Ranking

Encontrar mejores o peores elementos.

Ejemplo:

```sql
ORDER BY sales DESC
LIMIT 10
```

---

## Distribution

Analizar distribución.

Ejemplo:

```sql
SELECT age
FROM customers
```

---

## Correlation

Analizar relación entre variables.

Ejemplo:

```sql
SELECT price, revenue
```

---

## Composition

Analizar parte vs todo.

Ejemplo:

```sql
SELECT category,
       SUM(sales)
```

con pocas categorías.

---

## KPI

Mostrar estado.

Ejemplo:

```sql
SELECT SUM(revenue)
```

---

## Anomaly

Buscar comportamiento inusual.

Ejemplo:

```sql
SELECT day,
       revenue
```

con outliers significativos.

---

## Cohort

Analizar cohortes.

Ejemplo:

```text
cohort
retention
signup_month
```

---

## Retention

Analizar permanencia.

Ejemplo:

```text
retention_rate
churn
active_users
```

---

## Funnel

Analizar conversiones.

Ejemplo:

```text
lead
opportunity
purchase
```

---

## Detail

Exploración tabular.

Ejemplo:

```sql
SELECT *
```

---

# 6. Modelo de Intención

SQLviz nunca produce una única intención.

Produce probabilidades.

---

Ejemplo:

```python
{
    "trend": 0.92,
    "comparison": 0.15,
    "ranking": 0.04,
    "distribution": 0.01
}
```

---

# 7. Intent Vector

Representación interna.

```python
class IntentVector:

    trend: float

    comparison: float

    ranking: float

    distribution: float

    correlation: float

    composition: float

    kpi: float

    anomaly: float

    cohort: float

    retention: float

    funnel: float

    detail: float
```

---

# 8. Arquitectura

```text
Feature Vector
      │
      ▼
Intent Scorers
      │
      ▼
Intent Scores
      │
      ▼
Normalization
      │
      ▼
Intent Vector
```

---

# 9. Trend Scorer

Inputs:

```text
time_dimension
trend_strength
seasonality
```

---

Fórmula inicial:

```python
trend_score =
(
    time_dimension * 0.4
    +
    trend_strength * 0.4
    +
    seasonality * 0.2
)
```

---

Ejemplo:

```python
trend_score = 0.93
```

---

# 10. Comparison Scorer

Inputs:

```text
group_by_category
cardinality
aggregation
```

---

Fórmula:

```python
comparison_score =
(
    category_dimension * 0.5
    +
    aggregation_present * 0.3
    +
    cardinality_score * 0.2
)
```

---

# 11. Ranking Scorer

Inputs:

```text
ORDER BY DESC
LIMIT
```

---

Fórmula:

```python
ranking_score =
(
    has_order_desc * 0.6
    +
    has_limit * 0.4
)
```

---

# 12. Distribution Scorer

Inputs:

```text
single_numeric_column
histogram_candidate
```

---

Fórmula:

```python
distribution_score =
(
    numeric_dimension * 0.5
    +
    no_group_by * 0.5
)
```

---

# 13. Correlation Scorer

Inputs:

```text
numeric_x
numeric_y
correlation
```

---

Fórmula:

```python
correlation_score =
(
    two_numeric_columns * 0.5
    +
    abs(correlation) * 0.5
)
```

---

# 14. KPI Scorer

Inputs:

```text
rows == 1
single_metric
```

---

Fórmula:

```python
kpi_score =
(
    one_row * 0.7
    +
    one_metric * 0.3
)
```

---

# 15. Anomaly Scorer

Inputs:

```text
outlier_ratio
variance
```

---

Fórmula:

```python
anomaly_score =
(
    outlier_ratio * 0.6
    +
    variance_score * 0.4
)
```

---

# 16. Cohort Scorer

Inputs:

```text
cohort
signup
retention
```

---

Semantic driven.

---

# 17. Retention Scorer

Inputs:

```text
retention
active_users
churn
```

---

Semantic driven.

---

# 18. Funnel Scorer

Inputs:

```text
stage
lead
opportunity
purchase
conversion
```

---

Semantic driven.

---

# 19. Detail Scorer

Inputs:

```text
SELECT *
high_column_count
```

---

Fórmula:

```python
detail_score =
(
    many_columns * 0.6
    +
    no_aggregation * 0.4
)
```

---

# 20. Normalización

Después de calcular todos los scores:

```python
scores = [...]
```

Aplicar:

```python
softmax()
```

---

Resultado:

```python
{
    "trend": 0.84,
    "comparison": 0.09,
    "ranking": 0.03,
    ...
}
```

---

# 21. DuckDB Schema

Tabla principal.

```sql
CREATE TABLE intent_patterns (

    fingerprint_id VARCHAR,

    trend FLOAT,

    comparison FLOAT,

    ranking FLOAT,

    distribution FLOAT,

    correlation FLOAT,

    composition FLOAT,

    kpi FLOAT,

    anomaly FLOAT,

    cohort FLOAT,

    retention FLOAT,

    funnel FLOAT,

    detail FLOAT,

    occurrences INTEGER
);
```

---

# 22. Intent Confidence

Además del vector.

Guardar:

```python
confidence
```

---

Ejemplo:

```python
0.94
```

---

Interpretación:

```text
Alta confianza
```

---

# 23. Ambigüedad

Algunas consultas son ambiguas.

Ejemplo:

```sql
SELECT month,
       revenue
```

Puede ser:

```text
Trend
Distribution
```

---

No forzar decisión.

Guardar ambas.

---

# 24. Intent History

Aprendizaje.

Tabla:

```sql
CREATE TABLE intent_feedback (

    fingerprint_id VARCHAR,

    inferred_intent VARCHAR,

    accepted BOOLEAN,

    timestamp TIMESTAMP
);
```

---

# 25. Aprendizaje

Si un fingerprint:

```text
TIME_SUM
```

siempre termina usando:

```text
Trend
```

aumentar peso.

---

# 26. Integración con Chart Engine

Trend:

```text
Line
Area
```

---

Ranking:

```text
Horizontal Bar
```

---

Correlation:

```text
Scatter
```

---

KPI:

```text
Card
```

---

# 27. Integración con Layout Engine

Trend:

```text
Panel grande
```

---

KPI:

```text
Panel superior
```

---

Detail:

```text
Panel inferior
```

---

# 28. Integración con Dashboard Composer

El Dashboard Composer no trabaja con gráficos.

Trabaja con intenciones.

---

Ejemplo:

```text
Trend Intent
```

puede generar:

```text
Revenue KPI
Revenue Trend
Growth Rate
Forecast
```

---

# 29. El Gran Salto

La mayoría de herramientas hacen:

```text
Query → Visualization
```

SQLviz debe evolucionar hacia:

```text
Query → Intent
Intent → Dashboard
Dashboard → Insight
```

---

# 30. Principio Fundamental

El Intent Engine es el orquestador central de SQLviz.

Todos los demás motores consumen sus resultados.

Por esta razón debe considerarse el núcleo lógico del sistema y la base de la evolución futura hacia Autonomous BI.
