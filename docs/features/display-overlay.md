# Display Overlay

The `DisplayResolver` applies a sparse `binding.yaml` overlay to a list of `ScannedModule` objects, resolving surface-facing presentation fields — alias, description, guidance, tags, and documentation — into `metadata["display"]` for downstream CLI, MCP, and A2A consumers.

The overlay is provided either as pre-parsed `binding_data` or a `binding_path` pointing at YAML file(s). `ScannedModule` also carries an independent [`display` field](#scannedmoduledisplay-field) (since 0.5.0) used by `YAMLWriter` and `BindingLoader` for round-trip persistence — `DisplayResolver` itself does not consume that field.

## Overview

Scanners produce raw `ScannedModule` metadata from code. The display overlay layer sits between scanning and surface emission: it reads a `binding.yaml` (or a directory of `*.binding.yaml` files) and merges human-curated presentation overrides on top of the scanned values without modifying the underlying schema or target.

This approach is intentionally **sparse**: a binding file only needs to contain the fields you want to override. Unset fields fall through to sensible defaults.

## `DisplayResolver`

**Class**: `apcore_toolkit.display.DisplayResolver`
**Module**: `apcore_toolkit/display/resolver.py`

### Method Signature

```python
from apcore_toolkit.display import DisplayResolver

resolver = DisplayResolver()
resolved_modules = resolver.resolve(
    modules,
    *,
    binding_path=None,   # str | Path | None
    binding_data=None,   # dict | None
)
```

**Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `modules` | `list[ScannedModule]` | Modules to resolve. Returned list has the same length and order. |
| `binding_path` | `str \| Path \| None` | Path to a single `.binding.yaml` file **or** a directory containing `*.binding.yaml` files. Ignored when `binding_data` is provided. |
| `binding_data` | `dict \| None` | Pre-parsed binding dict. Takes precedence over `binding_path` when both are supplied. |

**Returns**: `list[ScannedModule]` — the same modules with `metadata["display"]` populated.

### Resolution Chain

For each presentation field (`alias`, `description`, `guidance`, `tags`, `documentation`), the resolver walks the following chain and uses the **first non-absent value**:

1. **Surface-specific override** — e.g., `modules.<id>.cli.alias`
2. **`display` default** — e.g., `modules.<id>.display.alias`
3. **Binding-level field** — top-level `alias`, `description`, etc. in the binding entry
4. **Scanner value** — the value already present in the `ScannedModule` (e.g., `description`, `tags`)

If none of the above is set, the field is omitted from `metadata["display"]`.

### `suggested_alias` Fallback

When a scanner runs with `simplify_ids=True`, it may emit `suggested_alias` inside `ScannedModule.metadata`. `DisplayResolver` uses this as an additional fallback for `alias` **before** falling back to `module_id`:

```
surface override > display.alias > binding alias > suggested_alias > module_id
```

No warning is emitted when `suggested_alias` is used as the alias source.

## Output Structure

After resolution, each module's `metadata["display"]` contains a flat dict with optional per-surface sub-dicts:

```python
module.metadata["display"] = {
    # Shared fields (resolved from the chain above)
    "alias":         "get-user",
    "description":   "Retrieve a user by their unique identifier.",
    "documentation": "Returns 404 if the user does not exist.",
    "guidance":      "Prefer this over listing all users when the ID is known.",
    "tags":          ["users", "read-only"],

    # Surface-specific overrides (only present when the binding sets them)
    "cli": {
        "alias":       "get-user",
        "description": "Fetch a user record (CLI variant).",
        "guidance":    "Pass --id as a positional argument.",
        "tags":        ["users"],
    },
    "mcp": {
        "alias":       "get_user",
        "description": "Retrieve a user by ID (MCP tool).",
        "guidance":    "Provide the numeric user ID.",
        "tags":        ["users", "read-only"],
        "documentation": "See /docs/api/users for the full schema.",
    },
    "a2a": {
        "alias":       "get-user",
        "description": "Agent-to-agent: retrieve a user record.",
        "tags":        ["users"],
    },
}
```

The three surfaces are `cli`, `mcp`, and `a2a`. A surface sub-dict is only added to `metadata["display"]` when at least one surface-specific field is present in the binding.

## Alias Constraints by Surface

### MCP Alias

MCP tool names must conform to `[a-zA-Z0-9_-]+` with a maximum of 64 characters.

`DisplayResolver` applies automatic sanitization to the resolved MCP alias:

1. Replace every character outside `[a-zA-Z0-9_-]` with `_`.
2. If the sanitized result starts with a digit, prepend `_`.
3. If the final sanitized alias exceeds 64 characters, raise `ValueError`.

```python
# "users.get user" → "users_get_user"  (dot and space replaced)
# "1get-user"      → "_1get-user"      (leading digit prefixed)
# "a" * 65         → ValueError        (exceeds 64-char limit)
```

### CLI Alias

CLI command names must match `^[a-z][a-z0-9_-]*$`.

`DisplayResolver` does **not** silently sanitize CLI aliases. Instead:

- If a CLI alias was set **explicitly** in the binding and does not match the pattern, a `WARNING` is logged and the resolver falls back to the `display.alias` value (or the next item in the resolution chain).
- If the alias originates from the scanner value or `suggested_alias` (i.e., was never explicitly set in the binding), it is accepted without warning even if it contains uppercase letters or other non-conforming characters — the downstream CLI adapter is expected to normalise it.

## Match-Count Logging

After resolving all modules against a binding map, `DisplayResolver` logs:

| Level | Condition | Message |
|-------|-----------|---------|
| `INFO` | Always (when a binding is loaded) | `"DisplayResolver: matched N of M modules"` |
| `WARNING` | Binding loaded but zero modules matched | `"DisplayResolver: binding map loaded but no modules matched — check module IDs"` |

No log is emitted when neither `binding_path` nor `binding_data` is provided (pass-through mode).

## Binding File Format

`binding_path` accepts:

- A **single file** with extension `.binding.yaml` (or `.yaml`).
- A **directory**, in which case all `*.binding.yaml` files are loaded and merged. Files are processed in lexicographic order; later files win on key collision.

`binding_data` takes precedence over `binding_path` when both are supplied.

### Sample `binding.yaml`

The top-level key is the `module_id`. All fields are optional — include only what you want to override.

```yaml
# users.get_user.binding.yaml

module_id: users.get_user

# Binding-level fields (lowest override priority, above scanner value)
alias: get-user
description: Retrieve a user by their unique identifier.
documentation: Returns 404 if the user does not exist.
guidance: Prefer this over listing all users when the ID is known.
tags:
  - users
  - read-only

# display block: shared defaults, override the binding-level fields above
display:
  alias: get-user
  description: Retrieve a user record.
  guidance: Prefer this over listing all users when the ID is known.
  tags:
    - users

# Surface-specific overrides (highest priority)
cli:
  alias: get-user
  description: Fetch a user record.
  guidance: Pass --id as a positional argument.
  tags:
    - users

mcp:
  alias: get_user
  description: Retrieve a user by ID.
  guidance: Provide the numeric user ID in the `id` field.
  documentation: See /docs/api/users for the full schema.
  tags:
    - users
    - read-only

a2a:
  alias: get-user
  description: Agent-to-agent user retrieval.
  tags:
    - users
```

## Code Example

```python
from apcore_toolkit.display import DisplayResolver

# modules is a list[ScannedModule] from any scanner
resolver = DisplayResolver()

# Option A: point to a directory of *.binding.yaml files
resolved = resolver.resolve(modules, binding_path="./bindings")

# Option B: point to a single file
resolved = resolver.resolve(modules, binding_path="./bindings/users.get_user.binding.yaml")

# Option C: supply pre-parsed data (e.g. from a config system)
binding_data = {
    "users.get_user": {
        "display": {"alias": "get-user"},
        "mcp": {"alias": "get_user"},
    }
}
resolved = resolver.resolve(modules, binding_data=binding_data)

# Inspect the result
for m in resolved:
    display = m.metadata.get("display", {})
    print(m.module_id, "→", display.get("alias"), display.get("mcp", {}).get("alias"))
```

## `ScannedModule.display` field

Since `apcore-toolkit` 0.5.0, `ScannedModule` carries an optional top-level `display` field that holds the **sparse overlay** (the same shape users write in `binding.yaml`) — *not* the fully resolved form produced by `DisplayResolver`.

```python
# types.py
display: dict[str, Any] | None = None
```

This field distinguishes two distinct pieces of data:

| Field | Shape | Produced by | Consumed by |
|-------|-------|-------------|-------------|
| `ScannedModule.display` | **sparse overlay** — only keys the user wants to override (`{"mcp": {"alias": "x"}}`) | User (manually), `BindingLoader`, convention scanners | `YAMLWriter`, round-trip tooling |
| `ScannedModule.metadata["display"]` | **resolved** — all surfaces expanded (`cli`/`mcp`/`a2a` complete) | `DisplayResolver.resolve()` | Surface adapters at render time |

### Persistence via `YAMLWriter`

When `module.display is not None`, `YAMLWriter` emits the overlay verbatim as a top-level `display:` key in the generated binding YAML. When `display is None`, the key is omitted entirely (keeps YAML clean).

This enables full round-trip through `BindingLoader` → `YAMLWriter`: display overrides committed to YAML survive re-scanning as long as scanners merge their output with pre-existing binding files (or delegate persistence to the loader).

### Relationship to `DisplayResolver`

`DisplayResolver` is unchanged — it reads overlays from `binding_data`/`binding_path` and writes resolved forms to `metadata["display"]`. The new `display` field is a **separate, persistence-oriented slot**: `YAMLWriter` emits it, `BindingLoader` reads it back.

Typical flow:

1. Scanner produces `ScannedModule` with `display=None`.
2. User hand-edits a `binding.yaml`, adding a `display:` section.
3. `BindingLoader` loads YAML → `ScannedModule.display` populated.
4. `YAMLWriter.write(modules)` re-emits YAML with `display` preserved verbatim.
5. Separately, `DisplayResolver.resolve(modules, binding_path=...)` expands the overlay into `metadata["display"]` for surface adapters.

To feed `DisplayResolver` from `ScannedModule.display`, pass it explicitly via `binding_data` (wrap in `{"bindings": [{"module_id": ..., "display": ...}, ...]}`).

---

## Contract: DisplayResolver.resolve

### Inputs
- `modules`: list of `ScannedModule`, required
- `binding_path`: string or Path, optional — path to a `.binding.yaml` file or directory for loading display overlays
- `binding_data`: dict/object, optional — in-memory binding data (alternative to `binding_path`)

### Errors
- `ValueError` (Python) / `Error` (TypeScript) / `Err(DisplayResolverError)` (Rust) — invalid binding format or conflicting resolution options

### Returns
- On success: list of `ScannedModule` with `.display` fields populated where overlay data was found; modules without matching overlay data have `display=None`

### Properties
- async: false
- pure: false (reads filesystem if `binding_path` is provided)
- idempotent: true (applying same overlay twice yields same result)

---

## Integration with `simplify_ids`

When the scanner is configured with `simplify_ids=True`, each `ScannedModule.metadata` may contain a `suggested_alias` key. `DisplayResolver` picks this up automatically as the alias fallback:

```python
from fastapi_apcore import get_scanner
from apcore_toolkit.display import DisplayResolver

scanner = get_scanner(app, simplify_ids=True)
modules = scanner.scan()

resolver = DisplayResolver()
resolved = resolver.resolve(modules, binding_path="./bindings")

# Modules without a binding entry still get a clean alias from suggested_alias
for m in resolved:
    print(m.metadata["display"].get("alias"))  # e.g. "get_user" not "users__get_user"
```
