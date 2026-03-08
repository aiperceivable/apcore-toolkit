# OpenAPI Integration

The `apcore_toolkit.openapi` module provides utilities for extracting JSON Schemas directly from an OpenAPI specification, either by parsing the JSON document or by interacting with live OpenAPI endpoints.

## JSON Schema Extraction

The toolkit handles the extraction and merging of OpenAPI operation parameters into canonical JSON Schemas.

| Method | Description |
|--------|-------------|
| `extract_input_schema(op, doc)` | Merges query, path, and request body parameters into a single object schema. |
| `extract_output_schema(op, doc)` | Extracts response schema for `200` or `201` status codes. |
| `resolve_ref(ref_string, doc)` | Resolves internal JSON pointer references (e.g., `#/components/schemas/User`). |
| `resolve_schema(schema, doc)` | Recursively resolves `$ref` in a schema object. |

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
    import { extractInputSchema, extractOutputSchema } from "@anthropic/apcore-toolkit/openapi";
    import { ScannedModule } from "@anthropic/apcore-toolkit";

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

The toolkit includes a standalone JSON pointer resolver (`resolve_ref`) that ensures complex, nested OpenAPI schemas are correctly flattened into standalone JSON Schema objects, even when components are shared across many endpoints.
