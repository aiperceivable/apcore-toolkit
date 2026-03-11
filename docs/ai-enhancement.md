# AI-Driven Metadata Enhancement

This document specifies how `apcore-toolkit` uses Small Language Models (SLMs) to fill metadata gaps that static analysis cannot resolve.

## 1. Goal

The toolkit's primary mission is to make existing code "AI-Perceivable". While static analysis (regex, AST, type hints) is efficient, it often fails to:

- Generate meaningful `description` and `documentation` for legacy code with no docstrings.
- Create effective `ai_guidance` for complex error handling paths.
- Infer `input_schema` for functions using `*args` or `**kwargs`.
- Determine behavioral `annotations` (e.g., is this function destructive?) from code logic.

A local SLM fills these gaps with high speed, zero cost, and no data leakage.

## 2. Architecture

To keep `apcore-toolkit` lightweight, we **do not** bundle model weights. Instead, we call an OpenAI-compatible local API provider.

### Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `APCORE_AI_ENABLED` | Enable SLM-based metadata enhancement. | `false` |
| `APCORE_AI_ENDPOINT` | URL of the OpenAI-compatible API. | `http://localhost:11434/v1` |
| `APCORE_AI_MODEL` | Model name (e.g., `qwen:0.6b`). | `qwen:0.6b` |
| `APCORE_AI_THRESHOLD` | Confidence threshold for accepting AI-generated metadata (0.0–1.0). | `0.7` |
| `APCORE_AI_BATCH_SIZE` | Number of modules to enhance per API call. | `5` |
| `APCORE_AI_TIMEOUT` | Timeout in seconds for each API call. | `30` |

### Recommended Setup (Ollama)

1. **Install Ollama**: [ollama.com](https://ollama.com/)
2. **Pull a model**:
    ```bash
    ollama run qwen:0.6b
    ```
3. **Configure**:
    ```bash
    export APCORE_AI_ENABLED=true
    ```

## 3. Enhancement Targets

The enhancer operates on `ScannedModule` instances **after** static scanning is complete. It only fills fields that are missing or below the confidence threshold.

### 3.1 Description Generation

**When**: `description` is empty or auto-generated (e.g., copied from function name).

**Prompt strategy**: Send the function signature, docstring (if partial), and first 50 lines of the function body. Ask for a ≤200-character description following apcore's convention.

**Audit tag**: `x-generated-by: slm` in `metadata`.

### 3.2 Documentation Generation

**When**: `documentation` is empty and the function has non-trivial logic (>10 lines).

**Prompt strategy**: Send the full function body. Ask for a ≤5000-character Markdown explanation covering purpose, parameters, return value, and error conditions.

### 3.3 Annotation Inference

**When**: All annotations are at their default values (no explicit annotation was set by the scanner).

This is where the SLM adds the most value — inferring behavioral semantics that static analysis cannot determine reliably.

**Prompt strategy**: Send the function body and ask the model to classify each annotation with a confidence score:

| Annotation | What the SLM looks for |
|-----------|----------------------|
| `readonly` | No writes to databases, files, or external services |
| `destructive` | Deletes data, overwrites files, drops resources |
| `idempotent` | Same input always produces same output, safe to retry |
| `requires_approval` | Sends money, deletes accounts, modifies permissions |
| `open_world` | HTTP calls, file I/O, database queries, subprocess calls |
| `streaming` | Yields/iterates results incrementally |

**Acceptance rule**: Only apply an annotation if the SLM's confidence ≥ `APCORE_AI_THRESHOLD`. Otherwise, leave as default and add a warning to `ScannedModule.warnings`.

!!! tip "Inspired by HARNESS.md"
    The annotation inference approach draws from [CLI-Anything's HARNESS.md](https://github.com/HKUDS/CLI-Anything) methodology, which catalogs undo/redo systems to determine destructiveness. For web frameworks, the equivalent is analyzing database transactions, file operations, and external API calls in the function body.

### 3.4 Schema Inference for Untyped Functions

**When**: `input_schema` is empty and the function uses `*args`, `**kwargs`, or `request` objects without type annotations.

**Prompt strategy**: Send the function body. Ask the model to infer parameter names, types, and whether they are required, based on how `kwargs` keys are accessed in the code.

**Output format**: A JSON Schema object that the toolkit merges into `ScannedModule.input_schema`.

## 4. Enhancement Workflow

```
Scanner.scan()
    │
    ▼
list[ScannedModule]          ← static metadata (may have gaps)
    │
    ▼
AIEnhancer.enhance(modules)  ← fills gaps using SLM
    │
    ├─ For each module:
    │   1. Check which fields are missing/default
    │   2. Build targeted prompt for each gap
    │   3. Call SLM API
    │   4. Parse response, check confidence
    │   5. Merge accepted enhancements
    │   6. Tag with x-generated-by: slm
    │   7. Add warnings for rejected/low-confidence results
    │
    ▼
list[ScannedModule]          ← enriched metadata
    │
    ▼
Writer.write(modules)        ← output as YAML/Python/Registry
```

### Integration with BaseScanner

The enhancer is **not** called automatically by `BaseScanner.scan()`. Framework adapters opt in explicitly:

=== "Python"

    ```python
    from apcore_toolkit import AIEnhancer

    scanner = MyFrameworkScanner()
    modules = scanner.scan()

    if AIEnhancer.is_enabled():
        enhancer = AIEnhancer()
        modules = enhancer.enhance(modules)

    writer.write(modules, output_dir="./bindings")
    ```

=== "TypeScript"

    ```typescript
    import { AIEnhancer } from "apcore-toolkit";

    const scanner = new MyFrameworkScanner();
    let modules = scanner.scan();

    if (AIEnhancer.isEnabled()) {
      const enhancer = new AIEnhancer();
      modules = await enhancer.enhance(modules);
    }

    writer.write(modules, { outputDir: "./bindings" });
    ```

## 5. Confidence Scoring

Each AI-generated field includes a confidence score (0.0–1.0) stored in `metadata`:

```yaml
metadata:
  x-generated-by: slm
  x-ai-confidence:
    description: 0.92
    annotations.destructive: 0.85
    annotations.readonly: 0.45    # below threshold, not applied
```

Fields below `APCORE_AI_THRESHOLD` are **not** applied to the module. Instead, a warning is added:

```
"Low confidence (0.45) for annotations.readonly — skipped. Review manually."
```

## 6. Security and Privacy

- **No data leakage**: The model runs locally. Source code never leaves the machine.
- **Auditability**: All AI-generated fields are tagged with `x-generated-by: slm` for human review.
- **Opt-in only**: Disabled by default (`APCORE_AI_ENABLED=false`).
- **Graceful degradation**: If the SLM endpoint is unreachable, the enhancer logs a warning and returns modules unchanged.
