# SQLviz Technical Specification v1

## Capítulo 20 — FastAPI Contracts

Versión 0.1

---

# 1. Objetivo

Definir la API pública completa de SQLviz.

---

La API debe ser:

```text
Simple

Predecible

Versionable

Documentable
```

---

# 2. Filosofía

SQLviz es:

```text
Inference Engine
```

---

No:

```text
CRUD Application
```

---

La API debe reflejar eso.

---

# 3. API Versioning

Siempre.

```text
/api/v1
```

---

# 4. Root Endpoint

```http
GET /
```

---

Response.

```json
{
  "name":"SQLviz",
  "version":"1.0"
}
```

---

# 5. Health Endpoint

```http
GET /health
```

---

# 6. Response

```json
{
  "status":"healthy"
}
```

---

# 7. Generate Dashboard

Endpoint principal.

---

```http
POST /api/v1/dashboard
```

---

# 8. Request Model

```python
class DashboardRequest:

    sql: str

    source: str | None
```

---

# 9. Example Request

```json
{
  "sql":"SELECT month, SUM(revenue) ..."
}
```

---

# 10. Response

```python
class DashboardResponse:

    dashboard_id: str

    dashboard: dict

    confidence: float
```

---

# 11. Response Example

```json
{
  "dashboard_id":"...",
  "confidence":0.95
}
```

---

# 12. Dashboard Retrieval

```http
GET /api/v1/dashboard/{id}
```

---

# 13. Dashboard Explanation

Muy importante.

---

```http
GET /api/v1/dashboard/{id}/explain
```

---

# 14. Response

```json
{
  "confidence":0.95,
  "reasoning":[...]
}
```

---

# 15. Chart Explanation

```http
GET /api/v1/chart/{id}/explain
```

---

# 16. Intent Explanation

```http
GET /api/v1/intent/{id}/explain
```

---

# 17. Metric Explanation

```http
GET /api/v1/metric/{id}/explain
```

---

# 18. Dimension Explanation

```http
GET /api/v1/dimension/{id}/explain
```

---

# 19. Execution Retrieval

```http
GET /api/v1/execution/{id}
```

---

# 20. Response

```json
{
  "execution_id":"...",
  "duration_ms":125
}
```

---

# 21. Execution Timeline

```http
GET /api/v1/execution/{id}/timeline
```

---

# 22. Response

```json
[
  {
    "event":"QUERY_PARSED"
  }
]
```

---

# 23. Events Endpoint

```http
GET /api/v1/events
```

---

# 24. Filtering

```http
GET /api/v1/events?type=...
```

---

# 25. Job Endpoint

```http
GET /api/v1/jobs
```

---

# 26. Job Detail

```http
GET /api/v1/jobs/{id}
```

---

# 27. Runtime Metrics

```http
GET /api/v1/runtime/metrics
```

---

# 28. Response

```json
{
  "cache_hit_rate":0.85
}
```

---

# 29. Cache Stats

```http
GET /api/v1/cache/stats
```

---

# 30. Response

```json
{
  "hits":1000,
  "misses":100
}
```

---

# 31. Cache Invalidation

Admin endpoint.

---

```http
POST /api/v1/cache/invalidate
```

---

# 32. Benchmark Run

```http
POST /api/v1/benchmark/run
```

---

# 33. Benchmark Results

```http
GET /api/v1/benchmark/results
```

---

# 34. Semantic Dictionary

```http
GET /api/v1/semantics/dictionary
```

---

# 35. Semantic Search

```http
GET /api/v1/semantics/search?q=revenue
```

---

# 36. Fingerprint Endpoint

Muy útil.

---

```http
POST /api/v1/fingerprint
```

---

# 37. Response

```json
{
  "fingerprint":"..."
}
```

---

# 38. Dashboard Snapshot

```http
GET /api/v1/dashboard/{id}/snapshot
```

---

# 39. Snapshot Restore

```http
POST /api/v1/dashboard/{id}/restore
```

---

# 40. OpenAPI Rule

Siempre generar.

```text
/swagger
```

---

# 41. Response Envelope

Estándar.

---

```python
class ApiResponse:

    success: bool

    data: dict

    errors: list
```

---

# 42. Error Model

```python
class ApiError:

    code: str

    message: str
```

---

# 43. Validation Errors

```http
422
```

---

# 44. Not Found

```http
404
```

---

# 45. Internal Error

```http
500
```

---

# 46. Async Dashboard Generation

Preparar desde V1.

---

```http
POST /api/v1/dashboard/async
```

---

# 47. Response

```json
{
  "job_id":"..."
}
```

---

# 48. Polling

```http
GET /api/v1/jobs/{id}
```

---

# 49. WebSocket Future

V2.

---

```text
/ws/jobs
```

---

# 50. Authentication

V1 simple.

---

```text
No Auth
```

para open source local.

---

# 51. Future Auth

V2.

---

```text
API Keys

JWT
```

---

# 52. Rate Limiting

V2.

---

No necesario inicialmente.

---

# 53. API Metrics

Medir.

```text
Latency

Errors

Requests
```

---

# 54. Performance Goal

Dashboard endpoint.

```text
P95 < 500 ms
```

---

# 55. Strategic Insight

La API no expone gráficos.

---

Expone:

```text
Inferencias
```

---

# 56. Una mejora crítica

Agregar endpoint.

```http
POST /api/v1/analyze
```

---

Response completa.

```json
{
  "features":[...],
  "semantics":[...],
  "intents":[...],
  "metrics":[...],
  "dimensions":[...],
  "charts":[...]
}
```

---

# 57. Why?

Debugging.

---

Testing.

---

Claude Code.

---

# 58. Otra mejora

```http
POST /api/v1/explain-sql
```

---

Devuelve.

```json
{
  "pipeline":[...]
}
```

---

# 59. Strategic Value

Estos endpoints permitirán construir:

```text
Web UI

CLI

VSCode Extension

Python SDK
```

sobre el mismo núcleo.

---

# 60. Principio Fundamental

FastAPI no es el producto.

Es la puerta de entrada al motor inferencial de SQLviz.