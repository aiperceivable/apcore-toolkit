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
| `deep_resolve_refs(schema, doc)` | Recursively resolves all `$ref` pointers in a schema, including nested `allOf`, `anyOf`, `oneOf`, `items` (single schema), `properties`, `additionalProperties` (when schema), `patternProperties`, `not`, `if`, `then`, `else`, `prefixItems`, and tuple-form `items` (array of schemas). Depth-limited to 16 levels. |

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
    import { extractInputSchema, extractOutputSchema, createScannedModule } from "apcore-toolkit";

    // Load an OpenAPI spec
    const openapiSpec = { ... };
    // Get an operation object
    const operation = openapiSpec.paths["/users"].post;

    // Extract metadata
    const inputSchema = extractInputSchema(operation, openapiSpec);
    const outputSchema = extractOutputSchema(operation, openapiSpec);

    // Create a ScannedModule (use createScannedModule — ScannedModule is an interface)
    const module = createScannedModule({
      moduleId: "users.create",
      description: "Create a user",
      target: "myapp/views:createUser",
      inputSchema,
      outputSchema,
      tags: [],
    });
    ```

## Reference Resolution

The toolkit provides three levels of `$ref` resolution:

1. **`resolve_ref(ref_string, doc)`** — Resolves a single JSON pointer (e.g., `#/components/schemas/User`) to the referenced schema dict.
2. **`resolve_schema(schema, doc)`** — If the top-level schema contains `$ref`, resolves it once. Returns inline schemas unchanged.
3. **`deep_resolve_refs(schema, doc)`** / **`deepResolveRefs(schema, doc)`** — Recursively walks the entire schema tree, resolving `$ref` inside the following keywords: `allOf`, `anyOf`, `oneOf`, `items` (single schema), `properties`, `additionalProperties` (when it is a schema object, not a boolean), `patternProperties`, `not`, `if`, `then`, `else`, `prefixItems`, and tuple-form `items` (array of schemas). Depth-limited to 16 levels to prevent infinite recursion on circular references. This is the canonical behavior across all three SDKs after Python and TypeScript fixes are applied.

Both `extract_input_schema` and `extract_output_schema` call `deep_resolve_refs` internally, so callers get fully resolved schemas by default.

---

## Contract: extract_input_schema

### Inputs
- `op` / `operation`: dict, required — a single OpenAPI operation object (e.g., `spec["paths"]["/users"]["post"]`)
- `doc` / `spec`: dict, required — the full OpenAPI spec document (used for `$ref` resolution)

### Errors
- None raised — returns `{}` if the operation has no parameters or request body

### Returns
- On success: dict — a flat JSON Schema `{ "type": "object", "properties": {...}, "required": [...] }` merging path params, query params, and request body properties

### Properties
- async: false
- pure: true
- depth_limit: 16 — `$ref` resolution stops at 16 levels to prevent infinite recursion on circular references

---

## Contract: extract_output_schema

### Inputs
- `op` / `operation`: dict, required — a single OpenAPI operation object
- `doc` / `spec`: dict, required — the full OpenAPI spec document

### Errors
- None raised — returns `{}` if the operation has no `200`/`201` response schema

### Returns
- On success: dict — resolved JSON Schema for the `200` or `201` response body

### Properties
- async: false
- pure: true
- depth_limit: 16

---

## Contract: resolve_ref

### Inputs
- `ref_string` / `refString`: string, required — a JSON Pointer reference (e.g., `"#/components/schemas/User"`)
- `doc` / `spec`: dict, required — the full OpenAPI spec document to resolve within

### Errors
- None raised — returns `None`/`null` if the reference path is not found in the document

### Returns
- On success: dict — the resolved schema at the referenced path
- On missing: `None` / `null`

### Properties
- async: false
- pure: true

---

## Contract: resolve_schema

### Inputs
- `schema`: dict, required — a schema object that may or may not have a top-level `$ref`
- `doc` / `spec`: dict, required — the full OpenAPI spec document

### Errors
- None raised — returns the schema unchanged if it has no `$ref`

### Returns
- On success: dict — the schema with its top-level `$ref` resolved (single level only); if no `$ref`, returns the input schema unchanged

### Properties
- async: false
- pure: true

---

## Contract: deep_resolve_refs

### Inputs
- `schema`: dict, required — a schema object (may contain nested `$ref` and any of the following keywords: `allOf`, `anyOf`, `oneOf`, `items` (single schema), `properties`, `additionalProperties` (when schema), `patternProperties`, `not`, `if`, `then`, `else`, `prefixItems`, tuple-form `items` (array of schemas))
- `doc` / `spec`: dict, required — the full OpenAPI spec document
- `depth`: int, optional, default=0 — internal recursion depth counter (callers should not set this)

### Errors
- None raised — returns the schema unchanged when `depth > 16` to prevent infinite recursion

### Returns
- On success: dict — fully resolved schema with all `$ref` pointers inlined (up to 16 levels deep), with all of the above listed keywords recursively walked

### Properties
- async: false
- pure: true
- idempotent: true (calling twice on an already-resolved schema is a no-op)
- depth_limit: 16
- note: This is the canonical behavior as implemented in all three SDKs after the Python and TypeScript fixes are applied.
