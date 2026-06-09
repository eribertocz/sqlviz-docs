# SQLviz Architecture Handbook

## Capítulo 18 — Recommendation Engine

Versión: 0.1

---

# 1. Introducción

Hasta ahora SQLviz puede:

```text
Entender SQL

Entender Métricas

Entender Dimensiones

Generar Dashboards

Generar Insights
```

---

Pero todavía espera que el usuario haga la siguiente pregunta.

---

Ejemplo:

Usuario:

```sql
SELECT
    country,
    SUM(revenue)
FROM sales
GROUP BY country
```

---

SQLviz responde.

```text
Revenue by Country
```

---

Pero un analista humano pensaría:

```text
¿Y por producto?

¿Y por cliente?

¿Y por canal?

¿Y la tendencia temporal?
```

---

# 2. Objetivo

Transformar:

```text
Consulta Actual
```

en:

```text
Próximos Análisis Relevantes
```

---

# 3. Filosofía

No recomendar contenido aleatorio.

---

Toda recomendación debe surgir de:

```text
Semántica

Métricas

Dimensiones

Insights

Historial

Patrones
```

---

# 4. Arquitectura

```text
Current Analysis
        │
        ▼
Context Builder
        │
        ▼
Candidate Generator
        │
        ▼
Recommendation Scoring
        │
        ▼
Recommendation Registry
        │
        ▼
Top Recommendations
```

---

# 5. Recommendation Types

SQLviz define:

```text
Metric Recommendation

Dimension Recommendation

Filter Recommendation

Panel Recommendation

Insight Recommendation

Dashboard Recommendation

Query Recommendation
```

---

# 6. Recommendation Model

```python
class Recommendation:

    recommendation_id: str

    recommendation_type: str

    title: str

    reason: str

    score: float

    confidence: float
```

---

# 7. Context Builder

Construye contexto analítico.

---

Ejemplo.

Consulta:

```sql
SELECT
    country,
    SUM(revenue)
FROM sales
GROUP BY country
```

---

Contexto:

```json
{
  "metric":"Revenue",
  "dimension":"Country",
  "intent":"Comparison"
}
```

---

# 8. Candidate Generation

Generar posibilidades.

---

Ejemplo.

```text
Revenue by Product

Revenue by Customer

Revenue by Channel

Revenue Trend
```

---

# 9. Candidate Registry

DuckDB.

```sql
CREATE TABLE recommendation_candidates (

    candidate_id VARCHAR,

    candidate_type VARCHAR,

    score DOUBLE
);
```

---

# 10. Metric Recommendations

Detectar métricas relacionadas.

---

Ejemplo.

```text
Revenue
```

↓

```text
Profit

Margin

Orders

Customers
```

---

# 11. Semantic Expansion

Semantic Engine aporta.

---

Ejemplo.

```text
Revenue
```

↓

```text
Revenue Growth

Revenue Forecast

Revenue Share
```

---

# 12. Dimension Recommendations

Ejemplo.

Usuario analiza:

```text
Country
```

---

SQLviz recomienda:

```text
Region

City

Product

Customer
```

---

# 13. Time Recommendation

Si existe:

```text
Revenue
```

pero no:

```text
Time
```

---

Recomendar.

```text
Revenue Trend
```

---

# 14. Missing Perspective Detection

Motor clave.

---

Pregunta:

```text
¿Qué perspectiva falta?
```

---

Existe:

```text
Country
```

---

Falta:

```text
Product

Customer

Channel
```

---

# 15. Recommendation Score

```python
score =
(
 relevance * 0.4
 +
 impact * 0.3
 +
 confidence * 0.2
 +
 novelty * 0.1
)
```

---

# 16. Relevance

¿Está relacionada con el análisis actual?

---

Ejemplo.

```text
Revenue by Product
```

↓

```text
0.95
```

---

# 17. Impact

¿Cuánto conocimiento nuevo aporta?

---

Ejemplo.

```text
Revenue Trend
```

↓

Alto.

---

# 18. Novelty

Evitar.

```text
Revenue by Country
```

si ya existe.

---

# 19. Confidence

Basada en:

```text
Semantic Match

Historical Success

Genome Similarity
```

---

# 20. Genome Integration

Dashboard Genome ayuda.

---

Ejemplo.

```text
Revenue Trend
```

---

Genome indica.

```text
Normalmente aparece con:

Revenue KPI

Revenue Country

Forecast
```

---

# 21. Insight Driven Recommendations

Insight Engine detecta.

```text
Revenue Concentration
```

---

Recomendar.

```text
Revenue by Product
```

---

para investigar causa.

---

# 22. Root Cause Suggestions

Concepto importante.

---

Insight:

```text
Revenue cayó 20%
```

---

SQLviz recomienda.

```text
Revenue by Product

Revenue by Country

Revenue by Customer
```

---

# 23. Query Recommendations

Generar SQL nuevo.

---

Entrada.

```sql
GROUP BY country
```

---

Sugerencia.

```sql
GROUP BY product
```

---

# 24. Rewrite Engine Integration

Reutilizar.

```text
Query Rewrite Engine
```

---

para generar consultas candidatas.

---

# 25. Dashboard Recommendations

Existe.

```text
Revenue Dashboard
```

---

Sugerir.

```text
Profit Dashboard
```

---

# 26. Filter Recommendations

Ejemplo.

```text
Country Filter
```

utilizado frecuentemente.

---

Recomendar.

```text
Product Filter
```

---

# 27. Recommendation Registry

```sql
CREATE TABLE recommendations (

    recommendation_id VARCHAR,

    recommendation_type VARCHAR,

    title VARCHAR,

    score DOUBLE,

    confidence DOUBLE
);
```

---

# 28. Recommendation Graph

Representación.

```text
Revenue
   │
   ├── Profit
   │
   ├── Margin
   │
   ├── Orders
   │
   └── Customers
```

---

# 29. Learning Integration

Registrar.

```text
Recommendation Accepted

Recommendation Ignored

Recommendation Rejected
```

---

# 30. Feedback Table

```sql
CREATE TABLE recommendation_feedback (

    recommendation_id VARCHAR,

    action VARCHAR,

    created_at TIMESTAMP
);
```

---

# 31. Recommendation Success Rate

```python
success_rate =
accepted
/
shown
```

---

# 32. Recommendation Ranking

SQLviz puede generar:

```text
100 recomendaciones
```

---

Mostrar:

```text
Top 5
```

---

# 33. Ranking Formula

```python
rank =
(
 score * 0.5
 +
 success_rate * 0.3
 +
 confidence * 0.2
)
```

---

# 34. Explainability

Toda recomendación debe explicarse.

---

Ejemplo.

```json
{
  "recommendation":"Revenue by Product",
  "reason":"Missing perspective",
  "confidence":0.94
}
```

---

# 35. Recommendation API

```python
GET /recommendations
```

---

```python
GET /recommendations/top
```

---

```python
POST /recommendations/generate
```

---

# 36. Recommendation Cache

```sql
CREATE TABLE recommendation_cache (

    cache_key VARCHAR,

    recommendations JSON
);
```

---

# 37. Recommendation Categories

Nivel usuario.

```text
Next Query
```

---

Nivel dashboard.

```text
Next Panel
```

---

Nivel negocio.

```text
Next Analysis
```

---

# 38. El Salto Conceptual

La mayoría de herramientas hacen:

```text
Pregunta
↓
Respuesta
```

---

SQLviz debe hacer:

```text
Pregunta
↓
Respuesta
↓
Siguiente Pregunta Relevante
```

---

# 39. Camino hacia Autonomous BI

Recommendation Engine introduce una capacidad nueva:

```text
Curiosidad Analítica
```

---

Porque comienza a explorar el espacio de análisis alrededor de una consulta.

---

# 40. Principio Fundamental

El Recommendation Engine transforma análisis estáticos en exploración guiada.

Es la capa que ayuda al usuario a descubrir qué debería analizar después, utilizando todo el conocimiento generado por los motores semánticos, métricos, dimensionales y de insights.

Sin él, SQLviz responde preguntas.

Con él, SQLviz ayuda a formular las siguientes.
