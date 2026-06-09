# SQLviz Technical Specification v1

## Capítulo 15 — Dashboard Composer

Versión 0.1

---

# 1. Objetivo

Construir el dashboard final a partir de todas las inferencias previas.

---

El Dashboard Composer es el último motor del pipeline V1.

---

# 2. Filosofía

No infiere.

---

No detecta semántica.

---

No detecta intención.

---

No selecciona charts.

---

Su responsabilidad es:

```text
Componer
```

---

# 3. Pipeline

```text
SQL
 ↓
Parser
 ↓
Features
 ↓
Semantics
 ↓
Intent
 ↓
Metrics
 ↓
Dimensions
 ↓
Charts
 ↓
Layout
 ↓
Dashboard Composer
 ↓
Dashboard
```

---

# 4. Input

```python
context.metrics

context.dimensions

context.charts

context.layout
```

---

# 5. Output

```python
context.dashboard
```

---

# 6. Dashboard Model

```python
class Dashboard:

    dashboard_id: str

    title: str

    panels: list[Panel]

    sections: list[Section]

    confidence: float

    explanation: Explanation
```

---

# 7. Panel Model

```python
class Panel:

    panel_id: str

    title: str

    chart: Chart

    metrics: list[Metric]

    dimensions: list[Dimension]

    placement: PanelPlacement
```

---

# 8. Dashboard Rule

Nunca generar dashboards vacíos.

---

# 9. Fallback Rule

Si no existe chart válido.

↓

```text
Table
```

---

# 10. Dashboard Title

Inferido automáticamente.

---

# 11. Title Sources

```text
Primary Metric

Primary Dimension

Primary Intent
```

---

# 12. Ejemplo

```text
Revenue Trend Analysis
```

---

# 13. Ejemplo

```text
Revenue by Country
```

---

# 14. Ejemplo

```text
Top Products by Revenue
```

---

# 15. Title Generator

```python
title =
 f"{metric} by {dimension}"
```

---

# 16. Section Generation

Generar automáticamente.

---

# 17. V1 Sections

```text
Overview

Trends

Comparisons

Details
```

---

# 18. KPI Section

Contiene.

```text
Primary KPIs
```

---

# 19. Trend Section

Contiene.

```text
Line Charts

Temporal Analysis
```

---

# 20. Comparison Section

Contiene.

```text
Bar Charts

Rankings
```

---

# 21. Details Section

Contiene.

```text
Tables

Raw Data
```

---

# 22. Panel Assembly

Crear paneles.

---

A partir de.

```text
Chart

Metric

Dimension
```

---

# 23. Example

```text
Metric:
Revenue

Dimension:
Country

Chart:
Bar
```

↓

```text
Revenue by Country
```

---

# 24. Panel Builder

```python
class PanelBuilder:

    def build(...)
```

---

# 25. Dashboard Confidence

Promedio ponderado.

---

# 26. Formula

```python
confidence =
(
 chart_confidence * 0.5
 +
 layout_confidence * 0.3
 +
 intent_confidence * 0.2
)
```

---

# 27. Dashboard Explanation

Combina explicaciones.

---

# 28. Example

```text
Dashboard generated because:

Revenue identified as primary metric

Trend identified as primary intent

Line chart selected

Executive layout selected
```

---

# 29. Dashboard Registry

DuckDB.

```sql
CREATE TABLE dashboards (

    dashboard_id VARCHAR,

    title VARCHAR,

    confidence DOUBLE,

    created_at TIMESTAMP
);
```

---

# 30. Panel Registry

```sql
CREATE TABLE dashboard_panels (

    panel_id VARCHAR,

    dashboard_id VARCHAR,

    title VARCHAR,

    chart_type VARCHAR
);
```

---

# 31. Dashboard Snapshot

Guardar.

---

Muy importante.

---

# 32. Snapshot Model

```sql
CREATE TABLE dashboard_snapshots (

    snapshot_id VARCHAR,

    dashboard_id VARCHAR,

    payload JSON
);
```

---

# 33. Why?

Debugging.

---

Benchmark.

---

Learning futuro.

---

# 34. Dashboard Fingerprint

Generar.

---

Ejemplo.

```text
EXECUTIVE_REVENUE_TREND
```

---

# 35. Dashboard Types

V1.

```text
Executive

Analytical

Operational
```

---

# 36. Executive Dashboard

Características.

```text
KPIs

Trend

Ranking
```

---

# 37. Analytical Dashboard

Características.

```text
Correlation

Distribution

Multi-Metric
```

---

# 38. Operational Dashboard

Características.

```text
Tables

Monitoring

Details
```

---

# 39. Explainability API

```text
GET /dashboard/{id}/explain
```

---

# 40. Example Response

```json
{
  "dashboard_type":"executive",
  "confidence":0.95,
  "reasons":[...]
}
```

---

# 41. Event

Publicar.

```text
DASHBOARD_CREATED
```

---

# 42. Payload

```python
{
   "dashboard_id":"...",
   "panel_count":6
}
```

---

# 43. Performance Goal

```text
< 10 ms
```

---

# 44. Cache Strategy

Clave.

```python
fingerprint_hash
```

---

# 45. Reuse Strategy

Si mismo fingerprint.

↓

Reutilizar dashboard.

---

# 46. Benchmark Cases

```text
Single KPI

Trend Dashboard

Executive Dashboard

Analytical Dashboard

Operational Dashboard
```

---

# 47. Success Metric

El usuario no modifica nada.

---

Objetivo.

```text
70%+
```

de dashboards aceptados sin cambios.

---

# 48. Strategic Insight

El Dashboard Composer es el lugar donde SQLviz deja de ser un conjunto de motores.

Y se convierte en un producto.

---

# 49. MVP Complete

Con este capítulo termina el MVP inferencial.

---

Motores V1:

```text
Parser ✅

Feature ✅

Semantic ✅

Intent ✅

Metric ✅

Dimension ✅

Chart ✅

Layout ✅

Dashboard Composer ✅
```

---

# 50. Principio Fundamental

El Dashboard Composer no genera gráficos.

No genera métricas.

No genera semántica.

Genera una experiencia analítica completa a partir de todo el conocimiento producido por el pipeline.
