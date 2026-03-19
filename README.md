<div align="center">
  <img src="./apcore-toolkit-logo.svg" alt="apcore logo" width="200"/>
</div>

# apcore-toolkit

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Python Version](https://img.shields.io/badge/python-3.11%2B-blue)](https://github.com/aipartnerup/apcore-toolkit-python)
[![Python SDK](https://img.shields.io/badge/python_sdk-0.3.0-green)](https://github.com/aipartnerup/apcore-toolkit-python)
[![TypeScript Version](https://img.shields.io/badge/typescript-5.0%2B-blue)](https://github.com/aipartnerup/apcore-toolkit-typescript)
[![TypeScript SDK](https://img.shields.io/badge/typescript_sdk-0.3.0-green)](https://github.com/aipartnerup/apcore-toolkit-typescript)

**apcore-toolkit** is a shared scanner, schema extraction, and output toolkit for the [apcore](https://github.com/aipartnerup/apcore-python) ecosystem. It provides framework-agnostic logic to extract metadata from existing code and make it "AI-Perceivable".

Available in:
- [🐍 Python](https://github.com/aipartnerup/apcore-toolkit-python)
- [📘 TypeScript](https://github.com/aipartnerup/apcore-toolkit-typescript)

---

## Key Features

- **🚀 Multi-Language Support**: Implementation available for both [🐍 Python](https://github.com/aipartnerup/apcore-toolkit-python) and [📘 TypeScript](https://github.com/aipartnerup/apcore-toolkit-typescript).
- **🔍 Smart Scanning**: Abstract base classes for framework scanners with filtering and deduplication.
- **📄 Output Generation**: Writers for YAML bindings, language-specific wrappers, and direct Registry registration.
- **🛠️ Schema Utilities**: Tools for Pydantic/Zod model flattening and OpenAPI schema extraction.
- **🤖 AI Enhancement**: Built-in `AIEnhancer` with local SLM support, pluggable `Enhancer` protocol, and [apcore-refinery](https://github.com/aipartnerup/apcore-refinery) for production use.
- **📝 Markdown Formatting**: Convert arbitrary data structures to structured Markdown.

---

## Installation

=== "🐍 Python"

    ```bash
    pip install apcore-toolkit
    ```
    Requires Python 3.11+ and apcore 0.13.0+.

=== "📘 TypeScript"

    ```bash
    npm install apcore-toolkit
    ```
    Requires Node.js 18+ and apcore 0.13.0+.

---

## Quick Example

=== "🐍 Python"

    ```python
    from apcore_toolkit import BaseScanner, ScannedModule

    class MyScanner(BaseScanner):
        def scan(self, **kwargs):
            return [
                ScannedModule(
                    module_id="users.get_user",
                    description="Get a user by ID",
                    input_schema={"type": "object", "properties": {"id": {"type": "integer"}}, "required": ["id"]},
                    output_schema={"type": "object", "properties": {"name": {"type": "string"}}},
                    target="myapp.views:get_user",
                )
            ]

    scanner = MyScanner()
    modules = scanner.scan()
    ```

=== "📘 TypeScript"

    ```typescript
    import { BaseScanner, ScannedModule, createScannedModule } from "apcore-toolkit";

    class MyScanner extends BaseScanner {
      scan(): ScannedModule[] {
        return [
          createScannedModule({
            moduleId: "users.get_user",
            description: "Get a user by ID",
            inputSchema: { type: "object", properties: { id: { type: "integer" } }, required: ["id"] },
            outputSchema: { type: "object", properties: { name: { type: "string" } } },
            target: "myapp/views:get_user",
          }),
        ];
      }
    }

    const scanner = new MyScanner();
    const modules = scanner.scan();
    ```

---

## Core Modules

| Module | Description |
|--------|-------------|
| `ScannedModule` | Canonical dataclass representing a scanned endpoint |
| `BaseScanner` | Abstract base class for framework scanners |
| `YAMLWriter` | Generates `.binding.yaml` files for `apcore.BindingLoader` |
| `PythonWriter` | Generates `@module`-decorated Python wrapper files |
| `TypeScriptWriter` | Generates `@module`-decorated TypeScript wrapper files |
| `RegistryWriter` | Registers modules directly into an `apcore.Registry` |
| `HTTPProxyRegistryWriter` | Registers HTTP proxy modules that forward requests to a running API (Python only) |
| `to_markdown` | Converts arbitrary dicts to Markdown with depth control and table heuristics |
| `enrich_schema_descriptions` | Merges docstring parameter descriptions into JSON Schema properties |
| `Enhancer` | Protocol/interface for pluggable metadata enrichment (see [AI Enhancement](docs/ai-enhancement.md)) |
| `AIEnhancer` | Built-in `Enhancer` implementation using OpenAI-compatible local APIs |
| `WriteResult` | Dataclass representing the outcome of a writer operation |
| `WriteError` | Exception raised when a writer fails due to I/O or other errors |
| `Verifier` / `VerifyResult` | Protocol and result type for pluggable output verification |

---

## Documentation

- **[Getting Started Guide](docs/getting-started.md)** — Installation and core usage
- **[Features Overview](docs/features/overview.md)** — Detailed look at toolkit capabilities
- **[AI Enhancement Guide](docs/ai-enhancement.md)** — Enhancer protocol, built-in AIEnhancer, and apcore-refinery
- **[Changelog](CHANGELOG.md)**

---

## License

Apache-2.0

