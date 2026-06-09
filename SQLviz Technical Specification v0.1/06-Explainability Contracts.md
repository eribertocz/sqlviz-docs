# SQLviz Technical Specification v1

## Capítulo 06 — Explainability Contracts

Versión 0.1

---

# 1. Objetivo

Convertir la explicabilidad en una obligación arquitectónica.

---

No una característica opcional.

---

No un "nice to have".

---

Un requisito del sistema.

---

# 2. Filosofía

La mayoría de herramientas hacen:

```text
Resultado
```

---

SQLviz debe hacer:

```text
Resultado

↓

Razón

↓

Evidencia

↓

Score
```

---

# 3. Regla Fundamental

Toda inferencia importante debe producir:

```python
Explanation
```

---

Siempre.

---

# 4. Objetos Obligatorios

V1.

```text
SemanticTag

Intent

Metric

Dimension

Chart

Layout

Dashboard
```

---

Todos deben explicar.

---

# 5. Prohibido

```python
chart = "line"
```

---

# 6. Obligatorio

```python
chart = "line"

explanation = ...
```

---

# 7. Base Contract

```python
class Explainable:

    explanation: Explanation
```

---

# 8. Explanation Model

```python
class Explanation:

    explanation_id: str

    confidence: float

    reasoning: list

    evidence: list

    scores: dict
```

---

# 9. Explanation ID

Siempre.

```python
uuid4()
```

---

# 10. Confidence

Siempre.

---

Rango.

```python
0.0
```

a

```python
1.0
```

---

# 11. Reasoning

Describe.

```text
Cómo se tomó la decisión
```

---

# 12. Ejemplo

```text
Date dimension detected

Aggregation detected

Time grouping detected
```

---

# 13. Evidence

Representa pruebas.

---

No opiniones.

---

# 14. Ejemplo

```text
GROUP BY month

SUM(revenue)

Column type DATE
```

---

# 15. Scores

Guardar contribución.

---

Ejemplo.

```python
{
  "date_score":0.95,
  "aggregation_score":0.90
}
```

---

# 16. Evidence Model

```python
class Evidence:

    evidence_id: str

    evidence_type: str

    payload: dict
```

---

# 17. Evidence Types

V1.

```text
SQL_EVIDENCE

COLUMN_EVIDENCE

FEATURE_EVIDENCE

SEMANTIC_EVIDENCE

STATISTIC_EVIDENCE
```

---

# 18. SQL Evidence

Ejemplo.

```text
GROUP BY
```

---

```text
ORDER BY
```

---

```text
LIMIT
```

---

# 19. Column Evidence

Ejemplo.

```text
order_date
```

---

```text
revenue
```

---

# 20. Feature Evidence

Ejemplo.

```text
HAS_GROUP_BY

HAS_DATE_COLUMN
```

---

# 21. Semantic Evidence

Ejemplo.

```text
Revenue

Country

Product
```

---

# 22. Statistic Evidence

Ejemplo.

```text
Cardinality

Distinct Ratio

Null Ratio
```

---

# 23. Explainable SemanticTag

```python
SemanticTag(

   name="Revenue",

   confidence=0.96,

   explanation=...
)
```

---

# 24. Explainable Intent

```python
Intent(

   name="Trend",

   score=0.93,

   explanation=...
)
```

---

# 25. Explainable Chart

```python
Chart(

   type="Line",

   confidence=0.95,

   explanation=...
)
```

---

# 26. Explainable Layout

```python
PanelLayout(

   size="xl",

   explanation=...
)
```

---

# 27. Explainable Dashboard

Dashboard también explica.

---

# 28. Pregunta

```text
¿Por qué existen estos paneles?
```

---

Debe responderse.

---

# 29. Dashboard Explanation

Ejemplo.

```text
Revenue fue identificada
como métrica principal.

Trend fue identificada
como intención dominante.
```

---

# 30. Engine Contract

Todos los engines producen explicaciones.

---

Sin excepción.

---

# 31. Semantic Engine

Debe explicar.

```text
Por qué detectó Revenue
```

---

# 32. Intent Engine

Debe explicar.

```text
Por qué detectó Trend
```

---

# 33. Chart Engine

Debe explicar.

```text
Por qué eligió Line
```

---

# 34. Layout Engine

Debe explicar.

```text
Por qué eligió XL
```

---

# 35. Explanation Levels

V1.

```text
Object Level
```

---

V2.

```text
Pipeline Level
```

---

# 36. Object Level

Explicación individual.

---

Ejemplo.

```text
Line Chart
```

---

# 37. Pipeline Level

Explicación completa.

---

Ejemplo.

```text
SQL

↓

Features

↓

Semantic

↓

Intent

↓

Chart
```

---

# 38. Trace Model

```python
class Trace:

    nodes: list

    edges: list
```

---

# 39. Benefit

Permite auditoría.

---

# 40. Explanation Registry

DuckDB.

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

# 41. Evidence Registry

```sql
CREATE TABLE evidences (

    evidence_id VARCHAR,

    explanation_id VARCHAR,

    evidence_type VARCHAR,

    payload JSON
);
```

---

# 42. Trace Registry

```sql
CREATE TABLE traces (

    trace_id VARCHAR,

    execution_id VARCHAR,

    payload JSON
);
```

---

# 43. Confidence Contract

Todos los scores deben ser reproducibles.

---

# 44. Prohibido

```python
random()
```

---

para scores.

---

# 45. Determinismo

Mismo SQL.

↓

Misma explicación.

---

# 46. Explainability API

```text
GET /explanations/{id}
```

---

# 47. Response

```json
{
  "confidence":0.95,
  "reasoning":[...],
  "evidence":[...]
}
```

---

# 48. UI Contract

Toda inferencia importante tiene.

```text
ⓘ Why?
```

---

# 49. Expand View

Mostrar.

```text
Confidence

Evidence

Rules

Scores
```

---

# 50. Explainability Score

Métrica interna.

---

```python
score =
(
 evidence_count * 0.4
 +
 reasoning_count * 0.3
 +
 trace_depth * 0.3
)
```

---

# 51. Explainability Coverage

Medir.

```python
explained_objects
/
total_objects
```

---

Objetivo.

```text
100%
```

---

# 52. Testing Rule

Todo test importante valida.

```text
Resultado
```

y

```text
Explicación
```

---

# 53. Ejemplo

No basta.

```text
Line Chart
```

---

También.

```text
Porque existe DATE
y agregación temporal.
```

---

# 54. Strategic Value

La mayoría de sistemas generan respuestas.

---

SQLviz genera:

```text
Respuestas justificadas
```

---

# 55. Principio Fundamental

La explicabilidad no es una capa adicional.

Es parte del contrato de dominio.

Si una inferencia no puede explicarse, para SQLviz esa inferencia está incompleta.