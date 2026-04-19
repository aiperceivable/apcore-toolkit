# apcore Toolkit

The **apcore Toolkit** is a multi-language SDK for scanning, transforming, and registering AI-callable modules in the [apcore](https://github.com/aipartnerup/apcore) ecosystem.

It provides:
- **Module scanning** — discover and extract callable modules from your codebase
- **Output writers** — serialize modules to YAML, Python/TypeScript code, or register directly into an apcore registry
- **Binding loader** — load pre-built `.binding.yaml` files
- **Display overlay** — attach display metadata for richer module presentations
- **AI enhancement** — enrich module metadata using a local SLM

## Language SDKs

| Language | Package | Install |
|---|---|---|
| Python | `apcore-toolkit` | `pip install apcore-toolkit` |
| TypeScript | `apcore-toolkit` | `npm install apcore-toolkit` |
| Rust | `apcore-toolkit` | `cargo add apcore-toolkit` |

## Quick Start

See the [Getting Started](features/scanning.md) guide.

## Features

- [Scanning](features/scanning.md)
- [Output Writers](features/output-writers.md)
- [Binding Loader](features/binding-loader.md)
- [Display Overlay](features/display-overlay.md)
- [Convention Scanning](features/convention-scanning.md)
- [AI Enhancement](ai-enhancement.md)
