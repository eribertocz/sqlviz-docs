# SQLviz Technical Specification v1

## Capítulo 14 — Layout Engine

Versión 0.1

---

# 1. Objetivo

Inferir automáticamente la disposición visual óptima de un dashboard.

---

El Chart Engine responde:

```text
¿Qué gráfico usar?
```

---

El Layout Engine responde:

```text
¿Dónde debe mostrarse?
```

---

# 2. Filosofía

La mayoría de herramientas BI:

```text
Usuario diseña layout
```

---

SQLviz:

```text
SQL
↓
Layout inferido
```

---

# 3. Importancia

Dos dashboards pueden tener exactamente los mismos gráficos.

---

Pero uno puede verse:

```text
Excelente
```

---

Y otro:

```text
Caótico
```

---

# 4. Input

```python
context.intents

context.metrics

context.dimensions

context.charts
```

---

# 5. Output

```python
context.layout
```

---

# 6. Layout Model

```python
class Layout:

    layout_id: str

    panels: list[PanelPlacement]

    confidence: float

    explanation: Explanation
```

---

# 7. Panel Placement

```python
class PanelPlacement:

    panel_id: str

    x: int

    y: int

    w: int

    h: int

    priority: float
```

---

# 8. Grid System

V1.

```text
12 columnas
```

---

Similar a:

```text
Bootstrap

Mantine

Dash
```

---

# 9. Grid Example

```text
|----12 cols----|

| KPI | KPI | KPI |

|    MAIN CHART    |

| CHART | CHART |
```

---

# 10. Layout Principle

No todos los paneles tienen la misma importancia.

---

# 11. Priority Score

Cada panel recibe.

```python
priority_score
```

---

# 12. Priority Formula

```python
score =
(
 metric_importance * 0.4
 +
 intent_importance * 0.3
 +
 chart_importance * 0.3
)
```

---

# 13. KPI Priority

Siempre alta.

---

# 14. Example

```sql
SELECT
 SUM(revenue)
```

↓

```text
Revenue KPI
```

↓

```text
Priority = 1.0
```

---

# 15. Trend Priority

Muy alta.

---

# 16. Example

```text
Revenue by Month
```

↓

```text
Priority = 0.95
```

---

# 17. Ranking Priority

Media.

---

# 18. Example

```text
Top Products
```

↓

```text
Priority = 0.75
```

---

# 19. Detail Tables

Baja.

---

# 20. Example

```text
Raw Data Table
```

↓

```text
Priority = 0.40
```

---

# 21. Panel Sizes

V1.

```text
XS

SM

MD

LG

XL
```

---

# 22. Mapping

```text
XS = 3 cols

SM = 4 cols

MD = 6 cols

LG = 8 cols

XL = 12 cols
```

---

# 23. KPI Rule

KPIs.

↓

```text
SM
```

---

# 24. Trend Rule

Line Charts.

↓

```text
XL
```

---

# 25. Comparison Rule

Bar Charts.

↓

```text
MD
```

---

# 26. Scatter Rule

Scatter.

↓

```text
LG
```

---

# 27. Table Rule

Tables.

↓

```text
XL
```

---

# 28. Layout Templates

V1.

```text
Executive

Analytical

Operational
```

---

# 29. Executive Template

Detectar.

```text
KPIs

Trend

Ranking
```

---

Layout.

```text
KPI KPI KPI KPI

Trend Trend Trend

Ranking Ranking
```

---

# 30. Analytical Template

Detectar.

```text
Correlation

Distribution

Multi Metric
```

---

Layout.

```text
Main Chart

Supporting Charts
```

---

# 31. Operational Template

Detectar.

```text
Tables

Details

Monitoring
```

---

Layout.

```text
KPIs

Table
```

---

# 32. Grouping Rule

Paneles relacionados deben estar juntos.

---

# 33. Example

```text
Revenue by Country

Revenue by Product
```

↓

Agrupar.

---

# 34. Semantic Clustering

Usar semantic tags.

---

# 35. Example

```text
Revenue
```

común.

↓

Mismo grupo.

---

# 36. Temporal Clustering

Todos los paneles temporales juntos.

---

# 37. Example

```text
Revenue by Month

Profit by Month
```

↓

Agrupar.

---

# 38. Layout Sections

Modelo.

```python
class Section:

    name: str

    panels: list
```

---

# 39. V1 Sections

```text
KPIs

Trends

Comparisons

Details
```

---

# 40. Visual Hierarchy

Regla crítica.

---

Usuario ve primero:

```text
KPIs
```

---

Luego:

```text
Trend
```

---

Luego:

```text
Comparison
```

---

Luego:

```text
Details
```

---

# 41. Eye Flow Rule

Diseñar.

```text
Izquierda → Derecha

Arriba → Abajo
```

---

# 42. Dashboard Importance Order

```text
Primary Metric

Primary Trend

Primary Comparison

Supporting Panels
```

---

# 43. Multi Dashboard Detection

Si paneles > 12.

↓

Considerar.

```text
Múltiples páginas
```

---

# 44. Layout Registry

DuckDB.

```sql
CREATE TABLE layouts (

    layout_id VARCHAR,

    execution_id VARCHAR,

    layout_type VARCHAR,

    confidence DOUBLE
);
```

---

# 45. Panel Registry

```sql
CREATE TABLE panel_layouts (

    panel_id VARCHAR,

    layout_id VARCHAR,

    x INTEGER,

    y INTEGER,

    w INTEGER,

    h INTEGER
);
```

---

# 46. Explainability

Ejemplo.

```text
Revenue Trend

Size: XL

Reason:

Primary metric detected

Trend intent detected

High analytical priority
```

---

# 47. Event

Publicar.

```text
LAYOUT_GENERATED
```

---

# 48. Payload

```python
{
   "panel_count": 6,
   "layout":"executive"
}
```

---

# 49. Performance Goal

```text
< 20 ms
```

---

# 50. Layout Confidence

Modelo.

```python
confidence =
(
 layout_template_score
 +
 clustering_score
) / 2
```

---

# 51. Benchmark Cases

```text
Single KPI

KPI + Trend

Executive Dashboard

Analytical Dashboard

Operational Dashboard
```

---

# 52. Strategic Insight

La mayoría de herramientas generan:

```text
Charts
```

---

SQLviz debe generar:

```text
Narrativas visuales
```

---

# 53. Future V2

```text
Adaptive Layouts

Mobile Layouts

User Preferences
```

---

# 54. Future V3

```text
Genome-Based Layouts

Learning Layouts

Self Optimizing Layouts
```

---

# 55. Una mejora que añadiría

No guardar solo posiciones.

Guardar también:

```python
class LayoutReasoning:

    primary_panel

    supporting_panels

    grouping_strategy

    hierarchy_strategy
```

---

Porque en el futuro eso alimentará:

```text
Dashboard Genome

Learning Engine

Recommendation Engine
```

---

# 56. Principio Fundamental

Un dashboard no es una colección de gráficos.

Es una historia visual.

El Layout Engine decide cómo contar esa historia para que el usuario entienda primero lo importante y después los detalles.