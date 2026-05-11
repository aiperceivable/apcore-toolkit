<div align="center">
  <img src="./apcore-toolkit-logo.svg" alt="apcore logo" width="200"/>
</div>

# apcore-toolkit

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Python Version](https://img.shields.io/badge/python-3.11%2B-blue)](https://github.com/aiperceivable/apcore-toolkit-python)
[![Python SDK](https://img.shields.io/badge/python_sdk-0.7.0-green)](https://github.com/aiperceivable/apcore-toolkit-python)
[![TypeScript Version](https://img.shields.io/badge/typescript-5.0%2B-blue)](https://github.com/aiperceivable/apcore-toolkit-typescript)
[![TypeScript SDK](https://img.shields.io/badge/typescript_sdk-0.7.0-green)](https://github.com/aiperceivable/apcore-toolkit-typescript)
[![Rust Version](https://img.shields.io/badge/rust-1.70%2B-blue)](https://github.com/aiperceivable/apcore-toolkit-rust)
[![Rust SDK](https://img.shields.io/badge/rust_sdk-0.7.0-green)](https://github.com/aiperceivable/apcore-toolkit-rust)
[![apcore](https://img.shields.io/badge/apcore-0.21.0%2B-orange)](https://github.com/aiperceivable/apcore-python)

**apcore-toolkit** is a shared scanner, schema extraction, and output toolkit for the [apcore](https://github.com/aiperceivable/apcore-python) ecosystem. It provides framework-agnostic logic to extract metadata from existing code and make it "AI-Perceivable".

Available in:
- [🐍 Python](https://github.com/aiperceivable/apcore-toolkit-python)
- [📘 TypeScript](https://github.com/aiperceivable/apcore-toolkit-typescript)
- [🦀 Rust](https://github.com/aiperceivable/apcore-toolkit-rust)

---

## Key Features

- **🚀 Multi-Language Support**: Implementation available for [🐍 Python](https://github.com/aiperceivable/apcore-toolkit-python), [📘 TypeScript](https://github.com/aiperceivable/apcore-toolkit-typescript), and [🦀 Rust](https://github.com/aiperceivable/apcore-toolkit-rust).
- **🔍 Smart Scanning**: Abstract base classes for framework scanners with filtering and deduplication.
- **📄 Output Generation**: Writers for YAML bindings, language-specific wrappers, and direct Registry registration.
- **🛠️ Schema Utilities**: Tools for Pydantic/Zod model flattening and OpenAPI schema extraction.
- **🤖 AI Enhancement**: Built-in `AIEnhancer` with local SLM support, pluggable `Enhancer` protocol, and [apcore-refinery](https://github.com/aiperceivable/apcore-refinery) for production use.
- **📝 Surface Formatting**: Convert structured data into LLM-ready Markdown, SKILL.md, CLI table-rows, or canonical JSON.
- **📊 Byte-Equivalent Tabular Formatters** _(v0.7.0)_: `format_csv` / `format_jsonl` produce identical bytes across Python / TypeScript / Rust, asserted via a shared conformance corpus. Replaces per-SDK reimplementations that had diverged on header derivation, line endings, and nested-value serialization.

---

## Installation

=== "🐍 Python"

    ```bash
    pip install apcore-toolkit
    ```
    Requires Python 3.11+ and apcore 0.21.0+.

=== "📘 TypeScript"

    ```bash
    npm install apcore-toolkit
    ```
    Requires Node.js 18+ and apcore-js 0.21.0+.

=== "🦀 Rust"

    ```toml
    [dependencies]
    apcore-toolkit = { git = "https://github.com/aiperceivable/apcore-toolkit-rust" }
    ```
    Requires Rust 1.70+ and apcore 0.21.0+.

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

=== "🦀 Rust"

    ```rust
    use apcore_toolkit::{Scanner, ScannedModule};
    use async_trait::async_trait;
    use serde_json::json;

    struct MyScanner;

    #[async_trait]
    impl Scanner<()> for MyScanner {
        async fn scan(&self, _app: &()) -> Vec<ScannedModule> {
            vec![
                ScannedModule::new(
                    "users.get_user".into(),
                    "Get a user by ID".into(),
                    json!({"type": "object", "properties": {"id": {"type": "integer"}}, "required": ["id"]}),
                    json!({"type": "object", "properties": {"name": {"type": "string"}}}),
                    vec!["users".into()],
                    "myapp:get_user".into(),
                )
            ]
        }
        fn source_name(&self) -> &str { "my-framework" }
    }

    #[tokio::main]
    async fn main() {
        let scanner = MyScanner;
        let modules = scanner.scan(&()).await;
    }
    ```

---

## Byte-Equivalent Tabular Formats (v0.7.0)

`format_csv` and `format_jsonl` emit identical bytes across Python / TypeScript / Rust for the same input. Consumers (apcore-cli, apcore-mcp, apcore-a2a, downstream CLIs) MUST delegate to these formatters rather than reimplementing — a shared conformance corpus at `conformance/fixtures/format_csv.json` and `format_jsonl.json` is run by every SDK to assert byte-identity.

=== "🐍 Python"

    ```python
    from apcore_toolkit import format_csv, format_jsonl

    rows = [
        {"sn": 1, "title": "First", "score": 78},
        {"sn": 2, "title": "Second", "score": 82, "description": "later-only field"},
    ]
    print(format_csv(rows))
    # sn,title,score,description\r\n
    # 1,First,78,\r\n
    # 2,Second,82,later-only field\r\n
    ```

=== "📘 TypeScript"

    ```typescript
    import { formatCsv, formatJsonl } from "apcore-toolkit";

    const rows = [
      { sn: 1, title: "First", score: 78 },
      { sn: 2, title: "Second", score: 82, description: "later-only field" },
    ];
    process.stdout.write(formatCsv(rows));
    ```

=== "🦀 Rust"

    ```rust
    use apcore_toolkit::format_csv;
    use serde_json::{json, Map, Value};

    let rows: Vec<Map<String, Value>> = vec![
        json!({"sn": 1, "title": "First", "score": 78}).as_object().unwrap().clone(),
        json!({"sn": 2, "title": "Second", "score": 82, "description": "later-only field"})
            .as_object().unwrap().clone(),
    ];
    print!("{}", format_csv(&rows, false));
    ```

Key contract guarantees:

- **CSV header = union of keys across all rows** (insertion-order). Rows missing a key emit an empty cell — no silent data loss when rows are heterogeneous.
- **Nested values = canonical compact JSON** in the cell (matching `JSON.stringify`). No Python repr `{'k': 'v'}`, no `[object Object]`.
- **RFC 4180 CRLF** terminator for CSV; LF for JSONL (JSONL convention).
- **Whole-number floats** render as integers (matching JS `JSON.stringify(1.0) === "1"`); NaN/Infinity → empty cell / `null`.
- **Insertion-order keys** in JSONL output (Python `dict`, JS object order, Rust `serde_json` `preserve_order`).
- **Optional UTF-8 BOM** on CSV via `bom=True` for Excel-locale users; default off for pipeline consumers.

See [`docs/features/formatting.md`](docs/features/formatting.md) § Tabular Formats for the full contract and `conformance/fixtures/` for the shared test corpus.

> **YAML byte-equivalence is deferred.** Each idiomatic YAML library (PyYAML, js-yaml, serde_yaml_ng) emits different forms even for identical input; a custom canonical emitter is a planned follow-up. YAML output today is SDK-native and may differ across languages.

---

## Core Modules

| Module | Description |
|--------|-------------|
| `ScannedModule` | Canonical dataclass representing a scanned endpoint |
| `BaseScanner` | Abstract base class for framework scanners |
| `YAMLWriter` | Generates `.binding.yaml` files for `apcore.BindingLoader` |
| `BindingLoader` | Parses `.binding.yaml` files back into `ScannedModule` objects (pure-data inverse of `YAMLWriter`); loose/strict modes; round-trip with `display`, `annotations`, `metadata` |
| `PythonWriter` | Generates `@module`-decorated Python wrapper files |
| `TypeScriptWriter` | Generates `@module`-decorated TypeScript wrapper files |
| `RegistryWriter` | Registers modules directly into an `apcore.Registry` |
| `HTTPProxyRegistryWriter` | Registers HTTP proxy modules that forward requests to a running API (Python only) |
| `to_markdown` | Converts arbitrary dicts to Markdown with depth control and table heuristics |
| `format_csv` _(v0.7.0)_ | Byte-equivalent RFC 4180 CSV emitter. Header = union of keys across all rows; canonical compact JSON for nested cells; CRLF terminator; optional UTF-8 BOM. |
| `format_jsonl` _(v0.7.0)_ | Byte-equivalent JSON Lines emitter. Canonical compact JSON per row, LF terminator, insertion-order preserved, NaN/Inf → null. |
| `enrich_schema_descriptions` | Merges docstring parameter descriptions into JSON Schema properties |
| `Enhancer` | Protocol/interface for pluggable metadata enrichment (see [AI Enhancement](docs/ai-enhancement.md)) |
| `AIEnhancer` | Built-in `Enhancer` implementation using OpenAI-compatible local APIs |
| `WriteResult` | Dataclass representing the outcome of a writer operation |
| `WriteError` | Exception raised when a writer fails due to I/O or other errors |
| `Verifier` / `VerifyResult` | Protocol and result type for pluggable output verification |
| `DisplayResolver` | Sparse binding.yaml display overlay — resolves surface-facing alias, description, guidance, tags into `metadata["display"]` (§5.13) |
| `ConventionScanner` | Scans a `commands/` directory of plain Python files for public functions and converts them to `ScannedModule` instances with schema inferred from type annotations (§5.14) |

---

## Documentation

- **[Getting Started Guide](docs/getting-started.md)** — Installation and core usage
- **[Features Overview](docs/features/overview.md)** — Detailed look at toolkit capabilities
- **[AI Enhancement Guide](docs/ai-enhancement.md)** — Enhancer protocol, built-in AIEnhancer, and apcore-refinery
- **[Changelog](CHANGELOG.md)**

---

## License

Apache-2.0

