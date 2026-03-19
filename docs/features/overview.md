# Features Overview

`apcore-toolkit` is a collection of framework-agnostic utilities designed to help you extract, refine, and export metadata from your existing codebase, making it "AI-Perceivable". Available for both **Python** and **TypeScript**.

## Core Capabilities

| Feature | Description |
|---------|-------------|
| **[Smart Scanning](scanning.md)** | Abstract base classes and utilities for framework-specific scanners, with a 5-phase ability extraction methodology. |
| **[OpenAPI Integration](openapi.md)** | Extract JSON Schemas directly from OpenAPI operation objects. |
| **[Schema Utilities](pydantic.md)** | Flatten complex models (Pydantic / Zod) for easier AI interaction. |
| **[Output Writers](output-writers.md)** | Export metadata to YAML bindings, source code wrappers, or direct Registry registration — with optional output verification. |
| **[Formatting](formatting.md)** | Convert data structures into beautiful, human-readable Markdown. |
| **[AI Enhancement](../ai-enhancement.md)** | Pluggable `Enhancer` protocol with built-in `AIEnhancer` for local SLMs; [apcore-refinery](https://github.com/aipartnerup/apcore-refinery) recommended for production. |

## Design Philosophy

- **Framework Agnostic**: The core logic has no dependency on specific web frameworks (Django, Flask, FastAPI).
- **Separation of Concerns**: Scanning (extraction), Schema Utilities (refinement), and Writers (export) are kept distinct.
- **Developer First**: Focuses on automating the tedious tasks of writing `apcore.yaml` or `@module` decorators.
- **AI-Native**: Built with the assumption that the ultimate consumer of this metadata is a Large Language Model (LLM) or AI agent.
- **Dual-Language Parity**: Every feature is implementable in both Python and TypeScript.

## Scope

For a detailed definition of what the toolkit does and does not do, see the [Scope & Boundaries](../scope.md) document.
