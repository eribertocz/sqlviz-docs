# SQLviz Architecture Handbook

# Capítulo 19 — Autonomous Analysis Engine

Este probablemente sea el componente más diferenciador de todo SQLviz.

Porque aquí dejamos de construir dashboards.

Y empezamos a construir analistas.

---

# ¿Qué es realmente?

Un dashboard responde:

```text
¿Qué pasó?
```

---

Un analista responde:

```text
¿Por qué pasó?
```

---

SQLviz debe acercarse al segundo caso.

---

Ejemplo.

Usuario:

```sql
SELECT
    month,
    SUM(revenue)
FROM sales
GROUP BY month
```

---

Insight Engine detecta:

```text
Revenue cayó 22%
```

---

La mayoría de herramientas terminan aquí.

---

SQLviz debe continuar.

```text
Revenue cayó 22%

↓ investigar

Country

↓ investigar

Product

↓ investigar

Customer Segment

↓ investigar

Channel

↓ generar conclusión
```

---

Resultado:

```text
Revenue cayó 22%.

El 78% de la caída proviene de Argentina.

Dentro de Argentina, la categoría Electronics
representa el 64% de la disminución.

El canal Retail explica el 81% de la pérdida.
```

---

# Filosofía

El usuario escribe:

```sql
SELECT ...
```

---

SQLviz ejecuta:

```text
1 análisis
```

---

Autonomous Analysis ejecuta:

```text
50 análisis
```

---

y devuelve:

```text
Los 3 hallazgos más importantes
```

---

# Arquitectura

```text
Insight
   │
   ▼
Hypothesis Generator
   │
   ▼
Investigation Planner
   │
   ▼
Query Generator
   │
   ▼
Analysis Executor
   │
   ▼
Finding Ranker
   │
   ▼
Final Findings
```

---

# Hipótesis

Cada insight genera preguntas.

---

Ejemplo.

Insight:

```text
Revenue cayó 22%
```

---

Hipótesis:

```text
¿Fue un país?

¿Fue un producto?

¿Fue un cliente?

¿Fue un canal?

¿Fue una región?
```

---

# Investigation Tree

Representación.

```text
Revenue ↓

├─ Country
│
├─ Product
│
├─ Customer
│
└─ Channel
```

---

# Query Expansion

Consulta original.

```sql
SELECT
    month,
    SUM(revenue)
FROM sales
GROUP BY month
```

---

SQLviz genera.

```sql
GROUP BY country
```

---

```sql
GROUP BY product
```

---

```sql
GROUP BY customer_segment
```

---

sin intervención humana.

---

# Search Space Explosion

Problema.

Si existen:

```text
20 dimensiones
```

y

```text
15 métricas
```

---

Las combinaciones son enormes.

---

Necesitamos priorizar.

---

# Investigation Score

```python
investigation_score =
(
 dimension_importance * 0.4
 +
 insight_relevance * 0.4
 +
 historical_success * 0.2
)
```

---

# Root Cause Analysis

Objetivo.

Encontrar:

```text
La explicación más simple
```

---

Ejemplo.

```text
Revenue ↓ 22%
```

↓

```text
Argentina ↓ 35%
```

↓

```text
Electronics ↓ 50%
```

↓

```text
Retail ↓ 61%
```

---

Root Cause encontrada.

---

# Finding Model

```python
class Finding:

    finding_id: str

    finding_type: str

    explanation: str

    confidence: float

    impact: float
```

---

# Findings Registry

DuckDB.

```sql
CREATE TABLE findings (

    finding_id VARCHAR,

    finding_type VARCHAR,

    explanation TEXT,

    confidence DOUBLE,

    impact DOUBLE,

    created_at TIMESTAMP
);
```

---

# Investigation Graph

Representación.

```text
Revenue Drop
      │
      ▼
Argentina
      │
      ▼
Electronics
      │
      ▼
Retail
```

---

# Explainability

Todo hallazgo debe ser explicable.

---

Ejemplo.

```json
{
  "finding":"Argentina Electronics Retail",
  "impact":0.78,
  "confidence":0.94
}
```

---

# Hallazgo vs Insight

Insight:

```text
Revenue cayó 22%
```

---

Finding:

```text
La caída se explica principalmente por
Electronics Retail en Argentina.
```

---

# Análisis Autónomo Nivel 1

Capacidad inicial.

---

Buscar:

```text
Top Drivers

Top Contributors

Top Declines

Top Growth
```

---

Fácil de implementar.

---

# Análisis Autónomo Nivel 2

Añadir.

```text
Root Cause Trees

Dimension Traversal

Metric Relationships
```

---

# Análisis Autónomo Nivel 3

Añadir.

```text
Hypothesis Ranking

Multi-step Investigation

Finding Graphs
```

---

# Análisis Autónomo Nivel 4

Visión futura.

---

SQLviz ejecuta:

```text
100+

consultas derivadas
```

---

y produce:

```text
Executive Summary

Top Risks

Top Opportunities

Top Findings
```

---

# Integración con Recommendation Engine

Recommendation:

```text
Analiza Product
```

---

Autonomous Engine:

```text
Ya lo analicé
```

---

y devuelve resultado.

---

# Integración con Dashboard Genome

Genome:

```text
Revenue Dashboard
```

---

sabe qué investigaciones suelen ser útiles.

---

# Integración con Learning Engine

Registrar:

```text
Finding leído

Finding ignorado

Finding compartido

Finding exportado
```

---

# Métrica más importante

No precisión.

---

Sino:

```text
Hallazgos útiles por consulta
```

---

Porque el objetivo final es:

```text
Menos dashboards

Más comprensión
```

---

# El Gran Cambio

La mayoría de herramientas BI hacen:

```text
Consulta
 ↓
Visualización
```

---

SQLviz busca:

```text
Consulta
 ↓
Visualización
 ↓
Insight
 ↓
Investigación
 ↓
Hallazgo
```

---

# Principio Fundamental

El Autonomous Analysis Engine transforma insights en investigaciones automáticas.

Es el motor que permite a SQLviz buscar explicaciones, descubrir causas y construir narrativas analíticas sin que el usuario tenga que explorar manualmente cada dimensión.

Sin él:

```text
SQLviz responde preguntas.
```

Con él:

```text
SQLviz empieza a investigar.
```