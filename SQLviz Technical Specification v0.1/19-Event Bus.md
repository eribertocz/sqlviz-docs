# SQLviz Technical Specification v1

## Capítulo 19 — Event Bus

Versión 0.1

---

# 1. Objetivo

Desacoplar completamente los motores de SQLviz.

---

Sin Event Bus:

```text
Parser
 ↓
Feature
 ↓
Semantic
 ↓
Intent
```

---

Dependencias directas.

---

Acoplamiento.

---

Difícil de extender.

---

# 2. Filosofía

Los motores NO deben conocerse entre sí.

---

Deben conocer solamente:

```text
RuntimeContext

EventBus
```

---

# 3. Beneficio

Permite agregar.

```text
Plugins

Learning

Benchmarking

Telemetry

Auditoría
```

---

sin modificar motores.

---

# 4. Arquitectura

```text
Engine
 ↓
Publish Event
 ↓
Event Bus
 ↓
Subscribers
```

---

# 5. Modelo Base

```python
class Event:

    event_id: str

    event_type: str

    execution_id: str

    payload: dict

    timestamp: datetime
```

---

# 6. Event Types

V1.

```text
QUERY_PARSED

FEATURES_EXTRACTED

SEMANTICS_INFERRED

INTENTS_INFERRED

METRICS_INFERRED

DIMENSIONS_INFERRED

CHARTS_INFERRED

LAYOUT_GENERATED

DASHBOARD_CREATED
```

---

# 7. Runtime Events

```text
JOB_CREATED

JOB_STARTED

JOB_COMPLETED

JOB_FAILED
```

---

# 8. Cache Events

```text
CACHE_HIT

CACHE_MISS

CACHE_REFRESHED
```

---

# 9. Explanation Events

```text
EXPLANATION_CREATED

TRACE_CREATED
```

---

# 10. Event Bus Contract

```python
class EventBus:

    def publish(...)

    def subscribe(...)
```

---

# 11. Publish

```python
event_bus.publish(
    Event(...)
)
```

---

# 12. Subscribe

```python
event_bus.subscribe(
    event_type,
    handler
)
```

---

# 13. Example

```python
event_bus.subscribe(
    "DASHBOARD_CREATED",
    analytics_handler
)
```

---

# 14. Event Handler

```python
class EventHandler:

    def handle(
        self,
        event: Event
    ):
        ...
```

---

# 15. Synchronous Events

V1.

---

Todos.

```text
In Process
```

---

# 16. Why?

Simpleza.

---

Proyecto sostenible.

---

# 17. No Kafka Rule

V1.

---

Evitar.

```text
Kafka

RabbitMQ

NATS

Pulsar
```

---

# 18. Why?

No aportan valor inicial.

---

# 19. Local Event Bus

Implementación.

```python
class LocalEventBus:
```

---

suficiente para V1.

---

# 20. Event Registry

DuckDB.

```sql
CREATE TABLE events (

    event_id VARCHAR,

    event_type VARCHAR,

    execution_id VARCHAR,

    payload JSON,

    created_at TIMESTAMP
);
```

---

# 21. Why?

Auditoría.

---

Debugging.

---

Replay.

---

# 22. Event Replay

Idea poderosa.

---

```text
Execution
 ↓
Events
 ↓
Replay
```

---

# 23. Benefit

Reproducir inferencias.

---

# 24. Event Trace

```python
class EventTrace:

    execution_id

    events
```

---

# 25. Trace Example

```text
QUERY_PARSED

FEATURES_EXTRACTED

SEMANTICS_INFERRED

INTENTS_INFERRED

CHARTS_INFERRED

DASHBOARD_CREATED
```

---

# 26. Telemetry Subscriber

Escucha.

```text
Todos los eventos
```

---

y genera métricas.

---

# 27. Learning Subscriber

Futuro.

---

Escucha.

```text
DASHBOARD_ACCEPTED

DASHBOARD_MODIFIED
```

---

# 28. Benchmark Subscriber

Escucha.

```text
DASHBOARD_CREATED
```

---

para ejecutar validaciones.

---

# 29. Cache Subscriber

Escucha.

```text
FEATURES_EXTRACTED
```

---

y actualiza cache.

---

# 30. Analytics Subscriber

Escucha.

```text
Todos
```

---

para reporting interno.

---

# 31. Event Versioning

Siempre.

```python
event.version
```

---

# 32. Why?

Compatibilidad futura.

---

# 33. Event Naming Rule

Siempre.

```text
PAST_TENSE
```

---

Ejemplos.

```text
QUERY_PARSED

CHARTS_INFERRED

DASHBOARD_CREATED
```

---

# 34. Nunca

```text
PARSE_QUERY

CREATE_DASHBOARD
```

---

porque son comandos.

---

No eventos.

---

# 35. Event Metadata

```python
class EventMetadata:

    source_engine

    execution_id

    version
```

---

# 36. Event Store

DuckDB.

---

Retención.

```text
90 días
```

---

# 37. Cleanup Job

Eliminar eventos antiguos.

---

Automático.

---

# 38. Event Filtering

Permitir.

```python
get_events(
    event_type=...
)
```

---

# 39. Dashboard Timeline

Future UI.

---

```text
Execution Timeline
```

---

# 40. Example

```text
10:00:01 QUERY_PARSED

10:00:01 FEATURES_EXTRACTED

10:00:02 CHARTS_INFERRED

10:00:02 DASHBOARD_CREATED
```

---

# 41. Explainability Integration

Muy importante.

---

Cada explicación referencia.

```text
Event IDs
```

---

# 42. Why?

Auditabilidad.

---

# 43. Performance Goal

Publish.

```text
< 1 ms
```

---

# 44. Subscriber Failures

Nunca romper pipeline.

---

# 45. Rule

```python
try:
   handler(...)
except:
   log(...)
```

---

Continuar.

---

# 46. Event Metrics

Medir.

```text
Events/sec

Subscriber Latency

Handler Failures
```

---

# 47. Runtime Dashboard

Future.

---

Mostrar.

```text
Jobs

Events

Latency

Cache
```

---

# 48. Strategic Insight

Los Engines generan conocimiento.

---

El Event Bus distribuye conocimiento.

---

# 49. Future V2

```text
Async Events

Distributed Events

Remote Subscribers
```

---

# 50. Future V3

```text
Event Streaming

Real Time Learning

Autonomous Agents
```

---

# 51. Una mejora importante

Agregar.

```python
event.correlation_id
```

---

para seguir toda una ejecución.

---

# 52. Otra mejora

Agregar.

```python
event.parent_event_id
```

---

para construir árboles.

---

# 53. Event Graph

Future.

---

```text
QUERY_PARSED
   ↓
FEATURES_EXTRACTED
   ↓
INTENTS_INFERRED
```

---

# 54. Strategic Value

Este componente es el puente hacia:

```text
Learning Engine

Plugins

Analytics

Telemetry
```

---

# 55. Principio Fundamental

Los motores producen eventos.

Los eventos producen extensibilidad.

La extensibilidad produce longevidad.