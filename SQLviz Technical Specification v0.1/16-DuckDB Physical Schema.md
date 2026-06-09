# SQLviz Technical Specification v1

## Capítulo 16 — DuckDB Physical Schema

Versión 0.1

---

# 1. Objetivo

Definir el esquema físico de DuckDB para SQLviz.

---

Hasta ahora diseñamos:

```text
Objetos

Motores

Contratos
```

---

Ahora definiremos:

```text
Tablas

Índices lógicos

Snapshots

Caches

Auditoría
```

---

# 2. Filosofía

DuckDB NO es solamente:

```text
Motor SQL
```

---

DuckDB también será:

```text
Metadata Store

Cache Store

Knowledge Store

Audit Store
```

---

# 3. Regla Fundamental

Separar:

```text
User Data
```

de

```text
SQLviz Metadata
```

---

# 4. Namespaces

Recomendación.

```text
sqlviz_metadata
```

---

y

```text
user_data
```

---

# 5. Metadata Schema

```sql
CREATE SCHEMA sqlviz_metadata;
```

---

# 6. Execution Registry

Tabla principal.

---

```sql
CREATE TABLE executions (

    execution_id VARCHAR,

    sql_text VARCHAR,

    fingerprint_hash VARCHAR,

    started_at TIMESTAMP,

    finished_at TIMESTAMP,

    duration_ms DOUBLE
);
```

---

# 7. Purpose

Permite.

```text
Auditoría

Benchmarking

Debugging
```

---

# 8. Feature Registry

```sql
CREATE TABLE features (

    execution_id VARCHAR,

    feature_name VARCHAR,

    feature_value VARCHAR,

    confidence DOUBLE
);
```

---

# 9. Semantic Registry

```sql
CREATE TABLE semantic_tags (

    execution_id VARCHAR,

    tag_name VARCHAR,

    category VARCHAR,

    confidence DOUBLE
);
```

---

# 10. Intent Registry

```sql
CREATE TABLE intents (

    execution_id VARCHAR,

    intent_name VARCHAR,

    score DOUBLE
);
```

---

# 11. Metrics Registry

```sql
CREATE TABLE metrics (

    execution_id VARCHAR,

    metric_name VARCHAR,

    metric_type VARCHAR,

    confidence DOUBLE
);
```

---

# 12. Dimensions Registry

```sql
CREATE TABLE dimensions (

    execution_id VARCHAR,

    dimension_name VARCHAR,

    dimension_type VARCHAR,

    confidence DOUBLE
);
```

---

# 13. Chart Predictions

```sql
CREATE TABLE chart_predictions (

    execution_id VARCHAR,

    chart_type VARCHAR,

    score DOUBLE
);
```

---

# 14. Layout Registry

```sql
CREATE TABLE layouts (

    execution_id VARCHAR,

    layout_type VARCHAR,

    confidence DOUBLE
);
```

---

# 15. Dashboard Registry

```sql
CREATE TABLE dashboards (

    dashboard_id VARCHAR,

    execution_id VARCHAR,

    title VARCHAR,

    dashboard_type VARCHAR,

    confidence DOUBLE,

    created_at TIMESTAMP
);
```

---

# 16. Dashboard Panels

```sql
CREATE TABLE dashboard_panels (

    panel_id VARCHAR,

    dashboard_id VARCHAR,

    title VARCHAR,

    chart_type VARCHAR,

    x INTEGER,

    y INTEGER,

    w INTEGER,

    h INTEGER
);
```

---

# 17. Explanations

Muy importante.

---

```sql
CREATE TABLE explanations (

    explanation_id VARCHAR,

    object_type VARCHAR,

    object_id VARCHAR,

    confidence DOUBLE,

    payload JSON
);
```

---

# 18. Evidences

```sql
CREATE TABLE evidences (

    evidence_id VARCHAR,

    explanation_id VARCHAR,

    evidence_type VARCHAR,

    payload JSON
);
```

---

# 19. Runtime Events

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

# 20. Cache Registry

Tabla crítica.

---

```sql
CREATE TABLE cache_entries (

    cache_key VARCHAR,

    cache_type VARCHAR,

    payload JSON,

    created_at TIMESTAMP,

    expires_at TIMESTAMP
);
```

---

# 21. Cache Types

```text
AST

FEATURES

SEMANTICS

INTENTS

CHARTS

DASHBOARDS
```

---

# 22. Fingerprint Registry

Muy importante.

---

```sql
CREATE TABLE fingerprints (

    fingerprint_hash VARCHAR,

    fingerprint_type VARCHAR,

    payload JSON
);
```

---

# 23. Why?

Reutilización.

---

# 24. Dashboard Snapshots

```sql
CREATE TABLE dashboard_snapshots (

    snapshot_id VARCHAR,

    dashboard_id VARCHAR,

    payload JSON,

    created_at TIMESTAMP
);
```

---

# 25. Why?

Versionado.

---

Benchmarking.

---

Learning futuro.

---

# 26. Semantic Dictionary

```sql
CREATE TABLE semantic_dictionary (

    term VARCHAR,

    semantic_tag VARCHAR,

    category VARCHAR,

    confidence DOUBLE
);
```

---

# 27. Semantic Relations

```sql
CREATE TABLE semantic_relations (

    source_tag VARCHAR,

    target_tag VARCHAR,

    relation_type VARCHAR
);
```

---

# 28. Intent Rules

```sql
CREATE TABLE intent_rules (

    intent_name VARCHAR,

    rule_name VARCHAR,

    weight DOUBLE
);
```

---

# 29. Chart Rules

```sql
CREATE TABLE chart_rules (

    chart_type VARCHAR,

    rule_name VARCHAR,

    weight DOUBLE
);
```

---

# 30. Runtime Metrics

```sql
CREATE TABLE runtime_metrics (

    metric_name VARCHAR,

    metric_value DOUBLE,

    created_at TIMESTAMP
);
```

---

# 31. Benchmark Corpus

Muy importante.

---

```sql
CREATE TABLE benchmark_cases (

    case_id VARCHAR,

    sql_text VARCHAR,

    expected_chart VARCHAR,

    expected_intent VARCHAR
);
```

---

# 32. Benchmark Results

```sql
CREATE TABLE benchmark_results (

    case_id VARCHAR,

    actual_chart VARCHAR,

    actual_intent VARCHAR,

    execution_time_ms DOUBLE
);
```

---

# 33. Future Learning Tables

NO implementar aún.

---

Pero reservar.

---

```sql
CREATE TABLE learning_events (...);
```

---

```sql
CREATE TABLE dashboard_genomes (...);
```

---

```sql
CREATE TABLE recommendation_events (...);
```

---

# 34. Storage Philosophy

DuckDB debe almacenar:

```text
Metadata
```

---

No.

```text
Datasets gigantes
```

---

# 35. User Data

Recomendación.

---

Mantener.

```text
Parquet
```

---

externo.

---

# 36. SQLviz Metadata

Mantener.

```text
DuckDB
```

---

interno.

---

# 37. Snapshot Strategy

Guardar JSON completos.

---

No solo fragmentos.

---

# 38. Example Snapshot

```json
{
  "dashboard": {...},
  "layout": {...},
  "charts": [...]
}
```

---

# 39. Performance Goal

Metadata queries.

```text
< 10 ms
```

---

# 40. Strategic Insight

Muchos proyectos usan DuckDB solamente como:

```text
Query Engine
```

---

SQLviz debe usar DuckDB como:

```text
Analytical Metadata Brain
```

---

# 41. Principio Fundamental

El Physical Schema no almacena dashboards.

No almacena charts.

No almacena consultas.

Almacena conocimiento sobre cómo SQLviz llegó a esas decisiones.