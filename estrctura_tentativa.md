La mejor analogía es:

# SQLviz como Sistema Operativo Analítico

Un sistema operativo no es “una app”. Es una plataforma que coordina recursos, procesos, memoria, drivers, seguridad, archivos, extensiones y UI.

SQLviz debería pensarse igual:

```text
SQLviz no es un dashboard builder.

SQLviz es un sistema operativo analítico donde:

SQL = lenguaje de usuario
Inference Engine = CPU analítica
Runtime = scheduler
DuckDB = filesystem/memoria local
Event Bus = sistema de señales
Plugins = drivers/extensiones
Frontend = desktop environment
Benchmarks = suite de diagnóstico
```

---

# 1. Mapa de analogía

```text
Sistema Operativo Tradicional       SQLviz

Kernel                              sqlviz-core
Process Scheduler                   sqlviz-runtime
Drivers                             sqlviz-sources
CPU / Execution Engine              sqlviz-inference
Filesystem                          sqlviz-storage
System Logs                         sqlviz-events
Memory Cache                        sqlviz-cache
Package Manager                     sqlviz-plugins
Shell / Terminal                    sqlviz-cli
Desktop Environment                 sqlviz-web
System Settings                     sqlviz-config
Diagnostics                         sqlviz-benchmarks
Security Layer                      sqlviz-security
Developer SDK                       sqlviz-sdk
```

---

# 2. Estructura grande recomendada

Para un proyecto grande, yo usaría monorepo por paquetes:

```text
sqlviz/

├── packages/
│   ├── sqlviz-core/
│   ├── sqlviz-parser/
│   ├── sqlviz-inference/
│   ├── sqlviz-viz/
│   ├── sqlviz-runtime/
│   ├── sqlviz-storage/
│   ├── sqlviz-events/
│   ├── sqlviz-cache/
│   ├── sqlviz-sources/
│   ├── sqlviz-api/
│   ├── sqlviz-web/
│   ├── sqlviz-cli/
│   ├── sqlviz-plugins/
│   ├── sqlviz-benchmarks/
│   ├── sqlviz-security/
│   └── sqlviz-sdk/
│
├── docs/
├── examples/
├── datasets/
├── scripts/
├── docker/
├── .github/
├── pyproject.toml
├── README.md
└── LICENSE
```

Esto es mucho más serio que:

```text
backend/
frontend/
utils/
```

Porque SQLviz no será solo una app. Será una plataforma.

---

# 3. Kernel del sistema: `sqlviz-core`

En Linux, el kernel define procesos, memoria, syscalls, drivers, permisos.

En SQLviz, `sqlviz-core` define los contratos universales.

```text
packages/sqlviz-core/

├── pyproject.toml
├── README.md
└── src/sqlviz_core/
    ├── __init__.py
    ├── context.py
    ├── engine.py
    ├── result.py
    ├── models/
    │   ├── query.py
    │   ├── feature.py
    │   ├── semantic.py
    │   ├── intent.py
    │   ├── metric.py
    │   ├── dimension.py
    │   ├── chart.py
    │   ├── layout.py
    │   ├── dashboard.py
    │   └── explanation.py
    ├── contracts/
    │   ├── engine_contract.py
    │   ├── repository_contract.py
    │   ├── event_contract.py
    │   └── plugin_contract.py
    ├── enums/
    │   ├── chart_type.py
    │   ├── intent_type.py
    │   ├── semantic_type.py
    │   └── confidence_level.py
    ├── errors/
    │   ├── base.py
    │   ├── inference_error.py
    │   ├── parser_error.py
    │   └── validation_error.py
    └── config/
        ├── settings.py
        └── defaults.py
```

## Qué contiene

### `context.py`

El `RuntimeContext` es como el “process control block” del sistema operativo.

Transporta todo:

```python
class RuntimeContext:
    execution_id: str
    sql: str
    ast: object | None
    fingerprint: str | None
    features: list
    semantic_tags: list
    intents: list
    metrics: list
    dimensions: list
    chart_candidates: list
    layout: object | None
    dashboard: object | None
    explanations: list
    warnings: list
    errors: list
```

### `engine.py`

Contrato universal:

```python
class Engine:
    name: str
    version: str

    def execute(self, context: RuntimeContext) -> RuntimeContext:
        raise NotImplementedError
```

Regla del kernel:

```text
Ningún engine llama directamente a otro engine.
Todos reciben y devuelven RuntimeContext.
```

### `models/`

Aquí viven los objetos base. No dependen de FastAPI, Svelte, DuckDB ni plugins.

Ejemplo:

```text
Metric
Dimension
Intent
ChartCandidate
Dashboard
Explanation
Evidence
```

Este paquete debe ser muy estable. Si `sqlviz-core` cambia mucho, todo el ecosistema sufre.

---

# 4. Parser: `sqlviz-parser`

En un sistema operativo, antes de ejecutar algo, el kernel debe entender instrucciones.

En SQLviz, el parser convierte SQL crudo en estructura.

```text
packages/sqlviz-parser/

├── pyproject.toml
└── src/sqlviz_parser/
    ├── __init__.py
    ├── engine.py
    ├── ast/
    │   ├── normalizer.py
    │   ├── nodes.py
    │   └── extractor.py
    ├── sqlglot_adapter.py
    ├── fingerprint/
    │   ├── generator.py
    │   ├── hasher.py
    │   └── patterns.py
    ├── extractors/
    │   ├── select_extractor.py
    │   ├── from_extractor.py
    │   ├── where_extractor.py
    │   ├── groupby_extractor.py
    │   ├── orderby_extractor.py
    │   ├── join_extractor.py
    │   └── window_extractor.py
    └── tests/
```

## Tecnología

```text
Python
SQLGlot
Pydantic/dataclasses
```

## Qué hace

```text
SQL crudo
↓
SQLGlot AST
↓
AST normalizado propio
↓
metadatos SQL
↓
fingerprint
```

Ejemplo:

```sql
SELECT month, SUM(revenue)
FROM sales
GROUP BY month
ORDER BY month
```

Produce:

```text
has_select = true
has_group_by = true
has_aggregation = true
aggregation = SUM
group_by = month
fingerprint = TIME_SERIES_AGGREGATION
```

---

# 5. CPU analítica: `sqlviz-inference`

Este es el corazón. Si SQLviz fuera Linux, esto sería la CPU + subsistema de ejecución analítica.

```text
packages/sqlviz-inference/

├── pyproject.toml
└── src/sqlviz_inference/
    ├── __init__.py
    ├── feature/
    │   ├── engine.py
    │   ├── rules.py
    │   ├── detectors/
    │   │   ├── aggregation_detector.py
    │   │   ├── temporal_detector.py
    │   │   ├── ranking_detector.py
    │   │   ├── filter_detector.py
    │   │   └── cardinality_detector.py
    │   └── scoring.py
    ├── semantic/
    │   ├── engine.py
    │   ├── dictionary.py
    │   ├── matcher.py
    │   ├── normalizer.py
    │   ├── ontology.py
    │   └── scoring.py
    ├── intent/
    │   ├── engine.py
    │   ├── rules/
    │   │   ├── trend.py
    │   │   ├── comparison.py
    │   │   ├── ranking.py
    │   │   ├── kpi.py
    │   │   ├── distribution.py
    │   │   └── correlation.py
    │   └── scorer.py
    ├── metric/
    │   ├── engine.py
    │   ├── detector.py
    │   ├── classifier.py
    │   └── scorer.py
    ├── dimension/
    │   ├── engine.py
    │   ├── detector.py
    │   ├── hierarchy.py
    │   └── scorer.py
    └── rulesets/
        ├── default_intent_rules.yaml
        ├── default_semantic_dictionary.yaml
        ├── default_metric_rules.yaml
        └── default_dimension_rules.yaml
```

## Submódulos

### `feature/`

Equivalente a sensores internos de CPU.

Detecta señales:

```text
HAS_GROUP_BY
HAS_AGGREGATION
HAS_DATE_COLUMN
HAS_LIMIT
HAS_ORDER_BY_DESC
HAS_WINDOW_FUNCTION
HAS_TOP_N
LOW_CARDINALITY
HIGH_CARDINALITY_IDENTIFIER
```

### `semantic/`

Convierte texto en significado:

```text
revenue → Revenue / Financial Metric
country → Geography / Dimension
customer_id → Identifier / Customer Entity
month → Temporal Dimension
```

### `intent/`

Interpreta objetivo analítico:

```text
Trend
Comparison
Ranking
KPI
Distribution
Correlation
Composition
```

Ejemplo:

```python
trend_score =
    has_temporal_dimension * 0.40 +
    has_group_by * 0.25 +
    has_aggregation * 0.25 +
    has_temporal_order * 0.10
```

### `metric/`

Detecta qué se mide:

```text
Revenue
Profit
Cost
Quantity
Margin
Orders
```

### `dimension/`

Detecta desde qué perspectiva se mide:

```text
Date
Country
Product
Customer
Segment
Channel
```

Regla importante:

```text
Metric Engine = qué medir
Dimension Engine = cómo segmentarlo
Intent Engine = por qué se analiza
```

---

# 6. Sistema visual: `sqlviz-viz`

Esto no es frontend. Es el motor que decide cómo representar.

```text
packages/sqlviz-viz/

├── pyproject.toml
└── src/sqlviz_viz/
    ├── __init__.py
    ├── chart/
    │   ├── engine.py
    │   ├── registry.py
    │   ├── candidates.py
    │   ├── scorers/
    │   │   ├── kpi_scorer.py
    │   │   ├── line_scorer.py
    │   │   ├── bar_scorer.py
    │   │   ├── pie_scorer.py
    │   │   ├── scatter_scorer.py
    │   │   └── table_scorer.py
    │   └── rules.yaml
    ├── layout/
    │   ├── engine.py
    │   ├── grid.py
    │   ├── placement.py
    │   ├── priority.py
    │   ├── templates/
    │   │   ├── executive.py
    │   │   ├── analytical.py
    │   │   └── operational.py
    │   └── scoring.py
    └── composer/
        ├── engine.py
        ├── dashboard_builder.py
        ├── panel_builder.py
        ├── title_generator.py
        └── section_builder.py
```

## Chart Engine

Recibe:

```text
Intent
Metric
Dimension
Features
Cardinality
```

Devuelve ranking:

```text
Line 0.96
Bar 0.72
Table 0.40
```

No devuelve solo un chart. Devuelve candidatos.

## Layout Engine

Decide:

```text
qué panel va arriba
qué panel es grande
qué panel es secundario
qué paneles se agrupan
```

## Composer

Construye el objeto final:

```text
Dashboard
├── Sections
├── Panels
├── Charts
├── Layout
└── Explanations
```

---

# 7. Scheduler / Kernel Runtime: `sqlviz-runtime`

En Linux, el scheduler decide qué proceso corre.

En SQLviz, el runtime decide qué engine corre.

```text
packages/sqlviz-runtime/

├── pyproject.toml
└── src/sqlviz_runtime/
    ├── __init__.py
    ├── pipeline/
    │   ├── runner.py
    │   ├── pipeline.py
    │   ├── stage.py
    │   └── registry.py
    ├── execution/
    │   ├── execution.py
    │   ├── lifecycle.py
    │   ├── status.py
    │   └── tracker.py
    ├── jobs/
    │   ├── job.py
    │   ├── queue.py
    │   ├── runner.py
    │   └── scheduler.py
    └── orchestration/
        ├── dependency_graph.py
        ├── fallback.py
        └── recovery.py
```

## Pipeline

```python
pipeline = [
    ParserEngine(),
    FeatureEngine(),
    SemanticEngine(),
    IntentEngine(),
    MetricEngine(),
    DimensionEngine(),
    ChartEngine(),
    LayoutEngine(),
    DashboardComposer(),
]
```

## Fallbacks

Si algo falla:

```text
Semantic Engine falla → usar nombres de columnas
Chart Engine baja confianza → usar Table
Layout Engine falla → usar layout básico
```

Esto es importante. El sistema no debe romperse por un motor débil.

---

# 8. Sistema eléctrico: `sqlviz-events`

En un OS, las señales permiten comunicar procesos.

En SQLviz, eventos desacoplan todo.

```text
packages/sqlviz-events/

├── pyproject.toml
└── src/sqlviz_events/
    ├── __init__.py
    ├── event.py
    ├── bus.py
    ├── registry.py
    ├── dispatcher.py
    ├── subscribers/
    │   ├── logging_subscriber.py
    │   ├── metrics_subscriber.py
    │   ├── cache_subscriber.py
    │   └── audit_subscriber.py
    └── schemas/
        ├── query_events.py
        ├── inference_events.py
        ├── dashboard_events.py
        └── runtime_events.py
```

Eventos:

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

Esto permite que en el futuro un plugin haga:

```python
subscribe("DASHBOARD_CREATED", export_to_pdf)
```

sin tocar el cerebro.

---

# 9. Memoria y filesystem: `sqlviz-storage`

En un OS, el filesystem guarda archivos, logs y configuración.

En SQLviz, storage guarda metadatos analíticos.

```text
packages/sqlviz-storage/

├── pyproject.toml
└── src/sqlviz_storage/
    ├── __init__.py
    ├── duckdb/
    │   ├── connection.py
    │   ├── migrations.py
    │   ├── schema.sql
    │   └── pragmas.py
    ├── repositories/
    │   ├── execution_repository.py
    │   ├── event_repository.py
    │   ├── cache_repository.py
    │   ├── dashboard_repository.py
    │   ├── explanation_repository.py
    │   ├── benchmark_repository.py
    │   └── dictionary_repository.py
    └── migrations/
        ├── 001_initial_schema.sql
        ├── 002_events.sql
        ├── 003_cache.sql
        └── 004_benchmarks.sql
```

## DuckDB guarda

```text
executions
events
features
semantic_tags
intents
metrics
dimensions
chart_predictions
layouts
dashboards
explanations
cache_entries
benchmark_results
```

Regla:

```text
DuckDB almacena memoria analítica.
No necesariamente almacena todos los datos del usuario.
```

Los datos grandes pueden vivir en Parquet.

---

# 10. Cache memory: `sqlviz-cache`

En un OS, hay memoria RAM, page cache, disk cache.

SQLviz necesita cache de inferencia.

```text
packages/sqlviz-cache/

├── pyproject.toml
└── src/sqlviz_cache/
    ├── __init__.py
    ├── memory_cache.py
    ├── duckdb_cache.py
    ├── snapshot_cache.py
    ├── keys.py
    ├── policies.py
    └── invalidation.py
```

Tipos:

```text
AST_CACHE
FEATURE_CACHE
SEMANTIC_CACHE
INTENT_CACHE
CHART_CACHE
LAYOUT_CACHE
DASHBOARD_CACHE
```

Clave estratégica:

```text
No cachear solo SQL.
Cachear fingerprint e inferencias.
```

---

# 11. Drivers: `sqlviz-sources`

En un sistema operativo, los drivers permiten hablar con hardware.

SQLviz necesita drivers de datos.

```text
packages/sqlviz-sources/

├── pyproject.toml
└── src/sqlviz_sources/
    ├── __init__.py
    ├── base.py
    ├── duckdb_source.py
    ├── parquet_source.py
    ├── csv_source.py
    ├── motherduck_source.py
    ├── postgres_source.py
    └── profiler/
        ├── schema_profiler.py
        ├── column_profiler.py
        └── sample_profiler.py
```

Regla:

```text
Los engines nunca deberían saber de dónde vienen los datos.
```

Solo consumen:

```text
SchemaProfile
ColumnProfile
DataSample
```

---

# 12. API / syscalls: `sqlviz-api`

En un OS, las syscalls permiten que programas hablen con el kernel.

En SQLviz, FastAPI expone el kernel analítico.

```text
packages/sqlviz-api/

├── pyproject.toml
└── src/sqlviz_api/
    ├── __init__.py
    ├── main.py
    ├── dependencies.py
    ├── routers/
    │   ├── dashboard.py
    │   ├── inference.py
    │   ├── execution.py
    │   ├── explanation.py
    │   ├── benchmark.py
    │   ├── cache.py
    │   └── health.py
    ├── schemas/
    │   ├── dashboard_request.py
    │   ├── dashboard_response.py
    │   ├── inference_response.py
    │   ├── explanation_response.py
    │   └── error_response.py
    └── middleware/
        ├── error_handler.py
        ├── timing.py
        └── cors.py
```

Endpoints clave:

```text
POST /api/v1/infer
POST /api/v1/dashboard
GET  /api/v1/executions/{id}
GET  /api/v1/explanations/{id}
GET  /api/v1/benchmark/results
```

---

# 13. Desktop environment: `sqlviz-web`

El frontend es el escritorio.

```text
packages/sqlviz-web/

├── package.json
├── svelte.config.js
├── vite.config.ts
└── src/
    ├── routes/
    │   ├── +layout.svelte
    │   ├── +page.svelte
    │   ├── dashboard/
    │   ├── runtime/
    │   └── benchmarks/
    ├── lib/
    │   ├── components/
    │   │   ├── sql-editor/
    │   │   ├── dashboard/
    │   │   ├── panels/
    │   │   ├── charts/
    │   │   ├── explainability/
    │   │   └── timeline/
    │   ├── stores/
    │   │   ├── dashboardStore.ts
    │   │   ├── executionStore.ts
    │   │   ├── explanationStore.ts
    │   │   └── runtimeStore.ts
    │   ├── api/
    │   │   ├── client.ts
    │   │   ├── dashboard.ts
    │   │   └── inference.ts
    │   └── types/
    │       ├── dashboard.ts
    │       ├── chart.ts
    │       ├── layout.ts
    │       └── explanation.ts
```

Regla:

```text
Frontend no infiere.
Frontend renderiza.
```

---

# 14. Shell: `sqlviz-cli`

Como Bash o PowerShell.

```text
packages/sqlviz-cli/

├── pyproject.toml
└── src/sqlviz_cli/
    ├── __init__.py
    ├── main.py
    ├── commands/
    │   ├── infer.py
    │   ├── dashboard.py
    │   ├── benchmark.py
    │   ├── cache.py
    │   └── init.py
    └── output/
        ├── json.py
        ├── markdown.py
        └── table.py
```

Ejemplo:

```bash
sqlviz infer "SELECT month, SUM(revenue) FROM sales GROUP BY month"
```

Devuelve:

```json
{
  "intent": "trend",
  "chart": "line",
  "confidence": 0.94
}
```

---

# 15. Package manager / Extensions: `sqlviz-plugins`

Este es el punto clave para que plugins no afecten el cerebro.

```text
packages/sqlviz-plugins/

├── pyproject.toml
└── src/sqlviz_plugins/
    ├── __init__.py
    ├── sdk.py
    ├── plugin.py
    ├── registry.py
    ├── loader.py
    ├── manifest.py
    ├── permissions.py
    └── hooks/
        ├── chart_hook.py
        ├── semantic_hook.py
        ├── source_hook.py
        ├── export_hook.py
        └── event_hook.py
```

Plugin contract:

```python
class SQLvizPlugin:
    name: str
    version: str

    def register(self, registry):
        pass
```

Plugins posibles:

```text
sqlviz-plugin-forecast
sqlviz-plugin-geospatial
sqlviz-plugin-pdf-export
sqlviz-plugin-motherduck
sqlviz-plugin-custom-charts
```

Regla de oro:

```text
Plugins dependen del SDK.
El core no depende de plugins.
```

---

# 16. Diagnostics: `sqlviz-benchmarks`

Como el banco de pruebas del sistema operativo.

```text
packages/sqlviz-benchmarks/

├── pyproject.toml
└── src/sqlviz_benchmarks/
    ├── __init__.py
    ├── runner.py
    ├── evaluator.py
    ├── metrics.py
    ├── reports.py
    └── corpus/
        ├── trend.yaml
        ├── comparison.yaml
        ├── ranking.yaml
        ├── kpi.yaml
        ├── distribution.yaml
        └── correlation.yaml
```

Mide:

```text
intent_accuracy
chart_accuracy
layout_accuracy
latency
cache_hit_rate
explainability_coverage
```

Sin esto, no sabes si SQLviz mejora.

---

# 17. Security: `sqlviz-security`

Aunque sea open source, debe existir.

```text
packages/sqlviz-security/

├── pyproject.toml
└── src/sqlviz_security/
    ├── __init__.py
    ├── sql_sandbox.py
    ├── query_guard.py
    ├── permissions.py
    ├── secrets.py
    └── audit.py
```

Responsabilidades:

```text
bloquear SQL peligroso
evitar escritura no autorizada
controlar fuentes
proteger credenciales
auditar ejecución
```

---

# 18. SDK: `sqlviz-sdk`

Para que otros construyan encima.

```text
packages/sqlviz-sdk/

├── pyproject.toml
└── src/sqlviz_sdk/
    ├── __init__.py
    ├── client.py
    ├── models.py
    ├── dashboard.py
    ├── inference.py
    └── plugins.py
```

Ejemplo:

```python
from sqlviz_sdk import SQLvizClient

client = SQLvizClient("http://localhost:8000")
result = client.infer(sql)
```

---

# 19. Orden real de construcción

Aunque la estructura sea grande, no implementes todo.

Implementa en capas:

```text
Fase 1: Kernel mínimo
- sqlviz-core
- RuntimeContext
- Engine contract
- Explanation model

Fase 2: Parser
- sqlviz-parser
- SQLGlot
- fingerprint

Fase 3: Inference mínimo
- features
- intent
- chart

Fase 4: Runtime
- pipeline runner
- execution tracking

Fase 5: API
- POST /infer
- POST /dashboard

Fase 6: Benchmark
- 100 SQL cases

Fase 7: UI
- SQL editor
- dashboard render
- explanation panel

Fase 8: Plugins
- SDK
- registry
- hooks
```

---

# 20. La arquitectura final como sistema operativo

```text
SQLviz Analytical OS

├── Kernel
│   └── sqlviz-core
│
├── CPU / Brain
│   ├── sqlviz-parser
│   ├── sqlviz-inference
│   └── sqlviz-viz
│
├── Scheduler
│   └── sqlviz-runtime
│
├── Signals
│   └── sqlviz-events
│
├── Filesystem
│   └── sqlviz-storage
│
├── Memory
│   └── sqlviz-cache
│
├── Drivers
│   └── sqlviz-sources
│
├── Syscalls
│   └── sqlviz-api
│
├── Desktop
│   └── sqlviz-web
│
├── Shell
│   └── sqlviz-cli
│
├── Package Manager
│   └── sqlviz-plugins
│
├── Diagnostics
│   └── sqlviz-benchmarks
│
└── Security
    └── sqlviz-security
```

La idea más importante:

```text
SQLviz debe tener un kernel estable.

Todo lo demás debe conectarse por contratos.
```

Así en el futuro podrás agregar:

```text
Forecasting
LLMs
Plugins
Autonomous Analysis
Genome
Collaboration
Cloud
```

sin romper el cerebro.