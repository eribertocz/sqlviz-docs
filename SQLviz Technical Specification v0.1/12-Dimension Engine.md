# SQLviz Technical Specification v1

## Capítulo 12 — Dimension Engine

Versión 0.1

---

# 1. Objetivo

Identificar automáticamente los ejes analíticos sobre los que se analizan las métricas.

---

Si Metric Engine responde:

```text
¿Qué estamos midiendo?
```

---

Dimension Engine responde:

```text
¿Respecto a qué lo estamos midiendo?
```

---

# 2. Filosofía

Las dimensiones representan contexto.

---

Las métricas representan magnitud.

---

Ejemplo.

```sql
SELECT
 country,
 SUM(revenue)
```

---

Resultado.

```text
Metric:
Revenue

Dimension:
Country
```

---

# 3. Input

```python
context.features

context.semantic_tags

context.ast

context.metrics
```

---

# 4. Output

```python
context.dimensions
```

---

# 5. Dimension Model

```python
class Dimension:

    dimension_id: str

    name: str

    semantic_tag: str

    cardinality: int

    datatype: str

    confidence: float

    explanation: Explanation
```

---

# 6. Regla Principal

Toda columna NO métrica es candidata a dimensión.

---

# 7. Candidatos Comunes

```text
Country

Product

Customer

Segment

Channel

Date
```

---

# 8. Ejemplo

```sql
SELECT
 country,
 SUM(revenue)
```

↓

```text
Country
```

↓

```text
Dimension
```

---

# 9. Semantic Categories

Fuertes candidatas.

```text
Location

Entity

Temporal

Category
```

---

# 10. Location Dimensions

```text
Country

Region

City

State
```

---

# 11. Product Dimensions

```text
Product

Category

Brand

SKU
```

---

# 12. Customer Dimensions

```text
Customer

Segment

Industry
```

---

# 13. Temporal Dimensions

Especiales.

---

# 14. Ejemplos

```text
Date

Month

Quarter

Year
```

---

# 15. Why?

Trend Analysis.

---

Line Charts.

---

Forecasting.

---

# 16. Dimension Score

Modelo inicial.

```python
score =
(
 semantic_score * 0.4
 +
 cardinality_score * 0.3
 +
 query_role_score * 0.3
)
```

---

# 17. Semantic Score

```text
Country = 1.0

Product = 1.0

Date = 1.0

Revenue = 0.0
```

---

# 18. Cardinality Score

Muy importante.

---

# 19. Low Cardinality

```text
< 10
```

↓

```text
Excelente dimensión
```

---

# 20. Medium Cardinality

```text
10-100
```

↓

```text
Buena dimensión
```

---

# 21. High Cardinality

```text
100+
```

↓

```text
Posible dimensión
```

---

# 22. Very High Cardinality

```text
10000+
```

↓

```text
Posible identificador
```

---

# 23. Query Role Score

Analiza dónde aparece.

---

# 24. Ejemplo

```sql
GROUP BY country
```

↓

```text
Country
```

↓

```text
Dimension Score +++
```

---

# 25. Temporal Priority

Las dimensiones temporales tienen prioridad especial.

---

# 26. Why?

Porque suelen dominar:

```text
Intent

Chart

Layout
```

---

# 27. Temporal Ranking

```text
Date       1.0

Month      0.95

Quarter    0.90

Year       0.85
```

---

# 28. Dimension Categories

V1.

```text
Temporal

Location

Product

Customer

Organization

Category

Identifier
```

---

# 29. Identifier Detection

Crítico.

---

Ejemplos.

```text
customer_id

order_id

invoice_id
```

---

# 30. Rule

Identificadores NO deben dominar charts.

---

# 31. Example

```sql
SELECT
 customer_id,
 SUM(revenue)
GROUP BY customer_id
```

---

Resultado.

```text
Identifier Dimension

High Cardinality Warning
```

---

# 32. Dimension Registry

DuckDB.

```sql
CREATE TABLE dimensions (

    dimension_id VARCHAR,

    execution_id VARCHAR,

    dimension_name VARCHAR,

    dimension_type VARCHAR,

    confidence DOUBLE
);
```

---

# 33. Primary Dimension

Siempre identificar.

---

# 34. Example

```sql
SELECT
 country,
 product,
 SUM(revenue)
```

---

Resultado.

```text
Country 0.95

Product 0.88
```

---

Primary.

```text
Country
```

---

# 35. Dimension Priority Formula

```python
score =
(
 semantic_weight * 0.4
 +
 cardinality_weight * 0.3
 +
 groupby_weight * 0.3
)
```

---

# 36. Multi-Dimension Analysis

Detectar.

---

Ejemplo.

```sql
SELECT
 country,
 product,
 SUM(revenue)
GROUP BY
 country,
 product
```

---

Resultado.

```text
Multi Dimension Analysis
```

---

# 37. Hierarchical Dimensions

Detectar.

---

Ejemplo.

```text
Country

Region

City
```

---

# 38. Why?

Drilldowns futuros.

---

# 39. Time Hierarchies

Especiales.

---

```text
Year

Quarter

Month

Day
```

---

# 40. Composite Dimensions

Generar.

---

Ejemplo.

```text
Country + Product
```

↓

```text
Composite Dimension
```

---

# 41. Explainability

Siempre.

---

Ejemplo.

```text
Country

Reason:

Location semantic detected

GROUP BY detected

Low cardinality detected
```

---

# 42. Event

Publicar.

```text
DIMENSIONS_INFERRED
```

---

# 43. Payload

```python
{
  "dimension_count": 2,
  "primary_dimension": "Country"
}
```

---

# 44. Dimension Warnings

Ejemplos.

```text
HIGH_CARDINALITY_DIMENSION

IDENTIFIER_DIMENSION

SPARSE_DIMENSION
```

---

# 45. Cache Strategy

Clave.

```python
semantic_tag

cardinality

datatype
```

---

# 46. Benchmark Corpus

```text
Country

Product

Customer

Segment

Channel

Date
```

---

# 47. Accuracy Goal

```text
98%+
```

para dimensiones comunes.

---

# 48. Strategic Insight

Muchos sistemas detectan dimensiones usando:

```text
Datatype
```

---

SQLviz debe usar:

```text
Semantics

Cardinality

Role in Query
```

---

# 49. Future V2

```text
Dimension Catalog

Business Hierarchies

Custom Dimensions
```

---

# 50. Future V3

```text
Dimension Learning

Dimension Graph

Cross Dashboard Relationships
```

---

# 51. Principio Fundamental

El Metric Engine identifica qué medir.

El Dimension Engine identifica cómo segmentar esa medición.

Juntos forman el núcleo analítico de SQLviz.
