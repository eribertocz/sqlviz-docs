# SQLviz Architecture Handbook

## Capítulo 12 — Learning Engine

Versión: 0.1

---

# 1. Introducción

Todos los motores construidos hasta ahora generan decisiones.

---

Ejemplos:

```text
Intent Engine
 ↓
Trend
```

```text
Chart Engine
 ↓
Line Chart
```

```text
Filter Engine
 ↓
Country Filter
```

```text
Layout Engine
 ↓
Top Position
```

---

Pero ninguna decisión es perfecta.

La pregunta es:

```text
¿Cómo aprende SQLviz?
```

---

# 2. Objetivo

Permitir que SQLviz mejore continuamente sin:

```text
OpenAI
Anthropic
Gemini
Servicios externos
Modelos propietarios
```

---

Todo el aprendizaje debe ocurrir localmente.

---

# 3. Filosofía

No entrenar un modelo gigante.

---

No necesitamos:

```text
100B parámetros
```

---

Necesitamos:

```text
Mejorar scores
Ajustar pesos
Detectar patrones
```

---

# 4. Arquitectura

```text
User Actions
      │
      ▼
Feedback Collector
      │
      ▼
Event Store
      │
      ▼
Learning Engine
      │
      ▼
Updated Scores
      │
      ▼
Improved Inference
```

---

# 5. Qué Aprende SQLviz

Inicialmente:

```text
Intent Preferences

Chart Preferences

Filter Preferences

Layout Preferences

Dashboard Preferences

Rewrite Preferences
```

---

# 6. Feedback Events

Todo es un evento.

---

Ejemplos:

```text
chart_changed

panel_deleted

panel_added

filter_used

filter_removed

panel_moved

dashboard_saved
```

---

# 7. Event Model

```python
class FeedbackEvent:

    event_id: str

    event_type: str

    fingerprint_id: str

    payload: dict

    created_at: datetime
```

---

# 8. DuckDB Event Store

```sql
CREATE TABLE feedback_events (

    event_id VARCHAR,

    event_type VARCHAR,

    fingerprint_id VARCHAR,

    payload JSON,

    created_at TIMESTAMP
);
```

---

# 9. Event Types

## Chart Override

Usuario:

```text
Line
```

↓

```text
Area
```

---

Evento:

```json
{
  "event":"chart_override",
  "from":"line",
  "to":"area"
}
```

---

## Filter Remove

```json
{
  "event":"filter_removed",
  "filter":"country"
}
```

---

## Layout Change

```json
{
  "event":"panel_moved"
}
```

---

# 10. Acceptance Model

Toda inferencia genera:

```python
prediction
```

---

Luego ocurre:

```python
accepted
```

o

```python
rejected
```

---

# 11. Acceptance Rate

Fórmula:

```python
acceptance_rate =
accepted
/
(
accepted + rejected
)
```

---

Ejemplo:

```text
Line Chart

Accepted = 920

Rejected = 80
```

↓

```text
92%
```

---

# 12. Learning Granularity

Aprender en varios niveles.

---

## Global

Todos los usuarios.

---

## Workspace

Un proyecto.

---

## Dashboard

Un dashboard.

---

## User

Un usuario.

---

# 13. Learning Hierarchy

Prioridad.

```text
User
 ↓
Workspace
 ↓
Global
```

---

# 14. Fingerprint Learning

Patrón:

```text
TIME_SUM
```

---

Históricamente:

```text
Line 95%
Area 5%
```

---

Actualizar pesos.

---

# 15. Intent Learning

Tabla:

```sql
CREATE TABLE learned_intents (

    fingerprint_id VARCHAR,

    intent VARCHAR,

    accepted_count INTEGER,

    rejected_count INTEGER
);
```

---

# 16. Chart Learning

Tabla:

```sql
CREATE TABLE learned_charts (

    fingerprint_id VARCHAR,

    chart_type VARCHAR,

    accepted_count INTEGER,

    rejected_count INTEGER
);
```

---

# 17. Filter Learning

Tabla:

```sql
CREATE TABLE learned_filters (

    filter_name VARCHAR,

    accepted_count INTEGER,

    rejected_count INTEGER
);
```

---

# 18. Layout Learning

Tabla:

```sql
CREATE TABLE learned_layouts (

    layout_type VARCHAR,

    accepted_count INTEGER,

    rejected_count INTEGER
);
```

---

# 19. Rewrite Learning

Tabla:

```sql
CREATE TABLE learned_rewrites (

    rewrite_name VARCHAR,

    accepted_count INTEGER,

    rejected_count INTEGER
);
```

---

# 20. Bayesian Update

En lugar de usar:

```text
accepted / total
```

usar:

```python
beta_distribution
```

---

Ejemplo:

```python
alpha = accepted + 1

beta = rejected + 1
```

---

Score:

```python
alpha / (alpha + beta)
```

---

Más estable.

---

# 21. Confidence

No basta con score.

---

Ejemplo:

```text
1 aceptación
0 rechazos
```

↓

```text
100%
```

---

Pero confianza baja.

---

Usar:

```python
sample_size
```

---

# 22. Exploration vs Exploitation

Problema clásico.

---

Si siempre mostramos:

```text
Line
```

---

Nunca aprendemos:

```text
Area
```

---

Solución:

```python
epsilon_greedy
```

---

# 23. Epsilon Greedy

```python
epsilon = 0.05
```

---

95%:

```text
best choice
```

---

5%:

```text
exploration
```

---

# 24. Thompson Sampling

Versión avanzada.

---

Usar:

```python
Beta Distribution
```

---

Para elegir candidatos.

---

Excelente para SQLviz.

---

# 25. Reward Function

Definir recompensa.

---

Ejemplo:

```text
chart_accepted = +1

chart_removed = -1

dashboard_saved = +3

manual_sql_edit = -2
```

---

# 26. User Embeddings (Sin IA)

Representación simple.

---

Ejemplo:

```python
{
 "trend":0.9,
 "comparison":0.3,
 "ranking":0.8
}
```

---

Permite personalización.

---

# 27. Dashboard Archetype Learning

Detectar patrones.

---

Ejemplo:

```text
Revenue KPI

Revenue Trend

Revenue Geography
```

---

Aparece frecuentemente.

↓

Nuevo arquetipo.

---

# 28. Feature Importance Learning

Aprender pesos.

---

Ejemplo:

```python
trend_score =
(
 time_dimension * 0.4
 +
 trend_strength * 0.4
 +
 seasonality * 0.2
)
```

---

Aprender:

```text
0.4
0.4
0.2
```

---

A partir de feedback.

---

# 29. Online Weight Update

Ejemplo simple:

```python
weight =
weight
+
learning_rate
*
error
```

---

Sin ML complejo.

---

# 30. Pattern Discovery

Detectar:

```text
TIME_SUM
```

↓

```text
Line
```

---

Frecuentemente.

↓

Nuevo patrón aprendido.

---

# 31. Learning Registry

```sql
CREATE TABLE learning_registry (

    pattern VARCHAR,

    learned_score DOUBLE,

    confidence DOUBLE
);
```

---

# 32. Explainability

Toda mejora debe ser explicable.

---

Ejemplo:

```json
{
  "chart":"area",
  "reason":"historically preferred",
  "confidence":0.92
}
```

---

# 33. Retraining Jobs

Job periódico.

---

Cada:

```text
24 horas
```

---

Actualizar:

```text
Scores
Patterns
Weights
Archetypes
```

---

# 34. Learning API

FastAPI.

---

```python
POST /feedback
```

---

```python
GET /learning/stats
```

---

```python
GET /patterns
```

---

# 35. Learning Dashboard

Panel interno.

---

Mostrar:

```text
Top Patterns

Top Charts

Top Filters

Top Layouts

Acceptance Rates
```

---

# 36. Local First

Todo aprendizaje debe almacenarse en:

```text
DuckDB
```

---

Sin dependencias externas.

---

# 37. Roadmap

V1

```text
Acceptance Rates
```

---

V2

```text
Bayesian Learning
```

---

V3

```text
Thompson Sampling
```

---

V4

```text
Adaptive Scoring Engine
```

---

# 38. El Cambio Conceptual

La mayoría de herramientas BI son:

```text
Rule Based
```

---

SQLviz debe evolucionar hacia:

```text
Rule Based
+
Learning Based
```

---

Sin abandonar:

```text
Explainability
```

---

# 39. Principio Fundamental

El Learning Engine no reemplaza los motores de inferencia.

Los mejora.

Su función es transformar decisiones estáticas en decisiones adaptativas manteniendo transparencia, reproducibilidad y control total sobre el sistema.

Por esta razón debe considerarse el sistema nervioso central de SQLviz y la base para cualquier evolución futura hacia Autonomous BI.
