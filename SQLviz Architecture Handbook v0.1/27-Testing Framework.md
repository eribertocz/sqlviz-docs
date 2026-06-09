# SQLviz Architecture Handbook

## Capítulo 27 — Testing Framework

Versión: 0.1

---

# 1. Introducción

¿Qué significa que SQLviz funcione correctamente?

---

Pregunta difícil.

---

Porque no basta con que:

```text id="6l7q2u"
No lance errores
```

---

También debe:

```text id="s6w7v1"
Inferir correctamente
```

---

# 2. Filosofía

No probar:

```text id="7tk3x5"
Código
```

---

Probar:

```text id="o9g8t1"
Comportamiento Analítico
```

---

# 3. Testing Pyramid

```text id="i8v2m5"
E2E Tests

Integration Tests

Inference Tests

Unit Tests
```

---

# 4. Unit Tests

Validan componentes aislados.

---

Ejemplo.

```python id="l1d3j7"
Semantic Engine
```

---

Input.

```text id="o7m6y2"
revenue
```

---

Output esperado.

```text id="x4n8r1"
METRIC_REVENUE
```

---

# 5. Intent Tests

Input.

```sql id="m7p2z4"
SELECT
 month,
 SUM(revenue)
FROM sales
GROUP BY month
```

---

Output esperado.

```text id="z6t1k8"
TREND_ANALYSIS
```

---

# 6. Chart Tests

Input.

```sql id="n4w9q3"
SELECT
 month,
 SUM(revenue)
FROM sales
GROUP BY month
```

---

Output esperado.

```text id="g5v2y6"
LINE_CHART
```

---

# 7. Inference Tests

Nueva categoría.

---

Muy importante.

---

Validan:

```text id="y8q1w4"
Semántica

Intención

Visualización

Insights
```

---

# 8. Test Corpus

Crear dataset oficial.

---

```text id="g3r6n9"
100 SQL
```

↓

```text id="m8k2t5"
Resultado esperado
```

---

# 9. Example Corpus

```yaml id="z1p7w3"
sql: |
  SELECT month,
         SUM(revenue)
  FROM sales
  GROUP BY month

expected:

  intent:
    trend

  chart:
    line
```

---

# 10. Gold Standard

Concepto fundamental.

---

Crear.

```text id="s9n5v7"
Ground Truth
```

---

para inferencias.

---

# 11. Gold Dataset

```text id="7m2kq8"
500 consultas

1000 consultas

5000 consultas
```

---

a largo plazo.

---

# 12. Regression Testing

Problema.

---

Mejoras Chart Engine.

↓

Rompe 40 casos.

---

Necesitamos detectarlo.

---

# 13. Regression Suite

Ejecutar siempre.

```text id="q3w6t9"
Corpus completo
```

---

antes de release.

---

# 14. Accuracy Metrics

Medir.

```text id="f5k8n2"
Chart Accuracy

Intent Accuracy

Insight Accuracy
```

---

# 15. Chart Accuracy

```python id="x7m4q1"
correct
/
total
```

---

# 16. Intent Accuracy

```python id="u2p8w5"
correct
/
total
```

---

# 17. Insight Accuracy

Más difícil.

---

Necesita datasets controlados.

---

# 18. Synthetic Data

Crear.

```text id="v6t3m9"
Revenue Growth

Revenue Drop

Anomaly

Seasonality
```

---

con respuestas conocidas.

---

# 19. Insight Validation

Ejemplo.

---

Dataset contiene.

```text id="n8w1q7"
Revenue ↓ 25%
```

---

SQLviz debe detectar.

```text id="k4t9x2"
Revenue cayó
```

---

# 20. Recommendation Tests

Validar.

```text id="w7q2n5"
Relevancia
```

---

# 21. Recommendation Benchmark

Guardar.

```text id="r5m8k3"
Recommendation Acceptance Rate
```

---

# 22. Autonomous Analysis Tests

Muy importante.

---

Input.

```text id="y3t7v1"
Revenue ↓
```

---

Output.

```text id="g6n2p4"
Root Cause
```

---

# 23. Root Cause Benchmark

Dataset preparado.

---

Con causa conocida.

---

# 24. Example

```text id="h8v4m6"
Argentina
```

explica.

```text id="q2k9t7"
80%
```

de caída.

---

SQLviz debe descubrirlo.

---

# 25. Explainability Tests

Toda inferencia debe producir.

```text id="m4n7w8"
Explanation
```

---

# 26. Validation Rule

Nunca permitir.

```text id="c9q2v5"
Resultado
sin explicación
```

---

# 27. Integration Tests

Pipeline completo.

---

```text id="q6w3t8"
SQL
 ↓
Semantic
 ↓
Intent
 ↓
Chart
```

---

# 28. Runtime Tests

Validar.

```text id="z7m4k1"
Eventos

Jobs

Workers
```

---

# 29. Event Tests

Ejemplo.

```text id="j3n8v6"
INSIGHT_CREATED
```

↓

```text id="w2q7t4"
Recommendation Generated
```

---

# 30. Cache Tests

Validar.

```text id="u9m5k2"
Hit

Miss

Expiration
```

---

# 31. Repository Tests

Validar.

```text id="v4q8n1"
DuckDB Persistence
```

---

# 32. API Tests

Validar.

```text id="r8m2k6"
/dashboard

/query

/insights
```

---

# 33. E2E Tests

Escenario completo.

---

Usuario.

```sql id="p6w4n8"
SELECT ...
```

---

Resultado.

```text id="y1k7t3"
Dashboard

Insights

Recommendations
```

---

# 34. Performance Tests

Medir.

```text id="s5n8v2"
Latencia
```

---

# 35. Objetivos

```text id="m2q9k5"
Dashboard < 1s

Insights < 3s

Findings < 10s
```

---

# 36. Load Tests

Simular.

```text id="z4v7m1"
10 usuarios

100 usuarios

1000 usuarios
```

---

# 37. Stress Tests

Validar.

```text id="w6n2k8"
Datasets grandes
```

---

# 38. Mutation Testing

Idea avanzada.

---

Cambiar reglas.

↓

Verificar.

↓

Tests fallan.

---

# 39. Confidence Testing

Validar.

```text id="g8m4t2"
Confidence Scores
```

---

No solo resultado.

---

También score.

---

# 40. Learning Tests

Validar.

```text id="f2q7n5"
Feedback
```

↓

```text id="u4m8k1"
Modelo mejora
```

---

# 41. Benchmark Registry

DuckDB.

```sql id="b7n4k9"
CREATE TABLE benchmarks (

    benchmark_id VARCHAR,

    metric_name VARCHAR,

    metric_value DOUBLE,

    created_at TIMESTAMP
);
```

---

# 42. Test Corpus Registry

```sql id="p2m8v6"
CREATE TABLE test_cases (

    case_id VARCHAR,

    sql TEXT,

    expected JSON
);
```

---

# 43. Release Gates

No publicar.

---

Si.

```text id="r7n2m4"
Chart Accuracy ↓

Intent Accuracy ↓

Insight Accuracy ↓
```

---

# 44. Strategic Insight

La mayoría de software BI prueba:

```text id="q8v3n7"
Funcionalidad
```

---

SQLviz debe probar:

```text id="k1m6t9"
Conocimiento
```

---

# 45. Analytical CI/CD

Pipeline.

```text id="v5q8m2"
Tests

Benchmarks

Inference Validation

Performance Validation
```

---

antes de cada release.

---

# 46. The Hidden Challenge

Un bug visual.

---

Molesta.

---

Un bug analítico.

---

Miente.

---

Y eso es mucho más grave.

---

# 47. Scientific Principle

Toda inferencia importante debe poder:

```text id="n3q7v8"
Medirse

Compararse

Validarse
```

---

# 48. Testing Layers

```text id="s9m2k4"
Code Layer

Engine Layer

Inference Layer

Knowledge Layer
```

---

# 49. El Gran Cambio

Software tradicional.

```text id="x5q8v3"
¿Funciona?
```

---

SQLviz.

```text id="h7m2n6"
¿Funciona?

¿Infiere correctamente?

¿Puede demostrarlo?
```

---

# 50. Principio Fundamental

El Testing Framework protege la credibilidad de SQLviz.

Porque cuando un sistema empieza a generar conocimiento, no basta con ejecutar código correctamente.

Debe demostrar que sus conclusiones son confiables.