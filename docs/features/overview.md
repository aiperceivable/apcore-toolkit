# Features Overview

`apcore-toolkit` is a collection of framework-agnostic utilities designed to help you extract, refine, and export metadata from your existing codebase, making it "AI-Perceivable". Available for **Python**, **TypeScript**, and **Rust**.

## Core Capabilities

| Feature | Description |
|---------|-------------|
| **[Smart Scanning](scanning.md)** | Abstract base classes and utilities for framework-specific scanners, with a 5-phase ability extraction methodology. |
| **[OpenAPI Integration](openapi.md)** | Extract JSON Schemas directly from OpenAPI operation objects. |
| **[Schema Utilities](pydantic.md)** | Flatten complex models (Pydantic / Zod) for easier AI interaction. |
| **[Output Writers](output-writers.md)** | Export metadata to YAML bindings, source code wrappers, or direct Registry registration — with optional output verification. |
| **[Binding Loader](binding-loader.md)** | Parse `.binding.yaml` files back into `ScannedModule` objects — the read-path counterpart to `YAMLWriter`. Supports strict and loose modes for verification, merging, and round-trip workflows. |
| **[Formatting](formatting.md)** | Convert data structures into Markdown and enrich JSON Schema descriptions from docstrings. |
| **[AI Enhancement](../ai-enhancement.md)** | Pluggable `Enhancer` protocol with built-in `AIEnhancer` for local SLMs; [apcore-refinery](https://github.com/aiperceivable/apcore-refinery) recommended for production. |
| **[Display Overlay](display-overlay.md)** | Sparse `binding.yaml` overlay that resolves surface-facing alias, description, guidance, and tags into `metadata["display"]` for CLI, MCP, and A2A surfaces (§5.13). |
| **[Convention Scanning](convention-scanning.md)** | Scan a `commands/` directory of plain Python files for public functions, inferring schemas from type annotations -- zero decorators, zero imports (§5.14). |

## Design Philosophy

- **Framework Agnostic**: The core logic has no dependency on specific web frameworks (Django, Flask, FastAPI).
- **Separation of Concerns**: Scanning (extraction), Schema Utilities (refinement), and Writers (export) are kept distinct.
- **Developer First**: Focuses on automating the tedious tasks of writing `apcore.yaml` or `@module` decorators.
- **AI-Native**: Built with the assumption that the ultimate consumer of this metadata is a Large Language Model (LLM) or AI agent.
- **Cross-Language Parity**: Every core feature has matching implementations in Python, TypeScript, and Rust with idiomatic-per-language APIs and wire-format compatibility.

## Scope

For a detailed definition of what the toolkit does and does not do, see the [Scope & Boundaries](../scope.md) document.
