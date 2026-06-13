# SQLviz — Technical Stack
**Version:** v0.1.0 (Draft)
**Status:** Work in Progress
**Last Updated:** 2026-06-08

---

## 1. Overview

SQLviz is built on a small set of carefully chosen tools.
Every tool was selected for a specific reason.
Every tool that was rejected was rejected for a specific reason.

```
Layer               Tool                Purpose
─────────────────────────────────────────────────────────────
CLI                 Python              Entry point
SQL Parser          sqlglot             AST generation
Inference Engine    Python + DuckDB     Analytical brain
Data Source Registry Python             Manages all data engines
Viz Engine Registry  Python             Manages all chart engines
Analytical Engine   DuckDB              Query execution (default)
Metadata Storage    SQLite (.sqlviz)    Project data
Brain Storage       DuckDB (brain)      Learned patterns
API                 FastAPI             HTTP interface
Frontend            SvelteKit + Svelte 5 UI
SQL Editor          Monaco Editor       Code editing
Charts              ECharts             Visualizations (default)
Styling             Tailwind CSS v4     UI design system
Event Bus           Python (in-memory)  Engine communication
```

---

## 2. The Registry Pattern — Designed from Day One

The most important architectural decision in SQLviz
is the Registry Pattern for both data sources and visualization engines.

This is not a future feature. It is the foundation from v0.1.

### Why the Registry Pattern

```
Without Registry (wrong):
    if engine == "duckdb":
        result = duckdb_execute(sql)
    elif engine == "clickhouse":
        result = clickhouse_execute(sql)
    # Adding PostgreSQL requires modifying this code
    # Violates Open/Closed Principle
    # Not sustainable

With Registry (correct):
    source = data_source_registry.get(engine)
    result = source.execute(sql)
    # Adding PostgreSQL means registering a new class
    # Core code never changes
    # Sustainable by one person
```

### DataSource Registry

```python
# sqlviz-core/contracts/data_source.py

class DataSourceContract(Protocol):
    """
    Universal contract for all data sources.
    DuckDB, ClickHouse, PostgreSQL, MySQL — all implement this.
    The inference engine never knows which source is being used.
    """
    name: str
    version: str

    def execute(
        self,
        sql: str,
        context: QueryContext
    ) -> QueryResult:
        """Execute a SQL query and return results."""
        ...

    def schema(
        self,
        table: str
    ) -> SchemaResult:
        """Return column names and types for a table."""
        ...

    def test_connection(self) -> bool:
        """Verify the connection is working."""
        ...

    def describe(
        self,
        sql: str
    ) -> list[ColumnSchema]:
        """Return column schema for a SQL query result."""
        ...


# sqlviz-core/registry/data_source_registry.py

class DataSourceRegistry:
    """
    Central registry for all data sources.
    Sources register themselves. Core never changes.
    """
    _sources: dict[str, DataSourceContract] = {}

    @classmethod
    def register(cls, source: DataSourceContract) -> None:
        cls._sources[source.name] = source

    @classmethod
    def get(cls, name: str) -> DataSourceContract:
        if name not in cls._sources:
            raise SourceNotFoundError(f"Data source '{name}' not registered")
        return cls._sources[name]

    @classmethod
    def available(cls) -> list[str]:
        return list(cls._sources.keys())


# Registration at startup — sqlviz-api/main.py

from sqlviz_sources import DuckDBSource, ClickHouseSource

DataSourceRegistry.register(DuckDBSource())      # always registered
# DataSourceRegistry.register(ClickHouseSource()) # registered if configured
```

### VizEngine Registry

```python
# sqlviz-core/contracts/viz_engine.py

class VizEngineContract(Protocol):
    """
    Universal contract for all visualization engines.
    ECharts, Plotly, D3 — all implement this.
    The frontend never knows which engine rendered the chart.
    """
    name: str
    supported_charts: list[ChartType]

    def render(
        self,
        chart_type: ChartType,
        data: list[dict],
        columns: list[ColumnSchema],
        config: ChartConfig
    ) -> ChartSpec:
        """
        Return a ChartSpec — a serializable object
        that the frontend knows how to render.
        """
        ...

    def supports(self, chart_type: ChartType) -> bool:
        """Returns True if this engine can render this chart type."""
        ...


# sqlviz-core/registry/viz_engine_registry.py

class VizEngineRegistry:
    _engines: dict[str, VizEngineContract] = {}

    @classmethod
    def register(cls, engine: VizEngineContract) -> None:
        cls._engines[engine.name] = engine

    @classmethod
    def get(cls, name: str) -> VizEngineContract:
        return cls._engines.get(name, cls._engines["echarts"])

    @classmethod
    def best_for(cls, chart_type: ChartType) -> VizEngineContract:
        """
        Returns the best engine for a given chart type.
        ECharts for business charts.
        Plotly for scientific/statistical charts.
        """
        for engine in cls._engines.values():
            if engine.supports(chart_type):
                return engine
        return cls._engines["echarts"]  # fallback


# Registration at startup

from sqlviz_viz import EChartsEngine, PlotlyEngine

VizEngineRegistry.register(EChartsEngine())   # always registered
# VizEngineRegistry.register(PlotlyEngine())  # registered if installed
```

### How panels use the registries

Every panel stores which engines it uses:

```
panels table in .sqlviz:
    engine     TEXT DEFAULT 'duckdb'    ← data source
    viz_engine TEXT DEFAULT 'echarts'  ← visualization engine
```

At execution time:

```python
# In the runtime pipeline

source = DataSourceRegistry.get(panel.engine)
result = source.execute(panel.sql_content, context)

viz = VizEngineRegistry.get(panel.viz_engine)
chart_spec = viz.render(result.chart_type, result.data, result.columns, config)
```

The inference engine, the runtime and the frontend
never contain if/elif chains for engine types.
They only use the registry.

---

## 3. Data Sources

### v0.1 — DuckDB (Default)

```
Status:     Implemented from day one
Use case:   Primary analytical engine for all users
```

DuckDB is the default data source.
It runs embedded — no server required.
It reads Parquet, CSV, JSON, and SQLite natively.

```python
class DuckDBSource:
    name = "duckdb"
    version = "1.x"

    def execute(self, sql, context) -> QueryResult:
        conn = duckdb.connect()
        df = conn.execute(sql).df()
        return QueryResult(data=df.to_dict('records'), ...)

    def describe(self, sql) -> list[ColumnSchema]:
        conn = duckdb.connect()
        result = conn.execute(f"DESCRIBE {sql}").fetchall()
        return [ColumnSchema(name=r[0], type=r[1]) for r in result]
```

### v0.2 — ClickHouse

```
Status:     Architecture ready from v0.1, implemented in v0.2
Use case:   High volume OLAP — billions of rows, sub-second queries
            Enterprise data warehouses
            Real-time analytics
```

ClickHouse is the second data source.
It requires a server but handles data volumes
that DuckDB cannot.

```python
class ClickHouseSource:
    name = "clickhouse"

    def execute(self, sql, context) -> QueryResult:
        client = clickhouse_connect.get_client(
            host=context.config.clickhouse_host,
            port=context.config.clickhouse_port,
            username=context.config.clickhouse_user,
            password=context.config.clickhouse_password,
        )
        result = client.query(sql)
        return QueryResult(data=result.named_results(), ...)
```

Configuration stored in .sqlviz settings:

```sql
INSERT INTO settings VALUES ('clickhouse_host', 'localhost', datetime('now'));
INSERT INTO settings VALUES ('clickhouse_port', '8123', datetime('now'));
INSERT INTO settings VALUES ('clickhouse_user', 'default', datetime('now'));
INSERT INTO settings VALUES ('clickhouse_password', '', datetime('now'));
```

### v0.3+ — Future Sources

```
PostgreSQL  → most common production database
MySQL       → widely used in web applications
MotherDuck  → DuckDB in the cloud
BigQuery    → Google Cloud data warehouse
Snowflake   → cloud data warehouse
```

Each source implements DataSourceContract.
No changes to the core when a new source is added.

---

## 4. Visualization Engines

### v0.1 — ECharts (Default)

```
Status:     Implemented from day one
Use case:   Business intelligence charts
Charts:     Bar, Line, Area, Pie, Scatter, KPI,
            Multiline, Combo, Bar Horizontal, Treemap
```

ECharts is designed for business intelligence dashboards.
It handles the 90% of charts that analysts need every day.

```typescript
class EChartsEngine implements VizEngineContract {
    name = "echarts"
    supported_charts = [
        ChartType.BAR, ChartType.LINE, ChartType.PIE,
        ChartType.KPI, ChartType.SCATTER, ChartType.AREA,
        ChartType.MULTILINE, ChartType.COMBO,
        ChartType.BAR_HORIZONTAL, ChartType.TABLE
    ]

    render(chart_type, data, columns, config): ChartSpec {
        const option = this.buildOption(chart_type, data, columns, config)
        return { engine: "echarts", option }
    }
}
```

### v0.2 — Plotly

```
Status:     Architecture ready from v0.1, implemented in v0.2
Use case:   Scientific and statistical charts
Charts:     Boxplot, Histogram, Heatmap, 3D Scatter,
            Violin, Waterfall, Funnel, Gantt, Bubble
```

Plotly is designed for statistical and scientific analysis.
When the Inference Engine detects statistical intent,
it recommends Plotly automatically.

```typescript
class PlotlyEngine implements VizEngineContract {
    name = "plotly"
    supported_charts = [
        ChartType.BOXPLOT, ChartType.HISTOGRAM,
        ChartType.HEATMAP, ChartType.SCATTER_3D,
        ChartType.VIOLIN, ChartType.WATERFALL,
        ChartType.FUNNEL, ChartType.BUBBLE
    ]

    render(chart_type, data, columns, config): ChartSpec {
        const traces = this.buildTraces(chart_type, data, columns)
        const layout = this.buildLayout(config)
        return { engine: "plotly", traces, layout }
    }
}
```

The frontend handles both engines:

```svelte
{#if chart_spec.engine === 'echarts'}
    <EChartsRenderer spec={chart_spec} />
{:else if chart_spec.engine === 'plotly'}
    <PlotlyRenderer spec={chart_spec} />
{/if}
```

### v0.4 — D3

```
Status:     Planned for plugin authors
Use case:   Fully custom charts
            Maximum flexibility
            Charts that no library provides out of the box
```

---

## 5. Backend Stack

### Python 3.12+

```
Why Python:
✅ sqlglot is Python — no language bridge
✅ DuckDB has first-class Python API
✅ FastAPI is Python — one language for everything
✅ uv for fast dependency management
✅ Readable — the inference engine must be maintainable
```

Package manager: **uv** (not pip, not poetry)

```bash
uv add sqlglot duckdb fastapi uvicorn
uv run sqlviz my_project
```

### sqlglot

**Purpose:** SQL parsing and AST generation — foundation of inference

```python
import sqlglot

sql = "SELECT month, SUM(revenue) FROM sales GROUP BY month ORDER BY month"
ast = sqlglot.parse_one(sql, dialect="duckdb")

# Robust AST analysis — not fragile regex
has_group_by    = ast.find(sqlglot.exp.Group) is not None
aggregations    = list(ast.find_all(sqlglot.exp.AggFunc))
order_by        = ast.find(sqlglot.exp.Order)
window_funcs    = list(ast.find_all(sqlglot.exp.Window))
```

Why sqlglot and not regex:

```
Regex fails with:
    GROUP  BY (extra space)
    -- GROUP BY (SQL comment)
    'GROUP BY' (string literal)
    nested subqueries
    CTEs

sqlglot handles all of these correctly.
It parses the SQL the same way DuckDB would execute it.
```

### FastAPI

**Rule:** FastAPI is only a door. No business logic inside routers.

```python
@router.post("/api/v1/infer")
async def infer(request: InferRequest) -> InferResponse:
    context = RuntimeContext(sql=request.sql)
    context = await runtime.execute(context)
    return InferResponse.from_context(context)
```

---

## 6. Frontend Stack

### SvelteKit + Svelte 5

**The rule:** Frontend never infers. Frontend only renders.

```
All inference → backend
All rendering → frontend

Frontend receives a complete DashboardResponse.
It renders it. No analytical logic in the frontend.
```

### Monaco Editor

**Why Monaco and not CodeMirror:**

```
SQLviz users write complex SQL:
→ CTEs with multiple levels
→ Window functions
→ Multiple JOINs with subqueries
→ Long analytical queries

Monaco is designed for complex code editing.
It is the editor that powers VS Code.
Maintained by Microsoft.
Full IntelliSense, autocomplete, go-to-definition.
The current standard for browser code editors.

CodeMirror is designed for simple embedded editors.
It is not the right tool for complex SQL.
```

### ECharts + Plotly (frontend)

Both engines are registered in the frontend.
The backend sends a ChartSpec that includes which engine to use.
The frontend renders with the correct engine.

### Tailwind CSS v4

```css
/* SQLviz design tokens — defined once, used everywhere */
:root {
  --sqlviz-primary:    #6366f1;
  --sqlviz-bg:         #0f172a;
  --sqlviz-bg-surface: #1e293b;
  --sqlviz-border:     #334155;
  --sqlviz-text:       #f1f5f9;
  --sqlviz-text-muted: #94a3b8;
  --sqlviz-radius:     6px;
}
```

---

## 7. Storage Architecture

```
~/.sqlviz/
└── brain.duckdb
    ├── sql_patterns       ← learned fingerprint → chart mappings
    ├── layout_patterns    ← learned layout preferences
    ├── semantic_dict      ← learned column name meanings
    ├── corrections        ← user correction history
    └── learning_stats     ← inference accuracy over time

    Scope: global to the user
    Persists across: virtual environments, upgrades, project deletions
    Belongs to: the SQLviz installation, not any project


my_project.sqlviz (SQLite)
    ├── _sqlviz_meta       ← project signature
    ├── dashboards         ← dashboard definitions
    ├── folders            ← folder organization
    ├── rows               ← layout rows
    ├── panels             ← panels with sql_content
    │                         engine: "duckdb" | "clickhouse"
    │                         viz_engine: "echarts" | "plotly"
    ├── filter_memory      ← last used filter values
    └── settings           ← ClickHouse config, theme, etc

    Scope: per project
    Belongs to: the user
    Can be: shared, deleted, moved, committed to Git
```

---

## 8. What We Explicitly Avoid

```
Tool            Reason rejected
──────────────────────────────────────────────────────────
Kafka           Requires server — violates Principle 5
Redis           Same reasons as Kafka
Kubernetes      DevOps overhead — not for a local tool
LLMs / GPT API  External API, costs money, privacy concerns,
                requires internet — SQLviz works offline
Pandas          DuckDB is faster — one less dependency
SQLAlchemy      Overkill — direct sqlite3 stdlib is sufficient
React/Vue       Svelte 5 is simpler, smaller bundle
CodeMirror      Not suitable for complex SQL — Monaco wins
Webpack         Vite is faster — SvelteKit uses Vite
```

---

## 9. Dependency Philosophy

Every dependency must answer yes to all of these:

```
1. Is it necessary?
   Can we implement this in < 200 lines?
   If yes → implement it ourselves

2. Is it maintained?
   Last commit < 6 months ago?
   Active issue tracker?

3. Does it work offline?
   No external API calls at runtime
   No license servers

4. Is it sustainable by one person?
   Can a single developer understand and maintain it?
   Does it require infrastructure?

5. Does it have a stable API?
   Breaking changes should be rare
```

---

*SQLviz Technical Stack — v0.1.0 Draft*
*"The right tool for the right job."*
*"Designed for extensibility from day one."*
