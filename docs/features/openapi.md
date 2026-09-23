---
description: "Extract JSON Schemas from OpenAPI operations: merge path/query/body params into input_schema, pull 200/201 output_schema, and resolve nested $ref."
---

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
| `deep_resolve_refs(schema, doc)` | Recursively resolves all `$ref` pointers in a schema, including nested `allOf`, `anyOf`, `oneOf`, `items` (single schema), `properties`, `additionalProperties` (when schema), `patternProperties`, `not`, `if`, `then`, `else`, `prefixItems`, and tuple-form `items` (array of schemas). Keys sitting beside a `$ref` are merged over the resolved target, sibling wins. Depth-limited to 16 levels. |

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
- None raised — returns an empty dict/object (`{}`) if the reference path is not found, the resolved value is not an object, or any pointer segment cannot be traversed.

### Returns
- On success: dict — the resolved schema at the referenced path
- On missing or non-object resolution: empty dict/object (`{}`). Distinguish "missing" from "explicitly empty" by walking the parent path with a membership check if needed.

### Properties
- async: false
- pure: true

### Language-specific hardening
- **TypeScript only:** `resolve_ref` blocks pointer segments matching `__proto__`, `constructor`, or `prototype` and returns `{}` immediately for any reference whose path traverses one of these names. This is a JS-runtime-specific guard against prototype-pollution attempts on plain object lookups; Python (`dict.get`) and Rust (`serde_json::Map::get`) are not vulnerable to the same attack and walk those segments like any other key. A user spec that legitimately names a schema `__proto__` will resolve in Python and Rust but return `{}` in TypeScript.

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
- On success: dict — fully resolved schema with all `$ref` pointers inlined (up to 16 levels deep), with all of the above listed keywords recursively walked, and with any **sibling keys** of a `$ref` merged over the resolved target (see below)

### `$ref` sibling keys are preserved

*Added in 0.12.0. Prior to it, all three SDKs discarded every key sitting
beside a `$ref`.*

A node of the form `{"$ref": "…", "description": "…", "x-sensitive": true}`
resolves to the referenced schema **merged with its siblings**, not to the
referenced schema alone:

1. Resolve the `$ref` target and walk it (this consumes one depth level).
2. Take every key of the original node except `$ref`, and walk those too, at
   the **same** depth as the node itself — following the reference already
   consumed a level, and the siblings are not a second hop.
3. Shallow-merge the walked siblings over the resolved target. **A sibling key
   wins** on conflict: it is the more specific, author-written value.

A `$ref` that fails to resolve still contributes its siblings, so
`{"$ref": "#/nope", "x-sensitive": true}` yields `{"x-sensitive": true}`
rather than `{}`.

!!! danger "Why this is a security property, not a fidelity nicety"
    apcore reads `x-sensitive` off the **resolved** schema to decide what to
    redact (apcore `PROTOCOL_SPEC` §10.6). This toolkit's `OpenAPIScanner`
    produces the `input_schema` / `output_schema` that the output writers then
    register into an apcore `Registry`. So a field whose OpenAPI document marks
    it sensitive **beside a `$ref`** had that marking stripped here, and reached
    apcore carrying nothing to redact on — the credential was then logged in
    plaintext.

    apcore closed the same hole in its own resolver in 0.31.0 (decision D-98,
    "`x-sensitive` dropped beside a `$ref` during schema resolution"). Fixing
    it there and not here leaves the leak intact for every schema this toolkit
    produces, because the marking is already gone by the time apcore sees it.

    The merge is therefore **not** limited to `x-` extension keys. Restricting
    it would fix the one symptom that has been noticed and leave `description`,
    `deprecated`, `title` and every future marker silently dropped.

**Note on OpenAPI 3.0.** OpenAPI 3.0's schema dialect says `$ref` siblings are
ignored, while OpenAPI 3.1 (JSON Schema 2020-12) says they apply. This toolkit
merges them for **both** versions, deliberately: the operation here is
"produce a faithful descriptor", not "validate under 3.0 semantics", and a
document that writes a key next to a `$ref` meant it. Honouring 3.0's rule
would mean deliberately discarding an `x-sensitive` an author wrote down.

### Properties
- async: false
- pure: true
- idempotent: true (calling twice on an already-resolved schema is a no-op)
- depth_limit: 16 — unchanged. Merging siblings adds no level of its own; every other traversal counts as it did before.
- note: This is the canonical behavior as implemented in all three SDKs.
