---
description: "Pure-data BindingLoader parses .binding.yaml back into ScannedModule objects (inverse of YAMLWriter), with strict/loose modes for validation and round-trip."
---

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
| `load(path, *, strict=False, recursive=False)` | Load a single `.binding.yaml` file or every `*.binding.yaml` in a directory. All three SDKs: pass `recursive=True` / `{ recursive: true }` to descend into subdirectories for `**/*.binding.yaml`. |
| `load_data(data, *, strict=False)` | Load pre-parsed YAML data (a `{"bindings": [...]}` dict). |

Both return `list[ScannedModule]` (or `ScannedModule[]` / `Vec<ScannedModule>`).

**Directory loading is all-or-nothing.** When given a directory, the loader walks files in lexicographic order; the first malformed file raises `BindingLoadError` and discards every previously-parsed file. Callers that need best-effort aggregation (e.g., a linter) should iterate files themselves and invoke `load` per file with their own exception handling.

### Modes

| Mode | Required fields | Use case |
|------|-----------------|----------|
| **loose** (`strict=False`, default) | `module_id`, `target` | Verification, merging, tooling — permissive parsing with sensible defaults. |
| **strict** (`strict=True`) | `module_id`, `target`, `input_schema`, `output_schema` | CI pipelines, production builds — fail fast on incomplete bindings. |

Missing optional fields fall back to `ScannedModule` dataclass defaults (empty schemas, empty tags, `version="1.0.0"`, `display=None`, etc.).

#### Loose-mode wrong-type policy

When a non-required field (`input_schema`, `output_schema`, `tags`) is present but holds the wrong type — e.g., `input_schema: 42` or `tags: "single-string"` — the policy is mode-dependent:

| Mode | Behaviour |
|------|-----------|
| **strict** (`strict=True`) | Raise `BindingLoadError` (Python) / `Err(BindingLoadError)` (Rust) / `throw BindingLoadError` (TypeScript). |
| **loose** (`strict=False`, default) | Log a warning naming the offending field and entry, then coerce the field to its empty default (`{}` for schemas, `[]` for tags). Cross-SDK guarantee — Python, Rust, and TypeScript all warn-and-coerce in loose mode. |

The loose-mode behaviour is intentional: callers running with `strict=False` have explicitly opted into permissive parsing, and a single wrong-type optional field should not abort scanning of an otherwise valid binding file.

Required fields (`module_id`, `target`) are always validated and reject wrong-type or empty-string values regardless of mode.

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

In Rust the error is an enum (`thiserror`-derived) with 7 variants — `PathNotFound`, `FileRead`, `YamlParse`, `MissingFields`, `InvalidStructure`, `FileTooLarge`, `TooManyFiles` — carrying per-variant payloads; callers pattern-match to recover structured information. The final two variants are safety caps introduced in 0.5.0 (see [Safety Caps](#safety-caps-rust-only) below).

## Safety Caps (Rust only)

The Rust loader enforces two defensive caps that surface as structured errors rather than unbounded resource use on untrusted input:

| Cap | Default | Error variant | Payload |
|-----|---------|---------------|---------|
| Max file size | 16 MiB | `FileTooLarge` | `{ path, size, max }` |
| Max files per directory scan | 10,000 | `TooManyFiles` | `{ path, max }` |

Python and TypeScript loaders do not currently enforce these caps. Callers in those SDKs that load untrusted directories should pre-validate file counts and sizes. Cross-SDK alignment for these caps is tracked as an open item — see the 0.5.0 changelog and cross-SDK sync reports.

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

## Contract: BindingLoader.load_data

### Inputs
- `data`: dict, required — pre-parsed YAML content. Must be a dict with a `"bindings"` key containing a list of binding entries (e.g., `{"bindings": [{...}, {...}]}`). A bare list is rejected with `BindingLoadError`.
- `strict`: bool, optional, default=false — if true, raises `BindingLoadError` on missing required fields (`input_schema`, `output_schema`)

### Errors
- `BindingLoadError` (Python raises, TypeScript throws, Rust returns `Err`) — top-level value is not a mapping, missing `bindings` key, invalid entry structure, or strict-mode violation

### Returns
- On success: `list[ScannedModule]` / `ScannedModule[]` / `Vec<ScannedModule>`

### Properties
- async: false
- pure: true (no filesystem access — operates on already-parsed data)
- thread_safe: true

---

## Contract: BindingLoadError

### Inputs
N/A — this is an exception class, not a callable function.

### Errors
N/A — exception classes are not called and do not raise secondary exceptions.

### Returns
N/A — exception classes are not called and do not return values.

### Fields (Python / TypeScript)
- `file_path` / `filePath`: string | None — path to the `.binding.yaml` file that triggered the error (if applicable)
- `module_id` / `moduleId`: string | None — module ID of the entry that failed (if applicable)
- `missing_fields` / `missingFields`: list[str] / string[] — field names missing in strict mode (empty list for non-strict errors)
- `reason`: string — human-readable description of the error

### Cross-SDK Shape

| SDK | Type | Shape |
|-----|------|-------|
| Python | `class BindingLoadError(Exception)` | Single class with all 4 fields as attributes |
| TypeScript | `class BindingLoadError extends Error` | Same 4 fields as camelCase properties |
| Rust | `enum BindingLoadError` (`thiserror`) | 7 variants: `PathNotFound { path }`, `FileRead { path, source }`, `YamlParse { path, source }`, `MissingFields { path: Option<String>, module_id: Option<String>, missing_fields: Vec<String> }`, `InvalidStructure { path: Option<String>, reason: String }`, `FileTooLarge { path, size, max }`, `TooManyFiles { path, max }` |

Rust callers pattern-match on the variant to recover structured information. Python/TypeScript callers access fields directly.

---

## Round-Trip Guarantee

`YAMLWriter.write` followed by `BindingLoader.load` preserves every persisted field: `module_id`, `target`, `description`, `documentation`, `tags`, `version`, `annotations` (including the 12 `ModuleAnnotations` fields such as `streaming`, `cache_ttl`), `examples`, `metadata`, `input_schema`, `output_schema`, and `display`. This is covered by round-trip tests in all three SDKs.

Fields that `YAMLWriter` does not emit (e.g., `warnings`) are not preserved — those default to empty on load.

---

## Contract: BindingLoader.load

### Inputs
- `path`: string or Path, required — path to a `.binding.yaml` file OR a directory containing `.binding.yaml` files
- `strict`: bool, optional, default=false — if true, raises on any malformed binding entry
- `recursive`: bool / `{ recursive?: boolean }`, optional, default=false — when `true`, all three SDKs walk subdirectories recursively for `*.binding.yaml` files (Python via `rglob`, TypeScript via `_collectRecursive`, Rust via `walkdir::WalkDir`).

### Errors
- `BindingLoadError` / `BindingLoadError` (Python raises, Rust returns `Err`) — path not found, YAML parse failure, or strict mode violation
- `BindingLoadError::FileRead` (Rust) — any OS/IO error on the *root* path; per-entry errors during recursive traversal are governed by the policy below
- `BindingLoadError` (Python) — OS errors on the root path wrapping `IOError`/`OSError` raise immediately

### Recursive scan error handling

When `recursive=true` and the scan encounters a per-entry I/O error
(e.g., `EACCES` / `EPERM` on a subdirectory or unreadable file), the canonical
behavior is **best-effort**: emit a warning, skip the unreadable entry, and
continue traversing.

| SDK        | Current behavior                                                                                 | Aligned? |
|------------|--------------------------------------------------------------------------------------------------|----------|
| TypeScript | Best-effort: warn + skip + continue                                                              | ✓        |
| Python     | Best-effort by default — `Path.glob` skips inaccessible subdirectories on most platforms         | partial — platform-dependent |
| Rust       | Currently fail-fast (`BindingLoadError::FileRead`) — pending alignment with best-effort policy   | ✗ pending |

`BindingLoadError` is still raised when the *root* path is inaccessible or
missing — the best-effort policy only applies to per-entry errors encountered
during recursive traversal.

### Returns
- On success: `list[ScannedModule]` / `ScannedModule[]` / `Vec<ScannedModule>` — all modules loaded from the file or directory
- On directory with no `.binding.yaml` files: returns empty list (not an error)

### Properties
- async: false
- pure: false (reads filesystem)
- thread_safe: true (no shared state)

---

## TypeScript-only Extensions

### BindingParser (TypeScript only)

The TypeScript implementation splits binding loading into two classes for browser/edge runtime compatibility:

- **`BindingParser`** — runtime-neutral in-memory parser. Accepts raw YAML string content and returns parsed `BindingDocument` objects. Has no filesystem dependency. Available via both the main and browser entry points.
- **`parseBindingDocument(content: string): BindingDocument`** — standalone function wrapping BindingParser for callers that don't need class instantiation.
- **`BindingLoader`** — extends BindingParser with Node.js filesystem I/O (`load(path, strict?, recursive?)`). Available via the main entry point only.

### Contract: BindingParser.parse (TypeScript only)

#### Inputs
- `content`: string, required — the raw YAML text of a binding document (the contents of a `.binding.yaml` file). Must be valid YAML and the top-level value must be a mapping.

#### Errors
- `BindingLoadError` (TypeScript throws) — content is not valid YAML, the top-level value is not a mapping, the `bindings` key is missing or not an array, an entry is missing required fields, or strict-mode validation fails.

#### Returns
- On success: `BindingDocument` (a parsed in-memory representation listing all binding entries).
- On failure: throws `BindingLoadError`.

#### Properties
- async: false
- pure: true (no filesystem access — operates purely on the input string)
- thread_safe: true

### Contract: parseBindingDocument (TypeScript only)

A standalone function that constructs a transient `BindingParser` and calls `parse(content)` once. Provided so callers that do not need to retain a parser instance can avoid the boilerplate. Behaviour, errors, and properties are identical to `BindingParser.parse` above — this is a thin wrapper.

#### Inputs
- `content`: string, required — see `BindingParser.parse`.

#### Errors
- `BindingLoadError` (TypeScript throws) — see `BindingParser.parse`.

#### Returns
- On success: `BindingDocument`.
- On failure: throws `BindingLoadError`.

#### Properties
- async: false
- pure: true
- thread_safe: true

### Browser / Edge Runtime Subpath

The package exports a `apcore-toolkit/browser` subpath that includes only:
- `BindingParser` and `parseBindingDocument` (filesystem-free)
- All verifiers, formatters, and display utilities that have no Node.js dependencies

Import from the browser subpath in edge runtimes:

```typescript
import { BindingParser } from 'apcore-toolkit/browser';
```

Python and Rust do not have an equivalent browser/edge runtime entry point — these are platform-specific concerns only relevant in the TypeScript/JavaScript ecosystem.

---

## Comparison with `apcore.BindingLoader`

| Aspect | `apcore-toolkit` `BindingLoader` | `apcore` `BindingLoader` |
|--------|-----------------------------------|---------------------------|
| Returns | `list[ScannedModule]` (data) | `list[FunctionModule]` (runtime) |
| Imports `target` | No | Yes (via `importlib.import_module`) |
| Registers modules | No | Yes (mutates a `Registry`) |
| Required fields | `module_id`, `target` (loose) | `module_id`, `target` (always) |
| Use case | Tooling, CI, merging, diffing | Runtime module loading |

Use the toolkit loader when you need to **inspect or transform** binding files. Use `apcore.BindingLoader` when you need to **execute** the modules they describe.
