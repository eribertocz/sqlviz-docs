# SQLviz Architecture Handbook

## Capítulo 6 — Chart Engine

Versión: 0.1

---

# 1. Introducción

El Chart Engine transforma intención analítica en representación visual.

NO decide basado únicamente en tipos de columnas.

NO utiliza reglas rígidas.

NO funciona con:

```python
if has_date:
    return "line"
```

Ese enfoque escala mal.

El objetivo es rankear candidatos.

---

# 2. Filosofía

SQLviz no selecciona gráficos.

SQLviz calcula probabilidades.

---

Incorrecto:

```python
chart = "line"
```

---

Correcto:

```python
[
    ("line", 0.94),
    ("area", 0.82),
    ("bar", 0.44)
]
```

---

# 3. Arquitectura

```text
Feature Vector
      │
      ▼
Intent Vector
      │
      ▼
Chart Candidate Generator
      │
      ▼
Chart Scorers
      │
      ▼
Chart Ranking
      │
      ▼
Top Candidate
```

---

# 4. Inputs

El Chart Engine consume:

```python
FeatureVector
IntentVector
Fingerprint
HistoricalFeedback
```

---

# 5. Chart Taxonomy

SQLviz debe soportar inicialmente:

```text
KPI
Line
Area
Bar
Horizontal Bar
Stacked Bar
Pie
Donut
Scatter
Bubble
Histogram
Boxplot
Heatmap
Treemap
Table
Funnel
Cohort
Waterfall
```

---

# 6. Candidate Generation

Antes de calcular scores.

Eliminar gráficos imposibles.

---

Ejemplo:

```sql
SELECT SUM(revenue)
```

Sólo puede generar:

```text
KPI
Gauge
Card
```

---

No tiene sentido generar:

```text
Scatter
Line
Heatmap
```

---

# 7. Candidate Rules

Ejemplo:

```python
if numeric_columns == 1:
    candidates.add("kpi")
```

---

Ejemplo:

```python
if time_dimension:
    candidates.add("line")
```

---

Ejemplo:

```python
if two_numeric_columns:
    candidates.add("scatter")
```

---

# 8. Line Score

Inputs:

```text
trend
time_dimension
seasonality
```

---

Fórmula:

```python
line_score =
(
    trend * 0.45
    +
    time_dimension * 0.35
    +
    seasonality * 0.20
)
```

---

# 9. Area Score

Inputs:

```text
trend
time_dimension
aggregation
```

---

Fórmula:

```python
area_score =
(
    trend * 0.40
    +
    aggregation * 0.40
    +
    time_dimension * 0.20
)
```

---

# 10. Bar Score

Inputs:

```text
comparison
category_dimension
cardinality
```

---

Fórmula:

```python
bar_score =
(
    comparison * 0.50
    +
    category_dimension * 0.30
    +
    cardinality_score * 0.20
)
```

---

# 11. Horizontal Bar Score

Especial para rankings.

---

Inputs:

```text
ranking
limit
order_desc
```

---

Fórmula:

```python
horizontal_bar_score =
(
    ranking * 0.60
    +
    order_desc * 0.20
    +
    limit_present * 0.20
)
```

---

# 12. Scatter Score

Inputs:

```text
correlation
numeric_x
numeric_y
```

---

Fórmula:

```python
scatter_score =
(
    correlation_intent * 0.50
    +
    numeric_pair * 0.50
)
```

---

# 13. Histogram Score

Inputs:

```text
distribution
single_numeric_column
```

---

Fórmula:

```python
histogram_score =
(
    distribution * 0.60
    +
    numeric_dimension * 0.40
)
```

---

# 14. Boxplot Score

Inputs:

```text
distribution
outlier_ratio
```

---

Fórmula:

```python
boxplot_score =
(
    distribution * 0.50
    +
    outlier_ratio * 0.50
)
```

---

# 15. Heatmap Score

Inputs:

```text
time_dimension
category_dimension
high_density
```

---

Fórmula:

```python
heatmap_score =
(
    density_score * 0.40
    +
    time_dimension * 0.30
    +
    category_dimension * 0.30
)
```

---

# 16. KPI Score

Inputs:

```text
one_row
one_metric
```

---

Fórmula:

```python
kpi_score =
(
    one_row * 0.70
    +
    one_metric * 0.30
)
```

---

# 17. Pie Score

Muy restringido.

---

Inputs:

```text
composition
low_cardinality
```

---

Fórmula:

```python
pie_score =
(
    composition * 0.70
    +
    low_cardinality * 0.30
)
```

---

Regla:

```python
cardinality <= 8
```

---

# 18. Treemap Score

Mejor alternativa al pie.

---

Inputs:

```text
composition
medium_cardinality
```

---

# 19. Funnel Score

Inputs:

```text
funnel_intent
```

---

Fórmula:

```python
funnel_score =
(
    funnel_intent
)
```

---

# 20. Cohort Score

Inputs:

```text
cohort_intent
```

---

Fórmula:

```python
cohort_score =
(
    cohort_intent
)
```

---

# 21. Confidence Score

Además del chart score.

---

Calcular:

```python
confidence
```

---

Ejemplo:

```python
line = 0.94
area = 0.91
```

---

Confianza:

```python
0.03
```

---

Interpretación:

```text
Alta ambigüedad
```

---

# 22. Ranking Final

Ejemplo:

```python
[
    ("line",0.94),
    ("area",0.82),
    ("bar",0.31)
]
```

---

Guardar Top-N.

```python
top_3
```

---

# 23. Explainability

Toda decisión debe explicarse.

---

Ejemplo:

```json
{
  "chart":"line",
  "score":0.94,
  "reasons":[
    "time_dimension",
    "trend_strength",
    "seasonality"
  ]
}
```

---

# 24. Learning Layer

Usuario:

```text
Line
```

↓

```text
Area
```

---

Registrar:

```python
feedback_event
```

---

# 25. Chart Pattern Table

DuckDB:

```sql
CREATE TABLE chart_patterns (

    fingerprint_id VARCHAR,

    chart_type VARCHAR,

    score DOUBLE,

    accepted_count INTEGER,

    rejected_count INTEGER
);
```

---

# 26. Acceptance Rate

Calcular:

```python
acceptance_rate =
accepted
/
(
accepted
+
rejected
)
```

---

Ejemplo:

```text
Line

92%
```

---

# 27. Learned Score

Score final:

```python
final_score =
(
base_score * 0.7
+
historical_score * 0.3
)
```

---

# 28. Community Learning (Futuro)

Si miles de usuarios usan:

```text
TIME_SUM
```

y seleccionan:

```text
Area
```

SQLviz aprende.

---

# 29. Chart Registry

Tabla:

```sql
CREATE TABLE chart_registry (

    chart_type VARCHAR,

    min_columns INTEGER,

    max_columns INTEGER,

    requires_time BOOLEAN,

    requires_numeric BOOLEAN
);
```

---

# 30. Resultado Final

Output oficial:

```python
ChartRecommendation(
    chart="line",
    score=0.94,
    confidence=0.89,
    alternatives=[
        ("area",0.82),
        ("bar",0.31)
    ]
)
```

---

# 31. Principio Fundamental

El Chart Engine no responde:

```text
¿Qué gráfico puedo dibujar?
```

Responde:

```text
¿Qué representación visual comunica mejor
la intención detectada?
```

Ese cambio conceptual es lo que separa a SQLviz de un simple generador automático de gráficos.
