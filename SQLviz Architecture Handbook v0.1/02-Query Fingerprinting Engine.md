# SQLviz Architecture Handbook

## Capítulo 2 — Query Fingerprinting Engine

Versión: 0.1

---

# 1. Introducción

El Query Fingerprinting Engine es probablemente el componente más importante de SQLviz.

Sin él:

* no existe aprendizaje
* no existe memoria
* no existe reutilización de patrones
* no existe inteligencia colectiva

El problema fundamental es que dos consultas SQL distintas pueden representar exactamente la misma intención analítica.

Ejemplo:

```sql
SELECT
    month,
    SUM(revenue)
FROM sales
GROUP BY month
```

```sql
SELECT
    fecha,
    SUM(monto)
FROM fact_ventas
GROUP BY fecha
```

Visualmente son diferentes.

Semánticamente son iguales.

Ambas significan:

```text
Serie temporal agregada
```

El objetivo del Fingerprinting Engine es detectar esto.

---

# 2. Objetivo

Transformar cualquier consulta SQL en una representación canónica independiente de:

* nombres de tablas
* nombres de columnas
* idioma
* alias
* nombres de métricas

para producir:

```text
ANALYTICAL_PATTERN
```

estable y reutilizable.

---

# 3. Qué es un Fingerprint

Un fingerprint es una firma semántica.

No representa SQL.

Representa intención analítica.

Ejemplo:

```sql
SELECT region,
       SUM(sales)
FROM sales
GROUP BY region
```

Fingerprint:

```text
CATEGORY_AGG_SUM
```

---

Ejemplo:

```sql
SELECT region,
       COUNT(*)
FROM customers
GROUP BY region
```

Fingerprint:

```text
CATEGORY_AGG_COUNT
```

---

Ejemplo:

```sql
SELECT month,
       SUM(revenue)
FROM sales
GROUP BY month
```

Fingerprint:

```text
TIME_AGG_SUM
```

---

# 4. Arquitectura

```text
SQL
 │
 ▼
Parser
 │
 ▼
AST
 │
 ▼
Normalizer
 │
 ▼
Feature Extractor
 │
 ▼
Fingerprint Builder
 │
 ▼
Fingerprint
```

---

# 5. Parser

Responsabilidad:

Convertir SQL a AST.

Recomendación:

```python
sqlglot
```

porque:

* rápido
* multiplataforma
* soporta DuckDB
* soporta normalización

Ejemplo:

```python
import sqlglot

tree = sqlglot.parse_one(sql)
```

---

# 6. AST

SQL:

```sql
SELECT month,
       SUM(revenue)
FROM sales
GROUP BY month
```

AST simplificado:

```text
SELECT
 ├── month
 ├── SUM(revenue)
FROM
 └── sales
GROUP BY
 └── month
```

---

AST es la base de toda inferencia.

---

# 7. Canonicalización

Problema:

```sql
SELECT month
```

```sql
SELECT fecha
```

No deberían generar fingerprints distintos.

---

Regla:

Columnas se convierten a tipos semánticos.

```text
month
fecha
date
period
```

↓

```text
TIME_DIMENSION
```

---

Ejemplo:

```text
sales
revenue
monto
importe
```

↓

```text
METRIC
```

---

# 8. Semantic Dictionary

Tabla:

```sql
CREATE TABLE semantic_dictionary (
    token VARCHAR,
    category VARCHAR
);
```

---

Ejemplos:

```text
fecha      → TIME
date       → TIME
month      → TIME
year       → TIME

sales      → METRIC
revenue    → METRIC
profit     → METRIC

country    → GEO
region     → GEO
city       → GEO
```

---

# 9. Structural Features

El fingerprint no depende solamente de nombres.

También depende de estructura.

Ejemplo:

```sql
GROUP BY
```

produce:

```text
HAS_GROUP_BY
```

---

```sql
ORDER BY
```

produce:

```text
HAS_ORDER_BY
```

---

```sql
LIMIT
```

produce:

```text
HAS_LIMIT
```

---

```sql
OVER(...)
```

produce:

```text
HAS_WINDOW
```

---

# 10. Feature Vector

El fingerprint engine genera un vector.

Ejemplo:

```python
{
    "group_by": 1,
    "order_by": 1,
    "limit": 1,
    "window": 0,

    "time_dimension": 1,
    "category_dimension": 0,

    "sum_metric": 1,
    "count_metric": 0
}
```

---

# 11. Pattern Families

SQLviz no aprende millones de consultas.

Aprende familias.

---

## Familia Temporal

```text
TIME_AGG_SUM
TIME_AGG_COUNT
TIME_AGG_AVG
TIME_RANKING
TIME_COMPARISON
```

---

## Familia Comparación

```text
CATEGORY_AGG_SUM
CATEGORY_AGG_COUNT
CATEGORY_AVG
```

---

## Familia Ranking

```text
TOP_N_SUM
TOP_N_COUNT
TOP_N_AVG
```

---

## Familia KPI

```text
SINGLE_SUM
SINGLE_COUNT
SINGLE_AVG
```

---

## Familia Correlación

```text
NUMERIC_NUMERIC
```

---

## Familia Distribución

```text
CATEGORY_DISTRIBUTION
NUMERIC_DISTRIBUTION
```

---

# 12. Fingerprint Builder

Construcción:

```python
parts = []

if time_dimension:
    parts.append("TIME")

if category_dimension:
    parts.append("CATEGORY")

if aggregation == "SUM":
    parts.append("SUM")

if limit:
    parts.append("TOP")

fingerprint = "_".join(parts)
```

Resultado:

```text
TIME_SUM
```

---

# 13. Hashing

Cada fingerprint tiene hash.

```python
import hashlib

hashlib.sha1(
    fingerprint.encode()
)
```

---

Ejemplo:

```text
TIME_SUM
```

↓

```text
a8f6c2...
```

---

# 14. DuckDB Schema

Tabla principal.

```sql
CREATE TABLE fingerprints (
    fingerprint_id VARCHAR PRIMARY KEY,

    fingerprint VARCHAR,

    frequency INTEGER,

    first_seen TIMESTAMP,

    last_seen TIMESTAMP
);
```

---

# 15. Query History

```sql
CREATE TABLE query_history (
    query_hash VARCHAR,

    fingerprint_id VARCHAR,

    timestamp TIMESTAMP
);
```

---

# 16. Similarity Engine

Problema:

Dos fingerprints pueden ser parecidos.

Ejemplo:

```text
TIME_SUM
```

```text
TIME_AVG
```

No son iguales.

Pero sí similares.

---

Tabla:

```sql
CREATE TABLE fingerprint_similarity (
    source_id VARCHAR,
    target_id VARCHAR,
    similarity FLOAT
);
```

---

# 17. Similaridad

Ejemplo:

```python
similarity =
(
    matching_features
    /
    total_features
)
```

---

Resultado:

```text
TIME_SUM ↔ TIME_AVG

0.87
```

---

# 18. Aprendizaje

Cuando aparece una consulta nueva:

```sql
SELECT month,
       SUM(revenue)
FROM sales
GROUP BY month
```

SQLviz:

1. genera fingerprint
2. busca patrón existente
3. reutiliza aprendizaje previo

---

# 19. Integración con Intent Engine

Input:

```text
TIME_SUM
```

Intent probable:

```python
{
    "trend": 0.94
}
```

---

# 20. Integración con Chart Engine

Input:

```text
TIME_SUM
```

Charts preferidos:

```python
line  = 0.95
area  = 0.82
bar   = 0.31
```

---

# 21. Integración con Learning Engine

Si el usuario corrige:

```text
Line
```

↓

```text
Area
```

se actualiza:

```sql
chart_patterns
```

asociado al fingerprint.

---

# 22. Escalabilidad

Objetivo:

No almacenar SQL.

Almacenar patrones.

Ejemplo:

```text
100,000 consultas
```

↓

```text
300 fingerprints
```

---

# 23. Beneficio Principal

El Fingerprinting Engine transforma:

```text
SQL
```

en

```text
Conocimiento reutilizable
```

Ese conocimiento alimenta:

* Intent Engine
* Chart Engine
* Layout Engine
* Learning Engine
* Community Learning

Por eso el Fingerprinting Engine es la base de toda la inteligencia de SQLviz.
