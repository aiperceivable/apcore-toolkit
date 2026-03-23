# Changelog

All notable changes to this project will be documented in this file.

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
