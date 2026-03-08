# Smart Scanning

The `BaseScanner` ABC (Abstract Base Class) provides a consistent interface and shared utilities for framework-specific adapters (e.g., `django-apcore`, `flask-apcore`).

## `BaseScanner` Core Methods

| Method | Description |
|--------|-------------|
| `scan(**kwargs)` | **Abstract.** Framework-specific implementation of the scan logic. |
| `get_source_name()` | **Abstract.** Returns the scanner's name (e.g., "django-ninja"). |
| `extract_docstring(func)` | Convenience wrapper around `apcore.parse_docstring()`. |
| `filter_modules(...)` | Apply regex-based include/exclude filters to module IDs. |
| `infer_annotations(...)` | Infer `readonly`, `destructive`, or `idempotent` from HTTP methods. |
| `deduplicate_ids(...)` | Automatically resolve duplicate module IDs by appending suffixes (`_2`, `_3`). |

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
    import { BaseScanner, ScannedModule } from "@anthropic/apcore-toolkit";

    class MyScanner extends BaseScanner {
      scan(options?: { include?: RegExp }): ScannedModule[] {
        // 1. Discover endpoints from your framework
        const endpoints = this.getMyEndpoints();

        // 2. Convert endpoints to ScannedModules
        let modules = endpoints.map((e) =>
          new ScannedModule({
            moduleId: e.name,
            description: this.extractDocstring(e.viewFunc).description,
            target: `${e.module}:${e.funcName}`,
            annotations: this.inferAnnotationsFromMethod(e.method),
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
- `GET` $\rightarrow$ `readonly=True`
- `DELETE` $\rightarrow$ `destructive=True`
- `PUT` $\rightarrow$ `idempotent=True`
- Others $\rightarrow$ Default (all False)
