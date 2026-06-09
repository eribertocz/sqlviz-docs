# SQLviz Technical Specification v1

## Capítulo 10 — Intent Engine

Versión 0.1

---

# 1. Objetivo

Inferir qué está intentando analizar el usuario.

---

No:

```text
Qué escribió
```

---

Eso ya lo hizo Parser.

---

No:

```text
Qué significa
```

---

Eso ya lo hizo Semantic.

---

Ahora queremos responder:

```text
¿Por qué escribió esta consulta?
```

---

# 2. Filosofía

La intención es la variable más importante para Chart Engine.

---

Más importante que:

```text
Tipos de datos

Cardinalidad

Número de columnas
```

---

# 3. Ejemplo

Consulta.

```sql
SELECT
 month,
 SUM(revenue)
FROM sales
GROUP BY month
```

---

Parser ve:

```text
GROUP BY
SUM
```

---

Semantic ve:

```text
Revenue
Month
```

---

Intent Engine ve:

```text
Trend Analysis
```

---

# 4. Contract

```python
class IntentEngine(Engine):

    def execute(
        self,
        context: RuntimeContext
    ) -> RuntimeContext:
        ...
```

---

# 5. Input

```python
context.features

context.semantic_tags

context.statistics
```

---

# 6. Output

```python
context.intents
```

---

# 7. Intent Model

```python
class Intent:

    intent_id: str

    name: str

    score: float

    explanation: Explanation
```

---

# 8. Intent Philosophy

Una consulta puede tener múltiples intenciones.

---

# 9. Ejemplo

```sql
SELECT
 country,
 SUM(revenue)
FROM sales
GROUP BY country
ORDER BY revenue DESC
LIMIT 10
```

---

Puede generar.

```text
Comparison      0.95

Ranking         0.92

Top N           0.90
```

---

# 10. Nunca

```text
Una sola intención
```

---

# 11. Intent Registry

V1.

```text
Trend Analysis

Comparison

Ranking

Top N

Bottom N

Distribution

Composition

Correlation

KPI Monitoring

Detail Inspection
```

---

# 12. Trend Analysis

Detecta evolución temporal.

---

# 13. Señales

```text
HAS_TIME_GROUPING

HAS_DATE_COLUMN

HAS_AGGREGATION
```

---

# 14. Scoring

```python
trend_score =
(
 time_grouping * 0.5
 +
 aggregation * 0.3
 +
 temporal_ordering * 0.2
)
```

---

# 15. Example

```sql
SELECT
 month,
 SUM(revenue)
GROUP BY month
```

---

Resultado.

```text
Trend Analysis
0.96
```

---

# 16. Comparison

Detecta comparación entre categorías.

---

# 17. Señales

```text
Dimension

Aggregation

No temporal grouping
```

---

# 18. Example

```sql
SELECT
 country,
 SUM(revenue)
GROUP BY country
```

---

Resultado.

```text
Comparison
0.94
```

---

# 19. Ranking

Detecta ordenamiento.

---

# 20. Señales

```text
ORDER BY

DESC

Metric
```

---

# 21. Example

```sql
ORDER BY revenue DESC
```

↓

```text
Ranking
```

---

# 22. Top N

---

# 23. Señales

```text
ORDER BY DESC

LIMIT
```

---

# 24. Example

```sql
ORDER BY revenue DESC
LIMIT 10
```

↓

```text
Top N
0.97
```

---

# 25. Bottom N

---

# 26. Señales

```text
ORDER BY ASC

LIMIT
```

---

# 27. Composition

Parte del total.

---

# 28. Señales

```text
Low Cardinality

Categorical Dimension

Aggregation
```

---

# 29. Example

```sql
SELECT
 channel,
 SUM(revenue)
GROUP BY channel
```

---

Resultado.

```text
Composition
```

---

# 30. Distribution

Distribución de valores.

---

# 31. Example

```sql
SELECT revenue
FROM sales
```

---

Resultado.

```text
Distribution
```

---

# 32. Correlation

Relación entre métricas.

---

# 33. Señales

```text
Metric Count >= 2
```

---

# 34. Example

```sql
SELECT
 revenue,
 profit
FROM sales
```

↓

```text
Correlation
```

---

# 35. KPI Monitoring

Indicador único.

---

# 36. Señales

```text
Metric Count = 1

Dimension Count = 0
```

---

# 37. Example

```sql
SELECT
 SUM(revenue)
FROM sales
```

↓

```text
KPI Monitoring
```

---

# 38. Detail Inspection

Tabla detallada.

---

# 39. Señales

```text
No Aggregation

Many Columns
```

---

# 40. Example

```sql
SELECT *
FROM sales
```

↓

```text
Detail Inspection
```

---

# 41. Intent Score Model

Todos los intents compiten.

---

# 42. Resultado

```python
[
 Intent(
   name="Trend",
   score=0.96
 ),

 Intent(
   name="Comparison",
   score=0.62
 )
]
```

---

# 43. Winner

Mayor score.

---

Pero conservar ranking completo.

---

# 44. Why?

Explainability.

---

Future Learning.

---

Chart Selection.

---

# 45. Intent Registry Table

DuckDB.

```sql
CREATE TABLE intents (

    intent_id VARCHAR,

    execution_id VARCHAR,

    intent_name VARCHAR,

    score DOUBLE
);
```

---

# 46. Intent Rules Table

```sql
CREATE TABLE intent_rules (

    rule_name VARCHAR,

    weight DOUBLE
);
```

---

# 47. Explainability

Siempre.

---

# 48. Example

```text
Trend Analysis

Reason:

Time grouping detected

Date dimension detected

Aggregation detected
```

---

# 49. Event

Publicar.

```text
INTENTS_INFERRED
```

---

# 50. Payload

```python
{
   "winner":"Trend",
   "score":0.96
}
```

---

# 51. Performance Goal

```text
< 20 ms
```

---

# 52. Cache Key

Principalmente.

```python
fingerprint_hash
```

---

# 53. Why?

Consultas similares.

↓

Misma intención.

---

# 54. Benchmark Corpus

Casos mínimos.

```text
Trend

Comparison

Ranking

TopN

BottomN

Distribution

Correlation

KPI
```

---

# 55. Accuracy Goal

```text
95%+
```

---

# 56. Strategic Insight

La mayoría de herramientas eligen gráficos usando:

```text
Tipos de columnas
```

---

SQLviz debe elegir gráficos usando:

```text
Intención analítica
```

---

# 57. Ejemplo

Ambas consultas.

```sql
country + revenue
```

---

Pueden parecer iguales.

---

Pero:

```sql
GROUP BY country
```

↓

```text
Comparison
```

---

Mientras.

```sql
ORDER BY revenue DESC
LIMIT 10
```

↓

```text
Ranking
```

---

Mismos datos.

Distinta intención.

Distinto gráfico.

---

# 58. Future Evolution

V2.

```text
Forecasting

Anomaly Detection

Root Cause

Cohort Analysis

Funnel Analysis
```

---

# 59. Future Evolution

V3.

```text
Intent Learning

Intent Embeddings

Intent Graph
```

---

# 60. Principio Fundamental

El Intent Engine es donde SQLviz deja de analizar consultas y empieza a entender objetivos analíticos.

No intenta descubrir qué datos existen.

Intenta descubrir qué pregunta está haciendo el usuario.