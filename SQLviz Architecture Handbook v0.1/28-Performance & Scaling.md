# SQLviz Architecture Handbook

## Capítulo 28 — Performance & Scaling

Versión: 0.1

---

# 1. Introducción

La mayoría de proyectos BI mueren por una razón:

```text
Funcionan bien con 1 millón de filas.
```

---

Pero luego llegan:

```text
100 millones

500 millones

1 billón
```

---

Y todo colapsa.

---

SQLviz tiene un desafío adicional.

No solo ejecuta consultas.

También ejecuta:

```text
Inferencia

Insights

Recomendaciones

Investigaciones

Aprendizaje
```

---

# 2. Objetivo

Mantener experiencia interactiva.

---

Incluso cuando:

```text
Datos crecen

Usuarios crecen

Motores crecen
```

---

# 3. Filosofía

Primera regla.

---

No optimizar consultas.

---

Optimizar trabajo repetido.

---

# 4. Regla Fundamental

La operación más rápida es:

```text
No ejecutar nada.
```

---

# 5. Performance Layers

```text
Query Layer

Inference Layer

Insight Layer

Dashboard Layer

Runtime Layer
```

---

# 6. Cache Strategy

SQLviz necesita múltiples caches.

---

No una sola.

---

# 7. Query Cache

Cachear resultado SQL.

---

Clave.

```python
hash(sql)
```

---

# 8. Fingerprint Cache

Más importante.

---

Cachear:

```text
Fingerprint
```

---

No resultado.

---

# 9. Ejemplo

Estas consultas.

```sql
SELECT month,
       SUM(revenue)
FROM sales
GROUP BY month
```

---

```sql
SELECT date,
       SUM(revenue)
FROM orders
GROUP BY date
```

---

Comparten.

```text
Trend Fingerprint
```

---

# 10. Beneficio

No recalcular.

```text
Intent

Chart

Layout
```

---

# 11. Semantic Cache

Guardar.

```text
Revenue

Country

Product
```

---

Semántica detectada.

---

# 12. Insight Cache

Guardar insights.

---

Ejemplo.

```text
Revenue cayó 15%
```

---

No regenerar.

---

# 13. Recommendation Cache

Guardar recomendaciones exitosas.

---

# 14. Dashboard Cache

Guardar dashboards completos.

---

# 15. Cache Registry

DuckDB.

```sql
CREATE TABLE cache_entries (

    cache_key VARCHAR,

    cache_type VARCHAR,

    payload JSON,

    expires_at TIMESTAMP
);
```

---

# 16. Cache Levels

```text
L1 Memory

L2 DuckDB

L3 Filesystem
```

---

# 17. L1

Ultra rápido.

---

```text
Últimos dashboards
```

---

# 18. L2

Persistente.

---

DuckDB.

---

# 19. L3

Snapshots.

---

Parquet.

---

# 20. Query Reuse

Idea poderosa.

---

Dos dashboards.

---

Misma consulta base.

---

No ejecutar dos veces.

---

# 21. Query Graph

```text
Revenue

├─ Country

├─ Product

└─ Segment
```

---

Compartir resultados.

---

# 22. Materialization

Para datasets enormes.

---

Crear.

```text
Summary Tables
```

---

# 23. Example

Tabla original.

```text
500M filas
```

---

Resumen.

```text
Revenue por día
```

---

```text
3650 filas
```

---

# 24. Smart Materialization

Detectar automáticamente.

---

Consultas frecuentes.

↓

Materializar.

---

# 25. Insight Reuse

Si insight existe.

---

Reutilizar.

---

No recalcular.

---

# 26. Investigation Reuse

Muy importante.

---

Root Cause ya calculado.

↓

Reusar.

---

# 27. Genome Reuse

Dashboard Genome.

---

Reutilizar layouts exitosos.

---

# 28. Async Architecture

Nunca bloquear usuario.

---

# 29. Fast Path

```text
SQL
 ↓
Dashboard
```

---

# 30. Slow Path

```text
Insights

Recommendations

Findings
```

---

en background.

---

# 31. Worker Model

```text
API

Workers

Schedulers
```

---

# 32. Job Queue

```sql
CREATE TABLE jobs (

    job_id VARCHAR,

    job_type VARCHAR,

    status VARCHAR
);
```

---

# 33. Priority Model

```text
HIGH

NORMAL

LOW
```

---

# 34. Ejemplo

Dashboard.

```text
HIGH
```

---

Autonomous Analysis.

```text
LOW
```

---

# 35. DuckDB Optimization

Regla.

---

Parquet nativo.

---

Siempre que sea posible.

---

# 36. Evitar

```text
CSV gigantes
```

---

# 37. Preferred Format

```text
Parquet
```

---

# 38. Statistics Registry

Guardar.

```text
Cardinalidad

Nulls

Min

Max
```

---

# 39. Benefit

Evitar scans innecesarios.

---

# 40. Large Dataset Strategy

10M filas.

↓

Normal.

---

100M filas.

↓

Summary tables.

---

1B filas.

↓

Pre-aggregation.

---

# 41. Sampling Engine

Cuando necesario.

---

Crear muestras inteligentes.

---

# 42. Sample Types

```text
Random

Stratified

Temporal
```

---

# 43. Insight Sampling

No siempre necesitamos.

```text
100%
```

de filas.

---

# 44. Approximation

Aceptar.

---

```text
98%
```

precisión.

---

si obtenemos.

```text
20x velocidad
```

---

# 45. Event Throughput

Meta.

---

Miles de eventos.

---

Sin bloquear.

---

# 46. Batch Processing

Agrupar eventos.

---

Procesar juntos.

---

# 47. Learning Batches

No aprender por evento.

---

Aprender por lote.

---

# 48. Recommendation Batches

Igual.

---

# 49. Memory Limits

DuckDB usa memoria local.

---

Controlar.

```text
memory_limit
```

---

# 50. Spill Strategy

Permitir.

```text
Disk Spill
```

---

cuando necesario.

---

# 51. Parallelism

DuckDB ya aprovecha múltiples núcleos.

---

No reinventar.

---

# 52. CPU Budget

Separar.

```text
Query CPU

Analysis CPU

Learning CPU
```

---

# 53. Resource Isolation

Autonomous Analysis nunca debe afectar dashboard interactivo.

---

# 54. Monitoring

Medir.

```text
Latency

Memory

CPU

Cache Hit
```

---

# 55. Metrics Table

```sql
CREATE TABLE runtime_metrics (

    metric_name VARCHAR,

    metric_value DOUBLE,

    created_at TIMESTAMP
);
```

---

# 56. Performance Targets

Dashboard.

```text
< 1 segundo
```

---

Insights.

```text
< 3 segundos
```

---

Findings.

```text
< 10 segundos
```

---

# 57. Scaling Philosophy

Primero.

```text
Optimizar
```

---

Luego.

```text
Escalar
```

---

# 58. No Kubernetes Rule

V1.

---

NO.

```text
Kubernetes
```

---

# 59. No Microservices Rule

V1.

---

NO.

```text
20 servicios
```

---

# 60. Arquitectura Recomendada

```text
FastAPI

DuckDB

Workers

Parquet

Svelte
```

---

# 61. Escalamiento Real

Primero.

```text
1 usuario
```

---

Luego.

```text
10
```

---

Luego.

```text
100
```

---

Nunca diseñar para:

```text
1 millón
```

antes de tiempo.

---

# 62. Strategic Insight

La mayoría de startups fracasan porque construyen:

```text
Escalabilidad imaginaria
```

---

Antes de tener usuarios.

---

# 63. SQLviz Rule

Optimizar.

```text
Trabajo repetido
```

---

No.

```text
Trabajo inexistente
```

---

# 64. El Secreto

El mejor sistema BI no es el que ejecuta consultas más rápido.

---

Es el que evita ejecutar consultas innecesarias.

---

# 65. Principio Fundamental

Performance en SQLviz no significa hacer más rápido el análisis.

Significa recordar lo suficiente para no tener que repetirlo.
