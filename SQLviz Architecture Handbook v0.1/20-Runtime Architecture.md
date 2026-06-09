# SQLviz Architecture Handbook

## Capítulo 20 — Runtime Architecture

Versión: 0.1

---

# 1. Introducción

Hasta ahora hemos diseñado:

```text
Semantic Engine

Intent Engine

Metric Engine

Dimension Engine

Genome Engine

Insight Engine

Recommendation Engine

Autonomous Analysis Engine
```

---

La pregunta ahora es:

```text
¿Cómo interactúan?
```

---

Porque un motor aislado no genera valor.

Un sistema coordinado sí.

---

# 2. Objetivo

Construir una arquitectura capaz de ejecutar:

```text
SQL
 ↓
Análisis
 ↓
Insights
 ↓
Recomendaciones
 ↓
Investigaciones
```

---

en segundos.

---

# 3. Filosofía

SQLviz no debe ser:

```text
Monolito Gigante
```

---

Debe ser:

```text
Conjunto de Motores
```

---

coordinados.

---

# 4. Runtime Conceptual

```text
User SQL
    │
    ▼
Query Pipeline
    │
    ▼
Knowledge Pipeline
    │
    ▼
Insight Pipeline
    │
    ▼
Dashboard Pipeline
```

---

# 5. Core Components

Runtime contiene:

```text
Query Engine

Semantic Engine

Inference Engine

Dashboard Engine

Insight Engine

Recommendation Engine

Learning Engine
```

---

# 6. High Level Architecture

```text
┌──────────────┐
│   Svelte UI  │
└──────┬───────┘
       │
       ▼

┌──────────────┐
│   FastAPI    │
└──────┬───────┘
       │
       ▼

┌──────────────┐
│ Runtime Core │
└──────┬───────┘
       │
       ▼

┌──────────────┐
│   DuckDB     │
└──────────────┘
```

---

# 7. Runtime Core

Responsabilidad.

```text
Orquestar motores
```

---

Nunca renderiza.

Nunca muestra UI.

---

Solo coordina.

---

# 8. Runtime Context

Toda ejecución genera contexto.

---

Modelo.

```python
class RuntimeContext:

    query_id: str

    user_id: str

    dataset_id: str

    metrics: list

    dimensions: list

    intents: list
```

---

# 9. Query Lifecycle

Paso 1.

```text
SQL recibido
```

---

Paso 2.

```text
Parse SQL
```

---

Paso 3.

```text
Semantic Analysis
```

---

Paso 4.

```text
Intent Detection
```

---

Paso 5.

```text
Chart Inference
```

---

Paso 6.

```text
Insight Generation
```

---

Paso 7.

```text
Dashboard Composition
```

---

# 10. Runtime Pipeline

```text
SQL
 │
 ▼
Parser
 │
 ▼
Feature Engine
 │
 ▼
Semantic Engine
 │
 ▼
Intent Engine
 │
 ▼
Metric Engine
 │
 ▼
Dimension Engine
 │
 ▼
Composer
```

---

# 11. Runtime State

DuckDB.

```sql
CREATE TABLE runtime_sessions (

    session_id VARCHAR,

    created_at TIMESTAMP,

    context JSON
);
```

---

# 12. Query State

```sql
CREATE TABLE runtime_queries (

    query_id VARCHAR,

    sql TEXT,

    status VARCHAR,

    created_at TIMESTAMP
);
```

---

# 13. Execution Status

Estados.

```text
PENDING

RUNNING

COMPLETED

FAILED
```

---

# 14. Engine Contract

Todos los motores implementan:

```python
class Engine:

    def execute(
        self,
        context
    ):
        pass
```

---

# 15. Semantic Engine Contract

```python
semantic.execute(
    context
)
```

---

Output.

```python
SemanticResult
```

---

# 16. Intent Engine Contract

Input.

```python
SemanticResult
```

---

Output.

```python
IntentResult
```

---

# 17. Pipeline Principle

Cada motor:

```text
Consume contexto

Produce contexto
```

---

# 18. Immutable Context

Principio importante.

---

Evitar.

```text
Modificar resultados previos
```

---

Crear.

```text
Nuevas capas
```

---

# 19. Runtime Events

Todo genera eventos.

---

Ejemplo.

```text
QUERY_RECEIVED

QUERY_PARSED

INTENT_DETECTED

INSIGHT_CREATED
```

---

# 20. Event Registry

```sql
CREATE TABLE runtime_events (

    event_id VARCHAR,

    event_type VARCHAR,

    payload JSON,

    created_at TIMESTAMP
);
```

---

# 21. Event Bus

V1.

```text
Python Event Dispatcher
```

---

Sin Kafka.

Sin RabbitMQ.

---

Mantener simple.

---

# 22. Cache Layer

Muchos cálculos se repiten.

---

Cachear:

```text
Semantic Results

Intent Results

Insights

Recommendations
```

---

# 23. Cache Registry

```sql
CREATE TABLE cache_entries (

    cache_key VARCHAR,

    payload JSON,

    expires_at TIMESTAMP
);
```

---

# 24. Cache Strategy

Clave.

```python
hash(
    sql
)
```

---

↓

```text
Cache Key
```

---

# 25. Background Jobs

No bloquear UI.

---

Ejemplos.

```text
Insight Discovery

Genome Updates

Learning Updates
```

---

# 26. Job Queue

DuckDB.

```sql
CREATE TABLE jobs (

    job_id VARCHAR,

    job_type VARCHAR,

    status VARCHAR
);
```

---

# 27. Worker Architecture

V1.

```text
FastAPI
+
Background Tasks
```

---

Suficiente.

---

# 28. Runtime Scheduler

Responsabilidad.

```text
Ejecutar jobs pendientes
```

---

# 29. Parallel Execution

Posible.

---

Ejemplo.

```text
Country Analysis

Product Analysis

Customer Analysis
```

---

en paralelo.

---

# 30. Autonomous Runtime

Cuando Insight Engine detecta:

```text
Revenue cayó 20%
```

---

Runtime crea.

```text
Investigation Jobs
```

---

automáticamente.

---

# 31. Investigation Queue

```sql
CREATE TABLE investigations (

    investigation_id VARCHAR,

    status VARCHAR,

    priority DOUBLE
);
```

---

# 32. Runtime Metrics

Medir.

```text
Latency

CPU

Memory

Cache Hits

Jobs Executed
```

---

# 33. Runtime Monitoring

DuckDB.

```sql
CREATE TABLE runtime_metrics (

    metric_name VARCHAR,

    metric_value DOUBLE,

    created_at TIMESTAMP
);
```

---

# 34. Failure Recovery

Si falla:

```text
Insight Engine
```

---

Dashboard sigue funcionando.

---

Principio:

```text
Graceful Degradation
```

---

# 35. Runtime Layers

```text
Presentation Layer

API Layer

Runtime Layer

Storage Layer
```

---

# 36. Runtime Knowledge Flow

```text
SQL
 ↓
Knowledge
 ↓
Insights
 ↓
Recommendations
 ↓
Investigations
```

---

# 37. Runtime Rule

Motores no hablan entre sí.

---

Siempre:

```text
Motor
 ↓
Contexto
 ↓
Motor
```

---

Reduce acoplamiento.

---

# 38. Runtime Explainability

Todo resultado conserva:

```text
Origen

Reglas usadas

Scores

Evidencias
```

---

# 39. Runtime Evolution

V1.

```text
Single Process
```

---

V2.

```text
Multiple Workers
```

---

V3.

```text
Distributed Runtime
```

---

# 40. La Idea Más Importante

La mayoría de herramientas BI tienen:

```text
Frontend
 ↓
Database
```

---

SQLviz tiene:

```text
Frontend
 ↓
Runtime Intelligence Layer
 ↓
Database
```

---

# 41. Principio Fundamental

El Runtime Architecture es el sistema nervioso de SQLviz.

Coordina motores, contexto, eventos, cachés, jobs e investigaciones para convertir una consulta SQL en conocimiento analítico.

Sin él:

```text
Motores aislados.
```

Con él:

```text
Sistema Inteligente.
```