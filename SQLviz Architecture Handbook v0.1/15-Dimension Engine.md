# SQLviz Architecture Handbook

## Capítulo 15 — Dimension Engine

Versión: 0.1

---

# 1. Introducción

Hasta ahora SQLviz entiende:

```text
Qué significa algo
    ↓
Semantic Engine

Qué se está midiendo
    ↓
Metric Engine
```

Pero todavía falta responder una pregunta fundamental:

```text
¿Desde qué perspectiva se está observando?
```

---

Ejemplo:

```sql
SELECT
    country,
    SUM(revenue)
FROM sales
GROUP BY country
```

---

La métrica es:

```text
Revenue
```

---

La dimensión es:

```text
Country
```

---

La dimensión define el contexto.

---

# 2. Objetivo

Transformar:

```text
country
product
customer
month
region
```

en:

```text
Dimensiones Analíticas
+
Jerarquías
+
Relaciones
+
Navegación
```

---

# 3. Filosofía

Las métricas responden:

```text
¿Cuánto?
```

---

Las dimensiones responden:

```text
¿Cuándo?
¿Dónde?
¿Quién?
¿Qué?
¿Cómo?
```

---

Sin dimensiones no existe análisis.

---

# 4. Arquitectura

```text
SQL
 │
 ▼
Semantic Engine
 │
 ▼
Dimension Detection
 │
 ▼
Dimension Registry
 │
 ▼
Dimension Graph
 │
 ▼
Dimension Intelligence
```

---

# 5. ¿Qué es una Dimensión?

Formalmente:

```python
class Dimension:

    id: str

    name: str

    semantic_type: str

    hierarchy_level: int

    cardinality: int
```

---

# 6. Tipos de Dimensiones

SQLviz define:

```text
TIME

GEOGRAPHY

PRODUCT

CUSTOMER

ORGANIZATION

CHANNEL

MARKETING

CUSTOM
```

---

# 7. Time Dimensions

Ejemplos:

```text
date
year
quarter
month
week
day
hour
```

---

Canonical:

```text
DIM_TIME
```

---

# 8. Geography Dimensions

Ejemplos:

```text
country
region
state
city
territory
market
```

---

Canonical:

```text
DIM_GEO
```

---

# 9. Product Dimensions

Ejemplos:

```text
category
subcategory
product
sku
brand
```

---

Canonical:

```text
DIM_PRODUCT
```

---

# 10. Customer Dimensions

Ejemplos:

```text
customer
client
account
segment
```

---

Canonical:

```text
DIM_CUSTOMER
```

---

# 11. Dimension Registry

DuckDB.

```sql
CREATE TABLE dimension_registry (

    dimension_name VARCHAR,

    semantic_type VARCHAR,

    hierarchy_type VARCHAR,

    confidence DOUBLE
);
```

---

# 12. Cardinality Analysis

Una dimensión se caracteriza por cardinalidad.

---

Ejemplo:

```text
Country
```

↓

```text
15 valores
```

---

```text
Customer
```

↓

```text
2,000,000 valores
```

---

Muy diferente.

---

# 13. Cardinality Classes

```text
LOW
   2-20

MEDIUM
   20-200

HIGH
   200-5000

VERY_HIGH
   5000+
```

---

# 14. Dimension Score

```python
dimension_score =
(
 semantic_confidence * 0.4
 +
 usage_frequency * 0.3
 +
 hierarchy_strength * 0.3
)
```

---

# 15. Dimension Hierarchies

Concepto crítico.

---

Tiempo.

```text
Year
 └── Quarter
      └── Month
           └── Day
```

---

# 16. Geography Hierarchy

```text
Country
 └── State
      └── City
```

---

# 17. Product Hierarchy

```text
Category
 └── Subcategory
      └── Product
           └── SKU
```

---

# 18. Hierarchy Registry

```sql
CREATE TABLE dimension_hierarchies (

    parent_dimension VARCHAR,

    child_dimension VARCHAR,

    hierarchy_type VARCHAR
);
```

---

# 19. Hierarchy Discovery

Fuentes:

```text
Semantic Dictionary

Column Names

Cardinality

Foreign Keys

Observed Patterns
```

---

# 20. Drill-Down Detection

Ejemplo.

```sql
GROUP BY country
```

---

SQLviz descubre:

```text
country
 ↓
state
 ↓
city
```

---

Automáticamente.

---

# 21. Drill-Up Detection

Ejemplo.

```sql
GROUP BY city
```

---

SQLviz descubre:

```text
city
 ↑
state
 ↑
country
```

---

# 22. Dimension Graph

Representación.

```text
Country
 │
 ├── State
 │
 └── City
```

---

# 23. Dimension Edges

DuckDB.

```sql
CREATE TABLE dimension_edges (

    source_dimension VARCHAR,

    target_dimension VARCHAR,

    relationship_type VARCHAR,

    confidence DOUBLE
);
```

---

# 24. Shared Dimension Detection

Panel A

```sql
GROUP BY country
```

---

Panel B

```sql
GROUP BY country, product
```

---

Shared:

```text
country
```

---

Esto alimenta:

```text
Relation Engine
```

---

# 25. Smart Filter Discovery

Dimensiones candidatas.

---

Ejemplo.

```text
Country
Region
Category
Year
```

↓

Filtros globales.

---

# 26. Filter Priority Score

```python
priority =
(
 dimension_score * 0.4
 +
 relation_strength * 0.4
 +
 cardinality_score * 0.2
)
```

---

# 27. Cross Filter Intelligence

Si usuario selecciona:

```text
Country = Bolivia
```

---

SQLviz sabe afectar:

```text
Revenue
Orders
Profit
Customers
```

---

Porque comparten dimensión.

---

# 28. Dimension Similarity

Ejemplo.

```text
country
nation
market
```

↓

```text
DIM_COUNTRY
```

---

# 29. Semantic Dimension Families

```text
Geography
 ├── Country
 ├── Region
 ├── State
 └── City
```

---

# 30. Time Intelligence

Si existe:

```text
Month
```

---

SQLviz sabe generar:

```text
MoM

Rolling Average

Growth

Forecast
```

---

Sin configuración.

---

# 31. Dimension Recommendation

Ejemplo.

Usuario:

```sql
GROUP BY month
```

---

SQLviz sugiere:

```text
country
product
customer
```

---

Como dimensiones alternativas.

---

# 32. Dimension Expansion

Input.

```text
Revenue by Country
```

---

Output.

```text
Revenue by Region

Revenue by City

Revenue by Product
```

---

# 33. Query Rewrite Support

Query:

```sql
GROUP BY country
```

---

Rewrite:

```sql
GROUP BY city
```

---

O:

```sql
GROUP BY region
```

---

# 34. Dashboard Composition Support

Composer pregunta:

```text
¿Qué otras dimensiones existen?
```

---

Dimension Engine responde.

```text
Country

Product

Customer

Channel
```

---

# 35. Learning

Registrar.

```text
Drill-down usado

Filtro usado

Dimensión ignorada
```

---

# 36. Feedback Table

```sql
CREATE TABLE dimension_feedback (

    dimension_name VARCHAR,

    accepted BOOLEAN,

    timestamp TIMESTAMP
);
```

---

# 37. Explainability

Ejemplo.

```json
{
  "dimension":"country",
  "type":"geography",
  "confidence":0.96,
  "hierarchy":"country > state > city"
}
```

---

# 38. Dimension Centrality

Dimensión usada por:

```text
Revenue

Orders

Profit

Customers
```

↓

Alta centralidad.

---

# 39. Dimension Importance

```python
importance =
(
 centrality * 0.4
 +
 usage_frequency * 0.3
 +
 hierarchy_strength * 0.3
)
```

---

# 40. Dimension Knowledge Graph

Representación final.

```text
DIM_GEO

 ├── Country
 │
 ├── State
 │
 └── City


DIM_TIME

 ├── Year
 │
 ├── Quarter
 │
 ├── Month
 │
 └── Day
```

---

# 41. Roadmap

V1

```text
Dictionary-Based
```

---

V2

```text
Hierarchy Discovery
```

---

V3

```text
Dimension Graph
```

---

V4

```text
Automatic Navigation
```

---

# 42. La Idea Más Importante

La mayoría de herramientas BI consideran una dimensión como:

```text
Una columna agrupada
```

---

SQLviz debe considerarla:

```text
Un espacio de navegación analítica
```

---

Porque:

```text
Country
```

no es una columna.

Es una puerta de entrada a:

```text
State
City
Region
Market
```

---

# 43. Principio Fundamental

El Dimension Engine transforma columnas en perspectivas analíticas.

Es la pieza que permite exploración automática, drill-down inteligente, filtros contextuales y generación de nuevas preguntas analíticas.

Sin él, SQLviz sabe qué medir.

Con él, SQLviz sabe desde dónde observar.