# SQLviz Architecture Handbook

## Capítulo 9 — Layout Engine

Versión: 0.1

---

# 1. Introducción

La mayoría de herramientas BI consideran el layout un problema visual.

SQLviz debe considerarlo un problema semántico.

---

La mayoría de sistemas hacen:

```text
Chart
 ↓
Position
```

SQLviz debe hacer:

```text
Intent Graph
+
Context Graph
+
Metric Graph
 ↓
Dashboard Graph
 ↓
Layout
```

---

# 2. Filosofía

El usuario jamás debe decidir:

```text
x = 4
y = 3

width = 6
height = 2
```

Ni:

```json
{
  "row": 2,
  "column": 3
}
```

---

El Layout Engine infiere automáticamente:

* posición
* tamaño
* agrupación
* jerarquía
* orden de lectura

---

# 3. Objetivo

Transformar:

```text
Dashboard Graph
```

en:

```text
Dashboard Layout
```

---

# 4. Dashboard Graph

Input:

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

Output:

```text
┌────────────┐
│ Revenue KPI│
└────────────┘

┌────────────────────┐
│ Revenue Trend      │
└────────────────────┘

┌──────────┬─────────┐
│ Country  │ Product │
└──────────┴─────────┘
```

---

# 5. Dashboard as Narrative

Un dashboard es una historia.

---

Pregunta:

```text
¿Qué pasó?
```

↓

```text
KPI
```

---

Pregunta:

```text
¿Cuándo pasó?
```

↓

```text
Trend
```

---

Pregunta:

```text
¿Dónde pasó?
```

↓

```text
Geography
```

---

Pregunta:

```text
¿Por qué pasó?
```

↓

```text
Breakdowns
```

---

# 6. Reading Order

Orden oficial.

```text
1 KPI

2 Trend

3 Comparison

4 Ranking

5 Detail
```

---

# 7. Dashboard Zones

SQLviz divide el dashboard.

```text
┌────────────────────┐
│ KPI ZONE           │
├────────────────────┤
│ TREND ZONE         │
├────────────────────┤
│ ANALYSIS ZONE      │
├────────────────────┤
│ DETAIL ZONE        │
└────────────────────┘
```

---

# 8. Grid System

Grid oficial.

```text
12 columnas
```

---

Regla:

```text
1 unidad = columna
```

---

# 9. Panel Classes

SQLviz define clases.

---

## KPI

```text
3x1
```

---

## Trend

```text
12x4
```

---

## Comparison

```text
6x4
```

---

## Ranking

```text
6x4
```

---

## Detail Table

```text
12x6
```

---

# 10. Layout Metadata

Cada panel produce.

```python
PanelLayoutMetadata(
    importance=0.95,
    complexity=0.40,
    intent="trend"
)
```

---

# 11. Importance Score

Probablemente la variable más importante.

---

Inputs:

```text
Intent Score
Relationship Count
Metric Relevance
Usage History
```

---

Fórmula:

```python
importance =
(
    intent_strength * 0.4
    +
    relation_count * 0.3
    +
    usage_score * 0.3
)
```

---

# 12. Complexity Score

Mide dificultad visual.

---

Ejemplos:

```text
KPI
```

↓

```text
0.1
```

---

```text
Heatmap
```

↓

```text
0.8
```

---

```text
Table
```

↓

```text
0.9
```

---

# 13. Space Score

Determina espacio requerido.

---

Fórmula:

```python
space_score =
(
    complexity * 0.5
    +
    cardinality_score * 0.5
)
```

---

# 14. KPI Placement

Regla:

```text
Siempre arriba
```

---

Ejemplo:

```text
Revenue
Orders
Customers
Margin
```

---

Layout:

```text
┌───┬───┬───┬───┐
│KPI│KPI│KPI│KPI│
└───┴───┴───┴───┘
```

---

# 15. Trend Placement

Regla:

```text
Debajo de KPI
```

---

Layout:

```text
┌──────────────┐
│ Trend Panel  │
└──────────────┘
```

---

# 16. Comparison Placement

Regla:

```text
Agrupar comparaciones
```

---

Ejemplo:

```text
Revenue by Country

Revenue by Product

Revenue by Channel
```

↓

```text
┌──────────┬──────────┐
│ Country  │ Product  │
├──────────┼──────────┤
│ Channel  │          │
└──────────┴──────────┘
```

---

# 17. Graph Clustering

Usar Dashboard Graph.

---

Paneles relacionados:

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

# 18. Cluster Layout

Representación:

```text
Revenue Analysis

 ├─ Trend
 ├─ Country
 └─ Product
```

---

Mantener proximidad visual.

---

# 19. Distance Function

Distancia semántica.

---

Fórmula:

```python
distance =
1 - relation_score
```

---

Ejemplo:

```text
Revenue Trend
Revenue Country
```

↓

```text
distance = 0.05
```

---

Ubicarlos juntos.

---

# 20. Graph Projection

Problema:

Convertir grafo.

↓

Grid.

---

Entrada:

```text
Graph
```

---

Salida:

```text
(x,y,w,h)
```

---

# 21. Node Weight

Cada nodo recibe peso.

---

Fórmula:

```python
node_weight =
(
    importance
    +
    centrality
)
```

---

# 22. Graph Centrality

Usar:

```text
Degree Centrality

Betweenness

PageRank
```

---

Panel más conectado.

↓

Más visible.

---

# 23. Dashboard Root

Encontrar nodo raíz.

---

Ejemplo:

```text
Revenue
```

↓

Raíz.

---

Todo gira alrededor de él.

---

# 24. Narrative Layout

Construir historia.

---

Ejemplo:

```text
Revenue KPI

↓

Revenue Trend

↓

Revenue Geography

↓

Revenue Products

↓

Revenue Details
```

---

# 25. Layout Optimization

Objetivo:

Minimizar.

```python
semantic_distance
```

---

Y maximizar.

```python
visual_coherence
```

---

# 26. Layout Score

Fórmula:

```python
layout_score =
(
    coherence * 0.4
    +
    narrative_flow * 0.3
    +
    cluster_quality * 0.3
)
```

---

# 27. DuckDB Schema

```sql
CREATE TABLE layout_patterns (

    fingerprint_id VARCHAR,

    layout_type VARCHAR,

    score DOUBLE,

    accepted_count INTEGER,

    rejected_count INTEGER
);
```

---

# 28. Panel Positions

```sql
CREATE TABLE panel_positions (

    panel_id VARCHAR,

    x INTEGER,

    y INTEGER,

    w INTEGER,

    h INTEGER
);
```

---

# 29. Learning

Usuario mueve panel.

---

Antes:

```text
Trend arriba
```

---

Después:

```text
Trend abajo
```

---

Registrar.

```sql
CREATE TABLE layout_feedback (

    panel_id VARCHAR,

    old_x INTEGER,

    old_y INTEGER,

    new_x INTEGER,

    new_y INTEGER
);
```

---

# 30. Dashboard Archetypes

SQLviz debe aprender plantillas.

---

## Executive Dashboard

```text
KPI
Trend
Comparison
```

---

## Sales Dashboard

```text
Revenue
Trend
Region
Product
```

---

## Operations Dashboard

```text
Volume
Efficiency
Queue
SLA
```

---

# 31. Layout Templates

Tabla:

```sql
CREATE TABLE dashboard_archetypes (

    archetype VARCHAR,

    layout_definition JSON
);
```

---

# 32. Explainability

Explicar layout.

---

Ejemplo:

```json
{
  "panel":"Revenue Trend",
  "position":"top-center",
  "reason":"highest importance score"
}
```

---

# 33. La Idea Más Importante

La mayoría de herramientas optimizan:

```text
Píxeles
```

---

SQLviz debe optimizar:

```text
Comprensión
```

---

# 34. Principio Fundamental

El Layout Engine no organiza gráficos.

Organiza conocimiento.

Su misión es proyectar el Dashboard Graph en una estructura visual que minimice esfuerzo cognitivo y maximice comprensión analítica.

Por eso el Layout Engine debe considerarse una extensión directa del Relation Engine y no un simple sistema de coordenadas.
