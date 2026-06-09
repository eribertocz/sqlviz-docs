# SQLviz Technical Specification v1

## Capítulo 21 — Svelte 5 State Model

Versión 0.1

---

# 1. Objetivo

Definir cómo el frontend representa el estado completo de SQLviz.

---

El backend genera conocimiento.

---

El frontend visualiza conocimiento.

---

# 2. Filosofía

La UI nunca reconstruye inferencias.

---

La UI consume inferencias.

---

# 3. Regla Fundamental

Toda decisión proviene del backend.

---

Nunca.

```text
Frontend decide charts
```

---

Nunca.

```text
Frontend decide layouts
```

---

# 4. Arquitectura

```text
FastAPI
 ↓
JSON
 ↓
Stores
 ↓
Components
```

---

# 5. Store Strategy

Utilizar.

```text
Svelte Stores
```

---

# 6. Root Stores

V1.

```text
dashboardStore

executionStore

explanationStore

runtimeStore
```

---

# 7. Dashboard Store

Store principal.

---

```typescript
interface DashboardStore {

   dashboard: Dashboard

   loading: boolean

   error: string | null
}
```

---

# 8. Dashboard Model

```typescript
interface Dashboard {

   id: string

   title: string

   panels: Panel[]

   confidence: number
}
```

---

# 9. Panel Model

```typescript
interface Panel {

   id: string

   title: string

   chart: Chart

   layout: Layout
}
```

---

# 10. Chart Model

```typescript
interface Chart {

   chartType: string

   confidence: number
}
```

---

# 11. Layout Model

```typescript
interface Layout {

   x: number

   y: number

   w: number

   h: number
}
```

---

# 12. Dashboard Flow

```text
SQL
 ↓
API
 ↓
Dashboard Store
 ↓
Dashboard View
```

---

# 13. Execution Store

Representa ejecución.

---

```typescript
interface ExecutionStore {

   executionId: string

   durationMs: number

   timeline: Event[]
}
```

---

# 14. Why?

Observabilidad.

---

Debugging.

---

Explainability.

---

# 15. Event Model

```typescript
interface Event {

   type: string

   timestamp: string

   payload: any
}
```

---

# 16. Explanation Store

Muy importante.

---

```typescript
interface ExplanationStore {

   explanations: Map<string, Explanation>
}
```

---

# 17. Explanation Model

```typescript
interface Explanation {

   confidence: number

   reasoning: string[]

   evidence: Evidence[]
}
```

---

# 18. Evidence Model

```typescript
interface Evidence {

   type: string

   payload: any
}
```

---

# 19. Runtime Store

Monitoreo.

---

```typescript
interface RuntimeStore {

   cacheHitRate: number

   activeJobs: number

   latencyMs: number
}
```

---

# 20. Component Hierarchy

```text
App

 ├─ DashboardPage

 ├─ RuntimePage

 ├─ BenchmarkPage

 └─ ExplainabilityPage
```

---

# 21. Dashboard Page

Contiene.

```text
Toolbar

Dashboard

Inspector
```

---

# 22. Dashboard Component

```text
Dashboard

 ├─ Panel

 ├─ Panel

 └─ Panel
```

---

# 23. Panel Component

```text
Panel

 ├─ Title

 ├─ Chart

 └─ Explain Button
```

---

# 24. Explain Button

Siempre visible.

---

```text
ⓘ Why?
```

---

# 25. Why?

Principio de explicabilidad.

---

# 26. Explanation Drawer

Panel lateral.

---

```text
Confidence

Reasoning

Evidence

Scores
```

---

# 27. Dashboard Inspector

Modo avanzado.

---

Muestra.

```text
Metrics

Dimensions

Intent

Chart
```

---

# 28. Execution Timeline View

Muy útil.

---

```text
QUERY_PARSED

FEATURES_EXTRACTED

SEMANTICS_INFERRED

CHARTS_INFERRED

DASHBOARD_CREATED
```

---

# 29. Timeline Component

```typescript
<Timeline />
```

---

# 30. Why?

Debugging.

---

Aprendizaje.

---

# 31. Inference Tree

Característica estrella.

---

```text
SQL

 ↓

Features

 ↓

Semantics

 ↓

Intent

 ↓

Chart
```

---

# 32. Inference Node

```typescript
interface InferenceNode {

   id: string

   label: string

   children: InferenceNode[]
}
```

---

# 33. Why?

Mostrar razonamiento.

---

# 34. Loading State

Siempre.

---

```text
Parsing SQL...

Extracting Features...

Inferring Intent...

Generating Dashboard...
```

---

# 35. Error State

```typescript
interface ErrorState {

   code: string

   message: string
}
```

---

# 36. Dashboard Grid

Usar.

```text
CSS Grid
```

---

# 37. Grid Mapping

```text
12 columns
```

---

Consistente con Layout Engine.

---

# 38. Responsive Strategy

V1.

```text
Desktop First
```

---

# 39. Mobile Strategy

V2.

---

No prioritaria.

---

# 40. Benchmark Page

Visualiza.

```text
Accuracy

Latency

Hit Rate
```

---

# 41. Runtime Page

Visualiza.

```text
Jobs

Events

Cache

Metrics
```

---

# 42. Theme Strategy

Muy importante.

---

# 43. Default Theme

```text
Dark
```

---

# 44. Why?

Charts lucen mejor.

---

Dashboards lucen mejor.

---

# 45. Explainability Theme

Mostrar.

```text
Confidence Colors

Evidence Badges

Reasoning Cards
```

---

# 46. Future Learning UI

Reservar espacio.

---

```text
Recommendations

Learned Patterns

Genome Explorer
```

---

# 47. Future Plugin UI

```text
Plugin Marketplace
```

---

# 48. Future Analytics UI

```text
Inference Analytics
```

---

# 49. Performance Goal

Dashboard render.

```text
< 100 ms
```

---

# 50. Store Rule

Nunca almacenar.

```text
SQL Results gigantes
```

---

Solo.

```text
Metadata

Dashboard

Inference
```

---

# 51. Component Rule

Componentes.

```text
Tontos
```

---

Stores.

```text
Inteligentes
```

---

# 52. API Rule

Toda información proviene de:

```text
FastAPI
```

---

# 53. Strategic Insight

La mayoría de herramientas muestran dashboards.

---

SQLviz debe mostrar dashboards y razonamiento.

---

# 54. UX Goal

El usuario no solo ve:

```text
Line Chart
```

---

También ve:

```text
¿Por qué es un Line Chart?
```

---

# 55. Strategic Value

La explicabilidad es parte de la UI.

---

No un panel secundario.

---

# 56. Future V2

```text
Collaborative Dashboards

Annotations

Comments
```

---

# 57. Future V3

```text
AI Analyst

Conversation Layer

Autonomous Insights
```

---

# 58. MVP UI

Con esto queda definido.

```text
Dashboard

Explainability

Timeline

Runtime
```

---

# 59. Estado Final

Frontend completamente especificado.

---

# 60. Principio Fundamental

La UI de SQLviz no muestra visualizaciones.

Muestra inferencias visualizadas.