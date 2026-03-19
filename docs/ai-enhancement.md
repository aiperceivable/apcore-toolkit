# AI-Driven Metadata Enhancement

This document specifies how `apcore-toolkit` fills metadata gaps that static analysis cannot resolve, through a layered enhancement architecture.

## 1. Goal

The toolkit's primary mission is to make existing code "AI-Perceivable". While static analysis (regex, AST, type hints) is efficient, it often fails to:

- Generate meaningful `description` and `documentation` for legacy code with no docstrings.
- Create effective guidance (stored in `metadata.ai_guidance`) for complex error handling paths.
- Infer `input_schema` for functions using `*args` or `**kwargs`.
- Determine behavioral `annotations` (e.g., is this function destructive?) from code logic.

## 2. Architecture: Protocol + Built-in + Recommended

The enhancement system is designed in three layers:

```
┌─────────────────────────────────────────────────────┐
│  Enhancer (Protocol/Interface)                      │  ← Generic contract
├─────────────────────────────────────────────────────┤
│  AIEnhancer (built-in)                              │  ← Quick-start default
│  ─ calls any OpenAI-compatible local API            │
│  ─ good for prototyping and small projects          │
├─────────────────────────────────────────────────────┤
│  apcore-refinery (recommended)                      │  ← Production-grade
│  ─ specialized prompts per enhancement target       │
│  ─ flexible model selection (local or cloud)        │
│  ─ CI-friendly, cacheable, diffable                 │
├─────────────────────────────────────────────────────┤
│  Custom Enhancer                                    │  ← User-defined
│  ─ implement the Enhancer protocol                  │
│  ─ any logic: rules, heuristics, LLM, manual, etc. │
└─────────────────────────────────────────────────────┘
```

This layered design means:

- **Quick start**: Enable `AIEnhancer` with a local model to get immediate results.
- **Scale up**: Switch to `apcore-refinery` when you need higher quality or CI integration.
- **Full control**: Implement your own `Enhancer` for custom logic.

## 3. Enhancer Protocol

The toolkit exports a generic `Enhancer` protocol that any enhancement implementation must satisfy:

=== "Python"

    ```python
    from apcore_toolkit import Enhancer, ScannedModule

    # Enhancer is defined as:
    class Enhancer(Protocol):
        def enhance(self, modules: list[ScannedModule]) -> list[ScannedModule]:
            """Fill metadata gaps in scanned modules. Return the enriched list."""
            ...
    ```

=== "TypeScript"

    ```typescript
    import { Enhancer, ScannedModule } from "apcore-toolkit";

    // Enhancer is defined as:
    interface Enhancer {
      enhance(modules: ScannedModule[]): Promise<ScannedModule[]>;
    }
    ```

### Contract

- The enhancer receives modules **after** static scanning is complete.
- It **must not** remove or overwrite fields that already have non-default values.
- It **should** tag AI-generated fields with `metadata.x-generated-by` for auditability.
- It **should** add warnings to `ScannedModule.warnings` for low-confidence results.

## 4. Built-in: AIEnhancer

`AIEnhancer` is the toolkit's built-in implementation of the `Enhancer` protocol. It calls any OpenAI-compatible local API (Ollama, vLLM, LM Studio, etc.) to fill metadata gaps.

### Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `APCORE_AI_ENABLED` | Enable AI-based metadata enhancement. | `false` |
| `APCORE_AI_ENDPOINT` | URL of the OpenAI-compatible API. | `http://localhost:11434/v1` |
| `APCORE_AI_MODEL` | Model name to use. | `qwen:0.6b` |
| `APCORE_AI_THRESHOLD` | Confidence threshold for accepting AI-generated metadata (0.0–1.0). | `0.7` |
| `APCORE_AI_BATCH_SIZE` | Number of modules to enhance per API call. | `5` |
| `APCORE_AI_TIMEOUT` | Timeout in seconds for each API call. | `30` |

### Quick Start

1. **Install Ollama**: [ollama.com](https://ollama.com/)
2. **Pull a model**:
    ```bash
    ollama run qwen:0.6b
    ```
3. **Enable**:
    ```bash
    export APCORE_AI_ENABLED=true
    ```

### Usage

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

### Limitations

`AIEnhancer` is designed for quick prototyping. Be aware of these trade-offs:

| Aspect | Detail |
|--------|--------|
| **Model quality** | Small local models (sub-1B) may produce unreliable descriptions or mis-classify behavioral annotations. Use a larger model (7B+) for better results. |
| **Prompt tuning** | The built-in prompts are generic. For domain-specific codebases, consider `apcore-refinery` or a custom enhancer with tailored prompts. |
| **No caching** | Results are not cached between runs. Each scan re-invokes the model. |

!!! tip "Upgrade path"
    Start with `AIEnhancer` for development, then switch to `apcore-refinery` for CI/production. Both satisfy the `Enhancer` protocol, so no code changes are needed beyond swapping the implementation.

## 5. Recommended: apcore-refinery

For production-grade enhancement, we recommend **[apcore-refinery](https://github.com/aipartnerup/apcore-refinery)** — a dedicated tool with specialized prompts, flexible model selection, and CI integration.

```bash
# Scan with toolkit, then refine with apcore-refinery
apcore scan --framework django --output ./bindings
apcore-refinery enhance ./bindings --model <your-preferred-model>
```

apcore-refinery can be used as a **CLI tool** (shown above) or as a **library** implementing the `Enhancer` protocol for programmatic integration.

Benefits over the built-in `AIEnhancer`:

- **Model flexibility**: Use any model (local or cloud) appropriate for your codebase size and annotation complexity.
- **Better prompts**: Refinery maintains specialized prompt templates per enhancement target, iterated separately from toolkit releases.
- **CI integration**: Run refinement as a separate CI step, cache results, and diff changes across runs.
- **Confidence tuning**: Per-project threshold configuration without affecting toolkit behavior.

## 6. Enhancement Targets

The following fields are considered "enhancement targets" — fields that static analysis often leaves empty or at default values. Both `AIEnhancer` and `apcore-refinery` target these fields.

### 6.1 Description Generation

**When**: `description` is empty or auto-generated (e.g., copied from function name).

**Prompt strategy**: Send the function signature, docstring (if partial), and first 50 lines of the function body. Ask for a ≤200-character description following apcore's convention.

### 6.2 Documentation Generation

**When**: `documentation` is empty and the function has non-trivial logic (>10 lines).

**Prompt strategy**: Send the full function body. Ask for a ≤5000-character Markdown explanation covering purpose, parameters, return value, and error conditions.

### 6.3 Annotation Inference

**When**: All annotations are at their default values (no explicit annotation was set by the scanner).

This is where AI adds the most value — inferring behavioral semantics that static analysis cannot determine reliably:

| Annotation | What to look for |
|-----------|----------------------|
| `readonly` | No writes to databases, files, or external services |
| `destructive` | Deletes data, overwrites files, drops resources |
| `idempotent` | Same input always produces same output, safe to retry |
| `requires_approval` | Sends money, deletes accounts, modifies permissions |
| `open_world` | HTTP calls, file I/O, database queries, subprocess calls |
| `streaming` | Yields/iterates results incrementally |
| `cacheable` | Deterministic output for given inputs, no side effects |
| `cache_ttl` | Expected data freshness window (seconds, 0 = no expiry) |
| `cache_key_fields` | Subset of input fields that uniquely identify cached results |
| `paginated` | Returns partial result sets, supports continuation |
| `pagination_style` | `"cursor"`, `"offset"`, or `"page"` strategy |

**Acceptance rule**: Only apply an annotation if the confidence ≥ the configured threshold. Otherwise, leave as default and add a warning to `ScannedModule.warnings`.

!!! tip "Inspired by HARNESS.md"
    The annotation inference approach draws from [CLI-Anything's HARNESS.md](https://github.com/HKUDS/CLI-Anything) methodology, which catalogs undo/redo systems to determine destructiveness. For web frameworks, the equivalent is analyzing database transactions, file operations, and external API calls in the function body.

### 6.4 Schema Inference for Untyped Functions

**When**: `input_schema` is empty and the function uses `*args`, `**kwargs`, or `request` objects without type annotations.

**Prompt strategy**: Send the function body. Ask the model to infer parameter names, types, and whether they are required, based on how `kwargs` keys are accessed in the code.

**Output format**: A JSON Schema object that the toolkit merges into `ScannedModule.input_schema`.

## 7. Enhancement Workflow

```
Scanner.scan()
    │
    ▼
list[ScannedModule]          ← static metadata (may have gaps)
    │
    ▼
enhancer.enhance(modules)    ← AIEnhancer, apcore-refinery, or custom
    │
    ├─ For each module:
    │   1. Check which fields are missing/default
    │   2. Build targeted prompt for each gap
    │   3. Call AI backend / apply rules
    │   4. Parse response, check confidence
    │   5. Merge accepted enhancements
    │   6. Tag with x-generated-by
    │   7. Add warnings for rejected/low-confidence results
    │
    ▼
list[ScannedModule]          ← enriched metadata
    │
    ▼
Writer.write(modules)        ← output as YAML/Python/Registry
```

## 8. Auditability

All AI-generated fields — regardless of which enhancer produced them — must follow the metadata tagging convention:

```yaml
metadata:
  x-generated-by: <enhancer-name>    # e.g., "ai-enhancer", "apcore-refinery", "custom"
  x-ai-confidence:
    description: 0.92
    annotations.destructive: 0.85
    annotations.readonly: 0.45       # below threshold, not applied
```

Fields below the configured confidence threshold should **not** be applied. Instead, a warning should be added to `ScannedModule.warnings`:

```
"Low confidence (0.45) for annotations.readonly — skipped. Review manually."
```

## 9. Security and Privacy

- **Auditability**: All AI-generated fields are tagged with `x-generated-by` for human review.
- **Local by default**: `AIEnhancer` calls a local API endpoint. Source code never leaves the machine unless explicitly configured otherwise.
- **Opt-in only**: Enhancement is disabled by default (`APCORE_AI_ENABLED=false`). Projects explicitly choose when and how to enhance.
- **Graceful degradation**: If the AI endpoint is unreachable, the enhancer logs a warning and returns modules unchanged.
