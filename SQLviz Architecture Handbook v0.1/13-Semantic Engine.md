# SQLviz Architecture Handbook

## Capítulo 13 — Semantic Engine

Versión: 0.1

---

# 1. Introducción

Hasta ahora SQLviz entiende:

```text
Estructura SQL
Tipos de columnas
Cardinalidad
Agregaciones
Relaciones
Intenciones
```

Pero todavía existe un problema.

---

Estas consultas:

```sql
SELECT
    SUM(revenue)
FROM sales
```

```sql
SELECT
    SUM(sales)
FROM orders
```

```sql
SELECT
    SUM(income)
FROM transactions
```

son diferentes estructuralmente.

---

Pero semánticamente pueden significar:

```text
Ingresos
```

---

El Semantic Engine existe para resolver este problema.

---

# 2. Objetivo

Transformar:

```text
Column Names
Table Names
Aliases
```

en:

```text
Business Meaning
```

---

# 3. Filosofía

El usuario escribe SQL.

---

SQLviz debe entender:

```text
Qué representa cada cosa
```

no solamente:

```text
Cómo está escrita
```

---

# 4. Arquitectura

```text
SQL
 │
 ▼
AST
 │
 ▼
Semantic Extraction
 │
 ▼
Entity Resolution
 │
 ▼
Semantic Registry
 │
 ▼
Business Concepts
```

---

# 5. Semantic Layers

SQLviz trabaja en cuatro capas.

---

## Physical Layer

```text
revenue
sales
income
```

---

## Canonical Layer

```text
BUSINESS_REVENUE
```

---

## Concept Layer

```text
Revenue
```

---

## Intent Layer

```text
Growth Analysis
Profitability Analysis
```

---

# 6. Semantic Categories

Categorías base.

```text
Revenue
Cost
Profit
Margin
Order
Customer
Product
Time
Location
Channel
Inventory
Marketing
Finance
Operations
```

---

# 7. Metric Concepts

Ejemplos.

---

```text
revenue
sales
income
gross_sales
```

↓

```text
BUSINESS_REVENUE
```

---

```text
cost
expenses
spend
```

↓

```text
BUSINESS_COST
```

---

# 8. Dimension Concepts

Ejemplos.

---

```text
country
nation
market
```

↓

```text
GEO_COUNTRY
```

---

```text
city
town
municipality
```

↓

```text
GEO_CITY
```

---

# 9. Semantic Registry

DuckDB.

```sql
CREATE TABLE semantic_registry (

    term VARCHAR,

    canonical_term VARCHAR,

    semantic_type VARCHAR,

    confidence DOUBLE
);
```

---

# 10. Initial Registry

Archivo YAML.

```yaml
revenue:
  canonical: BUSINESS_REVENUE

sales:
  canonical: BUSINESS_REVENUE

income:
  canonical: BUSINESS_REVENUE
```

---

# 11. Alias Resolution

Consulta:

```sql
SELECT
    SUM(revenue) AS sales
```

---

Detectar.

```text
sales
```

↓

```text
BUSINESS_REVENUE
```

---

# 12. Table Semantics

Tablas también tienen significado.

---

Ejemplo:

```text
sales
orders
transactions
```

↓

```text
FACT_REVENUE
```

---

# 13. Entity Resolution

Resolver equivalencias.

---

Ejemplo:

```text
customer
client
account
```

↓

```text
BUSINESS_CUSTOMER
```

---

# 14. Fuzzy Matching

Implementación.

---

Bibliotecas:

```text
RapidFuzz
difflib
```

---

Ejemplo.

```text
custmer
```

↓

```text
customer
```

---

# 15. Semantic Similarity

Función.

```python
semantic_similarity(
    "sales",
    "revenue"
)
```

↓

```text
0.95
```

---

# 16. Dictionary Score

Fórmula.

```python
dictionary_score =
matched_terms
/
total_terms
```

---

# 17. Semantic Confidence

Combinar:

```text
Dictionary Match
Alias Match
Fuzzy Match
Historical Match
```

---

```python
confidence =
(
 dictionary * 0.4
 +
 alias * 0.3
 +
 fuzzy * 0.2
 +
 history * 0.1
)
```

---

# 18. Business Entity Model

```python
class SemanticEntity:

    name: str

    canonical_name: str

    semantic_type: str

    confidence: float
```

---

# 19. Semantic Graph

Representación.

```text
Revenue
  ├─ sales
  ├─ income
  ├─ revenue
  └─ gross_sales
```

---

# 20. Semantic Edges

DuckDB.

```sql
CREATE TABLE semantic_edges (

    source_term VARCHAR,

    target_term VARCHAR,

    similarity DOUBLE
);
```

---

# 21. Metric Families

Ejemplo.

```text
Revenue
├─ Gross Revenue
├─ Net Revenue
├─ Revenue Growth
└─ Revenue Forecast
```

---

# 22. Dimension Families

Ejemplo.

```text
Geography
├─ Country
├─ Region
├─ State
└─ City
```

---

# 23. Time Families

Ejemplo.

```text
Time
├─ Year
├─ Quarter
├─ Month
├─ Week
└─ Day
```

---

# 24. Semantic Fingerprints

Antes:

```text
SUM(revenue)
```

---

Después:

```text
SUM(BUSINESS_REVENUE)
```

---

Permite fingerprints más robustos.

---

# 25. Semantic Intents

Ejemplo.

```text
BUSINESS_REVENUE
+
TIME
```

↓

```text
Revenue Trend
```

---

# 26. Semantic Filters

Consulta:

```sql
GROUP BY country
```

---

Detectar:

```text
GEO_COUNTRY
```

---

Sugerir filtro.

---

# 27. Semantic Relationships

Ejemplo.

```text
Revenue
```

relacionado con:

```text
Profit
Margin
Cost
Orders
```

---

# 28. Semantic Recommendation

Revenue detectado.

↓

Composer puede generar.

```text
Revenue KPI
Revenue Trend
Revenue by Geography
```

---

# 29. Semantic Learning

Usuario corrige.

---

Ejemplo.

```text
sales
```

↓

```text
BUSINESS_REVENUE
```

---

Guardar.

---

# 30. Semantic Feedback

```sql
CREATE TABLE semantic_feedback (

    term VARCHAR,

    canonical_term VARCHAR,

    accepted BOOLEAN
);
```

---

# 31. Semantic Score Update

Actualizar confianza.

---

```python
new_score =
old_score
+
learning_rate
*
feedback
```

---

# 32. Semantic API

FastAPI.

```python
GET /semantic/search
```

---

```python
POST /semantic/register
```

---

```python
GET /semantic/concepts
```

---

# 33. Semantic Registry Cache

DuckDB.

```sql
CREATE TABLE semantic_cache (

    term VARCHAR,

    canonical_term VARCHAR
);
```

---

# 34. Explainability

Toda inferencia debe explicar.

---

Ejemplo.

```json
{
  "term":"sales",
  "canonical":"BUSINESS_REVENUE",
  "confidence":0.94
}
```

---

# 35. Semantic Expansion

Consulta:

```sql
SUM(revenue)
```

---

Motor descubre.

```text
profit
margin
cost
orders
```

---

Esto alimenta:

```text
Dashboard Composer
```

---

# 36. Semantic Layers in SQLviz

```text
Physical SQL
      │
      ▼
Semantic Engine
      │
      ▼
Canonical Concepts
      │
      ▼
Intent Engine
      │
      ▼
Dashboard Composer
```

---

# 37. Roadmap

V1

```text
Dictionary Based
```

---

V2

```text
Graph Based
```

---

V3

```text
Embedding Based
(local)
```

---

V4

```text
Hybrid Semantic Engine
```

---

# 38. La Idea Más Importante

La mayoría de motores BI entienden:

```text
column names
```

---

SQLviz debe entender:

```text
business concepts
```

---

Porque:

```text
sales
revenue
income
```

no son tres columnas.

Son una misma idea.

---

# 39. Principio Fundamental

El Semantic Engine transforma estructuras técnicas en conceptos de negocio.

Es la capa que permite que SQLviz razone sobre significado en lugar de nombres, y constituye el puente entre SQL y conocimiento analítico.

Sin este motor, SQLviz puede visualizar datos.

Con este motor, SQLviz puede empezar a comprenderlos.
