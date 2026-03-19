# Changelog

All notable changes to this project will be documented in this file.

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
