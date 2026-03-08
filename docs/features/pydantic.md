# Schema Utilities

Modern web frameworks use structured models (Pydantic in Python, Zod/interfaces in TypeScript) for request and response validation. `apcore-toolkit` provides utilities to bridge the gap between structured models and the flat interface often required by AI tools.

## Model Flattening

=== "Python"

    The `flatten_pydantic_params()` function wraps a Python function, converting its Pydantic model parameters into flat, scalar keyword arguments.

=== "TypeScript"

    The `flattenParams()` function wraps a TypeScript function, converting its structured input types (Zod schemas or interfaces) into flat, scalar keyword arguments.

### Why Flatten?

AI agents and protocols like MCP (Model Context Protocol) interact best with flat schemas. A function like `create_task(body: TaskCreate)` is difficult for an AI to call because it needs to understand the `TaskCreate` structure. By flattening, the AI sees `create_task(title: str, description: str, ...)` directly.

### Example

=== "Python"

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

=== "TypeScript"

    ```typescript
    import { z } from "zod";
    import { flattenParams } from "@anthropic/apcore-toolkit";

    const UserCreate = z.object({
      username: z.string(),
      email: z.string(),
    });

    function createUser(body: z.infer<typeof UserCreate>) {
      return `Created ${body.username}`;
    }

    // Wrap the function
    const flatCreateUser = flattenParams(createUser, UserCreate);

    // Now it can be called with flat kwargs
    const result = flatCreateUser({ username: "jdoe", email: "joe@example.com" });
    ```

## Metadata Preservation

When flattening, `apcore-toolkit` preserves:

- **Type Hints**: Original field types are maintained in the wrapper's signature.
- **Field Metadata**: Descriptions, examples, and schema metadata from the model are preserved on the wrapper's parameters.

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
    import { resolveTarget } from "@anthropic/apcore-toolkit";

    // Dynamically load a callable
    const viewFunc = await resolveTarget("myapp/api/v1/users:createUser");
    // viewFunc is now the actual function from myapp
    ```
