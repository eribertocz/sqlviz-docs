# SQLviz Technical Specification v1

## Capítulo 07 — SQL Parser

Versión 0.1

---

# 1. Objetivo

Transformar SQL en una representación estructurada que todos los engines puedan consumir.

---

El Parser es el único componente que entiende SQL directamente.

---

Todos los demás engines trabajan sobre:

```text
AST

Features

Fingerprints
```

---

Nunca sobre SQL crudo.

---

# 2. Filosofía

Parser NO infiere.

---

Parser NO decide charts.

---

Parser NO detecta intención.

---

Parser solamente responde:

```text
¿Qué escribió el usuario?
```

---

# 3. Responsabilidades

V1.

```text
Parsear SQL

Generar AST

Extraer metadatos

Generar Fingerprints

Generar Features base
```

---

# 4. No Responsabilidades

```text
Chart Selection

Intent Detection

Semantic Detection

Layout Inference
```

---

# 5. Librería Recomendada

Mi recomendación:

[SQLGlot](https://github.com/tobymao/sqlglot?utm_source=chatgpt.com)

---

# 6. Por Qué SQLGlot

Ventajas.

```text
DuckDB Compatible

AST Excelente

Muy rápido

Open Source

Sin dependencias raras
```

---

# 7. Contract

```python
class ParserEngine(Engine):

    def execute(
        self,
        context: RuntimeContext
    ) -> RuntimeContext:
        ...
```

---

# 8. Input

```python
context.query.sql
```

---

# 9. Output

Añade.

```python
context.ast

context.fingerprint

context.features
```

---

# 10. AST Model

No guardar AST nativo.

---

Crear modelo propio.

---

# 11. AST Node

```python
class ASTNode:

    node_type: str

    children: list

    metadata: dict
```

---

# 12. Beneficio

Independencia de SQLGlot.

---

# 13. Parse Flow

```text
SQL
 ↓
SQLGlot
 ↓
AST
 ↓
Normalized AST
 ↓
Fingerprint
 ↓
Features
```

---

# 14. Query Classification

Primero detectar.

```text
SELECT

WITH

UNION

SUBQUERY
```

---

# 15. Query Type

```python
class QueryType(Enum):

    SELECT

    WITH

    UNION
```

---

# 16. SELECT Extraction

Extraer.

```text
Columnas

Aliases

Funciones
```

---

# 17. Example

```sql
SELECT
   month,
   SUM(revenue) revenue
```

---

Produce.

```python
[
  Column("month"),
  Aggregation(
      "SUM",
      "revenue"
  )
]
```

---

# 18. FROM Extraction

Extraer.

```text
Tablas

Views

Parquet

CSV
```

---

# 19. Example

```sql
FROM sales
```

---

```python
Source(
   type="table",
   name="sales"
)
```

---

# 20. WHERE Extraction

Extraer.

```text
Condiciones

Operadores

Variables
```

---

# 21. Example

```sql
WHERE country='AR'
```

---

Produce.

```python
FilterExpression(
    column="country",
    operator="="
)
```

---

# 22. GROUP BY Extraction

Crítico.

---

# 23. Example

```sql
GROUP BY month
```

---

Genera.

```python
Feature(
   "HAS_GROUP_BY"
)
```

---

# 24. ORDER BY Extraction

Extraer.

```text
Columna

Dirección

Posición
```

---

# 25. Example

```sql
ORDER BY revenue DESC
```

---

Genera.

```python
Feature(
  "HAS_ORDER_BY_DESC"
)
```

---

# 26. LIMIT Extraction

Extraer.

```python
LIMIT 10
```

---

Genera.

```python
Feature(
  "HAS_LIMIT"
)
```

---

# 27. Aggregation Extraction

Detectar.

```text
SUM

AVG

COUNT

MIN

MAX
```

---

# 28. Aggregation Model

```python
class Aggregation:

    function: str

    column: str
```

---

# 29. Window Functions

Detectar.

---

```sql
OVER (...)
```

---

# 30. Example

```sql
SUM(revenue)
OVER (...)
```

---

Genera.

```python
Feature(
   "HAS_WINDOW_FUNCTION"
)
```

---

# 31. CTE Detection

Detectar.

```sql
WITH sales_cte AS (...)
```

---

Genera.

```python
Feature(
   "HAS_CTE"
)
```

---

# 32. Join Detection

Extraer.

```text
INNER

LEFT

RIGHT

FULL
```

---

# 33. Join Model

```python
class Join:

    join_type: str

    left: str

    right: str
```

---

# 34. Join Features

```python
HAS_JOIN

HAS_LEFT_JOIN

HAS_MULTI_JOIN
```

---

# 35. Subquery Detection

Detectar.

```python
HAS_SUBQUERY
```

---

# 36. Set Operations

Detectar.

```text
UNION

INTERSECT

EXCEPT
```

---

# 37. Generated Features

Parser genera solamente features estructurales.

---

# 38. Examples

```text
HAS_SELECT

HAS_WHERE

HAS_GROUP_BY

HAS_ORDER_BY

HAS_LIMIT

HAS_JOIN

HAS_CTE
```

---

# 39. Parser Feature Registry

DuckDB.

```sql
CREATE TABLE parser_features (

    feature_name VARCHAR,

    description VARCHAR
);
```

---

# 40. Fingerprint Generation

Parte más importante.

---

# 41. Qué es un Fingerprint

Representación analítica.

---

No sintáctica.

---

# 42. Ejemplo

Estas dos consultas.

```sql
SELECT
 month,
 SUM(revenue)
GROUP BY month
```

---

```sql
SELECT
 date,
 SUM(profit)
GROUP BY date
```

---

Comparten.

```text
TIME_SERIES_AGGREGATION
```

---

# 43. Fingerprint Components

```text
Dimensions

Metrics

Aggregations

Filters

Ordering
```

---

# 44. Fingerprint Model

```python
class Fingerprint:

    dimensions: int

    metrics: int

    aggregations: int

    has_date: bool

    has_group_by: bool

    has_order_by: bool
```

---

# 45. Fingerprint Hash

Generar.

```python
sha256(...)
```

---

# 46. Fingerprint Cache

Clave principal.

---

```python
fingerprint_hash
```

---

# 47. Query Complexity

Calcular.

---

# 48. Complexity Score

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

# 49. Runtime Statistics

Añadir.

```python
context.statistics
```

---

# 50. Example

```python
{
   "join_count":2,
   "aggregation_count":1,
   "cte_count":1
}
```

---

# 51. Error Handling

Si SQL inválido.

---

Generar.

```python
context.errors.append(
   ...
)
```

---

# 52. Parser Event

Publicar.

```text
QUERY_PARSED
```

---

# 53. Payload

```python
{
  "fingerprint":"..."
}
```

---

# 54. Parser Explainability

Sí.

---

Incluso Parser.

---

# 55. Example

```text
GROUP BY detected

SUM detected

DATE column detected
```

---

# 56. Parser Performance Goal

```text
< 50 ms
```

---

# 57. Cache Strategy

SQL repetido.

↓

AST Cache.

---

# 58. Benchmark Cases

Corpus mínimo.

```text
Simple Select

Aggregation

Time Series

Ranking

Join

CTE

Subquery
```

---

# 59. Success Metric

```text
100% Parse Success
```

para SQL válido.

---

# 60. Principio Fundamental

El Parser no entiende negocios.

No entiende revenue.

No entiende charts.

No entiende dashboards.

Entiende únicamente estructura SQL.

Y esa estructura se convierte en la materia prima sobre la que trabajan todos los motores posteriores.
