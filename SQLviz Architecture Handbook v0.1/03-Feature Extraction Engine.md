# SQLviz Architecture Handbook

## Capítulo 3 — Feature Extraction Engine

Versión: 0.1

---

# 1. Introducción

El Feature Extraction Engine es el sistema nervioso de SQLviz.

Todos los motores posteriores dependen de él.

```text
Intent Engine
Chart Engine
Filter Engine
Relation Engine
Layout Engine
Learning Engine
```

no analizan SQL directamente.

Todos consumen un único objeto:

```python
FeatureVector
```

---

# 2. Objetivo

Transformar:

```sql
SELECT
    month,
    SUM(revenue)
FROM sales
GROUP BY month
```

en cientos de señales estructuradas.

Ejemplo:

```python
{
    "has_group_by": 1,
    "has_order_by": 0,

    "time_dimension": 1,
    "numeric_metric": 1,

    "trend_strength": 0.92,
    "outlier_ratio": 0.01,

    "cardinality": 24,

    "intent_hint_trend": 0.95
}
```

---

# 3. Arquitectura

```text
SQL
 │
 ▼
AST Parser
 │
 ▼
SQL Features
 │
 ├──── Schema Features
 │
 ├──── Data Features
 │
 ├──── Semantic Features
 │
 └──── Statistical Features
 │
 ▼
Feature Vector
 │
 ▼
Inference Engines
```

---

# 4. Categorías de Features

SQLviz extrae 5 grupos.

```text
1 SQL Features
2 Schema Features
3 Data Features
4 Statistical Features
5 Semantic Features
```

---

# 5. SQL Features

Información obtenida del AST.

---

## Aggregation Features

Detectar:

```sql
SUM()
COUNT()
AVG()
MIN()
MAX()
MEDIAN()
```

Vector:

```python
{
    "agg_sum": 1,
    "agg_count": 0,
    "agg_avg": 0,
}
```

---

## Clause Features

```sql
GROUP BY
ORDER BY
LIMIT
HAVING
DISTINCT
```

Vector:

```python
{
    "group_by": 1,
    "order_by": 1,
    "limit": 1,
    "having": 0
}
```

---

## Join Features

```sql
JOIN
LEFT JOIN
RIGHT JOIN
FULL JOIN
```

Vector:

```python
{
    "join_count": 3
}
```

---

## Window Features

```sql
OVER()
```

Vector:

```python
{
    "has_window": 1
}
```

---

## Case Features

```sql
CASE WHEN
```

Vector:

```python
{
    "has_case": 1
}
```

---

# 6. Schema Features

Obtenidas desde DuckDB.

---

## Column Types

Contar:

```text
DATE
TIMESTAMP
VARCHAR
INTEGER
DOUBLE
BOOLEAN
```

Vector:

```python
{
    "date_columns": 1,
    "varchar_columns": 2,
    "numeric_columns": 3
}
```

---

## Primary Keys

Detectar:

```text
id
customer_id
order_id
```

Vector:

```python
{
    "primary_key_count": 1
}
```

---

## Foreign Keys

Inferidas.

```python
{
    "foreign_key_count": 4
}
```

---

# 7. Cardinality Features

Extremadamente importantes.

---

Ejemplo:

```text
country
```

Cardinalidad:

```text
10
```

---

Ejemplo:

```text
customer
```

Cardinalidad:

```text
500000
```

---

Vector:

```python
{
    "cardinality": 10
}
```

---

Normalización:

```python
cardinality_score =
log(cardinality)
```

---

# 8. Data Features

Obtenidas sobre el resultado.

---

## Row Count

```python
{
    "rows": 24
}
```

---

## Column Count

```python
{
    "columns": 2
}
```

---

## Null Ratio

```python
{
    "null_ratio": 0.03
}
```

---

## Unique Ratio

```python
{
    "unique_ratio": 0.91
}
```

---

# 9. Statistical Features

Probablemente las más poderosas.

---

## Mean

```python
mean
```

---

## Median

```python
median
```

---

## Standard Deviation

```python
std
```

---

## Variance

```python
variance
```

---

## Range

```python
max - min
```

---

# 10. Distribution Features

---

## Skewness

Detecta asimetría.

```python
{
    "skewness": 1.8
}
```

---

## Kurtosis

Detecta colas.

```python
{
    "kurtosis": 4.2
}
```

---

## Entropy

Detecta diversidad.

```python
{
    "entropy": 0.87
}
```

---

# 11. Outlier Features

Usar IQR.

```python
Q1
Q3
IQR
```

---

Ratio:

```python
{
    "outlier_ratio": 0.04
}
```

---

Muy útil para:

```text
Anomaly Detection
```

---

# 12. Correlation Features

Si existen dos columnas numéricas.

---

Pearson:

```python
corr(x,y)
```

---

Vector:

```python
{
    "correlation": 0.91
}
```

---

Muy útil para:

```text
Scatter Plot
```

---

# 13. Trend Features

Una de las señales más importantes.

---

Ejemplo:

```text
month
sales
```

---

Calcular:

```python
r²
```

sobre regresión lineal.

---

Vector:

```python
{
    "trend_strength": 0.94
}
```

---

Interpretación:

```text
0.0 → sin tendencia

1.0 → tendencia perfecta
```

---

# 14. Seasonality Features

Detectar periodicidad.

---

Métodos:

```text
Auto Correlation

FFT

Rolling Correlation
```

---

Vector:

```python
{
    "seasonality": 0.82
}
```

---

# 15. Semantic Features

Extraídas desde nombres.

---

Ejemplo:

```text
revenue
sales
income
```

↓

```python
{
    "business_metric": 1
}
```

---

Ejemplo:

```text
country
region
city
```

↓

```python
{
    "geography": 1
}
```

---

Ejemplo:

```text
retention
churn
cohort
```

↓

```python
{
    "retention_metric": 1
}
```

---

# 16. Semantic Dictionary

DuckDB:

```sql
CREATE TABLE semantic_dictionary (
    token VARCHAR,
    category VARCHAR
);
```

---

Ejemplo:

```text
month
week
year
date
```

↓

```text
TIME
```

---

# 17. Explain Features

Aquí aparece una ventaja enorme de DuckDB.

---

Usar:

```sql
EXPLAIN
```

---

Extraer:

```python
{
    "estimated_rows": 1000000,
    "join_nodes": 3,
    "scan_nodes": 2
}
```

---

# 18. Execution Features

Usar:

```sql
EXPLAIN ANALYZE
```

---

Extraer:

```python
{
    "execution_ms": 250,
    "memory_mb": 30
}
```

---

Esto permite detectar:

```text
Heavy Queries
```

---

# 19. Feature Vector Final

Todo se combina.

---

Ejemplo:

```python
FeatureVector(
    sql_features=...,
    schema_features=...,
    data_features=...,
    stats_features=...,
    semantic_features=...,
)
```

---

# 20. DuckDB Storage

Tabla principal.

```sql
CREATE TABLE feature_vectors (

    fingerprint_id VARCHAR,

    feature_name VARCHAR,

    feature_value DOUBLE
);
```

---

# 21. Python Model

```python
class FeatureVector(BaseModel):

    sql_features: dict
    schema_features: dict
    data_features: dict
    statistical_features: dict
    semantic_features: dict
```

---

# 22. Pipeline Completo

```text
SQL
 │
 ▼
AST
 │
 ▼
Feature Extraction
 │
 ▼
Feature Vector
 │
 ▼
Intent Engine
 │
 ▼
Chart Engine
 │
 ▼
Layout Engine
```

---

# 23. Principio Fundamental

La calidad de SQLviz estará determinada por la calidad de sus features.

Si los features son pobres:

```text
Chart Engine pobre
Intent Engine pobre
Learning pobre
```

Si los features son ricos:

```text
Inferencia superior
Aprendizaje superior
Dashboards superiores
```

Por esta razón el Feature Extraction Engine debe considerarse la infraestructura central de todo SQLviz.
