# SQLviz Architecture Handbook

## Capítulo 17 — Insight Engine

Versión: 0.1

---

# 1. Introducción

Hasta este punto SQLviz puede:

```text
Entender SQL

Entender Métricas

Entender Dimensiones

Entender Intenciones

Generar Dashboards

Aprender del Usuario
```

---

Pero todavía existe una diferencia enorme entre:

```text
Dashboard
```

y

```text
Análisis
```

---

Un dashboard muestra datos.

Un analista explica lo que significan.

---

# 2. Objetivo

Transformar:

```text
Datos
```

en:

```text
Hallazgos
```

---

Sin prompts.

Sin LLMs.

Sin IA generativa.

---

Solo:

```text
Estadística
Semántica
Heurísticas
Knowledge Graph
```

---

# 3. Filosofía

El usuario no debería tener que descubrir todo manualmente.

---

SQLviz debe responder:

```text
¿Qué es importante aquí?
```

---

# 4. Arquitectura

```text
Query Result
      │
      ▼
Feature Extraction
      │
      ▼
Insight Detectors
      │
      ▼
Insight Ranking
      │
      ▼
Insight Registry
      │
      ▼
Dashboard Insights
```

---

# 5. ¿Qué es un Insight?

Modelo.

```python
class Insight:

    insight_id: str

    insight_type: str

    title: str

    description: str

    confidence: float

    impact_score: float

    evidence: dict
```

---

# 6. Categorías

SQLviz define:

```text
Trend

Change

Anomaly

Outlier

Pareto

Concentration

Growth

Decline

Seasonality

Forecast

Correlation

Ranking

Distribution

Volatility
```

---

# 7. Insight Registry

DuckDB.

```sql
CREATE TABLE insights (

    insight_id VARCHAR,

    insight_type VARCHAR,

    title VARCHAR,

    description TEXT,

    confidence DOUBLE,

    impact_score DOUBLE,

    created_at TIMESTAMP
);
```

---

# 8. Trend Detector

Ejemplo.

```text
Revenue
```

últimos 12 meses.

---

Detectar:

```text
Slope > 0
```

↓

Insight.

```text
Revenue muestra tendencia creciente.
```

---

# 9. Trend Score

```python
trend_score =
(
 slope_strength * 0.5
 +
 r_squared * 0.5
)
```

---

# 10. Growth Detector

Ejemplo.

```text
Revenue
```

---

Calcular:

```python
growth =
(
 current
 -
 previous
)
/
previous
```

---

Generar.

```text
Revenue creció 14.2%
```

---

# 11. Decline Detector

Misma lógica.

---

Pero:

```text
growth < 0
```

---

# 12. Anomaly Detector

Método inicial.

```python
z_score
```

---

Ejemplo.

```text
Revenue

100
101
102
98
99
450
```

↓

```text
Anomalía detectada
```

---

# 13. Anomaly Model

```python
if abs(zscore) > 3:
    anomaly
```

---

# 14. Outlier Detector

Para categorías.

---

Ejemplo.

```text
Country

A 10
B 12
C 9
D 200
```

↓

```text
Country D es outlier.
```

---

# 15. Pareto Detector

Uno de los más importantes.

---

Ejemplo.

```text
Revenue by Product
```

---

Ordenar.

↓

Acumulado.

↓

Detectar.

```text
20% productos generan 80% revenue.
```

---

# 16. Concentration Detector

Ejemplo.

```text
Revenue by Country
```

↓

```text
Brasil = 42%
```

↓

Insight.

```text
Revenue altamente concentrado.
```

---

# 17. Concentration Score

Basado en:

```text
Herfindahl Index
```

---

o

```text
Gini Coefficient
```

---

# 18. Seasonality Detector

Series temporales.

---

Detectar.

```text
Patrón repetitivo
```

---

Ejemplo.

```text
Diciembre
```

siempre alto.

---

# 19. Volatility Detector

Calcular.

```python
stddev
```

---

Detectar.

```text
Alta volatilidad.
```

---

# 20. Ranking Detector

Ejemplo.

```text
Top Country

Bottom Country
```

---

Generar.

```text
Brasil lidera revenue.
```

---

# 21. Share Detector

Ejemplo.

```text
Country Share
```

↓

```text
Brasil aporta 42%.
```

---

# 22. Change Point Detector

Detectar cambios bruscos.

---

Método inicial.

```python
rolling_mean
```

---

Luego.

```text
CUSUM
Bayesian Change Point
```

---

# 23. Correlation Detector

Ejemplo.

```text
Revenue

Orders
```

---

Calcular.

```python
pearson
```

---

Insight.

```text
Revenue altamente correlacionado con Orders.
```

---

# 24. Correlation Registry

```sql
CREATE TABLE metric_correlations (

    metric_a VARCHAR,

    metric_b VARCHAR,

    correlation DOUBLE
);
```

---

# 25. Insight Evidence

Todo insight debe tener evidencia.

---

Ejemplo.

```json
{
  "metric":"Revenue",
  "growth":14.2,
  "period":"May 2026"
}
```

---

# 26. Insight Confidence

```python
confidence =
(
 statistical_strength * 0.6
 +
 sample_quality * 0.4
)
```

---

# 27. Impact Score

Pregunta:

```text
¿Qué tan importante es?
```

---

Ejemplo.

```text
Revenue +20%
```

↓

Impacto alto.

---

```text
Revenue +0.3%
```

↓

Impacto bajo.

---

# 28. Impact Formula

```python
impact =
(
 magnitude * 0.5
 +
 business_importance * 0.5
)
```

---

# 29. Insight Ranking

SQLviz puede encontrar:

```text
200 insights
```

---

Mostrar:

```text
Top 10
```

---

# 30. Ranking Formula

```python
rank =
(
 confidence * 0.4
 +
 impact * 0.4
 +
 novelty * 0.2
)
```

---

# 31. Novelty

Evitar:

```text
Mismo insight
```

cada vez.

---

# 32. Insight Families

Ejemplo.

```text
Revenue
 ├── Growth
 ├── Trend
 ├── Forecast
 ├── Concentration
 └── Anomaly
```

---

# 33. Insight Graph

```text
Revenue
     │
     ├── Growth
     │
     ├── Trend
     │
     ├── Forecast
     │
     └── Anomaly
```

---

# 34. Semantic Insights

Usar Semantic Engine.

---

Detectar.

```text
Revenue
Profit
Margin
```

---

Insight.

```text
Revenue crece más rápido que Profit.
```

---

# 35. Multi-Panel Insights

No limitarse a un panel.

---

Ejemplo.

Panel A.

```text
Revenue
```

---

Panel B.

```text
Profit
```

---

Insight combinado.

```text
Revenue crece pero margen disminuye.
```

---

# 36. Cross-Dashboard Insights

Futuro.

---

Comparar dashboards.

---

Detectar patrones globales.

---

# 37. Insight Templates

DuckDB.

```sql
CREATE TABLE insight_templates (

    insight_type VARCHAR,

    template TEXT
);
```

---

Ejemplo.

```text
"{metric} creció {growth}% respecto al período anterior."
```

---

# 38. Explainability

Obligatorio.

---

Ejemplo.

```json
{
  "insight":"Revenue Growth",
  "reason":"14.2% increase",
  "confidence":0.95
}
```

---

# 39. Learning Integration

Registrar.

```text
Insight leído

Insight ignorado

Insight guardado
```

---

# 40. Insight Feedback

```sql
CREATE TABLE insight_feedback (

    insight_id VARCHAR,

    action VARCHAR
);
```

---

# 41. Insight API

```python
GET /insights
```

---

```python
GET /insights/top
```

---

```python
POST /insights/generate
```

---

# 42. Dashboard Composer Integration

Composer pregunta:

```text
¿Qué es importante aquí?
```

---

Insight Engine responde.

```text
Top Growth

Top Risk

Top Opportunity
```

---

# 43. La Idea Más Importante

La mayoría de herramientas BI terminan en:

```text
Chart
```

---

SQLviz debe continuar hasta:

```text
Insight
```

---

Porque el usuario realmente quiere:

```text
Entender
```

no simplemente:

```text
Ver
```

---

# 44. El Salto Estratégico

Competidores:

```text
SQL
 ↓
Chart
```

---

SQLviz:

```text
SQL
 ↓
Chart
 ↓
Insight
```

---

# 45. Principio Fundamental

El Insight Engine transforma datos en observaciones accionables.

Es la capa que detecta automáticamente patrones, anomalías, cambios, concentraciones y oportunidades sin que el usuario tenga que buscarlos manualmente.

Sin él, SQLviz visualiza información.

Con él, SQLviz empieza a comportarse como un analista.