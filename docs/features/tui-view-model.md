# TUI View Model — V1 Proposal

!!! warning "Status: PROPOSED — not implemented"
    This document is a design proposal awaiting consumer demand. No code ships
    against this spec in v0.7.x. Implementation begins when a fourth concrete
    consumer surfaces (most likely trigger: `aisee-cli` adopting `--format table`,
    or a browser dashboard in `tiptap-apcore` / `apcore-studio` needing
    structured module listings).

    | | |
    |---|---|
    | **Author** | apcore-toolkit maintainers |
    | **First drafted** | 2026-05-12 |
    | **Target release** | 0.8.0 (earliest) |
    | **Tracking issue** | [aiperceivable/apcore-toolkit#14](https://github.com/aiperceivable/apcore-toolkit/issues/14) |
    | **Depends on** | `apcore-toolkit/docs/features/formatting.md` (byte-equivalent contract precedent) |
    | **Affects** | `apcore-cli-{python,typescript,rust}` (Phase 2, separate PR) |

---

## Goal

Lift the **shape** of a module-list view (columns, rows, filter intent, sort
intent, color-by-tag rules) into the toolkit as a Tier-1 byte-equivalent
data structure, so every downstream consumer — `apcore-cli-*`, `aisee-cli`,
future browser dashboards, MCP/A2A surfaces — produces identical column
sets, identical filter semantics, and identical row order for the same
`ScannedModule` input.

The rendering itself (rich tables in Python, hand-rolled tables in TypeScript,
`comfy-table` in Rust, future `textual` / `ink` / `ratatui` for interactive
modes) stays Tier 2 and remains free to differ in pixels.

## Non-Goals

This proposal is intentionally narrow:

- **No detail-view model.** The `apcore-cli describe` command renders panels,
  schemas, and free-form key/value blocks. That is a fundamentally different
  shape than a row-oriented list and warrants its own future
  `TuiDetailViewModel` proposal. V1 covers list and grouped-list only.
- **No interactive renderer.** V1 emits a static `TuiViewModel`. Building an
  interactive TUI (live filter, drill-down, keyboard navigation) is a
  separate effort that *consumes* this model but is out of scope here.
- **No raw color in the wire format.** ANSI codes, RGB hex, library-specific
  style strings are renderer concerns. The wire format carries semantic
  *tones* only (`positive` / `negative` / `warning` / `info` / `neutral`).
- **No numeric/floating-point cells.** All numbers are pre-formatted to
  strings by the builder. Avoids cross-language float-rendering divergence
  (Java/Elixir/Jackson serialize `1.0` as `"1.0"`; JS/.NET serialize as
  `"1"`).
- **No HTML/markdown escaping in the wire format.** Cells carry plain
  strings; each renderer applies escaping appropriate to its surface.

## Motivation

### The divergence we have today

Three independent table renderers exist in the apcore ecosystem (as of
v0.7.0):

| Aspect | `apcore-toolkit` `style="table-row"` | `apcore-cli-python` `format_module_list` | `apcore-cli-typescript` `formatTable` | `apcore-cli-rust` `format_module_list` |
|---|---|---|---|---|
| Columns | `id │ alias │ description │ tags` | `ID, Description, Tags` (+ optional `Deps`, `Exposure`) | `ID, Description, Tags` (+ optional `Deps`, `Exposure`) | `ID, Description, Tags` |
| Grouped column rename | — | `ID` → `Command` (only in grouped mode) | — | — |
| Library | string concat with `│` U+2502 | `rich.Table` | hand-rolled `padEnd` | `comfy_table::Table` |
| Description truncation | none | 80 chars | 80 chars | `DESCRIPTION_TRUNCATE_LEN` |
| Filter axes | none | 6 (tag, search, annotation, deprecated, status, exposure) | subset | subset |
| Sort keys | none | 4 (`id`, `calls`, `errors`, `latency`) | subset | subset |
| Display-overlay precedence | toolkit `surface.py:175-182` | CLI `display_helpers.py:21-29` (different!) | CLI-local | CLI-local |

Three concrete consequences of this divergence already shipped to users:

1. **Grouped-mode column-name surprise.** Users invoking
   `apcore-cli user list --format table` in grouped mode see a `Command`
   column header; the same command in flat mode shows `ID`. The same module
   list in `apcore-cli-typescript` shows neither — it stays `ID` regardless.
2. **Filter semantics differ across SDKs.** `--tag X --tag Y` is AND-filter
   in Python but is an intersect-or-union toss-up in the other two CLIs.
3. **Display-overlay precedence drift.** The toolkit resolves
   `display.alias > module_id`; `apcore-cli-python` resolves
   `display.cli.alias > display.alias > canonical_id`. Two different answers
   from the same `ScannedModule` + binding overlay input.

### The csv/jsonl precedent

Before v0.7.0, `csv` and `jsonl` were Tier-2 SDK-native. The result: three
independent bugs (Python repr of nested dicts producing invalid JSON;
TypeScript header derivation losing data on heterogeneous rows; Rust
`\n` instead of CRLF). Lifting csv/jsonl to Tier 1 with a shared conformance
corpus eliminated this entire bug class at once. The user-visible cost was
zero; the maintenance saving has been substantial.

`TuiViewModel` is the same kind of lift, applied one layer higher: the data
*about* a table rather than the table bytes themselves.

## Tier Model (refines ADR-09 in `apcore-cli/docs/tech-design.md`)

| Tier | Members | Conformance | Owner |
|---|---|---|---|
| **Tier 1** — Byte-equivalent toolkit-delegated | `csv`, `jsonl`, `markdown`, `skill`, **`view_model` (NEW)** | Byte-identical JSON across all SDKs, asserted by `conformance/fixtures/` corpus | `apcore-toolkit` |
| **Tier 2** — SDK-native presentation | `table` (renders `view_model` via rich / hand-rolled / comfy-table); future `tui` (renders `view_model` via textual / ink / ratatui) | Behavioural equivalence only — pixels may differ | Each `apcore-cli-{lang}` |
| **Tier 3** — Trivial stdlib | `json` | Native JSON output | Each SDK |

The promotion: a `--format table` invocation now goes through two stages —
**(1)** the toolkit produces a `TuiViewModel` (Tier 1, byte-identical),
**(2)** the SDK's renderer turns that into terminal output (Tier 2, pixel
free). Row order, column set, filter semantics, and tone-classification
become byte-identical guarantees; rendered widths, separators, and colors
remain SDK-idiomatic.

## Wire Format

The `TuiViewModel` is a JSON-shaped data structure. Each SDK ships:

1. A native type (`@dataclass` in Python, `interface` in TypeScript, `struct`
   in Rust) representing the model.
2. A pure function `modules_to_view_model(modules, options) -> TuiViewModel`.
3. A canonical encoder `format_view_model(vm) -> str` producing byte-identical
   JSON, asserted by the shared conformance corpus.

### Canonical example

```jsonc
{
  "schema_version": 1,
  "kind": "list",
  "title": "Modules",
  "columns": [
    { "key": "module_id",   "label": "ID",          "justify": "left",  "tone_by": null },
    { "key": "description", "label": "Description", "justify": "left",  "tone_by": null },
    { "key": "tags",        "label": "Tags",        "justify": "left",  "tone_by": "tag_palette" }
  ],
  "rows": [
    {
      "cells": [
        { "kind": "text",  "value": "users.get_user" },
        { "kind": "text",  "value": "Get a user by ID" },
        { "kind": "tags",  "values": ["users", "read-only"] }
      ],
      "tags": ["users", "read-only"]
    }
  ],
  "sort":   { "key": "module_id", "direction": "asc" },
  "filter": { "tags": ["users"], "search": "", "annotations": [], "exposure": "all", "deprecated": true },
  "tone_palettes": [
    {
      "name": "tag_palette",
      "rules": [
        { "match": { "kind": "tag_equals", "value": "deprecated" }, "tone": "warning" },
        { "match": { "kind": "tag_equals", "value": "read-only" }, "tone": "info" }
      ]
    }
  ]
}
```

### Schema

#### Top-level envelope

| Field | Type | Required | Notes |
|---|---|---|---|
| `schema_version` | integer | yes | `1` for V1. Renderers fall back to `text` cell rendering on unknown future kinds. |
| `kind` | string enum | yes | `"list"` \| `"grouped"`. `"detail"` reserved for V2. |
| `title` | string | optional | Header text for the rendered view. Omit when absent. |
| `columns` | array of `Column` | yes | Ordered; defines render order and `key` lookup into rows. |
| `rows` | array of `Row` | yes | Pre-filtered, pre-sorted. Renderers do not re-order. |
| `groups` | array of `Group` | optional, only when `kind == "grouped"` | Each entry maps a group label to row indices in `rows`. Omit when absent. |
| `sort` | `Sort` | optional | Annotates which sort the toolkit applied; renderers may surface this in UI. Omit when no sort was requested. |
| `filter` | `Filter` | optional | Annotates which filter the toolkit applied. Omit when no filter. |
| `tone_palettes` | array of `TonePalette` | optional | Referenced by `Column.tone_by`. Omit when no column uses tone. |

#### Column

| Field | Type | Required | Notes |
|---|---|---|---|
| `key` | string | yes | Stable identifier; matches `Cell` position in row `cells` array (by index, not by key — see `Row` below). |
| `label` | string | yes | Header text for the rendered column. |
| `justify` | string enum | optional | `"left"` (default) \| `"right"` \| `"center"`. Omit when `"left"`. |
| `tone_by` | string | optional | References a `TonePalette.name`. Omit when no tone applies. |

#### Row

| Field | Type | Required | Notes |
|---|---|---|---|
| `cells` | array of `Cell` | yes | Position-indexed: `row.cells[i]` corresponds to `columns[i]`. Length must equal `columns.length`. |
| `tags` | array of strings | optional | Used by `tone_by` palette `tag_equals` rule. Omit when empty. |

#### Cell (discriminated union by `kind`)

| `kind` | Additional fields | Notes |
|---|---|---|
| `"text"` | `value: string` | Plain text. |
| `"tags"` | `values: array of strings` | Renderer joins with idiomatic separator. |
| `"badge"` | `value: string`, `tone?: Tone` | Short label; renderer may box it. |
| `"symbol"` | `value: string` (one of `"check"`, `"cross"`, `"warning"`, `"circle"`), `tone?: Tone` | Renderer chooses glyph: ✓ / ✗ / ⚠ / ○ or ASCII fallback. |

#### Sort

| Field | Type | Required | Notes |
|---|---|---|---|
| `key` | string | yes | Must match a `columns[].key`. |
| `direction` | string enum | yes | `"asc"` \| `"desc"`. |

Allowed `key` values in V1 (the toolkit pre-resolves these):
`module_id`, `alias`, `description`. Other keys (`calls`, `errors`,
`latency`) may appear in the `sort` annotation but the SDK is responsible
for delivering pre-sorted modules to `modules_to_view_model` — see
[Sort/Filter Execution Model](#sortfilter-execution-model) below.

#### Filter

| Field | Type | Required | Notes |
|---|---|---|---|
| `tags` | array of strings | yes (may be empty) | AND-filter across all listed tags. |
| `search` | string | yes (may be empty) | Substring match over `module_id` + `description`. Case-insensitive. |
| `annotations` | array of strings | yes (may be empty) | Names of `ModuleAnnotations` flag fields that must be `true`. |
| `exposure` | string enum | yes | `"exposed"` \| `"hidden"` \| `"all"`. |
| `deprecated` | boolean | yes | When `false`, modules with `annotations.deprecated == true` are excluded. |

Empty strings and empty arrays appear in the canonical encoding to make the
"filter applied" annotation unambiguous; `null` is never emitted.

#### TonePalette

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | Referenced by `Column.tone_by`. |
| `rules` | array of `ToneRule` | yes | First-match wins. |

#### ToneRule

| Field | Type | Required | Notes |
|---|---|---|---|
| `match` | object | yes | Currently only one shape supported: `{ "kind": "tag_equals", "value": "<tag>" }`. Future kinds reserved. |
| `tone` | Tone enum | yes | See below. |

#### Tone (semantic, not visual)

| Value | Conventional meaning | Suggested Python `rich` style | Suggested TypeScript `chalk` color | Suggested Rust `comfy-table` |
|---|---|---|---|---|
| `"neutral"` | default text | unstyled | unstyled | default |
| `"positive"` | success, present, healthy | `green` | `green` | `Color::Green` |
| `"negative"` | failure, deprecated, error | `red` | `red` | `Color::Red` |
| `"warning"` | attention, deprecated, partial | `yellow` | `yellow` | `Color::Yellow` |
| `"info"` | read-only, advisory | `cyan` | `cyan` | `Color::Cyan` |

Renderers MUST map these five tones. Other tones are not permitted in V1.

#### Group (only when `kind == "grouped"`)

| Field | Type | Required | Notes |
|---|---|---|---|
| `label` | string | yes | Group header text. |
| `row_indices` | array of integers | yes | Indices into the top-level `rows` array. Renderer iterates in this order; rows not referenced by any group are not rendered. |

## Sort/Filter Execution Model

Most filters and sorts execute inside the toolkit's
`modules_to_view_model(modules, ...)` so that row order is byte-identical
across SDKs.

Exception: **usage-based sort** (`calls`, `errors`, `latency`) requires per-
SDK telemetry data (e.g., `~/.apcore-cli/audit.jsonl` in Python). The toolkit
does not own that data. Contract:

- SDK reads its telemetry, computes a sorted `ScannedModule[]`, then passes
  that order to `modules_to_view_model(modules, sort=Sort(key="calls", direction="desc"))`.
- The toolkit honours the incoming module order verbatim when the
  `sort.key` is one of `"calls"`, `"errors"`, `"latency"`.
- The toolkit applies its own ordering only when `sort.key` is one of
  `"module_id"`, `"alias"`, `"description"`.
- The resulting ViewModel's `sort` field records what was requested,
  regardless of whether the toolkit or the SDK produced the order.

`status` filtering (enabled/disabled — depends on per-module runtime state
not present on `ScannedModule`) follows the same SDK-side pattern: the SDK
filters the module list before passing to `modules_to_view_model`. The
filter's `exposure` and `deprecated` flags are resolved by the toolkit
because they depend only on `ScannedModule.annotations`.

## Canonical JSON Encoding (Byte-Equivalent)

The same rules already governing `format_csv` and `format_jsonl` apply
(see [formatting.md](formatting.md) § Tabular Formats):

1. **Field declaration order = JSON key order.** No alphabetic sort; no
   omission of empty arrays for required fields.
2. **Optional fields are omitted, never `null`.** No `"title": null`.
   Required fields like `filter.search` may be empty strings (`""`) when
   absent — distinguish "not applicable" (omit) from "applied with empty
   value" (emit empty).
3. **Booleans: lowercase `true` / `false` only.** No `0/1`, no string
   coercion.
4. **Strings: UTF-8, no escaping beyond JSON minimum.**
5. **No floating-point.** Cells with numeric semantics carry pre-formatted
   strings.
6. **Line terminator: none.** The encoded model is a single JSON document
   (not JSONL).
7. **Compact JSON: no whitespace between tokens** — match `format_jsonl`
   exactly.

These rules are derived from the multi-language feasibility study
(see [Open Questions](#open-questions) for the underlying constraints) and
are intentionally stricter than minimum JSON to remain encodable in all
future SDK languages.

## Public API

### Python

```python
from apcore_toolkit import (
    TuiViewModel, Column, Row, Cell, Sort, Filter, TonePalette, ToneRule,
    modules_to_view_model, format_view_model,
)

vm = modules_to_view_model(
    modules,                                      # list[ScannedModule]
    view="list",                                  # "list" | "grouped"
    columns=("module_id", "description", "tags"),
    filter=Filter(tags=("users",), search="", annotations=(),
                  exposure="all", deprecated=True),
    sort=Sort(key="module_id", direction="asc"),
    group_by=None,                                # "tag" | "prefix" | None
    tone_palettes=(),
    display=True,                                 # honour ScannedModule.display overlay
)

canonical_json: str = format_view_model(vm)
```

### TypeScript

```typescript
import {
  TuiViewModel, Column, Row, Cell, Sort, Filter, TonePalette, ToneRule,
  modulesToViewModel, formatViewModel,
} from "apcore-toolkit";

const vm: TuiViewModel = modulesToViewModel(modules, {
  view: "list",
  columns: ["module_id", "description", "tags"],
  filter: { tags: ["users"], search: "", annotations: [],
            exposure: "all", deprecated: true },
  sort: { key: "module_id", direction: "asc" },
  groupBy: null,
  tonePalettes: [],
  display: true,
});

const canonicalJson: string = formatViewModel(vm);
```

Also re-exported from `apcore-toolkit/browser` (the model is pure data with
no Node.js dependency).

### Rust

```rust
use apcore_toolkit::{
    TuiViewModel, Column, Row, Cell, Sort, Filter, TonePalette,
    modules_to_view_model, format_view_model,
};

let vm = modules_to_view_model(
    &modules,
    &ModulesToViewModelOptions {
        view: View::List,
        columns: vec!["module_id".into(), "description".into(), "tags".into()],
        filter: Some(Filter { tags: vec!["users".into()], search: String::new(),
                              annotations: vec![], exposure: Exposure::All,
                              deprecated: true }),
        sort: Some(Sort { key: "module_id".into(), direction: Direction::Asc }),
        group_by: None,
        tone_palettes: vec![],
        display: true,
    },
);

let canonical_json: String = format_view_model(&vm);
```

## Conformance Corpus

A shared corpus lives at `apcore-toolkit/conformance/fixtures/view_model/`,
parallel to `format_csv.json` and `format_jsonl.json`. Each fixture is a
pair `<name>.input.json` (a list of `ScannedModule` plus options) and
`<name>.expected.json` (the byte-identical canonical encoding of the
resulting `TuiViewModel`).

Minimum V1 fixture set:

| Fixture | Purpose |
|---|---|
| `empty_list.json` | Zero modules, no filter, no sort |
| `basic_list_three_columns.json` | 3 modules, default columns |
| `filter_tag_intersect.json` | Two tags, AND semantics |
| `filter_search_case_insensitive.json` | Substring match across `module_id` + `description` |
| `filter_exposure_hidden.json` | Hidden-only filter |
| `sort_asc_module_id.json` | Ascending sort by `module_id` |
| `sort_desc_description.json` | Descending sort by `description` |
| `grouped_by_tag.json` | `kind: "grouped"` with two groups |
| `grouped_by_prefix.json` | `kind: "grouped"` by `module_id` prefix |
| `tone_palette_deprecated_warning.json` | `tag_equals` rule emits `warning` tone |
| `display_overlay_alias.json` | Honours `ScannedModule.display.alias` over `module_id` |

Each SDK's test suite runs the corpus; CI fails on any byte divergence.

## Migration Plan

### Phase 1 — Toolkit (this proposal, when greenlit)

In `apcore-toolkit-{python,typescript,rust}`:

1. Add `TuiViewModel` + supporting types.
2. Implement `modules_to_view_model` + `format_view_model`.
3. Add conformance corpus + per-SDK conformance tests.
4. Re-export from package root (and TypeScript `browser` subpath).
5. Add `## Contract: modules_to_view_model` and `## Contract: format_view_model`
   blocks to this file.

No CLI behaviour changes in this phase.

### Phase 2 — apcore-cli (separate PR per SDK)

In `apcore-cli-{python,typescript,rust}`:

1. Add `render_view_model(vm)` returning the SDK's native renderable
   (`rich.Table` in Python; rendered string in TypeScript;
   `comfy_table::Table` in Rust).
2. Rewrite `format_module_list("table", ...)` to build a `TuiViewModel`
   then call `render_view_model`.
3. Delete the per-SDK filter/sort implementations now living in
   `discovery.py:135-233` (Python) and equivalents elsewhere.
4. Resolve the column-name divergence: standardise on `ID` in both flat and
   grouped mode (current Python grouped mode rename to `Command` deprecated).
5. Align display-overlay precedence with the toolkit's resolution chain.

This is a user-visible behaviour change (column name in grouped mode, filter
semantics across SDKs converge) and warrants a minor version bump on
apcore-cli.

### Phase 3 — Future surfaces (deferred)

- **Interactive TUI**: `apcore-cli list --format tui` consuming the same
  `TuiViewModel` via `textual.DataTable` / `ink-table` / `ratatui::Table`.
- **Browser dashboards**: tiptap-apcore / apcore-studio consuming the
  `apcore-toolkit/browser` export to render HTML/React tables from the
  same view model.
- **`TuiDetailViewModel`** (separate proposal): the detail-view counterpart
  modelled as typed sections + key/value blocks + schema trees.

## Open Questions

These are intentionally listed so contributors can weigh in before V1
implementation locks the schema:

1. **Tone palette: 5 semantic tones vs more?** Current proposal:
   `neutral / positive / negative / warning / info`. Argument for: matches
   common terminal style mappings; renderer authors have <10 mappings to
   write. Argument against: not enough room for "muted" or "emphasis".
   **Recommendation: lock at 5 for V1; add via schema_version bump if needed.**
2. **Cell kinds: 4 vs more?** Current: `text / tags / badge / symbol`.
   Argument against: no way to embed nested JSON (e.g. annotations
   bag) in a cell. Argument for: nested JSON in a TUI table is a
   smell — that data belongs in a detail view.
   **Recommendation: lock at 4 for V1.**
3. **Status filter (enabled/disabled): toolkit or SDK?** Currently SDK
   because the data isn't on `ScannedModule`. Could be moved to toolkit
   if we ever add a `status` field to `ScannedModule`. **Recommendation:
   SDK in V1.**
4. **Display-overlay precedence: align CLI to toolkit, or vice versa?**
   Toolkit currently has `display.alias > module_id`; CLI has
   `display.cli.alias > display.alias > canonical_id`. The CLI chain is
   richer and probably correct.
   **Recommendation: lift the CLI chain into the toolkit's
   `_resolve_display_fields` and align both sides on it. Document in
   `display-overlay.md` Contract block.**
5. **Schema versioning policy.** Renderers MUST tolerate unknown future
   `Cell.kind` or `Tone` values by falling back to plain text. New fields
   on existing types are non-breaking. Renames of existing fields are
   breaking and require a `schema_version` bump. **Recommendation:
   document explicitly in this spec before V1 lands.**

## Future-Language Readiness

This spec is intentionally designed to remain consumable in Go, Kotlin,
Swift, C#/.NET, PHP, Elixir, Java (in addition to the current Python,
TypeScript, Rust). The constraints in [Canonical JSON Encoding](#canonical-json-encoding-byte-equivalent)
encode this directly:

| Constraint | Languages it protects |
|---|---|
| No open maps — list-of-objects everywhere | Go (`encoding/json` sorts `map[string]T` alphabetically); Swift (`Dictionary` unordered); Elixir (regular maps unordered above 32 entries) |
| No floats — pre-formatted strings | Java (`Double` whole-number `1.0` vs JS `1`); Elixir (`:erlang.float_to_binary` defaults to `1.0`); Jackson `BigDecimal` trailing zeros |
| No `null` — omit absent | Go (`omitempty` conflates zero-value with absent); Swift (`Codable` emits `nil` as `null` by default); Jackson opt-in required |
| Lowercase `true`/`false` | PHP loose typing; ORM adapters that leak `1`/`0` |
| `snake_case` ASCII keys + lowercase enum strings | Swift / STJ / Jackson all default to non-snake naming policies |

Locking these in V1 is cheap (consistent with the existing `formatting.md`
contract); adding them retroactively would force a `schema_version` bump
and corpus rewrite.

## Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| Over-engineering for current 3-consumer reality | Medium | Defer Phase 1 until a 4th consumer surfaces (most likely `aisee-cli` adopting `--format table`). Spec exists as proposal; implementation is gated. |
| Schema-evolution debt — V1 schema breaks needed | Medium | Lock the encoding rules conservatively (no floats, no null, no maps). Document `schema_version` bump policy. |
| `TuiCell.kind = "json"` becomes load-bearing later | Low-medium | Defer adding it until a specific consumer needs it. The discriminated-union design makes additions non-breaking. |
| `TonePalette` design (`tag_equals` only) too narrow | Low | Add `match.kind` variants (e.g. `annotation_true`, `regex`) when needed. The current rule is the only one demanded by existing CLI behaviour. |
| Apcore-cli migration (Phase 2) is more work than expected | Medium | Each SDK's renderer is independent. Migrate one SDK at a time. Original `format_module_list` can stay in place as fallback during transition. |
| Display-overlay precedence change breaks downstream users | Low-medium | Document explicitly in `display-overlay.md`; bump `apcore-cli` minor version. |

## Implementation Estimate

| Phase | Component | Estimated LOC | Notes |
|---|---|---|---|
| Phase 1 | Python types + builder + encoder + tests | ~400 | Includes 11 conformance fixtures × ~30 lines each |
| Phase 1 | TypeScript types + builder + encoder + tests | ~400 | Same |
| Phase 1 | Rust types + builder + encoder + tests | ~400 | Same |
| Phase 1 | Conformance corpus | ~300 | Shared across SDKs |
| Phase 1 | Spec doc updates (this file, `formatting.md`, README) | ~200 | |
| **Phase 1 total** | | **~1700 LOC** | ~2.5× the v0.7.0 csv/jsonl effort |
| Phase 2 | apcore-cli-python migration | ~-120 net (renderer +80, filter/sort -200) | |
| Phase 2 | apcore-cli-typescript migration | ~-120 net | |
| Phase 2 | apcore-cli-rust migration | ~-120 net | |
| **Phase 2 total** | | **~-360 net LOC** (less code overall) | |
| Phase 3 | Interactive TUI per SDK | ~150 LOC each | Deferred until concrete demand |

## See Also

- [`formatting.md`](formatting.md) § Tabular Formats — the existing
  byte-equivalent tier this proposal mirrors.
- [`scanning.md`](scanning.md) § Contract: ScannedModule — input data type.
- [`display-overlay.md`](display-overlay.md) — resolution chain to align.
- `apcore-cli/docs/tech-design.md` § ADR-09 — the tier model this proposal
  refines.
