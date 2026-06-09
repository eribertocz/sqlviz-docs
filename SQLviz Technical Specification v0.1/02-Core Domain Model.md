SQLviz Technical Specification v1
Capítulo 02 — Core Domain Model

Versión 0.1

1. Objetivo

Definir las entidades fundamentales del dominio.

Todo SQLviz debe construirse sobre este modelo.

Si este modelo es incorrecto:

Semantic Engine falla

Intent Engine falla

Chart Engine falla

Layout Engine falla
2. Principio

SQLviz NO gira alrededor de charts.

SQLviz gira alrededor de:

Knowledge Objects
3. Entidades Principales

V1.

Query

Feature

SemanticTag

Intent

Metric

Dimension

Chart

Panel

Dashboard

Explanation
4. Relación General
Query
  │
  ▼
Feature
  │
  ▼
SemanticTag
  │
  ▼
Intent
  │
  ▼
Chart
  │
  ▼
Panel
  │
  ▼
Dashboard
5. Query

Representa SQL enviado por usuario.

6. Query Model
class Query:

    query_id: str

    sql: str

    created_at: datetime
7. Query Rule

Query es inmutable.

Nunca modificar.

8. Query Fingerprint

Toda Query genera.

fingerprint: str

Ejemplo.

TIME_SERIES_AGGREGATION
9. Feature

Representa señal extraída.

No conocimiento.

Señal.

10. Ejemplos
HAS_GROUP_BY

HAS_ORDER_BY

HAS_LIMIT

HAS_DATE_COLUMN

HAS_AGGREGATION
11. Feature Model
class Feature:

    feature_id: str

    name: str

    value: any

    confidence: float
12. SemanticTag

Representa significado.

13. Ejemplos
Revenue

Profit

Customer

Country

Product
14. Semantic Model
class SemanticTag:

    tag_id: str

    name: str

    confidence: float
15. Semantic Rule

Una columna puede tener varios tags.

Ejemplo.

customer_country

↓

Country 0.95

Location 0.82
16. Intent

Representa objetivo analítico.

17. Ejemplos
Trend Analysis

Comparison

Ranking

Composition

Distribution

Correlation
18. Intent Model
class Intent:

    intent_id: str

    name: str

    score: float
19. Intent Rule

Una Query puede tener múltiples intenciones.

20. Example
Trend
0.95

Comparison
0.70
21. Metric

Representa medida numérica.

22. Ejemplos
Revenue

Profit

Cost

Quantity
23. Metric Model
class Metric:

    metric_id: str

    name: str

    expression: str

    semantic_tag: str
24. Dimension

Representa eje de análisis.

25. Ejemplos
Date

Country

Product

Segment
26. Dimension Model
class Dimension:

    dimension_id: str

    name: str

    datatype: str

    cardinality: int
27. Chart

Representa visualización inferida.

28. Chart Types V1
Line

Bar

Pie

Scatter

KPI

Table
29. Chart Model
class Chart:

    chart_id: str

    chart_type: str

    confidence: float
30. Chart Explainability

Siempre asociado a.

Explanation
31. Panel

Unidad visual.

32. Panel Model
class Panel:

    panel_id: str

    title: str

    chart: Chart

    metrics: list

    dimensions: list
33. Dashboard

Colección de paneles.

34. Dashboard Model
class Dashboard:

    dashboard_id: str

    panels: list[Panel]
35. Dashboard Rule

Dashboard es resultado.

Nunca entrada.

36. Explanation

Objeto transversal.

37. Propósito

Explicar inferencias.

38. Explanation Model
class Explanation:

    explanation_id: str

    confidence: float

    evidence: list

    reasoning: list
39. Evidence

Representa prueba.

Ejemplo.

Date column detected

GROUP BY month

SUM(revenue)
40. Evidence Model
class Evidence:

    evidence_id: str

    type: str

    payload: dict
41. RuntimeContext

Objeto compartido.

Todos los engines reciben.

RuntimeContext
42. RuntimeContext Model
class RuntimeContext:

    query: Query

    features: list[Feature]

    semantics: list[SemanticTag]

    intents: list[Intent]
43. Evolución

Cada engine añade información.

44. Flujo
Query
 ↓
Features
 ↓
Semantics
 ↓
Intent
 ↓
Chart
 ↓
Panel
 ↓
Dashboard
45. Principio

Los engines producen objetos de dominio.

No JSON arbitrario.

46. Beneficio

Tipado fuerte.

Menos bugs.

Más trazabilidad.

47. Ownership

Cada entidad tiene un dueño.

Feature Engine
  ↓
  Feature

Semantic Engine
  ↓
  SemanticTag

Intent Engine
  ↓
  Intent

Chart Engine
  ↓
  Chart
48. Dependencias Permitidas
Feature
 ↓
Semantic

Semantic
 ↓
Intent

Intent
 ↓
Chart

Chart
 ↓
Panel
49. Dependencias Prohibidas

Nunca.

Chart
 ↓
Semantic

Ni.

Dashboard
 ↓
Feature
50. Principio Fundamental

El Core Domain Model es el lenguaje interno de SQLviz.

Los engines no intercambian JSON.

No intercambian diccionarios.

Intercambian objetos de dominio explícitos.

Eso hace que el sistema sea mantenible, trazable y extensible.