# SQLviz Architecture Handbook

## Capítulo 8 — Relation Engine (Intent Graph & Context Graph)

Versión: 0.1

---

# 1. Introducción

Este capítulo define probablemente el componente más diferenciador de SQLviz.

La mayoría de herramientas BI ven un dashboard como:

```text
Dashboard
 ├─ Chart
 ├─ Chart
 ├─ Chart
 └─ Chart
```

SQLviz debe verlo como:

```text
Dashboard
 └─ Graph
      ├─ Panels
      ├─ Metrics
      ├─ Dimensions
      ├─ Filters
      ├─ Intents
      └─ Relationships
```

Los gráficos son sólo una visualización de un grafo analítico.

---

# 2. Problema Fundamental

Supongamos:

Panel 1

```sql
SELECT
    month,
    SUM(revenue)
FROM sales
GROUP BY month
```

Panel 2

```sql
SELECT
    country,
    SUM(revenue)
FROM sales
GROUP BY country
```

Panel 3

```sql
SELECT
    product,
    SUM(revenue)
FROM sales
GROUP BY product
```

Una herramienta BI tradicional ve:

```text
3 charts
```

SQLviz debe ver:

```text
1 métrica
3 perspectivas
1 historia analítica
```

---

# 3. Objetivo

Descubrir relaciones implícitas entre:

```text
Panels
Metrics
Dimensions
Filters
Intents
```

sin configuración manual.

---

# 4. Arquitectura

```text
SQL Queries
     │
     ▼
Feature Extraction
     │
     ▼
Panel Metadata
     │
     ▼
Relation Discovery
     │
     ▼
Context Graph
     │
     ▼
Intent Graph
     │
     ▼
Dashboard Graph
```

---

# 5. Entidades Fundamentales

El Relation Engine trabaja sobre cinco nodos.

---

## Panel

Representa una consulta.

```python
PanelNode
```

---

## Metric

Representa una medida.

```text
revenue
profit
orders
customers
```

---

## Dimension

Representa contexto.

```text
country
region
product
date
```

---

## Intent

Representa objetivo analítico.

```text
trend
comparison
ranking
```

---

## Filter

Representa restricción.

```text
country
year
product
```

---

# 6. Dashboard Graph

Representación interna.

```text
          Revenue
          /  |   \
         /   |    \
      Time Country Product
        |      |      |
      Trend Comparison Ranking
```

---

# 7. Panel Metadata

Cada panel genera metadatos.

---

Ejemplo:

```python
{
    "panel_id":"p1",

    "metrics":["revenue"],

    "dimensions":["month"],

    "intent":"trend"
}
```

---

# 8. Shared Metric Detection

Regla más importante.

---

Panel A

```sql
SUM(revenue)
```

Panel B

```sql
AVG(revenue)
```

---

Métrica compartida:

```text
revenue
```

---

Crear relación.

```text
PanelA ── revenue ── PanelB
```

---

# 9. Shared Dimension Detection

Panel A

```sql
GROUP BY country
```

Panel B

```sql
GROUP BY country, product
```

---

Dimensión compartida:

```text
country
```

---

Crear relación.

```text
PanelA ── country ── PanelB
```

---

# 10. Intent Relationship

Paneles pueden compartir intención.

---

Ejemplo:

```text
Revenue Trend
Orders Trend
Customers Trend
```

---

Todos comparten:

```text
TREND
```

---

# 11. Metric Similarity

No siempre los nombres coinciden.

---

Ejemplo:

```text
sales
revenue
income
```

---

Semantic Dictionary:

```text
BUSINESS_REVENUE
```

---

Relación detectada.

---

# 12. Dimension Similarity

Ejemplo:

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

Permite conectar paneles.

---

# 13. Temporal Relationship

Panel A

```sql
GROUP BY year
```

Panel B

```sql
GROUP BY month
```

---

Ambos pertenecen a:

```text
TIME
```

---

Relación parcial.

---

# 14. Granularity Detection

Muy importante.

---

Ejemplo:

```text
Year
Month
Day
```

---

Jerarquía:

```text
Year
 └── Month
       └── Day
```

---

Guardar en grafo.

---

# 15. Metric Hierarchies

Ejemplo:

```text
Revenue
 ├── Gross Revenue
 ├── Net Revenue
 └── Revenue Growth
```

---

Permite composición automática.

---

# 16. Context Graph

El Context Graph representa dimensiones compartidas.

---

Ejemplo:

```text
Country
 ├── Revenue
 ├── Orders
 ├── Customers
 └── Margin
```

---

Esto define filtros globales.

---

# 17. Intent Graph

Representa narrativa analítica.

---

Ejemplo:

```text
Revenue KPI
      │
      ▼
Revenue Trend
      │
      ▼
Revenue by Country
      │
      ▼
Top Products
```

---

No son gráficos.

Son ideas relacionadas.

---

# 18. Relationship Types

SQLviz define:

```text
SHARED_METRIC

SHARED_DIMENSION

SEMANTIC_SIMILARITY

INTENT_SIMILARITY

TIME_HIERARCHY

METRIC_HIERARCHY
```

---

# 19. Relationship Score

Cada relación recibe score.

---

Fórmula:

```python
relation_score =
(
    metric_similarity * 0.4
    +
    dimension_similarity * 0.4
    +
    intent_similarity * 0.2
)
```

---

# 20. Ejemplo

Panel A

```sql
Revenue by Country
```

Panel B

```sql
Orders by Country
```

---

Resultado:

```python
{
    "shared_dimension":"country",
    "score":0.92
}
```

---

# 21. DuckDB Schema

Nodo.

```sql
CREATE TABLE graph_nodes (

    node_id VARCHAR,

    node_type VARCHAR,

    name VARCHAR
);
```

---

# 22. Edge Table

```sql
CREATE TABLE graph_edges (

    source_id VARCHAR,

    target_id VARCHAR,

    relationship_type VARCHAR,

    score DOUBLE
);
```

---

# 23. Intent Graph Table

```sql
CREATE TABLE intent_graph (

    panel_id VARCHAR,

    intent VARCHAR,

    confidence DOUBLE
);
```

---

# 24. Metric Registry

```sql
CREATE TABLE metric_registry (

    metric_name VARCHAR,

    semantic_category VARCHAR
);
```

---

# 25. Dimension Registry

```sql
CREATE TABLE dimension_registry (

    dimension_name VARCHAR,

    semantic_category VARCHAR
);
```

---

# 26. Graph Traversal

Ejemplo:

Usuario selecciona:

```text
country = Bolivia
```

---

Relation Engine busca:

```text
Country
```

↓

```text
Revenue
Orders
Customers
Profit
```

↓

Actualiza paneles.

---

# 27. Dashboard Clustering

Agrupar paneles relacionados.

---

Ejemplo:

```text
Revenue Trend

Revenue by Country

Revenue by Product
```

---

Cluster:

```text
Revenue Analysis
```

---

# 28. Narrative Detection

Idea extremadamente poderosa.

---

Detectar secuencias.

```text
KPI
 ↓
Trend
 ↓
Comparison
 ↓
Ranking
```

---

Representa una historia.

---

# 29. Dashboard Story Graph

Futuro.

---

Ejemplo:

```text
Revenue
 ├── What happened?
 │      KPI
 │
 ├── When?
 │      Trend
 │
 ├── Where?
 │      Geography
 │
 └── Why?
        Product Breakdown
```

---

Esto permitiría generar dashboards completos automáticamente.

---

# 30. Learning

Registrar relaciones aceptadas.

---

Tabla:

```sql
CREATE TABLE graph_feedback (

    source_id VARCHAR,

    target_id VARCHAR,

    accepted BOOLEAN
);
```

---

# 31. Explainability

Toda relación debe ser explicable.

---

Ejemplo:

```json
{
  "panel_a":"Revenue",
  "panel_b":"Orders",
  "relationship":"country",
  "score":0.91
}
```

---

# 32. La Idea Más Importante

La mayoría de herramientas modelan dashboards como listas.

SQLviz debe modelarlos como grafos.

---

Lista:

```text
Chart
Chart
Chart
Chart
```

---

Grafo:

```text
Metric
 │
 ├── Trend
 │
 ├── Geography
 │
 ├── Product
 │
 └── Customer
```

---

# 33. Principio Fundamental

El Relation Engine convierte consultas independientes en una estructura analítica coherente.

Es el componente que permite que SQLviz evolucione desde:

```text
Auto Chart Builder
```

hasta:

```text
Autonomous BI System
```

porque descubre cómo todas las piezas del dashboard se relacionan entre sí.
