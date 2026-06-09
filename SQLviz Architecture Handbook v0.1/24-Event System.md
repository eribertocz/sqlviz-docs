# SQLviz Architecture Handbook

## Capítulo 24 — Event System

Versión: 0.1

---

# 1. Introducción

Hasta ahora tenemos:

```text
Semantic Engine

Intent Engine

Metric Engine

Dimension Engine

Dashboard Composer

Insight Engine

Recommendation Engine

Autonomous Analysis Engine
```

---

Pero aparece un problema.

---

¿Cómo colaboran?

---

Solución incorrecta:

```python
InsightEngine
    .call(
        RecommendationEngine
    )
```

---

Luego:

```python
RecommendationEngine
    .call(
        LearningEngine
    )
```

---

Luego:

```python
LearningEngine
    .call(
        GenomeEngine
    )
```

---

Resultado.

```text
Acoplamiento
```

---

# 2. Objetivo

Permitir que motores colaboren sin conocerse.

---

# 3. Filosofía

Los motores nunca deberían decir:

```text
Llama a este otro motor
```

---

Deben decir:

```text
Ocurrió algo
```

---

# 4. Concepto

Todo en SQLviz es un evento.

---

Ejemplo.

```text
Query Ejecutada
```

↓

Evento.

---

```text
Insight Generado
```

↓

Evento.

---

```text
Recomendación Creada
```

↓

Evento.

---

# 5. Arquitectura

```text
Engine
   │
   ▼
Publish Event
   │
   ▼
Event Bus
   │
   ▼
Subscribers
```

---

# 6. Beneficio

Motores desacoplados.

---

# 7. Event Model

```python
class Event:

    event_id: str

    event_type: str

    timestamp: datetime

    payload: dict
```

---

# 8. Event Types

Iniciales.

```text
QUERY_RECEIVED

QUERY_EXECUTED

QUERY_FAILED

SEMANTIC_DETECTED

INTENT_DETECTED

CHART_INFERRED

DASHBOARD_CREATED

INSIGHT_CREATED

RECOMMENDATION_CREATED

FINDING_CREATED

GENOME_UPDATED

LEARNING_UPDATED
```

---

# 9. Event Bus

V1.

```python
class EventBus:

    def publish(
        self,
        event
    ):
        pass

    def subscribe(
        self,
        event_type,
        handler
    ):
        pass
```

---

# 10. Principio

Publisher no conoce Subscriber.

---

# 11. Ejemplo

Insight Engine.

```text
Genera Insight
```

↓

Publica.

```text
INSIGHT_CREATED
```

---

No sabe quién lo consume.

---

# 12. Recommendation Engine

Escucha.

```text
INSIGHT_CREATED
```

---

y genera recomendaciones.

---

# 13. Learning Engine

Escucha.

```text
INSIGHT_CREATED
```

---

y actualiza estadísticas.

---

# 14. Event Registry

DuckDB.

```sql
CREATE TABLE events (

    event_id VARCHAR,

    event_type VARCHAR,

    payload JSON,

    created_at TIMESTAMP
);
```

---

# 15. Event Payload

Ejemplo.

```json
{
  "insight_id":"123",
  "type":"growth",
  "metric":"revenue"
}
```

---

# 16. Event Categories

Separar.

```text
System Events

Knowledge Events

Learning Events

Runtime Events
```

---

# 17. System Events

Infraestructura.

```text
QUERY_RECEIVED

QUERY_FAILED

JOB_COMPLETED
```

---

# 18. Knowledge Events

Analítica.

```text
INSIGHT_CREATED

FINDING_CREATED

RECOMMENDATION_CREATED
```

---

# 19. Learning Events

Aprendizaje.

```text
INSIGHT_ACCEPTED

INSIGHT_IGNORED

RECOMMENDATION_ACCEPTED
```

---

# 20. Runtime Events

Operación.

```text
CACHE_HIT

CACHE_MISS

JOB_STARTED
```

---

# 21. Event Dispatcher

Responsabilidad.

```text
Entregar eventos
```

---

# 22. Dispatcher Model

```python
event_type
    ↓
handlers[]
```

---

# 23. Ejemplo

```python
INSIGHT_CREATED
```

↓

```python
[
 recommendation_handler,

 learning_handler,

 genome_handler
]
```

---

# 24. Event Replay

Capacidad importante.

---

Reproducir.

```text
Eventos históricos
```

---

para reconstruir estado.

---

# 25. Event Sourcing Lite

No usar Event Sourcing completo.

---

Demasiado complejo.

---

Pero sí:

```text
Guardar eventos importantes
```

---

# 26. Event Retention

Regla.

---

Guardar:

```text
Insights

Findings

Recommendations
```

---

Mucho tiempo.

---

Eliminar.

```text
CACHE_HIT

CACHE_MISS
```

---

rápidamente.

---

# 27. Event Priority

```text
HIGH

MEDIUM

LOW
```

---

# 28. Event Queue

V1.

---

DuckDB.

```sql
CREATE TABLE event_queue (

    event_id VARCHAR,

    priority INTEGER,

    status VARCHAR
);
```

---

# 29. Async Processing

No bloquear.

---

```text
Usuario
```

↓

```text
Dashboard
```

↓

```text
Evento
```

↓

```text
Procesamiento
```

---

# 30. Event Handlers

Ejemplo.

```python
class InsightHandler:
```

---

```python
class RecommendationHandler:
```

---

```python
class LearningHandler:
```

---

# 31. Handler Rule

Un handler.

```text
Hace una sola cosa
```

---

# 32. Chain Reactions

Ejemplo.

```text
INSIGHT_CREATED
```

↓

```text
RECOMMENDATION_CREATED
```

↓

```text
LEARNING_UPDATED
```

---

# 33. Event Graph

Visualización.

```text
QUERY_EXECUTED
      │
      ▼
INSIGHT_CREATED
      │
      ▼
RECOMMENDATION_CREATED
      │
      ▼
LEARNING_UPDATED
```

---

# 34. Event Metrics

Medir.

```text
Events Published

Events Processed

Processing Time
```

---

# 35. Monitoring Table

```sql
CREATE TABLE event_metrics (

    metric_name VARCHAR,

    metric_value DOUBLE,

    created_at TIMESTAMP
);
```

---

# 36. Failure Handling

Si un handler falla.

---

No detener.

```text
Event Bus
```

---

# 37. Retry Strategy

```text
1

3

5
```

intentos.

---

# 38. Dead Letter Queue

Eventos imposibles.

---

```sql
CREATE TABLE dead_events (

    event_id VARCHAR,

    error TEXT
);
```

---

# 39. Event Explainability

Todo evento conserva.

```text
Origen

Destino

Resultado
```

---

# 40. Event Audit

Pregunta.

```text
¿Por qué existe este insight?
```

---

Respuesta.

```text
Buscar eventos
```

---

# 41. Integración con Runtime

Runtime publica.

```text
QUERY_EXECUTED
```

---

# 42. Integración con Insights

Insight Engine publica.

```text
INSIGHT_CREATED
```

---

# 43. Integración con Learning

Learning Engine escucha.

```text
INSIGHT_ACCEPTED
```

---

# 44. Integración con Genome

Genome escucha.

```text
DASHBOARD_CREATED
```

---

# 45. Integración con Autonomous Analysis

Autonomous Engine escucha.

```text
FINDING_CREATED
```

---

# 46. Evolución

V1.

```text
In-Memory Event Bus
```

---

V2.

```text
DuckDB Event Queue
```

---

V3.

```text
Redis Streams
```

---

V4.

```text
Kafka
```

---

# 47. Regla Estratégica

No empezar con Kafka.

---

No empezar con RabbitMQ.

---

No empezar con NATS.

---

Empezar simple.

---

# 48. Por Qué Importa

Sin eventos.

```text
Motores acoplados
```

---

Con eventos.

```text
Motores cooperan
```

---

# 49. El Gran Cambio

Arquitectura tradicional.

```text
Motor
 ↓
Motor
 ↓
Motor
```

---

SQLviz.

```text
Motor
 ↓
Evento
 ↓
Motor
```

---

# 50. Principio Fundamental

El Event System es el sistema circulatorio de SQLviz.

Transporta conocimiento, hallazgos, aprendizaje y señales entre motores sin crear dependencias directas.

Sin él:

```text
Arquitectura rígida.
```

Con él:

```text
Arquitectura evolutiva.
```