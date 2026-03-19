# Getting Started with apcore-toolkit

This guide walks you through installing **apcore-toolkit** and using its core modules to extract metadata from your code and generate AI-perceivable outputs.

## Prerequisites

=== "Python"

    - **Python**: 3.11+
    - **apcore**: 0.13.0+

=== "TypeScript"

    - **Node.js**: 18+
    - **apcore**: 0.13.0+

---

## Installation

=== "Python"

    ```bash
    pip install apcore-toolkit
    ```

=== "TypeScript"

    ```bash
    npm install apcore-toolkit
    ```

---

## Step 1: Create a Custom Scanner

The `BaseScanner` provides the foundation for extracting metadata from any framework.

=== "Python"

    ```python
    from apcore_toolkit import BaseScanner, ScannedModule

    class MyScanner(BaseScanner):
        def scan(self, **kwargs):
            # Scan your framework endpoints and return ScannedModule instances
            return [
                ScannedModule(
                    module_id="users.get_user",
                    description="Get a user by ID",
                    input_schema={"type": "object", "properties": {"id": {"type": "integer"}}, "required": ["id"]},
                    output_schema={"type": "object", "properties": {"name": {"type": "string"}}},
                    tags=["users"],
                    target="myapp.views:get_user",
                )
            ]

        def get_source_name(self):
            return "my-framework"

    scanner = MyScanner()
    modules = scanner.scan()
    ```

=== "TypeScript"

    ```typescript
    import { BaseScanner, ScannedModule, createScannedModule } from "apcore-toolkit";

    class MyScanner extends BaseScanner {
      scan(): ScannedModule[] {
        // Scan your framework endpoints and return ScannedModule instances
        return [
          createScannedModule({
            moduleId: "users.get_user",
            description: "Get a user by ID",
            inputSchema: { type: "object", properties: { id: { type: "integer" } }, required: ["id"] },
            outputSchema: { type: "object", properties: { name: { type: "string" } } },
            tags: ["users"],
            target: "myapp/views:get_user",
          }),
        ];
      }

      getSourceName(): string {
        return "my-framework";
      }
    }

    const scanner = new MyScanner();
    const modules = scanner.scan();
    ```

---

## Step 2: Filter and Deduplicate

Use built-in utilities to refine your scanned modules.

=== "Python"

    ```python
    # Filter modules by ID using regex
    modules = scanner.filter_modules(modules, include=r"^users\.")

    # Ensure all module IDs are unique
    modules = scanner.deduplicate_ids(modules)
    ```

=== "TypeScript"

    ```typescript
    // Filter modules by ID using regex
    modules = scanner.filterModules(modules, { include: "^users\\." });

    // Ensure all module IDs are unique
    modules = scanner.deduplicateIds(modules);
    ```

---

## Step 3: Generate Output

Choose the output format that fits your workflow.

### Option A: YAML Bindings

Generates `.binding.yaml` files for `apcore.BindingLoader`.

=== "Python"

    ```python
    from apcore_toolkit import YAMLWriter

    writer = YAMLWriter()
    writer.write(modules, output_dir="./bindings")
    ```

=== "TypeScript"

    ```typescript
    import { YAMLWriter } from "apcore-toolkit";

    const writer = new YAMLWriter();
    writer.write(modules, "./bindings");
    ```

### Option B: Python / TypeScript Wrappers

Generates decorator-based wrapper files for your language.

=== "Python"

    ```python
    from apcore_toolkit import PythonWriter

    writer = PythonWriter()
    writer.write(modules, output_dir="./generated")
    ```

=== "TypeScript"

    ```typescript
    import { TypeScriptWriter } from "apcore-toolkit";

    const writer = new TypeScriptWriter();
    writer.write(modules, "./generated");
    ```

### Option C: Direct Registration

Registers modules directly into an active `apcore.Registry`.

=== "Python"

    ```python
    from apcore import Registry
    from apcore_toolkit import RegistryWriter

    registry = Registry()
    writer = RegistryWriter()
    writer.write(modules, registry)
    ```

=== "TypeScript"

    ```typescript
    import { Registry } from "apcore-js";
    import { RegistryWriter } from "apcore-toolkit";

    const registry = new Registry();
    const writer = new RegistryWriter();
    writer.write(modules, registry);
    ```

---

## Step 4: Use Schema Utilities

`apcore-toolkit` includes powerful utilities for working with schemas.

### Model Flattening

Flatten complex models into scalar keyword arguments, perfect for MCP (Model Context Protocol) tools.

=== "Python"

    ```python
    from apcore_toolkit import flatten_pydantic_params, resolve_target

    # Resolve a target string to a callable
    func = resolve_target("myapp.views:create_task")

    # Wrap the function to accept flat kwargs
    wrapped = flatten_pydantic_params(func)
    ```

=== "TypeScript"

    ```typescript
    import { flattenParams, resolveTarget } from "apcore-toolkit";
    import { z } from "zod";

    // Resolve a target string to a callable (resolveTarget is async)
    const func = await resolveTarget("myapp/views:createTask") as (input: Record<string, unknown>) => unknown;

    // Define the input schema and wrap the function for flat-arg invocation
    const schema = z.object({ id: z.number() });
    const wrapped = flattenParams(func, schema);
    ```

### OpenAPI Extraction

Extract JSON Schemas directly from OpenAPI operation objects.

=== "Python"

    ```python
    from apcore_toolkit.openapi import extract_input_schema, extract_output_schema

    input_schema = extract_input_schema(operation, openapi_doc)
    output_schema = extract_output_schema(operation, openapi_doc)
    ```

=== "TypeScript"

    ```typescript
    import { extractInputSchema, extractOutputSchema } from "apcore-toolkit";

    const inputSchema = extractInputSchema(operation, openapiDoc);
    const outputSchema = extractOutputSchema(operation, openapiDoc);
    ```

---

## Step 5: Enhance Metadata with AI (Optional)

The toolkit includes a built-in `AIEnhancer` that calls any OpenAI-compatible local API to fill metadata gaps (missing descriptions, annotations, schemas).

1. **Install Ollama** and pull a model (e.g., `qwen:0.6b` or a larger model for better quality).
2. **Enable**:
   ```bash
   export APCORE_AI_ENABLED=true
   ```
3. **Run your scanner** with `--ai-enhance`: Missing descriptions, annotations, and schemas will be inferred automatically.

For production use, we recommend **[apcore-refinery](https://github.com/aipartnerup/apcore-refinery)** which provides better prompts, model flexibility, and CI integration:

```bash
apcore-refinery enhance ./bindings --model <your-preferred-model>
```

See the [AI Enhancement Guide](ai-enhancement.md) for the full `Enhancer` protocol, `AIEnhancer` configuration, and apcore-refinery details.

---

## Next Steps

- **[Features Overview](features/overview.md)** — Deep dive into all toolkit capabilities.
- **[AI Enhancement Guide](ai-enhancement.md)** — Enhancer protocol, built-in AIEnhancer, and apcore-refinery.
- **[Changelog](../CHANGELOG.md)** — See what's new in the latest release.
- **[apcore Documentation](https://aipartnerup.github.io/apcore/)** — Learn more about the core framework.
