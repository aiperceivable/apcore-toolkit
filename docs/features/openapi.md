# OpenAPI Integration

The `apcore_toolkit.openapi` module provides utilities for extracting JSON Schemas directly from an OpenAPI specification, either by parsing the JSON document or by interacting with live OpenAPI endpoints.

## JSON Schema Extraction

The toolkit handles the extraction and merging of OpenAPI operation parameters into canonical JSON Schemas.

| Method | Description |
|--------|-------------|
| `extract_input_schema(op, doc)` | Merges query, path, and request body parameters into a single object schema. Recursively resolves all nested `$ref`. |
| `extract_output_schema(op, doc)` | Extracts response schema for `200` or `201` status codes. Recursively resolves all nested `$ref`. |
| `resolve_ref(ref_string, doc)` | Resolves a single internal JSON pointer reference (e.g., `#/components/schemas/User`). |
| `resolve_schema(schema, doc)` | If the schema contains a top-level `$ref`, resolves it (single level only). |
| `deep_resolve_refs(schema, doc)` | Recursively resolves all `$ref` pointers in a schema, including nested `allOf`/`anyOf`/`oneOf`, `items`, and `properties`. Depth-limited to 16 levels. |

## Parameter Merging

The `extract_input_schema()` function performs intelligent merging:
1.  **Path Parameters**: Extracted and marked as `required: true`.
2.  **Query Parameters**: Extracted, with required status preserved.
3.  **Request Body**: Properties from the `application/json` request body are merged into the same input schema.

This produces the flat `input_schema` required by the `ScannedModule`.

## Example Usage

=== "Python"

    ```python
    from apcore_toolkit.openapi import extract_input_schema, extract_output_schema

    # Load an OpenAPI spec
    openapi_spec = { ... }
    # Get an operation object
    operation = openapi_spec["paths"]["/users"]["post"]

    # Extract metadata
    input_schema = extract_input_schema(operation, openapi_spec)
    output_schema = extract_output_schema(operation, openapi_spec)

    # Create a ScannedModule
    module = ScannedModule(
        module_id="users.create",
        input_schema=input_schema,
        output_schema=output_schema,
        # ... other metadata
    )
    ```

=== "TypeScript"

    ```typescript
    import { extractInputSchema, extractOutputSchema, ScannedModule } from "apcore-toolkit";

    // Load an OpenAPI spec
    const openapiSpec = { ... };
    // Get an operation object
    const operation = openapiSpec.paths["/users"].post;

    // Extract metadata
    const inputSchema = extractInputSchema(operation, openapiSpec);
    const outputSchema = extractOutputSchema(operation, openapiSpec);

    // Create a ScannedModule
    const module = new ScannedModule({
      moduleId: "users.create",
      inputSchema,
      outputSchema,
      // ... other metadata
    });
    ```

## Reference Resolution

The toolkit provides three levels of `$ref` resolution:

1. **`resolve_ref(ref_string, doc)`** — Resolves a single JSON pointer (e.g., `#/components/schemas/User`) to the referenced schema dict.
2. **`resolve_schema(schema, doc)`** — If the top-level schema contains `$ref`, resolves it once. Returns inline schemas unchanged.
3. **`deep_resolve_refs(schema, doc)`** / **`deepResolveRefs(schema, doc)`** — Recursively walks the entire schema tree, resolving `$ref` inside `properties`, `allOf`/`anyOf`/`oneOf`, and `items`. Depth-limited to 16 levels to prevent infinite recursion on circular references.

Both `extract_input_schema` and `extract_output_schema` call `deep_resolve_refs` internally, so callers get fully resolved schemas by default.
