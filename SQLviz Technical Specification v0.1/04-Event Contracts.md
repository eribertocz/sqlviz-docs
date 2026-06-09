SQLviz Technical Specification v1
Capítulo 04 — Event Contracts

Versión 0.1

1. Objetivo

Definir cómo se comunican los componentes de SQLviz.

Sin dependencias directas.

2. Filosofía

Malo.

dashboard_engine.call(
    recommendation_engine
)

Peor.

recommendation_engine.call(
    learning_engine
)

Bueno.

Engine
 ↓
Event
 ↓
Subscriber
3. Principio

Los componentes publican eventos.

Los componentes consumen eventos.

Nunca se llaman entre sí.

4. Beneficios
Menos acoplamiento

Más extensibilidad

Más testing

Más plugins
5. Event Model

Todos los eventos comparten estructura.

6. Base Event
class Event:

    event_id: str

    event_type: str

    execution_id: str

    created_at: datetime

    payload: dict
7. Event ID

Siempre.

uuid4()
8. Event Type

Ejemplo.

QUERY_EXECUTED

CHART_INFERRED

DASHBOARD_CREATED
9. Execution ID

Permite unir eventos.

Ejemplo.

run_123
10. Payload

Contiene datos específicos.

11. Event Categories

V1.

Query Events

Inference Events

Dashboard Events

System Events
12. Query Events

Relacionados con SQL.

13. QUERY_RECEIVED

Disparado cuando llega SQL.

Payload.

{
  "query_id": "...",
  "sql": "..."
}
14. QUERY_PARSED

Parser finalizado.

Payload.

{
  "fingerprint": "..."
}
15. Inference Events

Generados por engines.

16. FEATURES_EXTRACTED

Payload.

{
   "feature_count": 15
}
17. SEMANTICS_INFERRED

Payload.

{
   "tags": [
      "Revenue",
      "Country"
   ]
}
18. INTENTS_INFERRED

Payload.

{
   "intents": [
      "Trend",
      "Comparison"
   ]
}
19. CHARTS_INFERRED

Payload.

{
   "winner": "line",
   "confidence": 0.94
}
20. Layout Events
21. LAYOUT_GENERATED

Payload.

{
   "panel_count": 4
}
22. Dashboard Events
23. DASHBOARD_CREATED

Evento principal.

Payload.

{
   "dashboard_id": "..."
}
24. DASHBOARD_RENDERED

Frontend confirmó render.

Payload.

{
   "dashboard_id": "..."
}
25. Explainability Events
26. EXPLANATION_CREATED

Payload.

{
   "explanation_count": 12
}
27. System Events

Infraestructura.

28. CACHE_HIT

Payload.

{
   "cache_key": "...",
   "cache_type": "semantic"
}
29. CACHE_MISS

Payload.

{
   "cache_key": "..."
}
30. JOB_STARTED

Payload.

{
   "job_id": "...",
   "job_type": "insight"
}
31. JOB_COMPLETED

Payload.

{
   "job_id": "..."
}
32. Event Registry

DuckDB.

CREATE TABLE events (

    event_id VARCHAR,

    event_type VARCHAR,

    execution_id VARCHAR,

    payload JSON,

    created_at TIMESTAMP
);
33. Event Bus

Interfaz única.

34. Contract
class EventBus:

    def publish(
        self,
        event: Event
    ):
        pass

    def subscribe(
        self,
        event_type: str,
        handler
    ):
        pass
35. V1 Implementation

Simple.

Sin Kafka.

Sin RabbitMQ.

36. Recommended
in_memory_bus
duckdb persistence
37. Event Handlers

Ejemplo.

class CacheHandler:

    def handle(
        event: Event
    ):
        ...
38. Multi Subscriber

Un evento puede tener.

1

10

100

suscriptores.

39. Ejemplo
DASHBOARD_CREATED

↓

Cache System

Metrics System

Audit System

todos reciben.

40. Event Ordering

Importante.

Preservar orden.

por execution_id.

41. Idempotency

Regla crítica.

Procesar dos veces.

↓

Mismo resultado.

42. Event Versioning

Añadir.

event_version: int
43. Beneficio

Compatibilidad futura.

44. Event Schema Registry

Definir contratos.

Ejemplo.

EVENT_TYPES = {
   "QUERY_RECEIVED": QueryReceived,
   "INTENTS_INFERRED": IntentInferred
}
45. Strong Typing

Evitar.

payload["x"]

Preferir.

IntentInferredEvent
46. Event Replay

Capacidad de reproducir.

Todos los eventos.

47. Beneficio

Debugging.

Benchmark.

Auditoría.

48. Replay Example
QUERY_RECEIVED

↓

FEATURES_EXTRACTED

↓

SEMANTICS_INFERRED

↓

INTENTS_INFERRED

↓

CHARTS_INFERRED
49. Event Flow V1
QUERY_RECEIVED
 ↓
QUERY_PARSED
 ↓
FEATURES_EXTRACTED
 ↓
SEMANTICS_INFERRED
 ↓
INTENTS_INFERRED
 ↓
CHARTS_INFERRED
 ↓
LAYOUT_GENERATED
 ↓
DASHBOARD_CREATED
50. Event Flow Future
DASHBOARD_CREATED
 ↓
INSIGHT_DISCOVERED
 ↓
RECOMMENDATION_CREATED
 ↓
LEARNING_SIGNAL_CREATED
51. Plugin Compatibility

Los plugins escuchan eventos.

Nunca engines internos.

52. Ejemplo

Plugin.

Slack Plugin

Escucha.

DASHBOARD_CREATED
53. Learning Compatibility

Learning Engine escucha.

USER_ACCEPTED_CHART

sin modificar Chart Engine.

54. Metrics Compatibility

Monitoring escucha.

JOB_COMPLETED

sin modificar Workers.

55. Event Philosophy

Los eventos representan hechos.

No comandos.

Bueno.

DASHBOARD_CREATED

Malo.

CREATE_DASHBOARD
56. Why?

Los hechos ya ocurrieron.

Los comandos generan acoplamiento.

57. Event Naming Rule

Siempre.

PASADO

Ejemplos.

QUERY_RECEIVED

CHART_INFERRED

LAYOUT_GENERATED

DASHBOARD_CREATED
58. Principio Fundamental

Los eventos son el sistema circulatorio de SQLviz.

Los engines generan conocimiento.

Los eventos distribuyen conocimiento.

Gracias a ellos, SQLviz puede crecer desde una aplicación simple hasta una plataforma completa sin convertir cada componente en una dependencia del resto.