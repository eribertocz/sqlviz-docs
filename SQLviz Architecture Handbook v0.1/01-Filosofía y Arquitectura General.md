# SQLviz Architecture Handbook

## Capítulo 1 — Filosofía y Arquitectura General

Versión: 0.1

---

# 1. Introducción

SQLviz nace de una observación simple:

La mayoría de herramientas BI obligan al usuario a tomar decisiones que realmente no quiere tomar.

Un usuario normalmente desea responder preguntas de negocio:

* ¿Cómo evolucionan las ventas?
* ¿Qué región vende más?
* ¿Qué productos tienen mejor rendimiento?
* ¿Qué clientes están abandonando?

Sin embargo las herramientas actuales obligan a decidir:

* Tipo de gráfico
* Colores
* Layout
* Filtros
* Interacciones
* Relaciones entre paneles
* Formatos

SQLviz elimina esas decisiones.

El usuario únicamente escribe SQL.

Todo lo demás es inferido.

---

# 2. Principio Fundamental

SQLviz se basa en una idea radical:

> SQL contiene mucha más información semántica de la que la industria BI aprovecha actualmente.

Cuando un usuario escribe:

```sql
SELECT
    month,
    SUM(revenue)
FROM sales
GROUP BY month
```

No solamente está definiendo datos.

Está comunicando:

* Existe una dimensión temporal
* Existe una métrica agregada
* Desea analizar evolución
* Desea comparar períodos
* Existe una tendencia

La mayoría de herramientas ignoran esta información.

SQLviz la convierte en conocimiento.

---

# 3. Objetivo de Largo Plazo

El objetivo no es construir un generador automático de gráficos.

El objetivo es construir un sistema de BI autónomo.

Diferencia:

Generador de gráficos:

```text
SQL
 ↓
Chart
```

SQLviz:

```text
SQL
 ↓
Intención
 ↓
Dashboard
 ↓
Insights
```

La diferencia es enorme.

---

# 4. Definición de Autonomous BI

Autonomous BI significa:

El usuario escribe SQL.

El sistema:

* Entiende la intención
* Construye visualizaciones
* Diseña dashboard
* Genera filtros
* Relaciona paneles
* Aprende preferencias
* Mejora continuamente

Sin configuración manual.

---

# 5. Arquitectura de Alto Nivel

```text
                    SQL
                     │
                     ▼
          Query Fingerprinting
                     │
                     ▼
           Feature Extraction
                     │
                     ▼
              Intent Engine
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
  Chart Engine  Filter Engine Relation Engine
        │            │            │
        └────────────┼────────────┘
                     ▼
              Layout Engine
                     ▼
           Dashboard Composer
                     ▼
                 Dashboard
                     ▼
             Learning Engine
                     ▼
               Brain.duckdb
```

---

# 6. Componentes Principales

SQLviz está compuesto por motores especializados.

Cada motor resuelve un problema específico.

---

## Query Fingerprinting Engine

Responsabilidad:

Transformar SQL en patrones reutilizables.

Ejemplo:

```sql
SELECT month,
       SUM(revenue)
FROM sales
GROUP BY month
```

y

```sql
SELECT fecha,
       SUM(ventas)
FROM fact_ventas
GROUP BY fecha
```

representan exactamente el mismo patrón.

Fingerprint:

```text
TIME_SERIES_AGGREGATION
```

---

## Feature Extraction Engine

Responsabilidad:

Extraer señales del SQL y de los datos.

Produce:

```python
FeatureVector
```

utilizado por todos los motores posteriores.

---

## Intent Engine

Responsabilidad:

Determinar qué está intentando hacer el usuario.

Ejemplo:

```sql
GROUP BY month
```

puede sugerir:

```python
trend = 0.92
comparison = 0.18
ranking = 0.02
```

---

## Chart Engine

Responsabilidad:

Rankear visualizaciones posibles.

Nunca toma decisiones binarias.

Siempre produce probabilidades.

Ejemplo:

```python
[
    ("line", 0.93),
    ("area", 0.81),
    ("bar", 0.55)
]
```

---

## Filter Engine

Responsabilidad:

Descubrir filtros implícitos.

No requiere:

```sql
WHERE country = $country
```

Detecta automáticamente:

```text
country
region
category
date
```

como candidatos.

---

## Relation Engine

Responsabilidad:

Descubrir relaciones entre paneles.

Ejemplo:

Panel A:

```sql
GROUP BY region
```

Panel B:

```sql
GROUP BY region, product
```

Relación detectada:

```text
region
```

Cross filtering automático.

---

## Layout Engine

Responsabilidad:

Diseñar el dashboard.

Decide:

* posición
* tamaño
* agrupación
* jerarquía visual

---

## Learning Engine

Responsabilidad:

Aprender continuamente.

Ejemplo:

SQLviz propone:

```text
Line Chart
```

Usuario cambia a:

```text
Area Chart
```

El sistema registra:

```text
fingerprint
preferencia
contexto
resultado
```

y mejora futuras inferencias.

---

# 7. Filosofía de Diseño

SQLviz sigue cinco principios.

---

## Principio 1

SQL es la única interfaz.

No existe DSL adicional.

No existe configuración declarativa.

No existe YAML.

No existe JSON.

El usuario escribe SQL.

---

## Principio 2

La inferencia es probabilística.

Evitar reglas rígidas.

Incorrecto:

```python
IF date_column:
    chart = line
```

Correcto:

```python
line_score = 0.91
bar_score = 0.34
area_score = 0.72
```

---

## Principio 3

Aprendizaje local primero.

Todo funciona offline.

Sin servicios externos.

Sin dependencias SaaS.

---

## Principio 4

DuckDB como cerebro.

Toda la inteligencia persistente vive en:

```text
brain.duckdb
```

---

## Principio 5

Explicabilidad.

Toda inferencia debe poder explicarse.

Ejemplo:

```text
Line chart seleccionado porque:

- Existe dimensión temporal
- Trend score = 0.91
- 24 períodos detectados
- Ranking score bajo
```

---

# 8. Brain.duckdb

SQLviz mantiene dos bases.

Proyecto:

```text
empresa.sqlviz
```

Contiene:

* dashboards
* consultas
* paneles
* configuraciones

Cerebro:

```text
~/.sqlviz/brain.duckdb
```

Contiene:

* patrones
* aprendizaje
* feedback
* estadísticas
* embeddings
* inferencias

---

# 9. Ciclo de Vida de una Consulta

Cuando el usuario escribe SQL:

Paso 1

```text
Parse SQL
```

Paso 2

```text
Fingerprint
```

Paso 3

```text
Feature Extraction
```

Paso 4

```text
Intent Detection
```

Paso 5

```text
Chart Ranking
```

Paso 6

```text
Filter Discovery
```

Paso 7

```text
Relation Detection
```

Paso 8

```text
Layout Generation
```

Paso 9

```text
Dashboard Rendering
```

Paso 10

```text
Feedback Collection
```

Paso 11

```text
Learning
```

---

# 10. Roadmap de Evolución

## v0.1

Rule-Based Inference

Características:

* Chart inference
* Basic filters
* Basic titles

---

## v0.2

Probabilistic Inference

Características:

* Intent Engine
* Scoring Functions
* Confidence Scores

---

## v0.3

Local Learning

Características:

* Feedback Loop
* Pattern Learning
* Fingerprint Memory

---

## v0.4

Community Intelligence

Características:

* Shared Pattern Library
* Anonymous Contributions

---

## v0.5

Autonomous BI

Características:

* Dashboard Generation
* Automatic Insights
* Intent-Aware Composition

---

# Conclusión

SQLviz no es un generador de dashboards.

SQLviz es un sistema de inferencia.

El dashboard es simplemente la representación visual de una intención detectada a partir del SQL.

La misión del proyecto es transformar SQL en conocimiento sin requerir configuración manual.

Ese principio guía todas las decisiones arquitectónicas del sistema.
