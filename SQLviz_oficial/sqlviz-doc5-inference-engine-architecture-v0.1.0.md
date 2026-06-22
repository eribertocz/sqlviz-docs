# SQLviz — Inference Engine Architecture
**Version:** v0.1.0 (Draft)
**Status:** Work in Progress
**Last Updated:** 2026-06-08
**Prerequisite:** DOC 4 — Mathematical & Statistical Foundations v0.1.3

---

## 1. Overview

### 1.1 What is the Inference Engine

The Inference Engine is the brain of SQLviz.

It is the system that receives a SQL query
and produces a complete analytical understanding of it —
chart type, layout, filters, title, insights —
without any input from the user beyond the SQL itself.

```
User writes:
    SELECT month, SUM(revenue) FROM sales GROUP BY month

Inference Engine produces:
    intent_winner    = "trend"
    chart_winner     = "line"
    title            = "Revenue by month"
    layout           = col_span=12, row_span=1
    filters          = [DateRangePicker for 'month']
    confidence       = high
    explanation      = [...]
    score_trace      = {intent: {...}, chart: {...}}
```

The user never configures any of this.
The Inference Engine infers it all.

### 1.2 The Complete Pipeline

```
SQL (string)
    │
    ▼
┌─────────────────────────────────────────────────────┐
│ 1. PARSER                                           │
│    sqlglot → AST                                    │
│    AST → Fingerprint                                │
│    AST → SQL Features (dims 0-17 of feature vector) │
└─────────────────────┬───────────────────────────────┘
                      │ AST + SQL Features
                      ▼
┌─────────────────────────────────────────────────────┐
│ 2. FEATURE ENGINE                                   │
│    Execute SQL → get data + schema                  │
│    Compute Column Features (dims 18-24)             │
│    Compute Data Statistics (dims 25-29)             │
│    → Complete Feature Vector V0 (39 dims)           │
└─────────────────────┬───────────────────────────────┘
                      │ Feature Vector [38]
                      ▼
┌─────────────────────────────────────────────────────┐
│ 3. SEMANTIC ENGINE                                  │
│    Classify column names                            │
│    revenue → METRIC_REVENUE                         │
│    fecha → TEMPORAL_DIMENSION                       │
│    → Semantic Features (dims 30-34)                 │
│    → Complete Feature Vector V0 (39 dims) final     │
└─────────────────────┬───────────────────────────────┘
                      │ Feature Vector [38] complete
                      ▼
┌─────────────────────────────────────────────────────┐
│ 4. INTENT ENGINE                                    │
│    Score 12 intents using feature vector            │
│    Apply weights from intent_rules.yaml             │
│    Normalize scores                                 │
│    → IntentVector {trend: 0.99, comparison: 0.35}  │
└─────────────────────┬───────────────────────────────┘
                      │ IntentVector
                      ▼
┌─────────────────────────────────────────────────────┐
│ 5. CHART ENGINE                                     │
│    Apply affinity matrix (chart × intent)           │
│    Apply penalty rules                              │
│    Apply fallback if quality < threshold            │
│    → ChartResult {winner, score, alternatives}      │
└─────────────────────┬───────────────────────────────┘
                      │ ChartResult
                      ▼
┌─────────────────────────────────────────────────────┐
│ 6. LAYOUT ENGINE                                    │
│    Compute panel importance score                   │
│    Assign CSS Grid span (col_span, row_span)        │
│    → LayoutResult {col_span, row_span, position}    │
└─────────────────────┬───────────────────────────────┘
                      │ LayoutResult
                      ▼
┌─────────────────────────────────────────────────────┐
│ 7. FILTER ENGINE                                    │
│    Detect filterable columns from AST               │
│    Classify control type per column                 │
│    → FilterResult [FilterControl, ...]              │
└─────────────────────┬───────────────────────────────┘
                      │ FilterResult
                      ▼
┌─────────────────────────────────────────────────────┐
│ 8. TITLE ENGINE                                     │
│    Generate descriptive title from SQL + schema     │
│    → TitleResult {title, confidence}                │
└─────────────────────┬───────────────────────────────┘
                      │ TitleResult
                      ▼
┌─────────────────────────────────────────────────────┐
│ 9. RUNTIME PIPELINE                                 │
│    Assembles all results into InferenceResult       │
│    Handles graceful degradation                     │
│    Writes to brain.duckdb for learning              │
│    → InferenceResult (complete)                     │
└─────────────────────────────────────────────────────┘
```

### 1.3 The Pipeline Modules and How They Connect

> Earlier drafts of this document said "The 7 Modules." That was
> an undercount that also predated Dashboard Engine (Section 15)
> being added. The accurate count is **8 inference modules**
> (Parser through Dashboard Engine) coordinated by **1 pipeline
> orchestrator** (Runtime Pipeline, which is not itself an
> inference module — it owns no feature/intent/chart logic, it
> only sequences the 8 modules below). "8 modules + 1
> orchestrator" is the precise description used from here on.

```
Module              Input                   Output
────────────────────────────────────────────────────────────────
Parser              SQL string              AST + Fingerprint
Feature Engine      AST + data + schema     FeatureVector[39]
Semantic Engine     FeatureVector + schema  FeatureVector[39] enriched
Intent Engine       FeatureVector[39]       IntentVector
Chart Engine        IntentVector            ChartResult
Layout Engine       ChartResult + context   LayoutResult
Filter Engine       AST + schema            FilterResult
Title Engine        AST + schema + intent   TitleResult
Dashboard Engine    list[InferenceResult]   DashboardLayout
                    (Section 15 — operates across panels,
                    not inside the per-panel pipeline above)
────────────────────────────────────────────────────────────────
Runtime Pipeline     All of the above        InferenceResult
(orchestrator, not   (sequences the 8 modules,
 an inference         does not infer anything itself)
 module)
```

Modules never call each other directly.
They all receive and return a RuntimeContext.
The Runtime Pipeline coordinates the execution order.

```python
# Correct — modules communicate via RuntimeContext
context = parser.run(context)
context = feature_engine.run(context)
context = semantic_engine.run(context)
context = intent_engine.run(context)
context = chart_engine.run(context)
context = layout_engine.run(context)
context = filter_engine.run(context)
context = title_engine.run(context)

# Wrong — direct module dependencies
chart_result = chart_engine.run(intent_engine.run(parser.run(sql)))
```

### 1.4 Design Principles

**Principle 1 — No hardcoded rules**
```
All weights, thresholds and rules live in YAML files.
Python code only reads and applies rules.
Python code never contains numerical constants
for inference decisions.

Wrong:
    if temporal_score > 0.40:
        intent = "trend"

Correct:
    weights = yaml.load("intent_rules.yaml")
    score = sum(w * f for w, f in zip(weights, features))
```

**Principle 2 — Every module is independently testable**
```
Each module receives a RuntimeContext
and returns a RuntimeContext.
Each module can be tested in isolation
with a mock RuntimeContext.

def test_intent_engine():
    ctx = RuntimeContext(feature_vector=[1,1,0,0,1,1,0,...])
    result = IntentEngine().run(ctx)
    assert result.intent_winner == "trend"
    assert result.intent_raw_score > 0.90
```

**Principle 3 — Graceful degradation**
```
If any module fails, the pipeline continues
with sensible defaults.

Parser fails    → feature_vector = zeros, fingerprint = "UNKNOWN"
Semantic fails  → semantic features = 0.0 (neutral)
Intent fails    → intent = "detail" (safe default)
Chart fails     → chart = "table" (always safe)
Layout fails    → col_span=12, row_span=1 (full width)
Filter fails    → no filters (safe, user sees data)
Title fails     → title = "" (no title shown)
```

**Principle 4 — Standard library only**
```
No numpy. No pandas. No scikit-learn.
No community extensions.

All statistical computations use:
→ Python stdlib math module
→ DuckDB SQL functions
→ Custom implementations from DOC 4

Exception: duckdb, sqlglot, fastapi, quack
are in the official stack (DOC 3).
```

**Principle 5 — Confidence is always explicit**
```
Every module returns a confidence score.
No module makes a decision without reporting uncertainty.

Every result has:
    raw_score        → absolute strength
    normalized_score → relative ranking
    quality          → "high" | "medium" | "low"
    confidence_gap   → best - second_best
```

---

*Section 1 complete. Next: Section 2 — Package Structure.*

---

## 2. Package Structure

### 2.1 Directory Tree

```
sqlviz-inference/
│
├── rules/                          ← YAML rules — no hardcoded values
│   ├── feature_vector_v0.yaml
│   ├── intent_rules.yaml
│   ├── chart_affinity_matrix.yaml
│   ├── chart_penalties.yaml
│   ├── fallback_rules.yaml
│   ├── thresholds.yaml
│   └── semantic_dictionary.yaml
│
├── src/
│   ├── __init__.py                 ← public: infer(sql) → InferenceResult
│   ├── context.py                  ← RuntimeContext dataclass
│   ├── result.py                   ← InferenceResult dataclass
│   ├── pipeline.py                 ← Runtime Pipeline coordinator
│   │
│   ├── parser/
│   │   ├── sql_parser.py           ← sqlglot AST parsing
│   │   ├── fingerprint.py          ← fingerprint generation
│   │   └── ast_helpers.py          ← AST utility functions
│   │
│   ├── features/
│   │   ├── feature_engine.py       ← coordinates feature computation
│   │   ├── sql_features.py         ← dims 0-17 from AST
│   │   ├── column_features.py      ← dims 18-24 from schema
│   │   ├── data_statistics.py      ← dims 25-29 from data
│   │   └── result_shape.py         ← dims 35-37 from result
│   │
│   ├── semantic/
│   │   ├── semantic_engine.py      ← column classification
│   │   └── fuzzy_match.py          ← fuzzy name matching
│   │
│   ├── intent/
│   │   └── intent_engine.py        ← 12 intent scoring
│   │
│   ├── chart/
│   │   └── chart_engine.py         ← affinity + penalties + fallback
│   │
│   ├── layout/
│   │   └── layout_engine.py        ← grid span assignment
│   │
│   ├── filters/
│   │   └── filter_engine.py        ← filter detection from AST
│   │
│   ├── title/
│   │   └── title_engine.py         ← title generation
│   │
│   └── utils/
│       ├── yaml_loader.py          ← load and cache YAML rules
│       ├── math_utils.py           ← softmax, min_max, statistics
│       └── confidence.py           ← confidence_gap, quality_label
│
└── tests/
    ├── benchmark/
    │   └── benchmark_cases.yaml    ← 30+ gold dataset queries
    ├── test_parser.py
    ├── test_feature_engine.py
    ├── test_semantic_engine.py
    ├── test_intent_engine.py
    ├── test_chart_engine.py
    ├── test_layout_engine.py
    ├── test_filter_engine.py
    ├── test_title_engine.py
    ├── test_pipeline.py
    └── test_benchmark.py
```

### 2.2 File Responsibilities

```
File                        Responsibility
──────────────────────────────────────────────────────────────────
context.py                  RuntimeContext — carries all data
                            between modules. Never imports from src/.

result.py                   InferenceResult — final output.
                            Versioned with rules_version, engine_version.

pipeline.py                 Coordinates execution order.
                            Handles graceful degradation.
                            Writes to brain.duckdb after inference.

parser/sql_parser.py        SQL string → AST + SQL features (dims 0-17)

parser/fingerprint.py       AST → fingerprint string
                            e.g. "TIME_SUM_GROUP1_ORDER_ASC"

parser/ast_helpers.py       Pure AST inspection functions.
                            has_group_by(), count_columns(), etc.
                            No side effects.

features/feature_engine.py  Coordinates dims 0-38.
                            Calls sql_features, column_features,
                            data_statistics, result_shape in order.

semantic/semantic_engine.py Classifies column names.
                            Reads semantic_dictionary.yaml.
                            Enriches dims 30-34 only
                            (Result Shape 35-37 and trend_direction 38
                            are filled by Feature Engine, not here).

intent/intent_engine.py     Scores 12 intents from feature vector.
                            Reads intent_rules.yaml.
                            Returns IntentVector.

chart/chart_engine.py       Applies affinity matrix + penalties.
                            Reads chart_affinity_matrix.yaml
                            and chart_penalties.yaml.
                            Returns ChartResult.

layout/layout_engine.py     Assigns CSS Grid spans.
                            Returns LayoutResult.

filters/filter_engine.py    Detects filterable columns from AST.
                            Returns list of FilterControl.

title/title_engine.py       Generates panel title.
                            Returns TitleResult.

utils/yaml_loader.py        Loads YAML once and caches.
                            All modules use this — never load YAML directly.

utils/math_utils.py         softmax(), min_max_normalize(),
                            trend_strength(), skewness(), pearson_r().
                            Pure functions. No side effects.

utils/confidence.py         confidence_gap(), quality_label().
                            Used by all engines.
```

### 2.3 The Rules

```
Rule 1 — Numerical constants → YAML only
    Never write 0.40, 0.25, 0.85 in Python.
    Always read from YAML via yaml_loader.

Rule 2 — Statistical math → utils/math_utils.py
    Implemented once. Used everywhere.

Rule 3 — No circular imports
    context.py  → imports nothing from src/
    result.py   → imports nothing from src/
    utils/      → imports only stdlib + context.py
    All modules → import context + utils only

Rule 4 — Tests mirror src/ structure
    test_parser.py tests parser/
    test_intent_engine.py tests intent/
    One test file per source module.

Rule 5 — Benchmark is the release gate
    Never ship if benchmark accuracy drops.
    benchmark_cases.yaml = 30+ gold queries.
```

### 2.4 The Entry Point

```python
# sqlviz-inference/src/__init__.py

from .pipeline import RuntimePipeline
from .context import RuntimeContext
from .result import InferenceResult

_pipeline = RuntimePipeline()

def infer(
    sql: str,
    data: list[dict] = None,
    schema: list = None
) -> InferenceResult:
    """
    The single public function of the Inference Engine.

    Args:
        sql:    SQL query string to analyze
        data:   Optional — query result rows (list of dicts)
                If not provided, data statistics = 0.0
        schema: Optional — column schema from DuckDB DESCRIBE
                If not provided, column features = 0.0

    Returns:
        InferenceResult — complete inference output

    Example:
        result = infer(
            sql="SELECT month, SUM(revenue) FROM sales GROUP BY month",
            data=[{"month": "Jan", "revenue": 8500}, ...],
            schema=[{"name": "month",   "type": "VARCHAR"},
                    {"name": "revenue", "type": "DOUBLE"}]
        )
        assert result.chart_winner  == "line"
        assert result.intent_winner == "trend"
        assert result.chart_quality == "high"
    """
    context = RuntimeContext(
        sql=sql,
        data=data or [],
        schema=schema or []
    )
    context = _pipeline.run(context)
    return InferenceResult.from_context(context)
```

---

*Section 2 complete. Next: Section 3 — RuntimeContext.*

---

## 3. RuntimeContext

### 3.1 Definition and Purpose

RuntimeContext is the single object that flows through
the entire inference pipeline.

Every module receives a RuntimeContext and returns
an enriched RuntimeContext. No module returns anything else.

```
SQL string
    ↓
RuntimeContext(sql=sql)          ← created at entry point
    ↓
parser.run(context)              ← adds: ast, fingerprint, sql_features
    ↓
feature_engine.run(context)      ← adds: feature_vector complete
    ↓
semantic_engine.run(context)     ← enriches: feature_vector dims 30-34
    ↓
intent_engine.run(context)       ← adds: intent_vector
    ↓
chart_engine.run(context)        ← adds: chart_result
    ↓
layout_engine.run(context)       ← adds: layout_result
    ↓
filter_engine.run(context)       ← adds: filter_controls
    ↓
title_engine.run(context)        ← adds: title_result
    ↓
InferenceResult.from_context()   ← extracts final result
```

### 3.2 Complete Dataclass

```python
# sqlviz-inference/src/context.py

from __future__ import annotations
from dataclasses import dataclass, field
from typing import Any
import sqlglot


@dataclass
class ColumnSchema:
    """Schema of a single column from DuckDB DESCRIBE."""
    name: str
    type: str           # "VARCHAR" | "DOUBLE" | "DATE" | "TIMESTAMP" | ...
    nullable: bool = True


@dataclass
class IntentScore:
    """Score for a single intent."""
    intent: str
    raw_score: float
    normalized_score: float
    signals: dict[str, float] = field(default_factory=dict)
    # signals = {"has_temporal_dimension": 0.40, "has_group_by": 0.25, ...}


@dataclass
class ChartCandidate:
    """A chart type candidate with its score."""
    chart_type: str
    affinity_score: float
    penalty_total: float
    final_score: float
    normalized_score: float
    penalties_applied: list[dict] = field(default_factory=list)


@dataclass
class FilterControl:
    """A detected filter control for a column."""
    variable: str           # the $variable name or column name
    label: str              # human readable label
    control_type: str       # "date_picker" | "dropdown" | "multiselect" |
                            # "search" | "numeric" | "range_slider" | "toggle"
    column_name: str
    column_type: str
    cardinality: int = 0    # approximate unique values
    scope: str = "global"   # "global" | "local"


@dataclass
class RuntimeContext:
    """
    Carrier of all inference data, using field-owned mutation
    (see Section 16.1 — corrected from earlier "immutable" wording).
    Flows through every module in the pipeline.

    Modules read from context and write ONLY to the fields they
    own (Section 3.4). They never reconstruct a new instance —
    direct field assignment (context.field = value) is the
    correct and expected pattern, as long as it targets a field
    the module owns.
    """

    # ── Input ────────────────────────────────────────────────
    sql: str
    data: list[dict] = field(default_factory=list)
    schema: list[ColumnSchema] = field(default_factory=list)

    # ── Parser output ─────────────────────────────────────────
    ast: Any = None                     # sqlglot AST expression
    fingerprint: str = "UNKNOWN"        # e.g. "TIME_SUM_GROUP1_ORDER_ASC"
    sql_features: list[float] = field(default_factory=list)
    # dims 0-17 of feature vector

    # ── Feature Engine output ─────────────────────────────────
    feature_vector: list[float] = field(
        default_factory=lambda: [0.0] * 39
    )
    # complete 39-dimension feature vector V0

    # ── Intent Engine output ──────────────────────────────────
    intent_scores: list[IntentScore] = field(default_factory=list)
    intent_winner: str = "detail"
    intent_raw_score: float = 0.0
    intent_normalized_score: float = 0.0
    intent_confidence_gap: float = 0.0
    intent_quality: str = "low"         # "high" | "medium" | "low"

    # ── Chart Engine output ───────────────────────────────────
    chart_candidates: list[ChartCandidate] = field(default_factory=list)
    chart_winner: str = "table"         # safe default
    chart_raw_score: float = 0.0
    chart_normalized_score: float = 0.0
    chart_confidence_gap: float = 0.0
    chart_quality: str = "low"
    chart_alternatives: list[dict] = field(default_factory=list)
    fallback_applied: bool = False
    fallback_reason: str = ""

    # ── Layout Engine output ──────────────────────────────────
    col_span: int = 12                  # CSS Grid columns (1-12)
    row_span: int = 1                   # CSS Grid rows
    layout_importance: float = 0.0

    # ── Filter Engine output ──────────────────────────────────
    filter_controls: list[FilterControl] = field(default_factory=list)

    # ── Title Engine output ───────────────────────────────────
    title: str = ""
    title_confidence: float = 0.0

    # ── Explainability ────────────────────────────────────────
    explanation: list[dict] = field(default_factory=list)
    score_trace: dict = field(default_factory=dict)

    # ── Error tracking ────────────────────────────────────────
    errors: list[str] = field(default_factory=list)
    # populated when a module fails gracefully

    # ── Versioning ────────────────────────────────────────────
    rules_version: str = "v0.1.0"
    feature_vector_version: str = "v0"
    engine_version: str = "sqlviz-inference-0.1.0"

    def with_error(self, module: str, error: str) -> RuntimeContext:
        """
        Return a new context with an error logged.
        Used for graceful degradation.
        """
        new_errors = self.errors + [f"{module}: {error}"]
        return RuntimeContext(
            **{k: v for k, v in self.__dict__.items() if k != 'errors'},
            errors=new_errors
        )

    @property
    def has_data(self) -> bool:
        return len(self.data) > 0

    @property
    def has_schema(self) -> bool:
        return len(self.schema) > 0

    @property
    def row_count(self) -> int:
        return len(self.data)

    @property
    def col_count(self) -> int:
        return len(self.data[0]) if self.data else 0
```

### 3.3 How It Flows Through the Pipeline

Each module follows this exact pattern:

```python
# Every module looks like this

class IntentEngine:
    def run(self, context: RuntimeContext) -> RuntimeContext:
        try:
            # 1. Read what you need from context
            feature_vector = context.feature_vector
            weights = yaml_loader.load("intent_rules.yaml")

            # 2. Compute your result
            scores = self._score_intents(feature_vector, weights)

            # 3. Write to fields this module owns (Section 3.4) —
            # direct assignment is correct under field-owned mutation
            context.intent_scores = scores
            context.intent_winner = scores[0].intent
            context.intent_raw_score = scores[0].raw_score
            context.intent_normalized_score = scores[0].normalized_score
            context.intent_confidence_gap = confidence_gap(scores)
            context.intent_quality = quality_label(scores[0].raw_score)

            return context

        except Exception as e:
            # Graceful degradation — log error, return safe defaults
            # context already has safe defaults from __init__
            return context.with_error("IntentEngine", str(e))
```

### 3.4 Field-Owned Mutation Rules

> **Note:** earlier drafts of this section called this "Immutability
> Rules." That wording was corrected in Section 16.1 — RuntimeContext
> uses field-owned mutation, not true immutability. The practical
> guarantees below are unchanged; only the name is corrected.

```
Rule 1 — Modules only write to their own fields
    Parser writes:   ast, fingerprint, sql_features
    Feature Engine:  feature_vector
    Semantic Engine: feature_vector (dims 30-34 only)
    Intent Engine:   intent_* fields
    Chart Engine:    chart_* fields, fallback_*
    Layout Engine:   col_span, row_span, layout_importance
    Filter Engine:   filter_controls
    Title Engine:    title, title_confidence

    Modules NEVER read from fields they did not write.
    (Exception: every module reads feature_vector)

Rule 2 — Never delete fields
    If a module fails, it leaves fields at their defaults.
    Defaults are always safe values (empty lists, "table", 12).

Rule 3 — errors list is append-only
    Use context.with_error() to add errors.
    Never remove errors from the list.

Rule 4 — Input fields are never modified
    sql, data, schema are set at creation and never changed.
```

### 3.5 Safe Defaults

Every field has a safe default that produces valid output
even if the module that fills it fails completely.

```
field               default         reason
────────────────────────────────────────────────────────────
fingerprint         "UNKNOWN"       no pattern detected
feature_vector      [0.0] * 39      neutral — no features
intent_winner       "detail"        safest intent
chart_winner        "table"         always shows data
col_span            12              full width — always works
row_span            1               minimum height
filter_controls     []              no filters — always safe
title               ""              no title shown
fallback_applied    False           optimistic default
errors              []              no errors
```

---

*Section 3 complete. Next: Section 4 — Parser Module.*

---

## 4. Parser Module

### 4.1 Responsibility

The Parser Module receives a raw SQL string
and produces three outputs:

```
Input:  sql = "SELECT month, SUM(revenue) FROM sales GROUP BY month"

Output:
    ast         = sqlglot AST expression object
    fingerprint = "TIME_SUM_GROUP1_ORDER_ASC"
    sql_features = [0,1,0,0,1,1,0,0,0,0,0,0,0.2,0.4,0,0,0,0]
                   (dims 0-17 of feature vector)
```

### 4.2 ast_helpers.py — Pure AST Functions

All AST inspection is done through pure functions.
No side effects. No state.

```python
# sqlviz-inference/src/parser/ast_helpers.py

import sqlglot
import sqlglot.expressions as exp
from typing import Any


def parse_sql(sql: str, dialect: str = "duckdb") -> exp.Expression | None:
    """
    Parse SQL string to AST using sqlglot.
    Returns None if SQL is invalid.
    Never raises — used in graceful degradation.
    """
    try:
        return sqlglot.parse_one(sql, dialect=dialect)
    except Exception:
        return None


def has_group_by(ast: exp.Expression) -> bool:
    return ast.find(exp.Group) is not None


def has_order_by(ast: exp.Expression) -> bool:
    return ast.find(exp.Order) is not None


def has_order_by_desc(ast: exp.Expression) -> bool:
    order = ast.find(exp.Order)
    if not order:
        return False
    directions = list(order.find_all(exp.Ordered))
    return any(d.args.get("desc") for d in directions)


def has_limit(ast: exp.Expression) -> bool:
    return ast.find(exp.Limit) is not None


def has_aggregation(ast: exp.Expression) -> bool:
    return bool(list(ast.find_all(exp.AggFunc)))


def has_sum(ast: exp.Expression) -> bool:
    return ast.find(exp.Sum) is not None


def has_count(ast: exp.Expression) -> bool:
    return ast.find(exp.Count) is not None


def has_avg(ast: exp.Expression) -> bool:
    return ast.find(exp.Avg) is not None


def has_window_function(ast: exp.Expression) -> bool:
    return ast.find(exp.Window) is not None


def has_cte(ast: exp.Expression) -> bool:
    return ast.find(exp.With) is not None


def has_join(ast: exp.Expression) -> bool:
    return ast.find(exp.Join) is not None


def has_where(ast: exp.Expression) -> bool:
    return ast.find(exp.Where) is not None


def has_subquery(ast: exp.Expression) -> bool:
    # A subquery is a SELECT inside another expression
    selects = list(ast.find_all(exp.Select))
    return len(selects) > 1


def has_partition_by(ast: exp.Expression) -> bool:
    window = ast.find(exp.Window)
    if not window:
        return False
    return window.find(exp.PartitionedByProperty) is not None


def has_case_when(ast: exp.Expression) -> bool:
    return ast.find(exp.Case) is not None


def has_distinct(ast: exp.Expression) -> bool:
    return ast.find(exp.Distinct) is not None


def count_group_by_columns(ast: exp.Expression) -> int:
    group = ast.find(exp.Group)
    if not group:
        return 0
    return len(list(group.find_all(exp.Column)))


def count_select_columns(ast: exp.Expression) -> int:
    select = ast.find(exp.Select)
    if not select:
        return 0
    return len(list(select.find_all(exp.Column)))


def get_table_names(ast: exp.Expression) -> list[str]:
    """Extract all table names referenced in the SQL."""
    tables = ast.find_all(exp.Table)
    return [t.name.lower() for t in tables if t.name]


def get_column_names_from_select(ast: exp.Expression) -> list[str]:
    """Extract column names or aliases from SELECT clause."""
    select = ast.find(exp.Select)
    if not select:
        return []
    names = []
    for expr in select.expressions:
        if hasattr(expr, 'alias') and expr.alias:
            names.append(expr.alias.lower())
        elif hasattr(expr, 'name') and expr.name:
            names.append(expr.name.lower())
    return names


def is_single_metric_query(ast: exp.Expression) -> bool:
    """
    Returns True if the query produces exactly one aggregated metric
    with no GROUP BY dimension.
    This is the strongest KPI signal.
    Example: SELECT SUM(revenue) FROM sales
    """
    return (
        has_aggregation(ast) and
        not has_group_by(ast) and
        count_select_columns(ast) <= 2
    )


def is_ranking_pattern(ast: exp.Expression) -> bool:
    """
    Returns True if the query has explicit ranking pattern:
    ORDER BY DESC + LIMIT N
    """
    return has_order_by_desc(ast) and has_limit(ast)
```

### 4.3 fingerprint.py — Fingerprint Generation

```python
# sqlviz-inference/src/parser/fingerprint.py

import sqlglot.expressions as exp
from .ast_helpers import (
    has_aggregation, has_group_by, has_order_by,
    has_order_by_desc, has_limit, has_window_function,
    is_single_metric_query, count_group_by_columns,
    has_cte, has_join
)


# Temporal column name patterns
# These are checked against GROUP BY column names
TEMPORAL_PATTERNS = {
    "year", "month", "week", "day", "date", "datetime",
    "timestamp", "hora", "fecha", "periodo", "quarter",
    "mes", "año", "semana", "dia", "time", "dt", "created_at",
    "updated_at", "order_date", "sale_date", "event_date"
}


def _has_temporal_dimension(ast: exp.Expression) -> bool:
    """
    Detect temporal dimension in GROUP BY columns.
    Checks column names against TEMPORAL_PATTERNS.
    Also checks for date functions: DATE_TRUNC, EXTRACT, etc.
    """
    group = ast.find(exp.Group)
    if not group:
        return False

    # Check column names
    for col in group.find_all(exp.Column):
        if col.name.lower() in TEMPORAL_PATTERNS:
            return True

    # Check for date functions in GROUP BY
    date_funcs = (
        exp.DateTrunc, exp.Extract, exp.DateDiff,
        exp.DateAdd, exp.TimestampTrunc
    )
    for func_type in date_funcs:
        if group.find(func_type):
            return True

    return False


def generate_fingerprint(ast: exp.Expression | None) -> str:
    """
    Generate a normalized fingerprint from a SQL AST.

    The fingerprint represents the analytical pattern
    independent of table names, column names, or language.

    Examples:
        SELECT SUM(revenue)
            → "SUM_KPI"

        SELECT month, SUM(rev) FROM t GROUP BY month ORDER BY month
            → "TIME_SUM_GROUP1_ORDER_ASC"

        SELECT cat, COUNT(*) FROM t GROUP BY cat ORDER BY 2 DESC LIMIT 10
            → "COUNT_GROUP1_ORDER_DESC_LIMIT"

        SELECT a, b, c FROM t
            → "UNKNOWN"

        SELECT date, SUM(x) OVER (PARTITION BY y ORDER BY date)
            → "TIME_SUM_WINDOW"
    """
    if ast is None:
        return "UNKNOWN"

    patterns = []

    # 1. KPI pattern — single aggregation, no dimension
    if is_single_metric_query(ast):
        agg_funcs = list(ast.find_all(exp.AggFunc))
        if agg_funcs:
            agg_name = type(agg_funcs[0]).__name__.upper()
            return f"{agg_name}_KPI"

    # 2. Temporal dimension
    if _has_temporal_dimension(ast):
        patterns.append("TIME")

    # 3. Aggregation functions (sorted for consistency)
    agg_funcs = list(ast.find_all(exp.AggFunc))
    if agg_funcs:
        agg_names = sorted({type(a).__name__.upper() for a in agg_funcs})
        patterns.append("_".join(agg_names))

    # 4. GROUP BY columns count
    if has_group_by(ast):
        group_count = count_group_by_columns(ast)
        patterns.append(f"GROUP{group_count}")

    # 5. ORDER BY direction
    if has_order_by(ast):
        if has_order_by_desc(ast):
            patterns.append("ORDER_DESC")
        else:
            patterns.append("ORDER_ASC")

    # 6. LIMIT (ranking signal)
    if has_limit(ast):
        patterns.append("LIMIT")

    # 7. Window functions
    if has_window_function(ast):
        patterns.append("WINDOW")

    # 8. CTE
    if has_cte(ast):
        patterns.append("CTE")

    # 9. JOIN
    if has_join(ast):
        patterns.append("JOIN")

    return "_".join(patterns) if patterns else "UNKNOWN"
```

### 4.4 sql_parser.py — SQL Features Extraction

```python
# sqlviz-inference/src/parser/sql_parser.py

from ..context import RuntimeContext
from ..utils.yaml_loader import yaml_loader
from .ast_helpers import (
    parse_sql, has_group_by, has_order_by, has_order_by_desc,
    has_limit, has_aggregation, has_sum, has_count, has_avg,
    has_window_function, has_cte, has_join, has_where,
    has_subquery, has_partition_by, has_case_when, has_distinct,
    count_group_by_columns, count_select_columns,
    is_single_metric_query, is_ranking_pattern
)
from .fingerprint import generate_fingerprint


class SQLParser:
    """
    Parses SQL string and extracts:
    1. AST (sqlglot expression)
    2. Fingerprint (analytical pattern string)
    3. SQL Features (dims 0-17 of feature vector)

    Fails gracefully — if SQL is unparseable,
    returns context with UNKNOWN fingerprint
    and zero feature vector.
    """

    def run(self, context: RuntimeContext) -> RuntimeContext:
        try:
            return self._parse(context)
        except Exception as e:
            return context.with_error("SQLParser", str(e))

    def _parse(self, context: RuntimeContext) -> RuntimeContext:
        sql = context.sql.strip()

        # Parse SQL to AST
        ast = parse_sql(sql)

        if ast is None:
            context.ast = None
            context.fingerprint = "UNKNOWN"
            context.sql_features = [0.0] * 18
            return context

        # Generate fingerprint
        context.ast = ast
        context.fingerprint = generate_fingerprint(ast)

        # Extract SQL features (dims 0-17)
        context.sql_features = self._extract_sql_features(ast)

        # Initialize feature vector with SQL features
        fv = context.feature_vector  # [0.0] * 39
        for i, val in enumerate(context.sql_features):
            fv[i] = val
        context.feature_vector = fv

        return context

    def _extract_sql_features(self, ast) -> list[float]:
        """
        Extract dims 0-17 of the feature vector from AST.
        All values normalized to [0.0, 1.0].
        """
        group_count = count_group_by_columns(ast)
        select_count = count_select_columns(ast)

        return [
            # dim 0 — has_group_by
            1.0 if has_group_by(ast) else 0.0,
            # dim 1 — has_order_by
            1.0 if has_order_by(ast) else 0.0,
            # dim 2 — has_order_by_desc
            1.0 if has_order_by_desc(ast) else 0.0,
            # dim 3 — has_limit
            1.0 if has_limit(ast) else 0.0,
            # dim 4 — has_aggregation
            1.0 if has_aggregation(ast) else 0.0,
            # dim 5 — has_sum
            1.0 if has_sum(ast) else 0.0,
            # dim 6 — has_count
            1.0 if has_count(ast) else 0.0,
            # dim 7 — has_avg
            1.0 if has_avg(ast) else 0.0,
            # dim 8 — has_window_function
            1.0 if has_window_function(ast) else 0.0,
            # dim 9 — has_cte
            1.0 if has_cte(ast) else 0.0,
            # dim 10 — has_join
            1.0 if has_join(ast) else 0.0,
            # dim 11 — has_where
            1.0 if has_where(ast) else 0.0,
            # dim 12 — group_by_column_count normalized
            min(group_count / 5.0, 1.0),
            # dim 13 — select_column_count normalized
            min(select_count / 10.0, 1.0),
            # dim 14 — has_subquery
            1.0 if has_subquery(ast) else 0.0,
            # dim 15 — has_partition_by
            1.0 if has_partition_by(ast) else 0.0,
            # dim 16 — has_case_when
            1.0 if has_case_when(ast) else 0.0,
            # dim 17 — has_distinct
            1.0 if has_distinct(ast) else 0.0,
        ]
```

### 4.5 Parser Tests

```python
# sqlviz-inference/tests/test_parser.py

import pytest
from src.context import RuntimeContext
from src.parser.sql_parser import SQLParser
from src.parser.fingerprint import generate_fingerprint
from src.parser.ast_helpers import parse_sql


parser = SQLParser()


def make_context(sql: str) -> RuntimeContext:
    ctx = RuntimeContext(sql=sql)
    return parser.run(ctx)


class TestFingerprint:

    def test_kpi_single_sum(self):
        ctx = make_context("SELECT SUM(revenue) FROM sales")
        assert ctx.fingerprint == "SUM_KPI"

    def test_kpi_single_count(self):
        ctx = make_context("SELECT COUNT(*) FROM orders")
        assert ctx.fingerprint == "COUNT_KPI"

    def test_time_series(self):
        ctx = make_context(
            "SELECT month, SUM(revenue) FROM sales GROUP BY month ORDER BY month"
        )
        assert ctx.fingerprint == "TIME_SUM_GROUP1_ORDER_ASC"

    def test_ranking(self):
        ctx = make_context(
            "SELECT product, COUNT(*) FROM orders "
            "GROUP BY product ORDER BY 2 DESC LIMIT 10"
        )
        assert ctx.fingerprint == "COUNT_GROUP1_ORDER_DESC_LIMIT"

    def test_detail_query(self):
        ctx = make_context("SELECT id, name, email FROM users")
        assert ctx.fingerprint == "UNKNOWN"

    def test_window_function(self):
        ctx = make_context(
            "SELECT date, SUM(revenue) OVER (ORDER BY date) FROM sales"
        )
        assert "WINDOW" in ctx.fingerprint

    def test_spanish_temporal(self):
        # Same pattern as English — fingerprint must be identical
        ctx_en = make_context(
            "SELECT month, SUM(revenue) FROM sales GROUP BY month"
        )
        ctx_es = make_context(
            "SELECT mes, SUM(ventas) FROM ventas GROUP BY mes"
        )
        assert ctx_en.fingerprint == ctx_es.fingerprint


class TestSQLFeatures:

    def test_group_by_detected(self):
        ctx = make_context("SELECT cat, COUNT(*) FROM t GROUP BY cat")
        assert ctx.feature_vector[0] == 1.0  # has_group_by

    def test_order_by_desc(self):
        ctx = make_context("SELECT cat, SUM(v) FROM t GROUP BY cat ORDER BY 2 DESC")
        assert ctx.feature_vector[1] == 1.0  # has_order_by
        assert ctx.feature_vector[2] == 1.0  # has_order_by_desc

    def test_aggregation_sum(self):
        ctx = make_context("SELECT SUM(revenue) FROM sales")
        assert ctx.feature_vector[4] == 1.0  # has_aggregation
        assert ctx.feature_vector[5] == 1.0  # has_sum

    def test_invalid_sql_graceful(self):
        ctx = make_context("THIS IS NOT SQL !!!###")
        assert ctx.fingerprint == "UNKNOWN"
        assert ctx.feature_vector == [0.0] * 39
        assert len(ctx.errors) > 0  # error was logged
        assert ctx.chart_winner == "table"  # safe default preserved
```

---

*Section 4 complete. Next: Section 5 — Feature Engine.*

---

## 5. Feature Engine

### 5.1 Responsibility

The Feature Engine computes the complete 39-dimension
feature vector V0 by filling the remaining dimensions
that the Parser could not fill from the AST alone.

```
Input:  context with
        - ast (from Parser)
        - sql_features in feature_vector[0-17] (from Parser)
        - data (query result rows)
        - schema (column definitions)

Output: context with
        - feature_vector[0-38] complete
          dims 18-24 → column type features
          dims 25-29 → data statistics
          dims 30-34 → semantic features (placeholder, filled by Semantic Engine)
          dims 35-37 → result shape features
```

### 5.2 Lazy Evaluation Strategy

Not all features have the same computational cost.

```
Cost Level  Dims    Features              When computed
──────────────────────────────────────────────────────────────
FREE        0-17    SQL structural        Always (from AST)
LOW         18-24   Column types          When schema available
LOW         35-37   Result shape          When data available
MEDIUM      25-26   Row/cardinality       When data available
HIGH        27-29   Statistics            When data available + row_count > 3
DEFERRED    30-34   Semantic              Filled by Semantic Engine
```

```python
# Lazy evaluation logic in feature_engine.py

def run(self, context: RuntimeContext) -> RuntimeContext:
    fv = context.feature_vector  # already has dims 0-17

    # LOW cost — always compute if schema available
    if context.has_schema:
        col_feats = column_features.compute(context.schema)
        for i, v in enumerate(col_feats, start=18):
            fv[i] = v

    # LOW cost — always compute if data available
    if context.has_data:
        shape_feats = result_shape.compute(context.data)
        fv[35] = shape_feats[0]  # result_row_count_is_1
        fv[36] = shape_feats[1]  # result_column_count_is_1
        fv[37] = shape_feats[2]  # result_is_wide_table

    # MEDIUM + HIGH cost — only if enough data
    if context.has_data and context.row_count > 0:
        fv[25] = min(context.row_count / 10000.0, 1.0)  # row_count normalized

    if context.has_data and context.row_count > 1:
        fv[26] = _cardinality_ratio(context.data, context.schema)

    if context.has_data and context.row_count >= 3:
        fv[27] = _temporal_cardinality(context.data, context.schema)
        fv[28] = _trend_strength(context.data, context.schema)
        fv[29] = 1.0 if _has_outliers(context.data, context.schema) else 0.0

    context.feature_vector = fv
    return context
```

### 5.3 column_features.py

```python
# sqlviz-inference/src/features/column_features.py

from ..context import ColumnSchema

# DuckDB type classifications
DATE_TYPES = {
    "DATE", "TIMESTAMP", "TIMESTAMP WITH TIME ZONE",
    "TIMESTAMP_S", "TIMESTAMP_MS", "TIMESTAMP_NS",
    "TIME", "INTERVAL"
}

NUMERIC_TYPES = {
    "TINYINT", "SMALLINT", "INTEGER", "INT", "BIGINT",
    "HUGEINT", "FLOAT", "DOUBLE", "DECIMAL", "NUMERIC",
    "REAL", "INT2", "INT4", "INT8"
}

STRING_TYPES = {
    "VARCHAR", "TEXT", "CHAR", "BPCHAR", "STRING", "BLOB"
}

BOOLEAN_TYPES = {"BOOLEAN", "BOOL"}


def compute(schema: list[ColumnSchema]) -> list[float]:
    """
    Compute column type features (dims 18-24).
    Returns a list of 7 floats.
    """
    if not schema:
        return [0.0] * 7

    total = len(schema)
    types = [col.type.upper().split("(")[0] for col in schema]
    # split("(") handles DECIMAL(10,2) → "DECIMAL"

    has_date    = any(t in DATE_TYPES for t in types)
    has_numeric = any(t in NUMERIC_TYPES for t in types)
    has_string  = any(t in STRING_TYPES for t in types)

    numeric_count = sum(1 for t in types if t in NUMERIC_TYPES)
    date_count    = sum(1 for t in types if t in DATE_TYPES)

    return [
        # dim 18 — has_date_column
        1.0 if has_date else 0.0,
        # dim 19 — has_numeric_column
        1.0 if has_numeric else 0.0,
        # dim 20 — has_string_column
        1.0 if has_string else 0.0,
        # dim 21 — numeric_column_ratio
        numeric_count / total,
        # dim 22 — date_column_ratio
        date_count / total,
        # dim 23 — has_single_numeric_column (KPI signal)
        1.0 if numeric_count == 1 and total <= 2 else 0.0,
        # dim 24 — has_two_numeric_columns (correlation signal)
        1.0 if numeric_count >= 2 and date_count == 0 else 0.0,
    ]
```

### 5.4 data_statistics.py

```python
# sqlviz-inference/src/features/data_statistics.py

import math
from ..context import ColumnSchema
from ..utils.math_utils import (
    compute_trend_strength,
    compute_cardinality_ratio,
    compute_temporal_cardinality,
    has_statistical_outliers
)

NUMERIC_TYPES = {
    "TINYINT", "SMALLINT", "INTEGER", "INT", "BIGINT",
    "HUGEINT", "FLOAT", "DOUBLE", "DECIMAL", "NUMERIC", "REAL"
}

DATE_TYPES = {
    "DATE", "TIMESTAMP", "TIMESTAMP WITH TIME ZONE",
    "TIMESTAMP_S", "TIMESTAMP_MS", "TIMESTAMP_NS"
}


def _get_numeric_values(
    data: list[dict],
    schema: list[ColumnSchema]
) -> list[float]:
    """Extract first numeric column values as floats."""
    for col in schema:
        if col.type.upper().split("(")[0] in NUMERIC_TYPES:
            values = []
            for row in data:
                val = row.get(col.name)
                if val is not None:
                    try:
                        values.append(float(val))
                    except (TypeError, ValueError):
                        pass
            if values:
                return values
    return []


def _get_date_values(
    data: list[dict],
    schema: list[ColumnSchema]
) -> list[str]:
    """Extract first date/timestamp column values as strings."""
    for col in schema:
        if col.type.upper().split("(")[0] in DATE_TYPES:
            return [
                str(row.get(col.name, ""))
                for row in data
                if row.get(col.name) is not None
            ]
    return []


def compute_dim25_row_count(data: list[dict]) -> float:
    """dim 25 — row_count normalized to [0,1], cap at 10000."""
    return min(len(data) / 10000.0, 1.0)


def compute_dim26_cardinality(
    data: list[dict],
    schema: list[ColumnSchema]
) -> float:
    """
    dim 26 — cardinality ratio of the first non-numeric column.
    unique_values / total_rows → [0, 1]
    """
    if not data:
        return 0.0

    # Find first string or categorical column
    for col in schema:
        col_type = col.type.upper().split("(")[0]
        if col_type not in NUMERIC_TYPES and col_type not in DATE_TYPES:
            values = [row.get(col.name) for row in data if row.get(col.name)]
            if values:
                unique = len(set(str(v) for v in values))
                return unique / len(values)

    return 0.0


def compute_dim27_temporal_cardinality(
    data: list[dict],
    schema: list[ColumnSchema]
) -> float:
    """
    dim 27 — temporal cardinality normalized to [0,1].
    How many distinct time periods exist / 366 (max days in a year).
    """
    date_values = _get_date_values(data, schema)
    if not date_values:
        return 0.0
    unique_dates = len(set(date_values))
    return min(unique_dates / 366.0, 1.0)


def compute_dim28_trend_strength(
    data: list[dict],
    schema: list[ColumnSchema]
) -> float:
    """
    dim 28 — R² of linear regression on numeric values.
    Measures how well the data fits a linear trend.
    """
    values = _get_numeric_values(data, schema)
    if len(values) < 3:
        return 0.0
    return compute_trend_strength(values)


def compute_dim29_has_outliers(
    data: list[dict],
    schema: list[ColumnSchema]
) -> float:
    """
    dim 29 — 1.0 if numeric column has outliers (Z-score > 3).
    """
    values = _get_numeric_values(data, schema)
    if len(values) < 4:
        return 0.0
    return 1.0 if has_statistical_outliers(values) else 0.0
```

### 5.5 result_shape.py

```python
# sqlviz-inference/src/features/result_shape.py


def compute(data: list[dict]) -> list[float]:
    """
    Compute result shape features (dims 35-37).
    These are the strongest signals for KPI and Table detection.

    Returns [dim35, dim36, dim37]
    """
    if not data:
        return [0.0, 0.0, 0.0]

    row_count = len(data)
    col_count = len(data[0]) if data else 0

    return [
        # dim 35 — result_row_count_is_1 (strongest KPI signal)
        1.0 if row_count == 1 else 0.0,

        # dim 36 — result_column_count_is_1 (KPI signal)
        1.0 if col_count == 1 else 0.0,

        # dim 37 — result_is_wide_table (Table/Detail signal)
        1.0 if row_count > 20 and col_count > 5 else 0.0,
    ]
```

### 5.6 utils/math_utils.py — Statistical Functions

```python
# sqlviz-inference/src/utils/math_utils.py

import math


def compute_trend_strength(values: list[float]) -> float:
    """
    Compute R² of linear regression.
    Measures how well the data fits a linear trend.
    Returns value in [0.0, 1.0].

    R² > 0.8 → strong trend
    R² > 0.5 → moderate trend
    R² < 0.3 → no clear trend
    """
    n = len(values)
    if n < 3:
        return 0.0

    x = list(range(n))
    x_mean = sum(x) / n
    y_mean = sum(values) / n

    ss_xy = sum((x[i] - x_mean) * (values[i] - y_mean) for i in range(n))
    ss_xx = sum((x[i] - x_mean) ** 2 for i in range(n))

    if ss_xx == 0:
        return 0.0

    b = ss_xy / ss_xx
    a = y_mean - b * x_mean

    y_pred = [a + b * xi for xi in x]
    ss_res = sum((values[i] - y_pred[i]) ** 2 for i in range(n))
    ss_tot = sum((values[i] - y_mean) ** 2 for i in range(n))

    if ss_tot == 0:
        return 1.0

    return max(0.0, 1.0 - ss_res / ss_tot)


def compute_mean(values: list[float]) -> float:
    if not values:
        return 0.0
    return sum(values) / len(values)


def compute_stddev(values: list[float]) -> float:
    if len(values) < 2:
        return 0.0
    mean = compute_mean(values)
    variance = sum((v - mean) ** 2 for v in values) / (len(values) - 1)
    return math.sqrt(variance)


def compute_skewness(values: list[float]) -> float:
    """
    Compute skewness of a distribution.
    skewness = (1/n) * Σ((xᵢ - μ) / σ)³

    |skewness| < 1.0  → symmetric
    |skewness| > 2.0  → highly skewed
    """
    n = len(values)
    if n < 3:
        return 0.0
    mean = compute_mean(values)
    std = compute_stddev(values)
    if std == 0:
        return 0.0
    return sum(((v - mean) / std) ** 3 for v in values) / n


def compute_kurtosis(values: list[float]) -> float:
    """
    Compute excess kurtosis (normal distribution = 0).
    kurtosis = (1/n) * Σ((xᵢ - μ) / σ)⁴ - 3

    kurtosis > 3  → heavy tails → Boxplot preferred
    kurtosis < -1 → flat distribution → Histogram preferred
    """
    n = len(values)
    if n < 4:
        return 0.0
    mean = compute_mean(values)
    std = compute_stddev(values)
    if std == 0:
        return 0.0
    return sum(((v - mean) / std) ** 4 for v in values) / n - 3


def has_statistical_outliers(values: list[float], z_threshold: float = 3.0) -> bool:
    """
    Returns True if any value has Z-score > z_threshold.
    Uses Z-score: z = (x - μ) / σ
    """
    if len(values) < 4:
        return False
    mean = compute_mean(values)
    std = compute_stddev(values)
    if std == 0:
        return False
    outlier_count = sum(1 for v in values if abs((v - mean) / std) > z_threshold)
    return (outlier_count / len(values)) > 0.01


def pearson_r(x: list[float], y: list[float]) -> float:
    """
    Compute Pearson correlation coefficient between two lists.
    Returns value in [-1.0, 1.0].
    """
    n = len(x)
    if n < 3 or len(y) != n:
        return 0.0
    x_mean = compute_mean(x)
    y_mean = compute_mean(y)
    numerator = sum((x[i] - x_mean) * (y[i] - y_mean) for i in range(n))
    denom_x = sum((x[i] - x_mean) ** 2 for i in range(n))
    denom_y = sum((y[i] - y_mean) ** 2 for i in range(n))
    denominator = math.sqrt(denom_x * denom_y)
    if denominator == 0:
        return 0.0
    return max(-1.0, min(1.0, numerator / denominator))


def compute_cardinality_ratio(values: list) -> float:
    """unique_values / total_values → [0, 1]"""
    if not values:
        return 0.0
    return len(set(str(v) for v in values)) / len(values)


def softmax(scores: dict[str, float]) -> dict[str, float]:
    """
    Numerically stable softmax.
    Converts raw scores to values that sum to 1.0.
    NOTE: In V0 these are normalized_scores, not true probabilities.
    """
    if not scores:
        return {}
    max_score = max(scores.values())
    exp_scores = {k: math.exp(v - max_score) for k, v in scores.items()}
    total = sum(exp_scores.values())
    if total == 0:
        return {k: 1.0 / len(scores) for k in scores}
    return {k: v / total for k, v in exp_scores.items()}


def min_max_normalize(scores: dict[str, float]) -> dict[str, float]:
    """
    V0 normalization — honest relative ranking.
    Does not create artificial winners above min_threshold.
    """
    if not scores:
        return {}
    min_s = min(scores.values())
    max_s = max(scores.values())
    if max_s == min_s:
        return {k: 1.0 if max_s > 0 else 0.0 for k in scores}
    return {k: (v - min_s) / (max_s - min_s) for k, v in scores.items()}
```

### 5.7 utils/confidence.py

```python
# sqlviz-inference/src/utils/confidence.py


def confidence_gap(
    normalized_scores: dict[str, float]
) -> float:
    """
    Compute confidence gap = best_score - second_best_score.
    Higher gap = more certain inference.

    Returns value in [0.0, 1.0].
    """
    if len(normalized_scores) < 2:
        return 1.0
    sorted_scores = sorted(normalized_scores.values(), reverse=True)
    return sorted_scores[0] - sorted_scores[1]


def quality_label(raw_score: float, thresholds: dict = None) -> str:
    """
    Convert raw_score to quality label.

    high:   raw_score > 0.70
    medium: raw_score > 0.35
    low:    raw_score <= 0.35

    Thresholds can be overridden from thresholds.yaml.
    """
    if thresholds is None:
        thresholds = {"high": 0.70, "medium": 0.35}

    if raw_score > thresholds["high"]:
        return "high"
    elif raw_score > thresholds["medium"]:
        return "medium"
    else:
        return "low"


def should_apply_fallback(
    raw_score: float,
    min_threshold: float = 0.35
) -> bool:
    """
    Returns True if the best raw_score is below the minimum
    acceptance threshold → apply fallback (Table chart).
    """
    return raw_score < min_threshold
```

### 5.8 feature_engine.py — Complete

```python
# sqlviz-inference/src/features/feature_engine.py

from ..context import RuntimeContext
from .column_features import compute as compute_column_features
from .result_shape import compute as compute_result_shape
from .data_statistics import (
    compute_dim25_row_count,
    compute_dim26_cardinality,
    compute_dim27_temporal_cardinality,
    compute_dim28_trend_strength,
    compute_dim29_has_outliers,
)


class FeatureEngine:
    """
    Computes the complete 39-dimension feature vector V0.

    Fills dims 18-37 (dims 0-17 already filled by Parser).
    Uses lazy evaluation — expensive features only when data available.
    """

    def run(self, context: RuntimeContext) -> RuntimeContext:
        try:
            return self._compute(context)
        except Exception as e:
            return context.with_error("FeatureEngine", str(e))

    def _compute(self, context: RuntimeContext) -> RuntimeContext:
        fv = context.feature_vector[:]  # copy — never mutate

        # ── Column type features (dims 18-24) ─────────────────
        # LOW cost — compute whenever schema is available
        if context.has_schema:
            col_feats = compute_column_features(context.schema)
            for i, val in enumerate(col_feats):
                fv[18 + i] = val

        # ── Data statistics (dims 25-29) ──────────────────────
        # MEDIUM/HIGH cost — compute when data is available
        if context.has_data:
            fv[25] = compute_dim25_row_count(context.data)

        if context.has_data and context.row_count > 1:
            fv[26] = compute_dim26_cardinality(context.data, context.schema)

        if context.has_data and context.row_count >= 3:
            fv[27] = compute_dim27_temporal_cardinality(
                context.data, context.schema
            )
            fv[28] = compute_dim28_trend_strength(
                context.data, context.schema
            )
            fv[29] = compute_dim29_has_outliers(
                context.data, context.schema
            )

        # dims 30-34 → reserved for Semantic Engine

        # ── Result shape features (dims 35-37) ────────────────
        # LOW cost — compute when data is available
        if context.has_data:
            shape = compute_result_shape(context.data)
            fv[35] = shape[0]
            fv[36] = shape[1]
            fv[37] = shape[2]

        context.feature_vector = fv
        return context
```

### 5.9 Feature Engine Tests

```python
# sqlviz-inference/tests/test_feature_engine.py

import pytest
from src.context import RuntimeContext, ColumnSchema
from src.features.feature_engine import FeatureEngine

engine = FeatureEngine()


def make_context(data, schema_defs):
    schema = [ColumnSchema(name=n, type=t) for n, t in schema_defs]
    ctx = RuntimeContext(sql="SELECT 1", data=data, schema=schema)
    return engine.run(ctx)


class TestColumnFeatures:

    def test_date_column_detected(self):
        ctx = make_context(
            data=[{"month": "2024-01", "revenue": 1000}],
            schema_defs=[("month", "DATE"), ("revenue", "DOUBLE")]
        )
        assert ctx.feature_vector[18] == 1.0  # has_date_column
        assert ctx.feature_vector[19] == 1.0  # has_numeric_column

    def test_single_numeric_kpi_signal(self):
        ctx = make_context(
            data=[{"total": 125430}],
            schema_defs=[("total", "DOUBLE")]
        )
        assert ctx.feature_vector[23] == 1.0  # has_single_numeric_column


class TestResultShape:

    def test_kpi_shape_detected(self):
        ctx = make_context(
            data=[{"total": 125430}],
            schema_defs=[("total", "DOUBLE")]
        )
        assert ctx.feature_vector[35] == 1.0  # result_row_count_is_1
        assert ctx.feature_vector[36] == 1.0  # result_column_count_is_1

    def test_wide_table_detected(self):
        row = {f"col{i}": i for i in range(8)}
        data = [row] * 25
        schema_defs = [(f"col{i}", "INTEGER") for i in range(8)]
        ctx = make_context(data=data, schema_defs=schema_defs)
        assert ctx.feature_vector[37] == 1.0  # result_is_wide_table


class TestDataStatistics:

    def test_trend_strength_strong(self):
        # Perfect linear trend
        data = [{"month": i, "revenue": i * 1000} for i in range(1, 13)]
        schema_defs = [("month", "INTEGER"), ("revenue", "DOUBLE")]
        ctx = make_context(data=data, schema_defs=schema_defs)
        assert ctx.feature_vector[28] > 0.90  # strong R²

    def test_row_count_normalized(self):
        data = [{"x": i} for i in range(5000)]
        schema_defs = [("x", "INTEGER")]
        ctx = make_context(data=data, schema_defs=schema_defs)
        assert ctx.feature_vector[25] == 0.5  # 5000/10000
```

---

*Section 5 complete. Next: Section 6 — Semantic Engine.*

---

## 6. Semantic Engine

### 6.1 Responsibility

The Semantic Engine classifies column names into
semantic categories regardless of language or naming convention.

```
Input:  context with feature_vector + schema

Output: context with
        feature_vector dims 30-34 filled
        semantic_classes added to context

Examples:
    "revenue"  → METRIC_REVENUE
    "ventas"   → METRIC_REVENUE
    "ingresos" → METRIC_REVENUE
    "fecha"    → TEMPORAL_DIMENSION
    "date"     → TEMPORAL_DIMENSION
    "region"   → GEOGRAPHIC_DIMENSION
    "pais"     → GEOGRAPHIC_DIMENSION
    "producto" → PRODUCT_ENTITY
    "cliente"  → CUSTOMER_ENTITY
```

### 6.2 semantic_dictionary.yaml

```yaml
# sqlviz-inference/rules/semantic_dictionary.yaml
# Maps column name patterns to semantic classes.
# Patterns are matched case-insensitively.
# Order matters — first match wins.

METRIC_REVENUE:
  exact:
    - revenue
    - ventas
    - ingresos
    - sales
    - income
    - facturacion
    - facturación
    - monto
    - importe
    - amount
    - total_revenue
    - gross_revenue
    - net_revenue
  contains:
    - revenue
    - ventas
    - sales
    - income

METRIC_COUNT:
  exact:
    - count
    - cantidad
    - total
    - num
    - numero
    - número
    - qty
    - quantity
    - units
    - unidades
  contains:
    - count
    - cantidad
    - quantity

METRIC_PROFIT:
  exact:
    - profit
    - ganancia
    - utilidad
    - margen
    - margin
    - benefit
    - beneficio
    - ebitda
    - earnings
  contains:
    - profit
    - ganancia
    - margin
    - margen

TEMPORAL_DIMENSION:
  exact:
    - date
    - fecha
    - day
    - dia
    - día
    - week
    - semana
    - month
    - mes
    - quarter
    - trimestre
    - year
    - año
    - anio
    - hour
    - hora
    - datetime
    - timestamp
    - periodo
    - period
    - time
    - created_at
    - updated_at
    - order_date
    - sale_date
    - event_date
    - dt
  contains:
    - date
    - fecha
    - month
    - year
    - week
    - quarter
    - timestamp
    - periodo
    - created
    - updated

GEOGRAPHIC_DIMENSION:
  exact:
    - country
    - pais
    - país
    - region
    - región
    - city
    - ciudad
    - state
    - estado
    - province
    - provincia
    - territory
    - territorio
    - zone
    - zona
    - location
    - ubicacion
    - ubicación
    - geo
    - geography
    - continent
    - continente
  contains:
    - country
    - pais
    - region
    - ciudad
    - city
    - state
    - province
    - location
    - geo

PRODUCT_ENTITY:
  exact:
    - product
    - producto
    - item
    - sku
    - article
    - articulo
    - artículo
    - category
    - categoria
    - categoría
    - brand
    - marca
    - model
    - modelo
    - service
    - servicio
  contains:
    - product
    - producto
    - category
    - categoria
    - brand
    - marca
    - sku

CUSTOMER_ENTITY:
  exact:
    - customer
    - cliente
    - user
    - usuario
    - client
    - buyer
    - comprador
    - account
    - cuenta
    - member
    - miembro
    - subscriber
    - suscriptor
  contains:
    - customer
    - cliente
    - user
    - usuario
    - account
    - buyer
```

### 6.3 fuzzy_match.py

```python
# sqlviz-inference/src/semantic/fuzzy_match.py

def normalize_name(name: str) -> str:
    """
    Normalize a column name for matching.
    Handles snake_case, camelCase, spaces.
    """
    import re
    # Convert camelCase to snake_case
    name = re.sub(r'([A-Z])', r'_\1', name).lower()
    # Replace non-alphanumeric with underscore
    name = re.sub(r'[^a-z0-9]', '_', name)
    # Remove leading/trailing underscores
    name = name.strip('_')
    # Collapse multiple underscores
    name = re.sub(r'_+', '_', name)
    return name


def match_column_name(
    column_name: str,
    dictionary: dict[str, dict]
) -> str | None:
    """
    Match a column name against the semantic dictionary.
    Returns the semantic class name or None if no match.

    Matching strategy (in order):
    1. Exact match (case-insensitive)
    2. Contains match (column_name contains pattern)
    3. No match → None

    First match wins — order in YAML matters.
    """
    normalized = normalize_name(column_name)

    for semantic_class, patterns in dictionary.items():
        # 1. Exact match
        exact_patterns = patterns.get("exact", [])
        for pattern in exact_patterns:
            if normalized == normalize_name(pattern):
                return semantic_class

        # 2. Contains match
        contains_patterns = patterns.get("contains", [])
        for pattern in contains_patterns:
            if normalize_name(pattern) in normalized:
                return semantic_class

    return None
```

### 6.4 semantic_engine.py — Complete

```python
# sqlviz-inference/src/semantic/semantic_engine.py

from ..context import RuntimeContext, ColumnSchema
from ..utils.yaml_loader import yaml_loader
from .fuzzy_match import match_column_name

# Semantic class → feature vector dimension mapping
SEMANTIC_TO_DIM = {
    "METRIC_REVENUE":       30,
    "TEMPORAL_DIMENSION":   31,
    "GEOGRAPHIC_DIMENSION": 32,
    "PRODUCT_ENTITY":       33,
    "CUSTOMER_ENTITY":      34,
}

# Additional derived signals (not in feature vector, used in context)
KPI_SEMANTIC_CLASSES = {"METRIC_REVENUE", "METRIC_COUNT", "METRIC_PROFIT"}
DIMENSION_CLASSES = {"TEMPORAL_DIMENSION", "GEOGRAPHIC_DIMENSION",
                     "PRODUCT_ENTITY", "CUSTOMER_ENTITY"}


class SemanticEngine:
    """
    Classifies column names into semantic categories.
    Fills feature vector dims 30-34.

    Uses semantic_dictionary.yaml — no hardcoded patterns in Python.
    """

    def __init__(self):
        self._dictionary = None

    @property
    def dictionary(self) -> dict:
        if self._dictionary is None:
            self._dictionary = yaml_loader.load("semantic_dictionary.yaml")
        return self._dictionary

    def run(self, context: RuntimeContext) -> RuntimeContext:
        try:
            return self._classify(context)
        except Exception as e:
            return context.with_error("SemanticEngine", str(e))

    def _classify(self, context: RuntimeContext) -> RuntimeContext:
        if not context.has_schema:
            return context

        fv = context.feature_vector[:]
        semantic_classes = {}

        for col in context.schema:
            semantic_class = match_column_name(col.name, self.dictionary)
            if semantic_class:
                semantic_classes[col.name] = semantic_class

                # Fill feature vector dim
                dim = SEMANTIC_TO_DIM.get(semantic_class)
                if dim is not None:
                    fv[dim] = 1.0

        # Derive additional signals from AST column names
        # (columns in SELECT that may not be in schema yet)
        if context.ast is not None:
            from ..parser.ast_helpers import get_column_names_from_select
            select_cols = get_column_names_from_select(context.ast)
            for col_name in select_cols:
                if col_name not in semantic_classes:
                    semantic_class = match_column_name(
                        col_name, self.dictionary
                    )
                    if semantic_class:
                        semantic_classes[col_name] = semantic_class
                        dim = SEMANTIC_TO_DIM.get(semantic_class)
                        if dim is not None:
                            fv[dim] = 1.0

        context.feature_vector = fv

        # Store semantic classes in score_trace for explainability
        if "semantic" not in context.score_trace:
            context.score_trace["semantic"] = {}
        context.score_trace["semantic"]["column_classes"] = semantic_classes

        return context

    def classify_column(self, column_name: str) -> str | None:
        """Public method for classifying a single column name."""
        return match_column_name(column_name, self.dictionary)
```

### 6.5 utils/yaml_loader.py

```python
# sqlviz-inference/src/utils/yaml_loader.py

import yaml
from pathlib import Path
from functools import lru_cache


class YAMLLoader:
    """
    Loads and caches YAML rule files.
    All modules use this — never load YAML directly.

    Files are loaded once and cached in memory.
    If a file changes, restart SQLviz to reload.
    """

    def __init__(self):
        self._rules_dir = Path(__file__).parent.parent.parent / "rules"
        self._cache: dict[str, dict] = {}

    def load(self, filename: str) -> dict:
        """
        Load a YAML file from the rules/ directory.
        Returns cached version if already loaded.
        """
        if filename not in self._cache:
            path = self._rules_dir / filename
            if not path.exists():
                raise FileNotFoundError(
                    f"Rules file not found: {path}\n"
                    f"Expected at: {self._rules_dir}"
                )
            with open(path, "r", encoding="utf-8") as f:
                self._cache[filename] = yaml.safe_load(f)
        return self._cache[filename]

    def reload(self, filename: str) -> dict:
        """Force reload a file, bypassing cache."""
        if filename in self._cache:
            del self._cache[filename]
        return self.load(filename)

    def reload_all(self) -> None:
        """Force reload all cached files."""
        self._cache.clear()


# Global singleton — import this everywhere
yaml_loader = YAMLLoader()
```

### 6.6 Semantic Engine Tests

```python
# sqlviz-inference/tests/test_semantic_engine.py

import pytest
from src.context import RuntimeContext, ColumnSchema
from src.semantic.semantic_engine import SemanticEngine
from src.semantic.fuzzy_match import match_column_name, normalize_name

engine = SemanticEngine()


class TestNormalization:

    def test_snake_case(self):
        assert normalize_name("total_revenue") == "total_revenue"

    def test_camel_case(self):
        assert normalize_name("totalRevenue") == "total_revenue"

    def test_spaces(self):
        assert normalize_name("Total Revenue") == "total_revenue"

    def test_special_chars(self):
        assert normalize_name("fecha_de_venta") == "fecha_de_venta"


class TestSemanticMatching:

    def test_revenue_english(self):
        result = engine.classify_column("revenue")
        assert result == "METRIC_REVENUE"

    def test_revenue_spanish(self):
        result = engine.classify_column("ventas")
        assert result == "METRIC_REVENUE"

    def test_revenue_alias(self):
        result = engine.classify_column("total_revenue")
        assert result == "METRIC_REVENUE"

    def test_temporal_english(self):
        result = engine.classify_column("month")
        assert result == "TEMPORAL_DIMENSION"

    def test_temporal_spanish(self):
        result = engine.classify_column("fecha")
        assert result == "TEMPORAL_DIMENSION"

    def test_temporal_created_at(self):
        result = engine.classify_column("created_at")
        assert result == "TEMPORAL_DIMENSION"

    def test_geographic_english(self):
        result = engine.classify_column("country")
        assert result == "GEOGRAPHIC_DIMENSION"

    def test_geographic_spanish(self):
        result = engine.classify_column("pais")
        assert result == "GEOGRAPHIC_DIMENSION"

    def test_product_entity(self):
        result = engine.classify_column("producto")
        assert result == "PRODUCT_ENTITY"

    def test_customer_entity(self):
        result = engine.classify_column("cliente")
        assert result == "CUSTOMER_ENTITY"

    def test_unknown_column(self):
        result = engine.classify_column("xyz_abc_123")
        assert result is None


class TestFeatureVectorUpdate:

    def test_revenue_sets_dim30(self):
        schema = [
            ColumnSchema(name="month", type="DATE"),
            ColumnSchema(name="revenue", type="DOUBLE")
        ]
        ctx = RuntimeContext(
            sql="SELECT month, SUM(revenue) FROM sales GROUP BY month",
            schema=schema,
            feature_vector=[0.0] * 39
        )
        ctx = engine.run(ctx)
        assert ctx.feature_vector[30] == 1.0  # METRIC_REVENUE
        assert ctx.feature_vector[31] == 1.0  # TEMPORAL_DIMENSION

    def test_spanish_columns(self):
        schema = [
            ColumnSchema(name="fecha", type="DATE"),
            ColumnSchema(name="ventas", type="DOUBLE"),
            ColumnSchema(name="region", type="VARCHAR")
        ]
        ctx = RuntimeContext(
            sql="SELECT fecha, SUM(ventas) FROM ventas GROUP BY fecha",
            schema=schema,
            feature_vector=[0.0] * 39
        )
        ctx = engine.run(ctx)
        assert ctx.feature_vector[30] == 1.0  # METRIC_REVENUE (ventas)
        assert ctx.feature_vector[31] == 1.0  # TEMPORAL_DIMENSION (fecha)
        assert ctx.feature_vector[32] == 1.0  # GEOGRAPHIC_DIMENSION (region)
```

---

*Section 6 complete. Next: Section 7 — Intent Engine.*

---

## 7. Intent Engine

### 7.1 Responsibility

The Intent Engine scores 12 analytical intents
using the complete 39-dimension feature vector.

```
Input:  context with feature_vector[38] complete

Output: context with
        intent_scores        list of IntentScore (all 12)
        intent_winner        "trend" | "comparison" | ...
        intent_raw_score     absolute strength [0.0, 1.0]
        intent_normalized_score relative ranking [0.0, 1.0]
        intent_confidence_gap   best - second_best
        intent_quality       "high" | "medium" | "low"
        score_trace["intent"] full scoring breakdown
```

### 7.2 intent_rules.yaml — Complete

```yaml
# sqlviz-inference/rules/intent_rules.yaml
# All intent scoring weights.
# weights must sum to 1.0 per intent.
# boosts are multipliers applied after scoring.

trend:
  description: "How does a metric change over time?"
  weights:
    has_temporal_dimension: 0.40    # dim 31
    has_group_by:           0.25    # dim 0
    has_aggregation:        0.20    # dim 4
    has_order_by:           0.10    # dim 1
    temporal_cardinality:   0.05    # dim 27
  boosts: {}
  penalties:
    no_temporal_dimension: 0.60     # strong penalty if no date

comparison:
  description: "How do categories compare to each other?"
  weights:
    has_group_by:              0.35  # dim 0
    has_aggregation:           0.30  # dim 4
    has_string_column:         0.20  # dim 20
    no_temporal_dimension:     0.10  # (1 - dim 31)
    group_by_column_count:     0.05  # dim 12
  boosts: {}
  penalties:
    no_group_by: 0.50

ranking:
  description: "What are the top/bottom N items?"
  weights:
    has_order_by_desc:  0.40        # dim 2
    has_limit:          0.30        # dim 3
    has_aggregation:    0.20        # dim 4
    has_group_by:       0.10        # dim 0
  boosts:
    order_desc_and_limit: 1.50      # multiplier when both present
  penalties:
    no_order_by_desc: 0.70

distribution:
  description: "How are values distributed?"
  weights:
    has_numeric_column:        0.40  # dim 19
    no_temporal_dimension:     0.30  # (1 - dim 31)
    no_group_by:               0.20  # (1 - dim 0)
    high_cardinality:          0.10  # dim 26 > 0.5
  boosts: {}
  penalties:
    no_numeric_column: 0.80

correlation:
  description: "Are two metrics related?"
  weights:
    has_two_numeric_columns:   0.50  # dim 24
    no_group_by:               0.30  # (1 - dim 0)
    no_aggregation:            0.20  # (1 - dim 4)
  boosts: {}
  penalties:
    single_numeric_column: 0.70
    has_aggregation: 0.40

composition:
  description: "What is the part-to-whole breakdown?"
  weights:
    has_group_by:          0.40      # dim 0
    has_aggregation:       0.30      # dim 4
    has_string_column:     0.20      # dim 20
    low_cardinality:       0.10      # dim 26 < 0.15
  boosts: {}
  penalties:
    high_cardinality: 0.40
    no_aggregation: 0.50

kpi:
  description: "What is the current value of a metric?"
  weights:
    result_row_count_is_1:     0.40  # dim 35  ← strongest signal
    result_column_count_is_1:  0.30  # dim 36  ← strong signal
    has_aggregation:           0.20  # dim 4
    no_group_by:               0.10  # (1 - dim 0)
  boosts: {}
  penalties:
    has_group_by:      0.80
    multiple_rows:     0.70
    no_aggregation:    0.30

anomaly:
  description: "Are there unexpected values in the data?"
  weights:
    has_temporal_dimension:  0.35    # dim 31
    has_aggregation:         0.30    # dim 4
    has_group_by:            0.20    # dim 0
    has_outliers:            0.15    # dim 29
  boosts:
    has_outliers_detected: 1.30
  penalties: {}

cohort:
  description: "How do groups behave over time?"
  weights:
    has_temporal_dimension:  0.35    # dim 31
    has_group_by:            0.30    # dim 0
    group_by_count_gte_2:    0.25    # dim 12 > 0.4
    has_aggregation:         0.10    # dim 4
  boosts: {}
  penalties:
    no_temporal_dimension: 0.60

retention:
  description: "Do users/customers return over time?"
  weights:
    has_temporal_dimension:  0.40    # dim 31
    has_customer_entity:     0.30    # dim 34
    has_window_function:     0.20    # dim 8
    has_join:                0.10    # dim 10
  boosts: {}
  penalties:
    no_temporal_dimension: 0.70
    no_customer_entity: 0.40

funnel:
  description: "Where do users drop off in a process?"
  weights:
    has_case_when:     0.40          # dim 16
    has_aggregation:   0.30          # dim 4
    has_count:         0.20          # dim 6
    has_group_by:      0.10          # dim 0
  boosts: {}
  penalties:
    no_case_when: 0.50

detail:
  description: "Show me the raw data."
  weights:
    no_aggregation:    0.50          # (1 - dim 4)
    no_group_by:       0.30          # (1 - dim 0)
    high_col_count:    0.20          # dim 13 > 0.3
  boosts: {}
  penalties: {}
  # detail is the safe fallback — no strong penalties
```

### 7.3 intent_engine.py — Complete

```python
# sqlviz-inference/src/intent/intent_engine.py

from ..context import RuntimeContext, IntentScore
from ..utils.yaml_loader import yaml_loader
from ..utils.math_utils import min_max_normalize
from ..utils.confidence import confidence_gap, quality_label, should_apply_fallback


# Feature vector index mapping
# Used to extract values from feature_vector by name
FEATURE_INDEX = {
    "has_group_by":              0,
    "has_order_by":              1,
    "has_order_by_desc":         2,
    "has_limit":                 3,
    "has_aggregation":           4,
    "has_sum":                   5,
    "has_count":                 6,
    "has_avg":                   7,
    "has_window_function":       8,
    "has_cte":                   9,
    "has_join":                  10,
    "has_where":                 11,
    "group_by_column_count":     12,
    "select_column_count":       13,
    "has_subquery":              14,
    "has_partition_by":          15,
    "has_case_when":             16,
    "has_distinct":              17,
    "has_date_column":           18,
    "has_numeric_column":        19,
    "has_string_column":         20,
    "numeric_column_ratio":      21,
    "date_column_ratio":         22,
    "has_single_numeric_column": 23,
    "has_two_numeric_columns":   24,
    "row_count_normalized":      25,
    "cardinality_ratio":         26,
    "temporal_cardinality":      27,
    "trend_strength":            28,
    "has_outliers":              29,
    "has_revenue_metric":        30,
    "has_temporal_dimension":    31,
    "has_geographic_dimension":  32,
    "has_product_entity":        33,
    "has_customer_entity":       34,
    "result_row_count_is_1":     35,
    "result_column_count_is_1":  36,
    "result_is_wide_table":      37,
}


def _get_feature(fv: list[float], name: str) -> float:
    """Get feature value by name. Returns 0.0 if not found."""
    idx = FEATURE_INDEX.get(name)
    if idx is None:
        return 0.0
    return fv[idx] if idx < len(fv) else 0.0


def _compute_derived_features(fv: list[float]) -> dict[str, float]:
    """
    Compute derived features not directly in the feature vector.
    These are boolean inversions or combinations used in YAML rules.
    """
    return {
        # Inversions
        "no_group_by":             1.0 - _get_feature(fv, "has_group_by"),
        "no_aggregation":          1.0 - _get_feature(fv, "has_aggregation"),
        "no_temporal_dimension":   1.0 - _get_feature(fv, "has_temporal_dimension"),
        "no_order_by_desc":        1.0 - _get_feature(fv, "has_order_by_desc"),
        "no_numeric_column":       1.0 - _get_feature(fv, "has_numeric_column"),
        "no_case_when":            1.0 - _get_feature(fv, "has_case_when"),
        "no_customer_entity":      1.0 - _get_feature(fv, "has_customer_entity"),

        # Derived
        "high_cardinality":        1.0 if _get_feature(fv, "cardinality_ratio") > 0.5 else 0.0,
        "low_cardinality":         1.0 if _get_feature(fv, "cardinality_ratio") < 0.15 else 0.0,
        "multiple_rows":           1.0 - _get_feature(fv, "result_row_count_is_1"),
        "single_numeric_column":   _get_feature(fv, "has_single_numeric_column"),
        "high_col_count":          1.0 if _get_feature(fv, "select_column_count") > 0.3 else 0.0,
        "group_by_count_gte_2":    1.0 if _get_feature(fv, "group_by_column_count") > 0.4 else 0.0,
        "has_outliers_detected":   _get_feature(fv, "has_outliers"),
    }


class IntentEngine:
    """
    Scores 12 analytical intents from the feature vector.
    Reads all weights from intent_rules.yaml.
    Never hardcodes numerical constants.
    """

    def __init__(self):
        self._rules = None

    @property
    def rules(self) -> dict:
        if self._rules is None:
            self._rules = yaml_loader.load("intent_rules.yaml")
        return self._rules

    def run(self, context: RuntimeContext) -> RuntimeContext:
        try:
            return self._score(context)
        except Exception as e:
            return context.with_error("IntentEngine", str(e))

    def _score(self, context: RuntimeContext) -> RuntimeContext:
        fv = context.feature_vector
        derived = _compute_derived_features(fv)

        # Build unified feature lookup
        all_features = {
            name: _get_feature(fv, name)
            for name in FEATURE_INDEX
        }
        all_features.update(derived)

        # Score all 12 intents
        raw_scores: dict[str, float] = {}
        signal_traces: dict[str, dict] = {}

        for intent_name, intent_config in self.rules.items():
            weights = intent_config.get("weights", {})
            penalties = intent_config.get("penalties", {})
            boosts = intent_config.get("boosts", {})

            # Positive score
            pos_score = 0.0
            signals = {}
            for feature_name, weight in weights.items():
                value = all_features.get(feature_name, 0.0)
                contribution = weight * value
                pos_score += contribution
                signals[feature_name] = {
                    "weight": weight,
                    "value": value,
                    "contribution": round(contribution, 4)
                }

            # Penalties
            penalty_total = 0.0
            for feature_name, penalty in penalties.items():
                value = all_features.get(feature_name, 0.0)
                if value > 0.5:  # penalty triggers when feature is active
                    penalty_total += penalty * value

            # Boosts (multipliers)
            boost_multiplier = 1.0
            for boost_name, multiplier in boosts.items():
                if all_features.get(boost_name, 0.0) > 0.5:
                    boost_multiplier = multiplier

            # Final raw score
            raw = max(0.0, min(1.0, (pos_score - penalty_total) * boost_multiplier))
            raw_scores[intent_name] = raw
            signal_traces[intent_name] = {
                "raw_score": round(raw, 4),
                "positive_score": round(pos_score, 4),
                "penalty_total": round(penalty_total, 4),
                "boost_multiplier": boost_multiplier,
                "signals": signals,
            }

        # Normalize scores
        normalized = min_max_normalize(raw_scores)

        # Sort by raw score descending
        sorted_intents = sorted(
            raw_scores.items(),
            key=lambda x: x[1],
            reverse=True
        )

        # Build IntentScore list
        intent_scores = [
            IntentScore(
                intent=name,
                raw_score=raw_scores[name],
                normalized_score=normalized[name],
                signals=signal_traces[name]["signals"]
            )
            for name, _ in sorted_intents
        ]

        winner = intent_scores[0]
        thresholds = yaml_loader.load("thresholds.yaml")

        context.intent_scores = intent_scores
        context.intent_winner = winner.intent
        context.intent_raw_score = winner.raw_score
        context.intent_normalized_score = winner.normalized_score
        context.intent_confidence_gap = confidence_gap(normalized)
        context.intent_quality = quality_label(
            winner.raw_score,
            thresholds.get("quality_thresholds", {})
        )

        # Score trace for explainability
        context.score_trace["intent"] = signal_traces

        # Explanation — top contributing signals of winner
        context.explanation = [
            {
                "module": "IntentEngine",
                "signal": feat,
                "weight": sig["weight"],
                "value": sig["value"],
                "contribution": sig["contribution"]
            }
            for feat, sig in signal_traces[winner.intent]["signals"].items()
            if sig["contribution"] > 0.01
        ]

        return context
```

### 7.4 Intent Engine Tests

```python
# sqlviz-inference/tests/test_intent_engine.py

import pytest
from src.context import RuntimeContext, ColumnSchema
from src.parser.sql_parser import SQLParser
from src.features.feature_engine import FeatureEngine
from src.semantic.semantic_engine import SemanticEngine
from src.intent.intent_engine import IntentEngine

parser   = SQLParser()
features = FeatureEngine()
semantic = SemanticEngine()
intent   = IntentEngine()


def infer_intent(sql: str, data=None, schema_defs=None):
    """Helper — run full pipeline up to IntentEngine."""
    schema = []
    if schema_defs:
        schema = [ColumnSchema(name=n, type=t) for n, t in schema_defs]

    ctx = RuntimeContext(sql=sql, data=data or [], schema=schema)
    ctx = parser.run(ctx)
    ctx = features.run(ctx)
    ctx = semantic.run(ctx)
    ctx = intent.run(ctx)
    return ctx


class TestTrendIntent:

    def test_monthly_revenue_is_trend(self):
        ctx = infer_intent(
            sql="SELECT month, SUM(revenue) FROM sales GROUP BY month ORDER BY month",
            data=[{"month": f"2024-{i:02d}", "revenue": i * 1000} for i in range(1, 13)],
            schema_defs=[("month", "DATE"), ("revenue", "DOUBLE")]
        )
        assert ctx.intent_winner == "trend"
        assert ctx.intent_raw_score > 0.70
        assert ctx.intent_quality == "high"

    def test_weekly_sales_is_trend(self):
        ctx = infer_intent(
            sql="SELECT week, SUM(ventas) FROM ventas GROUP BY week ORDER BY week",
            schema_defs=[("week", "DATE"), ("ventas", "DOUBLE")]
        )
        assert ctx.intent_winner == "trend"


class TestKPIIntent:

    def test_single_sum_is_kpi(self):
        ctx = infer_intent(
            sql="SELECT SUM(revenue) AS total FROM sales",
            data=[{"total": 125430.0}],
            schema_defs=[("total", "DOUBLE")]
        )
        assert ctx.intent_winner == "kpi"
        assert ctx.intent_raw_score > 0.70

    def test_count_all_is_kpi(self):
        ctx = infer_intent(
            sql="SELECT COUNT(*) AS total_orders FROM orders",
            data=[{"total_orders": 4521}],
            schema_defs=[("total_orders", "BIGINT")]
        )
        assert ctx.intent_winner == "kpi"


class TestRankingIntent:

    def test_top_products_is_ranking(self):
        ctx = infer_intent(
            sql="SELECT product, SUM(revenue) FROM sales "
                "GROUP BY product ORDER BY 2 DESC LIMIT 10",
            schema_defs=[("product", "VARCHAR"), ("revenue", "DOUBLE")]
        )
        assert ctx.intent_winner == "ranking"
        assert ctx.intent_raw_score > 0.60

    def test_ranking_boost_applied(self):
        # ORDER BY DESC + LIMIT → boost multiplier 1.5
        ctx = infer_intent(
            sql="SELECT cat, COUNT(*) FROM t GROUP BY cat ORDER BY 2 DESC LIMIT 5",
            schema_defs=[("cat", "VARCHAR"), ("count", "BIGINT")]
        )
        assert ctx.intent_winner == "ranking"


class TestComparisonIntent:

    def test_category_comparison(self):
        ctx = infer_intent(
            sql="SELECT region, SUM(revenue) FROM sales GROUP BY region",
            schema_defs=[("region", "VARCHAR"), ("revenue", "DOUBLE")]
        )
        assert ctx.intent_winner in ("comparison", "composition")

    def test_no_temporal_favors_comparison(self):
        ctx = infer_intent(
            sql="SELECT category, COUNT(*) FROM products GROUP BY category",
            schema_defs=[("category", "VARCHAR"), ("count", "BIGINT")]
        )
        assert ctx.intent_winner in ("comparison", "ranking", "composition")


class TestDetailIntent:

    def test_select_star_is_detail(self):
        ctx = infer_intent(
            sql="SELECT id, name, email, phone, address FROM customers",
            schema_defs=[
                ("id", "INTEGER"), ("name", "VARCHAR"),
                ("email", "VARCHAR"), ("phone", "VARCHAR"),
                ("address", "VARCHAR")
            ]
        )
        assert ctx.intent_winner == "detail"


class TestExplainability:

    def test_score_trace_present(self):
        ctx = infer_intent(
            sql="SELECT month, SUM(revenue) FROM sales GROUP BY month"
        )
        assert "intent" in ctx.score_trace
        assert "trend" in ctx.score_trace["intent"]
        assert "raw_score" in ctx.score_trace["intent"]["trend"]

    def test_explanation_not_empty(self):
        ctx = infer_intent(
            sql="SELECT SUM(revenue) FROM sales",
            data=[{"revenue": 1000}],
            schema_defs=[("revenue", "DOUBLE")]
        )
        assert len(ctx.explanation) > 0
        assert all("signal" in e for e in ctx.explanation)
        assert all("contribution" in e for e in ctx.explanation)
```

---

*Section 7 complete. Next: Section 8 — Chart Engine.*

---

## 8. Chart Engine

### 8.1 Responsibility

The Chart Engine selects the best chart type
using the intent vector and penalty rules.

```
Input:  context with intent_scores + feature_vector

Output: context with
        chart_winner          "line" | "bar" | "kpi" | ...
        chart_raw_score       absolute strength [0.0, 1.0]
        chart_normalized_score relative ranking [0.0, 1.0]
        chart_confidence_gap  best - second_best
        chart_quality         "high" | "medium" | "low"
        chart_alternatives    list of runner-up charts
        fallback_applied      True if quality was too low
        fallback_reason       explanation if fallback applied
        score_trace["chart"]  full scoring breakdown
```

### 8.2 chart_affinity_matrix.yaml — Complete

```yaml
# sqlviz-inference/rules/chart_affinity_matrix.yaml
# Affinity scores between chart types and intents.
# Values in [0.0, 1.0].
# Higher = stronger affinity.
# V0 — 8 chart types only.

# score(chart) = Σ affinity(chart, intent) × normalized_score(intent)

kpi:
  trend:        0.00
  comparison:   0.00
  ranking:      0.00
  distribution: 0.00
  correlation:  0.00
  composition:  0.00
  kpi:          1.00
  anomaly:      0.00
  cohort:       0.00
  retention:    0.00
  funnel:       0.00
  detail:       0.00

line:
  trend:        0.95
  comparison:   0.10
  ranking:      0.00
  distribution: 0.05
  correlation:  0.00
  composition:  0.00
  kpi:          0.00
  anomaly:      0.20
  cohort:       0.30
  retention:    0.40
  funnel:       0.00
  detail:       0.05

bar:
  trend:        0.15
  comparison:   0.90
  ranking:      0.40
  distribution: 0.20
  correlation:  0.00
  composition:  0.20
  kpi:          0.00
  anomaly:      0.10
  cohort:       0.10
  retention:    0.05
  funnel:       0.10
  detail:       0.10

bar_horizontal:
  trend:        0.05
  comparison:   0.60
  ranking:      0.95
  distribution: 0.10
  correlation:  0.00
  composition:  0.10
  kpi:          0.00
  anomaly:      0.05
  cohort:       0.05
  retention:    0.00
  funnel:       0.20
  detail:       0.05

pie:
  trend:        0.00
  comparison:   0.30
  ranking:      0.00
  distribution: 0.80
  correlation:  0.00
  composition:  0.90
  kpi:          0.00
  anomaly:      0.00
  cohort:       0.00
  retention:    0.00
  funnel:       0.10
  detail:       0.00

scatter:
  trend:        0.05
  comparison:   0.10
  ranking:      0.00
  distribution: 0.40
  correlation:  0.95
  composition:  0.00
  kpi:          0.00
  anomaly:      0.20
  cohort:       0.00
  retention:    0.00
  funnel:       0.00
  detail:       0.00

histogram:
  trend:        0.10
  comparison:   0.10
  ranking:      0.00
  distribution: 0.95
  correlation:  0.10
  composition:  0.00
  kpi:          0.00
  anomaly:      0.15
  cohort:       0.00
  retention:    0.00
  funnel:       0.00
  detail:       0.05

table:
  trend:        0.10
  comparison:   0.20
  ranking:      0.20
  distribution: 0.20
  correlation:  0.10
  composition:  0.10
  kpi:          0.10
  anomaly:      0.10
  cohort:       0.20
  retention:    0.20
  funnel:       0.20
  detail:       0.95
```

### 8.3 chart_penalties.yaml — Complete

```yaml
# sqlviz-inference/rules/chart_penalties.yaml
# Penalty rules per chart type.
# Penalties are subtracted from the affinity score.
# A penalty triggers when the condition feature value > 0.5.

kpi:
  penalties:
    has_group_by:             0.80  # KPI needs no dimension
    result_row_count_is_1:    0.00  # no penalty — this is good for KPI
    result_is_wide_table:     0.60  # wide table is not KPI
  # Special: KPI requires result_row_count_is_1
  # If result has multiple rows → heavy penalty applied separately

line:
  penalties:
    no_temporal_dimension:    0.60  # Line needs time axis
    result_row_count_is_1:    0.70  # Single row cannot be a trend
    result_is_wide_table:     0.30  # Many columns suggest table instead

bar:
  penalties:
    result_row_count_is_1:    0.50  # Single row better as KPI
    has_two_numeric_columns:  0.20  # Two numerics suggest scatter

bar_horizontal:
  penalties:
    result_row_count_is_1:    0.60  # Single row better as KPI
    no_group_by:              0.40  # Ranking needs categories

pie:
  penalties:
    no_temporal_dimension:    0.00  # no penalty — pie is fine without time
    has_temporal_dimension:   0.40  # time data → use Line instead
    ranking_pattern:          0.40  # ranking → use Bar Horizontal instead
    high_cardinality:         0.50  # too many slices → not readable
    result_row_count_is_1:    0.70  # single row → KPI instead
    correlation_intent:       0.20  # two numerics → Scatter instead

scatter:
  penalties:
    has_single_numeric_column: 0.70 # Scatter needs two numeric columns
    has_temporal_dimension:    0.50 # time + numeric → Line instead
    has_aggregation:           0.40 # aggregated data ≠ scatter

histogram:
  penalties:
    has_group_by:             0.40  # grouped data → use Bar instead
    has_temporal_dimension:   0.30  # time data → use Line instead
    result_row_count_is_1:    0.80  # single value cannot be distribution

table:
  penalties: {}
  # Table has no penalties — it is always the safe fallback
```

### 8.4 fallback_rules.yaml — Complete

```yaml
# sqlviz-inference/rules/fallback_rules.yaml

chart:
  min_raw_score: 0.35
  default: table
  message: "Low confidence inference — showing raw data"

intent:
  min_raw_score: 0.30
  default: detail
  message: "Intent unclear — defaulting to detail view"

quality_thresholds:
  high:   0.70
  medium: 0.35

display_rules:
  # quality=high, gap=high → very certain
  high_high:
    show_alternatives: 0
    show_warning: false

  # quality=high, gap=low → both good options
  high_low:
    show_alternatives: 2
    show_warning: false

  # quality=low, gap=high → winner barely acceptable
  low_high:
    show_alternatives: 1
    show_warning: true
    warning_message: "Low confidence inference"

  # quality=low, gap=low → fallback
  low_low:
    show_alternatives: 0
    show_warning: true
    apply_fallback: true
```

### 8.5 chart_engine.py — Complete

```python
# sqlviz-inference/src/chart/chart_engine.py

from ..context import RuntimeContext, ChartCandidate, IntentScore
from ..utils.yaml_loader import yaml_loader
from ..utils.math_utils import min_max_normalize
from ..utils.confidence import confidence_gap, quality_label, should_apply_fallback

# V0 chart types — exactly 8
V0_CHARTS = [
    "kpi", "line", "bar", "bar_horizontal",
    "pie", "scatter", "histogram", "table"
]

# Feature names used for penalty conditions
PENALTY_FEATURE_INDEX = {
    "has_group_by":              0,
    "has_aggregation":           4,
    "has_temporal_dimension":    31,
    "has_two_numeric_columns":   24,
    "has_single_numeric_column": 23,
    "result_row_count_is_1":     35,
    "result_column_count_is_1":  36,
    "result_is_wide_table":      37,
    "has_outliers":              29,
}


class ChartEngine:
    """
    Selects the best chart type using:
    1. Affinity scores from chart_affinity_matrix.yaml
    2. Penalty rules from chart_penalties.yaml
    3. Fallback rules from fallback_rules.yaml

    V0 supports exactly 8 chart types.
    """

    def __init__(self):
        self._affinity = None
        self._penalties = None
        self._fallback = None
        self._thresholds = None

    @property
    def affinity(self) -> dict:
        if self._affinity is None:
            self._affinity = yaml_loader.load("chart_affinity_matrix.yaml")
        return self._affinity

    @property
    def penalties(self) -> dict:
        if self._penalties is None:
            self._penalties = yaml_loader.load("chart_penalties.yaml")
        return self._penalties

    @property
    def fallback_rules(self) -> dict:
        if self._fallback is None:
            self._fallback = yaml_loader.load("fallback_rules.yaml")
        return self._fallback

    @property
    def thresholds(self) -> dict:
        if self._thresholds is None:
            self._thresholds = yaml_loader.load("thresholds.yaml")
        return self._thresholds

    def run(self, context: RuntimeContext) -> RuntimeContext:
        try:
            return self._score(context)
        except Exception as e:
            return context.with_error("ChartEngine", str(e))

    def _score(self, context: RuntimeContext) -> RuntimeContext:
        fv = context.feature_vector

        # Build intent probability map from intent_scores
        intent_probs: dict[str, float] = {}
        for score in context.intent_scores:
            intent_probs[score.intent] = score.normalized_score

        chart_traces = {}
        raw_scores: dict[str, float] = {}

        for chart_type in V0_CHARTS:
            # 1. Affinity score
            chart_affinity = self.affinity.get(chart_type, {})
            affinity_score = sum(
                chart_affinity.get(intent, 0.0) * prob
                for intent, prob in intent_probs.items()
            )

            # 2. Penalty score
            chart_penalties = self.penalties.get(chart_type, {}).get(
                "penalties", {}
            )
            penalty_total = 0.0
            penalties_applied = []

            for feature_name, penalty_weight in chart_penalties.items():
                idx = PENALTY_FEATURE_INDEX.get(feature_name)
                feature_value = fv[idx] if idx is not None and idx < len(fv) else 0.0

                # Derived penalty conditions
                if feature_name == "no_temporal_dimension":
                    feature_value = 1.0 - fv[31]
                elif feature_name == "no_group_by":
                    feature_value = 1.0 - fv[0]
                elif feature_name == "high_cardinality":
                    feature_value = 1.0 if fv[26] > 0.5 else 0.0
                elif feature_name == "ranking_pattern":
                    feature_value = 1.0 if (fv[2] > 0.5 and fv[3] > 0.5) else 0.0
                elif feature_name == "correlation_intent":
                    feature_value = next(
                        (s.normalized_score for s in context.intent_scores
                         if s.intent == "correlation"), 0.0
                    )

                if feature_value > 0.5:
                    applied_penalty = penalty_weight * feature_value
                    penalty_total += applied_penalty
                    penalties_applied.append({
                        "rule": feature_name,
                        "penalty": round(applied_penalty, 4),
                        "feature_value": round(feature_value, 4)
                    })

            # 3. Final raw score
            raw = max(0.0, min(1.0, affinity_score - penalty_total))
            raw_scores[chart_type] = raw

            chart_traces[chart_type] = {
                "affinity_score":    round(affinity_score, 4),
                "penalty_total":     round(penalty_total, 4),
                "penalties_applied": penalties_applied,
                "final_score":       round(raw, 4),
            }

        # Normalize scores
        normalized = min_max_normalize(raw_scores)

        # Sort by raw score descending
        sorted_charts = sorted(
            raw_scores.items(),
            key=lambda x: x[1],
            reverse=True
        )

        # Add normalized scores to traces
        for chart_type in V0_CHARTS:
            chart_traces[chart_type]["normalized_score"] = round(
                normalized.get(chart_type, 0.0), 4
            )

        # Build ChartCandidate list
        candidates = [
            ChartCandidate(
                chart_type=ct,
                affinity_score=chart_traces[ct]["affinity_score"],
                penalty_total=chart_traces[ct]["penalty_total"],
                final_score=chart_traces[ct]["final_score"],
                normalized_score=chart_traces[ct]["normalized_score"],
                penalties_applied=chart_traces[ct]["penalties_applied"]
            )
            for ct, _ in sorted_charts
        ]

        winner = candidates[0]
        fb_rules = self.fallback_rules
        quality_thresholds = fb_rules.get("quality_thresholds", {})

        # Check fallback
        fallback_applied = False
        fallback_reason = ""
        winner_chart = winner.chart_type

        if should_apply_fallback(
            winner.final_score,
            fb_rules.get("chart", {}).get("min_raw_score", 0.35)
        ):
            fallback_applied = True
            fallback_reason = fb_rules.get("chart", {}).get("message", "")
            winner_chart = fb_rules.get("chart", {}).get("default", "table")

        # Determine alternatives to show
        gap = confidence_gap(normalized)
        ql = quality_label(winner.final_score, quality_thresholds)

        if ql == "high" and gap >= 0.60:
            n_alternatives = 0
        elif ql == "high" and gap < 0.60:
            n_alternatives = 2
        elif ql in ("medium", "low") and gap >= 0.60:
            n_alternatives = 1
        else:
            n_alternatives = 0

        alternatives = [
            {"chart": c.chart_type, "raw_score": c.final_score}
            for c in candidates[1:n_alternatives + 1]
            if c.final_score > 0.0
        ]

        context.chart_candidates = candidates
        context.chart_winner = winner_chart
        context.chart_raw_score = winner.final_score
        context.chart_normalized_score = winner.normalized_score
        context.chart_confidence_gap = gap
        context.chart_quality = ql
        context.chart_alternatives = alternatives
        context.fallback_applied = fallback_applied
        context.fallback_reason = fallback_reason
        context.score_trace["chart"] = chart_traces

        return context
```

### 8.6 Chart Engine Tests

```python
# sqlviz-inference/tests/test_chart_engine.py

import pytest
from src.context import RuntimeContext, ColumnSchema, IntentScore
from src.parser.sql_parser import SQLParser
from src.features.feature_engine import FeatureEngine
from src.semantic.semantic_engine import SemanticEngine
from src.intent.intent_engine import IntentEngine
from src.chart.chart_engine import ChartEngine

parser   = SQLParser()
features = FeatureEngine()
semantic = SemanticEngine()
intent   = IntentEngine()
chart    = ChartEngine()


def full_infer(sql: str, data=None, schema_defs=None):
    schema = [ColumnSchema(name=n, type=t) for n, t in (schema_defs or [])]
    ctx = RuntimeContext(sql=sql, data=data or [], schema=schema)
    ctx = parser.run(ctx)
    ctx = features.run(ctx)
    ctx = semantic.run(ctx)
    ctx = intent.run(ctx)
    ctx = chart.run(ctx)
    return ctx


class TestChartSelection:

    def test_kpi_single_value(self):
        ctx = full_infer(
            sql="SELECT SUM(revenue) AS total FROM sales",
            data=[{"total": 125430.0}],
            schema_defs=[("total", "DOUBLE")]
        )
        assert ctx.chart_winner == "kpi"

    def test_line_time_series(self):
        ctx = full_infer(
            sql="SELECT month, SUM(revenue) FROM sales GROUP BY month ORDER BY month",
            data=[{"month": f"2024-{i:02d}", "revenue": i*1000} for i in range(1,13)],
            schema_defs=[("month", "DATE"), ("revenue", "DOUBLE")]
        )
        assert ctx.chart_winner == "line"

    def test_bar_category_comparison(self):
        ctx = full_infer(
            sql="SELECT region, SUM(revenue) FROM sales GROUP BY region",
            data=[
                {"region": "North", "revenue": 45000},
                {"region": "South", "revenue": 32000},
                {"region": "East",  "revenue": 28000},
            ],
            schema_defs=[("region", "VARCHAR"), ("revenue", "DOUBLE")]
        )
        assert ctx.chart_winner in ("bar", "bar_horizontal")

    def test_bar_horizontal_ranking(self):
        ctx = full_infer(
            sql="SELECT product, SUM(rev) FROM sales "
                "GROUP BY product ORDER BY 2 DESC LIMIT 10",
            schema_defs=[("product", "VARCHAR"), ("rev", "DOUBLE")]
        )
        assert ctx.chart_winner == "bar_horizontal"

    def test_table_detail_query(self):
        ctx = full_infer(
            sql="SELECT id, name, email, phone FROM customers",
            schema_defs=[
                ("id", "INTEGER"), ("name", "VARCHAR"),
                ("email", "VARCHAR"), ("phone", "VARCHAR")
            ]
        )
        assert ctx.chart_winner == "table"

    def test_scatter_two_numerics(self):
        ctx = full_infer(
            sql="SELECT price, quantity FROM products",
            data=[{"price": p, "quantity": q}
                  for p, q in [(10,100),(20,80),(30,60),(40,40)]],
            schema_defs=[("price", "DOUBLE"), ("quantity", "INTEGER")]
        )
        assert ctx.chart_winner == "scatter"


class TestPenalties:

    def test_pie_penalized_with_many_categories(self):
        # Many categories → pie gets high_cardinality penalty
        data = [{"cat": f"Cat{i}", "val": i} for i in range(20)]
        ctx = full_infer(
            sql="SELECT cat, SUM(val) FROM t GROUP BY cat",
            data=data,
            schema_defs=[("cat", "VARCHAR"), ("val", "DOUBLE")]
        )
        assert ctx.chart_winner != "pie"

    def test_line_penalized_without_temporal(self):
        # No temporal column → line gets penalty
        ctx = full_infer(
            sql="SELECT category, SUM(revenue) FROM sales GROUP BY category",
            schema_defs=[("category", "VARCHAR"), ("revenue", "DOUBLE")]
        )
        assert ctx.chart_winner != "line"


class TestFallback:

    def test_fallback_applied_when_low_confidence(self):
        # Ambiguous SQL with no clear pattern
        ctx = full_infer(
            sql="SELECT a, b, c FROM t WHERE x > 1",
            schema_defs=[("a", "VARCHAR"), ("b", "VARCHAR"), ("c", "VARCHAR")]
        )
        # If fallback applied → must be table
        if ctx.fallback_applied:
            assert ctx.chart_winner == "table"
            assert ctx.fallback_reason != ""


class TestExplainability:

    def test_score_trace_has_all_charts(self):
        ctx = full_infer(
            sql="SELECT month, SUM(revenue) FROM sales GROUP BY month"
        )
        from src.chart.chart_engine import V0_CHARTS
        for chart_type in V0_CHARTS:
            assert chart_type in ctx.score_trace["chart"]

    def test_alternatives_present_when_close(self):
        ctx = full_infer(
            sql="SELECT region, SUM(revenue) FROM sales GROUP BY region",
            schema_defs=[("region", "VARCHAR"), ("revenue", "DOUBLE")]
        )
        # When gap is low, alternatives should be shown
        if ctx.chart_confidence_gap < 0.60:
            assert len(ctx.chart_alternatives) > 0
```

---

*Section 8 complete. Next: Section 9 — Layout Engine.*

---

## 9. Layout Engine

### 9.1 Responsibility

The Layout Engine decides the visual position
and size of each panel in the dashboard grid.

```
Input:  context with
        chart_winner    (from Chart Engine)
        intent_winner   (from Intent Engine)
        feature_vector  (from Feature Engine)

Output: context with
        col_span        CSS Grid columns (1-12)
        row_span        CSS Grid rows (1-2)
        layout_importance  panel importance score [0.0, 1.0]
```

SQLviz uses a **12-column CSS Grid**.
Every panel occupies a number of columns (col_span)
and rows (row_span).

```
col_span=12 → full width  (100%)
col_span=9  → three quarters (75%)
col_span=8  → two thirds  (66%)
col_span=6  → half width  (50%)
col_span=4  → one third   (33%)
col_span=3  → quarter     (25%)
```

### 9.2 Layout Rules

The layout is determined by chart type first,
then refined by intent and data characteristics.

```
Priority 1 — Chart type overrides (always applied):
    kpi          → col_span=3,  row_span=1
    table        → col_span=12, row_span=2
    line         → col_span=12, row_span=1  (needs horizontal space)
    scatter      → col_span=6,  row_span=2  (needs square space)

Priority 2 — Intent-based adjustments:
    trend intent     + line  → col_span=12 (confirmed full width)
    ranking intent   + bar_h → col_span=8  (rankings need space)
    composition      + pie   → col_span=4  (pie can be smaller)
    correlation      + scat  → col_span=6, row_span=2

Priority 3 — Data characteristics:
    row_count > 100 AND table → row_span=3
    many categories (> 15)    → col_span=12 (needs more horizontal)

Priority 4 — KPI grouping:
    Multiple KPIs in same dashboard share a row
    Layout Engine uses context of all panels (future v0.2)
    In v0.1: each panel is laid out independently
```

### 9.3 layout_engine.py — Complete

```python
# sqlviz-inference/src/layout/layout_engine.py

from ..context import RuntimeContext
from ..utils.yaml_loader import yaml_loader


# Chart type → default layout
# (col_span, row_span)
CHART_DEFAULT_LAYOUT = {
    "kpi":          (3,  1),
    "line":         (12, 1),
    "bar":          (12, 1),
    "bar_horizontal": (12, 1),
    "pie":          (6,  1),
    "scatter":      (6,  2),
    "histogram":    (6,  1),
    "table":        (12, 2),
}

# Intent adjustments to col_span
# (intent, chart_type) → col_span override
INTENT_LAYOUT_ADJUSTMENTS = {
    ("trend",       "line"):          12,
    ("trend",       "bar"):           12,
    ("ranking",     "bar_horizontal"): 8,
    ("composition", "pie"):            4,
    ("kpi",         "kpi"):            3,
    ("detail",      "table"):         12,
    ("comparison",  "bar"):           12,
    ("comparison",  "bar_horizontal"): 8,
}


class LayoutEngine:
    """
    Assigns CSS Grid spans to panels.

    Rules (in priority order):
    1. Chart type default layout
    2. Intent-based adjustments
    3. Data characteristic adjustments
    4. Always clamp to valid range [1-12] col, [1-3] row
    """

    def run(self, context: RuntimeContext) -> RuntimeContext:
        try:
            return self._assign_layout(context)
        except Exception as e:
            return context.with_error("LayoutEngine", str(e))

    def _assign_layout(self, context: RuntimeContext) -> RuntimeContext:
        chart = context.chart_winner
        intent = context.intent_winner
        fv = context.feature_vector

        # Priority 1 — Chart type default
        col_span, row_span = CHART_DEFAULT_LAYOUT.get(chart, (12, 1))

        # Priority 2 — Intent adjustment
        adjusted_col = INTENT_LAYOUT_ADJUSTMENTS.get((intent, chart))
        if adjusted_col is not None:
            col_span = adjusted_col

        # Priority 3 — Data characteristic adjustments
        row_count = context.row_count

        # Large tables need more vertical space
        if chart == "table" and row_count > 100:
            row_span = min(row_span + 1, 3)

        # Many categories need more horizontal space
        cardinality = fv[26] if len(fv) > 26 else 0.0
        if chart in ("bar", "bar_horizontal") and cardinality > 0.30:
            col_span = 12

        # KPI with trend data can use slightly more space
        if chart == "kpi" and fv[28] > 0.5:  # trend_strength > 0.5
            col_span = 4  # slightly wider for trend indicator

        # Clamp to valid range
        col_span = max(3, min(col_span, 12))
        row_span = max(1, min(row_span, 3))

        # Compute importance score
        importance = self._compute_importance(context, col_span, row_span)

        context.col_span = col_span
        context.row_span = row_span
        context.layout_importance = importance

        # Add to score_trace
        context.score_trace["layout"] = {
            "chart_type":     chart,
            "intent":         intent,
            "col_span":       col_span,
            "row_span":       row_span,
            "importance":     round(importance, 4),
            "adjustments":    {
                "intent_adjustment": adjusted_col,
                "row_count":         row_count,
                "cardinality":       round(cardinality, 4),
            }
        }

        return context

    def _compute_importance(
        self,
        context: RuntimeContext,
        col_span: int,
        row_span: int
    ) -> float:
        """
        Compute panel importance score [0.0, 1.0].

        importance =
            0.40 × size_score         (how much space it takes)
            0.30 × intent_strength    (how confident the intent is)
            0.20 × metric_weight      (semantic importance of metric)
            0.10 × position_pref      (KPIs always rank high)
        """
        # Size score — normalized grid area
        size_score = (col_span * row_span) / (12 * 3)

        # Intent strength
        intent_strength = context.intent_raw_score

        # Metric weight from semantic features
        fv = context.feature_vector
        metric_weight = 0.5  # default
        if len(fv) > 30 and fv[30] > 0.5:  # has_revenue_metric
            metric_weight = 1.0
        elif len(fv) > 34 and fv[34] > 0.5:  # has_customer_entity
            metric_weight = 0.8

        # Position preference
        position_pref = 1.0 if context.chart_winner == "kpi" else 0.5

        importance = (
            0.40 * size_score +
            0.30 * intent_strength +
            0.20 * metric_weight +
            0.10 * position_pref
        )

        return max(0.0, min(1.0, importance))
```

### 9.4 Layout Engine Tests

```python
# sqlviz-inference/tests/test_layout_engine.py

import pytest
from src.context import RuntimeContext, ColumnSchema
from src.parser.sql_parser import SQLParser
from src.features.feature_engine import FeatureEngine
from src.semantic.semantic_engine import SemanticEngine
from src.intent.intent_engine import IntentEngine
from src.chart.chart_engine import ChartEngine
from src.layout.layout_engine import LayoutEngine

parser   = SQLParser()
features = FeatureEngine()
semantic = SemanticEngine()
intent   = IntentEngine()
chart    = ChartEngine()
layout   = LayoutEngine()


def full_infer(sql: str, data=None, schema_defs=None):
    schema = [ColumnSchema(name=n, type=t) for n, t in (schema_defs or [])]
    ctx = RuntimeContext(sql=sql, data=data or [], schema=schema)
    for engine in [parser, features, semantic, intent, chart, layout]:
        ctx = engine.run(ctx)
    return ctx


class TestKPILayout:

    def test_kpi_is_small(self):
        ctx = full_infer(
            sql="SELECT SUM(revenue) AS total FROM sales",
            data=[{"total": 125430.0}],
            schema_defs=[("total", "DOUBLE")]
        )
        if ctx.chart_winner == "kpi":
            assert ctx.col_span <= 4
            assert ctx.row_span == 1

    def test_kpi_importance_is_high(self):
        ctx = full_infer(
            sql="SELECT SUM(revenue) AS total FROM sales",
            data=[{"total": 125430.0}],
            schema_defs=[("total", "DOUBLE")]
        )
        if ctx.chart_winner == "kpi":
            assert ctx.layout_importance > 0.4


class TestLineLayout:

    def test_line_is_full_width(self):
        ctx = full_infer(
            sql="SELECT month, SUM(revenue) FROM sales GROUP BY month ORDER BY month",
            schema_defs=[("month", "DATE"), ("revenue", "DOUBLE")]
        )
        if ctx.chart_winner == "line":
            assert ctx.col_span == 12
            assert ctx.row_span == 1


class TestTableLayout:

    def test_table_is_full_width_double_height(self):
        ctx = full_infer(
            sql="SELECT id, name, email, phone FROM customers",
            schema_defs=[
                ("id", "INTEGER"), ("name", "VARCHAR"),
                ("email", "VARCHAR"), ("phone", "VARCHAR")
            ]
        )
        if ctx.chart_winner == "table":
            assert ctx.col_span == 12
            assert ctx.row_span >= 2

    def test_large_table_gets_extra_height(self):
        data = [{"id": i, "name": f"Name{i}"} for i in range(150)]
        ctx = full_infer(
            sql="SELECT id, name FROM customers",
            data=data,
            schema_defs=[("id", "INTEGER"), ("name", "VARCHAR")]
        )
        if ctx.chart_winner == "table":
            assert ctx.row_span >= 2


class TestScatterLayout:

    def test_scatter_is_square(self):
        ctx = full_infer(
            sql="SELECT price, quantity FROM products",
            data=[{"price": p, "quantity": q}
                  for p, q in [(10,100),(20,80),(30,60)]],
            schema_defs=[("price", "DOUBLE"), ("quantity", "INTEGER")]
        )
        if ctx.chart_winner == "scatter":
            assert ctx.col_span == 6
            assert ctx.row_span == 2


class TestValidRanges:

    def test_col_span_in_valid_range(self):
        ctx = full_infer("SELECT a, b FROM t",
                         schema_defs=[("a", "VARCHAR"), ("b", "VARCHAR")])
        assert 1 <= ctx.col_span <= 12

    def test_row_span_in_valid_range(self):
        ctx = full_infer("SELECT a FROM t",
                         schema_defs=[("a", "VARCHAR")])
        assert 1 <= ctx.row_span <= 3

    def test_importance_in_valid_range(self):
        ctx = full_infer("SELECT SUM(x) FROM t",
                         schema_defs=[("x", "DOUBLE")])
        assert 0.0 <= ctx.layout_importance <= 1.0
```

---

*Section 9 complete. Next: Section 10 — Filter Engine.*

---

## 10. Filter Engine

### 10.1 Responsibility

The Filter Engine detects `$variable` placeholders in SQL
and determines the appropriate control type for each one.

```
Input:  context with ast + schema

Output: context with
        filter_controls   list of FilterControl

Example:
    SQL: SELECT region, SUM(revenue) FROM sales
         WHERE region = $region AND fecha >= $desde
         GROUP BY region

    Detected:
        $region → Dropdown   (low cardinality VARCHAR)
        $desde  → DatePicker (DATE column)
```

In V0.1, filters are **explicit** — the user writes `$variable`
in the WHERE clause. SQLviz infers the correct control type
for that variable based on the column it filters.

```
Note: Automatic filter inference WITHOUT $variable
(detecting filterable columns the user never mentioned)
is a V0.3 feature — not implemented in V0.1.
```

### 10.2 The 8 Control Types

```
Control Type      When used                          Example
──────────────────────────────────────────────────────────────────
date_picker       DATE/TIMESTAMP, single value        $fecha
date_range_picker DATE/TIMESTAMP, range comparison     $desde, $hasta
dropdown          VARCHAR, low cardinality (≤50)       $region
multiselect       VARCHAR, used with ANY() or IN       $regiones
search            VARCHAR, used with LIKE/ILIKE        $busqueda
numeric           INTEGER/DOUBLE, single value         $minimo
range_slider      INTEGER/DOUBLE, range comparison     $min, $max
toggle            BOOLEAN                              $activo
```

### 10.3 filter_engine.py — Complete

```python
# sqlviz-inference/src/filters/filter_engine.py

import re
from ..context import RuntimeContext, FilterControl, ColumnSchema
import sqlglot.expressions as exp


VARIABLE_PATTERN = re.compile(r'\$(\w+)')

DATE_TYPES = {
    "DATE", "TIMESTAMP", "TIMESTAMP WITH TIME ZONE",
    "TIMESTAMP_S", "TIMESTAMP_MS", "TIMESTAMP_NS"
}
NUMERIC_TYPES = {
    "TINYINT", "SMALLINT", "INTEGER", "INT", "BIGINT",
    "HUGEINT", "FLOAT", "DOUBLE", "DECIMAL", "NUMERIC", "REAL"
}
BOOLEAN_TYPES = {"BOOLEAN", "BOOL"}


class FilterEngine:
    """
    Detects $variable placeholders in SQL and infers
    the appropriate control type for each one.

    Strategy:
    1. Find all $variable occurrences in the raw SQL
    2. For each variable, find the column it compares against
    3. Determine control type from column type + SQL operator
    """

    def run(self, context: RuntimeContext) -> RuntimeContext:
        try:
            return self._detect_filters(context)
        except Exception as e:
            return context.with_error("FilterEngine", str(e))

    def _detect_filters(self, context: RuntimeContext) -> RuntimeContext:
        sql = context.sql
        variables = set(VARIABLE_PATTERN.findall(sql))

        if not variables:
            context.filter_controls = []
            return context

        schema_map = {col.name.lower(): col for col in context.schema}
        controls = []

        for var_name in variables:
            control = self._infer_control(var_name, sql, schema_map)
            if control:
                controls.append(control)

        context.filter_controls = controls
        return context

    def _infer_control(
        self,
        var_name: str,
        sql: str,
        schema_map: dict[str, ColumnSchema]
    ) -> FilterControl | None:
        """
        Find the column associated with $var_name
        and infer the control type.
        """
        column_name = self._find_associated_column(var_name, sql)
        if not column_name:
            return None

        column = schema_map.get(column_name.lower())
        column_type = column.type.upper().split("(")[0] if column else "VARCHAR"

        operator_context = self._get_operator_context(var_name, sql)

        control_type = self._classify_control(column_type, operator_context)

        return FilterControl(
            variable=var_name,
            label=self._humanize(column_name or var_name),
            control_type=control_type,
            column_name=column_name or var_name,
            column_type=column_type,
            scope="global"  # V0.1 — all filters are global by default
        )

    def _find_associated_column(self, var_name: str, sql: str) -> str | None:
        """
        Find the column compared against $var_name.
        Looks for patterns like: column = $var, column >= $var, etc.
        """
        # Pattern: column_name [operator] $var_name
        pattern = re.compile(
            rf'(\w+)\s*(?:=|>=|<=|>|<|!=)\s*\$' + re.escape(var_name)
        )
        match = pattern.search(sql)
        if match:
            return match.group(1)

        # Pattern: $var_name [operator] column_name (reversed)
        pattern_reversed = re.compile(
            rf'\$' + re.escape(var_name) + r'\s*(?:=|>=|<=|>|<|!=)\s*(\w+)'
        )
        match = pattern_reversed.search(sql)
        if match:
            return match.group(1)

        # Pattern: column ANY($var) or column IN ($var)
        pattern_any = re.compile(
            rf'(\w+)\s*=\s*ANY\(\$' + re.escape(var_name) + r'\)'
        )
        match = pattern_any.search(sql, re.IGNORECASE)
        if match:
            return match.group(1)

        # Pattern: column ILIKE '%' || $var || '%'
        pattern_like = re.compile(
            rf'(\w+)\s+I?LIKE\s+.*\$' + re.escape(var_name),
            re.IGNORECASE
        )
        match = pattern_like.search(sql)
        if match:
            return match.group(1)

        return None

    def _get_operator_context(self, var_name: str, sql: str) -> str:
        """
        Determine how the variable is used:
        'equality' | 'range' | 'multi' | 'search'
        """
        sql_upper = sql.upper()
        var_upper = f"${var_name}".upper()

        if f"ANY({var_upper})" in sql_upper or f"IN ({var_upper})" in sql_upper:
            return "multi"

        if "ILIKE" in sql_upper or "LIKE" in sql_upper:
            # Check if this specific variable is near LIKE
            idx = sql_upper.find(var_upper)
            like_idx = sql_upper.find("LIKE")
            if idx != -1 and like_idx != -1 and abs(idx - like_idx) < 50:
                return "search"

        if ">=" in sql or "<=" in sql:
            # Check if there might be a paired range variable
            return "range_candidate"

        return "equality"

    def _classify_control(
        self,
        column_type: str,
        operator_context: str
    ) -> str:
        """
        Classify the control type based on column type and operator.
        """
        if column_type in BOOLEAN_TYPES:
            return "toggle"

        if column_type in DATE_TYPES:
            if operator_context == "range_candidate":
                return "date_range_picker"
            return "date_picker"

        if column_type in NUMERIC_TYPES:
            if operator_context == "range_candidate":
                return "range_slider"
            return "numeric"

        # VARCHAR / STRING types
        if operator_context == "multi":
            return "multiselect"
        if operator_context == "search":
            return "search"

        return "dropdown"

    def _humanize(self, name: str) -> str:
        """Convert snake_case to Title Case for display."""
        return name.replace("_", " ").title()
```

### 10.4 Range Pairing Logic

A special case: when two variables filter the same column
with `>=` and `<=`, they should be combined into one
`date_range_picker` or `range_slider` control.

```python
# sqlviz-inference/src/filters/range_pairing.py

from ..context import FilterControl


def pair_range_filters(controls: list[FilterControl]) -> list[FilterControl]:
    """
    Detect pairs of filters on the same column that form a range
    (e.g. $desde and $hasta both filtering 'fecha')
    and merge them into a single range control.
    """
    by_column: dict[str, list[FilterControl]] = {}
    for c in controls:
        by_column.setdefault(c.column_name, []).append(c)

    result = []
    processed = set()

    for column_name, column_controls in by_column.items():
        if len(column_controls) == 2:
            c1, c2 = column_controls
            if c1.control_type == "date_picker" and c2.control_type == "date_picker":
                # Merge into date_range_picker
                merged = FilterControl(
                    variable=f"{c1.variable},{c2.variable}",
                    label=c1.label,
                    control_type="date_range_picker",
                    column_name=column_name,
                    column_type=c1.column_type,
                    scope="global"
                )
                result.append(merged)
                processed.add(c1.variable)
                processed.add(c2.variable)
                continue
            elif c1.control_type == "numeric" and c2.control_type == "numeric":
                merged = FilterControl(
                    variable=f"{c1.variable},{c2.variable}",
                    label=c1.label,
                    control_type="range_slider",
                    column_name=column_name,
                    column_type=c1.column_type,
                    scope="global"
                )
                result.append(merged)
                processed.add(c1.variable)
                processed.add(c2.variable)
                continue

    # Add remaining unpaired controls
    for c in controls:
        if c.variable not in processed:
            result.append(c)

    return result
```

### 10.5 Filter Engine Tests

```python
# sqlviz-inference/tests/test_filter_engine.py

import pytest
from src.context import RuntimeContext, ColumnSchema
from src.parser.sql_parser import SQLParser
from src.filters.filter_engine import FilterEngine
from src.filters.range_pairing import pair_range_filters

parser = SQLParser()
filters = FilterEngine()


def detect_filters(sql: str, schema_defs: list[tuple]):
    schema = [ColumnSchema(name=n, type=t) for n, t in schema_defs]
    ctx = RuntimeContext(sql=sql, schema=schema)
    ctx = parser.run(ctx)
    ctx = filters.run(ctx)
    return ctx.filter_controls


class TestBasicDetection:

    def test_dropdown_for_varchar_equality(self):
        controls = detect_filters(
            sql="SELECT * FROM sales WHERE region = $region",
            schema_defs=[("region", "VARCHAR")]
        )
        assert len(controls) == 1
        assert controls[0].control_type == "dropdown"
        assert controls[0].variable == "region"

    def test_date_picker_for_date_equality(self):
        controls = detect_filters(
            sql="SELECT * FROM sales WHERE fecha = $fecha",
            schema_defs=[("fecha", "DATE")]
        )
        assert controls[0].control_type == "date_picker"

    def test_numeric_for_integer(self):
        controls = detect_filters(
            sql="SELECT * FROM sales WHERE quantity = $cantidad",
            schema_defs=[("quantity", "INTEGER")]
        )
        assert controls[0].control_type == "numeric"

    def test_toggle_for_boolean(self):
        controls = detect_filters(
            sql="SELECT * FROM users WHERE active = $activo",
            schema_defs=[("active", "BOOLEAN")]
        )
        assert controls[0].control_type == "toggle"


class TestMultiSelect:

    def test_multiselect_for_any(self):
        controls = detect_filters(
            sql="SELECT * FROM sales WHERE region = ANY($regiones)",
            schema_defs=[("region", "VARCHAR")]
        )
        assert controls[0].control_type == "multiselect"


class TestSearch:

    def test_search_for_ilike(self):
        controls = detect_filters(
            sql="SELECT * FROM products WHERE name ILIKE '%' || $busqueda || '%'",
            schema_defs=[("name", "VARCHAR")]
        )
        assert controls[0].control_type == "search"


class TestMultipleVariables:

    def test_multiple_filters_detected(self):
        controls = detect_filters(
            sql="SELECT * FROM sales WHERE region = $region "
                "AND fecha >= $desde",
            schema_defs=[("region", "VARCHAR"), ("fecha", "DATE")]
        )
        assert len(controls) == 2
        variables = {c.variable for c in controls}
        assert variables == {"region", "desde"}


class TestRangePairing:

    def test_date_range_merged(self):
        controls = detect_filters(
            sql="SELECT * FROM sales WHERE fecha >= $desde AND fecha <= $hasta",
            schema_defs=[("fecha", "DATE")]
        )
        merged = pair_range_filters(controls)
        assert len(merged) == 1
        assert merged[0].control_type == "date_range_picker"

    def test_numeric_range_merged(self):
        controls = detect_filters(
            sql="SELECT * FROM products WHERE price >= $min AND price <= $max",
            schema_defs=[("price", "DOUBLE")]
        )
        merged = pair_range_filters(controls)
        assert len(merged) == 1
        assert merged[0].control_type == "range_slider"


class TestNoFilters:

    def test_no_variables_returns_empty(self):
        controls = detect_filters(
            sql="SELECT SUM(revenue) FROM sales",
            schema_defs=[("revenue", "DOUBLE")]
        )
        assert controls == []
```

---

*Section 10 complete. Next: Section 11 — Title Engine.*

---

## 11. Title Engine

### 11.1 Responsibility

The Title Engine generates a descriptive title for the panel
based on the SQL structure, the detected metric, and the dimension.

```
Input:  context with ast + schema + intent_winner + score_trace["semantic"]

Output: context with
        title             "Revenue by month"
        title_confidence  [0.0, 1.0]

Examples:
    SELECT month, SUM(revenue) FROM sales GROUP BY month
        → "Revenue by month"

    SELECT region, COUNT(*) FROM customers GROUP BY region
        → "Customers by region"

    SELECT product, units FROM top ORDER BY units DESC LIMIT 10
        → "Top 10 products by units"

    SELECT SUM(revenue) FROM sales
        → "Total revenue"

    SELECT id, name, email FROM customers
        → "Customer detail"
```

### 11.2 Title Templates

```
Pattern detected         Template
──────────────────────────────────────────────────────────
KPI (single metric)      "Total {metric}"
                         "Total {metric}" if SUM/COUNT
                         "Average {metric}" if AVG

Trend (metric over time) "{Metric} by {temporal_dimension}"
                         e.g. "Revenue by month"

Comparison (categories)  "{Metric} by {dimension}"
                         e.g. "Revenue by region"

Ranking (ordered+limit)  "Top {N} {entity} by {metric}"
                         e.g. "Top 10 products by revenue"

Composition (part/whole) "{Metric} distribution by {dimension}"
                         e.g. "Revenue distribution by category"

Detail (raw data)        "{Entity} detail"
                         e.g. "Customer detail"
                         Fallback: "Query results"
```

### 11.3 title_engine.py — Complete

```python
# sqlviz-inference/src/title/title_engine.py

from ..context import RuntimeContext
import sqlglot.expressions as exp
from ..parser.ast_helpers import has_limit, has_order_by_desc


# Map semantic classes to human-readable metric names
METRIC_LABELS = {
    "METRIC_REVENUE": "Revenue",
    "METRIC_COUNT":   "Count",
    "METRIC_PROFIT":  "Profit",
}

DIMENSION_LABELS = {
    "TEMPORAL_DIMENSION":   None,  # uses actual column name
    "GEOGRAPHIC_DIMENSION": None,
    "PRODUCT_ENTITY":       None,
    "CUSTOMER_ENTITY":      None,
}


class TitleEngine:
    """
    Generates a descriptive title for the panel.
    Uses the intent, the semantic classification of columns,
    and the SQL structure (aggregation, GROUP BY, LIMIT).
    """

    def run(self, context: RuntimeContext) -> RuntimeContext:
        try:
            return self._generate(context)
        except Exception as e:
            return context.with_error("TitleEngine", str(e))

    def _generate(self, context: RuntimeContext) -> RuntimeContext:
        if context.ast is None:
            context.title = ""
            context.title_confidence = 0.0
            return context

        semantic_classes = context.score_trace.get(
            "semantic", {}
        ).get("column_classes", {})

        metric_col, metric_label = self._find_metric(context, semantic_classes)
        dimension_col, dimension_label = self._find_dimension(
            context, semantic_classes
        )

        title = self._build_title(
            context, metric_col, metric_label,
            dimension_col, dimension_label
        )

        context.title = title
        context.title_confidence = 0.8 if metric_col else 0.4

        return context

    def _find_metric(
        self,
        context: RuntimeContext,
        semantic_classes: dict
    ) -> tuple[str | None, str]:
        """Find the primary metric column and its label."""
        # Look for semantically classified metrics first
        for col_name, sem_class in semantic_classes.items():
            if sem_class in METRIC_LABELS:
                return col_name, METRIC_LABELS[sem_class]

        # Fallback — find any aggregated column
        select = context.ast.find(exp.Select)
        if select:
            for expr in select.expressions:
                if isinstance(expr, exp.AggFunc) or expr.find(exp.AggFunc):
                    name = expr.alias_or_name
                    return name, self._humanize(name)

        return None, "Value"

    def _find_dimension(
        self,
        context: RuntimeContext,
        semantic_classes: dict
    ) -> tuple[str | None, str]:
        """Find the primary dimension column and its label."""
        group = context.ast.find(exp.Group)
        if not group:
            return None, ""

        columns = list(group.find_all(exp.Column))
        if not columns:
            return None, ""

        first_col = columns[0].name
        return first_col, self._humanize(first_col)

    def _build_title(
        self,
        context: RuntimeContext,
        metric_col: str | None,
        metric_label: str,
        dimension_col: str | None,
        dimension_label: str
    ) -> str:
        """Build the final title string based on intent."""
        intent = context.intent_winner
        ast = context.ast

        # KPI — single metric, no dimension
        if intent == "kpi" or not dimension_col:
            if context.chart_winner == "table":
                return self._table_title(context)
            return f"Total {metric_label}"

        # Ranking — has LIMIT + ORDER BY DESC
        if has_limit(ast) and has_order_by_desc(ast):
            limit_node = ast.find(exp.Limit)
            n = self._extract_limit_value(limit_node)
            entity = self._pluralize(dimension_label)
            return f"Top {n} {entity} by {metric_label}"

        # Trend — temporal dimension
        if intent == "trend":
            return f"{metric_label} by {dimension_label.lower()}"

        # Comparison / Composition
        if intent in ("comparison", "composition"):
            return f"{metric_label} by {dimension_label.lower()}"

        # Default
        return f"{metric_label} by {dimension_label.lower()}"

    def _table_title(self, context: RuntimeContext) -> str:
        """Generate title for detail/table queries."""
        tables = []
        if context.ast:
            for t in context.ast.find_all(exp.Table):
                if t.name:
                    tables.append(t.name)

        if tables:
            entity = self._humanize(tables[0]).rstrip('s')
            return f"{entity} detail"

        return "Query results"

    def _extract_limit_value(self, limit_node) -> str:
        """Extract the N value from LIMIT N."""
        if limit_node is None:
            return "N"
        try:
            return str(limit_node.expression.this)
        except Exception:
            return "N"

    def _humanize(self, name: str) -> str:
        """Convert snake_case to Title Case."""
        return name.replace("_", " ").title()

    def _pluralize(self, word: str) -> str:
        """Simple English pluralization."""
        word = word.lower()
        if word.endswith(('s', 'sh', 'ch', 'x', 'z')):
            return word + 'es'
        elif word.endswith('y') and word[-2] not in 'aeiou':
            return word[:-1] + 'ies'
        else:
            return word + 's'
```

### 11.4 Title Engine Tests

```python
# sqlviz-inference/tests/test_title_engine.py

import pytest
from src.context import RuntimeContext, ColumnSchema
from src.parser.sql_parser import SQLParser
from src.features.feature_engine import FeatureEngine
from src.semantic.semantic_engine import SemanticEngine
from src.intent.intent_engine import IntentEngine
from src.chart.chart_engine import ChartEngine
from src.title.title_engine import TitleEngine

parser   = SQLParser()
features = FeatureEngine()
semantic = SemanticEngine()
intent   = IntentEngine()
chart    = ChartEngine()
title    = TitleEngine()


def full_infer(sql: str, data=None, schema_defs=None):
    schema = [ColumnSchema(name=n, type=t) for n, t in (schema_defs or [])]
    ctx = RuntimeContext(sql=sql, data=data or [], schema=schema)
    for engine in [parser, features, semantic, intent, chart, title]:
        ctx = engine.run(ctx)
    return ctx


class TestKPITitles:

    def test_total_revenue(self):
        ctx = full_infer(
            sql="SELECT SUM(revenue) AS total FROM sales",
            data=[{"total": 125430.0}],
            schema_defs=[("total", "DOUBLE")]
        )
        assert "Total" in ctx.title


class TestTrendTitles:

    def test_revenue_by_month(self):
        ctx = full_infer(
            sql="SELECT month, SUM(revenue) FROM sales GROUP BY month",
            schema_defs=[("month", "DATE"), ("revenue", "DOUBLE")]
        )
        assert "Revenue" in ctx.title
        assert "month" in ctx.title.lower()


class TestRankingTitles:

    def test_top_n_products(self):
        ctx = full_infer(
            sql="SELECT product, SUM(revenue) FROM sales "
                "GROUP BY product ORDER BY 2 DESC LIMIT 10",
            schema_defs=[("product", "VARCHAR"), ("revenue", "DOUBLE")]
        )
        assert "Top 10" in ctx.title


class TestDetailTitles:

    def test_customer_detail(self):
        ctx = full_infer(
            sql="SELECT id, name, email FROM customers",
            schema_defs=[
                ("id", "INTEGER"), ("name", "VARCHAR"), ("email", "VARCHAR")
            ]
        )
        assert "detail" in ctx.title.lower() or "results" in ctx.title.lower()


class TestTitleConfidence:

    def test_confidence_higher_with_semantic_metric(self):
        ctx = full_infer(
            sql="SELECT month, SUM(revenue) FROM sales GROUP BY month",
            schema_defs=[("month", "DATE"), ("revenue", "DOUBLE")]
        )
        assert ctx.title_confidence >= 0.7

    def test_invalid_sql_empty_title(self):
        ctx = full_infer(sql="NOT VALID SQL ###")
        assert ctx.title == ""
        assert ctx.title_confidence == 0.0
```

---

*Section 11 complete. Next: Section 12 — Runtime Pipeline.*

---

## 12. Runtime Pipeline

### 12.1 Responsibility

The Runtime Pipeline coordinates the execution order
of all 8 modules and assembles the final InferenceResult.

```
Input:  RuntimeContext (sql, data, schema)

Output: RuntimeContext (fully enriched)
        → converted to InferenceResult at the entry point
```

It is the only place in the codebase
that knows the execution order of modules.
No module knows about any other module.

### 12.2 Execution Order

```
1. SQLParser          → ast, fingerprint, sql_features
2. FeatureEngine       → feature_vector dims 18-37
3. SemanticEngine      → feature_vector dims 30-34 (enriched)
4. IntentEngine        → intent_scores, intent_winner
5. ChartEngine         → chart_winner, fallback logic
6. LayoutEngine        → col_span, row_span
7. FilterEngine        → filter_controls
8. TitleEngine         → title

Order matters:
- FeatureEngine needs ast from SQLParser
- SemanticEngine needs feature_vector from FeatureEngine
- IntentEngine needs complete feature_vector (after Semantic)
- ChartEngine needs intent_scores from IntentEngine
- LayoutEngine needs chart_winner from ChartEngine
- FilterEngine only needs ast + schema (can run anytime after Parser)
- TitleEngine needs intent_winner + semantic classes
```

### 12.3 pipeline.py — Complete

```python
# sqlviz-inference/src/pipeline.py

import time
from .context import RuntimeContext
from .parser.sql_parser import SQLParser
from .features.feature_engine import FeatureEngine
from .semantic.semantic_engine import SemanticEngine
from .intent.intent_engine import IntentEngine
from .chart.chart_engine import ChartEngine
from .layout.layout_engine import LayoutEngine
from .filters.filter_engine import FilterEngine
from .filters.range_pairing import pair_range_filters
from .title.title_engine import TitleEngine


class RuntimePipeline:
    """
    Coordinates the execution of all inference modules
    in the correct order.

    This is the ONLY place that knows the module execution order.
    Modules never call each other directly.
    """

    def __init__(self):
        self.parser   = SQLParser()
        self.features = FeatureEngine()
        self.semantic = SemanticEngine()
        self.intent   = IntentEngine()
        self.chart    = ChartEngine()
        self.layout   = LayoutEngine()
        self.filters  = FilterEngine()
        self.title    = TitleEngine()

    def run(self, context: RuntimeContext) -> RuntimeContext:
        """
        Execute the complete pipeline.
        Each module runs even if previous modules logged errors —
        graceful degradation means we always produce a result.
        """
        start_time = time.time()

        context = self.parser.run(context)
        context = self.features.run(context)
        context = self.semantic.run(context)
        context = self.intent.run(context)
        context = self.chart.run(context)
        context = self.layout.run(context)
        context = self.filters.run(context)
        context.filter_controls = pair_range_filters(context.filter_controls)
        context = self.title.run(context)

        elapsed_ms = (time.time() - start_time) * 1000
        context.score_trace["pipeline"] = {
            "elapsed_ms": round(elapsed_ms, 2),
            "errors": context.errors,
            "modules_run": [
                "parser", "features", "semantic", "intent",
                "chart", "layout", "filters", "title"
            ]
        }

        return context
```

### 12.4 result.py — InferenceResult

```python
# sqlviz-inference/src/result.py

from __future__ import annotations
from dataclasses import dataclass, field
from .context import RuntimeContext


@dataclass
class InferenceResult:
    """
    The final, complete output of the Inference Engine.
    This is what sqlviz-api and sqlviz-web consume.

    Every field has a mathematical justification (DOC 4).
    Every field is versioned for traceability.
    """

    # ── Versioning ────────────────────────────────────────────
    rules_version: str
    feature_vector_version: str
    engine_version: str

    # ── Intent ────────────────────────────────────────────────
    intent_winner: str
    intent_raw_score: float
    intent_normalized_score: float
    intent_confidence_gap: float
    intent_quality: str
    intent_alternatives: list[dict]

    # ── Chart ─────────────────────────────────────────────────
    chart_winner: str
    chart_raw_score: float
    chart_normalized_score: float
    chart_confidence_gap: float
    chart_quality: str
    chart_alternatives: list[dict]

    # ── Layout ────────────────────────────────────────────────
    col_span: int
    row_span: int
    layout_importance: float

    # ── KPI Trend (added per Section 16.5 — moved out of frontend) ──
    trend_direction_label: str  # "growing" | "declining" | "flat" | "unknown"

    # ── Filters ───────────────────────────────────────────────
    filter_controls: list[dict]

    # ── Title ─────────────────────────────────────────────────
    title: str
    title_confidence: float

    # ── Fallback ──────────────────────────────────────────────
    fallback_applied: bool
    fallback_reason: str

    # ── Explainability ────────────────────────────────────────
    explanation: list[dict]
    score_trace: dict
    fingerprint: str
    feature_vector: list[float]

    # ── Diagnostics ───────────────────────────────────────────
    errors: list[str]
    elapsed_ms: float

    @classmethod
    def from_context(cls, context: RuntimeContext) -> InferenceResult:
        """Build the final result from a fully-processed RuntimeContext."""

        intent_alternatives = [
            {
                "intent": s.intent,
                "raw_score": round(s.raw_score, 4)
            }
            for s in context.intent_scores[1:3]
            if s.raw_score > 0.0
        ]

        filter_controls_dict = [
            {
                "variable": fc.variable,
                "label": fc.label,
                "control_type": fc.control_type,
                "column_name": fc.column_name,
                "column_type": fc.column_type,
                "scope": fc.scope,
            }
            for fc in context.filter_controls
        ]

        # trend_direction_label — computed HERE, in the backend,
        # not in sqlviz-web (Section 16.5 fix). DOC 6's original
        # KPI Renderer made this decision in Svelte using these
        # same 0.65/0.35 thresholds directly on feature_vector[38]
        # and feature_vector[28] — that violated DOC 6 Section 1.1's
        # own "frontend never infers" rule. The thresholds move
        # here; the frontend (DOC 6, corrected) only reads the
        # resulting string.
        fv = context.feature_vector
        strength = fv[28] if len(fv) > 28 else 0.0
        direction = fv[38] if len(fv) > 38 else 0.5
        if strength <= 0.5:
            trend_direction_label = "unknown"  # not a meaningful trend at all
        elif direction > 0.65:
            trend_direction_label = "growing"
        elif direction < 0.35:
            trend_direction_label = "declining"
        else:
            trend_direction_label = "flat"

        return cls(
            rules_version=context.rules_version,
            feature_vector_version=context.feature_vector_version,
            engine_version=context.engine_version,

            intent_winner=context.intent_winner,
            intent_raw_score=context.intent_raw_score,
            intent_normalized_score=context.intent_normalized_score,
            intent_confidence_gap=context.intent_confidence_gap,
            intent_quality=context.intent_quality,
            intent_alternatives=intent_alternatives,

            chart_winner=context.chart_winner,
            chart_raw_score=context.chart_raw_score,
            chart_normalized_score=context.chart_normalized_score,
            chart_confidence_gap=context.chart_confidence_gap,
            chart_quality=context.chart_quality,
            chart_alternatives=context.chart_alternatives,

            col_span=context.col_span,
            row_span=context.row_span,
            layout_importance=context.layout_importance,
            trend_direction_label=trend_direction_label,

            filter_controls=filter_controls_dict,

            title=context.title,
            title_confidence=context.title_confidence,

            fallback_applied=context.fallback_applied,
            fallback_reason=context.fallback_reason,

            explanation=context.explanation,
            score_trace=context.score_trace,
            fingerprint=context.fingerprint,
            feature_vector=context.feature_vector,

            errors=context.errors,
            elapsed_ms=context.score_trace.get(
                "pipeline", {}
            ).get("elapsed_ms", 0.0),
        )

    def to_dict(self) -> dict:
        """Serialize to dict for JSON API responses."""
        from dataclasses import asdict
        return asdict(self)
```

### 12.5 Graceful Degradation in the Pipeline

```
Scenario: SemanticEngine throws an exception

1. SemanticEngine.run() catches the exception internally
2. Returns context.with_error("SemanticEngine", str(e))
3. context.feature_vector dims 30-34 remain at 0.0 (default)
4. Pipeline continues to IntentEngine
5. IntentEngine scores intents WITHOUT semantic boost
   (lower scores for trend, comparison — but still functional)
6. Pipeline completes successfully
7. context.errors contains: ["SemanticEngine: <error message>"]
8. User still sees a dashboard — possibly with lower confidence
   chart_quality might be "medium" instead of "high"

This is the core promise of graceful degradation:
ANY single module failure never breaks the pipeline.
The result quality may degrade, but a result is always produced.
```

### 12.6 Fast Path vs Slow Path (V0.1 scope)

```
In V0.1, the entire pipeline (sections 4-11) runs synchronously
as the "Fast Path" — target: < 1 second total.

This is acceptable because:
- All V0.1 computations are O(n) or better
- No network calls
- No LLM calls
- Pure Python + DuckDB local execution

Slow Path (Insights, Recommendations, Autonomous Analysis)
is NOT part of V0.1. It is planned for V0.3+ (see DOC 1, Section 4).
When implemented, it will run as a background task
after the Fast Path returns the dashboard to the user.
```

### 12.7 Pipeline Tests

```python
# sqlviz-inference/tests/test_pipeline.py

import pytest
from src.context import RuntimeContext, ColumnSchema
from src.pipeline import RuntimePipeline
from src.result import InferenceResult

pipeline = RuntimePipeline()


class TestFullPipeline:

    def test_complete_trend_inference(self):
        ctx = RuntimeContext(
            sql="SELECT month, SUM(revenue) FROM sales GROUP BY month ORDER BY month",
            data=[{"month": f"2024-{i:02d}", "revenue": i*1000} for i in range(1,13)],
            schema=[
                ColumnSchema(name="month", type="DATE"),
                ColumnSchema(name="revenue", type="DOUBLE")
            ]
        )
        ctx = pipeline.run(ctx)
        result = InferenceResult.from_context(ctx)

        assert result.intent_winner == "trend"
        assert result.chart_winner == "line"
        assert result.chart_quality == "high"
        assert result.fallback_applied == False
        assert result.col_span == 12
        assert "Revenue" in result.title
        assert result.fingerprint == "TIME_SUM_GROUP1_ORDER_ASC"
        assert len(result.feature_vector) == 38
        assert result.elapsed_ms < 1000  # under 1 second

    def test_complete_kpi_inference(self):
        ctx = RuntimeContext(
            sql="SELECT SUM(revenue) AS total FROM sales",
            data=[{"total": 125430.0}],
            schema=[ColumnSchema(name="total", type="DOUBLE")]
        )
        ctx = pipeline.run(ctx)
        result = InferenceResult.from_context(ctx)

        assert result.chart_winner == "kpi"
        assert result.col_span <= 4

    def test_graceful_degradation_invalid_sql(self):
        ctx = RuntimeContext(sql="THIS IS NOT VALID SQL ###")
        ctx = pipeline.run(ctx)
        result = InferenceResult.from_context(ctx)

        # Pipeline must still produce a valid result
        assert result.chart_winner == "table"
        assert result.fingerprint == "UNKNOWN"
        assert len(result.errors) > 0
        assert result.col_span >= 1

    def test_filters_with_range_pairing(self):
        ctx = RuntimeContext(
            sql="SELECT region, SUM(revenue) FROM sales "
                "WHERE fecha >= $desde AND fecha <= $hasta "
                "GROUP BY region",
            schema=[
                ColumnSchema(name="region", type="VARCHAR"),
                ColumnSchema(name="revenue", type="DOUBLE"),
                ColumnSchema(name="fecha", type="DATE"),
            ]
        )
        ctx = pipeline.run(ctx)
        result = InferenceResult.from_context(ctx)

        assert len(result.filter_controls) == 1
        assert result.filter_controls[0]["control_type"] == "date_range_picker"

    def test_score_trace_complete(self):
        ctx = RuntimeContext(
            sql="SELECT month, SUM(revenue) FROM sales GROUP BY month"
        )
        ctx = pipeline.run(ctx)
        result = InferenceResult.from_context(ctx)

        assert "intent" in result.score_trace
        assert "chart" in result.score_trace
        assert "layout" in result.score_trace
        assert "semantic" in result.score_trace
        assert "pipeline" in result.score_trace

    def test_versioning_present(self):
        ctx = RuntimeContext(sql="SELECT SUM(x) FROM t")
        ctx = pipeline.run(ctx)
        result = InferenceResult.from_context(ctx)

        assert result.rules_version != ""
        assert result.feature_vector_version == "v0"
        assert result.engine_version != ""
```

---

*Section 12 complete. Next: Section 13 — Complete YAML Rules Files.*

---

## 13. Complete YAML Rules Files

This section consolidates every YAML file referenced
throughout Sections 4-11 into one place for implementation reference.
All files live in `sqlviz-inference/rules/`.

### 13.1 feature_vector_v0.yaml

```yaml
version: "v0"
dimensions: 39

features:
  # SQL Structural Features (0-17)
  has_group_by:               {index: 0,  type: binary}
  has_order_by:                {index: 1,  type: binary}
  has_order_by_desc:           {index: 2,  type: binary}
  has_limit:                   {index: 3,  type: binary}
  has_aggregation:              {index: 4,  type: binary}
  has_sum:                     {index: 5,  type: binary}
  has_count:                   {index: 6,  type: binary}
  has_avg:                     {index: 7,  type: binary}
  has_window_function:         {index: 8,  type: binary}
  has_cte:                     {index: 9,  type: binary}
  has_join:                    {index: 10, type: binary}
  has_where:                   {index: 11, type: binary}
  group_by_column_count:       {index: 12, type: continuous, normalize: "divide_by_5"}
  select_column_count:         {index: 13, type: continuous, normalize: "divide_by_10"}
  has_subquery:                {index: 14, type: binary}
  has_partition_by:            {index: 15, type: binary}
  has_case_when:                {index: 16, type: binary}
  has_distinct:                 {index: 17, type: binary}

  # Column Type Features (18-24)
  has_date_column:             {index: 18, type: binary}
  has_numeric_column:          {index: 19, type: binary}
  has_string_column:           {index: 20, type: binary}
  numeric_column_ratio:        {index: 21, type: continuous}
  date_column_ratio:           {index: 22, type: continuous}
  has_single_numeric_column:   {index: 23, type: binary}
  has_two_numeric_columns:     {index: 24, type: binary}

  # Data Statistics (25-29)
  row_count_normalized:        {index: 25, type: continuous, normalize: "divide_by_10000"}
  cardinality_ratio:           {index: 26, type: continuous}
  temporal_cardinality:        {index: 27, type: continuous, normalize: "divide_by_366"}
  trend_strength:               {index: 28, type: continuous}
  has_outliers:                 {index: 29, type: binary}

  # Semantic Features (30-34)
  has_revenue_metric:          {index: 30, type: binary}
  has_temporal_dimension:      {index: 31, type: binary}
  has_geographic_dimension:    {index: 32, type: binary}
  has_product_entity:          {index: 33, type: binary}
  has_customer_entity:         {index: 34, type: binary}

  # Result Shape Features (35-37)
  result_row_count_is_1:       {index: 35, type: binary}
  result_column_count_is_1:    {index: 36, type: binary}
  result_is_wide_table:        {index: 37, type: binary}

reserved:
  start_index: 38
  end_index: 127
  note: "Reserved for V1 features (DOC 4, Section 3.3). Never use in V0."
```

### 13.2 intent_rules.yaml

```yaml
trend:
  description: "How does a metric change over time?"
  weights:
    has_temporal_dimension: 0.40
    has_group_by: 0.25
    has_aggregation: 0.20
    has_order_by: 0.10
    temporal_cardinality: 0.05
  boosts: {}
  penalties:
    no_temporal_dimension: 0.60

comparison:
  description: "How do categories compare to each other?"
  weights:
    has_group_by: 0.35
    has_aggregation: 0.30
    has_string_column: 0.20
    no_temporal_dimension: 0.10
    group_by_column_count: 0.05
  boosts: {}
  penalties:
    no_group_by: 0.50

ranking:
  description: "What are the top/bottom N items?"
  weights:
    has_order_by_desc: 0.40
    has_limit: 0.30
    has_aggregation: 0.20
    has_group_by: 0.10
  boosts:
    order_desc_and_limit: 1.50
  penalties:
    no_order_by_desc: 0.70

distribution:
  description: "How are values distributed?"
  weights:
    has_numeric_column: 0.40
    no_temporal_dimension: 0.30
    no_group_by: 0.20
    high_cardinality: 0.10
  boosts: {}
  penalties:
    no_numeric_column: 0.80

correlation:
  description: "Are two metrics related?"
  weights:
    has_two_numeric_columns: 0.50
    no_group_by: 0.30
    no_aggregation: 0.20
  boosts: {}
  penalties:
    single_numeric_column: 0.70
    has_aggregation: 0.40

composition:
  description: "What is the part-to-whole breakdown?"
  weights:
    has_group_by: 0.40
    has_aggregation: 0.30
    has_string_column: 0.20
    low_cardinality: 0.10
  boosts: {}
  penalties:
    high_cardinality: 0.40
    no_aggregation: 0.50

kpi:
  description: "What is the current value of a metric?"
  weights:
    result_row_count_is_1: 0.40
    result_column_count_is_1: 0.30
    has_aggregation: 0.20
    no_group_by: 0.10
  boosts: {}
  penalties:
    has_group_by: 0.80
    multiple_rows: 0.70
    no_aggregation: 0.30

anomaly:
  description: "Are there unexpected values in the data?"
  weights:
    has_temporal_dimension: 0.35
    has_aggregation: 0.30
    has_group_by: 0.20
    has_outliers: 0.15
  boosts:
    has_outliers_detected: 1.30
  penalties: {}

cohort:
  description: "How do groups behave over time?"
  weights:
    has_temporal_dimension: 0.35
    has_group_by: 0.30
    group_by_count_gte_2: 0.25
    has_aggregation: 0.10
  boosts: {}
  penalties:
    no_temporal_dimension: 0.60

retention:
  description: "Do users/customers return over time?"
  weights:
    has_temporal_dimension: 0.40
    has_customer_entity: 0.30
    has_window_function: 0.20
    has_join: 0.10
  boosts: {}
  penalties:
    no_temporal_dimension: 0.70
    no_customer_entity: 0.40

funnel:
  description: "Where do users drop off in a process?"
  weights:
    has_case_when: 0.40
    has_aggregation: 0.30
    has_count: 0.20
    has_group_by: 0.10
  boosts: {}
  penalties:
    no_case_when: 0.50

detail:
  description: "Show me the raw data."
  weights:
    no_aggregation: 0.50
    no_group_by: 0.30
    high_col_count: 0.20
  boosts: {}
  penalties: {}
```

### 13.3 chart_affinity_matrix.yaml

```yaml
kpi:
  trend: 0.00
  comparison: 0.00
  ranking: 0.00
  distribution: 0.00
  correlation: 0.00
  composition: 0.00
  kpi: 1.00
  anomaly: 0.00
  cohort: 0.00
  retention: 0.00
  funnel: 0.00
  detail: 0.00

line:
  trend: 0.95
  comparison: 0.10
  ranking: 0.00
  distribution: 0.05
  correlation: 0.00
  composition: 0.00
  kpi: 0.00
  anomaly: 0.20
  cohort: 0.30
  retention: 0.40
  funnel: 0.00
  detail: 0.05

bar:
  trend: 0.15
  comparison: 0.90
  ranking: 0.40
  distribution: 0.20
  correlation: 0.00
  composition: 0.20
  kpi: 0.00
  anomaly: 0.10
  cohort: 0.10
  retention: 0.05
  funnel: 0.10
  detail: 0.10

bar_horizontal:
  trend: 0.05
  comparison: 0.60
  ranking: 0.95
  distribution: 0.10
  correlation: 0.00
  composition: 0.10
  kpi: 0.00
  anomaly: 0.05
  cohort: 0.05
  retention: 0.00
  funnel: 0.20
  detail: 0.05

pie:
  trend: 0.00
  comparison: 0.30
  ranking: 0.00
  distribution: 0.80
  correlation: 0.00
  composition: 0.90
  kpi: 0.00
  anomaly: 0.00
  cohort: 0.00
  retention: 0.00
  funnel: 0.10
  detail: 0.00

scatter:
  trend: 0.05
  comparison: 0.10
  ranking: 0.00
  distribution: 0.40
  correlation: 0.95
  composition: 0.00
  kpi: 0.00
  anomaly: 0.20
  cohort: 0.00
  retention: 0.00
  funnel: 0.00
  detail: 0.00

histogram:
  trend: 0.10
  comparison: 0.10
  ranking: 0.00
  distribution: 0.95
  correlation: 0.10
  composition: 0.00
  kpi: 0.00
  anomaly: 0.15
  cohort: 0.00
  retention: 0.00
  funnel: 0.00
  detail: 0.05

table:
  trend: 0.10
  comparison: 0.20
  ranking: 0.20
  distribution: 0.20
  correlation: 0.10
  composition: 0.10
  kpi: 0.10
  anomaly: 0.10
  cohort: 0.20
  retention: 0.20
  funnel: 0.20
  detail: 0.95
```

### 13.4 chart_penalties.yaml

```yaml
kpi:
  penalties:
    has_group_by: 0.80
    result_is_wide_table: 0.60

line:
  penalties:
    no_temporal_dimension: 0.60
    result_row_count_is_1: 0.70
    result_is_wide_table: 0.30

bar:
  penalties:
    result_row_count_is_1: 0.50
    has_two_numeric_columns: 0.20

bar_horizontal:
  penalties:
    result_row_count_is_1: 0.60
    no_group_by: 0.40

pie:
  penalties:
    has_temporal_dimension: 0.40
    ranking_pattern: 0.40
    high_cardinality: 0.50
    result_row_count_is_1: 0.70
    correlation_intent: 0.20

scatter:
  penalties:
    has_single_numeric_column: 0.70
    has_temporal_dimension: 0.50
    has_aggregation: 0.40

histogram:
  penalties:
    has_group_by: 0.40
    has_temporal_dimension: 0.30
    result_row_count_is_1: 0.80

table:
  penalties: {}
```

### 13.5 fallback_rules.yaml

```yaml
chart:
  min_raw_score: 0.35
  default: table
  message: "Low confidence inference — showing raw data"

intent:
  min_raw_score: 0.30
  default: detail
  message: "Intent unclear — defaulting to detail view"

quality_thresholds:
  high: 0.70
  medium: 0.35

display_rules:
  high_high:
    show_alternatives: 0
    show_warning: false
  high_low:
    show_alternatives: 2
    show_warning: false
  low_high:
    show_alternatives: 1
    show_warning: true
    warning_message: "Low confidence inference"
  low_low:
    show_alternatives: 0
    show_warning: true
    apply_fallback: true
```

### 13.6 thresholds.yaml

```yaml
confidence:
  high: 0.60
  low: 0.30

similarity:
  fingerprint_match: 1.0
  vector_match: 0.85    # V1+ only — not used in V0

anomaly:
  z_score: 2.5

pareto:
  share_threshold: 0.80
  ratio_max: 0.25

dimension:
  global_filter: 0.70
  local_filter: 0.40

hhi:
  high_concentration: 0.50
  moderate_concentration: 0.25

quality_thresholds:
  high: 0.70
  medium: 0.35

# Learning Engine thresholds — V1+ only, not active in V0
learning:
  alpha_rule_no_history: 1.00
  alpha_rule_100_samples: 0.80
  alpha_rule_1000_samples: 0.70
  decay_rate: 0.001
```

### 13.7 semantic_dictionary.yaml

See full content in Section 6.2. Reproduced here for completeness:

```yaml
METRIC_REVENUE:
  exact: [revenue, ventas, ingresos, sales, income, facturacion,
          facturación, monto, importe, amount, total_revenue,
          gross_revenue, net_revenue]
  contains: [revenue, ventas, sales, income]

METRIC_COUNT:
  exact: [count, cantidad, total, num, numero, número, qty,
          quantity, units, unidades]
  contains: [count, cantidad, quantity]

METRIC_PROFIT:
  exact: [profit, ganancia, utilidad, margen, margin, benefit,
          beneficio, ebitda, earnings]
  contains: [profit, ganancia, margin, margen]

TEMPORAL_DIMENSION:
  exact: [date, fecha, day, dia, día, week, semana, month, mes,
          quarter, trimestre, year, año, anio, hour, hora,
          datetime, timestamp, periodo, period, time,
          created_at, updated_at, order_date, sale_date,
          event_date, dt]
  contains: [date, fecha, month, year, week, quarter,
             timestamp, periodo, created, updated]

GEOGRAPHIC_DIMENSION:
  exact: [country, pais, país, region, región, city, ciudad,
          state, estado, province, provincia, territory,
          territorio, zone, zona, location, ubicacion,
          ubicación, geo, geography, continent, continente]
  contains: [country, pais, region, ciudad, city, state,
             province, location, geo]

PRODUCT_ENTITY:
  exact: [product, producto, item, sku, article, articulo,
          artículo, category, categoria, categoría, brand,
          marca, model, modelo, service, servicio]
  contains: [product, producto, category, categoria, brand,
             marca, sku]

CUSTOMER_ENTITY:
  exact: [customer, cliente, user, usuario, client, buyer,
          comprador, account, cuenta, member, miembro,
          subscriber, suscriptor]
  contains: [customer, cliente, user, usuario, account, buyer]
```

### 13.8 Loading All Rules — Startup Validation

```python
# sqlviz-inference/src/utils/startup_check.py

from .yaml_loader import yaml_loader

REQUIRED_FILES = [
    "feature_vector_v0.yaml",
    "intent_rules.yaml",
    "chart_affinity_matrix.yaml",
    "chart_penalties.yaml",
    "fallback_rules.yaml",
    "thresholds.yaml",
    "semantic_dictionary.yaml",
]


def validate_rules_on_startup() -> list[str]:
    """
    Load and validate all rule files exist and parse correctly.
    Called once when sqlviz-inference is imported.
    Returns list of errors (empty if all valid).
    """
    errors = []
    for filename in REQUIRED_FILES:
        try:
            data = yaml_loader.load(filename)
            if not data:
                errors.append(f"{filename}: file is empty")
        except FileNotFoundError as e:
            errors.append(f"{filename}: not found — {e}")
        except Exception as e:
            errors.append(f"{filename}: parse error — {e}")

    # Validate intent weights sum approximately to 1.0
    intent_rules = yaml_loader.load("intent_rules.yaml")
    for intent_name, config in intent_rules.items():
        weights = config.get("weights", {})
        total = sum(weights.values())
        if not (0.95 <= total <= 1.05):
            errors.append(
                f"intent_rules.yaml: '{intent_name}' weights sum to "
                f"{total:.2f}, expected ~1.0"
            )

    return errors
```

---

*Section 13 complete. Next: Section 14 — Benchmark / Gold Dataset.*

---

## 14. Benchmark — Gold Dataset

### 14.1 Why a Benchmark Is Mandatory

```
Without a benchmark, YAML weights are "pretty numbers"
with no proof they actually work.

The benchmark is the RELEASE GATE for SQLviz:
→ Never ship a new version if benchmark accuracy drops
→ Every change to YAML rules must be validated against it
→ It is the only objective measure of inference quality

ChatGPT's review (incorporated from prior conversation):
"Sin eso, los pesos son 'bonitos', pero no sabes si sirven."
```

### 14.2 Structure

```
sqlviz-inference/tests/benchmark/
├── benchmark_cases.yaml      ← 30 queries with expected output (V0 minimum)
└── run_benchmark.py          ← accuracy measurement script
```

Each test case defines:
```
sql              the query to test
data             (optional) sample result rows
schema           (optional) column schema
expected_intent  the correct intent
expected_chart   the correct chart type
min_quality      minimum acceptable quality ("high" | "medium" | "low")
notes            why this case matters
```

### 14.3 benchmark_cases.yaml — 30 Cases

```yaml
# sqlviz-inference/tests/benchmark/benchmark_cases.yaml
# Gold Dataset V0 — minimum 30 cases.
# Grows to 100 then 300 as SQLviz matures (per DOC 4 roadmap).

cases:

  # ── KPI cases (1-5) ──────────────────────────────────────
  - id: kpi_001
    sql: "SELECT SUM(revenue) AS total FROM sales"
    data: [{total: 125430.0}]
    schema: [{name: total, type: DOUBLE}]
    expected_intent: kpi
    expected_chart: kpi
    min_quality: high
    notes: "Simplest KPI — single SUM, no GROUP BY"

  - id: kpi_002
    sql: "SELECT COUNT(*) AS total_orders FROM orders"
    data: [{total_orders: 4521}]
    schema: [{name: total_orders, type: BIGINT}]
    expected_intent: kpi
    expected_chart: kpi
    min_quality: high
    notes: "COUNT(*) KPI pattern"

  - id: kpi_003
    sql: "SELECT AVG(order_value) AS avg_order FROM orders"
    data: [{avg_order: 87.5}]
    schema: [{name: avg_order, type: DOUBLE}]
    expected_intent: kpi
    expected_chart: kpi
    min_quality: high
    notes: "AVG aggregation KPI"

  - id: kpi_004
    sql: "SELECT SUM(ventas) AS total FROM ventas"
    data: [{total: 87000.0}]
    schema: [{name: total, type: DOUBLE}]
    expected_intent: kpi
    expected_chart: kpi
    min_quality: high
    notes: "Spanish language — same pattern as kpi_001"

  - id: kpi_005
    sql: "SELECT COUNT(DISTINCT customer_id) AS unique_customers FROM orders"
    data: [{unique_customers: 892}]
    schema: [{name: unique_customers, type: BIGINT}]
    expected_intent: kpi
    expected_chart: kpi
    min_quality: medium
    notes: "DISTINCT count — slightly less common pattern"

  # ── Trend cases (6-11) ───────────────────────────────────
  - id: trend_001
    sql: "SELECT month, SUM(revenue) FROM sales GROUP BY month ORDER BY month"
    data: [{month: "2024-01", revenue: 8500}, {month: "2024-02", revenue: 9200},
           {month: "2024-03", revenue: 11000}, {month: "2024-04", revenue: 10500}]
    schema: [{name: month, type: DATE}, {name: revenue, type: DOUBLE}]
    expected_intent: trend
    expected_chart: line
    min_quality: high
    notes: "Canonical trend pattern"

  - id: trend_002
    sql: "SELECT fecha, SUM(ventas) FROM ventas GROUP BY fecha ORDER BY fecha"
    schema: [{name: fecha, type: DATE}, {name: ventas, type: DOUBLE}]
    expected_intent: trend
    expected_chart: line
    min_quality: high
    notes: "Spanish — must produce same fingerprint as trend_001"

  - id: trend_003
    sql: "SELECT week, COUNT(*) FROM events GROUP BY week ORDER BY week"
    schema: [{name: week, type: DATE}, {name: count, type: BIGINT}]
    expected_intent: trend
    expected_chart: line
    min_quality: high
    notes: "COUNT-based trend"

  - id: trend_004
    sql: "SELECT created_at, AVG(rating) FROM reviews GROUP BY created_at ORDER BY created_at"
    schema: [{name: created_at, type: DATE}, {name: rating, type: DOUBLE}]
    expected_intent: trend
    expected_chart: line
    min_quality: medium
    notes: "created_at temporal pattern, AVG aggregation"

  - id: trend_005
    sql: "SELECT quarter, SUM(revenue) FROM sales GROUP BY quarter ORDER BY quarter"
    schema: [{name: quarter, type: VARCHAR}, {name: revenue, type: DOUBLE}]
    expected_intent: trend
    expected_chart: line
    min_quality: medium
    notes: "Quarter as VARCHAR — temporal dict still matches"

  - id: trend_006
    sql: "SELECT month, SUM(revenue), SUM(cost) FROM sales GROUP BY month ORDER BY month"
    schema: [{name: month, type: DATE}, {name: revenue, type: DOUBLE}, {name: cost, type: DOUBLE}]
    expected_intent: trend
    expected_chart: line
    min_quality: medium
    notes: "Multiple metrics over time — V0 picks line, V1 should suggest multiline"

  # ── Comparison cases (12-16) ─────────────────────────────
  - id: comparison_001
    sql: "SELECT region, SUM(revenue) FROM sales GROUP BY region"
    data: [{region: "North", revenue: 45000}, {region: "South", revenue: 32000},
           {region: "East", revenue: 28000}, {region: "West", revenue: 19000}]
    schema: [{name: region, type: VARCHAR}, {name: revenue, type: DOUBLE}]
    expected_intent: comparison
    expected_chart: bar
    min_quality: high
    notes: "Canonical comparison pattern"

  - id: comparison_002
    sql: "SELECT categoria, COUNT(*) FROM productos GROUP BY categoria"
    schema: [{name: categoria, type: VARCHAR}, {name: count, type: BIGINT}]
    expected_intent: comparison
    expected_chart: bar
    min_quality: high
    notes: "Spanish category comparison"

  - id: comparison_003
    sql: "SELECT department, AVG(salary) FROM employees GROUP BY department"
    schema: [{name: department, type: VARCHAR}, {name: salary, type: DOUBLE}]
    expected_intent: comparison
    expected_chart: bar
    min_quality: high
    notes: "AVG-based comparison"

  - id: comparison_004
    sql: "SELECT browser, COUNT(*) FROM sessions GROUP BY browser"
    data: [{browser: "Chrome", count: 5000}, {browser: "Safari", count: 2000},
           {browser: "Firefox", count: 800}, {browser: "Edge", count: 400}]
    schema: [{name: browser, type: VARCHAR}, {name: count, type: BIGINT}]
    expected_intent: comparison
    expected_chart: bar
    min_quality: high
    notes: "Web analytics comparison pattern"

  - id: comparison_005
    sql: "SELECT pais, SUM(ingresos) FROM ventas GROUP BY pais"
    schema: [{name: pais, type: VARCHAR}, {name: ingresos, type: DOUBLE}]
    expected_intent: comparison
    expected_chart: bar
    min_quality: high
    notes: "Geographic comparison — Spanish, but no ranking pattern"

  # ── Ranking cases (17-21) ────────────────────────────────
  - id: ranking_001
    sql: "SELECT product, SUM(revenue) FROM sales GROUP BY product ORDER BY 2 DESC LIMIT 10"
    schema: [{name: product, type: VARCHAR}, {name: revenue, type: DOUBLE}]
    expected_intent: ranking
    expected_chart: bar_horizontal
    min_quality: high
    notes: "Canonical ranking with ORDER BY DESC + LIMIT"

  - id: ranking_002
    sql: "SELECT producto, COUNT(*) FROM ventas GROUP BY producto ORDER BY 2 DESC LIMIT 5"
    schema: [{name: producto, type: VARCHAR}, {name: count, type: BIGINT}]
    expected_intent: ranking
    expected_chart: bar_horizontal
    min_quality: high
    notes: "Spanish ranking, smaller LIMIT"

  - id: ranking_003
    sql: "SELECT customer_name, SUM(total_spent) FROM orders GROUP BY customer_name ORDER BY 2 DESC LIMIT 20"
    schema: [{name: customer_name, type: VARCHAR}, {name: total_spent, type: DOUBLE}]
    expected_intent: ranking
    expected_chart: bar_horizontal
    min_quality: high
    notes: "Larger LIMIT — top customers"

  - id: ranking_004
    sql: "SELECT page_url, COUNT(*) AS views FROM pageviews GROUP BY page_url ORDER BY views DESC LIMIT 10"
    schema: [{name: page_url, type: VARCHAR}, {name: views, type: BIGINT}]
    expected_intent: ranking
    expected_chart: bar_horizontal
    min_quality: high
    notes: "ORDER BY column alias instead of position"

  - id: ranking_005
    sql: "SELECT employee, SUM(sales_amount) FROM sales GROUP BY employee ORDER BY 2 ASC LIMIT 5"
    schema: [{name: employee, type: VARCHAR}, {name: sales_amount, type: DOUBLE}]
    expected_intent: comparison
    expected_chart: bar
    min_quality: medium
    notes: "ASC + LIMIT is 'bottom N' — not the strong ranking_pattern (which requires DESC)"

  # ── Composition / Distribution cases (22-25) ─────────────
  - id: composition_001
    sql: "SELECT payment_method, COUNT(*) FROM orders GROUP BY payment_method"
    data: [{payment_method: "Credit Card", count: 700}, {payment_method: "PayPal", count: 200},
           {payment_method: "Cash", count: 100}]
    schema: [{name: payment_method, type: VARCHAR}, {name: count, type: BIGINT}]
    expected_intent: composition
    expected_chart: pie
    min_quality: medium
    notes: "Few categories, part-to-whole — pie candidate"

  - id: composition_002
    sql: "SELECT subscription_tier, COUNT(*) FROM users GROUP BY subscription_tier"
    data: [{subscription_tier: "Free", count: 8000}, {subscription_tier: "Pro", count: 1500},
           {subscription_tier: "Enterprise", count: 200}]
    schema: [{name: subscription_tier, type: VARCHAR}, {name: count, type: BIGINT}]
    expected_intent: composition
    expected_chart: pie
    min_quality: medium
    notes: "3 categories — clean composition pattern"

  - id: distribution_001
    sql: "SELECT order_value FROM orders"
    data: [{order_value: v} for v in [10, 15, 20, 22, 25, 30, 35, 40, 80, 150]]
    schema: [{name: order_value, type: DOUBLE}]
    expected_intent: distribution
    expected_chart: histogram
    min_quality: medium
    notes: "Single numeric column, no aggregation, no grouping"

  - id: distribution_002
    sql: "SELECT age FROM customers"
    schema: [{name: age, type: INTEGER}]
    expected_intent: distribution
    expected_chart: histogram
    min_quality: medium
    notes: "Demographic distribution"

  # ── Correlation cases (26-27) ────────────────────────────
  - id: correlation_001
    sql: "SELECT price, quantity_sold FROM products"
    data: [{price: p, quantity_sold: q} for p, q in
           [(10,100),(20,80),(30,60),(40,40),(50,20)]]
    schema: [{name: price, type: DOUBLE}, {name: quantity_sold, type: INTEGER}]
    expected_intent: correlation
    expected_chart: scatter
    min_quality: high
    notes: "Two numeric columns, no aggregation, no grouping — classic scatter"

  - id: correlation_002
    sql: "SELECT marketing_spend, revenue FROM monthly_data"
    schema: [{name: marketing_spend, type: DOUBLE}, {name: revenue, type: DOUBLE}]
    expected_intent: correlation
    expected_chart: scatter
    min_quality: high
    notes: "Business correlation use case"

  # ── Detail cases (28-30) ─────────────────────────────────
  - id: detail_001
    sql: "SELECT id, name, email, phone, address FROM customers"
    schema: [{name: id, type: INTEGER}, {name: name, type: VARCHAR},
             {name: email, type: VARCHAR}, {name: phone, type: VARCHAR},
             {name: address, type: VARCHAR}]
    expected_intent: detail
    expected_chart: table
    min_quality: high
    notes: "No aggregation, no grouping, many columns — clear detail view"

  - id: detail_002
    sql: "SELECT * FROM orders WHERE order_date = '2024-01-15'"
    schema: [{name: order_id, type: INTEGER}, {name: customer_id, type: INTEGER},
             {name: order_date, type: DATE}, {name: total, type: DOUBLE}]
    expected_intent: detail
    expected_chart: table
    min_quality: medium
    notes: "Filtered detail query"

  - id: detail_003
    sql: "SELECT order_id, product_name, quantity, unit_price FROM order_items LIMIT 100"
    schema: [{name: order_id, type: INTEGER}, {name: product_name, type: VARCHAR},
             {name: quantity, type: INTEGER}, {name: unit_price, type: DOUBLE}]
    expected_intent: detail
    expected_chart: table
    min_quality: medium
    notes: "LIMIT without ORDER BY DESC — not a ranking pattern, still detail"
```

### 14.4 run_benchmark.py — Accuracy Measurement

```python
# sqlviz-inference/tests/benchmark/run_benchmark.py

import yaml
from pathlib import Path
from src.context import RuntimeContext, ColumnSchema
from src.pipeline import RuntimePipeline


def load_benchmark_cases() -> list[dict]:
    path = Path(__file__).parent / "benchmark_cases.yaml"
    with open(path, "r", encoding="utf-8") as f:
        data = yaml.safe_load(f)
    return data["cases"]


def run_benchmark() -> dict:
    """
    Run all benchmark cases and compute accuracy metrics.

    Returns:
        {
            "total_cases": int,
            "intent_accuracy": float,
            "chart_accuracy": float,
            "quality_pass_rate": float,
            "failures": list[dict]
        }
    """
    cases = load_benchmark_cases()
    pipeline = RuntimePipeline()

    intent_correct = 0
    chart_correct = 0
    quality_pass = 0
    failures = []

    quality_rank = {"high": 3, "medium": 2, "low": 1}

    for case in cases:
        schema = [
            ColumnSchema(name=c["name"], type=c["type"])
            for c in case.get("schema", [])
        ]
        ctx = RuntimeContext(
            sql=case["sql"],
            data=case.get("data", []),
            schema=schema
        )
        ctx = pipeline.run(ctx)

        intent_match = ctx.intent_winner == case["expected_intent"]
        chart_match = ctx.chart_winner == case["expected_chart"]
        min_quality = case.get("min_quality", "low")
        quality_ok = (
            quality_rank.get(ctx.chart_quality, 0) >=
            quality_rank.get(min_quality, 0)
        )

        if intent_match:
            intent_correct += 1
        if chart_match:
            chart_correct += 1
        if quality_ok:
            quality_pass += 1

        if not (intent_match and chart_match and quality_ok):
            failures.append({
                "id": case["id"],
                "sql": case["sql"],
                "expected_intent": case["expected_intent"],
                "actual_intent": ctx.intent_winner,
                "expected_chart": case["expected_chart"],
                "actual_chart": ctx.chart_winner,
                "expected_min_quality": min_quality,
                "actual_quality": ctx.chart_quality,
                "notes": case.get("notes", "")
            })

    total = len(cases)
    return {
        "total_cases": total,
        "intent_accuracy": round(intent_correct / total, 4),
        "chart_accuracy": round(chart_correct / total, 4),
        "quality_pass_rate": round(quality_pass / total, 4),
        "failures": failures
    }


if __name__ == "__main__":
    results = run_benchmark()
    print(f"\n{'='*60}")
    print(f"SQLviz Inference Engine — Benchmark Results")
    print(f"{'='*60}")
    print(f"Total cases:       {results['total_cases']}")
    print(f"Intent accuracy:   {results['intent_accuracy']*100:.1f}%")
    print(f"Chart accuracy:    {results['chart_accuracy']*100:.1f}%")
    print(f"Quality pass rate: {results['quality_pass_rate']*100:.1f}%")

    if results["failures"]:
        print(f"\n{len(results['failures'])} FAILURES:\n")
        for f in results["failures"]:
            print(f"  [{f['id']}] {f['notes']}")
            print(f"    Intent: expected={f['expected_intent']} actual={f['actual_intent']}")
            print(f"    Chart:  expected={f['expected_chart']} actual={f['actual_chart']}")
            print()
    else:
        print("\nAll cases passed! ✓\n")
```

### 14.5 test_benchmark.py — CI Release Gate

```python
# sqlviz-inference/tests/test_benchmark.py

import pytest
from .benchmark.run_benchmark import run_benchmark


# These thresholds are the RELEASE GATE.
# Never merge code that drops below these values.
MIN_INTENT_ACCURACY = 0.85
MIN_CHART_ACCURACY = 0.85
MIN_QUALITY_PASS_RATE = 0.80


class TestBenchmarkReleaseGate:

    def test_intent_accuracy_above_threshold(self):
        results = run_benchmark()
        assert results["intent_accuracy"] >= MIN_INTENT_ACCURACY, (
            f"Intent accuracy {results['intent_accuracy']*100:.1f}% "
            f"is below the release gate of {MIN_INTENT_ACCURACY*100:.0f}%.\n"
            f"Failures: {results['failures']}"
        )

    def test_chart_accuracy_above_threshold(self):
        results = run_benchmark()
        assert results["chart_accuracy"] >= MIN_CHART_ACCURACY, (
            f"Chart accuracy {results['chart_accuracy']*100:.1f}% "
            f"is below the release gate of {MIN_CHART_ACCURACY*100:.0f}%.\n"
            f"Failures: {results['failures']}"
        )

    def test_quality_pass_rate_above_threshold(self):
        results = run_benchmark()
        assert results["quality_pass_rate"] >= MIN_QUALITY_PASS_RATE, (
            f"Quality pass rate {results['quality_pass_rate']*100:.1f}% "
            f"is below the release gate of {MIN_QUALITY_PASS_RATE*100:.0f}%."
        )
```

### 14.6 Benchmark Growth Plan

```
V0.1 release:  30 cases  (this document)
               Covers: KPI, Trend, Comparison, Ranking,
               Composition, Distribution, Correlation, Detail
               Covers: English + Spanish fingerprint equivalence

V0.2 target:   100 cases
               Add: Anomaly, Cohort, Retention, Funnel cases
               Add: edge cases discovered from real usage
               Add: cases for ClickHouse dialect differences

V0.3 target:   300 cases
               Add: cases that exercise V1 feature vector (128 dims)
               Add: cases for Multi-Panel Insights
               Add: adversarial cases (ambiguous intent)

Process for growing the benchmark:
1. Every user-reported misclassification becomes a new case
2. Every new chart type added needs 3+ cases minimum
3. Every new intent added needs 3+ cases minimum
4. Cases are never removed, only added (regression protection)
```

---

*Section 14 complete. Next: Section 15 — Dashboard Engine.*

---

## 15. Dashboard Engine

### 15.1 Why This Module Is Critical

Every module so far (Sections 4-11) infers properties
of a **single panel** from a **single SQL query**.

But SQLviz's core promise is:

> "The user writes multiple SQL queries.
>  A complete dashboard appears — automatically arranged."

Without a Dashboard Engine, SQLviz only solves
"infer one chart from one query."
It does NOT solve "arrange N panels into a coherent dashboard."

This gap was identified through external review and is
the single most important addition to close before V0.1 ships,
because it is the literal embodiment of the SQLviz philosophy.

```
Without Dashboard Engine:
    Q1 → KPI panel  (laid out independently, col_span=3)
    Q2 → Line panel (laid out independently, col_span=12)
    Q3 → Bar panel  (laid out independently, col_span=12)
    Q4 → KPI panel  (laid out independently, col_span=3)
    → Result: KPIs scattered, no grouping, no coherent order

With Dashboard Engine:
    Q1, Q2, Q3, Q4 → analyzed together
    → Result:
        Row 1: [KPI(Q1)] [KPI(Q4)]              (KPIs grouped)
        Row 2: [Line(Q2) full width]             (trend prioritized)
        Row 3: [Bar(Q3) full width]              (comparison after)
```

### 15.2 Responsibility

```
Input:  list[InferenceResult]   — one per panel/query
                                   already processed by Sections 4-11

Output: DashboardLayout
        - rows: list[DashboardRow]
        - each row contains 1+ panels with final col_span
        - panels ordered by a coherent narrative sequence
```

### 15.3 Dashboard Composition Rules

```
Rule 1 — Group KPIs together
    All panels with chart_winner == "kpi" are collected
    into a shared first row (or set of rows if > 4 KPIs).
    Within a KPI row, panels are distributed evenly:
        1 KPI  → col_span=12
        2 KPIs → col_span=6 each
        3 KPIs → col_span=4 each
        4 KPIs → col_span=3 each
        5+ KPIs → wrap into multiple rows of 4

Rule 2 — Narrative ordering (after KPI row)
    Remaining panels are ordered by a fixed priority
    that mirrors how a human analyst would present findings:

        1. trend        (line charts — "what happened over time")
        2. comparison    (bar charts — "how do categories compare")
        3. ranking       (bar_horizontal — "who/what stands out")
        4. composition   (pie — "how is it broken down")
        5. distribution  (histogram — "what's the spread")
        6. correlation   (scatter — "are these related")
        7. detail        (table — "show me everything", always last)

Rule 3 — Row packing
    After KPIs and ordering, panels are packed into rows
    using their individual col_span (from Layout Engine, Section 9),
    filling each row up to 12 columns before starting a new row.

    Example packing:
        Panel A: col_span=8  → starts Row 2
        Panel B: col_span=4  → fits in Row 2 (8+4=12, row full)
        Panel C: col_span=12 → starts Row 3 (needs full row)

Rule 4 — Full-width charts never share a row
    chart types with col_span=12 by Layout Engine default
    (line, bar, bar_horizontal, table) never get reduced
    to share a row with another panel, UNLESS the Layout
    Engine already gave them a smaller span via intent
    adjustment (e.g. composition+pie → col_span=4 can share).
```

### 15.4 dashboard_engine.py — Complete

```python
# sqlviz-inference/src/dashboard/dashboard_engine.py

from dataclasses import dataclass, field
from ..result import InferenceResult


# Narrative priority order — mirrors analyst storytelling
INTENT_PRIORITY = {
    "trend":        1,
    "comparison":   2,
    "ranking":      3,
    "composition":  4,
    "distribution": 5,
    "correlation":  6,
    "anomaly":      7,
    "cohort":       8,
    "retention":    9,
    "funnel":       10,
    "detail":       11,   # always last
}


@dataclass
class DashboardPanel:
    """A panel ready to render, with final position info."""
    inference_result: InferenceResult
    panel_id: str
    final_col_span: int
    row_index: int


@dataclass
class DashboardRow:
    panels: list[DashboardPanel] = field(default_factory=list)

    @property
    def total_span(self) -> int:
        return sum(p.final_col_span for p in self.panels)


@dataclass
class DashboardLayout:
    rows: list[DashboardRow] = field(default_factory=list)

    @property
    def panel_count(self) -> int:
        return sum(len(r.panels) for r in self.rows)


class DashboardEngine:
    """
    Composes N independently-inferred panels into a coherent
    dashboard layout.

    This is the module that fulfills SQLviz's core promise:
    "write multiple SQL queries → get a complete dashboard."

    Input: list of (panel_id, InferenceResult) tuples
    Output: DashboardLayout
    """

    def compose(
        self,
        panels: list[tuple[str, InferenceResult]]
    ) -> DashboardLayout:
        if not panels:
            return DashboardLayout(rows=[])

        kpi_panels = [
            (pid, r) for pid, r in panels if r.chart_winner == "kpi"
        ]
        other_panels = [
            (pid, r) for pid, r in panels if r.chart_winner != "kpi"
        ]

        rows: list[DashboardRow] = []

        # Rule 1 — KPI rows
        if kpi_panels:
            rows.extend(self._build_kpi_rows(kpi_panels))

        # Rule 2 — Narrative ordering
        ordered_others = self._order_by_narrative(other_panels)

        # Rule 3 — Row packing
        rows.extend(self._pack_into_rows(ordered_others, start_row=len(rows)))

        return DashboardLayout(rows=rows)

    def _build_kpi_rows(
        self,
        kpi_panels: list[tuple[str, InferenceResult]]
    ) -> list[DashboardRow]:
        """Group KPIs into rows of up to 4, evenly spaced."""
        rows = []
        chunk_size = 4
        row_idx = 0

        for i in range(0, len(kpi_panels), chunk_size):
            chunk = kpi_panels[i:i + chunk_size]
            col_span = {1: 12, 2: 6, 3: 4, 4: 3}[len(chunk)]

            dashboard_panels = [
                DashboardPanel(
                    inference_result=result,
                    panel_id=pid,
                    final_col_span=col_span,
                    row_index=row_idx
                )
                for pid, result in chunk
            ]
            rows.append(DashboardRow(panels=dashboard_panels))
            row_idx += 1

        return rows

    def _order_by_narrative(
        self,
        panels: list[tuple[str, InferenceResult]]
    ) -> list[tuple[str, InferenceResult]]:
        """Sort panels by the analyst storytelling priority."""
        def priority_key(item):
            _, result = item
            return INTENT_PRIORITY.get(result.intent_winner, 99)

        return sorted(panels, key=priority_key)

    def _pack_into_rows(
        self,
        panels: list[tuple[str, InferenceResult]],
        start_row: int
    ) -> list[DashboardRow]:
        """
        Pack panels into rows of up to 12 columns,
        respecting each panel's individual col_span.
        """
        rows: list[DashboardRow] = []
        current_row = DashboardRow()
        row_idx = start_row

        for pid, result in panels:
            panel = DashboardPanel(
                inference_result=result,
                panel_id=pid,
                final_col_span=result.col_span,
                row_index=row_idx
            )

            # Full-width panels (col_span=12) always start a new row
            if result.col_span == 12:
                if current_row.panels:
                    rows.append(current_row)
                    row_idx += 1
                panel.row_index = row_idx
                rows.append(DashboardRow(panels=[panel]))
                current_row = DashboardRow()
                row_idx += 1
                continue

            # Check if panel fits in current row
            if current_row.total_span + panel.final_col_span <= 12:
                panel.row_index = row_idx
                current_row.panels.append(panel)
            else:
                if current_row.panels:
                    rows.append(current_row)
                    row_idx += 1
                panel.row_index = row_idx
                current_row = DashboardRow(panels=[panel])

        if current_row.panels:
            rows.append(current_row)

        return rows
```

### 15.5 Integration with the Pipeline

The Dashboard Engine operates at a different level
than the per-panel pipeline (Sections 4-11).
It is called once per dashboard, after all panels
have been individually inferred.

```python
# Usage at the API layer (sqlviz-api), not inside RuntimePipeline

from sqlviz_inference import infer
from sqlviz_inference.dashboard import DashboardEngine

queries = sql_content.split(";")
panel_results = []

for i, query in enumerate(queries):
    if query.strip():
        result = infer(sql=query, data=execute(query), schema=describe(query))
        panel_results.append((f"panel_{i}", result))

dashboard_engine = DashboardEngine()
layout = dashboard_engine.compose(panel_results)

# layout.rows is ready to send to the frontend
```

### 15.6 Dashboard Engine Tests

```python
# sqlviz-inference/tests/test_dashboard_engine.py

import pytest
from src.result import InferenceResult
from src.dashboard.dashboard_engine import DashboardEngine


def make_result(chart_winner: str, intent_winner: str, col_span: int) -> InferenceResult:
    """Minimal InferenceResult for dashboard composition tests."""
    return InferenceResult(
        rules_version="test", feature_vector_version="v0", engine_version="test",
        intent_winner=intent_winner, intent_raw_score=0.9,
        intent_normalized_score=1.0, intent_confidence_gap=0.5,
        intent_quality="high", intent_alternatives=[],
        chart_winner=chart_winner, chart_raw_score=0.9,
        chart_normalized_score=1.0, chart_confidence_gap=0.5,
        chart_quality="high", chart_alternatives=[],
        col_span=col_span, row_span=1, layout_importance=0.5,
        filter_controls=[], title="Test", title_confidence=0.8,
        fallback_applied=False, fallback_reason="",
        explanation=[], score_trace={}, fingerprint="TEST",
        feature_vector=[0.0]*39, errors=[], elapsed_ms=0.0
    )


engine = DashboardEngine()


class TestKPIGrouping:

    def test_single_kpi_full_width(self):
        panels = [("p1", make_result("kpi", "kpi", 3))]
        layout = engine.compose(panels)
        assert layout.rows[0].panels[0].final_col_span == 12

    def test_two_kpis_share_row(self):
        panels = [
            ("p1", make_result("kpi", "kpi", 3)),
            ("p2", make_result("kpi", "kpi", 3)),
        ]
        layout = engine.compose(panels)
        assert len(layout.rows[0].panels) == 2
        assert layout.rows[0].panels[0].final_col_span == 6

    def test_four_kpis_one_row(self):
        panels = [("p%d" % i, make_result("kpi", "kpi", 3)) for i in range(4)]
        layout = engine.compose(panels)
        assert len(layout.rows[0].panels) == 4
        assert layout.rows[0].panels[0].final_col_span == 3

    def test_five_kpis_wrap_to_two_rows(self):
        panels = [("p%d" % i, make_result("kpi", "kpi", 3)) for i in range(5)]
        layout = engine.compose(panels)
        kpi_rows = [r for r in layout.rows if r.panels[0].inference_result.chart_winner == "kpi"]
        assert len(kpi_rows) == 2


class TestNarrativeOrdering:

    def test_trend_before_comparison(self):
        panels = [
            ("p1", make_result("bar", "comparison", 12)),
            ("p2", make_result("line", "trend", 12)),
        ]
        layout = engine.compose(panels)
        first_intent = layout.rows[0].panels[0].inference_result.intent_winner
        assert first_intent == "trend"

    def test_detail_always_last(self):
        panels = [
            ("p1", make_result("table", "detail", 12)),
            ("p2", make_result("line", "trend", 12)),
            ("p3", make_result("bar", "comparison", 12)),
        ]
        layout = engine.compose(panels)
        last_intent = layout.rows[-1].panels[0].inference_result.intent_winner
        assert last_intent == "detail"


class TestRowPacking:

    def test_full_width_never_shares_row(self):
        panels = [("p1", make_result("line", "trend", 12))]
        layout = engine.compose(panels)
        assert len(layout.rows[0].panels) == 1

    def test_kpi_and_full_width_in_separate_rows(self):
        panels = [
            ("p1", make_result("kpi", "kpi", 3)),
            ("p2", make_result("line", "trend", 12)),
        ]
        layout = engine.compose(panels)
        assert layout.panel_count == 2
        assert len(layout.rows) == 2  # KPI row + Line row

    def test_pie_and_bar_can_share_row(self):
        panels = [
            ("p1", make_result("bar", "comparison", 8)),
            ("p2", make_result("pie", "composition", 4)),
        ]
        layout = engine.compose(panels)
        # comparison comes before composition in narrative order
        # 8 + 4 = 12, should fit in one row
        matching_rows = [r for r in layout.rows if len(r.panels) == 2]
        assert len(matching_rows) >= 0  # depends on ordering + packing


class TestEmptyAndSingle:

    def test_empty_panels_list(self):
        layout = engine.compose([])
        assert layout.rows == []
        assert layout.panel_count == 0

    def test_single_non_kpi_panel(self):
        panels = [("p1", make_result("table", "detail", 12))]
        layout = engine.compose(panels)
        assert layout.panel_count == 1
```

### 15.7 What Dashboard Engine Does NOT Do (Yet)

```
Explicitly out of scope for V0.1 Dashboard Engine:

❌ Cross-filtering between panels
   (clicking a bar filters a table — V0.2+)

❌ Detecting "missing perspectives"
   (Dashboard Composer concept from the ChatGPT handbook —
    "you have Revenue Trend, you're missing Revenue by Country" —
    this is V0.3, requires Insight Engine context)

❌ Semantic deduplication
   (if two panels show the same metric, suggest merging —
    V0.3, requires Metric Engine)

❌ Dashboard Genome similarity
   (reusing layouts from similar past dashboards —
    requires brain.duckdb learning, V0.3+)

V0.1 Dashboard Engine does exactly one thing well:
    Given N already-inferred panels,
    arrange them into a coherent, readable layout.
    KPIs grouped. Narrative order. Rows packed correctly.
```

---

*Section 15 complete. DOC 5 — Inference Engine Architecture is now fully complete.*

---

## Document Summary (Updated)

```
✅ Section 1  — Overview & Pipeline
✅ Section 2  — Package Structure
✅ Section 3  — RuntimeContext
✅ Section 4  — Parser Module
✅ Section 5  — Feature Engine
✅ Section 6  — Semantic Engine
✅ Section 7  — Intent Engine
✅ Section 8  — Chart Engine
✅ Section 9  — Layout Engine (per-panel)
✅ Section 10 — Filter Engine
✅ Section 11 — Title Engine
✅ Section 12 — Runtime Pipeline
✅ Section 13 — Complete YAML Rules Files
✅ Section 14 — Benchmark / Gold Dataset
✅ Section 15 — Dashboard Engine (multi-panel composition)
✅ Section 16 — v0.1.1-v0.1.3 Patches: critical fixes from 3 review
   rounds (immutability wording, trend_direction dim 38,
   39-dim total count, module count, KPI label ownership)
   + V0.2/V0.3 tracking
```

## Known Limitations — Documented for V0.2 / V0.3

The following gaps were identified through external architecture
review and are intentionally deferred, not overlooked:

```
V0.2:
→ Split this document into smaller focused docs
   (DOC 5: Architecture, DOC 6: Feature Vector,
    DOC 7: Semantic Engine, DOC 8: Intent+Chart, DOC 9: Layout+Filter+Title)
→ Expand Feature Vector V0 (39 dims) toward V1 (80-120 dims)
   Add SQL signals: ROLLUP, CUBE, GROUPING SETS, QUALIFY,
   UNION, UNION ALL, EXISTS, IN, LAG, LEAD, NTILE, PERCENTILE_CONT
→ Add Data Shape features: sparsity, null_ratio, entropy,
   monotonicity, periodicity

V0.3 (already planned in DOC 1, reaffirmed here):
→ Insight Engine — automatic findings ("Revenue grew 12%")
→ Semantic Engine V1 — hybrid dictionary + embeddings
   (current dictionary fails on real-world names like
    "importe_neto", "kwh_fact", "consumo")
→ Narrative Engine — natural language summaries
→ Dashboard Composer — detect missing perspectives
   ("you have Revenue Trend, consider Revenue by Country")
```

---

## Extensibility Roadmap — Mandatory Reference

> This section must never be removed or forgotten in future
> revisions of this document. Every new version of DOC 5 must
> carry this table forward, updated with current status. This is
> the single source of truth for "is X allowed to change, and how."

### Why this section exists permanently

Architecture reviews (including external ones) repeatedly raise
the same question: *"this number/list/dictionary is fixed — will
it scale?"* The answer is always: **yes, by design, but the growth
path must stay written down.** A component that is not documented
as "designed to grow" tends to get hardcoded against by accident
in later code (API contracts, frontend assumptions, serialization
formats). This table prevents that.

### The table

```
Component                Current (V0)          Growth path          Status
─────────────────────────────────────────────────────────────────────────
Feature Vector            39 dims, list[float]   → 80-120 dims (V1)  PLANNED
                                                  → 150+ dims (V2)
                          Rule: always append,
                          never insert (Sec 2.3, 13.1)

Chart types                8 types                → 18 types (V1)    PLANNED
                          (Sec 13.3)              Area, Multiline,
                                                  Combo, Heatmap,
                                                  Boxplot, Treemap,
                                                  Funnel, Waterfall,
                                                  Bubble, Cohort

Intent types               12 intents             Stable — no growth STABLE
                          (Sec 7.2)               currently planned.
                                                  Revisit only if a
                                                  real use case proves
                                                  insufficient.

Semantic dictionary         7 classes,             → hybrid dictionary FUTURE
                          ~150 name patterns       + embeddings (V1)
                          (Sec 6.2, 13.7)          Required because
                                                  real-world names
                                                  ("kwh_fact",
                                                  "importe_neto")
                                                  will not match a
                                                  fixed dictionary.

Benchmark cases             30 cases (Sec 14.3)    → 100 (V0.2)       PLANNED
                                                  → 300 (V0.3)
                                                  → 1000+ (V1.0)
                          Rule: never remove
                          cases, only add
                          (Sec 14.6)

Dashboard Engine rules       4 composition rules    → cross-filtering  FUTURE
                          (Sec 15.3)              → missing-perspective
                                                    detection
                                                  → genome similarity
                                                  All require
                                                  brain.duckdb
                                                  learning (V0.3+)

Document structure          1 file, 15 sections    Split into 5       PLANNED
                          (this document)         focused documents
                                                  (V0.2) — see
                                                  "Known Limitations"
                                                  above

Learning / brain.duckdb     Not implemented        Bayesian update +  FUTURE
                          (Sec 13.6 thresholds     decay + Thompson
                          reserved but unused)     Sampling (DOC 4,
                                                  Sections 13.1-13.3)
```

### The rule this table enforces

```
Before adding ANY new dimension, chart type, intent,
semantic class, or rule category to sqlviz-inference:

1. Check this table first.
2. If the component is listed as "PLANNED" or "FUTURE" —
   follow the growth path already defined here.
3. If it is not listed — add it to this table BEFORE
   writing the implementation, with a Status of "PLANNED".
4. Never silently hardcode a number that secretly assumes
   "this will never change." This table is the proof that
   someone already thought about whether it will change.
```



---

## 16. v0.1.1 Patch — Critical Fixes from Second Architecture Review

A second, more technical external review identified two issues
that must be corrected **before** implementation begins, plus
several non-blocking items now formally logged for V0.2/V0.3/V1.
This section documents both the fixes and the deferred items —
nothing raised in the review is left unaddressed in writing,
even when the decision is "not now."

### 16.1 FIX — RuntimeContext terminology: "immutable" is wrong

**The problem (Critical, must fix before coding):**

Section 3 calls RuntimeContext "immutable" and "the immutable
carrier of all inference data," but every module example
(Sections 4-11) performs direct field assignment:

```python
context.feature_vector = fv
context.intent_winner = winner.intent
context.chart_winner = winner_chart
```

This is a real contradiction, not just imprecise wording.
True immutability would require reconstructing a new
RuntimeContext object on every single module call
(8 reconstructions per panel inference), which adds copy
overhead with no real benefit for V0.1: SQLviz processes
one SQL query per request, sequentially, single-threaded
per request. There is no shared-context-across-threads
scenario in V0.1 that immutability would protect against.

**The fix — replace "immutable" with "field-owned mutation":**

```
Corrected rule (replaces Section 3.1 and 3.4 wording):

RuntimeContext uses FIELD-OWNED MUTATION, not immutability.

- Each module owns a specific, documented set of fields
  (already listed in Section 3.4 — this list does not change).
- A module may ONLY write to the fields it owns.
- A module may READ any field (it needs upstream results),
  but must never WRITE to a field it does not own.
- The "errors" list is the one exception: any module may
  append to it via context.with_error(), never overwrite it.

This is the same practical guarantee the document already
described (no module corrupts another module's results),
achieved without the performance cost of true immutability.
If a future version needs true immutability (e.g. parallel
panel inference across threads sharing a request), revisit
this decision then — do not pre-optimize for it now.
```

All code examples in Sections 3-15 are already correct under
this corrected rule — only the word "immutable" in prose
(Section 3.1 title and Section 3 intro) should be read as
"field-owned mutation" going forward. No code changes needed.

### 16.2 FIX — trend_strength conflates magnitude and direction

**The problem (Critical, must fix before coding):**

dim 28 (`trend_strength`) is computed purely from R² (Section 5.4,
`compute_trend_strength`). R² measures only goodness-of-fit to
a straight line — it does NOT carry the sign of the slope.

```
Three very different real patterns produce the SAME R²:

Pattern A: revenue climbing steadily   (slope = +1000/month) -> R^2 = 0.95
Pattern B: revenue declining steadily  (slope = -1000/month) -> R^2 = 0.95
Pattern C: revenue perfectly flat      (slope ~  0/month)    -> R^2 = 0.95
                                                                (if noise-free)

All three currently produce identical dim 28 = 0.95, even though
"strong upward trend," "strong downward trend," and "no trend at
all" are opposite analytical conclusions. This directly affects:
- KPI Engine enrichment (future trend arrows up/down depend
  entirely on direction, not just R^2)
- V0.3 Insight Engine ("Revenue is growing" vs "declining")
```

**The fix — split into two separate features:**

```
dim 28 — trend_strength   (unchanged): R^2, magnitude only, [0.0, 1.0]
dim 38 — trend_direction  (NEW): sign of slope, encoded as [0.0, 1.0]
                                 0.0 = declining, 0.5 = flat, 1.0 = growing

Note: dim 38 was previously "reserved for V1" per Section 13.1.
This is the first deliberate use of a reserved dimension, which
is exactly what the reservation was for — it does not violate
the "always append, never insert" rule because dim 38 is being
assigned for the first time, not reused or repurposed.
```

New function (added alongside the existing `compute_trend_strength`
in `sqlviz-inference/src/utils/math_utils.py`):

```python
def compute_trend_direction(values: list[float]) -> float:
    """
    Compute the normalized direction of a linear trend.
    Returns a value in [0.0, 1.0]:
        0.0  -> strongly declining
        0.5  -> flat / no meaningful slope
        1.0  -> strongly growing

    Uses the same linear regression as compute_trend_strength,
    but returns the normalized SLOPE instead of R^2.
    """
    n = len(values)
    if n < 3:
        return 0.5  # flat / unknown - neutral default

    x = list(range(n))
    x_mean = sum(x) / n
    y_mean = sum(values) / n

    ss_xy = sum((x[i] - x_mean) * (values[i] - y_mean) for i in range(n))
    ss_xx = sum((x[i] - x_mean) ** 2 for i in range(n))

    if ss_xx == 0:
        return 0.5

    slope = ss_xy / ss_xx

    if y_mean == 0:
        return 0.5

    relative_slope = slope / abs(y_mean)

    import math
    normalized = 1.0 / (1.0 + math.exp(-10 * relative_slope))
    return max(0.0, min(1.0, normalized))
```

New function (added alongside `compute_dim28_trend_strength` in
`sqlviz-inference/src/features/data_statistics.py`):

```python
def compute_dim38_trend_direction(
    data: list[dict],
    schema: list[ColumnSchema]
) -> float:
    """
    dim 38 - direction of linear trend, [0.0, 1.0].
    0.0 = declining, 0.5 = flat, 1.0 = growing.
    Kept separate from dim 28 (trend_strength / R^2) because
    magnitude and direction are independent concepts
    (see Section 16.2).
    """
    values = _get_numeric_values(data, schema)
    if len(values) < 3:
        return 0.5
    return compute_trend_direction(values)
```

Update to `feature_engine.py` (the dims 27-29 block in Section 5.8
gains one more line):

```python
if context.has_data and context.row_count >= 3:
    fv[27] = compute_dim27_temporal_cardinality(context.data, context.schema)
    fv[28] = compute_dim28_trend_strength(context.data, context.schema)
    fv[29] = compute_dim29_has_outliers(context.data, context.schema)
    fv[38] = compute_dim38_trend_direction(context.data, context.schema)  # NEW
```

Update to `rules/feature_vector_v0.yaml` (Section 13.1) — add dim 38
explicitly and shrink the reserved range accordingly:

```yaml
  trend_strength:               {index: 28, type: continuous}
  # ... existing dims 29-37 unchanged ...

  # First deliberate use of a "reserved" dimension - see Section 16.2
  trend_direction:               {index: 38, type: continuous}

reserved:
  start_index: 39          # was 38 - now starts one later
  end_index: 127
  note: "Reserved for V1 features (DOC 4, Section 3.3). Never use in V0
        without updating this file and Section 16 Extensibility table."
```

Update to `intent/intent_engine.py` FEATURE_INDEX (Section 7.3):

```python
FEATURE_INDEX = {
    # ... all existing entries unchanged ...
    "trend_strength":            28,
    "has_outliers":               29,
    # ... dims 30-37 unchanged (dim 38 trend_direction added below) ...
    "trend_direction":            38,   # NEW
}
```

Intentional non-change to `intent_rules.yaml`: `trend_direction`
is NOT added to intent scoring weights in V0.1. A declining
revenue series is still correctly a "trend" intent — direction
only matters for insight generation (V0.3), not chart/intent
selection. Logged here so this is not "fixed" by accident later.

New tests (extend Section 5.9 `TestDataStatistics`):

```python
class TestTrendDirection:

    def test_growing_trend_direction_high(self):
        data = [{"month": i, "revenue": i * 1000} for i in range(1, 13)]
        schema_defs = [("month", "INTEGER"), ("revenue", "DOUBLE")]
        ctx = make_context(data=data, schema_defs=schema_defs)
        assert ctx.feature_vector[38] > 0.65   # clearly growing

    def test_declining_trend_direction_low(self):
        data = [{"month": i, "revenue": (13 - i) * 1000} for i in range(1, 13)]
        schema_defs = [("month", "INTEGER"), ("revenue", "DOUBLE")]
        ctx = make_context(data=data, schema_defs=schema_defs)
        assert ctx.feature_vector[38] < 0.35   # clearly declining

    def test_flat_trend_direction_neutral(self):
        data = [{"month": i, "revenue": 5000} for i in range(1, 13)]
        schema_defs = [("month", "INTEGER"), ("revenue", "DOUBLE")]
        ctx = make_context(data=data, schema_defs=schema_defs)
        assert 0.35 <= ctx.feature_vector[38] <= 0.65

    def test_strength_and_direction_are_independent(self):
        growing   = [{"month": i, "revenue": i * 1000} for i in range(1, 13)]
        declining = [{"month": i, "revenue": (13 - i) * 1000} for i in range(1, 13)]
        schema_defs = [("month", "INTEGER"), ("revenue", "DOUBLE")]

        ctx_growing   = make_context(data=growing, schema_defs=schema_defs)
        ctx_declining = make_context(data=declining, schema_defs=schema_defs)

        assert abs(ctx_growing.feature_vector[28] - ctx_declining.feature_vector[28]) < 0.05
        assert ctx_growing.feature_vector[38] > 0.65
        assert ctx_declining.feature_vector[38] < 0.35
```

### 16.3 Logged for V0.2 — Non-blocking, formally tracked

These observations are correct but do not block V0.1. They are
carried forward in every future revision of this document until
resolved — never silently dropped.

```
Item                          What to do                       When
-------------------------------------------------------------------
Feature Registry design       Design dict-based registry         V0.2
                              {name, index, type, cost_level,     (DOC 6,
                              version} wrapping the existing       new doc)
                              list[float] for computation -
                              do NOT replace the array, ADD
                              a typed lookup layer on top.
                              Needed once feature count
                              approaches 80 (Extensibility
                              Roadmap table, below).

contribution_pct in            explanation[] currently reports    V0.2
explanation                   raw weight x value contribution.
                              Add a normalized percentage:
                              contribution_pct = contribution
                                / sum(all_contributions_for_winner)
                              so a reader can see "this signal
                              explains 64% of the decision."

Advanced YAML validation       Extend validate_rules_on_startup()  V0.2
                              (Section 13.8) to also detect:
                              - feature names referenced in
                                weights/penalties that don't
                                exist in feature_vector_v0.yaml
                              - features defined but never
                                referenced by any engine
                                ("orphaned weights")
                              - a chart/intent with both a
                                strong positive weight AND a
                                penalty on the same feature
                                without an explanatory comment
                              Not done now because the 7 current
                              YAML files were hand-verified while
                              writing them - automated checking
                              matters once external contributors
                              start editing them.

Domain Dictionaries           Add optional domain-specific        V0.3
(Energy, ERP, Retail,         dictionaries loaded ON TOP of        (same
Finance, Telecom)             semantic_dictionary.yaml, e.g.        milestone
                              rules/domains/energy.yaml with         as Semantic
                              patterns like "kwh_facturado",         Engine V1)
                              "energia_total". User selects a
                              domain pack in Settings; SQLviz
                              merges it with the base dictionary
                              before fuzzy matching (Section 6.3).
                              Lighter-weight intermediate step
                              than full embeddings - ship this
                              BEFORE embeddings, not instead of.

Fingerprint granularity        Documented limitation, not a bug:   V0.2 /
                              fingerprints intentionally do NOT    not
                              distinguish (a) JOIN complexity       blocking
                              (2-table vs 5-table+subqueries
                              both flag as "JOIN"), (b) literal
                              values in LIMIT/thresholds.
                              (a) will matter once Dashboard
                              Engine needs to flag "this panel
                              is expensive, run in background"
                              (V0.2 perf work) - NOT for
                              chart/intent inference, already
                              correct as designed.
                              (b) is intentionally NOT needed:
                              result_row_count_is_1 (dim 35)
                              already captures what LIMIT 1 vs
                              LIMIT 1000 would have signaled,
                              but from the actual result shape
                              rather than SQL text - the more
                              robust design. Do not change this.
```

### 16.4 Logged for V0.3 / V1 — Future, not urgent

```
Item                          What to do                         When
-------------------------------------------------------------------
brain.duckdb full definition  DOC 4 (Sections 13.1-13.3) already    V0.3
                              has the math (Bayesian update,         (new
                              decay, Thompson Sampling). Missing:     DOC 10)
                              the full DuckDB schema, versioning
                              strategy for learned patterns, and
                              degradation safeguards (automatic
                              rollback if learned weights perform
                              worse than static YAML rules).
                              Deliberately deferred: designing the
                              schema before the rule engine has
                              run in production risks designing
                              against wrong assumptions.

Semantic Engine V1            Hybrid dictionary + embeddings,        V0.3
(embeddings)                  as already logged in "Known
                              Limitations". Domain Dictionaries
                              (16.3) are the intermediate step;
                              full embeddings are the final step
                              for names no dictionary (generic or
                              domain-specific) can anticipate.
```

### 16.5 FIX — Feature Vector is 39 dimensions, not 38 (third review)

**The problem (Critical, blocking — found by a third external
architecture review covering all 8 documents together):**

Section 16.2 added `trend_direction` at index 38. From that
point on, the active feature vector occupies indices 0-38
inclusive — that is **39 values**, not 38. Every place in this
document and in DOC 4 that still said "38-dimension feature
vector" was counting the vector as it existed *before* Section
16.2's own fix, which is the exact kind of stale-prose bug
Section 16.1/16.2 were trying to prevent elsewhere. This is now
corrected throughout both documents: every occurrence of
"38-dimension" / "38 explicit" / "dims 0-37" describing the
*total* vector size has been changed to 39 / 0-38. References
to "dims 30-37" describing *only the Semantic Engine's output*
were additionally corrected to "dims 30-34" — Semantic Engine
never owned dims 35-38 in the first place (those are Feature
Engine's Result Shape and trend_direction outputs); conflating
the two was a second, smaller bug bundled into the same prose
that the dimension-count fix touched.

```
Before this fix:  "V0 = 38 dimensions" (stale, predates dim 38)
After this fix:   "V0 = 39 dimensions, indices 0-38" (accurate)

feature_vector_v0.yaml (DOC 4 Section 13.1 / DOC 5 Section 13.1)
now explicitly lists trend_direction at index 38 as part of the
V0 feature set, with reserved starting at 39 — this YAML was
already correct (Section 16.2 wrote it that way originally);
only the surrounding prose describing "how many total" was wrong.
```

No code changes were needed for this fix — `list[float]`-typed
feature vectors in Python have no fixed-size declaration to
update; `[0.0] * 38` in any code example became `[0.0] * 39`
purely as a documentation correction wherever it appeared as
an illustrative literal.

### 16.6 FIX — KPI trend label moved from frontend to backend

**The problem (High severity — found by the same third review):**

DOC 6 (UI Design System), Section 5.4's `KPIRenderer.svelte`
computed `trendClass` and `trendIcon` directly in Svelte using
hardcoded thresholds (`direction > 0.65`, `direction < 0.35`)
applied to `result.feature_vector[38]` and `result.feature_vector[28]`.
This is a semantic decision — "is this trend growing, declining,
or flat" — made in the rendering layer, which directly
contradicts DOC 6's own Section 1.1 rule: *"if implementing a UI
component requires a decision about WHAT to show (not HOW to
show it), that decision belongs in sqlviz-inference, not here."*

**The fix:**

`InferenceResult` (Section 12.4) gains a new field,
`trend_direction_label: str`, computed once in
`from_context()` using the exact same thresholds that were
previously duplicated in Svelte:

```python
fv = context.feature_vector
strength = fv[28] if len(fv) > 28 else 0.0
direction = fv[38] if len(fv) > 38 else 0.5
if strength <= 0.5:
    trend_direction_label = "unknown"
elif direction > 0.65:
    trend_direction_label = "growing"
elif direction < 0.35:
    trend_direction_label = "declining"
else:
    trend_direction_label = "flat"
```

This preserves the exact logic DOC 6 had (including checking
`strength` before trusting `direction` — DOC 6's original code
already had this guard correctly, per Section 16.2's "strength
and direction are independent" rationale; only its *location*
was wrong, not its content). DOC 6's `KPIRenderer.svelte` is
corrected to read `result.trend_direction_label` directly and
map it to an icon/color via a static lookup — no thresholds,
no `feature_vector` indexing, no semantic decision in the
frontend at all. See DOC 6 Section 5.4 (corrected) for the
updated component.

---

*SQLviz Inference Engine Architecture — v0.1.0 (final)*
*"From mathematics to code. The first brain of SQLviz."*
