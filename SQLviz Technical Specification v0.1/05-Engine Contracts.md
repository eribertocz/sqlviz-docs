# SQLviz Technical Specification v1

## Capítulo 05 — Engine Contracts

Versión 0.1

---

# 1. Objetivo

Definir la interfaz oficial que implementarán todos los engines de SQLviz.

---

Sin excepciones.

---

Todos los motores deben comportarse igual.

---

# 2. Problema

Si cada engine define su propia interfaz:

```python
FeatureEngine.run()

SemanticEngine.execute()

IntentEngine.process()

ChartEngine.infer()
```

---

terminamos con:

```text
Inconsistencia

Testing complejo

Acoplamiento
```

---

# 3. Solución

Contrato único.

---

# 4. Base Engine

```python
from abc import ABC
from abc import abstractmethod

class Engine(ABC):

    @abstractmethod
    def execute(
        self,
        context: RuntimeContext
    ) -> RuntimeContext:
        pass
```

---

# 5. Regla Fundamental

Todo engine:

```text
Recibe RuntimeContext

Devuelve RuntimeContext
```

---

Siempre.

---

# 6. Beneficio

Pipeline uniforme.

---

# 7. Ejemplo

```python
context = parser.execute(context)

context = feature.execute(context)

context = semantic.execute(context)

context = intent.execute(context)

context = chart.execute(context)
```

---

# 8. Feature Engine Contract

```python
class FeatureEngine(Engine):

    def execute(
        self,
        context: RuntimeContext
    ) -> RuntimeContext:
        ...
```

---

# 9. Semantic Engine Contract

```python
class SemanticEngine(Engine):

    def execute(
        self,
        context: RuntimeContext
    ) -> RuntimeContext:
        ...
```

---

# 10. Intent Engine Contract

```python
class IntentEngine(Engine):

    def execute(
        self,
        context: RuntimeContext
    ) -> RuntimeContext:
        ...
```

---

# 11. Chart Engine Contract

```python
class ChartEngine(Engine):

    def execute(
        self,
        context: RuntimeContext
    ) -> RuntimeContext:
        ...
```

---

# 12. Layout Engine Contract

```python
class LayoutEngine(Engine):

    def execute(
        self,
        context: RuntimeContext
    ) -> RuntimeContext:
        ...
```

---

# 13. Dashboard Engine Contract

```python
class DashboardComposer(Engine):

    def execute(
        self,
        context: RuntimeContext
    ) -> RuntimeContext:
        ...
```

---

# 14. Engine Responsibilities

Cada engine tiene dueño único.

---

# 15. Feature Engine

Produce.

```text
Features
```

---

# 16. Semantic Engine

Produce.

```text
SemanticTags
```

---

# 17. Intent Engine

Produce.

```text
Intents
```

---

# 18. Metric Engine

Produce.

```text
Metrics
```

---

# 19. Dimension Engine

Produce.

```text
Dimensions
```

---

# 20. Chart Engine

Produce.

```text
Charts
```

---

# 21. Layout Engine

Produce.

```text
Panel Layout
```

---

# 22. Dashboard Composer

Produce.

```text
Dashboard
```

---

# 23. Ownership Rule

Un objeto tiene un único creador.

---

Ejemplo.

```text
Chart
```

↓

```text
Chart Engine
```

---

Nunca.

```text
Semantic Engine
```

---

# 24. Engine Metadata

Todos los engines exponen.

```python
@property
def name(self):
    ...
```

---

# 25. Ejemplo

```python
@property
def name(self):
    return "semantic_engine"
```

---

# 26. Engine Version

Todos los engines exponen.

```python
@property
def version(self):
    ...
```

---

# 27. Ejemplo

```python
@property
def version(self):
    return "1.0.0"
```

---

# 28. Explainability Contract

Todos los engines deben explicar.

---

Siempre.

---

# 29. Malo

```python
chart = "line"
```

---

# 30. Bueno

```python
chart = "line"

explanation = [
   ...
]
```

---

# 31. Engine Output Rule

Todo resultado importante debe incluir.

```text
Confidence
```

---

# 32. Ejemplo

```python
Intent(
    name="trend",
    score=0.93
)
```

---

# 33. Engine Confidence Contract

Prohibido.

```python
score = None
```

---

# 34. Siempre

```python
0.0
```

a

```python
1.0
```

---

# 35. Engine Events

Todo engine publica eventos.

---

# 36. Ejemplo

Feature Engine.

---

Publica.

```text
FEATURES_EXTRACTED
```

---

# 37. Semantic Engine

Publica.

```text
SEMANTICS_INFERRED
```

---

# 38. Intent Engine

Publica.

```text
INTENTS_INFERRED
```

---

# 39. Chart Engine

Publica.

```text
CHARTS_INFERRED
```

---

# 40. Error Contract

Motores no lanzan excepciones funcionales.

---

# 41. Malo

```python
raise Exception()
```

---

# 42. Bueno

```python
context.errors.append(...)
```

---

# 43. Warning Contract

Baja confianza.

↓

Warning.

---

No excepción.

---

# 44. Ejemplo

```text
LOW_CONFIDENCE_INTENT
```

---

# 45. Engine Dependencies

Permitidas.

```text
Feature
 ↓
Semantic
 ↓
Intent
 ↓
Chart
 ↓
Layout
```

---

# 46. Dependencias Prohibidas

Nunca.

```text
Chart
 ↓
Feature
```

---

# 47. Engine Pipeline

V1.

```python
PIPELINE = [

 ParserEngine(),

 FeatureEngine(),

 SemanticEngine(),

 MetricEngine(),

 DimensionEngine(),

 IntentEngine(),

 ChartEngine(),

 LayoutEngine(),

 DashboardComposer()
]
```

---

# 48. Engine Registry

Centralizado.

---

# 49. Registry Model

```python
class EngineRegistry:

    engines: list[Engine]
```

---

# 50. Beneficio

Permite:

```text
Plugins

Testing

Custom Engines
```

---

# 51. Engine Configuration

Cada engine recibe.

```python
EngineConfig
```

---

# 52. Ejemplo

```python
class SemanticConfig:

    min_confidence: float
```

---

# 53. No Hardcoding Rule

Evitar.

```python
if score > 0.73:
```

---

Preferir.

```python
config.min_score
```

---

# 54. Determinism Rule

Muy importante.

---

Mismo input.

↓

Mismo output.

---

# 55. Why?

Testing.

---

Benchmarking.

---

Caching.

---

# 56. Stateless Rule

V1.

---

Los engines son stateless.

---

# 57. Malo

```python
self.previous_query
```

---

# 58. Bueno

```python
context.query
```

---

# 59. Benchmark Contract

Todo engine debe poder medir.

```python
execution_time_ms
```

---

# 60. Observability Contract

Todo engine reporta.

```text
Inicio

Fin

Duración
```

---

# 61. Plugin Compatibility

Plugins implementan.

```python
Engine
```

---

Sin privilegios especiales.

---

# 62. Engine Categories

```text
Core Engine

Plugin Engine

Experimental Engine
```

---

# 63. Core Engines

V1.

```text
Parser

Feature

Semantic

Metric

Dimension

Intent

Chart

Layout

Dashboard
```

---

# 64. Future Engines

```text
Learning

Genome

Recommendation

Autonomous Analysis
```

---

# 65. Testing Contract

Todo engine debe tener.

```text
Unit Tests

Corpus Tests

Benchmark Tests
```

---

# 66. Example Test

Input.

```sql
SELECT month,
       SUM(revenue)
FROM sales
GROUP BY month
```

---

Output.

```text
Trend
```

---

# 67. Engine Scorecard

Guardar.

```python
accuracy

latency

coverage
```

---

por engine.

---

# 68. Principio Fundamental

Los engines son las neuronas de SQLviz.

El RuntimeContext es el sistema nervioso.

Los Event Contracts son el sistema circulatorio.

Y el contrato común permite que todos los motores evolucionen independientemente sin romper el resto del sistema.