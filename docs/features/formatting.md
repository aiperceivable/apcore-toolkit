# Formatting Utilities

`apcore-toolkit` includes powerful tools for converting complex data structures into formatted, human-readable Markdown. This is especially useful for creating "AI-perceivable" documentation or for logging results in a readable format.

## `to_markdown()`

The `to_markdown()` function converts an arbitrary dictionary or list into a structured Markdown string.

### Features
- **Depth Control**: Specify how many levels deep the conversion should go.
- **Table Heuristics**: Automatically detects when data can be better represented as a Markdown table.
- **Recursive Processing**: Handles nested dictionaries and lists gracefully.

### Example

=== "Python"

    ```python
    from apcore_toolkit import to_markdown

    user_data = {
        "name": "Alice",
        "role": "admin",
        "preferences": {
            "theme": "dark",
            "notifications": True
        },
        "recent_activity": [
            {"action": "login", "timestamp": "2024-03-07T12:00:00Z"},
            {"action": "upload", "timestamp": "2024-03-07T12:05:00Z"}
        ]
    }

    # Convert to Markdown with a title
    md = to_markdown(user_data, title="User Profile")
    print(md)
    ```

=== "TypeScript"

    ```typescript
    import { toMarkdown } from "apcore-toolkit";

    const userData = {
      name: "Alice",
      role: "admin",
      preferences: {
        theme: "dark",
        notifications: true,
      },
      recentActivity: [
        { action: "login", timestamp: "2024-03-07T12:00:00Z" },
        { action: "upload", timestamp: "2024-03-07T12:05:00Z" },
      ],
    };

    // Convert to Markdown with a title
    const md = toMarkdown(userData, { title: "User Profile" });
    console.log(md);
    ```

## Schema Enrichment

The `enrich_schema_descriptions()` utility helps bridge the gap when a JSON Schema lacks parameter descriptions but they are available in a function's docstring.

### Features
- **Description Merging**: Merges descriptions from a dictionary into the `properties` of a JSON Schema.
- **Safe by Default**: Won't overwrite existing descriptions unless explicitly requested.
- **Scanned Integration**: Used by concrete scanners to supplement schemas extracted from source code or OpenAPI with docstring-level documentation.

=== "Python"

    ```python
    from apcore_toolkit import enrich_schema_descriptions

    raw_schema = {
        "type": "object",
        "properties": {
            "user_id": {"type": "integer"}
        }
    }

    param_descriptions = {
        "user_id": "The ID of the user to retrieve."
    }

    # Enrich the schema with parameter descriptions
    enriched = enrich_schema_descriptions(raw_schema, param_descriptions)
    ```

=== "TypeScript"

    ```typescript
    import { enrichSchemaDescriptions } from "apcore-toolkit";

    const rawSchema = {
      type: "object",
      properties: {
        user_id: { type: "integer" },
      },
    };

    const paramDescriptions = {
      user_id: "The ID of the user to retrieve.",
    };

    // Enrich the schema with parameter descriptions
    const enriched = enrichSchemaDescriptions(rawSchema, paramDescriptions);
    ```

## Surface-Aware Formatters

`to_markdown()` is a generic data-to-Markdown converter — it knows nothing about `ScannedModule`. Three higher-level helpers turn modules and schemas into output shaped for a specific consumer surface (LLM, CLI, JSON, agent skill files), so each surface does not have to roll its own renderer.

| Function | Purpose |
|---|---|
| `format_schema(schema, *, style)` | Render a JSON Schema as natural-language prose, a Markdown table, or pass-through JSON. |
| `format_module(module, *, style, display=True)` | Render a single `ScannedModule` for the chosen surface. |
| `format_modules(modules, *, style, group_by)` | Aggregate version that calls `format_module` per item with optional grouping. |

### Style Matrix

| Style | Receiver | Output shape |
|---|---|---|
| `markdown` | LLM context, docs site | Sections: title, description, parameters (prose via `format_schema(style="prose")`), returns, behavior (annotations rendered as natural-language facts), examples |
| `skill` | `.claude/skills/<id>/SKILL.md`, `.gemini/skills/<id>/SKILL.md` | Same body as `markdown`, prefixed with minimal YAML frontmatter (`name`, `description` only — vendor extensions like `allowed-tools` or `paths` are NOT emitted; toolkit takes no vendor side) |
| `table-row` | CLI listing | Single line, pipe-separated: `id │ alias │ description │ tags` |
| `json` | Programmatic API | `module_to_dict(module)` pass-through |

The `display` flag (default `true`) honours the `ScannedModule.display` overlay (alias, description, guidance, tags) when rendering. Passing `display=False` shows the raw `module_id` / `description` instead, useful for debugging scanner output.

### Annotations Rendering

When `style="markdown"` or `style="skill"`, the formatter MUST emit `ModuleAnnotations` as a plain Markdown table of fact rows (not as imperative English sentences) — translating "readonly = true" into "this module does not modify state" is a localisation decision the toolkit does not own. LLMs and downstream renderers can interpret a fact table in any language.

The fact table MUST contain **only fields that differ from `ModuleAnnotations` defaults**. Implementations construct a default `ModuleAnnotations` instance, serialise both it and the module's annotations via `annotations_to_dict` (snake_case wire form), and emit only entries where the module's value differs from the default. The `extra` free-form bag is always skipped.

Rows MUST be sorted alphabetically by key (snake_case). Boolean values MUST be rendered as lowercase `true` / `false`. Numbers, strings, arrays, and objects render via the language's natural JSON conversion (e.g. `300`, `cursor`, `["id"]`, `{"region": "us"}`).

If every annotation field equals its default (or `annotations` is `null`), the entire `## Behavior` section is omitted.

This three-rule alignment (default-comparison + alphabetical + lowercase bool) is what makes `format_module(module, style="markdown")` produce byte-identical output across the three SDKs for the same `ScannedModule` fixture.

### Example

=== "Python"

    ```python
    from apcore_toolkit import format_module, format_schema, format_modules

    md_for_llm = format_module(scanned_module, style="markdown")
    skill_text = format_module(scanned_module, style="skill")
    cli_row    = format_module(scanned_module, style="table-row")
    api_dict   = format_module(scanned_module, style="json")

    listing = format_modules(modules, style="markdown", group_by="tag")
    prose   = format_schema(scanned_module.input_schema, style="prose")
    ```

=== "TypeScript"

    ```typescript
    import { formatModule, formatModules, formatSchema } from "apcore-toolkit";

    const mdForLlm = formatModule(scannedModule, { style: "markdown" });
    const skillTxt = formatModule(scannedModule, { style: "skill" });
    const cliRow   = formatModule(scannedModule, { style: "table-row" });
    const apiObj   = formatModule(scannedModule, { style: "json" });

    const listing = formatModules(modules, { style: "markdown", groupBy: "tag" });
    const prose   = formatSchema(scannedModule.inputSchema, { style: "prose" });
    ```

=== "Rust"

    ```rust
    use apcore_toolkit::{format_module, format_modules, format_schema, ModuleStyle, SchemaStyle};

    let md_for_llm = format_module(&scanned_module, ModuleStyle::Markdown, true);
    let skill_text = format_module(&scanned_module, ModuleStyle::Skill, true);
    let cli_row    = format_module(&scanned_module, ModuleStyle::TableRow, true);
    let api_value  = format_module(&scanned_module, ModuleStyle::Json, true);
    ```

---

## Contract: to_markdown

### Inputs
- `data`: dict or list, required — arbitrary nested data structure to convert
- `title`: string, optional — if provided, prepended as an H1 or H2 header
- `depth`: int, optional, default=1 — heading level to start at (1 = `#`, 2 = `##`, etc.)

### Errors
- None raised — non-serializable values are converted via `str()` / `String()` as a fallback

### Returns
- On success: string — Markdown-formatted representation of the input data

### Properties
- async: false
- pure: true
- thread_safe: true

---

## Contract: enrich_schema_descriptions

### Inputs
- `schema`: dict / `Record<string, unknown>`, required — a JSON Schema object with a `"properties"` key; mutated in place
- `descriptions`: dict[str, str] / `Record<string, string>`, required — mapping of property name → description to inject

### Errors
- None raised — silently skips properties not found in `schema.properties`

### Returns
- Python/Rust: `None` — schema is mutated in place
- TypeScript: `void` — schema is mutated in place

### Properties
- async: false
- pure: false (mutates the schema dict in place)
- overwrite_safe: true — existing `"description"` fields are NOT overwritten (only missing descriptions are filled in)
- thread_safe: true (assuming no concurrent mutation of the same schema dict)

---

## Contract: format_schema

### Inputs
- `schema`: dict / `Record<string, unknown>` / `serde_json::Value`, required — JSON Schema (typically `ScannedModule.input_schema` or `output_schema`)
- `style`: enum, required — `"prose"` (Python/TS string literal) / `SchemaStyle::Prose` (Rust)
  - `"prose"`: Markdown bullet list, one line per top-level property, format `- \`name\` (type, required|optional) — description.`
  - `"table"`: Markdown pipe table with columns `Name | Type | Required | Default | Description`
  - `"json"`: pass-through (returns the input unchanged for symmetry with the other styles)
- `max_depth`: int, optional, default=3 — recursion limit for nested object schemas; deeper levels collapse into a fenced ` ```json ... ``` ` block to avoid unbounded prose

### Errors
- None raised — schemas missing `properties` render as an empty list / table; non-object schemas (e.g. arrays at top level) render as a one-line "schema accepts <type>" summary

### Returns
- `style="prose"` / `style="table"`: string
- `style="json"`: same shape as input (dict / `Record` / `Value`)

### Properties
- async: false
- pure: true
- thread_safe: true

---

## Contract: format_module

### Inputs
- `module`: `ScannedModule`, required
- `style`: enum, required — one of `"markdown"`, `"skill"`, `"table-row"`, `"json"`
- `display`: boolean, optional, default=`true` — when true, prefer `module.display.alias` / `display.description` / `display.guidance` / `display.tags` over the raw fields. When false, render only the raw `module_id` / `description` etc.

### Style behaviour

| Style | Output type | Format |
|---|---|---|
| `markdown` | string | `# {alias or module_id}\n\n{description}\n\n## Parameters\n{format_schema(input_schema, style="prose")}\n\n## Returns\n{format_schema(output_schema, style="prose")}\n\n## Behavior\n{annotations as fact table}\n\n## Examples\n{examples}` |
| `skill` | string | YAML frontmatter `---\nname: {alias or module_id}\ndescription: {description}\n---\n` + the `markdown` body. ONLY `name` + `description` are emitted in frontmatter; vendor-specific keys (`allowed-tools`, `paths`, `when_to_use`, etc.) are NOT emitted. |
| `table-row` | string | `\`{module_id}\` │ \`{alias}\` │ {description} │ {tags joined with ", "}` |
| `json` | dict / `Record` / `Value` | Equivalent to `module_to_dict(module)` — see [scanning.md](scanning.md#contract-serializationutilities) |

### Errors
- `ValueError` (Python) / `Error` (TypeScript) / `Err(FormatError)` (Rust) — when `style` is not one of the four canonical values

### Returns
- string for `markdown`, `skill`, `table-row`
- dict / `Record` / `serde_json::Value` for `json`

### Properties
- async: false
- pure: true
- thread_safe: true

---

## Contract: format_modules

### Inputs
- `modules`: list / array / Vec of `ScannedModule`, required
- `style`: enum, required — same set as `format_module`
- `group_by`: enum, optional, default=`None` — `"tag"`, `"prefix"` (groups by everything before the first `.` in `module_id`), or `None`/`null`/`Option::None` for ungrouped output
- `display`: boolean, optional, default=`true` — forwarded to each `format_module` call

### Returns
- For string styles (`markdown`, `skill`, `table-row`): single concatenated string. With `group_by`, each group is preceded by an `## {group}` header (`markdown`/`skill`) or a `── {group} ──` divider (`table-row`).
- For `style="json"`: list / array / Vec of dict / `Record` / `Value` (group_by is ignored).

### Errors
- Same as `format_module` for invalid `style`; invalid `group_by` raises `ValueError` / `Error` / `Err(FormatError)`

### Properties
- async: false
- pure: true
- thread_safe: true

### Skill output and vendor neutrality

`style="skill"` emits the portable minimum agreed by Anthropic Agent Skills and Gemini CLI. Vendor-specific frontmatter keys diverge (`allowed-tools` is Claude-only; `paths` is Gemini-only) and the toolkit deliberately does not pick a side. Consumers that want vendor extensions should post-process the output before writing to disk.

When emitted into per-module files (e.g. `.claude/skills/<module_id>/SKILL.md`), the recommended file naming is the sanitised `module_id` (replace any character outside `[A-Za-z0-9_-]` with `_`). The formatter itself only returns the file body; directory layout is the caller's responsibility.

---

## Use Case: AI Documentation

By converting complex internal states to Markdown tables or sections, you provide an LLM with a highly structured and easy-to-parse context. This improves the agent's ability to reason about the system's current state and available actions.

`format_module(module, style="markdown")` is the single primitive surfaces should reach for when injecting an apcore module description into an LLM prompt. `format_module(module, style="skill")` produces the same body wrapped in the minimal SKILL.md frontmatter accepted by Claude / Gemini agent runtimes — useful when exporting a registry as discoverable agent skills.

---

## Tabular Formats: `format_csv` / `format_jsonl`

Byte-equivalent tabular data emitters for the apcore ecosystem. The toolkit owns these formats — apcore-cli SDKs and other downstream consumers (`apcore-mcp`, `apcore-a2a`, downstream CLIs like `aisee-cli`) MUST delegate to these reference implementations rather than reimplementing.

### Why these live in the toolkit

CSV and JSONL are deterministic text formats whose output is observable byte-for-byte. The 3-SDK governance model (Python / TypeScript / Rust) makes per-SDK reimplementations a known divergence source: each SDK's natural choice of `str()` / `JSON.stringify` / `serde_json::to_string` differs in subtle ways (Python repr vs JSON, JS dropping trailing `.0`, varying `serde_json::Map` key ordering, RFC 4180 escaping correctness). Lifting the algorithm into the toolkit replaces those independent failure modes with a single conformance-tested implementation.

See `apcore-cli/docs/tech-design.md` ADR-07 for the tier definition: csv/jsonl/markdown/skill are **byte-equivalent toolkit-delegated**; table/tui are **SDK-native presentation**.

### Contract: `format_csv`

#### Inputs

- `rows`: iterable of `Mapping[str, Any]` / `Iterable<Record<string, unknown>>` / `&[Map<String, Value>]`
- `bom`: optional boolean (default `false`) — prepend UTF-8 BOM (`U+FEFF`) for Excel-locale users

#### Behavior

- **Header derivation**: union of keys across **all** rows, preserved in insertion-order from first occurrence. Rows missing a key emit an empty cell. (This explicitly fixes the prior single-row-keys data-loss bug present in every per-SDK reimplementation.)
- **Cell value serialization**:
    - `null` / `None` / `undefined` → empty cell
    - `bool` → `"true"` / `"false"`
    - `string` → raw
    - `int` → `str(value)`
    - `float` → canonical form: whole-number floats render as integers (e.g. `1.0 → "1"`) matching JS/Rust default `String(n)` behavior; fractional floats use shortest round-trip representation
    - `NaN` / `±Infinity` → empty cell
    - `object` / `array` → canonical compact JSON (see `format_jsonl` cell encoding below) embedded in the cell
- **RFC 4180 escaping**: cells containing `,`, `"`, `\n`, or `\r` are wrapped in `"..."`; embedded `"` is doubled to `""`.
- **Line terminator**: `\r\n` (CRLF) per RFC 4180.
- **Empty input**: returns the empty string `""`.

#### Returns

- string

#### Errors

- None — pure function over JSON-shaped input.

#### Properties

- async: false
- pure: true
- thread_safe: true

### Contract: `format_jsonl`

#### Inputs

- `rows`: iterable of `Mapping[str, Any]` / `Iterable<Record<string, unknown>>` / `&[Map<String, Value>]`

#### Behavior

- Each row is serialized as canonical compact JSON: no whitespace between tokens, `ensure_ascii=False` (unicode preserved), insertion-order preserved.
- Numbers: whole-number floats render without the trailing `.0` (matching JS `JSON.stringify`); `NaN` / `±Infinity` are emitted as `null` (matching JS `JSON.stringify`).
- Object key ordering: insertion order (Python dict order; TS object key order per ES2015+; Rust `serde_json::Map` with `preserve_order` feature).
- Lines are terminated by `\n` (LF — JSONL convention, NOT CRLF).
- No trailing blank line.
- Empty input returns the empty string `""`.

#### Returns

- string

#### Errors

- None.

#### Properties

- async: false
- pure: true
- thread_safe: true

### Conformance corpus

A shared corpus at `apcore-toolkit/conformance/fixtures/format_csv.json` and `format_jsonl.json` documents the expected output for ~25 test cases covering: heterogeneous keys, nested objects, RFC 4180 escaping (comma / quote / newline), scalar type coercion (bool / int / float / null), unicode, BOM, JS Number safe-range integers. Each SDK runs the corpus as part of its test suite and asserts byte-identical output. Adding a new edge case to either fixture automatically expands coverage across all 3 SDKs without per-language test edits.

### Number portability note

Integers exceeding `Number.MAX_SAFE_INTEGER` (`2^53 - 1 = 9007199254740991`) are not portable across SDKs: JavaScript `Number` loses precision above this threshold while Python and Rust have arbitrary-precision (Python `int`) or 64-bit signed (Rust `i64`) integer support. Callers passing large numeric identifiers should serialize them as strings to ensure cross-SDK fidelity.

### YAML is intentionally not in this tier

`format_yaml` is **not** part of this toolkit-delegated tier as of v0.7. Byte-equivalent YAML across SDKs requires a custom emitter (each idiomatic YAML library — PyYAML, js-yaml, serde_yaml_ng — uses different quoting / indent / null heuristics and is non-trivial to align). Until that custom emitter is designed, YAML output remains SDK-native and may differ across languages — accepted as a `table`-tier divergence under ADR-07. Tracked as a follow-up.

