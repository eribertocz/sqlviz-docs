# SQLviz Architecture Handbook

## Capítulo 25 — Explainability Framework

Versión: 0.1

---

# 1. Introducción

Este capítulo es probablemente uno de los más importantes de todo SQLviz.

---

La mayoría de herramientas modernas tienen un problema.

---

Generan respuestas.

Pero no explican:

```text
¿Por qué?
```

---

Ejemplos.

```text
Este gráfico fue seleccionado.

¿Por qué?
```

---

```text
Este insight fue generado.

¿Por qué?
```

---

```text
Esta recomendación apareció.

¿Por qué?
```

---

Si SQLviz quiere ser confiable, debe ser explicable.

---

# 2. Filosofía

Regla absoluta:

```text
Toda inferencia debe poder explicarse.
```

---

Nunca:

```text
Caja Negra
```

---

Siempre:

```text
Resultado
↓
Evidencia
↓
Reglas
↓
Scores
```

---

# 3. Explainability Scope

Explicar:

```text
Chart Selection

Layout Selection

Insight Generation

Recommendation Generation

Finding Generation

Filter Inference

Relationship Inference
```

---

# 4. Explainability Model

Modelo base.

```python
class Explanation:

    explanation_id: str

    object_type: str

    object_id: str

    reasoning: list

    evidence: list

    confidence: float
```

---

# 5. Reasoning Chain

Toda decisión conserva:

```text
Inputs

Reglas

Scores

Resultado
```

---

Ejemplo.

```text
Columna DATE detectada
+
Métrica agregada
+
GROUP BY temporal
```

↓

```text
Line Chart
```

---

# 6. Chart Explainability

Ejemplo.

```text
Chart Type:
Line
```

---

Razón.

```text
Temporal dimension detected

Score: 0.95
```

---

Evidencia.

```text
order_date

GROUP BY month

SUM(revenue)
```

---

# 7. Chart Explanation Object

```json
{
  "chart":"line",
  "confidence":0.95,
  "evidence":[
    "date_dimension",
    "time_grouping",
    "aggregated_metric"
  ]
}
```

---

# 8. Layout Explainability

Pregunta.

```text
¿Por qué este panel es grande?
```

---

Respuesta.

```text
Primary KPI

High Insight Score

High Business Impact
```

---

# 9. Layout Explanation

```json
{
  "panel":"revenue",
  "size":"xl",
  "reasons":[
    "primary_metric",
    "high_priority"
  ]
}
```

---

# 10. Insight Explainability

Insight.

```text
Revenue cayó 22%
```

---

Explicación.

```text
Comparación entre períodos.

Impacto alto.

Variación significativa.
```

---

# 11. Insight Evidence

Debe incluir.

```text
Current Value

Previous Value

Difference

Percentage
```

---

# 12. Example

```json
{
  "current":780000,
  "previous":1000000,
  "delta":-220000,
  "delta_pct":-22
}
```

---

# 13. Recommendation Explainability

Pregunta.

```text
¿Por qué recomiendas Product?
```

---

Respuesta.

```text
Alta correlación histórica.

Frecuentemente analizado.

Impacto esperado alto.
```

---

# 14. Recommendation Model

```json
{
  "recommendation":"product",
  "score":0.91,
  "reasons":[
    "historical_success",
    "high_variance"
  ]
}
```

---

# 15. Finding Explainability

Crítico.

---

Ejemplo.

```text
Argentina explica 78%
de la caída.
```

---

Usuario debe poder abrir.

---

# 16. Finding Drilldown

```text
Revenue ↓ 22%

↓
Argentina ↓ 35%

↓
Electronics ↓ 50%

↓
Retail ↓ 61%
```

---

# 17. Investigation Trace

Toda investigación genera un árbol.

---

```python
class InvestigationTrace:

    root_metric: str

    nodes: list

    edges: list
```

---

# 18. Evidence Registry

DuckDB.

```sql
CREATE TABLE evidence (

    evidence_id VARCHAR,

    object_type VARCHAR,

    object_id VARCHAR,

    payload JSON
);
```

---

# 19. Explanation Registry

```sql
CREATE TABLE explanations (

    explanation_id VARCHAR,

    object_type VARCHAR,

    object_id VARCHAR,

    confidence DOUBLE,

    payload JSON
);
```

---

# 20. Rule Registry

Muy importante.

---

Guardar reglas ejecutadas.

---

```sql
CREATE TABLE rule_executions (

    execution_id VARCHAR,

    rule_name VARCHAR,

    score DOUBLE,

    payload JSON
);
```

---

# 21. Example Rule

```text
Temporal Line Rule
```

---

Payload.

```json
{
  "date_dimension":true,
  "aggregation":true,
  "score":0.95
}
```

---

# 22. Explainability UI

Cada objeto debe tener:

```text
ⓘ Why?
```

---

Al expandir.

---

Mostrar:

```text
Evidence

Rules

Scores

Confidence
```

---

# 23. Confidence Model

Todos los motores producen.

```python
confidence
```

---

Rango.

```text
0.0 - 1.0
```

---

# 24. Confidence Interpretation

```text
0.95+
Muy Alta

0.85+
Alta

0.70+
Media

<0.70
Baja
```

---

# 25. Explainability Contract

Todos los engines devuelven.

```python
EngineResult
```

*

```python
Explanation
```

---

# 26. Engine Output

```python
class EngineResult:

    payload: dict

    explanation: Explanation
```

---

# 27. Audit Trail

Pregunta futura.

```text
¿Por qué este dashboard existe?
```

---

Respuesta.

```text
Consultar trazabilidad
```

---

# 28. Trace Graph

```text
SQL
 │
 ▼
Semantic
 │
 ▼
Intent
 │
 ▼
Chart
 │
 ▼
Insight
```

---

# 29. Full Traceability

Cada objeto conoce.

```text
Parent Object

Source Query

Source Rule

Source Evidence
```

---

# 30. Explainability API

Endpoint.

```text
GET /explanations/{id}
```

---

Retorna.

```json
{
  "reasoning":[],
  "evidence":[],
  "confidence":0.94
}
```

---

# 31. Learning Integration

Registrar.

```text
Explanation Viewed
```

---

```text
Explanation Ignored
```

---

# 32. Valuable Metric

Medir.

```text
Explanation Open Rate
```

---

Porque indica confianza.

---

# 33. Explainability Score

```python
score =
(
 evidence_quality * 0.4
 +
 trace_depth * 0.3
 +
 rule_coverage * 0.3
)
```

---

# 34. Autonomous Analysis

Toda investigación conserva.

```text
Query usada

Dimensión analizada

Score obtenido

Motivo
```

---

# 35. Explainability Principle

Nunca mostrar:

```text
Porque el algoritmo lo decidió.
```

---

Siempre mostrar:

```text
Porque observamos estas evidencias.
```

---

# 36. Gran Diferencia

BI tradicional.

```text
Visualización
```

---

SQLviz.

```text
Visualización
+
Explicación
```

---

# 37. El Cambio Filosófico

La confianza no proviene de:

```text
Tener razón.
```

---

La confianza proviene de:

```text
Poder explicar.
```

---

# 38. Explainability Layers

```text
Evidence Layer

Rule Layer

Scoring Layer

Decision Layer
```

---

# 39. Explainability Rule

Toda inferencia debe ser:

```text
Observable

Trazable

Reproducible
```

---

# 40. Principio Fundamental

El Explainability Framework convierte SQLviz de un sistema que genera respuestas en un sistema que justifica sus respuestas.

Sin explainability:

```text
Inferencia.
```

Con explainability:

```text
Inferencia confiable.
```