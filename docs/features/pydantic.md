# Schema Utilities

Modern web frameworks use structured models (Pydantic in Python, Zod/interfaces in TypeScript) for request and response validation. `apcore-toolkit` provides utilities to bridge the gap between structured models and the flat interface often required by AI tools.

## Model Flattening (Python only)

The `flatten_pydantic_params()` function wraps a Python function, converting its Pydantic model parameters into flat, scalar keyword arguments.

!!! info "TypeScript and Rust equivalents"
    TypeScript functions natively accept object arguments — e.g. `function createUser(body: { username, email })` — so what `flatten_pydantic_params` does for Python (unwrapping a `UserCreate` Pydantic model into flat kwargs) is already idiomatic in TypeScript. Users who need to iterate a Zod schema's fields at runtime can do so directly: `Object.keys(schema.shape)`. No toolkit wrapper ships.

    Rust uses compile-time proc macros for schema-derived flat parameter lists; there is no runtime equivalent.

### Why Flatten?

AI agents and protocols like MCP (Model Context Protocol) interact best with flat schemas. A function like `create_task(body: TaskCreate)` is difficult for an AI to call because it needs to understand the `TaskCreate` structure. By flattening, the AI sees `create_task(title: str, description: str, ...)` directly.

### Example

```python
from pydantic import BaseModel
from apcore_toolkit import flatten_pydantic_params

class UserCreate(BaseModel):
    username: str
    email: str

def create_user(body: UserCreate):
    return f"Created {body.username}"

# Wrap the function
flat_create_user = flatten_pydantic_params(create_user)

# Now it can be called with flat kwargs
result = flat_create_user(username="jdoe", email="joe@example.com")
```

## Contract: flatten_pydantic_params

### Inputs
- `func`: callable, required — the function whose Pydantic model parameters to flatten

### Errors
- None raised — if the function has no Pydantic model parameters, returns the function unchanged. `NameError` from unresolved forward references and `TypeError` from invalid annotations are also treated as "no flattening possible" and return the original callable; all other exceptions propagate.

### Returns
- On success: a wrapper function with the same return type but with Pydantic model params replaced by flat scalar kwargs. The wrapper's `__signature__` and `__annotations__` reflect the flat surface.

### Properties
- async: false
- pure: true (produces a new wrapper function, does not mutate the original)
- availability: Python only. TypeScript uses object-argument idioms natively; Rust uses compile-time proc macros.

---

## Metadata Preservation

When flattening, `apcore-toolkit` preserves:

- **Type Hints**: Original field types are maintained in the wrapper's signature.
- **Field Metadata**: Descriptions, examples, and schema metadata from the model are preserved on the wrapper's parameters.

## Contract: resolve_target / resolveTarget

### Inputs
- `target`: string, required — dotted import path in the form `"module.path:attribute"` (Python/Rust) or `"module/path:attribute"` (TypeScript)
- `allowed_prefixes` / `allowedPrefixes`: list[str] / string[], optional, default=None/null — allowlist that mitigates arbitrary-code-execution via forged binding files (e.g. a malicious `target: "os:system"` injected into untrusted YAML).
    - **Python**: list of **module-name** prefixes (e.g. `["myapp", "myorg.adapters"]`). When set, `module_path` must equal one of the prefixes or be a dotted descendant; otherwise `PermissionError` is raised.
    - **TypeScript**: list of **directory** prefixes for file-path imports. When set, only file-path imports resolving under one of these directories are permitted; bare package names and `node:` builtins are rejected.
    - **Rust**: not applicable — `resolve_target` is parse-only. Target-to-handler mapping runs through the application's `HandlerFactory`, which provides stronger isolation than any runtime allowlist.

### Errors
- `ValueError` (Python) / `Error` (TypeScript) — invalid target format (missing `:` separator)
- `PermissionError` (Python) / `Error` (TypeScript) — `allowed_prefixes` set and `module_path` not permitted
- `ImportError` / `ModuleNotFoundError` (Python) — the module path cannot be imported
- `Error` (TypeScript) — dynamic import failed or attribute not found
- `Err(ResolveTargetError)` (Rust) — parse failure (5 variants: `MissingSeparator`, `EmptyModulePath`, `EmptyQualname`, `InvalidQualname`, `InvalidModulePath`); Rust does NOT perform runtime import

### Returns
- Python: the callable (actual function/class loaded via `importlib`)
- TypeScript: `Promise<callable>` — always async; must be awaited
- Rust: `ResolvedTarget { module_path: String, qualname: String }` — parse-only result; Rust does not import or execute anything

### Properties
- async: false (Python, Rust) / true (TypeScript — always `await resolveTarget(...)`)
- pure: false (Python/TypeScript: performs side-effectful module import; Rust: pure string parsing)
- availability: All three SDKs, but Rust is parse-only (no runtime import capability)

---

## Target Resolution

The `resolve_target()` function resolves a string reference into the actual callable. This is essential for dynamically loading view functions referenced in metadata files.

=== "Python"

    ```python
    from apcore_toolkit import resolve_target

    # Dynamically load a callable
    view_func = resolve_target("myapp.api.v1.users:create_user")
    # view_func is now the actual function from myapp
    ```

=== "TypeScript"

    ```typescript
    import { resolveTarget } from "apcore-toolkit";

    // Dynamically load a callable
    const viewFunc = await resolveTarget("myapp/api/v1/users:createUser");
    // viewFunc is now the actual function from myapp
    ```
