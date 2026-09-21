---
description: "Cross-SDK compatibility and conformance contracts for apcore-toolkit."
status: reference
---

# Cross-SDK Conformance

`apcore-toolkit` maintains Python, TypeScript, and Rust SDKs with idiomatic
APIs and shared contracts where downstream consumers need portable results.

## Byte-equivalent contracts

The shared conformance fixtures currently cover:

| Contract | Fixture | Purpose |
|---|---|---|
| OpenAPI scanning | `conformance/fixtures/openapi_scan.json` | One module per operation and stable module-ID derivation |
| TUI view model | `conformance/fixtures/view_model.json` | Columns, rows, grouping, filtering, sorting, and tone metadata |
| Device authorization flow | `conformance/fixtures/device_auth.json` | RFC 8628 polling state machine, token lifecycle, provider-compatibility normalisation, and discovery URL construction |
| Binding pattern | `conformance/fixtures/binding_pattern.json` | `BindingLoader.load` name-matching syntax, rejected patterns, and composition with `recursive` |
| Display overlay | `conformance/fixtures/display_resolve.json` | Portable display metadata resolution |
| CSV / JSONL | `conformance/fixtures/format_csv.json`, `format_jsonl.json` | Canonical text output and line-ending rules |
| Module formatting | formatting fixtures in each SDK | Markdown, Skill, table-row, and JSON surface contracts |

Every SDK runs the relevant shared fixtures. A new cross-language observable
contract should add a fixture here rather than rely on three independently
maintained examples.

## Intentional SDK-native surfaces

Not every API is byte-equivalent or available in every language:

- `ConventionScanner` is Python-only because it depends on Python import and
  type-introspection semantics.
- `flatten_pydantic_params` is Python-only; TypeScript and Rust use native
  type-system and adapter patterns.
- Code-generation writers emit language-native source and therefore have
  language-specific syntax.
- YAML output is currently SDK-native; identical YAML emission is deferred.
- Rust's `http-proxy` feature controls whether HTTP proxy support is included.

Feature pages should document the local contract and link here for the
cross-SDK policy instead of duplicating the full governance explanation.

## Ownership rule for downstream consumers

If two consumers in different languages must produce identical bytes, the
format belongs in `apcore-toolkit` and needs a shared fixture. If the format is
renderer- or terminal-library-specific, it belongs in the consuming product.
Toolkit owns CSV, JSONL, Markdown/Skill module formatting, and the TUI
view-model shape; terminal rendering remains in `apcore-cli`.
