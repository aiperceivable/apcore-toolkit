# Changelog

All notable changes to this project will be documented in this file.

## [0.5.0] - 2026-04-19

Aligned release across Python, TypeScript, and Rust. Tracks apcore 0.19.0 features (expanded `ModuleAnnotations`, `display` field, declarative config spec).

### Added

- **`BindingLoader`** (three SDKs) — parses `.binding.yaml` files back into `ScannedModule` objects. Pure-data reader: no target import, no Registry side effects. Distinct from `apcore.BindingLoader` which does both. Enables verification, merging, diffing, and round-trip workflows.
  - Loose mode (default): only `module_id + target` required.
  - Strict mode: additionally requires `input_schema + output_schema`.
  - `spec_version` validated; missing/unsupported values WARN but do not fail.
  - Errors: `BindingLoadError` with `file_path`, `module_id`, `missing_fields`, `reason` (Rust: 5-variant `thiserror` enum).
- **`ScannedModule.display`** (three SDKs) — new optional top-level field holding the sparse display overlay for binding YAML persistence. Distinct from `metadata["display"]` (resolved form produced by `DisplayResolver`).
- **New feature doc**: `docs/features/binding-loader.md`.
- **`docs/features/display-overlay.md`** — new section explaining the sparse-overlay vs resolved-display distinction, and the round-trip flow.
- **`docs/features/output-writers.md`** — notes on conditional `display:` emission, `spec_version` stamping, and a Rust example.
- **`docs/features/overview.md`** — Binding Loader entry; tri-language parity note.
- **`mkdocs.yml`** nav — adds Binding Loader, Display Overlay, Convention Scanning entries.

### Changed

- **`YAMLWriter`** — emits top-level `display:` key only when `ScannedModule.display` is non-empty. Rust writer refactored from `json!` macro to `serde_json::Map` to support conditional keys.
- **`serializers.module_to_dict / moduleToDict / module_to_value`** — include `display` field.
- **Python `AIEnhancer._build_prompt`** — confidence template now built dynamically from `gaps`. When `annotations` is in gaps, requests per-field confidence for every validator key (`annotations.readonly`, `annotations.streaming`, `annotations.cache_ttl`, …). Previously hard-coded to `{"description": 0.0, "documentation": 0.0}`, which caused annotation-field confidence lookups to fall back to `0.0` and fail the threshold check — annotation enhancement silently never took effect.

### Dependencies

- **`apcore >= 0.19.0` / `apcore-js >= 0.19.0`** — picks up the expanded `ModuleAnnotations` (12 fields: adds `streaming`, `cacheable`, `cache_ttl`, `cache_key_fields`, `paginated`, `pagination_style`, `extra`), `FunctionModule.display`, and new binding/schema error types. No toolkit type changes were needed for annotations — reflection (Python), serde (Rust), and `annotationsFromJSON` (TypeScript) propagate new fields automatically.

### Fixed (Rust)

- **`output::registry_writer` / `http_proxy_writer`** — fixed pre-existing `ModuleDescriptor` initialization errors following the apcore 0.19.0 upgrade: adds required `display` field; updates `annotations` to `Option<ModuleAnnotations>`; fills all descriptor fields in `http_proxy_writer` (previously partial).

### Tests

| SDK | Δ tests | Total |
|-----|---------|-------|
| Python | +34 | 440 |
| TypeScript | +29 | 320 |
| Rust | +26 | 304 |

All round-trip tests (`YAMLWriter.write` → `BindingLoader.load`) verify end-to-end preservation of `display`, `annotations` (12 fields), `metadata`, `examples`, and schemas.

### Hardening (post-review)

A code-forge:review pass on the 0.5.0 delta produced a short list of cross-SDK inconsistencies; they are fixed as part of this same release:

- **Silent drop of malformed `display` values** — Python, TypeScript, and Rust all used to swallow non-mapping overlays without warning. All three SDKs now emit a WARN with the offending module_id and return `None`/`null`, matching the behaviour of `annotations`/`examples` parsers.
- **`ScannedModule.display` defensive copy** — Python's `YAMLWriter` used to emit a bare reference (`binding["display"] = module.display`); it now deep-copies, matching TypeScript's `structuredClone` and Rust's `.clone()`.
- **Python `ScannedModule.display` field position** — moved from the middle of the dataclass to the end, preserving positional construction compatibility with pre-0.5.0 callers.
- **Python `BindingLoader` polish** — `load()` gained `recursive: bool = False`; `read_text` forces UTF-8; error wording changed from "missing required fields" to "missing or null required fields".
- **TypeScript `BindingLoader` polish** — `_parseExamples` uses `structuredClone`; `fs.statSync` failures distinguish `ENOENT` from other `errno` codes.
- **Rust `BindingLoader` polish** — `read_dir` per-entry errors are now surfaced as `BindingLoadError::FileRead` (previously silently discarded); `MissingFields`/`InvalidStructure` `Display` no longer leak `Some(...)` / `None` debug wrappers.

## [0.4.0] - 2026-03-23

### Added

- **`DisplayResolver`** (`apcore_toolkit.display`) — sparse binding.yaml display overlay (§5.13). Merges per-surface presentation fields (alias, description, guidance, tags, documentation) into `ScannedModule.metadata["display"]` for downstream CLI/MCP/A2A surfaces.
  - Resolution chain per field: surface-specific override > `display` default > binding-level field > scanner value.
  - `resolve(modules, *, binding_path=..., binding_data=...)` — accepts pre-parsed dict or a path to a `.binding.yaml` file / directory of `*.binding.yaml` files. `binding_data` takes precedence over `binding_path`.
  - MCP alias auto-sanitization: replaces characters outside `[a-zA-Z0-9_-]` with `_`; prepends `_` if result starts with a digit.
  - MCP alias hard limit: raises `ValueError` if sanitized alias exceeds 64 characters.
  - CLI alias validation: warns and falls back to `display.alias` when user-explicitly-set alias does not match `^[a-z][a-z0-9_-]*$` (module_id fallback always accepted without warning).
  - `suggested_alias` in `ScannedModule.metadata` (emitted by `simplify_ids=True` scanner) used as fallback when no `display.alias` is set.
  - Match-count logging: `INFO` for match count, `WARNING` when binding map loaded but zero modules matched.
- **New feature spec**: `docs/features/display-overlay.md`

### Tests

- 30 new tests in `tests/test_display_resolver.py` covering: no-binding fallthrough, alias-only overlay, surface-specific overrides, MCP sanitization, MCP 64-char limit, `suggested_alias` fallback, sparse overlay (10 modules / 1 binding), tags resolution, `binding_path` file and directory loading, guidance chain, CLI invalid alias warning and fallback, `binding_data` vs `binding_path` precedence.

### Added (Convention Module Discovery — §5.14)

- **`ConventionScanner`** — scans a `commands/` directory of plain Python files for public functions and converts them to `ScannedModule` instances with schema inferred from PEP 484 type annotations.
  - Module ID: `{file_prefix}.{function_name}` with `MODULE_PREFIX` override.
  - Description from first line of docstring (`"(no description)"` fallback).
  - `input_schema` / `output_schema` inferred from type hints.
  - `CLI_GROUP` and `TAGS` module-level constants stored in metadata.
  - `include` / `exclude` regex filters on module IDs.
- **New feature spec**: `docs/features/convention-scanning.md`

### Tests (Convention Module Discovery)

- 15 new tests in `tests/test_convention_scanner.py`.

---

## [0.3.1] - 2026-03-22

### Changed
- Rebrand: aipartnerup → aiperceivable

## [0.3.0] - 2026-03-19

### Added

- `deep_resolve_refs` / `deepResolveRefs` documented in OpenAPI Integration page
  with three-level resolution explanation
- `HTTPProxyRegistryWriter` section in Output Writers page with features,
  installation instructions, and usage example
- `"http-proxy"` format in `get_writer()` factory table with Language column
- `HTTPProxyRegistryWriter` and `enrich_schema_descriptions` added to README
  Core Modules table
- `modules_to_dicts()` / `modulesToDicts()` added to Serialization Utilities table
- "Choosing a Writer" table updated with `HTTPProxyRegistryWriter` row

### Fixed

- `resolve_schema` description corrected from "recursively resolves" to
  "single level only" — the recursive resolver is `deep_resolve_refs`
- Reference Resolution section rewritten with accurate three-level explanation
- Duplicate `## Reference Resolution` heading removed

---

## [0.2.0] - 2026-03-12

### Added

- `TypeScriptWriter` to Core Modules tables in README and docs index
- `"typescript"` format to `get_writer()` factory example (TypeScript only)
- `documentation` and `warnings` fields to `ScannedModule` Fields table
- Serialization Utilities section (`annotationsToDict`, `moduleToDict`)
  in Smart Scanning docs
- Changelog page for documentation site

### Fixed

- Updated `WriteResult` table in Output Writers docs with TypeScript naming
  and type columns
- Fixed `ai_guidance` → `metadata.ai_guidance` in AI Enhancement docs
- TypeScript examples: replaced `new ScannedModule({...})` with
  `createScannedModule({...})` (`ScannedModule` is an interface, not a class)
- TypeScript `YAMLWriter` and `TypeScriptWriter` `write()` call signatures
  corrected from `write(modules, { outputDir })` to `write(modules, outputDir, options?)`
- `filterModules` include pattern corrected from `RegExp` literal to `string`
  matching the actual TypeScript signature
- `resolveTarget` example now shows `await` (the function is async)
- `flattenParams` example now shows required `zodSchema` second argument
- OpenAPI import corrected from `"apcore-toolkit/openapi"` to `"apcore-toolkit"`
  (no sub-path export in TypeScript)
- TypeScript Registry import corrected from `"@anthropic/apcore"` to `"apcore-js"`
- Broken changelog links fixed (`docs/changelog.md` → root `CHANGELOG.md`)
- `docs/features/scanning.md` — TypeScript example `include` option type
  corrected to `string`, import updated
- `docs/index.md` — Python-only framing updated; toolkit is dual-language

---

## [0.1.0] - 2026-03-06

Initial release. Extracts shared framework-agnostic logic from `django-apcore`
and `flask-apcore` into a standalone toolkit package.

### Added

- `ScannedModule` dataclass — canonical representation of a scanned endpoint
- `BaseScanner` ABC with `filter_modules()`, `deduplicate_ids()`,
  `infer_annotations_from_method()`, and `extract_docstring()` utilities
- `YAMLWriter` — generates `.binding.yaml` files for `apcore.BindingLoader`
- `PythonWriter` — generates `@module`-decorated Python wrapper files
- `RegistryWriter` — registers modules directly into `apcore.Registry`
- `to_markdown()` — generic dict-to-Markdown conversion with depth control
  and table heuristics
- `flatten_pydantic_params()` — flattens Pydantic model parameters into
  scalar kwargs for MCP tool invocation
- `resolve_target()` — resolves `module.path:qualname` target strings
- `enrich_schema_descriptions()` — merges docstring parameter descriptions
  into JSON Schema properties
- `annotations_to_dict()` / `module_to_dict()` — serialization utilities
- OpenAPI utilities: `resolve_ref()`, `resolve_schema()`,
  `extract_input_schema()`, `extract_output_schema()`
- Output format factory via `get_writer()`
- 150 tests with 94% code coverage

### Dependencies

- apcore >= 0.9.0
- pydantic >= 2.0
- PyYAML >= 6.0
