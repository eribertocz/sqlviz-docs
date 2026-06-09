# SQLviz Architecture Handbook

## Capítulo 23 — Svelte 5 Frontend Architecture

Versión: 0.1

---

# 1. Introducción

La mayoría de herramientas BI diseñan la UI alrededor de:

```text id="3tlj9r"
Charts
```

---

Por eso tienen:

```text id="l7lr0j"
Chart Builder

Axis Editor

Color Editor

Filter Editor

Dashboard Editor
```

---

SQLviz tiene una filosofía distinta.

---

El usuario no construye dashboards.

---

El usuario escribe:

```sql id="9b04t8"
SELECT ...
```

---

Y SQLviz hace el resto.

---

# 2. Objetivo

Construir una interfaz centrada en:

```text id="7kocgi"
Consulta

Conocimiento

Insights

Exploración
```

---

No en configuración.

---

# 3. Filosofía

Eliminar la mayor cantidad posible de:

```text id="u92vuy"
Dropdowns

Checkboxes

Configuraciones

Wizards
```

---

# 4. Principio

La pantalla principal debe responder:

```text id="qjifkh"
¿Qué quiero saber?
```

---

No:

```text id="rv5xiw"
¿Cómo configuro un gráfico?
```

---

# 5. Arquitectura General

```text id="z36yul"
App Shell
   │
   ├── Query Workspace
   │
   ├── Dashboard View
   │
   ├── Insight Panel
   │
   ├── Recommendation Panel
   │
   └── Investigation View
```

---

# 6. Layout Principal

```text id="ewjlwm"
+--------------------------------------+

 SQL Editor

+--------------------------------------+

 Dashboard

+--------------------------------------+

 Insights

 Recommendations

 Findings

+--------------------------------------+
```

---

# 7. SQL First

Elemento central.

---

```text id="k4r5ir"
SQL Editor
```

---

Siempre visible.

---

# 8. Query Workspace

Componente.

```text id="3i5zsl"
<QueryWorkspace />
```

---

Responsabilidad.

```text id="itbzmv"
Editar SQL

Ejecutar SQL

Ver historial
```

---

# 9. Dashboard View

Componente.

```text id="n6ljh6"
<DashboardView />
```

---

Responsabilidad.

```text id="29aqvt"
Renderizar paneles inferidos
```

---

# 10. Dashboard Principle

Nunca mostrar:

```text id="5ef8w7"
Config Chart
```

---

Mostrar:

```text id="iwt2qh"
Chart generado
```

---

# 11. Insight Panel

Componente.

```text id="p4m2ye"
<InsightPanel />
```

---

Responsabilidad.

```text id="j93v8g"
Mostrar hallazgos
```

---

# 12. Ejemplo

```text id="85s5qe"
Revenue creció 12%

Confidence 0.94
```

---

# 13. Recommendation Panel

Componente.

```text id="j7mrrj"
<RecommendationPanel />
```

---

Responsabilidad.

```text id="v3ghrw"
Mostrar siguientes análisis
```

---

# 14. Ejemplo

```text id="s6zb3w"
Revenue by Product

Revenue Trend

Profit by Country
```

---

# 15. Findings Panel

Componente.

```text id="9ig62u"
<FindingPanel />
```

---

Responsabilidad.

```text id="g7tvb4"
Mostrar resultados autónomos
```

---

# 16. Ejemplo

```text id="m56w0u"
La caída se concentra
en Electronics Retail.
```

---

# 17. Investigation View

Componente.

```text id="obp6df"
<InvestigationView />
```

---

Responsabilidad.

```text id="76vvf5"
Visualizar árboles de análisis
```

---

# 18. Investigation Tree

```text id="azd4ye"
Revenue

└─ Argentina

   └─ Electronics

      └─ Retail
```

---

# 19. Frontend State

Dividir estado.

---

```text id="u3s8jz"
UI State

Data State

Knowledge State
```

---

# 20. UI State

Ejemplo.

```typescript id="8zqlr4"
selectedPanel

selectedInsight

selectedFinding
```

---

# 21. Data State

Ejemplo.

```typescript id="v9k9vq"
queryResult

dashboard

panels
```

---

# 22. Knowledge State

Ejemplo.

```typescript id="igfxqj"
insights

recommendations

findings
```

---

# 23. Store Architecture

Svelte Stores.

---

```text id="n74jhl"
stores/

query.ts

dashboard.ts

insights.ts

recommendations.ts

runtime.ts
```

---

# 24. Query Store

```typescript id="h6l0rd"
currentSQL

history

lastExecution
```

---

# 25. Dashboard Store

```typescript id="d5pr3g"
panels

layout

filters
```

---

# 26. Insight Store

```typescript id="m6tkr8"
insights

selectedInsight
```

---

# 27. Recommendation Store

```typescript id="z20lpx"
recommendations
```

---

# 28. Runtime Store

```typescript id="b3zj9q"
jobs

events

status
```

---

# 29. Component Hierarchy

```text id="n5gk5q"
App

├── QueryWorkspace

├── DashboardView

├── InsightPanel

├── RecommendationPanel

└── InvestigationView
```

---

# 30. Event Flow

```text id="8i8s2x"
SQL Executed
     │
     ▼
API
     │
     ▼
Dashboard Updated
     │
     ▼
Insights Updated
```

---

# 31. Streaming

Ideal para:

```text id="9xhj2f"
Autonomous Analysis
```

---

# 32. WebSocket Layer

```text id="v9tijw"
/ws/runtime
```

---

Enviar.

```text id="rlyz63"
Insight Created

Finding Created

Job Finished
```

---

# 33. UX Principle

Mostrar resultados progresivamente.

---

No esperar:

```text id="b3f0wi"
todo terminado
```

---

# 34. Ejemplo

Paso 1.

```text id="7u7x8z"
Dashboard
```

---

Paso 2.

```text id="h9dkz7"
Insights
```

---

Paso 3.

```text id="3uivg4"
Recommendations
```

---

Paso 4.

```text id="4hrh7j"
Findings
```

---

# 35. Skeleton Loading

Siempre.

---

Evitar:

```text id="t74izs"
pantalla vacía
```

---

# 36. Explainability UI

Cada insight debe expandirse.

---

Mostrar.

```text id="0e4knj"
Reason

Evidence

Confidence

Impact
```

---

# 37. Recommendation UX

Nunca mostrar.

```text id="j9hn9z"
Tal vez...
```

---

Mostrar.

```text id="vr9wye"
Recomendado porque...
```

---

# 38. Finding UX

Formato.

```text id="l4nkpb"
Hallazgo

Evidencia

Impacto

Confianza
```

---

# 39. Layout Principle

Los gráficos NO son protagonistas.

---

Los insights sí.

---

# 40. Dashboard Evolution

BI tradicional.

```text id="1olyo4"
Chart-Centric
```

---

SQLviz.

```text id="s6f5l8"
Insight-Centric
```

---

# 41. Mobile Strategy

Solo lectura.

---

Evitar.

```text id="i2vk1f"
Editor SQL complejo
```

---

# 42. Desktop Strategy

Experiencia completa.

---

Editor.

Dashboard.

Investigaciones.

---

# 43. Accessibility

Todo insight debe existir como texto.

---

Nunca depender solo del gráfico.

---

# 44. Frontend Rule

La UI nunca infiere.

---

La inferencia ocurre en backend.

---

# 45. Frontend Mission

Presentar conocimiento.

---

No generarlo.

---

# 46. Error Handling

Mostrar.

```text id="gzrskx"
Query Error

Inference Error

Runtime Error
```

---

Nunca.

```text id="g57jgi"
Stack Trace
```

---

# 47. Performance Goal

Primera respuesta.

```text id="5qk4mw"
< 1 segundo
```

---

Insights.

```text id="r2l1yv"
< 3 segundos
```

---

Findings.

```text id="4hjc4z"
< 10 segundos
```

---

# 48. Gran Diferencia

La mayoría de BI muestran:

```text id="d6u0h9"
Datos
```

---

SQLviz debe mostrar:

```text id="zxv4vs"
Conocimiento
```

---

# 49. Principio Fundamental

El frontend de SQLviz es una ventana hacia el conocimiento generado por los motores analíticos.

No está diseñado para configurar dashboards.

Está diseñado para consumir análisis.