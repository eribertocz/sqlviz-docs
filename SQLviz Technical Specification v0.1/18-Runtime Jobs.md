# SQLviz Technical Specification v1

## Capítulo 18 — Runtime Jobs

Versión 0.1

---

# 1. Objetivo

Definir cómo SQLviz ejecuta trabajo en segundo plano.

---

Hasta ahora definimos:

```text
Motores

Contratos

Persistencia

Cache
```

---

Ahora debemos definir:

```text
Orquestación
```

---

# 2. Filosofía

El usuario nunca debería esperar por trabajo innecesario.

---

Regla:

```text
Interactive First
```

---

# 3. Clasificación

Todos los trabajos pertenecen a una categoría.

---

# 4. Interactive Jobs

Ejecutados inmediatamente.

---

Objetivo.

```text
< 300 ms
```

---

# 5. Ejemplos

```text
Dashboard Generation

Chart Inference

Layout Inference

Explainability
```

---

# 6. Background Jobs

No bloquean UI.

---

# 7. Ejemplos

```text
Benchmarking

Snapshot Cleanup

Cache Cleanup

Metrics Aggregation
```

---

# 8. Scheduled Jobs

Ejecución periódica.

---

# 9. Ejemplos

```text
Nightly Benchmarks

Dictionary Refresh

Cache Vacuum
```

---

# 10. Runtime Job Model

```python
class RuntimeJob:

    job_id: str

    job_type: str

    status: str

    payload: dict

    created_at: datetime
```

---

# 11. Job States

```text
PENDING

RUNNING

SUCCESS

FAILED

CANCELLED
```

---

# 12. Job Registry

DuckDB.

```sql
CREATE TABLE jobs (

    job_id VARCHAR,

    job_type VARCHAR,

    status VARCHAR,

    payload JSON,

    created_at TIMESTAMP,

    finished_at TIMESTAMP
);
```

---

# 13. Dashboard Job

Trabajo principal.

---

# 14. Flow

```text
SQL
 ↓
Create Job
 ↓
Run Pipeline
 ↓
Store Dashboard
```

---

# 15. Job Type

```text
GENERATE_DASHBOARD
```

---

# 16. Cache Refresh Job

Actualiza inferencias populares.

---

# 17. Job Type

```text
REFRESH_CACHE
```

---

# 18. Benchmark Job

Muy importante.

---

# 19. Job Type

```text
RUN_BENCHMARKS
```

---

# 20. Why?

Medir regresiones.

---

# 21. Benchmark Flow

```text
Corpus
 ↓
Run Pipeline
 ↓
Compare Results
 ↓
Store Metrics
```

---

# 22. Snapshot Job

Guardar dashboards importantes.

---

# 23. Job Type

```text
CREATE_SNAPSHOT
```

---

# 24. Cleanup Job

Eliminar basura.

---

# 25. Job Types

```text
CACHE_CLEANUP

SNAPSHOT_CLEANUP

EVENT_CLEANUP
```

---

# 26. Job Runner

V1 simple.

---

```python
class JobRunner:

    def submit(...)

    def execute(...)
```

---

# 27. Queue Model

```python
class JobQueue:

    pending

    running

    completed
```

---

# 28. Prioridades

V1.

```text
HIGH

NORMAL

LOW
```

---

# 29. HIGH

```text
Dashboard Generation
```

---

# 30. NORMAL

```text
Cache Refresh
```

---

# 31. LOW

```text
Cleanup
```

---

# 32. Retry Policy

Automática.

---

# 33. Example

```python
max_retries = 3
```

---

# 34. Failure Registry

```sql
CREATE TABLE failed_jobs (

    job_id VARCHAR,

    error_message VARCHAR,

    retry_count INTEGER
);
```

---

# 35. Job Metrics

Medir.

```text
Execution Time

Success Rate

Failure Rate
```

---

# 36. Runtime Metrics Table

```sql
CREATE TABLE runtime_job_metrics (

    job_type VARCHAR,

    duration_ms DOUBLE,

    success BOOLEAN
);
```

---

# 37. Dashboard SLA

Objetivo.

```text
< 500 ms
```

---

para dashboard completo.

---

# 38. Async Strategy

FastAPI.

---

```python
BackgroundTasks
```

---

Inicialmente suficiente.

---

# 39. No Celery Rule

V1.

---

Evitar.

```text
Celery

RabbitMQ

Kafka
```

---

# 40. Why?

Proyecto sostenible para una persona.

---

# 41. Scheduler

V1.

---

```python
APScheduler
```

---

suficiente.

---

# 42. Scheduled Tasks

```text
Nightly Benchmarks

Weekly Cleanup

Cache Vacuum
```

---

# 43. Vacuum Job

DuckDB.

---

```sql
VACUUM;
```

---

programado.

---

# 44. Job Events

Publicar.

```text
JOB_CREATED

JOB_STARTED

JOB_COMPLETED

JOB_FAILED
```

---

# 45. Event Payload

```python
{
   "job_id":"...",
   "job_type":"..."
}
```

---

# 46. Explainability

Incluso los jobs.

---

# 47. Example

```text
Cache Refresh

Reason:

Hot fingerprint detected
```

---

# 48. Dashboard Warmup

Nueva idea.

---

Precalcular.

```text
Top Fingerprints
```

---

al iniciar.

---

# 49. Why?

Menor latencia.

---

# 50. Resource Limits

V1.

```python
MAX_CONCURRENT_JOBS = 4
```

---

# 51. Backpressure

Si cola llena.

---

```text
Reject Low Priority Jobs
```

---

# 52. Observability

Todo job registra.

```text
Start

Finish

Duration

Status
```

---

# 53. Monitoring View

Future UI.

---

```text
Jobs Dashboard
```

---

# 54. Strategic Insight

La mayoría de herramientas BI tienen:

```text
Query Engine
```

---

SQLviz debe tener:

```text
Inference Runtime
```

---

# 55. Future V2

```text
Distributed Jobs

Remote Workers

Cluster Execution
```

---

# 56. Future V3

```text
Autonomous Jobs

Self-Healing Jobs

Learning Jobs
```

---

# 57. Una mejora crítica

Agregar.

```python
class JobResult:

    execution_id

    dashboard_id

    metrics
```

---

Porque facilitará:

```text
Tracing

Debugging

Learning
```

---

# 58. Performance Goal

95% de jobs interactivos.

```text
< 500 ms
```

---

# 59. Strategic Goal

Usuario percibe.

```text
Instant Dashboard Generation
```

---

aunque internamente existan múltiples motores.

---

# 60. Principio Fundamental

Los Runtime Jobs son el sistema circulatorio de SQLviz.

Los Engines producen inteligencia.

Los Jobs permiten que esa inteligencia se ejecute de forma eficiente, observable y escalable.