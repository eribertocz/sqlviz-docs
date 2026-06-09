# SQLviz Architecture Handbook

## Capítulo 21 — DuckDB Physical Model

Versión: 0.1

---

# 1. Introducción

DuckDB no será solamente:

```text
Motor SQL
```

---

Será también:

```text
Runtime Store

Knowledge Store

Metadata Store

Learning Store
```

---

# 2. Filosofía

No crear 200 tablas.

No crear un ERP.

No crear un Data Warehouse interno.

---

Principio:

```text
Pocas tablas

Muy expresivas

Muy flexibles
```

---

# 3. Capas de Datos

SQLviz almacena cuatro tipos de información.

```text
User Data

Metadata

Knowledge

Runtime
```

---

# 4. Physical Layout

```text
duckdb/

├── user_data
├── metadata
├── knowledge
├── runtime
└── learning
```

---

# 5. Metadata Layer

Contiene:

```text
Datasets

Columns

Metrics

Dimensions

Semantics
```

---

# 6. Dataset Registry

```sql
CREATE TABLE datasets (

    dataset_id VARCHAR,

    dataset_name VARCHAR,

    source_type VARCHAR,

    created_at TIMESTAMP
);
```

---

# 7. Column Registry

Una de las tablas más importantes.

```sql
CREATE TABLE columns (

    dataset_id VARCHAR,

    column_name VARCHAR,

    data_type VARCHAR,

    cardinality BIGINT,

    null_ratio DOUBLE,

    distinct_ratio DOUBLE
);
```

---

# 8. Column Statistics

SQLviz calcula automáticamente:

```text
Min

Max

Avg

Median

Distinct

Nulls
```

---

# 9. Column Profile Table

```sql
CREATE TABLE column_profiles (

    dataset_id VARCHAR,

    column_name VARCHAR,

    profile JSON
);
```

---

# 10. Semantic Registry

Base del Semantic Engine.

```sql
CREATE TABLE semantic_entities (

    entity_id VARCHAR,

    entity_name VARCHAR,

    semantic_type VARCHAR,

    confidence DOUBLE
);
```

---

# 11. Semantic Dictionary

Ejemplos.

```text
revenue

sales

income
```

↓

```text
METRIC_REVENUE
```

---

# 12. Metric Registry

```sql
CREATE TABLE metrics (

    metric_id VARCHAR,

    metric_name VARCHAR,

    semantic_type VARCHAR,

    score DOUBLE
);
```

---

# 13. Dimension Registry

```sql
CREATE TABLE dimensions (

    dimension_id VARCHAR,

    dimension_name VARCHAR,

    semantic_type VARCHAR,

    score DOUBLE
);
```

---

# 14. Hierarchy Registry

```sql
CREATE TABLE dimension_hierarchies (

    parent_dimension VARCHAR,

    child_dimension VARCHAR
);
```

---

# 15. Knowledge Layer

Aquí vive el conocimiento generado.

---

# 16. Insight Registry

```sql
CREATE TABLE insights (

    insight_id VARCHAR,

    insight_type VARCHAR,

    confidence DOUBLE,

    impact DOUBLE,

    payload JSON
);
```

---

# 17. Finding Registry

Autonomous Analysis.

```sql
CREATE TABLE findings (

    finding_id VARCHAR,

    explanation TEXT,

    confidence DOUBLE,

    impact DOUBLE
);
```

---

# 18. Recommendation Registry

```sql
CREATE TABLE recommendations (

    recommendation_id VARCHAR,

    recommendation_type VARCHAR,

    score DOUBLE,

    payload JSON
);
```

---

# 19. Genome Registry

```sql
CREATE TABLE genomes (

    genome_id VARCHAR,

    genome JSON,

    score DOUBLE
);
```

---

# 20. Knowledge Graph Tables

Nodo.

```sql
CREATE TABLE graph_nodes (

    node_id VARCHAR,

    node_type VARCHAR,

    payload JSON
);
```

---

# 21. Graph Edges

```sql
CREATE TABLE graph_edges (

    source_node VARCHAR,

    target_node VARCHAR,

    edge_type VARCHAR,

    weight DOUBLE
);
```

---

# 22. Runtime Layer

Información temporal.

---

# 23. Query Registry

```sql
CREATE TABLE queries (

    query_id VARCHAR,

    sql TEXT,

    created_at TIMESTAMP
);
```

---

# 24. Query Results

Opcional.

```sql
CREATE TABLE query_results (

    query_id VARCHAR,

    result_hash VARCHAR,

    payload JSON
);
```

---

# 25. Session Registry

```sql
CREATE TABLE sessions (

    session_id VARCHAR,

    created_at TIMESTAMP
);
```

---

# 26. Event Store

Uno de los componentes más importantes.

```sql
CREATE TABLE events (

    event_id VARCHAR,

    event_type VARCHAR,

    payload JSON,

    created_at TIMESTAMP
);
```

---

# 27. Event Types

```text
QUERY_EXECUTED

INSIGHT_CREATED

FILTER_USED

PANEL_CREATED

GENOME_UPDATED
```

---

# 28. Learning Layer

Todo aprendizaje va aquí.

---

# 29. Feedback Registry

```sql
CREATE TABLE feedback (

    feedback_id VARCHAR,

    object_type VARCHAR,

    object_id VARCHAR,

    action VARCHAR
);
```

---

# 30. Recommendation Feedback

```sql
CREATE TABLE recommendation_feedback (

    recommendation_id VARCHAR,

    action VARCHAR
);
```

---

# 31. Insight Feedback

```sql
CREATE TABLE insight_feedback (

    insight_id VARCHAR,

    action VARCHAR
);
```

---

# 32. Genome Feedback

```sql
CREATE TABLE genome_feedback (

    genome_id VARCHAR,

    action VARCHAR
);
```

---

# 33. Cache Layer

DuckDB puede almacenar cache.

---

# 34. Cache Registry

```sql
CREATE TABLE cache_entries (

    cache_key VARCHAR,

    payload JSON,

    expires_at TIMESTAMP
);
```

---

# 35. Job System

V1 simple.

---

```sql
CREATE TABLE jobs (

    job_id VARCHAR,

    job_type VARCHAR,

    status VARCHAR,

    payload JSON
);
```

---

# 36. Investigation Queue

Autonomous BI.

```sql
CREATE TABLE investigations (

    investigation_id VARCHAR,

    priority DOUBLE,

    status VARCHAR,

    payload JSON
);
```

---

# 37. Physical Domains

Agrupar tablas.

```text
metadata.*

knowledge.*

runtime.*

learning.*
```

---

# 38. JSON First Principle

Regla importante.

---

Evitar:

```text
100 columnas
```

---

Preferir:

```text
payload JSON
```

cuando estructura evoluciona rápido.

---

# 39. Materialized Intelligence

SQLviz puede materializar.

---

Ejemplo.

```sql
CREATE TABLE insight_summary AS
SELECT ...
```

---

para acelerar consultas.

---

# 40. Knowledge Persistence

Lo importante no son dashboards.

---

Lo importante es guardar:

```text
Semántica

Insights

Hallazgos

Aprendizaje
```

---

# 41. Backup Strategy

Simple.

---

```text
DuckDB File
```

↓

```text
Git Snapshot

Filesystem Backup

Cloud Sync
```

---

# 42. Migration Strategy

Versionado.

```sql
schema_version
```

---

Tabla.

```sql
CREATE TABLE schema_versions (

    version INTEGER,

    applied_at TIMESTAMP
);
```

---

# 43. Physical Design Principle

La mayoría de BI almacenan:

```text
Dashboards
```

---

SQLviz almacena:

```text
Conocimiento Analítico
```

---

# 44. El Gran Cambio

Power BI guarda:

```text
Visualizaciones
```

---

SQLviz guarda:

```text
Semántica

Intenciones

Insights

Hallazgos

Genomes
```

---

# 45. Principio Fundamental

DuckDB no es solamente la base de datos de SQLviz.

Es la memoria persistente del sistema analítico.

Es donde se almacena todo lo que SQLviz aprende, descubre e infiere.

Sin este modelo:

```text
SQLviz olvida.
```

Con él:

```text
SQLviz acumula conocimiento.
```