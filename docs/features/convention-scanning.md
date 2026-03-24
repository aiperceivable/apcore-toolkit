# Convention Scanning

The `ConventionScanner` discovers plain Python functions as apcore modules from a `commands/` directory -- zero decorators, zero imports required. Each public function becomes a `ScannedModule` with schema inferred from type annotations and description extracted from docstrings.

**Cross-reference**: PROTOCOL_SPEC Section 5.14 (Convention-based Module Discovery)

## Overview

Convention scanning complements decorator-based and OpenAPI-based scanning by allowing developers to write plain functions in a file-system hierarchy. The scanner walks a directory tree, imports each `.py` file, and converts every public function into a `ScannedModule`.

This is the lowest-friction path to creating apcore modules: drop a `.py` file into `commands/`, define a typed function, and the scanner handles the rest.

## `ConventionScanner`

**Class**: `apcore_toolkit.convention_scanner.ConventionScanner`
**Module**: `apcore_toolkit/convention_scanner.py`

### Method Signature

```python
from apcore_toolkit.convention_scanner import ConventionScanner

scanner = ConventionScanner()
modules = scanner.scan(
    commands_dir,           # str | Path
    *,
    include=None,           # str | None  — regex to include module IDs
    exclude=None,           # str | None  — regex to exclude module IDs
)
```

**Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `commands_dir` | `str \| Path` | Path to the commands directory to scan. |
| `include` | `str \| None` | Regex pattern; only module IDs matching this pattern are kept. |
| `exclude` | `str \| None` | Regex pattern; module IDs matching this pattern are removed. |

**Returns**: `list[ScannedModule]` — discovered modules sorted by file path.

## File Discovery Rules

The scanner applies the following rules when walking the `commands_dir` tree:

1. **Recursive glob** — all `*.py` files under `commands_dir` are considered, including nested subdirectories.
2. **Skip `_`-prefixed files** — any file whose name starts with `_` (e.g., `__init__.py`, `_helpers.py`) is silently skipped.
3. **Skip private functions** — functions whose name starts with `_` are ignored.
4. **Skip imported functions** — only functions defined in the file itself (where `func.__module__` matches the loaded module name) are included. This prevents re-exporting helpers from polluting the module list.
5. **Skip reserved parameters** — parameters named `self`, `cls`, `ctx`, or `context` are excluded from the input schema.
6. **Error isolation** — if a file fails to import, a WARNING is logged and scanning continues with the remaining files.

If `commands_dir` does not exist or is not a directory, the scanner logs a WARNING and returns an empty list.

## Module ID Generation

Each discovered function receives a module ID of the form:

```
{prefix}.{function_name}
```

The **prefix** is determined by:

1. **`MODULE_PREFIX` constant** — if the file defines a module-level `MODULE_PREFIX: str` variable, its value is used as the prefix.
2. **File path fallback** — otherwise, the prefix is derived from the file's path relative to `commands_dir`, with directory separators replaced by `.` and the `.py` extension stripped.

**Examples**:

| File Path | `MODULE_PREFIX` | Function | Module ID |
|-----------|-----------------|----------|-----------|
| `commands/users.py` | (not set) | `create` | `users.create` |
| `commands/ops/deploy.py` | (not set) | `run` | `ops.deploy.run` |
| `commands/billing.py` | `"payments"` | `charge` | `payments.charge` |

## Schema Inference from Type Hints

The scanner builds JSON Schema `input_schema` and `output_schema` from function signatures and type annotations (subset of PROTOCOL_SPEC Section 5.11.5).

### Input Schema

For each parameter (excluding reserved names), the scanner:

1. Maps the type annotation to JSON Schema via the built-in type map.
2. Marks parameters without a default value as `required`.
3. Records non-`None` default values in the schema's `default` field.

**Type mapping**:

| Python Type | JSON Schema |
|-------------|-------------|
| `str` | `{"type": "string"}` |
| `int` | `{"type": "integer"}` |
| `float` | `{"type": "number"}` |
| `bool` | `{"type": "boolean"}` |
| `list` | `{"type": "array"}` |
| `dict` | `{"type": "object"}` |
| `list[X]` | `{"type": "array", "items": <schema of X>}` |
| (no annotation) | `{"type": "string"}` (fallback) |

### Output Schema

The return type annotation is converted using the same type map. If the return type is `None` or absent, the output schema is an empty dict.

## Metadata Constants

Files may define module-level constants to control scanning behavior:

| Constant | Type | Effect |
|----------|------|--------|
| `MODULE_PREFIX` | `str` | Overrides the file-path-derived prefix for all functions in the file. |
| `CLI_GROUP` | `str` | Sets `metadata["display"]["cli"]["group"]` on every module in the file, controlling CLI group placement (see FE-09). |
| `TAGS` | `list[str]` | Applied as `tags` on every `ScannedModule` produced from the file. |

## Include / Exclude Filters

Both `include` and `exclude` accept Python regex patterns and are applied **after** all files are scanned:

- `include` — only module IDs where `re.search(include, module_id)` is truthy are kept.
- `exclude` — module IDs where `re.search(exclude, module_id)` is truthy are removed.

When both are provided, `include` is applied first, then `exclude`.

```python
# Only scan modules under the "ops" namespace, but skip anything with "debug"
modules = scanner.scan("commands/", include=r"^ops\.", exclude=r"debug")
```

## Target Format

Each `ScannedModule.target` is set to `{file_path}:{function_name}`, providing a locator that downstream executors can use to import and call the function.

## Code Example

```python
from apcore_toolkit.convention_scanner import ConventionScanner

scanner = ConventionScanner()
modules = scanner.scan("commands/")

for m in modules:
    print(f"{m.module_id}: {m.description}")
    print(f"  target: {m.target}")
    print(f"  params: {list(m.input_schema.get('properties', {}).keys())}")
    print(f"  tags:   {m.tags}")
```

Given a file `commands/ops.py`:

```python
"""Operations commands."""

MODULE_PREFIX = "ops"
CLI_GROUP = "operations"
TAGS = ["infra", "deploy"]


def deploy(environment: str, dry_run: bool = False) -> dict:
    """Deploy to the target environment."""
    return {"status": "deployed", "env": environment}
```

The scanner produces a single `ScannedModule`:

```python
ScannedModule(
    module_id="ops.deploy",
    description="Deploy to the target environment.",
    input_schema={
        "type": "object",
        "properties": {
            "environment": {"type": "string"},
            "dry_run": {"type": "boolean", "default": False},
        },
        "required": ["environment"],
    },
    output_schema={"type": "object"},
    tags=["infra", "deploy"],
    target="commands/ops.py:deploy",
    metadata={"display": {"cli": {"group": "operations"}}},
)
```
