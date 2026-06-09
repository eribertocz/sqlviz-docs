# SQLviz Architecture Handbook

## Capítulo 4 — Feature Registry & Feature Store

Versión: 0.1

---

# 1. Introducción

A medida que SQLviz evoluciona aparecerán cientos de señales.

Ejemplos:

```text
group_by_count
join_count
trend_strength
seasonality
outlier_ratio
entropy
correlation
time_dimension
geo_dimension
retention_metric
```

Al principio parece manejable.

Pero cuando SQLviz alcance:

```text
100+
200+
300+
features
```

aparecerá un problema serio.

Nadie sabrá:

* qué feature existe
* cómo se calcula
* quién la usa
* cuánto cuesta calcularla
* si sigue siendo útil

Por eso se necesita un Feature Registry.

---

# 2. Objetivo

Centralizar todas las features del sistema.

Cada feature debe ser:

```text
Identificable
Documentada
Versionable
Auditable
Medible
```

---

# 3. Arquitectura

```text
SQL
 │
 ▼
Feature Extraction
 │
 ▼
Feature Registry
 │
 ▼
Feature Store
 │
 ▼
Intent Engine
Chart Engine
Filter Engine
Layout Engine
```

---

# 4. Qué es una Feature

Una feature es una señal cuantificable.

Ejemplo:

```python
trend_strength = 0.92
```

No es una visualización.

No es una inferencia.

Es una observación.

---

# 5. Metadatos de una Feature

Cada feature debe tener metadatos.

---

Ejemplo:

```python
FeatureDefinition(
    name="trend_strength",
    category="statistical",
    data_type="float",
    min_value=0,
    max_value=1,
    description="Linear trend coefficient"
)
```

---

# 6. Categorías Oficiales

SQLviz define categorías.

```text
sql
schema
data
statistical
semantic
execution
behavioral
learning
```

---

# 7. SQL Features

Ejemplos:

```text
group_by_count
order_by_count
limit_count
window_count
join_count
aggregation_count
```

---

# 8. Schema Features

Ejemplos:

```text
date_columns
numeric_columns
varchar_columns
primary_keys
foreign_keys
```

---

# 9. Statistical Features

Ejemplos:

```text
skewness
kurtosis
entropy
variance
trend_strength
seasonality
correlation
```

---

# 10. Semantic Features

Ejemplos:

```text
contains_revenue
contains_profit
contains_region
contains_country
contains_retention
contains_cohort
```

---

# 11. Behavioral Features

Aparecen después.

Ejemplos:

```text
user_accept_rate
chart_override_rate
filter_usage_rate
```

---

# 12. Learning Features

Generadas por el sistema.

Ejemplos:

```text
historical_line_score
historical_bar_score
historical_kpi_score
```

---

# 13. Feature Definition Model

Python:

```python
from pydantic import BaseModel

class FeatureDefinition(BaseModel):

    name: str

    category: str

    datatype: str

    description: str

    min_value: float | None

    max_value: float | None

    default_value: float | None

    enabled: bool = True
```

---

# 14. DuckDB Registry

Tabla principal.

```sql
CREATE TABLE feature_registry (

    feature_name VARCHAR PRIMARY KEY,

    category VARCHAR,

    datatype VARCHAR,

    description VARCHAR,

    min_value DOUBLE,

    max_value DOUBLE,

    default_value DOUBLE,

    enabled BOOLEAN,

    created_at TIMESTAMP
);
```

---

# 15. Ejemplo Real

```sql
INSERT INTO feature_registry
VALUES (

    'trend_strength',

    'statistical',

    'FLOAT',

    'Strength of linear trend',

    0,

    1,

    0,

    TRUE,

    NOW()
);
```

---

# 16. Feature Store

El Registry describe.

El Store almacena valores.

---

Ejemplo:

```python
trend_strength = 0.92
```

---

# 17. DuckDB Feature Store

```sql
CREATE TABLE feature_store (

    fingerprint_id VARCHAR,

    feature_name VARCHAR,

    feature_value DOUBLE,

    computed_at TIMESTAMP
);
```

---

# 18. Beneficios

Permite:

```text
Versionado
Auditoría
Debugging
Entrenamiento
Reproducibilidad
```

---

# 19. Cost Model

No todas las features cuestan igual.

---

Ejemplo:

```text
join_count
```

Costo:

```text
muy bajo
```

---

Ejemplo:

```text
seasonality
```

Costo:

```text
alto
```

---

Por eso cada feature debe tener costo.

---

Tabla:

```sql
CREATE TABLE feature_costs (

    feature_name VARCHAR,

    estimated_ms DOUBLE
);
```

---

# 20. Lazy Features

No calcular todo siempre.

---

Nivel 1

```text
SQL Features
```

Costo muy bajo.

---

Nivel 2

```text
Schema Features
```

Costo bajo.

---

Nivel 3

```text
Statistical Features
```

Costo medio.

---

Nivel 4

```text
Seasonality
FFT
AutoCorrelation
```

Costo alto.

---

Sólo calcular cuando sea necesario.

---

# 21. Feature Dependencies

Algunas features dependen de otras.

Ejemplo:

```text
seasonality
```

requiere:

```text
time_dimension
```

---

Tabla:

```sql
CREATE TABLE feature_dependencies (

    feature_name VARCHAR,

    depends_on VARCHAR
);
```

---

# 22. Feature Lineage

Permite saber:

```text
Quién creó la feature
Qué motor la usa
Cuándo fue calculada
```

---

Tabla:

```sql
CREATE TABLE feature_usage (

    feature_name VARCHAR,

    engine_name VARCHAR
);
```

---

# 23. Explainability

Cuando SQLviz toma decisiones:

```text
Line Chart
```

debe explicar:

```text
trend_strength = 0.92
time_dimension = TRUE
seasonality = 0.74
```

---

El Registry hace esto posible.

---

# 24. Future Feature Catalog

Objetivo largo plazo:

```text
300+
features
```

organizadas formalmente.

---

# 25. Principio Fundamental

Las features son el combustible.

Los motores son consumidores.

La inteligencia de SQLviz depende de:

```text
Calidad de Features
+
Organización de Features
+
Aprendizaje sobre Features
```

Por esta razón el Feature Registry y Feature Store deben existir desde el inicio del proyecto y no como una optimización futura.
