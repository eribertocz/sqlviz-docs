# SQLviz Technical Specification v1

## Capítulo 17 — Cache System

Versión 0.1

---

# 1. Objetivo

Evitar trabajo repetido.

---

La optimización más importante de SQLviz no será:

```text
Más CPU
```

---

Ni:

```text
Más RAM
```

---

Será:

```text
No recalcular
```

---

# 2. Filosofía

La operación más rápida es:

```text
La que no ocurre
```

---

# 3. Cache Layers

V1.

```text
L1 Memory Cache

L2 DuckDB Cache

L3 Snapshot Cache
```

---

# 4. L1 Memory Cache

Ultra rápida.

---

Duración.

```text
Proceso actual
```

---

# 5. Uso

```text
AST

Features

Semantics

Intents
```

---

# 6. L2 DuckDB Cache

Persistente.

---

Duración.

```text
Días
```

---

# 7. Uso

```text
Dashboards

Charts

Fingerprints
```

---

# 8. L3 Snapshot Cache

Persistencia completa.

---

Duración.

```text
Semanas/Meses
```

---

# 9. Uso

```text
Dashboard completo

Layout

Explanations
```

---

# 10. Cache Philosophy

No cachear SQL.

---

Cachear conocimiento.

---

# 11. Malo

```python
hash(sql)
```

---

# 12. Mejor

```python
hash(fingerprint)
```

---

# 13. Ejemplo

Consulta A.

```sql
SELECT
 month,
 SUM(revenue)
GROUP BY month
```

---

Consulta B.

```sql
SELECT
 date,
 SUM(profit)
GROUP BY date
```

---

Mismo.

```text
TIME_SERIES_AGGREGATION
```

---

# 14. Resultado

Compartir inferencias.

---

# 15. Cache Key Model

```python
class CacheKey:

    cache_type: str

    fingerprint_hash: str

    version: str
```

---

# 16. Versioning

Muy importante.

---

# 17. Ejemplo

```text
semantic-v1

semantic-v2
```

---

# 18. Why?

Evitar corrupción.

---

# 19. Cache Types

V1.

```text
AST

FEATURES

SEMANTICS

INTENTS

METRICS

DIMENSIONS

CHARTS

LAYOUTS

DASHBOARDS
```

---

# 20. AST Cache

Primer candidato.

---

# 21. Key

```python
sha256(sql)
```

---

# 22. Payload

```python
ast
```

---

# 23. Feature Cache

---

Key.

```python
fingerprint_hash
```

---

Payload.

```python
features
```

---

# 24. Semantic Cache

---

Key.

```python
column_name
+
datatype
```

---

Payload.

```python
semantic_tags
```

---

# 25. Example

```text
revenue
```

↓

```text
Revenue
```

---

Nunca recalcular.

---

# 26. Intent Cache

---

Key.

```python
fingerprint_hash
```

---

Payload.

```python
intents
```

---

# 27. Why?

La intención suele repetirse.

---

# 28. Chart Cache

---

Key.

```python
fingerprint_hash
```

---

Payload.

```python
chart_ranking
```

---

# 29. Layout Cache

---

Key.

```python
dashboard_fingerprint
```

---

Payload.

```python
layout
```

---

# 30. Dashboard Cache

Más valioso.

---

# 31. Key

```python
dashboard_fingerprint
```

---

# 32. Payload

```python
dashboard_snapshot
```

---

# 33. Cache Entry Model

```python
class CacheEntry:

    cache_key: str

    cache_type: str

    payload: dict

    created_at: datetime

    expires_at: datetime
```

---

# 34. Cache Registry

DuckDB.

```sql
CREATE TABLE cache_entries (

    cache_key VARCHAR,

    cache_type VARCHAR,

    payload JSON,

    created_at TIMESTAMP,

    expires_at TIMESTAMP
);
```

---

# 35. TTL Policy

V1.

```text
AST          7 días

Features     30 días

Semantics    90 días

Intents      90 días

Charts       90 días

Layouts      90 días

Dashboards   30 días
```

---

# 36. Cache Flow

```text
Request
 ↓
L1
 ↓
L2
 ↓
L3
 ↓
Compute
```

---

# 37. Cache Hit

Evento.

```text
CACHE_HIT
```

---

# 38. Cache Miss

Evento.

```text
CACHE_MISS
```

---

# 39. Metrics

Medir.

```python
cache_hit_rate
```

---

# 40. Formula

```python
hits
/
(hits + misses)
```

---

# 41. Goal

```text
> 80%
```

---

# 42. Warm Cache

Al iniciar.

---

Precargar.

```text
Top fingerprints

Top semantics

Top dashboards
```

---

# 43. Fingerprint Cache

La joya de SQLviz.

---

# 44. Why?

Porque reutiliza inteligencia.

---

No datos.

---

# 45. Example

```text
Revenue by Month

Profit by Month

Sales by Month
```

↓

```text
Trend Intent

Line Chart

Trend Layout
```

---

Compartidos.

---

# 46. Semantic Cache Table

```sql
CREATE TABLE semantic_cache (

    key VARCHAR,

    semantic_payload JSON
);
```

---

# 47. Fingerprint Cache Table

```sql
CREATE TABLE fingerprint_cache (

    fingerprint_hash VARCHAR,

    payload JSON
);
```

---

# 48. Dashboard Snapshot Cache

```sql
CREATE TABLE dashboard_cache (

    dashboard_hash VARCHAR,

    payload JSON
);
```

---

# 49. Invalidation Strategy

Muy importante.

---

# 50. Invalidate When

```text
Engine Version Changes

Rules Change

Dictionary Changes
```

---

# 51. Cache Version Model

```python
CACHE_VERSION = "1.0"
```

---

# 52. Composite Key

```python
fingerprint
+
version
```

---

# 53. Benchmark Metrics

```text
Hit Rate

Miss Rate

Latency Saved
```

---

# 54. Latency Saved

```python
saved_ms =
compute_time
-
cache_time
```

---

# 55. Strategic Insight

La mayoría de herramientas cachean:

```text
Resultados SQL
```

---

SQLviz debe cachear:

```text
Inferencias
```

---

# 56. Why?

Porque los datos cambian.

---

Pero la intención suele repetirse.

---

# 57. Future V2

```text
Distributed Cache

Redis Layer

Cache Analytics
```

---

# 58. Future V3

```text
Predictive Cache

Genome Cache

Recommendation Cache
```

---

# 59. Performance Goal

Dashboard completo.

```text
< 100 ms
```

con cache hit.

---

# 60. Principio Fundamental

El sistema de cache no existe para acelerar SQL.

Existe para acelerar el conocimiento.

La verdadera ventaja de SQLviz será recordar inferencias previas y reutilizarlas inteligentemente.