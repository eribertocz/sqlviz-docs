# SQLviz Technical Specification v1

## Capítulo 13 — Chart Engine

Versión 0.1

---

# 1. Objetivo

Inferir automáticamente la mejor visualización para una consulta SQL.

---

Sin configuración.

---

Sin opciones.

---

Sin wizard.

---

El usuario escribe:

```sql
SELECT ...
```

---

SQLviz decide:

```text
Chart
```

---

# 2. Filosofía

La mayoría de herramientas usan:

```text
Datatype
```

---

SQLviz debe usar:

```text
Intent

Metric

Dimension

Cardinality

Shape

Semantics
```

---

# 3. Input

```python
context.features

context.intents

context.metrics

context.dimensions

context.statistics
```

---

# 4. Output

```python
context.charts
```

---

# 5. Chart Model

```python
class Chart:

    chart_id: str

    chart_type: str

    score: float

    confidence: float

    explanation: Explanation
```

---

# 6. V1 Chart Types

```text
KPI

Line

Bar

Pie

Scatter

Table
```

---

# 7. Philosophy

Nunca elegir directamente.

---

Siempre rankear.

---

# 8. Example

```python
[
    Line(0.96),
    Bar(0.74),
    Scatter(0.20)
]
```

---

# 9. Winner

Mayor score.

---

# 10. Chart Scoring

Cada chart implementa.

```python
class ChartScorer:

    def score(
        context
    ) -> float:
        ...
```

---

# 11. Chart Registry

```python
CHARTS = [

    KPIChart(),

    LineChart(),

    BarChart(),

    PieChart(),

    ScatterChart(),

    TableChart()
]
```

---

# 12. KPI Chart

Caso especial.

---

# 13. KPI Signals

```text
Metric Count = 1

Dimension Count = 0

Intent = KPI
```

---

# 14. KPI Formula

```python
score =
(
 kpi_intent * 0.5
 +
 single_metric * 0.5
)
```

---

# 15. Example

```sql
SELECT
 SUM(revenue)
```

---

Resultado.

```text
KPI
0.99
```

---

# 16. Line Chart

Probablemente el chart más frecuente.

---

# 17. Signals

```text
Trend Intent

Temporal Dimension

Aggregation
```

---

# 18. Formula

```python
score =
(
 trend_intent * 0.5
 +
 temporal_dimension * 0.3
 +
 aggregation * 0.2
)
```

---

# 19. Example

```sql
SELECT
 month,
 SUM(revenue)
GROUP BY month
```

↓

```text
Line
0.97
```

---

# 20. Bar Chart

Comparación.

---

# 21. Signals

```text
Comparison Intent

Categorical Dimension

Aggregation
```

---

# 22. Formula

```python
score =
(
 comparison_intent * 0.5
 +
 category_dimension * 0.3
 +
 aggregation * 0.2
)
```

---

# 23. Example

```sql
SELECT
 country,
 SUM(revenue)
GROUP BY country
```

↓

```text
Bar
0.96
```

---

# 24. Pie Chart

Uso restringido.

---

# 25. Signals

```text
Composition Intent

Low Cardinality
```

---

# 26. Rule

Nunca.

```text
cardinality > 10
```

---

# 27. Formula

```python
score =
(
 composition_intent * 0.6
 +
 low_cardinality * 0.4
)
```

---

# 28. Example

```sql
SELECT
 channel,
 SUM(revenue)
GROUP BY channel
```

---

Si.

```text
4 canales
```

↓

```text
Pie
0.90
```

---

# 29. Scatter Chart

Correlación.

---

# 30. Signals

```text
Correlation Intent

2 Metrics
```

---

# 31. Formula

```python
score =
(
 correlation_intent * 0.7
 +
 metric_count_2 * 0.3
)
```

---

# 32. Example

```sql
SELECT
 revenue,
 profit
```

↓

```text
Scatter
0.95
```

---

# 33. Table Chart

Fallback universal.

---

# 34. Signals

```text
Low Confidence

Detail Inspection

Many Columns
```

---

# 35. Rule

Siempre disponible.

---

# 36. Never Return Empty

Si ningún chart supera:

```python
0.5
```

↓

```text
Table
```

---

# 37. Multi Metric Rules

Ejemplo.

```text
Revenue

Profit
```

---

Permitir.

```text
Line

Bar

Scatter
```

---

# 38. Temporal Priority

Si existe:

```text
Trend Intent
```

---

Line recibe boost.

---

# 39. Identifier Penalty

Ejemplo.

```text
customer_id
```

---

Penalizar.

```python
-0.4
```

---

# 40. High Cardinality Penalty

Ejemplo.

```text
5000 categorías
```

---

Penalizar.

```python
-0.5
```

---

# 41. Semantic Boost

Ejemplo.

```text
Revenue by Month
```

---

Boost.

```text
Line
```

---

# 42. Semantic Pattern Rules

```text
Revenue + Month

Profit + Month

Sales + Month
```

↓

```text
Line Preferred
```

---

# 43. Ranking Analysis

```text
Revenue DESC

LIMIT 10
```

↓

```text
Bar Boost
```

---

# 44. TopN Special Rule

Top N.

---

Nunca Pie.

---

Siempre.

```text
Bar
```

preferido.

---

# 45. Explainability

Obligatoria.

---

# 46. Example

```text
Line Chart

Reason:

Trend Intent 0.96

Temporal Dimension

Aggregation detected
```

---

# 47. Chart Registry Table

DuckDB.

```sql
CREATE TABLE chart_predictions (

    prediction_id VARCHAR,

    execution_id VARCHAR,

    chart_type VARCHAR,

    score DOUBLE
);
```

---

# 48. Chart Rules Table

```sql
CREATE TABLE chart_rules (

    chart_type VARCHAR,

    rule_name VARCHAR,

    weight DOUBLE
);
```

---

# 49. Event

Publicar.

```text
CHARTS_INFERRED
```

---

# 50. Payload

```python
{
   "winner":"Line",
   "confidence":0.97
}
```

---

# 51. Performance Goal

```text
< 10 ms
```

---

# 52. Cache Strategy

Clave.

```python
fingerprint_hash

intent_hash
```

---

# 53. Benchmark Corpus

```text
Trend → Line

Comparison → Bar

Composition → Pie

Correlation → Scatter

KPI → KPI
```

---

# 54. Accuracy Goal

```text
95%+
```

---

# 55. Chart Confidence

Modelo.

```python
confidence =
(
 winner_score
 -
 second_score
)
```

---

# 56. Example

```text
Line 0.95

Bar 0.93
```

↓

```text
Confidence baja
```

---

# 57. Example

```text
Line 0.95

Bar 0.40
```

↓

```text
Confidence alta
```

---

# 58. Strategic Insight

La mayoría de herramientas hacen:

```text
Column Types
↓
Chart
```

---

SQLviz hace:

```text
Intent
↓
Chart
```

---

# 59. Future V2

```text
Area

Heatmap

Treemap

Funnel

Waterfall
```

---

# 60. Future V3

```text
Custom Charts

Learned Charts

Chart Genome
```

---

# 61. Principio Fundamental

El Chart Engine no intenta visualizar datos.

Intenta visualizar la intención analítica detrás de los datos.

Por eso el mismo dataset puede producir gráficos distintos dependiendo de la pregunta implícita que detecte SQLviz.
