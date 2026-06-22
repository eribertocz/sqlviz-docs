# SQLviz — UI Design System
**Version:** v0.1.0 (Draft)
**Status:** Work in Progress
**Last Updated:** 2026-06-08
**Prerequisites:** DOC 1 (Vision & Philosophy), DOC 3 (Technical Stack, Sections 3 & 6-7),
DOC 5 (Inference Engine — consumes its output directly), DOC 7 (Security & Roles)

---

## 1. Purpose of This Document

DOC 1 states the philosophy: "the user writes SQL, SQLviz infers
everything else." DOC 5 is the engine that makes that true. This
document is the **face** that makes it *visible* — the exact
visual language, components, and layout rules that turn an
`InferenceResult` (DOC 5, Section 12.4) and a `DashboardLayout`
(DOC 5, Section 15.4) into pixels on screen.

```
DOC 1  →  the user writes SQL, sees a dashboard      (the promise)
DOC 5  →  InferenceResult + DashboardLayout           (the data)
DOC 6  →  exact components, colors, spacing            (this document)
          that render that data without the user
          ever touching layout, color, or chart config
```

### 1.1 The One Rule That Governs Every Decision in This Document

```
The frontend NEVER infers. The frontend ONLY renders.

Every chart_winner, col_span, row_span, title, and
filter_controls value arrives already decided from
sqlviz-api (DOC 5's output, DOC 3 Section 9's "frontend
never infers" rule, restated here because DOC 6 is where
violating it would actually happen if not careful).

If implementing a UI component requires a decision about
WHAT to show (not HOW to show it) — that decision belongs
in sqlviz-inference (DOC 5), not here.
```

---

## 2. Design Tokens

These are already defined in DOC 3, Section 6, reproduced here
as the canonical source because DOC 6 is where they are
actually consumed. If DOC 3's copy and this one ever diverge,
**this document wins** — DOC 3 will be updated to match.

```css
:root {
  /* Color */
  --sqlviz-primary:       #6366f1;  /* indigo — brand, CTAs, links */
  --sqlviz-primary-hover: #818cf8;
  --sqlviz-bg:            #0f172a;  /* dark background (default theme) */
  --sqlviz-bg-surface:    #1e293b;  /* card / panel background */
  --sqlviz-border:        #334155;
  --sqlviz-text:          #f1f5f9;  /* primary text */
  --sqlviz-text-muted:    #94a3b8;  /* secondary text, metadata */

  /* Semantic colors — used ONLY for KPI trend indicators
     (DOC 4, Section 8.2 trend direction; DOC 5 Section 16.2
     dim 38 trend_direction feeds this directly) */
  --sqlviz-positive:      #22c55e;  /* green — growing KPI */
  --sqlviz-negative:      #ef4444;  /* red — declining KPI */
  --sqlviz-neutral:       #94a3b8;  /* gray — flat KPI */

  /* Geometry */
  --sqlviz-radius:        6px;
  --sqlviz-radius-lg:     10px;
  --sqlviz-gap:           16px;     /* the gap between grid panels —
                                       see Section 4.2, must match
                                       the spacing assumed by
                                       DOC 5 Section 9 col_span math */

  /* Typography */
  --sqlviz-font-sans:     'Inter', system-ui, sans-serif;
  --sqlviz-font-mono:     'JetBrains Mono', 'Fira Code', monospace;
                          /* used ONLY in the Monaco editor and
                             execution-time/row-count metadata —
                             DOC 3 Section 3 Monaco integration */
}

[data-theme="light"] {
  --sqlviz-bg:            #ffffff;
  --sqlviz-bg-surface:    #f8fafc;
  --sqlviz-border:        #e2e8f0;
  --sqlviz-text:          #0f172a;
  --sqlviz-text-muted:    #64748b;
  /* --sqlviz-primary and semantic colors stay the same in both themes —
     this is a deliberate consistency choice, not an oversight */
}
```

### 2.1 Why These Specific Choices

```
Indigo (#6366f1) as primary:
    Distinct from the "default Bootstrap blue" (#0d6efd) that
    signals "generic admin template." SQLviz should not look
    like every other internal tool. Already used consistently
    in v0.1 prototype screenshots reviewed earlier in this
    project — kept for continuity, not re-litigated here.

Dark theme as default, not light:
    SQL editors and data tools are conventionally dark-first
    (VS Code, DataGrip, most modern BI tools default dark).
    Matches Monaco's natural habitat (DOC 3, Section 3).

JetBrains Mono / Fira Code for monospace:
    Both have programming ligatures and are free/open license
    (DOC 3, Section 9, dependency philosophy applies to fonts
    too — no paid font licenses).
```

---

## 3. Layout Philosophy — What Changed From the V0.1 Prototype

The original SQLviz prototype (rescued lessons, referenced in
DOC 1's "lessons learned") had the user manually create rows,
drag panels, and configure widths. This was identified during
that prototype's review as **breaking the core philosophy** —
the UI made the user think about the tool instead of the data.

DOC 5's Dashboard Engine (Section 15) now does this work
automatically. This document's job is to render its output,
**not to provide manual row/drag/width controls as the primary
interaction model.**

```
V0.1 Prototype (rejected pattern — DO NOT reintroduce):
    [+ Add row] → [+ Add panel] → drag to position →
    configure width % → configure height px

SQLviz V0.1 (this document):
    User writes SQL → panel appears, already positioned by
    DashboardEngine.compose() (DOC 5, Section 15.4) →
    DONE. No manual layout step exists in the primary flow.
```

A minimal override exists (Section 7) for the rare case where
inference gets it wrong — but it is explicitly an *escape
hatch*, never the expected interaction.

---

## 4. The Dashboard Grid

### 4.1 CSS Grid, 12 Columns — Consuming DOC 5's Output Directly

DOC 5 Section 9 (Layout Engine) and Section 15 (Dashboard
Engine) already compute `col_span` (1-12) and `row_span` (1-3)
per panel, and group panels into `DashboardRow` objects. This
section is the literal CSS that renders that data structure —
no additional layout logic exists in the frontend.

```css
.dashboard-grid {
  display: grid;
  grid-template-columns: repeat(12, 1fr);
  gap: var(--sqlviz-gap);
  padding: var(--sqlviz-gap);
}

.dashboard-panel {
  grid-column: span var(--panel-col-span);
  grid-row: span var(--panel-row-span);
  background: var(--sqlviz-bg-surface);
  border: 1px solid var(--sqlviz-border);
  border-radius: var(--sqlviz-radius-lg);
  overflow: hidden;
}
```

```svelte
<!-- DashboardGrid.svelte — renders DashboardLayout from DOC 5 Section 15.4 -->
<script lang="ts">
  import type { DashboardLayout } from '$lib/types';
  export let layout: DashboardLayout;
</script>

<div class="dashboard-grid">
  {#each layout.rows as row}
    {#each row.panels as panel}
      <div
        class="dashboard-panel"
        style="--panel-col-span: {panel.final_col_span};
               --panel-row-span: {panel.inference_result.row_span}"
      >
        <PanelRenderer result={panel.inference_result} />
      </div>
    {/each}
  {/each}
</div>
```

**Note the absence of any client-side logic deciding position,
span, or order** — `layout.rows` arrives pre-computed and is
iterated in the exact sequence `DashboardEngine.compose()`
produced. This is the direct, literal embodiment of Section 1.1.

### 4.2 Why the Gap Must Match Both Themes' Visual Weight

```
--sqlviz-gap: 16px applies identically in dark and light theme.
A panel with col_span=6 in a 12-column grid with 16px gaps
occupies exactly the same proportion of the viewport regardless
of theme — DashboardEngine's row-packing math (DOC 5, Section
15.3, Rule 3) assumes panels of a given col_span are visually
equal-weight; an inconsistent gap between themes would silently
break that assumption's visual truth even though the underlying
data is identical.
```

---

## 5. Panel Anatomy

### 5.1 Structure

```
┌─────────────────────────────────────┐
│ Title                          [···] │  ← header: title (DOC 5 §11) +
│                                       │     overflow menu (Section 7)
│                                       │
│            [chart renders here]      │  ← body: ECharts/Plotly per
│                                       │     chart_winner (DOC 5 §8)
│                                       │
├───────────────────────────────────────┤
│ 64ms · 1,204 rows · DuckDB            │  ← footer: execution metadata
└─────────────────────────────────────┘     (edit mode only — Section 6)
```

### 5.2 Title Rendering

```svelte
<!-- PanelHeader.svelte -->
<script lang="ts">
  export let title: string;          // DOC 5 Section 11 TitleResult.title
  export let titleConfidence: number; // DOC 5 Section 11 title_confidence
</script>

<div class="panel-header">
  <h3 class="panel-title">{title || 'Untitled query'}</h3>
  <!-- fallback "Untitled query" only fires when TitleEngine's
       graceful degradation (DOC 5 Section 11.3) returned ""
       — this is the one place the frontend supplies a string
       the backend didn't, and it is explicitly NOT an inference,
       just a placeholder label, consistent with Section 1.1 -->
</div>
```

```
titleConfidence is NOT shown visually as a number or badge in
V0.1. DOC 5's explainability data (score_trace, confidence_gap)
is surfaced through the "Why this chart?" panel (Section 8),
not scattered as inline confidence indicators next to every
title — that would clutter the dashboard and contradicts DOC 1
Principle 2 ("infer everything, the user should rarely need to
think about confidence at all").
```

### 5.3 Chart Body — Engine Selection

```svelte
<!-- PanelRenderer.svelte -->
<script lang="ts">
  import type { InferenceResult } from '$lib/types';
  import EChartsRenderer from './EChartsRenderer.svelte';
  import KPIRenderer from './KPIRenderer.svelte';
  import TableRenderer from './TableRenderer.svelte';

  export let result: InferenceResult;
</script>

<PanelHeader title={result.title} titleConfidence={result.title_confidence} />

{#if result.chart_winner === 'kpi'}
  <KPIRenderer {result} />
{:else if result.chart_winner === 'table'}
  <TableRenderer {result} />
{:else}
  <EChartsRenderer chartType={result.chart_winner} {result} />
{/if}

<PanelFooter {result} />
```

```
Why KPI and Table get dedicated Svelte components instead of
going through ECharts:
    KPI (DOC 5 Section 13.3 — col_span=3, the smallest panel
    type) is a number + label + optional trend arrow, not a
    chart in the visualization-library sense. Forcing it
    through ECharts would mean configuring an invisible axis
    system to display one number — unnecessary complexity.
    Table is plain HTML <table> for the same reason: DOC 5's
    8 V0.1 chart types (Section 13.3) include exactly one
    "show the raw rows" type, and HTML tables already do that
    correctly with zero charting-library overhead.
```

### 5.4 KPI Renderer — Consuming trend_direction (DOC 5 Section 16.2)

```svelte
<!-- KPIRenderer.svelte -->
<script lang="ts">
  export let result: InferenceResult;

  // feature_vector[38] is trend_direction (DOC 5 Section 16.2,
  // added in the v0.1.1 patch specifically so KPI Renderer
  // could show trend arrows correctly)
  $: direction = result.feature_vector[38];
  $: trendClass = direction > 0.65 ? 'positive'
                : direction < 0.35 ? 'negative'
                : 'neutral';
  $: trendIcon = direction > 0.65 ? '↑'
               : direction < 0.35 ? '↓'
               : '→';
</script>

<div class="kpi-body">
  <div class="kpi-value">{formatNumber(result.kpi_value)}</div>
  {#if result.feature_vector[28] > 0.5}
    <!-- only show a trend arrow if trend_strength (dim 28) is
         meaningful — a flat/noisy series shouldn't claim a
         confident direction (see DOC 5 Section 16.2 rationale:
         strength and direction are independent; UI must check
         BOTH, never direction alone) -->
    <span class="kpi-trend {trendClass}">{trendIcon}</span>
  {/if}
</div>

<style>
  .kpi-trend.positive { color: var(--sqlviz-positive); }
  .kpi-trend.negative { color: var(--sqlviz-negative); }
  .kpi-trend.neutral   { color: var(--sqlviz-neutral); }
</style>
```

### 5.5 Footer — Execution Metadata, Edit Mode Only

```svelte
<!-- PanelFooter.svelte -->
<script lang="ts">
  export let result: InferenceResult;
  import { editMode } from '$lib/stores/dashboard';
</script>

{#if $editMode}
  <div class="panel-footer">
    {formatMs(result.elapsed_ms)} · {formatRowCount(result.row_count)} rows · DuckDB
  </div>
{/if}
```

```
This was an explicit bug fix during the V0.1 prototype review
(rescued lesson): preview mode was showing execution metadata
and editor chrome that should only appear in edit mode. The
fix — gating on $editMode — is now a first-class rule of this
design system, not an afterthought: Section 6 makes the
preview/edit distinction explicit everywhere it matters.
```

---

## 6. Preview Mode vs Edit Mode

This distinction governs visibility of every "tool" element
(footers, overflow menus, drag handles, the SQL editor itself)
versus every "content" element (title, chart, filters).

```
                          Preview mode    Edit mode
─────────────────────────────────────────────────────
Panel title                  ✓                ✓
Chart / KPI / Table           ✓                ✓
FilterBar (if filters exist)  ✓                ✓
Execution metadata footer     ✗                ✓
Panel overflow menu [···]     ✗                ✓
SQL editor                    ✗                ✓ (Section 7)
Row/panel borders             subtle           visible
```

```css
/* Preview mode — rows have no visible chrome, panels float
   with just enough border to separate them visually */
.dashboard-grid:not(.edit-mode) .dashboard-row {
  border: none;
  background: transparent;
  padding: 0;
}

/* Edit mode — affordances become visible */
.dashboard-grid.edit-mode .dashboard-panel:hover .panel-overflow-menu {
  opacity: 1;
}
.dashboard-grid.edit-mode .dashboard-panel:hover .panel-footer {
  /* footer is always rendered in edit mode (Section 5.5);
     hover only affects the overflow menu's opacity, not the
     footer's existence */
}
```

---

## 7. The Minimal Override — When Inference Gets It Wrong

Per Section 3, manual layout controls are explicitly NOT the
primary interaction model. But DOC 1's Principle 2 also states
inference failures need *some* override — minimal and rare,
never the default path.

### 7.1 The Overflow Menu — Exactly Three Options

```
┌─────────────────┐
│ 📊 Change chart  │   → opens a dropdown of DOC 5's 8 V0.1
│                  │     chart types (Section 13.3); selecting
│                  │     one calls PATCH /panels/{id} with an
│                  │     explicit chart_type override, which
│                  │     sqlviz-api stores and uses to SKIP
│                  │     ChartEngine on subsequent re-renders
│                  │     of that panel (the override persists)
│ ✏️ Edit SQL       │   → opens the Monaco editor (Section 7.2)
│                  │     for this panel's query only
│ 🗑 Delete         │   → removes the panel; DashboardEngine
│                  │     re-composes the remaining panels
│                  │     (re-runs Section 15's row-packing,
│                  │     since removing a panel can change
│                  │     how the rest fit together)
└─────────────────┘
```

```
No layout/size/position options appear here — by design
(Section 3). If a chart override changes col_span needs
(e.g. switching KPI → Table needs more space), the
DashboardEngine recomputes layout automatically on save;
the user never manually sets a width or row.
```

### 7.2 The SQL Editor (Monaco)

```
Per DOC 3, Section 3: Monaco is the editor, chosen specifically
for handling CTEs, window functions, multi-line JOINs — the
real-world SQL complexity a lightweight editor (CodeMirror) was
rejected for in the V0.1 prototype.

Editor surface:
    - One Monaco instance per dashboard (DOC 2, Section 8 — "one
      SQL file per dashboard, queries separated by ;"), NOT one
      per panel. The "Edit SQL" overflow option (Section 7.1)
      scrolls/focuses the editor to that panel's specific
      statement rather than opening a separate per-panel editor.
    - Syntax highlighting: SQL, dialect-aware per DOC 2's
      connection model (DOC 2, Section "Engine roadmap" —
      DuckDB dialect in V0.1)
    - Theme: follows the global dark/light toggle (Section 2)
    - No autocomplete against live schema in V0.1 (logged as
      deferred in DOC 8, Section 6 features table — "schema
      autocomplete" is a V0.3 item, not V0.1)
```

---

## 8. Explainability UI — "Why This Chart?"

DOC 5's entire explainability apparatus (`score_trace`,
`explanation`, `confidence_gap`, `quality` — Section 1.4
Principle 5, and Sections 7.3/8.5 in detail) needs a UI surface,
or it exists only in API responses nobody sees. This is that
surface.

### 8.1 Trigger and Placement

```
A small "ⓘ" icon appears next to the panel title — but ONLY
when chart_quality is "medium" or "low" (DOC 5 Section
8.5/16.1), or when fallback_applied is true. A "high" quality,
high-confidence-gap inference shows NO indicator at all —
consistent with DOC 1 Principle 2 ("the user should rarely
need to think about this").
```

```svelte
<!-- PanelHeader.svelte, extended -->
{#if result.chart_quality !== 'high' || result.fallback_applied}
  <button class="explain-trigger" on:click={() => openExplainPanel(result)}>
    ⓘ
  </button>
{/if}
```

### 8.2 The Explain Panel Content

```
┌────────────────────────────────────────┐
│ Why a Line chart?                       │
│                                          │
│ Confidence: Medium                       │
│                                          │
│ Top signals:                             │
│   ✓ Temporal dimension detected   40%   │
│   ✓ Aggregation (SUM) detected    25%   │
│   ✓ GROUP BY present              20%   │
│                                          │
│ Other charts considered:                 │
│   Bar    — score 0.31                    │
│   Area   — score 0.28                    │
│                                          │
│ [📊 Use Bar instead]  [Keep Line]        │
└────────────────────────────────────────┘
```

```
Data sourced directly from InferenceResult, no client-side
computation:
    Confidence label  ← chart_quality (DOC 5 Section 8.5: maps
                         "high"/"medium"/"low" to a display label)
    Top signals        ← explanation[] (DOC 5 Section 7.4),
                         using contribution_pct once DOC 5's
                         V0.2 patch item lands (Section 16.3 —
                         until then, falls back to raw
                         contribution values, clearly NOT
                         percentages, per the same honesty
                         principle as DOC 4's "normalized_score
                         is not a true probability" stance)
    Other charts        ← chart_alternatives (DOC 5 Section 8.5)
    "Use X instead"     ← same override mechanism as Section 7.1
```

---

## 9. Filter Controls — Rendering DOC 5's 8 Control Types

DOC 5, Section 10.2 defines exactly 8 control types. This
section is their Svelte implementation — one component per
type, no client-side logic deciding WHICH type to use (that
decision already happened in FilterEngine).

```svelte
<!-- FilterBar.svelte -->
<script lang="ts">
  import type { FilterControl } from '$lib/types';
  import DatePicker from './filters/DatePicker.svelte';
  import DateRangePicker from './filters/DateRangePicker.svelte';
  import Dropdown from './filters/Dropdown.svelte';
  import MultiSelect from './filters/MultiSelect.svelte';
  import SearchInput from './filters/SearchInput.svelte';
  import NumericInput from './filters/NumericInput.svelte';
  import RangeSlider from './filters/RangeSlider.svelte';
  import Toggle from './filters/Toggle.svelte';

  export let controls: FilterControl[];  // DOC 5 Section 10.3 output

  const componentMap = {
    date_picker:       DatePicker,
    date_range_picker: DateRangePicker,
    dropdown:           Dropdown,
    multiselect:        MultiSelect,
    search:             SearchInput,
    numeric:            NumericInput,
    range_slider:       RangeSlider,
    toggle:             Toggle,
  };
</script>

<div class="filter-bar">
  {#each controls as control}
    <svelte:component
      this={componentMap[control.control_type]}
      variable={control.variable}
      label={control.label}
    />
  {/each}
</div>
```

```
FilterBar only renders when controls.length > 0 — there is no
"no filters" placeholder state; a dashboard with zero $variable
usage simply has no filter bar, taking zero vertical space
(consistent with DOC 1's "infer everything, show nothing the
user didn't ask for" philosophy).
```

---

## 10. Authentication Screens — Rendering DOC 7's Flows

### 10.1 Admin Login

```
┌──────────────────────┐
│       SQLviz          │
│                       │
│  Admin password        │
│  [____________]       │
│                       │
│  [    Sign in    ]    │
└──────────────────────┘
```

```
Maps directly to DOC 7, Section 3.2 login flow. The generic
error message requirement (DOC 7 Section 3.2 — "never reveal
whether the project even has a password set") means this screen
NEVER shows "incorrect password" vs "no account" as distinct
states — there is exactly one error string for any failure:
"Invalid password."
```

### 10.2 Viewer — Password-Protected Share

```
┌──────────────────────┐
│   Revenue Analysis     │
│   (password protected) │
│                        │
│  [____________]       │
│                        │
│  [     View      ]    │
└──────────────────────┘
```

```
This is the ONLY screen a viewer ever sees that isn't the
dashboard itself, and only for mode='password' shares (DOC 7,
Section 4.2). Private and public mode viewers go straight to
the dashboard with zero intermediate screen — consistent with
DOC 7 Section 4.2's exact flow description.
```

---

## 11. What Is Deferred — Cross-Referenced

Consistent with the discipline established in DOC 5 (Section
16.3-16.4), DOC 7 (Section 7), and DOC 8 (Section 6): nothing
discussed and set aside during this document's writing is
silently dropped.

```
Item                              Why deferred              Returns
─────────────────────────────────────────────────────────────────
Manual layout override            DOC 1 v0.2 vision           V0.2
 (drag, resize, custom rows)       explicitly removes manual
                                  layout as a PRIMARY flow;
                                  if real usage shows the
                                  Section 7.1 minimal override
                                  is insufficient for edge
                                  cases, a manual escape hatch
                                  beyond "change chart type"
                                  may be reconsidered then —
                                  not before.

Schema-aware SQL autocomplete      Logged in DOC 8 Section 6   V0.3
 in Monaco (Section 7.2)

contribution_pct in Explain         Depends on DOC 5 Section    V0.2
Panel (Section 8.2)                16.3 item landing first
                                  in sqlviz-inference — UI
                                  consumes whatever shape
                                  the API returns, cannot
                                  get ahead of it

Cross-filtering (click a bar         DOC 5 Section 15.7 +       V0.2
 to filter another panel)            DOC 8 Section 6 — this
                                    is a Dashboard Engine
                                    capability first, UI
                                    second; UI work waits
                                    for that API shape to exist

Insight/Narrative text in panels      DOC 5 Section 16.4         V0.3
 ("Revenue grew 12%")                 (Insight Engine,
                                     Narrative Engine) —
                                     no UI surface designed
                                     until the backend
                                     produces this data

Domain-specific theming               not previously discussed;   not
 (e.g. an "Energy" color               logged here only because    planned
 preset for utility dashboards)        DOC 5 Section 16.3's
                                     Domain Dictionaries raised
                                     the general "domain
                                     packs" idea — a themed
                                     visual counterpart is
                                     speculative and not
                                     committed to any version
```

---

## 12. Definition of Done — UI Design System (feeds DOC 8, Phase 5)

```
[ ] Design tokens (Section 2) implemented as CSS variables,
    dark theme default, light theme via [data-theme="light"]
[ ] DashboardGrid renders DashboardLayout (DOC 5 Section 15.4)
    with zero client-side layout decisions — col_span/row_span
    consumed directly
[ ] Panel anatomy (Section 5) — KPI, Table, and ECharts-backed
    panels all render through PanelRenderer's single dispatch
[ ] KPI trend arrow correctly reads BOTH dim 28 (strength) AND
    dim 38 (direction) before showing any arrow (Section 5.4 —
    this is the one place a UI bug could silently misrepresent
    DOC 5's explicit strength/direction separation from Section
    16.2; must be tested explicitly)
[ ] Preview mode hides footer, overflow menu, and SQL editor;
    edit mode shows all three (Section 6)
[ ] Overflow menu offers exactly three options — chart change,
    edit SQL, delete — no layout/size controls (Section 7.1)
[ ] One Monaco instance per dashboard, not per panel (Section 7.2)
[ ] Explain panel (Section 8) appears ONLY when quality != high
    or fallback_applied — never on high-confidence panels
[ ] All 8 filter control types (Section 9) render from
    FilterControl.control_type with no client-side type logic
[ ] Admin login and password-protected viewer screens (Section
    10) match DOC 7's exact flows, including the generic
    error-message requirement
[ ] Section 11's deferred list reviewed and accurate before
    V0.1 ships
```

---

*SQLviz UI Design System — v0.1.0 Draft*
*"The frontend never infers. The frontend only renders."*
