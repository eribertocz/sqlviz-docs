SQLviz Technical Specification v1
Capítulo 01 — Repository Structure

Versión 0.1

1. Objetivo

Diseñar una estructura que soporte:

FastAPI

DuckDB

Svelte

Workers

Engines

Plugins

durante años.

2. Principio

Organizar por dominio.

NO por tecnología.

Malo:

models/

services/

utils/

helpers/

Bueno:

semantic/

intent/

chart/

layout/
3. Monorepo

Recomendado.

sqlviz/
4. Root Structure
sqlviz/

backend/

frontend/

docs/

tests/

scripts/

examples/
5. Backend
backend/

contiene:

API

Engines

DuckDB

Runtime
6. Backend Structure
backend/

src/

tests/
7. Main Source Tree
backend/src/sqlviz/
8. Core Layout
sqlviz/

core/

engines/

runtime/

repositories/

api/

schemas/
9. Core Module

Responsabilidad.

Contratos

Nunca lógica específica.

10. Core Structure
core/

contracts/

types/

exceptions/

constants/
11. Contracts

Contiene.

Engine

Repository

Event

Explanation
12. Engine Folder
engines/

Contiene motores.

13. Engine Tree
engines/

feature/

semantic/

intent/

chart/

layout/

dashboard/
14. Engine Structure

Ejemplo.

semantic/

engine.py

models.py

rules.py

scores.py

tests.py
15. Rule Separation

Muy importante.

NO.

semantic_engine.py

de 5000 líneas.

SÍ.

rules/

metric_rules.py

dimension_rules.py

business_rules.py
16. Runtime Folder
runtime/

Responsabilidad.

Jobs

Workers

Scheduler

Events
17. Runtime Tree
runtime/

events/

jobs/

workers/
18. Repository Layer
repositories/

Responsabilidad.

DuckDB Access
19. Repository Tree
repositories/

dashboard_repo.py

query_repo.py

cache_repo.py
20. Regla

DuckDB nunca se usa directamente desde engines.

Siempre.

Engine
 ↓
Repository
 ↓
DuckDB
21. API Layer
api/

Contiene.

Routers

Dependencies

Middleware
22. API Tree
api/

routers/

middleware/

dependencies/
23. Router Example
routers/

query.py

dashboard.py

explanation.py
24. Schemas
schemas/

Pydantic.

25. Schema Tree
schemas/

query.py

dashboard.py

insight.py
26. Frontend Structure
frontend/
27. Frontend Tree
src/

components/

stores/

routes/

services/
28. Components
components/

dashboard/

query/

insights/
29. Stores
stores/

dashboard.ts

query.ts

runtime.ts
30. Services
services/

api.ts

websocket.ts
31. Documentation
docs/

Contiene.

architecture/

technical/

api/
32. Examples
examples/

Muy importante.

Contiene.

sales/

finance/

marketing/
33. Tests

Separados.

No mezclados.

34. Test Tree
tests/

unit/

integration/

e2e/

benchmark/
35. Benchmark Folder

Contiene.

Inference Corpus
36. Corpus Structure
corpus/

intent/

chart/

layout/
37. SQL Corpus Example
sql: |
  SELECT month,
         SUM(revenue)
  FROM sales
  GROUP BY month

expected:

  chart: line

  intent: trend
38. Scripts
scripts/

Contiene.

seed

benchmark

migration
39. Migrations
scripts/migrations/

DuckDB schema.

40. Principle

Cada carpeta responde:

¿Qué dominio representa?

No:

¿Qué tecnología usa?
41. V1 Engines

Solamente.

Feature

Semantic

Intent

Chart

Layout

Dashboard
42. NO Crear Todavía
Genome

Learning

Recommendation

Autonomous
43. Future Ready

La estructura ya soporta.

Plugins

Workers

Learning

Sin implementarlos.

44. Success Metric

Un desarrollador nuevo debería entender el proyecto en:

15 minutos
45. Principio Fundamental

La estructura del repositorio es la primera API de SQLviz.

Si la estructura es clara, el código será más fácil de mantener.

Si la estructura es confusa, cada nueva funcionalidad costará más que la anterior.