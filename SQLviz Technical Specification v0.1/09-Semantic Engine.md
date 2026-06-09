# SQLviz Technical Specification v1

## Capítulo 09 — Semantic Engine

Versión 0.1

---

# 1. Objetivo

Transformar columnas y expresiones SQL en conceptos de negocio.

---

Hasta ahora SQLviz ve:

```text
revenue
```

---

Después del Semantic Engine debe ver:

```text
Revenue
↓
Financial Metric
↓
Business KPI
```

---

# 2. Filosofía

El Parser entiende SQL.

---

El Feature Engine entiende estructura.

---

El Semantic Engine entiende significado.

---

# 3. Importancia

Este es probablemente el motor más valioso de SQLviz.

---

Porque diferencia:

```text
Herramienta BI
```

de

```text
Sistema Analítico
```

---

# 4. Input

```python
RuntimeContext
```

---

Contiene:

```python
features

columns

aliases

aggregations
```

---

# 5. Output

Añade.

```python
semantic_tags
```

---

# 6. Semantic Tag

Representa significado.

---

# 7. Modelo

```python
class SemanticTag:

    tag_id: str

    name: str

    category: str

    confidence: float

    explanation: Explanation
```

---

# 8. Categorías V1

```text
Metric

Dimension

Temporal

Location

Entity

Identifier

Financial

Operational
```

---

# 9. Ejemplos

```text
revenue
```

↓

```text
Metric

Financial

Revenue
```

---

# 10. Ejemplo

```text
country
```

↓

```text
Dimension

Location

Country
```

---

# 11. Semantic Sources

V1.

```text
Column Names

Aliases

SQL Expressions

Data Types
```

---

# 12. Semantic Dictionaries

Base principal.

---

# 13. Revenue Dictionary

```python
{
 "revenue",
 "sales",
 "income",
 "billing",
 "turnover"
}
```

---

# 14. Profit Dictionary

```python
{
 "profit",
 "margin",
 "earnings"
}
```

---

# 15. Customer Dictionary

```python
{
 "customer",
 "client",
 "buyer",
 "consumer"
}
```

---

# 16. Location Dictionary

```python
{
 "country",
 "city",
 "region",
 "state"
}
```

---

# 17. Date Dictionary

```python
{
 "date",
 "day",
 "month",
 "year",
 "quarter"
}
```

---

# 18. Registry

DuckDB.

```sql
CREATE TABLE semantic_dictionary (

    term VARCHAR,

    semantic_tag VARCHAR,

    confidence DOUBLE
);
```

---

# 19. Matching Types

```text
Exact Match

Normalized Match

Fuzzy Match

Alias Match
```

---

# 20. Exact Match

```text
revenue
```

↓

```text
Revenue
```

---

Confidence.

```text
1.0
```

---

# 21. Normalized Match

```text
total_revenue
```

↓

```text
Revenue
```

---

# 22. Fuzzy Match

```text
revenues
```

↓

```text
Revenue
```

---

# 23. Alias Match

```sql
SUM(amount) revenue
```

---

Alias domina.

---

# 24. Semantic Scoring

Modelo.

```python
score =
(
 exact_match * 0.5
 +
 alias_match * 0.3
 +
 datatype_match * 0.2
)
```

---

# 25. Multiple Tags

Permitido.

---

Ejemplo.

```text
customer_country
```

↓

```text
Country 0.95

Location 0.92

Customer Attribute 0.75
```

---

# 26. Semantic Categories

Primer nivel.

---

# 27. Metric Tags

```text
Revenue

Profit

Cost

Quantity

Price
```

---

# 28. Dimension Tags

```text
Country

Product

Customer

Segment

Channel
```

---

# 29. Temporal Tags

```text
Date

Month

Quarter

Year
```

---

# 30. Financial Tags

```text
Revenue

Profit

Margin

Cost
```

---

# 31. Operational Tags

```text
Orders

Inventory

Shipment

Lead Time
```

---

# 32. Identifier Tags

```text
ID

UUID

Key

Code
```

---

# 33. ID Detection

Muy importante.

---

Ejemplos.

```text
customer_id

product_id

order_id
```

---

Generan.

```text
Identifier
```

---

# 34. Why?

Chart Engine debe evitar.

```text
Pie por customer_id
```

---

# 35. Semantic Hierarchy

Ejemplo.

```text
Revenue
```

↓

```text
Metric
```

↓

```text
Financial
```

---

# 36. Semantic Graph

Modelo futuro.

---

```text
Revenue

├─ Financial

├─ KPI

└─ Metric
```

---

# 37. Column Profile Integration

Usar.

```text
Cardinality

Distinct Ratio

Datatype
```

---

# 38. Example

```text
customer_id
```

*

```text
High Cardinality
```

↓

```text
Identifier
```

---

# 39. Temporal Detection

No depender solo del nombre.

---

Usar también.

```text
DATE

TIMESTAMP
```

---

# 40. Example

```text
created_at
```

↓

```text
Date
```

---

aunque el nombre no diga "date".

---

# 41. Expression Semantics

Analizar expresiones.

---

# 42. Example

```sql
SUM(revenue)
```

↓

```text
Revenue Metric
```

---

# 43. Example

```sql
COUNT(customer_id)
```

↓

```text
Customer Count
```

---

# 44. Composite Semantics

Generar.

---

Ejemplo.

```text
Revenue By Country
```

---

a partir de.

```text
Revenue

Country
```

---

# 45. Semantic Registry

DuckDB.

```sql
CREATE TABLE semantic_tags (

    tag_id VARCHAR,

    execution_id VARCHAR,

    tag_name VARCHAR,

    confidence DOUBLE
);
```

---

# 46. Semantic Relationships

DuckDB.

```sql
CREATE TABLE semantic_relations (

    source_tag VARCHAR,

    target_tag VARCHAR,

    relation_type VARCHAR
);
```

---

# 47. Relation Types

```text
Parent

Child

Related

Equivalent
```

---

# 48. Explainability

Siempre.

---

Ejemplo.

```text
Revenue
```

↓

```text
Matched dictionary term

Financial metric detected

Alias confidence high
```

---

# 49. Semantic Event

Publicar.

```text
SEMANTICS_INFERRED
```

---

# 50. Payload

```python
{
   "tags":[
      "Revenue",
      "Country"
   ]
}
```

---

# 51. Performance Goal

```text
< 100 ms
```

---

# 52. Cache Strategy

Clave.

```python
column_name
```

*

```python
datatype
```

---

# 53. Cache Example

```text
revenue
```

↓

```text
Revenue
```

---

No recalcular.

---

# 54. Benchmark Corpus

Casos mínimos.

```text
Revenue

Profit

Country

Customer

Product

Date
```

---

# 55. Accuracy Goal

Objetivo inicial.

```text
95%+
```

para conceptos comunes.

---

# 56. Strategic Insight

La mayoría de herramientas BI ven:

```text
Columnas
```

---

SQLviz debe ver:

```text
Conceptos
```

---

# 57. Why It Matters

Intent Engine utilizará.

```text
Revenue
```

---

No.

```text
revenue
```

---

Chart Engine utilizará.

```text
Country
```

---

No.

```text
country
```

---

# 58. Future Evolution

V2.

```text
Business Ontologies

Industry Dictionaries

Custom Semantic Packs
```

---

# 59. Future Evolution

V3.

```text
Semantic Learning

Knowledge Graph

Cross Dashboard Semantics
```

---

# 60. Principio Fundamental

El Semantic Engine es donde SQLviz deja de procesar texto y empieza a entender significado.

A partir de este punto, el sistema ya no trabaja con nombres de columnas.

Trabaja con conceptos de negocio.