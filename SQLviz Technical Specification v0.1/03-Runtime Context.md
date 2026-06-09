SQLviz Technical Specification v1
Capítulo 03 — Runtime Context

Versión 0.1

1. Objetivo

Definir el objeto que transporta el conocimiento a través del pipeline de inferencia.

Todos los engines reciben:

RuntimeContext

Todos los engines devuelven:

RuntimeContext
2. Filosofía

Los engines no deben comunicarse entre sí.

Malo:

semantic_engine.call(intent_engine)

Peor:

intent_engine.call(chart_engine)

Bueno:

Engine
  ↓
RuntimeContext
  ↓
Engine
3. Pipeline
SQL
 ↓
Parser
 ↓
Feature
 ↓
Semantic
 ↓
Intent
 ↓
Chart
 ↓
Layout
 ↓
Dashboard
4. RuntimeContext

Es la memoria temporal de una ejecución.

5. RuntimeContext Model
class RuntimeContext:

    execution_id: str

    query: Query

    fingerprint: str

    features: list[Feature]

    semantic_tags: list[SemanticTag]

    intents: list[Intent]

    metrics: list[Metric]

    dimensions: list[Dimension]

    charts: list[Chart]

    explanations: list[Explanation]
6. Regla Fundamental

RuntimeContext vive únicamente durante una ejecución.

No es persistente.

7. Persistencia

Si queremos guardar algo:

Se transforma en:

Dashboard

Cache

Benchmark

Learning Event
8. Execution ID

Toda ejecución recibe.

uuid4()

Ejemplo.

run_8d3f2e...
9. Beneficio

Trazabilidad completa.

10. Query

Siempre existe.

Es obligatoria.

11. Fingerprint

Siempre existe después del parser.

Ejemplo.

TIME_SERIES_AGGREGATION
12. Features

Generadas por Feature Engine.

Ejemplo.

HAS_GROUP_BY

HAS_AGGREGATION

HAS_DATE_COLUMN
13. Semantic Tags

Generadas por Semantic Engine.

Ejemplo.

Revenue

Country

Product
14. Intents

Generadas por Intent Engine.

Ejemplo.

Trend
0.92

Ranking
0.35
15. Metrics

Generadas por Metric Engine.

Ejemplo.

Revenue

Profit
16. Dimensions

Generadas por Dimension Engine.

Ejemplo.

Month

Country
17. Charts

Generadas por Chart Engine.

Ejemplo.

Line
0.94

Bar
0.62
18. Explanations

Acumulativas.

Cada engine añade explicaciones.

19. Ejemplo

Semantic Engine.

Añade.

Column revenue

matches revenue dictionary
20. Context Lifecycle

Inicio.

Query

Fin.

Dashboard
21. State Evolution

Estado inicial.

RuntimeContext(
    query=query
)
22. Después del Parser
context.fingerprint

Completo.

23. Después de Features
context.features

Completo.

24. Después de Semantic
context.semantic_tags

Completo.

25. Después de Intent
context.intents

Completo.

26. Inmutabilidad Parcial

Regla importante.

Un engine puede:

Agregar

Nunca:

Eliminar

datos previos.

27. Ejemplo

Semantic Engine.

Puede añadir.

semantic_tags.append(...)

No puede borrar.

features.clear()
28. Reason

Evitar efectos colaterales.

29. Confidence Registry

Todos los objetos tienen score.

Ejemplo.

Intent(
    name="trend",
    score=0.93
)
30. Ranking

RuntimeContext siempre mantiene rankings.

No solo ganador.

31. Example
Line 0.95

Bar 0.70

Scatter 0.20
32. Benefit

Explainability.

33. Metadata Section

Añadir.

metadata: dict
34. Uso

Información auxiliar.

Ejemplo.

Row Count

Execution Time

Dataset Size
35. Statistics Section

Añadir.

statistics: dict
36. Ejemplo
{
   "row_count": 145000,
   "column_count": 7
}
37. Runtime Warnings

Añadir.

warnings: list
38. Ejemplos
High Cardinality

Missing Dates

Sparse Dataset
39. Runtime Errors

Añadir.

errors: list
40. Error Philosophy

Capturar.

No lanzar excepción inmediatamente.

41. Example

Chart Engine.

No encuentra chart ideal.

Añade warning.

LOW_CONFIDENCE_CHART
42. Explainability Section

Muy importante.

Cada engine agrega.

Explanation
43. Explanation Tree

Permite reconstruir.

Por qué ocurrió algo
44. RuntimeContext Snapshot

Permitir serialización.

45. Snapshot Model
context.to_json()
46. Uso

Debugging.

Benchmarks.

Learning.

47. Audit Trail

Guardar snapshots.

Cuando necesario.

48. Benchmark Integration

Corpus puede almacenar.

Input Context

Output Context
49. Future Compatibility

RuntimeContext debe soportar.

Genome

Learning

Recommendations

Plugins

sin cambios mayores.

50. Principio Fundamental

RuntimeContext es el sistema nervioso de SQLviz.

Los engines no conocen a otros engines.

Los engines conocen únicamente el contexto.

Eso reduce acoplamiento, facilita testing y permite agregar nuevos motores sin modificar los existentes.