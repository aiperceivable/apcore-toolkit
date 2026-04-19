# Binding Loader

The `BindingLoader` parses `.binding.yaml` files back into `ScannedModule` objects — the inverse of [`YAMLWriter`](output-writers.md#yamlwriter). Available in **Python**, **TypeScript**, and **Rust**.

## Overview

`YAMLWriter` emits binding files from `ScannedModule` objects (write path). `BindingLoader` reads them back (read path), enabling:

- **Validation** — re-scan + load, diff to catch drift between code and committed bindings.
- **Merging** — load a curated binding file, merge with a fresh scan, re-emit.
- **Round-trip** — edit `display`, `annotations`, `metadata` manually in YAML and preserve them across regenerations.
- **Inspection tooling** — build CLIs, dashboards, linters that consume binding files without running the underlying code.

Unlike `apcore.BindingLoader` (which `import_module`s the `target` and registers a runtime `FunctionModule`), toolkit's `BindingLoader` is **pure data**: it does not import code, does not call the target, and does not touch the apcore Registry. Result is a list of plain `ScannedModule` values.

## `BindingLoader`

**Python module**: `apcore_toolkit.binding_loader` (also re-exported from `apcore_toolkit`)
**TypeScript module**: `apcore-toolkit`
**Rust module**: `apcore_toolkit::binding_loader` (also re-exported from `apcore_toolkit`)

### Methods

| Method | Description |
|--------|-------------|
| `load(path, *, strict=False, recursive=False)` | Load a single `.binding.yaml` file or every `*.binding.yaml` in a directory. Python only: pass `recursive=True` to descend into subdirectories. |
| `load_data(data, *, strict=False)` | Load pre-parsed YAML data (a `{"bindings": [...]}` dict). |

Both return `list[ScannedModule]` (or `ScannedModule[]` / `Vec<ScannedModule>`).

**Directory loading is all-or-nothing.** When given a directory, the loader walks files in lexicographic order; the first malformed file raises `BindingLoadError` and discards every previously-parsed file. Callers that need best-effort aggregation (e.g., a linter) should iterate files themselves and invoke `load` per file with their own exception handling.

### Modes

| Mode | Required fields | Use case |
|------|-----------------|----------|
| **loose** (`strict=False`, default) | `module_id`, `target` | Verification, merging, tooling — permissive parsing with sensible defaults. |
| **strict** (`strict=True`) | `module_id`, `target`, `input_schema`, `output_schema` | CI pipelines, production builds — fail fast on incomplete bindings. |

Missing optional fields fall back to `ScannedModule` dataclass defaults (empty schemas, empty tags, `version="1.0.0"`, `display=None`, etc.).

## Field Mapping

| YAML key | `ScannedModule` field | Strict required | Loose default |
|----------|-----------------------|-----------------|---------------|
| `module_id` | `module_id` | ✓ (always required) | — |
| `target` | `target` | ✓ (always required) | — |
| `description` | `description` | — | `""` |
| `documentation` | `documentation` | — | `None` |
| `tags` | `tags` | — | `[]` |
| `version` | `version` | — | `"1.0.0"` |
| `annotations` | `annotations` | — | `None` (parsed via `ModuleAnnotations.from_dict` when present) |
| `examples` | `examples` | — | `[]` (malformed entries skipped with warning) |
| `metadata` | `metadata` | — | `{}` |
| `input_schema` | `input_schema` | ✓ strict | `{}` |
| `output_schema` | `output_schema` | ✓ strict | `{}` |
| `display` | `display` | — | `None` |
| `suggested_alias` | `suggested_alias` | — | `None` |
| `warnings` | `warnings` | — | `[]` |

## `spec_version` Handling

The top-level `spec_version` field is advisory:

| State | Behaviour |
|-------|-----------|
| Missing | Warn and assume `"1.0"`. |
| `"1.0"` | Silent. |
| Anything else | Warn and proceed best-effort (forward compatibility). |

## Errors

`BindingLoadError` is raised on:

- Path not found.
- Malformed YAML (parse error).
- Top-level document not a mapping; `bindings` key missing or not a list; entry not a mapping.
- Missing required fields per selected mode.

The error carries `file_path`/`module_id`/`missing_fields` plus a human-readable `reason` in Python; TypeScript exposes the same data as camelCase fields `filePath`/`moduleId`/`missingFields`/`reason` on `BindingLoadError extends Error`.

In Rust the error is an enum (`thiserror`-derived) with 5 variants — `PathNotFound`, `FileRead`, `YamlParse`, `MissingFields`, `InvalidStructure` — carrying per-variant payloads; callers pattern-match to recover structured information.

## Code Examples

=== "Python"

    ```python
    from apcore_toolkit import BindingLoader, BindingLoadError

    loader = BindingLoader()

    # Load a directory
    modules = loader.load("./bindings")

    # Load a single file in strict mode
    try:
        modules = loader.load("users.binding.yaml", strict=True)
    except BindingLoadError as exc:
        print(exc.missing_fields)

    # Load pre-parsed data
    modules = loader.load_data({
        "spec_version": "1.0",
        "bindings": [{"module_id": "x.y", "target": "pkg:f"}],
    })
    ```

=== "TypeScript"

    ```typescript
    import { BindingLoader, BindingLoadError } from "apcore-toolkit";

    const loader = new BindingLoader();

    // Load a directory
    const modules = loader.load("./bindings");

    // Strict mode
    try {
      const strictModules = loader.load("users.binding.yaml", { strict: true });
    } catch (exc) {
      if (exc instanceof BindingLoadError) console.log(exc.missingFields);
    }

    // Pre-parsed data
    loader.loadData({
      spec_version: "1.0",
      bindings: [{ module_id: "x.y", target: "pkg:f" }],
    });
    ```

=== "Rust"

    ```rust
    use apcore_toolkit::{BindingLoader, BindingLoadError};
    use std::path::Path;

    let loader = BindingLoader::new();
    let modules = loader.load(Path::new("./bindings"), false)?;

    match loader.load(Path::new("users.binding.yaml"), true) {
        Ok(mods) => { /* ... */ }
        Err(BindingLoadError::MissingFields { missing_fields, .. }) => {
            println!("{missing_fields:?}");
        }
        Err(e) => return Err(e.into()),
    }
    ```

## Round-Trip Guarantee

`YAMLWriter.write` followed by `BindingLoader.load` preserves every persisted field: `module_id`, `target`, `description`, `documentation`, `tags`, `version`, `annotations` (including the 12 `ModuleAnnotations` fields such as `streaming`, `cache_ttl`), `examples`, `metadata`, `input_schema`, `output_schema`, and `display`. This is covered by round-trip tests in all three SDKs.

Fields that `YAMLWriter` does not emit (e.g., `warnings`) are not preserved — those default to empty on load.

## Comparison with `apcore.BindingLoader`

| Aspect | `apcore-toolkit` `BindingLoader` | `apcore` `BindingLoader` |
|--------|-----------------------------------|---------------------------|
| Returns | `list[ScannedModule]` (data) | `list[FunctionModule]` (runtime) |
| Imports `target` | No | Yes (via `importlib.import_module`) |
| Registers modules | No | Yes (mutates a `Registry`) |
| Required fields | `module_id`, `target` (loose) | `module_id`, `target` (always) |
| Use case | Tooling, CI, merging, diffing | Runtime module loading |

Use the toolkit loader when you need to **inspect or transform** binding files. Use `apcore.BindingLoader` when you need to **execute** the modules they describe.
