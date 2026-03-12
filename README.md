<div align="center">
  <img src="./apcore-toolkit-logo.svg" alt="apcore logo" width="200"/>
</div>

# apcore-toolkit

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Python Version](https://img.shields.io/badge/python-3.11%2B-blue)](https://github.com/aipartnerup/apcore-toolkit-python)

**apcore-toolkit** is a shared scanner, schema extraction, and output toolkit for the [apcore](https://github.com/aipartnerup/apcore-python) ecosystem. It provides framework-agnostic logic to extract metadata from existing code and make it "AI-Perceivable".

---

## Key Features

- **🔍 Smart Scanning**: Abstract base classes for framework scanners with filtering and deduplication.
- **📄 Output Generation**: Writers for YAML bindings, Python wrappers, and direct Registry registration.
- **🛠️ Schema Utilities**: Tools for Pydantic model flattening and OpenAPI schema extraction.
- **🤖 AI Enhancement**: Metadata enrichment using local SLMs (Small Language Models).
- **📝 Markdown Formatting**: Convert arbitrary data structures to structured Markdown.

---

## Installation

**Python**
```bash
pip install apcore-toolkit
```
Requires Python 3.11+ and apcore 0.13.0+.

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
| `to_markdown` | Converts arbitrary dicts to Markdown with depth control and table heuristics |
| `AIEnhancer` | SLM-based metadata enrichment for missing descriptions and schemas |
| `WriteResult` | Dataclass representing the outcome of a writer operation |
| `WriteError` | Exception raised when a writer fails due to I/O or other errors |
| `Verifier` / `VerifyResult` | Protocol and result type for pluggable output verification |

---

## Documentation

- **[Getting Started Guide](docs/getting-started.md)** — Installation and core usage
- **[Features Overview](docs/features/overview.md)** — Detailed look at toolkit capabilities
- **[AI Enhancement Guide](docs/ai-enhancement.md)** — Metadata enrichment strategy
- **[Changelog](docs/changelog.md)**

---

## License

Apache-2.0

