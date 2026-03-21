# Scope & Boundaries

This document defines what `apcore-toolkit` **does** and **does not do**, to guide contributors and downstream projects.

## What apcore-toolkit Is

A **shared utility library** for framework-specific adapters (e.g., `django-apcore`, `flask-apcore`, `fastapi-apcore`). It extracts duplicated, framework-agnostic logic into a single package available in both Python and TypeScript.

**Core responsibilities:**

| Responsibility | Description |
|---------------|-------------|
| **Scanning abstractions** | `BaseScanner` ABC, filtering, deduplication, behavioral inference |
| **Schema extraction** | OpenAPI parsing, Pydantic/Zod model flattening |
| **Output generation** | Writers for YAML bindings, source code wrappers, and direct Registry registration |
| **Output verification** | Pluggable `Verifier` protocol for validating written artifacts are well-formed |
| **AI metadata enhancement** | `Enhancer` protocol with built-in `AIEnhancer`; enrichment for missing descriptions, annotations, and schemas |
| **Formatting** | Data-to-Markdown conversion for AI-perceivable documentation, schema description enrichment |

## What apcore-toolkit Is NOT

### Not a CLI Tool Scanner

Scanning CLI tools (parsing `--help`, man pages, subcommand trees) is a **fundamentally different problem domain** from scanning web framework code:

| Dimension | Web Framework Scanning (toolkit) | CLI Tool Scanning |
|-----------|----------------------------------|-------------------|
| **Input source** | Python/TS source code with importable routes | Binary executables, shell commands |
| **Discovery** | Import framework objects, inspect decorators | Parse `--help` text, man pages |
| **Schema inference** | Type annotations, Pydantic models, OpenAPI specs | Flag/argument text parsing, heuristics |
| **Dependencies** | Can `import` the target code | Target may be a compiled binary |
| **Complexity** | BaseScanner + a few hundred lines per adapter | Requires per-tool adapters with deep domain knowledge |
| **Users** | Framework adapter developers | Automation engineers, AI agent developers |

CLI tool scanning should live in a **separate project** that:

- Consumes `apcore-toolkit`'s `BaseScanner` and Writer classes as building blocks.
- Owns the CLI-specific parsing, heuristics, and per-tool adapter logic.
- Can grow independently without bloating the toolkit.

!!! info "Reference: CLI-Anything"
    The [CLI-Anything](https://github.com/HKUDS/CLI-Anything) project demonstrates the scale of CLI tool adaptation: 10 software adapters, 30K+ lines of production code, and a 622-line methodology SOP (HARNESS.md). This validates the decision to keep CLI scanning as a separate project.

### Not apcore Itself

`apcore-toolkit` does **not** modify or extend:

- The Module protocol, ModuleAnnotations, or Registry (defined by `apcore`)
- The Executor pipeline, ACL, or middleware system
- Schema validation or multi-format export (already in `apcore-python`)
- Parameter constraints (min/max/default) — these are already supported via Pydantic `Field()` in `apcore-python`

The toolkit's job is to **extract** metadata from existing code and **produce** artifacts that apcore can consume. It does not change how apcore consumes them.

### Not a Workflow Engine

Orchestration, token management, and multi-agent coordination belong to `apflow`, not the toolkit.

## Relationship to Other Projects

```
┌─────────────────────────────────────────────────────────┐
│                    apcore (spec + SDK)                   │
│  Module protocol · Registry · Executor · Schema system  │
└──────────────────────────┬──────────────────────────────┘
                           │ consumes artifacts from
          ┌────────────────┼────────────────┐
          │                │                │
   ┌──────┴──────┐  ┌─────┴──────┐  ┌──────┴──────┐
   │ django-apcore│  │flask-apcore│  │  ...        │
   │ (adapter)   │  │ (adapter)  │  │ (adapter)   │
   └──────┬──────┘  └─────┬──────┘  └──────┬──────┘
          │                │                │
          └────────────────┼────────────────┘
                           │ all use shared logic from
                ┌──────────┴──────────┐
                │   apcore-toolkit    │
                │  BaseScanner · Writers │
                │  Schema utils · AI  │
                └─────────────────────┘
```

## Guiding Principles

1. **Stay framework-agnostic.** No imports from Django, Flask, FastAPI, Express, or NestJS.
2. **Stay lightweight.** The toolkit is a library, not an application. No CLI entry points, no daemon processes.
3. **Produce, don't consume.** Generate artifacts (YAML, Python, Registry entries) for apcore. Don't run the Executor or manage module lifecycle.
4. **Dual-language parity.** Every feature documented here must be implementable in both Python and TypeScript. If a feature is language-specific, it belongs in the adapter, not the toolkit.
