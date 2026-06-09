# SQLviz Architecture Handbook

## Capítulo 16 — Dashboard Genome

Versión: 0.1

---

# 1. Introducción

Hasta ahora SQLviz entiende:

```text
SQL
 ↓
Semántica
 ↓
Métricas
 ↓
Dimensiones
 ↓
Intenciones
 ↓
Paneles
```

Pero aún falta responder una pregunta mucho más profunda:

```text
¿Qué es realmente un dashboard?
```

---

La mayoría de herramientas BI lo consideran:

```text
Una colección de gráficos
```

---

SQLviz debe considerarlo:

```text
Una estructura analítica
```

---

# 2. La Idea Central

Un dashboard no es un conjunto aleatorio de paneles.

Existe una razón por la cual aparecen juntos.

---

Ejemplo:

```text
Revenue KPI

Revenue Trend

Revenue by Country

Revenue by Product

Revenue Forecast
```

---

Esto no es casualidad.

Representa un patrón.

---

SQLviz llamará a este patrón:

```text
Dashboard Genome
```

---

# 3. Definición

Un Genome es:

```text
La representación estructural
de un dashboard.
```

---

# 4. Filosofía

Igual que un ADN.

---

El ADN:

```text
Define un organismo
```

---

El Genome:

```text
Define un dashboard
```

---

# 5. Arquitectura

```text
Dashboard
     │
     ▼
Genome Extraction
     │
     ▼
Genome Registry
     │
     ▼
Genome Similarity
     │
     ▼
Dashboard Intelligence
```

---

# 6. Dashboard Genes

SQLviz define 5 genes principales.

---

## Metric Gene

```text
Revenue

Profit

Orders
```

---

## Dimension Gene

```text
Country

Product

Customer
```

---

## Intent Gene

```text
Trend

Comparison

Ranking

Distribution
```

---

## Layout Gene

```text
KPI Row

Main Trend

Detail Panels
```

---

## Relation Gene

```text
Cross Filter

Shared Dimension
```

---

# 7. Genome Model

```python
class DashboardGenome:

    genome_id: str

    metrics: list[str]

    dimensions: list[str]

    intents: list[str]

    layout_type: str

    score: float
```

---

# 8. Genome Registry

DuckDB.

```sql
CREATE TABLE dashboard_genomes (

    genome_id VARCHAR,

    metrics JSON,

    dimensions JSON,

    intents JSON,

    layout_type VARCHAR,

    score DOUBLE
);
```

---

# 9. Genome Extraction

Dashboard:

```text
Revenue KPI

Revenue Trend

Revenue by Country

Revenue by Product
```

---

Genome:

```json
{
  "metrics":[
    "Revenue"
  ],

  "dimensions":[
    "Time",
    "Country",
    "Product"
  ],

  "intents":[
    "Trend",
    "Comparison"
  ]
}
```

---

# 10. Genome Fingerprint

Representación compacta.

```text
REVENUE

TIME

COUNTRY

PRODUCT

TREND

COMPARISON
```

---

↓

```text
GENOME_8A7F91
```

---

# 11. Genome Similarity

Comparar dashboards.

---

Dashboard A:

```text
Revenue
Trend
Country
```

---

Dashboard B:

```text
Sales
Trend
Region
```

---

Semantic Engine detecta.

↓

```text
Similarity = 0.92
```

---

# 12. Similarity Function

```python
similarity =
(
 metric_similarity * 0.4
 +
 dimension_similarity * 0.3
 +
 intent_similarity * 0.3
)
```

---

# 13. Genome Clusters

Muchos dashboards terminan agrupándose.

---

Ejemplo.

```text
Revenue Analysis
```

cluster.

---

```text
Marketing Analysis
```

cluster.

---

```text
Operations Analysis
```

cluster.

---

# 14. Archetypes

Un cluster estable crea un arquetipo.

---

Ejemplo.

```text
Revenue Monitoring Dashboard
```

---

```text
Executive Dashboard
```

---

```text
Sales Performance Dashboard
```

---

# 15. Archetype Registry

```sql
CREATE TABLE dashboard_archetypes (

    archetype_id VARCHAR,

    archetype_name VARCHAR,

    genome JSON
);
```

---

# 16. Dashboard DNA

Representación visual.

```text
Revenue Dashboard

Metric:
 Revenue

Dimensions:
 Time
 Country
 Product

Intents:
 Trend
 Ranking
 Comparison
```

---

# 17. Genome Score

Calidad estructural.

---

```python
genome_score =
(
 coverage * 0.4
 +
 consistency * 0.3
 +
 usefulness * 0.3
)
```

---

# 18. Coverage

¿El dashboard cubre varias perspectivas?

---

Ejemplo.

```text
Revenue KPI
Revenue Trend
Revenue Geography
Revenue Product
```

---

Alta cobertura.

---

# 19. Consistency

¿Todos los paneles cuentan la misma historia?

---

Ejemplo.

```text
Revenue KPI

Revenue Trend

Revenue by Country
```

---

Consistente.

---

# 20. Usefulness

Medido mediante:

```text
Interacciones

Filtros usados

Tiempo de permanencia

Guardados
```

---

# 21. Missing Genes

SQLviz puede detectar piezas faltantes.

---

Ejemplo.

Existe:

```text
Revenue KPI

Revenue Trend
```

---

Falta:

```text
Revenue Geography
```

---

# 22. Genome Gap Analysis

Función.

```python
missing_genes(
    genome
)
```

↓

```python
[
 "country_analysis",
 "product_analysis"
]
```

---

# 23. Dashboard Completion

Dashboard incompleto.

↓

Composer agrega.

```text
Revenue by Country
```

---

automáticamente.

---

# 24. Genome Evolution

Los dashboards evolucionan.

---

Versión 1.

```text
Revenue KPI
```

---

Versión 2.

```text
Revenue KPI

Revenue Trend
```

---

Versión 3.

```text
Revenue KPI

Revenue Trend

Forecast
```

---

# 25. Genome History

```sql
CREATE TABLE genome_history (

    genome_id VARCHAR,

    version INTEGER,

    genome JSON,

    created_at TIMESTAMP
);
```

---

# 26. Genome Learning

Registrar.

```text
Panel añadido

Panel eliminado

Panel movido
```

---

Actualizar genome.

---

# 27. Genome Reuse

Dashboard nuevo.

---

Genome similar encontrado.

↓

Reutilizar conocimiento.

---

# 28. Genome Recommendation

Usuario crea.

```text
Revenue Trend
```

---

SQLviz detecta.

```text
Revenue Monitoring Genome
```

---

Sugiere:

```text
Revenue KPI

Revenue by Country

Forecast
```

---

# 29. Genome Graph

Representación.

```text
Revenue
   │
   ├── KPI
   │
   ├── Trend
   │
   ├── Country
   │
   ├── Product
   │
   └── Forecast
```

---

# 30. Genome Centrality

Panel más importante.

---

Ejemplo.

```text
Revenue KPI
```

---

Aparece en:

```text
95%
```

de dashboards Revenue.

---

# 31. Genome Compression

Dashboard completo.

↓

Representación compacta.

```text
GENOME_8A7F91
```

---

# 32. Genome Search

FastAPI.

```python
GET /genomes
```

---

```python
GET /genomes/similar
```

---

```python
POST /genomes/extract
```

---

# 33. Explainability

Ejemplo.

```json
{
  "recommended_panel":
    "Revenue by Country",

  "reason":
    "present in 93% of similar genomes"
}
```

---

# 34. Composer Integration

Dashboard Composer pregunta:

```text
¿Qué suele faltar aquí?
```

---

Genome responde.

```text
Forecast

Country

Product
```

---

# 35. Learning Integration

Learning Engine actualiza:

```text
Genome Quality

Genome Popularity

Genome Acceptance
```

---

# 36. El Salto Conceptual

La mayoría de herramientas modelan:

```text
Dashboard = Lista de paneles
```

---

SQLviz modela:

```text
Dashboard = Organismo Analítico
```

---

# 37. Beneficio Real

Esto permite:

```text
Completar dashboards

Detectar paneles faltantes

Reutilizar patrones

Descubrir arquetipos

Aprender estructuras exitosas
```

---

# 38. Roadmap

V1

```text
Genome Extraction
```

---

V2

```text
Genome Similarity
```

---

V3

```text
Genome Clustering
```

---

V4

```text
Genome Evolution
```

---

# 39. La Idea Más Importante

Los usuarios no construyen dashboards desde cero.

Construyen variaciones de patrones recurrentes.

---

El Genome permite que SQLviz reconozca esos patrones.

---

# 40. Principio Fundamental

El Dashboard Genome transforma dashboards en conocimiento reutilizable.

Es la capa que permite a SQLviz reconocer estructuras exitosas, completar análisis incompletos y aprender cómo se construyen los mejores dashboards.

Sin él, SQLviz genera paneles.

Con él, SQLviz comprende dashboards.