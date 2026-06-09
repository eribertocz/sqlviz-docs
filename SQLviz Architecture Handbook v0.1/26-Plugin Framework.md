# SQLviz Architecture Handbook

## Capítulo 26 — Plugin Framework

Versión: 0.1

---

# 1. Introducción

Hoy SQLviz tiene:

```text
Semantic Engine

Metric Engine

Dimension Engine

Chart Engine

Insight Engine
```

---

Pero mañana alguien querrá agregar:

```text
Forecasting

Anomaly Detection

ML Scoring

Geospatial Analytics

Cohort Analysis

Funnel Analysis
```

---

La pregunta es:

```text
¿Cómo agregar eso
sin tocar el núcleo?
```

---

# 2. Filosofía

El Core de SQLviz debe ser:

```text
Pequeño
```

---

Los plugins deben ser:

```text
Grandes
```

---

Nunca al revés.

---

# 3. Regla de Oro

El Core define:

```text
Contratos
```

---

Los plugins implementan:

```text
Comportamientos
```

---

# 4. Arquitectura

```text
SQLviz Core
      │
      ▼
Plugin Manager
      │
      ▼
Plugins
```

---

# 5. Plugin Types

Desde el inicio definir categorías.

```text
Engine Plugins

Chart Plugins

Insight Plugins

Recommendation Plugins

Analysis Plugins

Datasource Plugins
```

---

# 6. Engine Plugin

Ejemplo.

```text
Forecast Engine
```

---

Añade:

```text
Predicciones
```

---

sin tocar Runtime.

---

# 7. Chart Plugin

Ejemplo.

```text
Heatmap

Treemap

Sankey

Radar
```

---

# 8. Insight Plugin

Ejemplo.

```text
Anomaly Insight
```

---

Detecta anomalías automáticamente.

---

# 9. Analysis Plugin

Ejemplo.

```text
Root Cause Analysis V2
```

---

# 10. Datasource Plugin

Ejemplo.

```text
Snowflake

BigQuery

Postgres

ClickHouse
```

---

# 11. Plugin Contract

Todo plugin implementa:

```python
class Plugin:

    name: str

    version: str

    description: str
```

---

# 12. Engine Contract

```python
class EnginePlugin:

    def execute(
        self,
        context
    ):
        pass
```

---

# 13. Registration

Al arrancar.

```python
plugin_manager.load()
```

---

# 14. Plugin Discovery

Escanear.

```text
plugins/
```

---

Ejemplo.

```text
plugins/

forecast/

anomaly/

geospatial/
```

---

# 15. Plugin Manifest

Cada plugin declara.

```json
{
  "name":"forecast",
  "version":"1.0.0",
  "author":"community"
}
```

---

# 16. Registry

DuckDB.

```sql
CREATE TABLE plugins (

    plugin_id VARCHAR,

    name VARCHAR,

    version VARCHAR,

    enabled BOOLEAN
);
```

---

# 17. Plugin Lifecycle

Estados.

```text
INSTALLED

LOADED

ENABLED

DISABLED
```

---

# 18. Runtime Integration

Plugin recibe.

```python
RuntimeContext
```

---

Igual que cualquier engine.

---

# 19. Event Integration

Plugin escucha eventos.

---

Ejemplo.

```text
INSIGHT_CREATED
```

↓

```text
Anomaly Plugin
```

---

# 20. Service Injection

Plugin puede registrar.

```python
services.register()
```

---

# 21. Engine Registry

Antes.

```python
ENGINE_REGISTRY
```

---

Ahora.

```python
ENGINE_REGISTRY
+
PLUGIN_ENGINES
```

---

# 22. Chart Registry

```python
CHART_REGISTRY
```

---

Contiene.

```text
Line

Bar

Pie

Scatter
```

---

Plugins añaden.

```text
Treemap

Heatmap

Sankey
```

---

# 23. Insight Registry

```python
INSIGHT_REGISTRY
```

---

Permite nuevos insights.

---

# 24. Recommendation Registry

```python
RECOMMENDATION_REGISTRY
```

---

Permite nuevas recomendaciones.

---

# 25. Plugin Metadata

Guardar.

```text
Author

Version

Capabilities
```

---

# 26. Capability Model

Ejemplo.

```json
{
  "capabilities":[
    "forecast",
    "timeseries"
  ]
}
```

---

# 27. Compatibility Check

Antes de cargar.

---

Verificar.

```text
Core Version

API Version

Dependencies
```

---

# 28. Sandboxing

Regla importante.

---

Plugin no debe:

```text
Modificar tablas internas
```

directamente.

---

# 29. Plugin API

Acceso mediante:

```python
Repository API
```

---

Nunca:

```python
duckdb.execute(...)
```

directamente.

---

# 30. Plugin Permissions

Modelo futuro.

---

```text
READ

WRITE

EVENTS

ANALYSIS
```

---

# 31. Community Plugins

Visión.

---

```text
sqlviz-forecast

sqlviz-anomaly

sqlviz-geospatial

sqlviz-cohort

sqlviz-funnel
```

---

# 32. Plugin Marketplace

Mucho después.

---

No en V1.

---

# 33. Hot Reload

Futuro.

---

Instalar plugin.

↓

Sin reiniciar.

↓

Disponible.

---

# 34. Plugin Testing

Cada plugin debe incluir.

```text
Unit Tests

Integration Tests
```

---

# 35. Plugin Metrics

Medir.

```text
Usage

Errors

Latency
```

---

# 36. Plugin Registry Table

```sql
CREATE TABLE plugin_metrics (

    plugin_name VARCHAR,

    executions BIGINT,

    failures BIGINT
);
```

---

# 37. Plugin Explainability

Si genera insight.

---

Debe generar.

```text
Evidence

Confidence

Rules
```

---

Obligatorio.

---

# 38. Plugin Learning

Plugins pueden aprender.

---

Pero usando.

```text
Learning Engine
```

---

Nunca por cuenta propia.

---

# 39. Strategic Insight

Aquí hay algo importante.

---

Muchos proyectos diseñan:

```text
Plugin System
```

---

después de tener:

```text
200.000 líneas
```

---

Y sufren muchísimo.

---

SQLviz puede diseñarlo desde el principio.

---

# 40. Ecosystem Vision

Imagina dentro de 5 años.

---

Core.

```text
30.000 líneas
```

---

Plugins.

```text
200+
```

---

Eso significa:

```text
El ecosistema crece
sin que el Core explote.
```

---

# 41. The Hidden Superpower

Si haces bien este capítulo.

---

SQLviz deja de ser:

```text
Una herramienta BI
```

---

y se convierte en:

```text
Un sistema operativo analítico
```

---

Porque otros desarrolladores construirán encima de él.

---

# 42. Principio Fundamental

El Plugin Framework protege al Core del crecimiento.

Permite que SQLviz evolucione durante años sin convertirse en un monolito inmanejable.

Sin plugins:

```text
Todo cambio modifica el Core.
```

Con plugins:

```text
El Core permanece estable.
```