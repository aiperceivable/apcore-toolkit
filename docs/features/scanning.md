# Smart Scanning

The `BaseScanner` ABC (Abstract Base Class) provides a consistent interface and shared utilities for framework-specific adapters (e.g., `django-apcore`, `flask-apcore`).

## `BaseScanner` Core Methods

| Method | Description |
|--------|-------------|
| `scan(**kwargs)` | **Abstract.** Framework-specific implementation of the scan logic. |
| `get_source_name()` | **Abstract.** Returns the scanner's name (e.g., "django-ninja"). |
| `extract_docstring(func)` | Convenience wrapper around `apcore.parse_docstring()`. |
| `filter_modules(...)` | Apply regex-based include/exclude filters to module IDs. |
| `infer_annotations_from_method(...)` | Infer `readonly`, `destructive`, or `idempotent` from HTTP methods. |
| `deduplicate_ids(...)` | Automatically resolve duplicate module IDs by appending suffixes (`_2`, `_3`). |

## Ability Extraction Methodology

When building a scanner for a new framework, follow this systematic approach to ensure comprehensive metadata extraction. This methodology is adapted from real-world experience scanning 10+ software systems.

### Phase 1: Identify the Backend Engine

Separate the framework's **routing/dispatch layer** from its **business logic layer**. The scanner should target the dispatch layer to discover endpoints, then reach into the business logic layer for metadata.

| Framework | Dispatch Layer | Business Logic |
|-----------|---------------|----------------|
| Django REST | `urlpatterns` + `ViewSet` | serializer methods, queryset logic |
| Flask | `@app.route` + Blueprints | view function body |
| FastAPI | `@router.get/post` | endpoint function with type hints |
| Express | `router.get/post` | handler functions |
| NestJS | `@Controller` + `@Get/@Post` | service methods |

### Phase 2: Map Operations to Modules

For each discovered endpoint, extract the canonical mapping:

```
Framework endpoint  →  ScannedModule
─────────────────     ─────────────
route path            module_id
handler function      target
request schema        input_schema
response schema       output_schema
docstring             description + documentation
(default)             version       (default: "1.0.0")
usage samples         examples      (default: [])
extra data            metadata      (default: {})
```

### `ScannedModule` Fields

| Field | Python Type | TypeScript Type | Default |
|-------|------------|-----------------|---------|
| `module_id` / `moduleId` | `str` | `string` | *(required)* |
| `description` | `str` | `string` | *(required)* |
| `target` | `str` | `string` | *(required)* |
| `input_schema` / `inputSchema` | `dict` | `Record<string, unknown>` | *(required)* |
| `output_schema` / `outputSchema` | `dict` | `Record<string, unknown>` | *(required)* |
| `tags` | `list[str]` | `string[]` | *(required)* |
| `annotations` | `ModuleAnnotations \| None` | `ModuleAnnotations \| null` | `None` / `null` |
| `version` | `str` | `string` | `"1.0.0"` |
| `examples` | `list[ModuleExample]` | `ModuleExample[]` | `[]` |
| `metadata` | `dict` | `Record<string, unknown>` | `{}` |
| `documentation` | `str` | `string` | `None` / `null` |
| `warnings` | `list[str]` | `string[]` | `[]` |

### Phase 3: Extract Data Models

Leverage the framework's native schema system:

- **Python**: Pydantic models, Django serializers, marshmallow schemas
- **TypeScript**: Zod schemas, class-validator decorators, interfaces

Use the toolkit's `flatten_pydantic_params()` or `flattenParams()` to convert nested models into flat schemas when needed.

### Phase 4: Discover Existing API Contracts

Check for existing machine-readable API definitions that can supplement or replace code scanning:

- OpenAPI/Swagger specs (use `extract_input_schema()` / `extract_output_schema()`)
- GraphQL schemas
- gRPC/Protobuf definitions
- Existing MCP server manifests

### Phase 5: Infer Behavioral Annotations

Go beyond HTTP method heuristics. Analyze the function body for behavioral signals:

| Signal in Code | Inferred Annotation |
|---------------|-------------------|
| `DELETE` method, `.delete()` calls, `DROP` SQL | `destructive=True` |
| `GET` method, no DB writes, pure computation | `readonly=True` |
| `PUT` method, upsert patterns | `idempotent=True` |
| Sends email/SMS, processes payment, modifies permissions | `requires_approval=True` |
| HTTP client calls, file I/O, subprocess | `open_world=True` |
| `yield`, `StreamingResponse`, `async for` | `streaming=True` |
| `GET` with stable results, no auth-dependent data | `cacheable=True` |
| `Paginator`, `LIMIT/OFFSET`, cursor tokens | `paginated=True` |

Static analysis can detect some of these patterns. For ambiguous cases, the built-in [AIEnhancer](../ai-enhancement.md) or [apcore-refinery](https://github.com/aiperceivable/apcore-refinery) can assist with AI-based inference.

## Implementation Example

When implementing a custom scanner, you inherit from `BaseScanner`:

=== "Python"

    ```python
    from apcore_toolkit import BaseScanner, ScannedModule

    class MyScanner(BaseScanner):
        def scan(self, **kwargs) -> list[ScannedModule]:
            # 1. Discover endpoints from your framework
            endpoints = self.get_my_endpoints()

            # 2. Convert endpoints to ScannedModules
            modules = []
            for e in endpoints:
                desc, docs, params = self.extract_docstring(e.view_func)
                modules.append(ScannedModule(
                    module_id=e.name,
                    description=desc,
                    target=f"{e.module}:{e.func_name}",
                    annotations=self.infer_annotations_from_method(e.method),
                    # ... other metadata
                ))

            # 3. Apply shared refining utilities
            modules = self.filter_modules(modules, include=kwargs.get("include"))
            modules = self.deduplicate_ids(modules)

            return modules

        def get_source_name(self) -> str:
            return "my-framework-scanner"
    ```

=== "TypeScript"

    ```typescript
    import { BaseScanner, ScannedModule, createScannedModule } from "apcore-toolkit";

    class MyScanner extends BaseScanner {
      scan(options?: { include?: string }): ScannedModule[] {
        // 1. Discover endpoints from your framework
        const endpoints = this.getMyEndpoints();

        // 2. Convert endpoints to ScannedModules
        let modules = endpoints.map((e) =>
          createScannedModule({
            moduleId: e.name,
            description: this.extractDocstring(e.viewFunc).description ?? '',
            inputSchema: {},
            outputSchema: {},
            tags: [],
            target: `${e.module}:${e.funcName}`,
            annotations: MyScanner.inferAnnotationsFromMethod(e.method),
            // ... other metadata
          })
        );

        // 3. Apply shared refining utilities
        modules = this.filterModules(modules, { include: options?.include });
        modules = this.deduplicateIds(modules);

        return modules;
      }

      getSourceName(): string {
        return "my-framework-scanner";
      }
    }
    ```

## Module Deduplication

Scanners often encounter naming collisions (e.g., `GET /users` and `POST /users` both being called `users`). `deduplicate_ids()` handles this by:
1.  Tracking encountered IDs.
2.  Appending `_2`, `_3` etc. to duplicates.
3.  Injecting a warning into the `ScannedModule.warnings` list for human audit.

## Behavioral Inference

`infer_annotations_from_method()` provides a sensible default for mapping HTTP verbs to apcore's `ModuleAnnotations`:
- `GET` → `readonly=True, cacheable=True`
- `DELETE` → `destructive=True`
- `PUT` → `idempotent=True`
- Others → Default (all False)

For deeper behavioral analysis beyond HTTP methods, see [Phase 5](#phase-5-infer-behavioral-annotations) above and the [AI Enhancement](../ai-enhancement.md) module.

---

## Contract: BaseScanner.filter_modules

### Inputs
- `modules`: list of `ScannedModule`, required
- `include`: string regex pattern, optional — if provided, only modules whose `module_id` matches are kept; invalid regex raises error
- `exclude`: string regex pattern, optional — if provided, modules whose `module_id` matches are removed; invalid regex raises error

### Errors
- `re.error` (Python) / `Error` (TypeScript) / `Err(regex::Error)` (Rust) — when `include` or `exclude` is an invalid regex pattern

### Returns
- On success: filtered `list[ScannedModule]` / `ScannedModule[]` / `Vec<ScannedModule>`

### Properties
- async: false
- thread_safe: true (no mutation of inputs)
- pure: true

---

## Contract: BaseScanner.deduplicate_ids

### Inputs
- `modules`: list of `ScannedModule`, required

### Errors
- None raised

### Returns
- On success: list with duplicate `module_id` values renamed (`_2`, `_3` suffix); original module order preserved; renamed modules have a warning appended to their `warnings` list

### Properties
- async: false
- pure: false (mutates warnings field of ScannedModule)

---

## Contract: BaseScanner.scan

### Inputs
- Framework-specific — subclass defines the signature. Python implementations typically accept `**kwargs`; TypeScript implementations accept an options object.

### Errors
- Framework-specific — subclass defines error behavior
- `re.error` (Python) / `Error` (TypeScript) — if `include`/`exclude` kwargs are forwarded to `filter_modules` with invalid patterns

### Returns
- On success: `list[ScannedModule]` / `ScannedModule[]` / `Vec<ScannedModule>` — all discovered modules after filtering and deduplication

### Properties
- async: false (Python, most TypeScript) / async: true (Rust — `async fn scan`)
- pure: false (reads framework state, imports modules, may have I/O side effects)

---

## Contract: BaseScanner.infer_annotations_from_method

### Inputs
- `method`: string HTTP verb (e.g., `"GET"`, `"POST"`, `"DELETE"`, `"PUT"`), required

### Errors
- None raised — unknown methods return default annotations (all fields false/None)

### Returns
- On success: `ModuleAnnotations` instance with `readonly`, `cacheable`, `destructive`, `idempotent` set based on HTTP method heuristics

### Properties
- async: false
- pure: true
- thread_safe: true

---

## Serialization Utilities

Two helper functions convert apcore objects to plain dictionaries for JSON/YAML serialization:

| Function (Python / TypeScript) | Description |
|-------------------------------|-------------|
| `annotations_to_dict()` / `annotationsToDict()` | Converts a `ModuleAnnotations` instance to a plain dict with snake_case keys |
| `module_to_dict()` / `moduleToDict()` | Converts a `ScannedModule` to a dict with snake_case keys, suitable for YAML/JSON output |
| `modules_to_dicts()` / `modulesToDicts()` | Converts a list of `ScannedModule` to a list of dicts (batch version of `module_to_dict`) |
