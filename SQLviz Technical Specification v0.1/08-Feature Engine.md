# SQLviz Technical Specification v1

## Capítulo 08 — Feature Engine

Versión 0.1

---

# 1. Objetivo

Transformar SQL parseado en señales analíticas reutilizables.

---

El Parser responde:

```text
¿Qué escribió el usuario?
```

---

El Feature Engine responde:

```text
¿Qué señales contiene esa consulta?
```

---

# 2. Filosofía

El Feature Engine NO entiende negocio.

---

No sabe:

```text
Revenue

Profit

Customer
```

---

Solo sabe:

```text
Patrones

Estructuras

Características
```

---

# 3. Importancia

Probablemente el engine más importante de V1.

---

Porque:

```text
Semantic Engine

Intent Engine

Chart Engine

Layout Engine
```

dependen de él.

---

# 4. Pipeline

```text
SQL
 ↓
Parser
 ↓
AST
 ↓
Feature Engine
 ↓
Features
```

---

# 5. Contract

```python
class FeatureEngine(Engine):

    def execute(
        self,
        context: RuntimeContext
    ) -> RuntimeContext:
        ...
```

---

# 6. Input

```python
context.ast
```

---

# 7. Output

```python
context.features
```

---

# 8. Feature Definition

Una feature es una señal observable.

---

# 9. Modelo

```python
class Feature:

    feature_id: str

    name: str

    value: any

    confidence: float

    source: str
```

---

# 10. Categorías

V1.

```text
Structural

Aggregation

Temporal

Filter

Ordering

Cardinality

Statistical
```

---

# 11. Structural Features

Detectan forma SQL.

---

# 12. Ejemplos

```text
HAS_SELECT

HAS_WHERE

HAS_GROUP_BY

HAS_ORDER_BY

HAS_LIMIT

HAS_JOIN

HAS_CTE

HAS_SUBQUERY
```

---

# 13. Aggregation Features

Detectan análisis agregado.

---

# 14. Ejemplos

```text
HAS_SUM

HAS_AVG

HAS_COUNT

HAS_MIN

HAS_MAX
```

---

# 15. Composite Features

Más útiles.

---

Ejemplo.

```text
HAS_AGGREGATION
```

---

derivada de:

```text
SUM

AVG

COUNT
```

---

# 16. Temporal Features

Críticas para charts.

---

# 17. Ejemplos

```text
HAS_DATE_COLUMN

HAS_TIME_GROUPING

HAS_TEMPORAL_ORDERING

HAS_TIME_FILTER
```

---

# 18. Example

```sql
GROUP BY month
```

---

Produce.

```text
HAS_TIME_GROUPING
```

---

# 19. Filter Features

Capturan filtros.

---

# 20. Ejemplos

```text
HAS_EQUALITY_FILTER

HAS_RANGE_FILTER

HAS_IN_FILTER

HAS_NULL_FILTER
```

---

# 21. Example

```sql
WHERE revenue > 1000
```

↓

```text
HAS_RANGE_FILTER
```

---

# 22. Ordering Features

---

Ejemplos.

```text
HAS_ORDER_BY_ASC

HAS_ORDER_BY_DESC

HAS_TOP_N
```

---

# 23. Example

```sql
ORDER BY revenue DESC
LIMIT 10
```

↓

```text
HAS_TOP_N
```

---

# 24. Join Features

---

Ejemplos.

```text
HAS_JOIN

HAS_MULTI_JOIN

HAS_LEFT_JOIN
```

---

# 25. Complexity Features

---

Ejemplos.

```text
QUERY_COMPLEXITY_LOW

QUERY_COMPLEXITY_MEDIUM

QUERY_COMPLEXITY_HIGH
```

---

# 26. Complexity Formula

```python
score =
(
 joins * 2
 +
 ctes * 2
 +
 subqueries * 3
)
```

---

# 27. Result Shape Features

MUY importantes.

---

# 28. Ejemplos

```text
ROW_COUNT_SMALL

ROW_COUNT_MEDIUM

ROW_COUNT_LARGE
```

---

# 29. Column Shape Features

---

Ejemplos.

```text
DIMENSION_COUNT_1

DIMENSION_COUNT_2

METRIC_COUNT_1

METRIC_COUNT_N
```

---

# 30. Example

```sql
SELECT
 country,
 SUM(revenue)
```

↓

```text
DIMENSION_COUNT_1

METRIC_COUNT_1
```

---

# 31. Cardinality Features

Probablemente las más importantes para charts.

---

# 32. Ejemplos

```text
LOW_CARDINALITY

MEDIUM_CARDINALITY

HIGH_CARDINALITY
```

---

# 33. Thresholds

Inicialmente.

```python
LOW = < 10

MEDIUM = < 50

HIGH = >= 50
```

---

Configurable.

---

# 34. Example

```text
Country
```

↓

```text
LOW_CARDINALITY
```

---

# 35. Example

```text
Customer
```

↓

```text
HIGH_CARDINALITY
```

---

# 36. Distribution Features

---

Ejemplos.

```text
HAS_NEGATIVE_VALUES

HAS_ZERO_VALUES

HAS_NULL_VALUES
```

---

# 37. Temporal Density

Nueva feature.

---

Ejemplo.

```text
DAILY

MONTHLY

YEARLY
```

---

# 38. Why?

Chart Engine lo usará.

---

# 39. Ranking Features

---

Ejemplos.

```text
TOP_N_ANALYSIS

BOTTOM_N_ANALYSIS
```

---

# 40. Example

```sql
ORDER BY revenue DESC
LIMIT 5
```

↓

```text
TOP_N_ANALYSIS
```

---

# 41. Correlation Features

---

Ejemplo.

```sql
SELECT
 revenue,
 profit
```

↓

```text
MULTI_METRIC_ANALYSIS
```

---

# 42. Statistical Features

Extraídas del resultado.

---

# 43. Ejemplos

```text
ROW_COUNT

COLUMN_COUNT

DISTINCT_RATIO

NULL_RATIO
```

---

# 44. Data Profile Features

Muy valiosas.

---

```text
SKEWED_DISTRIBUTION

BALANCED_DISTRIBUTION

SPARSE_DATA
```

---

# 45. Feature Registry

DuckDB.

```sql
CREATE TABLE features (

    feature_id VARCHAR,

    execution_id VARCHAR,

    feature_name VARCHAR,

    feature_value VARCHAR,

    confidence DOUBLE
);
```

---

# 46. Feature Groups

Agrupar.

```text
SQL Features

Data Features

Statistical Features
```

---

# 47. Explainability

Toda feature genera explicación.

---

# 48. Example

```text
HAS_GROUP_BY
```

↓

```text
GROUP BY clause detected
```

---

# 49. Feature Confidence

Parser Features.

```text
1.0
```

---

porque son determinísticas.

---

# 50. Statistical Features

Pueden tener.

```text
0.8

0.9

0.95
```

---

# 51. Feature Naming Rule

Siempre.

```text
UPPERCASE

VERBO_IMPLÍCITO
```

---

Ejemplos.

```text
HAS_GROUP_BY

HAS_JOIN

HAS_LIMIT
```

---

# 52. No Naming

Evitar.

```text
GroupByFeature

JoinFeature
```

---

# 53. Feature Families

Preparar.

```python
feature.namespace
```

---

Ejemplo.

```text
sql.has_group_by

sql.has_join

data.low_cardinality
```

---

# 54. Feature Store

Futuro.

---

Cachear features.

---

Por fingerprint.

---

# 55. Feature Reuse

Si fingerprint igual.

↓

No recalcular.

---

# 56. Benchmark Metric

Medir.

```text
Feature Coverage
```

---

# 57. Fórmula

```python
coverage =
 detected_features
 /
 possible_features
```

---

# 58. Success Metric

Feature Engine debe capturar.

```text
90%+
```

de señales relevantes.

---

# 59. Strategic Insight

La mayoría de motores BI saltan directamente:

```text
SQL
 ↓
Chart
```

---

SQLviz hace:

```text
SQL
 ↓
Features
 ↓
Semantics
 ↓
Intent
 ↓
Chart
```

---

Y esa capa intermedia es donde aparece gran parte de la inteligencia.

---

# 60. Principio Fundamental

El Feature Engine convierte SQL en señales.

No interpreta.

No decide.

No concluye.

Solo observa y describe.

Y esas observaciones son la materia prima de todos los motores inteligentes que vienen después.