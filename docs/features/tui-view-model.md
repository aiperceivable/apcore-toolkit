---
description: "Tier-1 byte-equivalent TuiViewModel lifting module-list shape (columns, rows, filter/sort/color-by-tag) across consumers."
---

# TUI View Model — V1

!!! success "Status: IMPLEMENTED (Phase 1 — toolkit)"
    `TuiViewModel` + `modules_to_view_model` / `format_view_model` ship in
    `apcore-toolkit-{python,typescript,rust}`, asserted byte-identical
    across all three SDKs by the shared conformance corpus below. Phase 2
    (`apcore-cli-*` adopting this as its `--format table` renderer,
    replacing each SDK's local filter/sort/table code) is separate,
    tracked work — no CLI behaviour has changed yet.

    | | |
    |---|---|
    | **Author** | apcore-toolkit maintainers |
    | **First drafted** | 2026-05-12 |
    | **Implemented** | 2026-09-04 |
    | **Tracking issue** | [aiperceivable/apcore-toolkit#14](https://github.com/aiperceivable/apcore-toolkit/issues/14) |
    | **Depends on** | `apcore-toolkit/docs/features/formatting.md` (byte-equivalent contract precedent) |
    | **Affects** | `apcore-cli-{python,typescript,rust}` (Phase 2, separate PR — not yet started) |

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

---

## Contract: modules_to_view_model

### Inputs
- `modules`: `list[ScannedModule]` / `ScannedModule[]` / `&[ScannedModule]`, required
- `view` / `View`: `"list"` \| `"grouped"`, optional, default `"list"`
- `columns`: array of column-key strings, optional, default **empty** — there is no built-in default column set; an empty `columns` array yields an empty `columns` array in the output (see conformance fixture `view_model_001_empty_list`). Callers wanting the conventional layout pass `["module_id", "description", "tags"]` explicitly.
- `title`: string, optional
- `filter` / `Filter`: optional
- `sort` / `Sort`: optional — only `sort.key` of `module_id` / `alias` / `description` is executed by the toolkit; any other key is annotated in the output but the caller must pre-sort the incoming `modules` (see [Sort/Filter Execution Model](#sortfilter-execution-model))
- `group_by` / `groupBy`: `"tag"` \| `"prefix"` \| `None`, optional — meaningful only when `view == "grouped"`; `None` groups everything under a single `"(all)"` group
- `tone_palettes` / `tonePalettes`: array of `TonePalette`, optional, default empty — V1 wires the **first** supplied palette's `name` to the `"tags"` column's `tone_by`; there is no per-column palette-selection parameter in V1. Per-tag-value tone resolution (which tag chip in a multi-tag cell gets which colour) is a Tier-2 renderer concern, not computed by the builder.
- `display`: boolean, optional, default `true` — honour `ScannedModule.display` overlay for `alias`/`description` (the toolkit's existing `display.alias > module_id` precedence — see Open Question 4 on aligning with the richer CLI chain)

### Errors
- None raised — degrades gracefully. An unrecognised column key (anything other than `module_id` / `alias` / `description` / `tags`) renders as an empty `"text"` cell rather than raising.

### Returns
- On success: `TuiViewModel`

### Properties
- async: false
- pure: true
- deterministic: true — identical input yields byte-identical `format_view_model(...)` output across all three SDKs
- thread_safe: true — no mutation of input `modules`

---

## Contract: format_view_model

### Inputs
- `vm` / `view_model`: `TuiViewModel`, required

### Errors
- None

### Returns
- On success: string — compact canonical JSON per [Canonical JSON Encoding](#canonical-json-encoding-byte-equivalent): declaration-order keys, optional fields omitted (never `null`), no whitespace between tokens

### Properties
- async: false
- pure: true
- deterministic: true — the primary subject of the conformance corpus below

---

## Contract: TuiViewModel

### Inputs
N/A — this is a data type, not a callable function. Produced by [`modules_to_view_model`](#contract-modules_to_view_model) and consumed by [`format_view_model`](#contract-format_view_model) and by each SDK's Tier-2 renderer.

### Errors
N/A — data types are not called and do not raise secondary exceptions.

### Returns
N/A — data types are not called and do not return values.

### Construction
- Python: `@dataclass` — always built via `modules_to_view_model(...)`, never constructed directly by consumers
- TypeScript: `interface TuiViewModel` — a plain object shape, likewise always produced by `modulesToViewModel(...)`
- Rust: `struct TuiViewModel` — produced by `modules_to_view_model(...)`

### Fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `schema_version` | integer | yes | `1` for V1. Renderers fall back to plain `"text"` cell rendering on unknown future `Cell.kind` / `Tone` values rather than erroring. |
| `kind` | string enum | yes | `"list"` \| `"grouped"`. `"detail"` is reserved for a future `TuiDetailViewModel`. |
| `title` | string | optional | Header text for the rendered view. Omitted from the canonical encoding when absent — never emitted as `null`. |
| `columns` | array of [`Column`](#contract-column) | yes | Ordered; defines render order and index-based lookup into each `Row.cells`. May be empty (there is no built-in default column set). |
| `rows` | array of [`Row`](#contract-row) | yes | Pre-filtered, pre-sorted by the toolkit (or by the caller, for usage-based sort keys — see [Sort/Filter Execution Model](#sortfilter-execution-model)). Renderers MUST NOT re-order. |
| `groups` | array of [`Group`](#contract-group) | optional | Present only when `kind == "grouped"`. Omitted (never an empty array) when absent. |
| `sort` | [`Sort`](#contract-sort) | optional | Annotates which sort the toolkit applied or was asked to apply. Omitted when no sort was requested. |
| `filter` | [`Filter`](#contract-filter) | optional | Annotates which filter the toolkit applied. Omitted when no filter. |
| `tone_palettes` | array of [`TonePalette`](#contract-tonepalette) | optional | Referenced by `Column.tone_by`. Omitted when no column uses tone. |

### Properties
- immutable by convention (Python/TypeScript); owned value (Rust)
- camelCase field names in TypeScript; snake_case in Python and Rust
- pure data: never carries a callable, a file handle, or any other non-serializable value — the whole point is that `format_view_model(vm)` is a pure, byte-identical encoding across all three SDKs
- schema-versioned: new *optional* fields on any of the nine types below are non-breaking; renaming or removing an existing field is breaking and requires a `schema_version` bump (Decision 5)

---

## Contract: Column

### Inputs
N/A — this is a data type, not a callable function. See [`TuiViewModel`](#contract-tuiviewmodel).

### Errors
N/A — data types are not called and do not raise secondary exceptions.

### Returns
N/A — data types are not called and do not return values.

### Construction
- Python: `@dataclass` — `Column(key=..., label=..., justify="left", tone_by=None)`
- TypeScript: `interface Column` — plain object literal
- Rust: `struct Column` — struct literal

### Fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `key` | string | yes | Stable identifier. Matches a `Row.cells` entry **by index**, not by key — `row.cells[i]` corresponds to `columns[i]`. |
| `label` | string | yes | Header text for the rendered column. |
| `justify` | string enum | optional | `"left"` (default) \| `"right"` \| `"center"`. Omitted when `"left"`. |
| `tone_by` | string | optional | References a [`TonePalette`](#contract-tonepalette).`name`. Omitted when no tone applies to the column. |

### Properties
- immutable by convention (Python/TypeScript); owned value (Rust)
- ordering is significant: `columns` array order fixes render order and the position every `Row.cells` array must align to

---

## Contract: Row

### Inputs
N/A — this is a data type, not a callable function. See [`TuiViewModel`](#contract-tuiviewmodel).

### Errors
N/A — data types are not called and do not raise secondary exceptions.

### Returns
N/A — data types are not called and do not return values.

### Construction
- Python: `@dataclass` — `Row(cells=[...], tags=[...])`
- TypeScript: `interface Row` — plain object literal
- Rust: `struct Row` — struct literal

### Fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `cells` | array of [`Cell`](#contract-cell) | yes | Position-indexed: `cells[i]` corresponds to `columns[i]`. Length MUST equal `columns.length`. |
| `tags` | array of strings | optional | Consulted by a `tone_by` palette's `tag_equals` rule. Omitted when empty. |

### Properties
- immutable by convention (Python/TypeScript); owned value (Rust)
- invariant enforced by the builder, not by the type system: `len(cells) == len(columns)` for every row in a given `TuiViewModel`

---

## Contract: Cell

### Inputs
N/A — this is a data type, not a callable function. See [`TuiViewModel`](#contract-tuiviewmodel).

### Errors
N/A — data types are not called and do not raise secondary exceptions.

### Returns
N/A — data types are not called and do not return values.

### Construction
- Discriminated union on `kind`, in all three SDKs (Python via a tagged `@dataclass` hierarchy or literal-typed field, TypeScript via a discriminated `interface` union, Rust via a tagged `enum`) — never a single flat struct with all variants' fields present at once.

### Fields

`Cell` carries a `kind` discriminant plus kind-specific fields:

| `kind` | Additional fields | Notes |
|---|---|---|
| `"text"` | `value: string` | Plain text. The only kind an unrecognised `Column` key degrades to (see [`modules_to_view_model`](#contract-modules_to_view_model) Errors: an unknown column key renders as an empty `"text"` cell rather than raising). |
| `"tags"` | `values: array of strings` | Renderer joins with an idiomatic separator; not joined by the toolkit itself. |
| `"badge"` | `value: string`, `tone?: Tone` | Short label; renderer may box or highlight it. |
| `"symbol"` | `value: string` (one of `"check"`, `"cross"`, `"warning"`, `"circle"`), `tone?: Tone` | Renderer chooses the glyph (✓ / ✗ / ⚠ / ○) or an ASCII fallback — the wire format carries the symbolic name, never the glyph itself. |

`Tone`, referenced by `"badge"` and `"symbol"`, is one of exactly five values: `"neutral"` \| `"positive"` \| `"negative"` \| `"warning"` \| `"info"` (Decision 1 — locked for V1; see [Tone](#tone-semantic-not-visual)).

### Properties
- immutable by convention (Python/TypeScript); owned value (Rust)
- exhaustive union: exactly 4 `kind` values in V1 (Decision 2 — locked, no `"json"` or `"link"` kind); renderers MUST tolerate an unrecognised future `kind` by falling back to plain-text rendering rather than erroring, per the schema-versioning policy (Decision 5)
- no floats, ever: numeric-looking cell content is pre-formatted to a string by the builder before it reaches a `Cell`, avoiding cross-language float-rendering divergence
- no raw color: `tone` is a semantic value (`Tone`), never an ANSI code, RGB hex, or library-specific style string — that mapping is a Tier-2 renderer concern

---

## Contract: Sort

### Inputs
N/A — this is a data type, not a callable function. See [`TuiViewModel`](#contract-tuiviewmodel).

### Errors
N/A — data types are not called and do not raise secondary exceptions.

### Returns
N/A — data types are not called and do not return values.

### Construction
- Python: `@dataclass` — `Sort(key="module_id", direction="asc")`
- TypeScript: `interface Sort` — `{ key, direction }`
- Rust: `struct Sort` — struct literal, `direction: Direction` (`Direction::Asc` / `Direction::Desc`)

### Fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `key` | string | yes | Must match a `columns[].key` in V1's allowed set: `module_id`, `alias`, `description`. Other keys (`calls`, `errors`, `latency`) may appear here — annotating what the caller requested — but the toolkit does not execute that ordering itself; see [Sort/Filter Execution Model](#sortfilter-execution-model). |
| `direction` | string enum | yes | `"asc"` \| `"desc"`. |

### Properties
- immutable by convention (Python/TypeScript); owned value (Rust)
- annotation only for non-toolkit-executed keys: `sort` always records what was *requested*, regardless of whether the toolkit or the caller actually produced the row order

---

## Contract: Filter

### Inputs
N/A — this is a data type, not a callable function. See [`TuiViewModel`](#contract-tuiviewmodel).

### Errors
N/A — data types are not called and do not raise secondary exceptions.

### Returns
N/A — data types are not called and do not return values.

### Construction
- Python: `@dataclass` — `Filter(tags=(...), search="", annotations=(...), exposure="all", deprecated=True)`
- TypeScript: `interface Filter` — plain object literal
- Rust: `struct Filter` — struct literal, `exposure: Exposure` enum

### Fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `tags` | array of strings | yes (may be empty) | AND-filter — a module must carry every listed tag. |
| `search` | string | yes (may be empty) | Case-insensitive substring match over `module_id` + `description`. |
| `annotations` | array of strings | yes (may be empty) | Names of `ModuleAnnotations` flag fields that must be `true`. |
| `exposure` | string enum | yes | `"exposed"` \| `"hidden"` \| `"all"`. |
| `deprecated` | boolean | yes | When `false`, modules with `annotations.deprecated == true` are excluded. |

### Properties
- immutable by convention (Python/TypeScript); owned value (Rust)
- required-but-empty, deliberately: empty strings and empty arrays are emitted in the canonical encoding (never omitted, never `null`) so that "a filter was applied with an empty value" stays distinguishable from "no filter field annotation at all" (the enclosing `TuiViewModel.filter` itself is what gets omitted when no filter was requested)

---

## Contract: TonePalette

### Inputs
N/A — this is a data type, not a callable function. See [`TuiViewModel`](#contract-tuiviewmodel).

### Errors
N/A — data types are not called and do not raise secondary exceptions.

### Returns
N/A — data types are not called and do not return values.

### Construction
- Python: `@dataclass` — `TonePalette(name="tag_palette", rules=[...])`
- TypeScript: `interface TonePalette` — plain object literal
- Rust: `struct TonePalette` — struct literal

### Fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | Referenced by a [`Column`](#contract-column).`tone_by`. |
| `rules` | array of [`ToneRule`](#contract-tonerule) | yes | Evaluated in array order; **first match wins**. |

### Properties
- immutable by convention (Python/TypeScript); owned value (Rust)
- V1 wiring note: the builder wires only the **first** supplied palette's `name` to the `"tags"` column's `tone_by` — there is no per-column palette-selection parameter yet (see [`modules_to_view_model`](#contract-modules_to_view_model) Inputs)

---

## Contract: ToneRule

### Inputs
N/A — this is a data type, not a callable function. See [`TuiViewModel`](#contract-tuiviewmodel).

### Errors
N/A — data types are not called and do not raise secondary exceptions.

### Returns
N/A — data types are not called and do not return values.

### Construction
- Python: `@dataclass` — `ToneRule(match={"kind": "tag_equals", "value": "deprecated"}, tone="warning")`
- TypeScript: `interface ToneRule` — plain object literal
- Rust: `struct ToneRule` — struct literal

### Fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `match` | object | yes | V1 supports exactly one shape: `{ "kind": "tag_equals", "value": "<tag>" }`. Other `match.kind` values are reserved for future schema-version bumps. |
| `tone` | `Tone` enum | yes | One of the five values in [Tone](#tone-semantic-not-visual). |

### Properties
- immutable by convention (Python/TypeScript); owned value (Rust)
- narrow by design: `tag_equals` is the only rule kind demonstrated by existing CLI behaviour (see Risks — "`TonePalette` design too narrow"); additional `match.kind` variants are additive and non-breaking under Decision 5

---

## Contract: Group

### Inputs
N/A — this is a data type, not a callable function. See [`TuiViewModel`](#contract-tuiviewmodel).

### Errors
N/A — data types are not called and do not raise secondary exceptions.

### Returns
N/A — data types are not called and do not return values.

### Construction
- Python: `@dataclass` — `Group(label="users", row_indices=[0, 2, 5])`
- TypeScript: `interface Group` — plain object literal
- Rust: `struct Group` — struct literal, `row_indices: Vec<usize>`

### Fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `label` | string | yes | Group header text. |
| `row_indices` | array of integers | yes | Indices into the **top-level** `TuiViewModel.rows` array, not a nested copy of the rows themselves. |

### Properties
- immutable by convention (Python/TypeScript); owned value (Rust)
- present only when `kind == "grouped"`: `TuiViewModel.groups` is omitted entirely for `kind == "list"`
- renderer contract: a renderer iterates groups in array order, and within each group iterates `row_indices` in the order given; a row not referenced by **any** group's `row_indices` is not rendered at all — grouping is a projection, not a guaranteed partition of `rows`
- index validity: every value in `row_indices` MUST be a valid index into `rows`; the same row index MAY appear in more than one group (e.g. `group_by: "tag"` when a module carries multiple tags)

---

## Conformance Corpus

A shared corpus lives at `apcore-toolkit/conformance/fixtures/view_model.json`,
following the same structure as the shipped `format_csv.json`,
`format_jsonl.json`, and `display_resolve.json`: **one JSON document** with
top-level `$schema`, `title`, `description`, and `version`, plus a `test_cases`
array whose entries each carry `id`, `description`, `input`, and `expected`.

Here, `input` is a list of `ScannedModule` plus the builder options, and
`expected` is the byte-identical canonical encoding of the resulting
`TuiViewModel`.

```jsonc
{
  "$schema": "https://apcore.dev/schemas/conformance-fixture.json",
  "title": "TUI View Model — modules_to_view_model() / format_view_model()",
  "description": "Byte-equivalent TuiViewModel encoding across Python / TypeScript / Rust SDKs.",
  "version": "1.0.0",
  "test_cases": [
    {
      "id": "view_model_001_empty_list",
      "description": "Zero modules, no filter, no sort",
      "input": { "modules": [], "options": { "view": "list" } },
      "expected": "{\"schema_version\":1,\"kind\":\"list\",\"columns\":[],\"rows\":[]}"
    }
  ]
}
```

Minimum V1 fixture set:

| Test case `id` | Purpose |
|---|---|
| `view_model_001_empty_list` | Zero modules, no filter, no sort |
| `view_model_002_basic_list_three_columns` | 3 modules, default columns |
| `view_model_003_filter_tag_intersect` | Two tags, AND semantics |
| `view_model_004_filter_search_case_insensitive` | Substring match across `module_id` + `description` |
| `view_model_005_filter_exposure_hidden` | Hidden-only filter |
| `view_model_006_sort_asc_module_id` | Ascending sort by `module_id` |
| `view_model_007_sort_desc_description` | Descending sort by `description` |
| `view_model_008_grouped_by_tag` | `kind: "grouped"` with two groups |
| `view_model_009_grouped_by_prefix` | `kind: "grouped"` by `module_id` prefix |
| `view_model_010_tone_palette_deprecated_warning` | `tag_equals` rule emits `warning` tone |
| `view_model_011_display_overlay_alias` | Honours `ScannedModule.display.alias` over `module_id` |

Each SDK's test suite runs the corpus; CI fails on any byte divergence.

## Migration Plan

### Phase 1 — Toolkit (done)

In `apcore-toolkit-{python,typescript,rust}`:

1. [x] Add `TuiViewModel` + supporting types.
2. [x] Implement `modules_to_view_model` + `format_view_model`.
3. [x] Add conformance corpus + per-SDK conformance tests.
4. [x] Re-export from package root (and TypeScript `browser` subpath).
5. [x] Add `## Contract: modules_to_view_model` and `## Contract: format_view_model`
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

## Decisions

*Recorded 2026-09-08, after verifying what actually shipped in
`apcore-toolkit-{python,typescript,rust}` 0.11.1. Four of the five
recommendations below were adopted as written; the fifth was not, and is now
tracked as follow-up work rather than left implicit.*

| # | Question | Outcome | Evidence |
|---|---|---|---|
| 1 | Tone palette size | **Locked at 5** as recommended | `Tone` = `"neutral"` \| `"positive"` \| `"negative"` \| `"warning"` \| `"info"` |
| 2 | Cell kinds | **Locked at 4** as recommended — no `json`, no `link` in V1 | `CellKind` = `"text"` \| `"tags"` \| `"badge"` \| `"symbol"` |
| 3 | Status filter (enabled/disabled) | **SDK-side** as recommended; `ScannedModule` was not extended | no `status` handling in any SDK's view-model source |
| 4 | Display-overlay precedence | **NOT adopted.** See below. | the builder resolves `display.alias > module_id` — the narrow toolkit chain, not the CLI's |
| 5 | Schema-versioning policy | **Adopted**: `schema_version: 1`, additive-only; renames require a bump; renderers fall back to plain text on unknown `Cell.kind` / `Tone` | `schema_version: int = 1` emitted in every envelope |

### Decision 4 was not implemented — and the divergence it named still ships

This is the one item worth stating plainly rather than closing quietly. The
recommendation was to lift `apcore-cli`'s richer chain
(`display.cli.alias > display.alias > canonical_id`) into the toolkit. What
shipped instead reads the **sparse** overlay's top-level alias only:

```
display.alias  >  module_id
```

It consults neither the surface-scoped `display.cli.alias` nor the *resolved*
form that [`DisplayResolver`](display-overlay.md) writes into
`metadata["display"]` — even though the resolver already implements the full
chain (`surface override > display.alias > binding alias > suggested_alias >
module_id`).

The practical effect is bounded but real: a binding that sets only
`display.cli.alias` renders its alias in `apcore-cli` and its bare `module_id`
in the view model, so the two disagree on the same input. This is precisely the
"display-overlay precedence drift" that issue #14 listed among the user-visible
consequences already shipping — so the lift closed the *column and filter*
divergences it set out to close, and left this one open.

**Follow-up, not a V1 blocker.** Two candidate fixes, in preference order:

1. Have the builder consume `metadata["display"]` when present, falling back to
   the sparse `display` overlay when it is absent — reusing the resolver rather
   than reimplementing precedence a third time.
2. Failing that, extend `_resolve_alias` with the surface-scoped lookup and
   pin the chain with `display_resolve.json` cases.

Either is additive under Decision 5's policy — no `schema_version` bump — and
neither belongs in the same change as the Phase 2 CLI migration, which should
be able to assume a settled chain.

## Open Questions

*All five are resolved — see [Decisions](#decisions) for the verdicts and the
evidence from the shipped 0.11.1 sources. Retained verbatim because Decision 4
was **not** adopted, and the original reasoning is what makes that legible.*

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
| Over-engineering for current 3-consumer reality | Medium | ~~Defer Phase 1 until a 4th consumer surfaces.~~ **Superseded 2026-09-04:** the gate was passed and Phase 1 shipped in all three SDKs. The risk now reads forward, to Phase 2: if no consumer adopts the view model, the toolkit carries a surface nothing renders. Mitigated by Phase 2 being a net **−360 LOC** migration for the three existing CLIs — adoption removes more code than it adds. |
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
