# SQLviz Architecture Handbook

## Capítulo 10 — Dashboard Composer

Versión: 0.1

---

# 1. Introducción

Este es probablemente el componente más importante de toda la arquitectura.

Hasta ahora SQLviz puede:

```text
SQL
 ↓
Features
 ↓
Intent
 ↓
Charts
 ↓
Filters
 ↓
Layout
```

Eso ya supera muchos sistemas BI tradicionales.

Pero aún existe una limitación:

SQLviz sigue respondiendo únicamente a lo que el usuario pidió.

---

Ejemplo:

Usuario escribe:

```sql
SELECT
    month,
    SUM(revenue)
FROM sales
GROUP BY month
```

---

Un sistema tradicional genera:

```text
Revenue Trend
```

---

Pero un analista humano haría inmediatamente:

```text
Revenue KPI
Revenue Trend
Revenue Growth
Revenue by Country
Revenue by Product
Top Customers
Anomalies
Forecast
```

---

Aquí nace Dashboard Composer.

---

# 2. Objetivo

Responder:

```text
¿Qué paneles faltan?
```

en lugar de:

```text
¿Qué gráfico corresponde?
```

---

# 3. Filosofía

El SQL del usuario es una pista.

No es el dashboard completo.

---

Entrada:

```sql
Revenue by Month
```

---

Salida:

```text
Revenue Dashboard
```

---

# 4. Arquitectura

```text
SQL
 │
 ▼
Feature Engine
 │
 ▼
Intent Engine
 │
 ▼
Relation Engine
 │
 ▼
Dashboard Composer
 │
 ▼
Dashboard Graph
 │
 ▼
Layout Engine
```

---

# 5. Concepto Fundamental

El usuario normalmente muestra una perspectiva.

El Composer debe inferir las perspectivas faltantes.

---

Ejemplo:

```sql
Revenue by Month
```

representa:

```text
Metric = Revenue
Dimension = Time
Intent = Trend
```

---

El sistema debe preguntarse:

```text
¿Qué otras perspectivas existen
para Revenue?
```

---

# 6. Analytical Perspectives

SQLviz define perspectivas universales.

---

## KPI

```text
¿Cuánto?
```

---

## Trend

```text
¿Cuándo?
```

---

## Geography

```text
¿Dónde?
```

---

## Product

```text
¿Qué?
```

---

## Customer

```text
¿Quién?
```

---

## Segment

```text
¿Qué segmento?
```

---

## Ranking

```text
¿Cuáles son los mejores?
```

---

## Anomaly

```text
¿Qué es extraño?
```

---

## Forecast

```text
¿Qué ocurrirá?
```

---

# 7. Dashboard Expansion

Consulta inicial:

```sql
Revenue by Month
```

---

Composer genera:

```text
Revenue KPI
Revenue Trend
Revenue by Country
Revenue by Product
Top Products
Revenue Anomalies
```

---

# 8. Dashboard Graph Expansion

Input:

```text
Revenue Trend
```

---

Output:

```text
Revenue KPI

Revenue Trend

Revenue by Country

Revenue by Product

Revenue Ranking
```

---

# 9. Panel Templates

Cada intención posee paneles derivados.

---

Trend:

```text
Trend
Growth Rate
Moving Average
Forecast
```

---

Comparison:

```text
Comparison
Top N
Bottom N
Share %
```

---

Distribution:

```text
Distribution
Boxplot
Percentiles
```

---

# 10. Metric-Centric Composition

El Composer gira alrededor de métricas.

---

Ejemplo:

```text
Revenue
```

genera:

```text
Revenue KPI
Revenue Trend
Revenue Geography
Revenue Product
```

---

# 11. Intent Expansion Matrix

Tabla conceptual:

```text
Trend
 ├─ KPI
 ├─ Growth
 ├─ Forecast

Comparison
 ├─ Ranking
 ├─ Share

Distribution
 ├─ Histogram
 ├─ Percentiles
```

---

# 12. Composition Rules

Ejemplo:

```python
if intent == "trend":
    generate("kpi")
    generate("growth")
    generate("forecast")
```

---

# 13. Dashboard Archetypes

El Composer identifica arquetipos.

---

Ejemplo:

```text
Revenue
Sales
Orders
Margin
```

↓

```text
Sales Dashboard
```

---

Ejemplo:

```text
Active Users
Retention
Churn
```

↓

```text
Product Analytics Dashboard
```

---

# 14. Archetype Registry

DuckDB:

```sql
CREATE TABLE dashboard_archetypes (

    archetype_name VARCHAR,

    metric_patterns JSON,

    panel_templates JSON
);
```

---

# 15. Metric Registry

```sql
CREATE TABLE metric_semantics (

    metric_name VARCHAR,

    semantic_type VARCHAR
);
```

---

Ejemplo:

```text
revenue
sales
income
```

↓

```text
BUSINESS_REVENUE
```

---

# 16. Composition Candidates

Cada panel generado recibe score.

---

Ejemplo:

```python
CandidatePanel(
    panel_type="forecast",
    score=0.81
)
```

---

# 17. Composition Score

Fórmula:

```python
composition_score =
(
    relevance * 0.5
    +
    archetype_fit * 0.3
    +
    historical_usage * 0.2
)
```

---

# 18. Relevance Score

Mide utilidad.

---

Ejemplo:

```text
Revenue KPI
```

↓

```text
0.98
```

---

Ejemplo:

```text
Revenue Forecast
```

↓

```text
0.62
```

---

# 19. Dashboard Completeness

Concepto extremadamente importante.

---

Ejemplo:

Dashboard:

```text
Revenue Trend
```

---

Completeness:

```text
25%
```

---

Faltan:

```text
KPI
Geography
Product
Ranking
```

---

# 20. Completeness Formula

```python
completeness =
present_perspectives
/
expected_perspectives
```

---

Ejemplo:

```python
2 / 8
```

↓

```text
25%
```

---

# 21. Dashboard Recommendation

Output:

```json
{
  "dashboard":"Revenue Analysis",
  "completeness":0.25,
  "recommended_panels":[
      "Revenue KPI",
      "Revenue Geography",
      "Revenue Product"
  ]
}
```

---

# 22. Auto-Generated Queries

Composer puede generar SQL.

---

Input:

```sql
SELECT
 month,
 SUM(revenue)
FROM sales
GROUP BY month
```

---

Generar:

```sql
SELECT
 country,
 SUM(revenue)
FROM sales
GROUP BY country
```

---

# 23. Query Template Registry

```sql
CREATE TABLE query_templates (

    template_name VARCHAR,

    intent VARCHAR,

    sql_template TEXT
);
```

---

# 24. Forecast Generation

Si existe:

```text
Time
+
Metric
```

↓

Generar:

```text
Forecast Panel
```

---

# 25. Growth Panel

Si existe:

```text
Revenue Trend
```

↓

Generar:

```text
Growth %
```

---

# 26. Anomaly Panel

Si existe:

```text
Trend
```

↓

Generar:

```text
Anomaly Detection
```

---

# 27. Narrative Graph

Dashboard generado:

```text
Revenue KPI
      │
      ▼
Revenue Trend
      │
      ▼
Revenue Geography
      │
      ▼
Top Products
      │
      ▼
Anomalies
```

---

Esto ya representa una historia.

---

# 28. Learning

Registrar:

```text
Panel aceptado

Panel eliminado

Panel agregado
```

---

Tabla:

```sql
CREATE TABLE composition_feedback (

    fingerprint_id VARCHAR,

    panel_type VARCHAR,

    accepted BOOLEAN
);
```

---

# 29. Composition Learning

Si usuarios eliminan siempre:

```text
Forecast
```

↓

Reducir score.

---

Si siempre aceptan:

```text
Revenue KPI
```

↓

Aumentar score.

---

# 30. Explainability

Toda recomendación debe explicarse.

---

Ejemplo:

```json
{
  "panel":"Revenue KPI",
  "reason":"required perspective for metric revenue"
}
```

---

# 31. Dashboard DNA

Idea avanzada.

---

Cada dashboard puede representarse como:

```python
DashboardDNA(
    metrics,
    dimensions,
    intents,
    perspectives
)
```

---

Esto permite clustering.

---

# 32. Dashboard Similarity

Comparar dashboards.

---

Ejemplo:

```text
Revenue Dashboard
```

similar a:

```text
Sales Dashboard
```

---

Score:

```python
0.92
```

---

# 33. Dashboard Genome

Visión futura.

---

Miles de dashboards.

↓

Aprender:

```text
Patrones recurrentes
```

↓

Generar dashboards automáticamente.

---

# 34. El Salto Conceptual

La mayoría de herramientas responden:

```text
¿Qué gráfico mostrar?
```

---

SQLviz debe responder:

```text
¿Qué análisis falta?
```

---

# 35. Principio Fundamental

El Dashboard Composer es el componente que transforma SQLviz desde un sistema de visualización automática a un sistema de razonamiento analítico.

No genera gráficos.

No genera layouts.

No genera filtros.

Genera hipótesis analíticas.

Y esas hipótesis son la materia prima de un verdadero sistema Autonomous BI.
