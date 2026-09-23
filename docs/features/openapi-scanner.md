---
description: "OpenAPIScanner lifting a whole OpenAPI document into ScannedModule list, with a byte-identical module_id derivation and an HTTP-proxy execution contract."
---

# OpenAPI Scanner — V1

!!! success "Status: IMPLEMENTED"
    Ships in `apcore-toolkit-{python,typescript,rust}`: `derive_module_id`,
    `OpenAPIScanner`, and `load_spec`, asserted byte-/structurally-identical
    across all three SDKs by the shared conformance corpus below. The two
    blocking `HTTPProxyRegistryWriter` defects (W1, W2 — see
    [Blocking prerequisites](#blocking-prerequisites-in-the-shipped-writers))
    were fixed in `apcore-toolkit-rust` alongside this work. It is the
    relocated, re-scoped form of a proposal originally filed against
    `apcore-cli` as "OpenAPI Spec-Driven Command Discovery and Mapping".

    | | |
    |---|---|
    | **Author** | apcore-toolkit maintainers |
    | **First drafted** | 2026-09-03 |
    | **Implemented** | 2026-09-04 |
    | **Tracking issue** | [aiperceivable/apcore-toolkit#16](https://github.com/aiperceivable/apcore-toolkit/issues/16) |
    | **Depends on** | [`openapi.md`](openapi.md) (schema extraction primitives, already shipped) · [`scanning.md`](scanning.md) (`BaseScanner` contract) · [`output-writers.md`](output-writers.md) (`HTTPProxyRegistryWriter`, `YAMLWriter`) |
    | **Affects** | `apcore-cli-{python,typescript,rust}` (Phase 3 consumer work, not yet started — no toolkit-side change required) |

!!! note "Rust: standalone struct, not a `BaseScanner` impl"
    [`apcore-toolkit-rust#4`](https://github.com/aiperceivable/apcore-toolkit-rust/issues/4)
    anticipated `OpenAPIScanner` as the `Scanner` trait's first non-framework
    implementation. In practice, the shipped `BaseScanner` trait's fixed
    shape (`async fn scan(&self, app: &App) -> Vec<ScannedModule>` — single
    argument, infallible) doesn't fit `OpenAPIScanner`'s multiple named
    options and fallible (`Result`) return. Rust's `OpenAPIScanner` is a
    standalone struct with an inherent `scan` method instead, reusing the
    trait's free functions (`filter_modules`, `deduplicate_ids`,
    `infer_annotations_from_method`) directly — exactly as they're designed
    to be reused outside a `BaseScanner` impl. Evolving the trait itself to
    fit this shape remains open on issue #4.

---

## Goal

Turn an **OpenAPI document** into a list of `ScannedModule` — one module per
operation — so that any surface in the ecosystem (CLI, MCP, A2A, REST,
registry) can expose a remote HTTP API without hand-writing a schema or a
routing table.

The unit of work is deliberately `spec → Vec<ScannedModule>`, **not**
`spec → CLI commands`. Producing `ScannedModule` means every downstream
surface is reached at once; producing CLI commands would reach exactly one.

## Non-Goals

- **No CLI command generation.** Mapping `ScannedModule` onto argv, `--help`
  text, and man pages is `apcore-cli`'s job and already exists there. This
  proposal stops at `ScannedModule`.
- **No source-code generation.** The "build-time codegen" half of the original
  proposal is already served by the shipped `YAMLWriter` → `.binding.yaml` →
  `BindingLoader` path (see [Static Mode](#static-mode-build-time)). No new
  codegen surface is introduced.
- **No CLI-tool scanning.** Parsing `--help` output or man pages of arbitrary
  binaries remains out of scope per [Scope & Boundaries](../scope.md#not-a-cli-tool-scanner).
  Scanning a *machine-readable API contract* is a different problem and is
  explicitly listed as Phase 4 of the [ability-extraction methodology](scanning.md#phase-4-discover-existing-api-contracts).
- **No spec authoring, validation, or linting.** The scanner consumes a spec
  it is given. Spec correctness is the API owner's problem; malformed input
  degrades to warnings, never to an exception (see [Error Model](#error-model)).
- **No OpenAPI 2.0 (Swagger) support in V1.** 3.0 and 3.1 only. 2.0 input is
  detected and rejected with a clear error rather than mis-parsed.
- **No authentication.** Acquiring credentials for the target API is the
  subject of the sibling proposal [`device-auth.md`](device-auth.md); the two
  compose at `HTTPProxyRegistryWriter` (see [End-to-End Composition](#end-to-end-composition)).

## Motivation

### What already exists, and what is actually missing

The toolkit already ships the hard part of OpenAPI handling — the
**operation-level** primitives documented in [`openapi.md`](openapi.md):

| Shipped primitive | What it does |
|---|---|
| `extract_input_schema(op, doc)` | Merges path + query + body params into one flat JSON Schema |
| `extract_output_schema(op, doc)` | Pulls the `200`/`201` response schema |
| `deep_resolve_refs(schema, doc)` | Recursively inlines `$ref`, merging any sibling keys over the target (sibling wins — carries `x-sensitive` through); depth-limited to 16 |
| `infer_annotations_from_method(method)` | Maps HTTP verb → `ModuleAnnotations` (canonical, RFC 9110-derived) |

What is missing is only the **document-level** layer above them: the traversal
of `paths` × `methods`, the derivation of a stable `module_id`, and the
population of the fields `HTTPProxyRegistryWriter` needs in order to actually
call the endpoint.

That missing layer is roughly a `for` loop. Its value is not that it is hard —
it is that of everything standing between "an existing HTTP API" and "modules
every apcore surface can consume", it is the only piece nobody has.

### Why this matters more than its size suggests

**This is the on-ramp for existing APIs.** The ecosystem's premise is making
capabilities AI-perceivable; the largest supply of not-yet-perceivable
capability is the REST APIs organisations already run. Today, adopting apcore
for one of them means hand-writing a `ScannedModule` per operation. A typical
OpenAPI document describes dozens to hundreds of operations, so that cost is
not merely tedious — it is high enough to decide against adoption entirely.

Turning it into one function call changes the unit of adoption from "per
endpoint" to "per API".

**And one scan reaches every surface.** Because the output is `ScannedModule`,
a single wrap yields CLI, MCP, A2A, and REST exposure simultaneously — all four
already consume that type. This is the property the ecosystem is built around,
and an OpenAPI scanner is the cheapest way to feed it.

**The naming algorithm is the part that must not be duplicated.** Anyone
writing this loop themselves must invent `module_id` derivation, and that is
exactly the kind of trivial-looking decision that diverges silently — the
failure shape documented in [`formatting.md`](formatting.md), where three
independent `csv`/`jsonl` implementations produced three different outputs
before v0.7.0. Specifying the derivation once, with a conformance corpus,
prevents that class of divergence from arising rather than repairing it after
the fact.

### Why the toolkit and not `apcore-cli`

The original issue scoped this to `apcore-cli`. Implemented there, the result
is an OpenAPI-to-CLI generator — a crowded category already served by
`openapi-generator`, `restish`, and others, with no differentiation and a large
maintenance surface.

Implemented as a `Scanner`, the same work reaches CLI, MCP, A2A, and REST
surfaces simultaneously, because all four already consume `ScannedModule`.
"Wrap once, fan out to every surface" is the property that is *not*
commoditised by existing tooling; scoping to CLI trades it away for a strictly
weaker version of the same feature.

## Architecture

```
                    OpenAPI document (dict / JSON value)
                                  │
              ┌───────────────────┴───────────────────┐
              │           OpenAPIScanner              │
              │  (this proposal — document traversal) │
              │                                       │
              │  for each (path, method, operation):  │
              │    module_id  ← derive_module_id()    │  ← NEW, byte-identical
              │    input      ← extract_input_schema()│  ← shipped
              │    output     ← extract_output_schema()  ← shipped
              │    annots     ← infer_annotations_…() │  ← shipped
              │    metadata   ← http invocation facts │  ← NEW contract
              └───────────────────┬───────────────────┘
                                  │
                          Vec<ScannedModule>
                                  │
            ┌─────────────────────┼─────────────────────┐
            ▼                     ▼                     ▼
   HTTPProxyRegistryWriter    YAMLWriter          any other surface
   (live, executable)      (.binding.yaml)       (MCP / A2A / REST)
            │                     │
     runtime execution     BindingLoader → static mode
```

The scanner is a thin, pure composition of already-shipped parts. Its entire
novel surface is: **`derive_module_id`** and the **`metadata` invocation
contract**. Those two are what need a conformance corpus; everything else
inherits the guarantees it already has.

## Separation of I/O from Traversal

`BaseScanner.scan` is synchronous in Python and TypeScript and `async` in Rust
(see [Contract: BaseScanner.scan](scanning.md#contract-basescannerscan)).
Loading a spec from a URL is network I/O and cannot be made synchronous in
Rust, nor should it be made async in Python purely to accommodate a remote
fetch.

**Decision: I/O is not part of the scanner.** The split is:

| Function | I/O | Async | Owner |
|---|---|---|---|
| `load_spec(source)` | yes — file read or HTTP GET | Python/TS: sync + optional async variant; Rust: `async` | toolkit (thin convenience) |
| `OpenAPIScanner.scan(spec)` | **no** | matches `BaseScanner` per SDK | toolkit (the real work) |

`scan` takes an already-parsed document. This keeps the conformance-tested
surface **pure** — fixtures are plain JSON in, plain JSON out, with no network
mocking in any SDK — and lets consumers who already have a spec in hand (a
FastAPI app can produce its own `app.openapi()` dict in-process) skip the
loader entirely.

`load_spec` is a convenience wrapper, explicitly outside the conformance
corpus, and consumers are free to ignore it and pass their own parsed
document.

## Spec Location and Base URL

Both of these vary per vendor and per stack, and neither may be guessed by the
toolkit.

### The spec URL is always supplied, never inferred

Every framework publishes its OpenAPI document somewhere different — FastAPI,
springdoc, ASP.NET's Swashbuckle, and drf-spectacular all use different default
paths, and any of them can be reconfigured or served from a gateway. The
scanner therefore **never probes candidate paths**. `load_spec` takes the
complete URL or file path the caller provides, and a wrong URL produces an
honest 404 rather than a silent fallback to some other document.

### Fetching a protected spec

An internal API's spec endpoint frequently requires the same credentials as the
API itself. `load_spec` therefore accepts `headers` (and an optional
`auth_header_factory` with the same shape the HTTP proxy writer uses), so the
spec can be fetched with a bearer token:

```python
spec = load_spec(
    "https://api.example.com/v3/api-docs",
    headers={"X-Api-Version": "2024-01"},
    auth_header_factory=client.as_auth_header_factory(),   # device-auth.md
)
```

This is the second place the two proposals compose: credentials are needed not
only to *call* the API but often to *discover* it.

### Base URL resolution

`servers[0].url` is unreliable in practice — it is frequently absent, a
relative path, a developer's `localhost`, or a template with variables. The
resolution order is therefore:

1. **A `base_url` the caller passes to the writer always wins.** This is the
   only value guaranteed correct, because the caller knows which environment it
   is targeting.
2. Otherwise `servers[0].url`, resolved as follows:
    - **Relative** (`"/v1"`) — resolved against the URL the spec was loaded
      from, so a spec fetched from `https://api.example.com/openapi.json`
      yields `https://api.example.com/v1`. A relative server URL in a spec
      loaded from a *local file* cannot be resolved and is skipped.
    - **Templated** (`"https://{tenant}.example.com"`) — variables are
      substituted from `servers[0].variables[*].default`. If any variable has
      no default, the URL is unusable and skipped rather than emitted with
      braces intact.
3. If neither yields a usable absolute URL, `metadata["openapi"]["server_url"]`
   is omitted and the caller **must** supply `base_url` to the writer.

`server_url` remains advisory in all cases — the writer takes `base_url`
explicitly and never reads it. Recording it is a convenience for a consumer
deciding what to pass, not a value the execution path depends on.

## `module_id` Derivation

This is the one genuinely new algorithm in the proposal, and the one place
where SDKs will diverge if it is not pinned exactly. Implementations MUST
follow it byte-for-byte.

### Algorithm

```
derive_module_id(path, method, operation) -> string

1. If operation has a non-empty string "operationId":
       candidate = operationId
       sanitize(candidate), preserving case          → return
2. Otherwise, derive from path + method:
       segments = path.split("/"), dropping empty segments
       for each segment:
           strip a single leading "{" and trailing "}" if both present
       candidate = ".".join(segments + [method])
       candidate = candidate.lowercased()
       sanitize(candidate)                            → return
3. If the result is empty, return "root." + method.lowercased()

sanitize(s):
   replace every character not in [A-Za-z0-9_.-] with "_"
   collapse runs of "." into a single "."
   strip leading and trailing "." and "_"
```

### Worked examples

| Input | `operationId` | Derived `module_id` |
|---|---|---|
| `GET /users` | — | `users.get` |
| `POST /users` | — | `users.post` |
| `GET /users/{user_id}` | — | `users.user_id.get` |
| `DELETE /users/{user_id}/orders/{order_id}` | — | `users.user_id.orders.order_id.delete` |
| `GET /` | — | `root.get` |
| `GET /v1/pets` | — | `v1.pets.get` |
| `GET /users/{id}` | `getUserById` | `getUserById` |
| `POST /a b/c` | — | `a_b.c.post` |

### Design decisions, and why

**Path parameters are kept as segments, not dropped.** Dropping them makes
`GET /users` and `GET /users/{id}` both derive to `users.get`, which then
collides and gets silently renamed to `users.get_2` by
[`deduplicate_ids`](scanning.md#contract-basescannerdeduplicate_ids). The
surviving names would then depend on the iteration order of the spec's `paths`
object — an unstable, meaningless distinction between "get the collection" and
"get one item". Keeping the parameter segment makes both names stable and
self-describing at the cost of some verbosity.

**`operationId` is used verbatim, with case preserved.** The tempting
alternative — normalising `getUserById` → `get_user_by_id` — requires a
camelCase-to-snake_case transformation, and that transformation is a
well-known source of cross-language divergence at acronym boundaries
(`getHTTPResponse` → `get_http_response` or `get_h_t_t_p_response`, depending
on the regex). Since the entire point of this spec is byte-identical output
across three SDKs, V1 takes the option with zero divergence surface. See
[Open Questions](#open-questions) Q1.

**Derived IDs are lowercased; `operationId` IDs are not.** The derived path is
built from HTTP methods and URL paths, which are conventionally lowercase in
module IDs and where the input casing carries no author intent. An
`operationId`, by contrast, was written deliberately by the spec author and is
preserved as-authored.

**Collisions remain possible and are handled downstream.** Two operations may
legitimately carry the same `operationId` in a malformed spec.
`deduplicate_ids` handles this with `_2`/`_3` suffixes and appends a warning to
`ScannedModule.warnings` — the scanner MUST call it, exactly as framework
scanners do.

## Execution Contract (`metadata` keys)

For a framework scanner, `ScannedModule.target` is a dotted import path to a
local callable — `"myapp.views:create_user"`. An OpenAPI operation has **no
local callable**: the implementation lives behind an HTTP endpoint on another
host.

The shipped `HTTPProxyRegistryWriter` resolves this by reading the route from
**`metadata`, never from `target`**. This section is therefore not a design
choice — it is a description of what the writer already requires. A scanner
that emits anything else produces modules the writer cannot execute.

### Required keys (normative — these exact names)

| `metadata` key | Value | Example |
|---|---|---|
| `http_method` | **Uppercase** HTTP method | `"GET"` |
| `url_path` | Path template, `{param}` braces retained, leading slash | `"/users/{user_id}"` |

These two flat, snake_case keys are what all three writers read. Getting either
name wrong is not a soft failure: the writer silently falls back to `"GET"` and
`"/"`, and every scanned module quietly proxies to the API root.

!!! danger "Uppercase is mandatory, not cosmetic"
    The Rust writer matches the method against exact uppercase literals and
    returns `Unsupported HTTP method: post` for a lowercase value. The Python
    writer compares case-sensitively when deciding whether a request carries a
    body, so a lowercase `"post"` would send its arguments as a **query
    string** instead of a JSON body. Only the TypeScript writer upper-cases
    defensively.

    OpenAPI path-item keys are lowercase by specification, so the scanner
    **MUST** upper-case the method when writing `http_method`. This is the
    single most likely way to get a working Python/TypeScript integration and a
    broken Rust one.

### Informational keys

OpenAPI-specific facts that no writer reads, recorded under a nested key so
they cannot collide with the two normative names above:

| `metadata["openapi"]` field | Value | Omitted when |
|---|---|---|
| `operation_id` | The spec's `operationId` | absent from the spec |
| `spec_version` | The document's `openapi` version string | never |
| `server_url` | `servers[0].url` | no `servers` entry |
| `summary` | `operation.summary` | absent |

### What the scanner deliberately does *not* record

The writer partitions inputs by itself: it extracts path-parameter names from
`url_path`, removes those from the remaining inputs, and sends the rest as a
JSON body for `POST`/`PUT`/`PATCH` or as a query string otherwise.

An earlier draft of this proposal had the scanner emit explicit
`path_params` / `query_params` / `body_params` lists, on the reasoning that
`extract_input_schema` merges all three into one flat schema (see
[`openapi.md` § Parameter Merging](openapi.md#parameter-merging)) and destroys
the distinction. That reasoning was wrong in practice: the writer reconstructs
the path/non-path split from `url_path`, and its body-vs-query decision is
driven by the HTTP method rather than by the spec. Emitting the lists would
create a second source of truth that the writer ignores — worse than useless,
because it would look authoritative.

The one real casualty is a query parameter that OpenAPI declares for a
`POST`: the writer will place it in the body. That is a limitation of the
shipped writer, not something the scanner can fix by emitting more metadata,
and it is recorded in [Risks](#risks).

### Which field gets `target`, then?

`target` is a required `ScannedModule` field and must hold something. The
scanner sets it to `"<METHOD> <url_path>"` (e.g. `"GET /users/{user_id}"`) as
a stable, human-readable identifier that surfaces in listings and generated
documentation.

This **deviates from `target`'s documented meaning** ("callable reference in
`module.path:callable` format"). The deviation is safe today because no writer
reads the field, but it should be made explicit rather than left implicit:
[`scanning.md`](scanning.md#contract-scannedmodule) should note that modules
representing remote endpoints carry a route descriptor in `target` instead of
an import path. Filed as [Open Questions](#open-questions) Q2.

### Blocking prerequisites in the shipped writers

Verification of the three implementations surfaced defects that this scanner
would trigger in normal use. These are **pre-existing writer bugs**, not
scanner design issues, but the scanner is what makes them reachable, so they
must be fixed alongside — or the OpenAPI integration will be quietly broken on
Rust.

| # | Defect | Impact on OpenAPI-scanned modules |
|---|---|---|
| W1 | ~~Rust's path-parameter regex is `\{(\w+)\}` (word characters only), while Python/TypeScript use `\{[^}]+\}`~~ — **see correction below** | ~~OpenAPI permits `{item-id}` and `{id:int}`. On Rust these are not recognised as path parameters, so the value is sent as a query argument~~ — see correction |
| W2 | Rust accepts only `GET`/`POST`/`PUT`/`PATCH`/`DELETE` and errors on anything else | `HEAD`, `OPTIONS`, and `TRACE` operations scan fine but fail at execution on Rust only |
| W3 | Rust reads `metadata` only; Python/TypeScript also honour top-level `http_method`/`url_path` attributes, and TypeScript additionally accepts camelCase `metadata.httpMethod`/`metadata.urlPath` | Not triggered by this scanner (it writes snake_case metadata, the intersection all three read) — documented as intentional in `output-writers.md` rather than reconciled |
| W4 | Rust does not reject an absolute URL in `url_path`; Python/TypeScript do | A spec whose path is an absolute URL yields a malformed concatenated URL on Rust instead of a failed `WriteResult` — not yet fixed, tracked separately |

!!! success "W1 and W2 fixed in `apcore-toolkit-rust` — W1 diagnosis corrected"
    Re-verifying against the actual code during implementation found the W1
    row above overstated the bug: Rust's path-parameter *extraction and
    substitution* (used to build the actual outgoing request) already went
    through `extract_path_param_names`, which uses the same broad
    `\{[^}]+\}` pattern as Python/TypeScript — a hyphenated param like
    `{item-id}` was already correctly recognised and substituted. The narrow
    `\{(\w+)\}` regex was used only by a separate, later
    *unfilled-placeholder* check (verifying no `{...}` template syntax was
    left in the URL after substitution) — so the real, narrower gap was
    that an unfilled hyphenated param would not have been flagged by that
    check, not that a filled one was silently mis-routed to the query
    string. Fixed by having the unfilled-placeholder check reuse
    `extract_path_param_names` directly, removing the second, divergent
    regex literal. W2 was confirmed exactly as described (HEAD/OPTIONS/
    TRACE rejected before any network call) and fixed by adding the three
    missing method-match arms. Both fixes shipped with regression tests.
    W4 remains open — out of scope for this proposal, filed separately.

## Annotations

Annotations come from the shipped
[`infer_annotations_from_method`](scanning.md#contract-basescannerinfer_annotations_from_method),
which the scanner MUST call rather than reimplement. OpenAPI adds exactly one
signal of its own:

| OpenAPI field | Effect |
|---|---|
| `operation.deprecated == true` | Sets `annotations.deprecated = true` |

No other OpenAPI field is mapped to an annotation in V1. In particular,
`security` requirements are **not** mapped to `requires_approval` — needing a
token is not the same as needing a human to approve the call, and conflating
them would spuriously gate every operation of an authenticated API behind an
approval prompt.

## Descriptions and Documentation

| `ScannedModule` field | Source, in precedence order |
|---|---|
| `description` | `operation.summary` → first line of `operation.description` → `""` |
| `documentation` | `operation.description` → `None` |
| `tags` | `operation.tags` → `[]` |
| `version` | `document.info.version` → `"1.0.0"` |
| `examples` | Not populated in V1 — see below |

`summary` is preferred for `description` because OpenAPI defines it as the
short form, matching `ScannedModule.description`'s one-line contract, while
`description` is explicitly allowed to be long-form CommonMark.

`examples` stays empty in V1. OpenAPI carries examples in at least four
distinct places (`operation.requestBody.content.*.example`, `.examples`,
per-parameter `example`, and response examples), with differing shapes between
3.0 and 3.1. Mapping them onto `ModuleExample` correctly is a self-contained
problem worth its own iteration, and getting it wrong produces confidently
misleading documentation. Emitting nothing is the honest V1 answer.

## Dynamic and Static Modes

The original issue asked for both a runtime mode and a build-time mode. Both
fall out of existing pieces; neither needs new machinery beyond the scanner.

### Dynamic mode (runtime)

Load the spec, scan it, register the modules. The CLI's command surface
reflects whatever the API currently advertises.

```
load_spec(url) → scan() → HTTPProxyRegistryWriter.write() → Registry
```

Cost: one spec fetch and parse per process start. Benefit: zero redeploy when
the API changes.

### Static mode (build time)

Scan once during the build, persist to `.binding.yaml`, load that at runtime.

```
build:    load_spec(url) → scan() → YAMLWriter.write() → api.binding.yaml
runtime:  BindingLoader.load("api.binding.yaml") → HTTPProxyRegistryWriter.write()
```

Cost: a re-sync step when the API changes. Benefit: no network dependency and
no parse cost at startup, plus the generated bindings are reviewable artifacts
that can be diffed in a pull request — which is how an API contract change
becomes visible to a human before it ships.

**Both writers and the loader already exist.** The original issue's
"build-time utility to generate optimized, type-safe command stubs embedded
into the binary" is served by this path without generating any source code.
This removes what would otherwise have been the largest and least maintainable
part of the work.

## End-to-End Composition

This proposal and [`device-auth.md`](device-auth.md) are two halves of one
scenario — *"expose a remote, authenticated HTTP API as apcore modules"* —
and they meet at exactly one point: `HTTPProxyRegistryWriter`'s existing
`auth_header_factory` hook.

```python
# 1. Credentials  (device-auth.md proposal)
tokens = DeviceAuthClient(config).login(on_user_code=cli_prompt)

# 2. Modules      (this proposal)
spec    = load_spec("https://api.example.com/openapi.json")
modules = OpenAPIScanner().scan(spec)

# 3. Execution    (both — already shipped)
writer = HTTPProxyRegistryWriter(
    base_url="https://api.example.com",
    auth_header_factory=tokens.as_auth_header_factory(),
)
writer.write(modules, registry)
```

Neither proposal depends on the other being implemented: the scanner works
against unauthenticated or statically-keyed APIs, and the device-auth client
is useful to anything holding a bearer token. They are specified separately
and can ship in either order.

## Public API

### Python

```python
from apcore_toolkit import OpenAPIScanner, load_spec

spec = load_spec("https://api.example.com/openapi.json")   # or a local path
# or: spec = json.load(open("openapi.json"))               # loader is optional

scanner = OpenAPIScanner()
modules = scanner.scan(
    spec,
    include=r"^users\.",       # forwarded to filter_modules
    exclude=None,
    base_path_prefix=None,     # optional module_id namespace, e.g. "petstore"
    include_deprecated=True,
)
```

### TypeScript

```typescript
import { OpenAPIScanner, loadSpec } from "apcore-toolkit";

const spec = await loadSpec("https://api.example.com/openapi.json");

const scanner = new OpenAPIScanner();
const modules = scanner.scan(spec, {
  include: "^users\\.",
  exclude: undefined,
  basePathPrefix: undefined,
  includeDeprecated: true,
});
```

### Rust

```rust
use apcore_toolkit::{OpenAPIScanner, ScanOptions, load_spec};

let spec = load_spec("https://api.example.com/openapi.json").await?;

let scanner = OpenAPIScanner::new();
let modules = scanner.scan(&spec, &ScanOptions {
    include: Some("^users\\.".into()),
    exclude: None,
    base_path_prefix: None,
    include_deprecated: true,
}).await?;
```

Rust's `scan` is `async`, matching the ecosystem convention that a
`Scanner`'s `scan` method is async — even though this implementation
performs no I/O, the document is already parsed by the time it arrives.
`OpenAPIScanner` is a standalone struct with this inherent `scan` method,
**not** an `impl BaseScanner for OpenAPIScanner`: the shipped
`BaseScanner` trait's fixed shape
(`async fn scan(&self, app: &App) -> Vec<ScannedModule>` — one argument,
infallible) doesn't accommodate `OpenAPIScanner`'s multiple named options
and fallible `Result` return. See the note at the top of this document and
[`apcore-toolkit-rust#4`](https://github.com/aiperceivable/apcore-toolkit-rust/issues/4)
(evolving the trait to fit this shape remains open, tracked separately).
`OpenAPIScanner` does reuse the trait's free functions (`filter_modules`,
`deduplicate_ids`, `infer_annotations_from_method`) directly.

## Error Model

A scanner that raises on the first malformed operation is useless against
real-world specs, which are frequently non-conforming in small ways. The
scanner therefore **degrades rather than fails**:

| Condition | Behaviour |
|---|---|
| Spec is not OpenAPI 3.x (`openapi` key missing, or `swagger: "2.0"`) | Raise / throw / `Err` — this is a caller error, not spec noise |
| `paths` missing or empty | Return an empty module list; no error |
| Operation has no `operationId` | Derive from path + method (normal path, not an error) |
| Operation has unresolvable `$ref` | Schema degrades to `{}` per the shipped [`resolve_ref` contract](openapi.md#contract-resolve_ref); module is still emitted; a warning is appended to `ScannedModule.warnings` |
| Operation has no request body and no parameters | `input_schema` is `{}` — valid, means "takes no arguments" |
| Operation has no `200`/`201` response | `output_schema` is `{}`; warning appended |
| Path item contains non-operation keys (`summary`, `parameters`, `servers`, `$ref`) | Skipped, not treated as HTTP methods |
| Duplicate derived `module_id` | Resolved by `deduplicate_ids`, warning appended |

Only the first row raises. Everything else produces a module plus a warning,
because a partially-understood operation is more useful than a hard failure —
and `warnings` is exactly the field
[`ScannedModule`](scanning.md#contract-scannedmodule) provides for
human audit of scan-time degradation.

### Recognised HTTP methods

Only these keys in a path item are treated as operations, matching the OpenAPI
3.x Path Item Object specification:

`get`, `put`, `post`, `delete`, `options`, `head`, `patch`, `trace`

The scanner recognises all eight, because scanning is about faithfully
describing the API. Note that only five of them are currently *executable*
through the Rust proxy writer — see defect W2 in
[Blocking prerequisites](#blocking-prerequisites-in-the-shipped-writers).
Scanning them anyway is correct: a `HEAD` operation is a real part of the API
surface and belongs in the module list even on an SDK that cannot yet proxy it.

Iteration order MUST be the document's own key order (not this list's order,
and not alphabetical) so that `deduplicate_ids` suffix assignment is
reproducible across SDKs. Implementations in languages with unordered map
types MUST use an insertion-ordered parse (Python `dict` and JS objects
preserve insertion order natively; Rust requires `serde_json`'s
`preserve_order` feature, which pulls in `indexmap`).

## Extension Hooks

Real specs are non-conforming in vendor-specific ways, and organisations have
their own module-naming conventions. Declarative options cannot anticipate
either, so the scanner accepts three hooks. All are optional; with none
installed, behaviour is exactly what the conformance corpus asserts.

| Hook | Signature | Purpose |
|---|---|---|
| `transform_operation` | `(path, method, operation) -> operation \| None` | Patch or normalise an operation before extraction. Returning `None` **skips** the operation entirely |
| `derive_module_id` | `(path, method, operation) -> string \| None` | Override the naming algorithm. Returning `None` falls back to the [default derivation](#module_id-derivation) |
| `transform_module` | `(module) -> ScannedModule \| None` | Adjust the finished module — extra metadata, tag rewriting. Returning `None` drops it from the result |

`transform_operation` is where vendor `x-*` extensions are handled. OpenAPI's
extension mechanism is open by design, and specs in the wild carry pagination
hints, auth requirements, and rate-limit annotations under vendor-prefixed
keys. Rather than guessing at a mapping for extensions the toolkit cannot know
about, it gives the consumer the operation object and lets them fold whatever
matters into a shape the scanner already understands.

### The trade this makes explicit

Unlike the sibling proposal, this scanner has **no protocol-decision layer** to
protect — it is a pure transformation. The property at risk is different:

!!! warning "`derive_module_id` overrides the corpus guarantee — deliberately"
    Module ID derivation is the *primary* subject of the conformance corpus,
    precisely because it is the thing most likely to drift across SDKs.
    Overriding it hands that guarantee back to the consumer: if they run
    scanners in more than one language, **they must implement the same naming
    logic in each**, and nothing will warn them if they diverge.

    The override exists anyway, because an organisation with an established
    module-naming convention would otherwise be unable to adopt the scanner at
    all — and would write their own loop, which is the duplication this
    proposal set out to eliminate. Offering the hook keeps them on the shared
    extraction path while they diverge only on naming.

    Consumers running a single SDK, which is most of them, pay nothing.

### Invocation order

```
for each (path, method, operation) in document order:
    operation ← transform_operation(...)        skip if None
    module_id ← derive_module_id(...) ?? default derivation
    module    ← build from shipped extraction primitives
    module    ← transform_module(module)        drop if None
then: filter_modules → deduplicate_ids
```

Hooks run **before** filtering and deduplication, so `include`/`exclude`
patterns match hook-produced IDs and collisions introduced by a custom naming
scheme are still resolved by `deduplicate_ids`.

## Security Considerations

| Risk | Mitigation |
|---|---|
| **SSRF via `load_spec(url)`** | `load_spec` performs an arbitrary GET against a caller-supplied URL. It is a convenience helper, outside the conformance corpus, and MUST document that the URL is trusted input. Callers taking a spec URL from an untrusted source are responsible for their own allowlisting. |
| **External `$ref` resolution** | The shipped `resolve_ref` handles internal JSON pointers (`#/...`) only. External refs (`https://…`, `./other.yaml`) MUST NOT be fetched — resolving them would turn a spec parse into an arbitrary outbound request chain. They degrade to `{}` plus a warning. |
| **Billion-laughs / deeply nested `$ref`** | Already bounded by the shipped depth limit of 16 (see [`deep_resolve_refs`](openapi.md#contract-deep_resolve_refs)). |
| **Circular `$ref`** | Same depth limit; terminates rather than hanging. |
| **Prototype pollution (TypeScript)** | Already handled by the shipped `safe-keys` guard in `resolve_ref` (blocks `__proto__`, `constructor`, `prototype`). Inherited at no cost. |

!!! note "This guard is a known, intentional TS-only source of cross-SDK output divergence"
    Because the guard blocks `$ref` path segments and schema/parameter keys literally named `__proto__`, `constructor`, or `prototype`, a spec using one of those names (valid JSON, if unusual) produces a *different* `scan()` result in TypeScript (the field is dropped, with a warning) than in Python or Rust (the field is resolved/kept normally, since neither language is vulnerable to the underlying attack). This is out of scope for the byte-identical conformance corpus below — no fixture exercises a `__proto__`-named schema key — and is not a bug to reconcile: the vulnerability class this guard defends against is JS-runtime-specific.
| **Spec-driven credential leakage** | The scanner MUST NOT copy `securitySchemes`, `servers[].variables` defaults, or any `example` value containing credentials into `metadata`. Only the fields enumerated in [Execution Contract](#execution-contract-metadata-keys) are emitted. |
| **Enormous specs** | A spec with tens of thousands of operations produces a module per operation. No limit is imposed in V1, but consumers should apply `include`/`exclude` filters. Noted in [Risks](#risks). |

## Conformance Corpus

A fixture file lives at `conformance/fixtures/openapi_scan.json`, following the
structure already used by `format_csv.json` and `display_resolve.json`
(a single JSON document with `$schema`, `title`, `description`, `version`, and
a `test_cases` array of `{id, description, input, expected}`).

!!! note "Fixture layout"
    The corpus convention is **one file containing many `test_cases`**, not one
    file per case. This matches the shipped `format_csv.json` /
    `display_resolve.json` fixtures and the other proposed corpora.

Because `scan` is pure, each case is a spec fragment in and a module list out,
with no network mocking required in any SDK.

| Test case | Purpose |
|---|---|
| `openapi_scan_001_empty_paths` | `paths: {}` → empty list, no error |
| `openapi_scan_002_single_get` | Minimal `GET /users` → one module, `users.get` |
| `openapi_scan_003_operation_id_preserved` | `operationId: getUserById` used verbatim, case preserved |
| `openapi_scan_004_path_param_segment` | `GET /users/{user_id}` → `users.user_id.get` |
| `openapi_scan_005_collection_vs_item` | `GET /users` and `GET /users/{id}` coexist without collision |
| `openapi_scan_006_all_methods` | All 8 recognised methods on one path |
| `openapi_scan_007_non_operation_keys_skipped` | `summary` / `parameters` / `servers` in a path item are not scanned as methods |
| `openapi_scan_008_annotations_from_method` | `GET`→readonly+cacheable, `DELETE`→destructive, `PUT`→idempotent; also asserts `metadata.http_method` is **uppercase** |
| `openapi_scan_009_deprecated_flag` | `deprecated: true` → `annotations.deprecated` |
| `openapi_scan_010_execution_metadata` | `metadata.http_method` and `metadata.url_path` present, correctly named, and consumable by `HTTPProxyRegistryWriter` |
| `openapi_scan_011_description_precedence` | `summary` wins over `description` for the `description` field |
| `openapi_scan_012_ref_resolution` | Nested `$ref` in request body fully inlined |
| `openapi_scan_013_unresolvable_ref_warns` | Bad `$ref` → `{}` schema + warning, module still emitted |
| `openapi_scan_014_external_ref_refused` | `$ref: "https://…"` not fetched → `{}` + warning |
| `openapi_scan_015_duplicate_operation_id` | Two ops share an `operationId` → `_2` suffix + warning |
| `openapi_scan_016_sanitize_illegal_chars` | `POST /a b/c` → `a_b.c.post` |
| `openapi_scan_017_root_path` | `GET /` → `root.get` |
| `openapi_scan_018_no_success_response` | Only a `404` defined → empty `output_schema` + warning |
| `openapi_scan_019_document_order_preserved` | Module order follows document key order, not alphabetical |
| `openapi_scan_020_swagger_2_rejected` | `swagger: "2.0"` raises in all three SDKs |
| `openapi_scan_021_hook_skip_operation` | `transform_operation` returning `None` omits that operation |
| `openapi_scan_022_hook_custom_module_id` | `derive_module_id` override is used; returning `None` falls back to the default |
| `openapi_scan_023_hook_before_dedup` | A custom naming scheme producing a collision is still resolved by `deduplicate_ids` |
| `openapi_scan_024_no_hooks_matches_baseline` | With no hooks installed, output is identical to the corresponding hook-free case |

Each SDK's test suite runs the corpus; CI fails on divergence.

!!! danger "Prerequisite: fixtures must actually run"
    [Issue #15](https://github.com/aiperceivable/apcore-toolkit/issues/15)
    ("Conformance fixtures are silently skipped in all three SDK CIs") is
    closed, but this proposal's entire cross-SDK guarantee rests on the corpus
    genuinely executing. Verify that the new fixture file is picked up and
    failing-on-purpose in each SDK before trusting a green build.

---

## Contract: OpenAPIScanner.scan

### Inputs
- `spec` / `document`: dict / `Record<string, unknown>` / `serde_json::Value`, required — a parsed OpenAPI 3.0 or 3.1 document
- `include`: string regex, optional — forwarded to `filter_modules`
- `exclude`: string regex, optional — forwarded to `filter_modules`
- `base_path_prefix` / `basePathPrefix`: string, optional, default `None` — when set, prepended to every derived `module_id` as `"<prefix>.<id>"`; applied **before** filtering and deduplication
- `include_deprecated` / `includeDeprecated`: boolean, optional, default `true` — when `false`, operations with `deprecated: true` are omitted entirely rather than annotated
- `transform_operation`, `derive_module_id`, `transform_module`: callables, optional — see [Extension Hooks](#extension-hooks). Each may return `None` to mean "no opinion" (`derive_module_id`) or "drop this" (the other two)

### Errors
- Non-OpenAPI-3.x input (missing `openapi` key, or `swagger: "2.0"`): Python raises `ValueError`; TypeScript throws `InvalidSpecError`; Rust returns `Err(ScannerError::InvalidSpec)`
- Invalid `include`/`exclude` regex: inherits the [`filter_modules` error contract](scanning.md#contract-basescannerfilter_modules) exactly (Python `ValueError`, TypeScript `SyntaxError`, Rust `Err(regex::Error)`)
- No other condition raises — malformed operations degrade to warnings per the [Error Model](#error-model)

### Returns
- On success: `list[ScannedModule]` / `ScannedModule[]` / `Vec<ScannedModule>` — one module per recognised operation, in document key order, after filtering and deduplication

### Properties
- async: false (Python, TypeScript) / true (Rust — satisfies the `Scanner` trait)
- pure: true — no I/O; the document is already parsed
- thread_safe: true — no mutation of the input document
- deterministic: true — identical input yields byte-identical output across all three SDKs

---

## Contract: OpenAPIScanner.get_source_name

### Inputs
- None (self/this only)

### Errors
- None

### Returns
- On success: the literal string `"openapi"` in all three SDKs

### Properties
- async: false
- pure: true

---

## Contract: derive_module_id

### Inputs
- `path`: string, required — the OpenAPI path template (e.g., `"/users/{user_id}"`)
- `method`: string, required — the HTTP method key as written in the document (e.g., `"get"`)
- `operation`: dict / object, required — the operation object, consulted only for `operationId`

### Errors
- None raised — every input produces some identifier; empty results fall back to `"root.<method>"`

### Returns
- On success: string — the derived module ID per the [algorithm](#algorithm)

### Properties
- async: false
- pure: true
- deterministic: true — this function is the primary subject of the conformance corpus
- availability: exported publicly in all three SDKs, so that adapters building non-OpenAPI HTTP scanners can reuse the same naming convention

---

## Contract: load_spec

### Inputs
- `source`: string or Path, required — a local filesystem path or an `http(s)://` URL. Taken verbatim; no candidate paths are probed
- `headers`: mapping of string→string, optional — extra request headers (API-version headers, tenant selectors, and similar vendor requirements)
- `auth_header_factory` / `authHeaderFactory`: callable, optional — same shape the HTTP proxy writer uses; invoked once per fetch, for specs behind authentication
- `timeout`: number, optional — request timeout; unit follows each SDK's existing convention (Python/Rust seconds, TypeScript milliseconds), matching `HTTPProxyRegistryWriter`

### Errors
- File not found / unreadable: Python `OSError`; TypeScript `Error`; Rust `Err(io::Error)`
- HTTP non-2xx or network failure: Python `httpx.HTTPError`; TypeScript `Error`; Rust `Err(reqwest::Error)`
- Malformed JSON/YAML: Python `ValueError`; TypeScript `SyntaxError`; Rust `Err(serde_json::Error)`

### Returns
- On success: the parsed document as the SDK's native mapping type

### Properties
- async: false in Python (sync `httpx`), true in TypeScript and Rust
- pure: false — performs file or network I/O
- conformance: **excluded from the corpus** — I/O behaviour is deliberately not byte-specified
- security: the URL is trusted input; see [Security Considerations](#security-considerations)

---

## Migration Plan

### Phase 1 — Close the writer gaps (blocking) — done

Three independent pieces of work, none of which is scanner code.

1. [x] **Document the `metadata` contract** in `output-writers.md` — see
   § "`metadata` Contract" under `HTTPProxyRegistryWriter`. Includes the
   uppercase requirement, the `GET` / `/` fallback behaviour, and the
   method-driven body-vs-query rule.
2. [x] **Fix Rust defects W1 and W2** (see
   [Blocking prerequisites](#blocking-prerequisites-in-the-shipped-writers)).
   Re-verifying W1 against the actual code during implementation found the
   original diagnosis above overstated it: path-parameter *extraction and
   substitution* already used the broad pattern (same breadth as Python/
   TypeScript) via `extract_path_param_names`; only the separate
   post-substitution *unfilled-placeholder* check used the narrow
   `\{(\w+)\}` regex, so a hyphenated param left unfilled by the caller
   went undetected — a real but narrower gap than "silently wrong request"
   implied. Fixed by having that check reuse `extract_path_param_names`
   directly, removing the second, divergent regex literal. W2 (`HEAD`/
   `OPTIONS`/`TRACE` rejected before any network call) was confirmed
   exactly as described and fixed by adding the three missing method-match
   arms. Both fixes shipped with regression tests in `apcore-toolkit-rust`.
3. [x] **Reader asymmetry (W3, W4)** — documented as intentional rather
   than reconciled; see the "Cross-SDK reader asymmetry" note in
   `output-writers.md`. No shipped scanner (including this one) relies on
   the wider Python/TypeScript reader surface, so the scanner's own
   correctness doesn't depend on closing this gap.

### Phase 2 — Toolkit implementation — done

In `apcore-toolkit-{python,typescript,rust}`:

1. [x] Implement `derive_module_id` as a standalone public function.
2. [x] Implement `OpenAPIScanner` on top of the shipped extraction primitives.
3. [x] Implement `load_spec`.
4. [x] Add `conformance/fixtures/openapi_scan.json` and wire it into all three
   test suites.
5. [x] Add the Contract blocks above to this file and flip the status banner.
6. [x] Update [`overview.md`](overview.md) and `mkdocs.yml`. (`README.md`
   left as-is — it lists SDKs and top-level capabilities, not individual
   feature proposals.)

Rust already had `serde_json`'s `preserve_order` feature enabled crate-wide
(see [Recognised HTTP methods](#recognised-http-methods)) — no change needed
there.

### Phase 3 — Consumers (separate PRs, per repo)

- `apcore-cli-*`: an `--from-openapi <url|path>` entry point that runs the
  scanner and registers via `HTTPProxyRegistryWriter`. No toolkit change.
- Framework adapters (`fastapi-apcore`, `django-apcore`, …) may optionally
  offer OpenAPI-based scanning as an alternative to code introspection for
  routes whose handlers cannot be imported.

## Open Questions

1. **`operationId` normalisation.** V1 preserves case and does not convert
   camelCase to snake_case, avoiding acronym-boundary divergence across SDKs.
   The cost is that `module_id`s from an `operationId`-rich spec look unlike
   those from framework scanners (`getUserById` vs `users.get_user`).
   **Recommendation: keep verbatim for V1**; revisit only if a consumer
   demonstrates concrete friction, since normalisation can be added later as
   an opt-in flag without breaking existing IDs.
2. **`target` carries a route descriptor, not an import path.** Resolved as a
   fact, open as a documentation question: the writers ignore `target`
   entirely, so `"GET /users/{id}"` is safe there, but it contradicts the
   field's documented meaning in
   [`scanning.md`](scanning.md#contract-scannedmodule).
   **Recommendation: amend `scanning.md` to state that modules representing
   remote endpoints carry a route descriptor**, rather than inventing a fake
   import path or leaving the contradiction undocumented.
3. **`base_path_prefix` before or after filtering?** Specified above as
   *before*, so `include` patterns match the final visible ID. The alternative
   (filter on the bare ID, then prefix) would make filters independent of
   namespacing. **Recommendation: before**, on the principle that a filter
   should match what the user actually sees.
4. **Multiple `servers[]` entries and per-operation overrides.** Relative and
   templated server URLs are specified in
   [Base URL resolution](#base-url-resolution); what remains open is a spec
   listing several servers (prod/staging/regional) or overriding `servers` at
   the path-item or operation level. V1 records only the document-level
   `servers[0]`. **Recommendation: keep it at one advisory value.** The caller
   passes the real `base_url` to the writer, so modelling an environment list
   the writer cannot act on would add surface without capability. If per-
   operation servers turn out to matter, the natural V2 shape is an extra
   `metadata["openapi"]["server_url"]` override on just those modules.
5. **Response-status selection.** Inherited from the shipped
   `extract_output_schema`, which handles `200`/`201` only. Operations
   returning `202 Accepted` or `204 No Content` get an empty `output_schema`.
   Widening this is a change to a shipped contract and belongs in its own
   issue, not here.
6. **Should `examples` be populated in V2?** Deferred; see
   [Descriptions and Documentation](#descriptions-and-documentation).

## Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| Rust defect W1 makes `{item-id}`-style parameters silently produce wrong requests | **High** (any spec using a hyphenated path parameter) | Phase 1 item 2 fixes it before the scanner can reach it. This is the single most dangerous interaction, because it fails silently rather than loudly. |
| Rust defect W2 makes `HEAD`/`OPTIONS`/`TRACE` operations unexecutable | Medium | Phase 1 item 2. Scanning them is still correct; only proxy execution is affected, and only on Rust. |
| A `POST` operation declaring query parameters has them sent in the body | Medium | Limitation of the shipped writer's method-driven partitioning, not fixable from the scanner. Documented in [Execution Contract](#execution-contract-metadata-keys); a writer-side fix would need explicit parameter-location metadata and is out of scope here. |
| Lowercase `http_method` reaching the writer | Low (spec mandates uppercasing) | Conformance fixture `openapi_scan_008` asserts the emitted method is uppercase. |
| `module_id` derivation diverges across SDKs despite the spec | Medium | 20-case conformance corpus, with the derivation as its primary subject. This is exactly the failure the corpus mechanism exists to catch. |
| Real-world specs violate assumptions in ways not covered by fixtures | Medium | Degrade-with-warning error model means unknown shapes produce imperfect modules, never crashes. Add fixtures as real specs surface problems. |
| Huge specs produce unmanageable module counts | Low-medium | `include`/`exclude` filters shipped from day one; documented in the API examples. |
| Scope creep back toward CLI generation / codegen | Medium | Non-Goals are explicit, and the static-mode path deliberately reuses `YAMLWriter` instead of generating source. |
| `derive_module_id` override causes silent cross-SDK ID divergence for multi-language consumers | Low-medium | Documented as an explicit trade in [Extension Hooks](#extension-hooks) rather than buried. Single-SDK consumers — the majority — are unaffected, and the alternative (no override) pushes those users into hand-rolling the whole scan loop. |
| OpenAPI 3.1's JSON Schema 2020-12 divergences (e.g., `exclusiveMinimum` as number, `type` arrays) surprise consumers | Low-medium | The toolkit passes schemas through without interpretation; downstream validators own their dialect handling. Note it in the docs rather than transforming. |

## Implementation Estimate

| Phase | Component | Estimated LOC | Notes |
|---|---|---|---|
| 1 | `output-writers.md` reconciliation | ~60 | Docs only; unblocks everything else |
| 2 | Python — scanner + derivation + loader + tests | ~350 | Extraction primitives already exist |
| 2 | TypeScript — same | ~350 | |
| 2 | Rust — same (+ `preserve_order`) | ~400 | Async trait wiring adds a little |
| 2 | Conformance corpus (24 cases) | ~520 | Shared; the bulk is spec fragments. 4 cases assert hook contracts |
| 2 | Spec doc updates (this file, overview, README, mkdocs) | ~150 | |
| **Phase 2 total** | | **~1700 LOC** | Comparable to the TuiViewModel estimate |
| 3 | `apcore-cli` `--from-openapi` per SDK | ~80 each | Consumer-side, no toolkit change |

The estimate is dominated by fixtures rather than logic, which is the expected
shape when the feature is a thin composition of shipped primitives whose main
risk is cross-SDK drift.

## See Also

- [`openapi.md`](openapi.md) — the shipped operation-level extraction primitives this builds on.
- [`scanning.md`](scanning.md) — `BaseScanner` contract, `ScannedModule` fields, annotation inference.
- [`output-writers.md`](output-writers.md) — `HTTPProxyRegistryWriter` (execution) and `YAMLWriter` (static mode).
- [`binding-loader.md`](binding-loader.md) — the read path completing static mode.
- [`device-auth.md`](device-auth.md) — the sibling proposal supplying credentials for authenticated APIs.
- [Scope & Boundaries](../scope.md) — why CLI-tool scanning stays out and API-contract scanning comes in.
- [`apcore-toolkit-rust#4`](https://github.com/aiperceivable/apcore-toolkit-rust/issues/4) — the `Scanner` trait this implements.
