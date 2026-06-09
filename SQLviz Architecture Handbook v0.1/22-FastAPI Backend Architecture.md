# SQLviz Architecture Handbook

## Capítulo 22 — FastAPI Backend Architecture

Versión: 0.1

---

# 1. Introducción

Hasta ahora hemos diseñado:

```text
Motores

Conocimiento

Insights

Recomendaciones

Análisis Autónomo

Runtime

DuckDB
```

---

Pero todavía falta responder:

```text
¿Cómo se organiza el código?
```

---

Porque una mala arquitectura termina así:

```text
app.py

25.000 líneas
```

---

Y después de seis meses:

```text
Nadie entiende nada.
```

---

# 2. Objetivo

Construir una arquitectura que permita:

```text
Agregar motores nuevos

Mantener código simple

Escalar gradualmente

Ser mantenida por una sola persona
```

---

# 3. Filosofía

SQLviz NO debe ser:

```text
Microservicios
```

---

Demasiado complejos.

---

SQLviz tampoco debe ser:

```text
Monolito desordenado
```

---

Debe ser:

```text
Monolito Modular
```

---

# 4. Arquitectura General

```text
Svelte
   │
   ▼
FastAPI
   │
   ▼
Runtime
   │
   ▼
DuckDB
```

---

# 5. Estructura Inicial

```text
sqlviz/

├── app/
│
├── api/
│
├── engines/
│
├── services/
│
├── runtime/
│
├── repositories/
│
├── models/
│
├── schemas/
│
├── workers/
│
├── cache/
│
├── database/
│
└── tests/
```

---

# 6. Principio Fundamental

Separar:

```text
Qué sabe hacer el sistema
```

de:

```text
Cómo se expone al usuario
```

---

# 7. Engines

Los engines contienen inteligencia.

---

Ejemplos.

```text
Semantic Engine

Intent Engine

Metric Engine

Dimension Engine

Insight Engine

Recommendation Engine
```

---

# 8. Regla de Engines

Un engine:

```text
NO conoce FastAPI

NO conoce UI

NO conoce HTTP
```

---

Solo conoce:

```python
input
↓
proceso
↓
output
```

---

# 9. Ejemplo

```python
class SemanticEngine:

    def execute(
        self,
        context
    ):
        pass
```

---

# 10. Services

Los services coordinan engines.

---

Ejemplo.

```python
DashboardService
```

---

Responsabilidad.

```text
Coordinar

No inferir
```

---

# 11. Ejemplo

```python
class DashboardService:

    def create_dashboard(
        self,
        sql
    ):
        pass
```

---

Internamente.

```text
Semantic Engine

Intent Engine

Composer
```

---

# 12. Repositories

Acceso a DuckDB.

---

Nunca escribir SQL dentro de engines.

---

# 13. Ejemplo

```python
class InsightRepository:

    def save(
        self,
        insight
    ):
        pass
```

---

# 14. Beneficio

Si cambias:

```text
DuckDB
```

por:

```text
Postgres
```

---

Los engines sobreviven.

---

# 15. Models

Modelos internos.

---

Ejemplo.

```python
class Metric:
    pass
```

---

# 16. Schemas

Pydantic.

---

Ejemplo.

```python
class MetricResponse:
    pass
```

---

# 17. Diferencia

Models:

```text
Dominio interno
```

---

Schemas:

```text
API
```

---

# 18. API Layer

Solo expone endpoints.

---

Nunca contiene lógica.

---

# 19. Ejemplo

```python
@router.post(
    "/dashboard"
)
```

↓

```python
dashboard_service.create()
```

---

# 20. Runtime Layer

Coordina ejecución.

---

Responsable de:

```text
Pipelines

Eventos

Jobs

Contexto
```

---

# 21. Runtime Structure

```text
runtime/

context.py

pipeline.py

dispatcher.py

scheduler.py
```

---

# 22. Pipeline

```python
SQL

↓

Semantic

↓

Intent

↓

Metric

↓

Dimension

↓

Composer
```

---

# 23. Workers

Tareas largas.

---

Ejemplos.

```text
Genome Update

Insight Discovery

Autonomous Analysis
```

---

# 24. Worker Layout

```text
workers/

genome_worker.py

insight_worker.py

analysis_worker.py
```

---

# 25. Cache Layer

```text
cache/

cache_manager.py
```

---

Responsabilidad.

```text
Get

Set

Invalidate
```

---

# 26. Database Layer

```text
database/

connection.py

migrations.py
```

---

# 27. Dependency Injection

FastAPI facilita esto.

---

Ejemplo.

```python
Depends(
    get_dashboard_service
)
```

---

# 28. Engine Registry

Concepto importante.

---

Registrar motores.

```python
ENGINE_REGISTRY = {}
```

---

# 29. Ejemplo

```python
ENGINE_REGISTRY[
   "semantic"
]
=
SemanticEngine()
```

---

# 30. Beneficio

Motores intercambiables.

---

# 31. Plugin Ready

Más adelante.

---

Usuario instala:

```text
sqlviz-plugin-forecast
```

---

SQLviz registra.

```text
Forecast Engine
```

---

automáticamente.

---

# 32. Service Layer

Servicios sugeridos.

```text
Dashboard Service

Insight Service

Recommendation Service

Analysis Service

Learning Service
```

---

# 33. Repository Layer

Repositorios sugeridos.

```text
Dataset Repository

Metric Repository

Insight Repository

Genome Repository

Query Repository
```

---

# 34. API Versioning

Desde V1.

---

```text
/api/v1/
```

---

# 35. REST Endpoints

Principales.

---

```text
POST /query
```

---

```text
POST /dashboard
```

---

```text
GET /insights
```

---

```text
GET /recommendations
```

---

```text
POST /analysis
```

---

# 36. Internal Contracts

Todo engine devuelve.

```python
EngineResult
```

---

Nunca:

```text
dict arbitrarios
```

---

# 37. Engine Result

```python
class EngineResult:

    success: bool

    payload: dict

    metadata: dict
```

---

# 38. Error Strategy

Errores controlados.

---

```python
EngineError
```

---

```python
ValidationError
```

---

```python
InferenceError
```

---

# 39. Logging

Todo engine registra.

---

```text
Input

Output

Duration
```

---

# 40. Observability

Medir.

```text
Execution Time

Cache Hit Rate

Insight Count

Recommendation Count
```

---

# 41. Testing Structure

```text
tests/

unit/

integration/

e2e/
```

---

# 42. Unit Tests

Validan.

```text
Semantic Engine

Metric Engine

Intent Engine
```

---

# 43. Integration Tests

Validan.

```text
Pipeline completo
```

---

# 44. E2E Tests

Validan.

```text
SQL

↓

Dashboard

↓

Insight
```

---

# 45. Monolith First

Regla estratégica.

---

V1:

```text
1 proceso

1 FastAPI

1 DuckDB
```

---

# 46. Escalamiento

Después.

```text
Workers separados
```

---

Después.

```text
Distributed Runtime
```

---

# 47. Error Común

Muchos proyectos empiezan con:

```text
Kafka

Redis

Kubernetes

10 microservicios
```

---

y nunca terminan.

---

# 48. SQLviz V1

Debe ser:

```text
Ridículamente simple
```

---

# 49. Arquitectura Objetivo

```text
Svelte
  │
  ▼

FastAPI
  │
  ▼

Services
  │
  ▼

Engines
  │
  ▼

Repositories
  │
  ▼

DuckDB
```

---

# 50. Principio Fundamental

FastAPI no es el cerebro de SQLviz.

Los Engines son el cerebro.

FastAPI es simplemente la puerta de entrada.

---
