# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

Two independent bodies of work, deliberately kept as separate releases rather than one. **0.12.0** closes a latent spec gap in `BindingLoader`; **0.13.0** adds the toolkit's first credential-handling surface. They are separated because Python and TypeScript have no feature flag for the latter, so a consumer's only lever for keeping an OAuth client out of its dependency graph is the version number. (Rust additionally gates it behind a `device-auth` cargo feature; `cargo check --no-default-features` is asserted so `apexe`, which depends on this crate that way, is untouched.)

---

## [0.13.0] - unreleased

Adds the **RFC 8628 Device Authorization Flow client** ([#17](https://github.com/aiperceivable/apcore-toolkit/issues/17)) — the protocol half of the credential story, in all three SDKs, asserted byte-equivalent by a shared 64-case corpus.

### Added

- **`DeviceAuthClient`, `DeviceAuthConfig`, `TokenSet`, `TokenStore`, `FileTokenStore`, `Grant` / `DeviceCodeGrant`** and the four extension hooks. The toolkit writes nothing to a terminal: it emits events, and the consumer renders them. See [`docs/features/device-auth.md`](docs/features/device-auth.md).
- **`conformance/fixtures/device_auth.json`** — 64 cases across ten case kinds. **No HTTP mocking is required in any SDK**: the state machine is pure over an injected monotonic clock and a scripted response sequence, so a harness feeds responses in order and records the sleeps. Written, reviewed, and committed *before* any SDK started, the same ordering that gave `TuiViewModel` zero divergences and whose absence gave `csv`/`jsonl` three independent bugs.
- **V1 ships device flow only**, with the `Grant` interface in place so a second grant is one implementation against a stable seam rather than a rewrite. This is a deliberate scope-down from the proposal's own recommendation (device flow + manual code entry + PKCE): PKCE widens the exact surface whose risk mitigation depends on being narrow, and no consumer has yet named a provider lacking device-flow support. The decision reverses the moment one does.

### Changed

- **`HTTPProxyRegistryWriter.authHeaderFactory` (TypeScript) widened to `Record | Promise<Record>`** and awaited — backward-compatible, since awaiting a non-promise is a no-op. Without it `asAuthHeaderFactory()` could not plug into its own stated integration point. Rust adds a separate `as_async_auth_header_factory()` rather than changing the existing signature; Python stays synchronous because its `httpx` path already is.

### Spec amendments made during implementation

Each was found by an SDK implementer and is recorded in `device-auth.md` with its date and reason:

- **The cross-origin endpoint rule contradicted itself and the corpus.** It said discovered endpoints "SHOULD share the issuer's origin" and that a cross-origin one "is rejected rather than followed" — `SHOULD` and "rejected" cannot both hold, and case 029 requires such an endpoint to be used. Resolved: `https://` is a hard error; a different origin is a warning and is followed. Rejecting breaks providers that host the token endpoint on a separate host, and the document's authority is already established by fetching from the issuer's own well-known path over TLS plus a verbatim issuer comparison.
- **`classify_error`'s invocation rule was never stated.** "Always call, alias wins" and "call only when unresolved" produce identical results for a pure hook — so both pass a corpus that does not pin it, while differing observably for a hook with side effects. Now: consulted only when the identifier is still unresolved.
- **The injected-clock table named two clocks; there are three.** The monotonic clock measures the polling deadline; a separate wall clock computes `expires_at`. Case 015 pinned the wall clock while the prose never named it, so three SDKs would have named it three ways.
- **Poll-loop ordering is now normative** (sleep → check deadline → poll). The corpus pinned it; the prose did not, and it is the most likely off-by-one in the implementation.
- **The store key was undefined when no `issuer` is configured** — a first-class path, since explicit endpoints are supported. Now falls back to the token endpoint.
- **`error_aliases` does not apply on the refresh path.** Otherwise the one alias the spec explicitly describes (`invalid_grant` → `expired_token`) would disable the one terminal rule protecting the store.
- **`on_user_code` receives the *effective* deadline, never null.** Found as a live three-way divergence after all three landed: Python passed the 900-second fallback, TypeScript and Rust passed the absent wire value.
- **`client_secret_basic` form-urlencodes before base64** (RFC 6749 §2.3.1). Raw concatenation is a real defect — a secret containing `+` reaches the server as one containing a space.

### Corpus cases that could not fail, and were rewritten

Recorded rather than quietly fixed, because a case that cannot fail reports coverage that does not exist:

- **045 and 046** were vacuous. 045 supplied no error body, so a harness inventing a *standard* identifier resolved it before the hook was ever reached; 046 used `access_denied`, which resolves without the hook, so it passed whatever the hook did. Both now carry a deliberately unresolved identifier.
- **054 and 055** both omitted `device_authorization_endpoint`, so discovery failed whether or not the issuer check fired — an implementation that normalises the issuer before comparing passed both. Found by mutation testing in the TypeScript SDK.
- **058 and 061 were written as exact-byte assertions and rewritten as round trips.** The three languages' form encoders disagree on space, `*` and `~`, and every one of those spellings decodes identically — so pinning one would have forced two SDKs to hand-roll an encoder for no functional gain. This is the repository's own ownership rule applied rather than assumed: byte-equivalence is for formats two consumers must compare, and the consumer here is a third-party authorization server that decodes.

---

## [0.12.0] - unreleased

### Added

- **`BindingLoader.load` gains `pattern`** ([#18](https://github.com/aiperceivable/apcore-toolkit/issues/18)) — apcore 0.30 made `bindings.pattern` a canonical config default, but the toolkit loader hardcoded the value and had no parameter through which a caller could honour a configured one. Any consumer needing the loader's *return value* rather than apcore's registration side effect (e.g. `apexe`, which builds its own `CliModule` from each descriptor) silently dropped the key. `load` now takes the resolved pattern; **the loader takes a value and does not read `Config`**, keeping the pure-data layer dependency-free and leaving resolution with the caller that actually holds a `Config`. Python and TypeScript gain an optional parameter; Rust gains `load_with_pattern` and keeps `load`'s arity, mirroring apcore's own `load_binding_dir` / `load_binding_dir_with_config` two-tier shape. See [`docs/features/binding-loader.md`](docs/features/binding-loader.md#pattern-matching).
- **`conformance/fixtures/binding_pattern.json`** — 43 shared cases (5 `validate`, 27 `match`, 11 `select`) pinning the matcher, the rejected patterns, how `pattern` composes with `recursive`, and the filesystem-entry rules (directories never match; the file-type check follows symlinks, so a symlinked binding file is selected while a symlinked directory is neither selected nor descended into; a dangling link is skipped, not read). The spec carries the matching **algorithm** in pseudocode, not just the syntax, because three independent implementations converge only if the algorithm is fixed: the two-pointer single-star-backtracking match, over Unicode code points, in bounded time.

### Changed

- **`docs/scope.md` — the "token management" exclusion is disambiguated.** It previously read as an undifferentiated exclusion covering anything token-shaped. It now separates `apflow`'s two token concerns (LLM context-token budgeting; issuing `apflow`'s own service JWTs) from third-party credential *acquisition*, and admits the latter to the toolkit on the narrow grounds that the toolkit already ships the `auth_header_factory` seam it fills. This unblocks [#17](https://github.com/aiperceivable/apcore-toolkit/issues/17) Phase 0.
- **`docs/features/device-auth.md` — status PROPOSED → ACCEPTED.** Both blocking prerequisites are resolved and all six open questions have recorded verdicts. Still **no code**: Phase 2 is ~2000 LOC across three SDKs and its conformance corpus, the gating artifact, is not written. Notably, V1 grant scope was narrowed from the document's own recommendation (device flow + manual code entry + PKCE) to device flow only, with the `Grant` interface retained so the wider scope stays one implementation away rather than a rewrite.
- **`docs/features/tui-view-model.md` — the five open questions have recorded verdicts**, checked against what actually shipped in 0.11.1 rather than what was recommended. Four were adopted as written. The fifth was not: the builder resolves `display.alias > module_id`, not the richer `display.cli.alias > display.alias > canonical_id` chain the proposal asked for, so a binding setting only `display.cli.alias` still renders differently in `apcore-cli` than in the view model. Recorded as follow-up instead of closed quietly.

### Fixed

- **`docs/features/output-writers.md` — the `auth_header_factory` invocation contract is documented**, verified against the shipped 0.11.1 sources in all three SDKs: invoked **once per outbound request** from inside `execute` (so a rotating credential works without a writer change), and **synchronous** in all three (so an async credential source cannot refresh inside it). This resolves [#17](https://github.com/aiperceivable/apcore-toolkit/issues/17) Phase 1.
- **`docs/features/binding-loader.md` — three TypeScript examples were wrong.** The `load` examples showed `loader.load("users.binding.yaml", { strict: true })` and described `recursive` as `{ recursive: true }`; `load`'s 2nd and 3rd parameters have always been positional booleans, and `BindingLoadOptions` belongs to `loadData` / `parseBindingDocument`. Confirmed by compiling the documented line (`TS2345`). The failure mode was self-concealing outside TypeScript: `{ strict: true }` is truthy, so a JavaScript caller following the docs got strict mode by accident.
- **`docs/features/binding-loader.md` — the Rust error enum was documented as 7 variants in two places**; `InvalidPattern` makes 8.

### Known cross-SDK issues found while specifying this

- **apcore's own three SDKs interpret `bindings.pattern` three different ways.** `apcore-python` uses `Path.glob` (full glob incl. character classes); `apcore-rust` strips a leading `*` and suffix-matches; `apcore-typescript` does `pattern.replace('*','')` then `endsWith`. All three agree on the default `*.binding.yaml` and on essentially nothing else — `data*.yaml` matches `data1.yaml` in Python only. This is the same latent-divergence shape as #18, one layer up. The toolkit therefore specifies its own matcher and pins it with a fixture rather than inheriting an accident. To be filed upstream against `apcore`.
- **The corpus's `empty_pattern` / `path_separator` identifiers are corpus vocabulary, not public API.** Python and TypeScript expose a single `BindingLoadError` with no machine-readable discriminator for any failure kind, so a caller cannot read the identifier off a thrown error; Rust callers can match `InvalidPattern`. A cross-SDK error-code field is worth its own issue.
- **Non-UTF-8 file names are handled inconsistently in `apcore-toolkit-rust`** — the recursive branch converts with `to_string_lossy` (invalid bytes become U+FFFD, which can then match `*`), the flat branch uses `to_str()` and skips. Pre-existing, not covered by the fixture.

## [0.11.0] - 2026-09-06

Patch release across Python, TypeScript, and Rust: raises the required apcore floor to `0.30.0`.

Feature release across Python, TypeScript, and Rust: ships two previously-proposed Tier-1 toolkit features — **OpenAPI Scanner** and **TUI View Model** — plus two blocking `HTTPProxyRegistryWriter` defect fixes in `apcore-toolkit-rust`. `docs/features/device-auth.md` remains a design proposal and does not ship in this release.

### Added

- **OpenAPI Scanner** (`OpenAPIScanner`, `derive_module_id`, `load_spec`) — turns a whole OpenAPI 3.0/3.1 document into a `ScannedModule` list, one module per operation, with a byte-identical `module_id` derivation across all three SDKs. Built on the already-shipped `extract_input_schema` / `extract_output_schema` / `deep_resolve_refs` primitives. See [`docs/features/openapi-scanner.md`](docs/features/openapi-scanner.md). Verified against the published Swagger Petstore OpenAPI 3.0 reference spec — byte-identical `ScannedModule` output across all three SDKs, in addition to the shared 24-case conformance corpus (`conformance/fixtures/openapi_scan.json`).
- **TUI View Model** (`TuiViewModel`, `modules_to_view_model`, `format_view_model`) — lifts the module-list view shape (columns, rows, filter/sort/color-by-tag semantics) into a Tier-1 byte-equivalent structure, ending three independent per-SDK table-rendering implementations. See [`docs/features/tui-view-model.md`](docs/features/tui-view-model.md). Asserted byte-identical via the shared 11-case conformance corpus (`conformance/fixtures/view_model.json`), and cross-checked end-to-end against the Petstore-scanned modules above (identical grouped/filtered/sorted output across all three SDKs).
- **`output-writers.md` § `metadata` Contract** — documents the previously-unspecified `http_method` / `url_path` metadata keys `HTTPProxyRegistryWriter` reads, including the uppercase requirement and the method-driven body-vs-query rule.

### Changed

- **Required apcore floor raised to 0.29.0** across all three SDKs. apcore 0.29.0 is approval- and ACL-layer work plus a large documentation-conformance pass. `ApprovalRequest` gains `caller_id` and `action` (additive, spec v1.32.0 decision D-03); `CancelToken.raise_if_cancelled()` lands in Python and TypeScript as an additive alias for `check()`; and the ACL pattern array (`callers` / `targets`) has its shape closed at every entry point — an empty array, `["$or"]`, `["$not"]` and the multi-operand `["$not", p1, p2]` are now refused rather than silently matching nothing, which under `default_effect: allow` had been permitting the very call the rule named ([apcore#112](https://github.com/aiperceivable/apcore/issues/112)). Per-SDK it additionally carries `AsyncTaskManager.startReaper()` returning a `Promise` in TypeScript, and in Rust `APCore::on`/`off` returning `Result` plus `ACLRule` becoming `#[non_exhaustive]`. None of it touches the Registry / Module / annotations surface this toolkit uses in any of the three languages — confirmed per-SDK by grepping every apcore import in `src/` against the symbols apcore 0.29.0 changed. No code or API changes; all three test suites pass unmodified against apcore 0.29.0 (Python 760, TypeScript 657, Rust 477 lib + 13 integration + 8 doc).

- **Required apcore floor raised to 0.30.0** across all three SDKs. apcore 0.30.0 is confined to the `Config`/`BindingLoader` layer: the §9.2.2 deprecation-warning cadence is now spec-normative (once per configuration load, never once per process — TypeScript additionally gains `Config.projectRoot`, `userLevelConfigPaths()`, and a new out-of-tree-relative-path deprecation notice building toward it); a set-but-empty path-typed `APCORE_*` override (e.g. `APCORE_ACL_ROOT=`) is now discarded instead of silently resolving to the working directory; the missing-binding-directory error names a directory rather than a file; and `bindings.dir` / `bindings.pattern` become canonical defaults reachable from `Config.get()` in all three SDKs. None of it touches the Registry / Module / annotations surface this toolkit uses — confirmed per-SDK by grepping every apcore import in `src/` against the symbols apcore 0.30.0 changed: no `Config`, `BindingLoader`, `ACL`, `ApprovalRequest`, `CancelToken`, `ExecutionPolicy`, or `Executor` reference exists in any of the three toolkits. No code or API changes; all three test suites pass unmodified against apcore 0.30.0 (Python 798, TypeScript 680, Rust 485 lib + 13 integration + 8 doc).

### Fixed

- **`apcore-toolkit-rust` — `HTTPProxyRegistryWriter` rejected `HEAD`/`OPTIONS`/`TRACE` requests before any network call.** Only `GET`/`POST`/`PUT`/`PATCH`/`DELETE` were handled; OpenAPI-scanned modules using the other three recognised HTTP methods failed at execution time on Rust only. Fixed.
- **`apcore-toolkit-rust` — the post-substitution "unfilled path parameter" check used a narrower regex than extraction/substitution**, so a hyphenated path parameter (e.g. `{item-id}`) left unfilled by the caller was not flagged. Path-parameter *extraction and substitution* were already correct — this is a narrower, corrected diagnosis than the defect as originally filed; see the correction in `docs/features/openapi-scanner.md`. Fixed by reusing the same extraction function for both checks.

### Cross-SDK audit findings (fixed in this release)

A post-implementation line-by-line audit comparing all three SDKs' new code — beyond what the shared conformance corpus exercises — found and fixed:

- **TypeScript — `TuiViewModel`'s `Filter.annotations` silently excluded every module when filtering by `requires_approval` or `open_world`.** The filter takes the spec's snake_case flag names, but the installed `apcore-js` `ModuleAnnotations` type uses camelCase fields, so the raw property lookup always missed — a documented, valid usage path, not a malformed-input edge case. Fixed with an explicit name mapping.
- **Python — `OpenAPIScanner` mis-parsed a non-array `tags` value.** `list(tags or [])` silently character-split a string value (`"tags": "foo"` → `["f","o","o"]`) instead of degrading to an empty list, as TypeScript and Rust already did.
- Several lower-severity type-coercion divergences (a truthy-vs-strict `deprecated` boolean check, non-string `operationId`/`info.version`/`summary` leniency, Unicode- vs ASCII-only digit matching for 2xx response-status detection) were also found and aligned toward the stricter, more-correct behaviour across all three SDKs. None affect a valid OpenAPI document; all are gated behind malformed/non-conforming input.

### Docs

- `docs/features/tui-view-model.md` and `docs/features/openapi-scanner.md` status flipped from "PROPOSED — not implemented" to "IMPLEMENTED"; both moved from **Proposed Capabilities** to the main feature table in `docs/features/overview.md`; `mkdocs.yml` nav labels dropped the "(proposed)" suffix. `docs/features/device-auth.md` is unaffected and remains a proposal.

## [0.10.2] - 2026-09-01

Patch release, version-aligned across Python, TypeScript, and Rust. Bumps the required apcore floor to 0.28.0. apcore 0.27.0 and 0.28.0 are almost entirely ACL/Executor governance work (argument-scoped approval, `ACLRule.approval`/`approval_required`, `ExecutionPolicy.resolve()` call-site parameters, `Executor.governance_state()`/`governanceState()`, ACL effect/condition hardening) — none of it touches the Registry/Module/annotations surface this toolkit uses in any of the three languages (confirmed per-SDK via grep and cross-checked against the current apcore source). No code or API changes; all three test suites pass unmodified against apcore 0.28.0 (Python 719, TypeScript 617, Rust 453 lib + integration).

Note: apcore 0.28.0 also makes dict-declared `input_schema`/`output_schema` validation actually enforce (previously a no-op in the Python SDK). This toolkit's Python `HTTPProxyRegistryWriter` registers such modules but is never itself routed through a live apcore `Executor`, so nothing here changes — but a framework adapter that executes a dict-schema module built by this writer through apcore 0.28.0 will now see real validation where it previously saw none.

## [0.10.1] - 2026-07-14

Patch release, version-aligned across Python, TypeScript, and Rust. Bumps the required apcore floor to 0.26.0 to align the ecosystem on the 0.26.0 governance layer (Execution Policy, governance events, no-handler fail-loud) — additive, no breaking changes for this toolkit. No code or API changes; all three test suites pass unmodified (Python 719, TypeScript 617, Rust 453 lib + integration).

## [0.10.0] - 2026-07-07

Version-aligned feature release across Python, TypeScript, and Rust: a shared conformance verifier that guards registry writers against silently dropping behavioral annotations (which would disable approval/ACL gating), plus — in Python — a centralized writer so subclass adapters can no longer omit a field. Additive; no breaking changes. All three test suites pass (Python 719, TypeScript 617, Rust 453 lib + integration).

### Added

- **All SDKs — `assert_annotations_preserved` / `assertAnnotationsPreserved` conformance check.** A framework-agnostic helper (raises / throws / panics) that registers a scanned module and asserts its behavioral annotations (`requires_approval`, `destructive`) survive `get_definition`. Adapters import it into their own test suites so a writer that silently drops annotations — disabling approval/ACL gating — fails loudly. Exported from the package root in each SDK.

### Changed

- **Python — `RegistryWriter` field mapping is centralized.** `_to_function_module` no longer builds the `FunctionModule` inline; a new `_build_function_module` forwards **all** carried fields (including `annotations` and `display`). Subclass writers now override only the narrow hooks `_adapt_func` / `_build_input_schema` / `_build_output_schema` and can no longer omit a field. Behavior is unchanged for callers that do not override the hooks. (The TypeScript and Rust writers already preserved annotations — no change was required there.)

### Cross-SDK note

The motivating bug — dropped annotations silently disabling governance — existed only in two Python framework adapters (fixed in their own repos). The TypeScript (`nestjs-apcore`) and Rust (`axum-apcore`) adapters already preserved annotations; this release adds the shared guard across all three languages so no future adapter can regress unnoticed. A new exported symbol ⇒ MINOR; no breaking changes.

## [0.9.0] - 2026-06-23

Version-aligned bug-fix release across Python, TypeScript, and Rust. Each SDK fixes a distinct correctness defect surfaced by real framework integration against the apcore 0.24.0 runtime. No conformance-fixture changes; all three test suites pass (Python 713, TypeScript 614, Rust all green).

### Fixed

- **Python — OpenAPI parser crashed on integer status-code keys and explicit `null`.** `extract_output_schema` assumed string response keys, so django-ninja's integer keys (`{200: ...}`) raised `TypeError` in `re.match`/`sorted`; `null` responses/content/requestBody/properties/required raised `AttributeError`. Keys are now normalized to strings and null containers fall back to empty. (7 new regression tests.)
- **TypeScript — registry verification and async registration were broken against the real `Registry`.** `RegistryVerifier` called the nonexistent `getModule()` (apcore exposes `get()`), so `verify: true` always failed; and `RegistryWriter.write` did not `await` the async `register()`, risking unhandled rejections and premature verification. Both fixed.
- **Rust — registry writers required `&mut Registry`, blocking framework integration.** apcore's `Registry` is interior-mutable (`register(&self, …)`) and executors only expose `Arc<Registry>`. `RegistryWriter::write` and `HttpProxyWriter::write` now take `&Registry` (source-compatible — `&mut T` coerces to `&T`).

### Cross-SDK note

These were independent per-language defects, not a shared-contract change — each SDK's stable surface is otherwise unchanged. The Rust signature relaxation is the only API-surface change and is backward-compatible at existing call sites.

## [0.8.1] - 2026-06-12

Aligned release across Python, TypeScript, and Rust. Bumps the required apcore runtime to 0.24.0 (skipping 0.23.0 — no in-scope changes). No changes to toolkit API surface or conformance fixtures.

### Changed

- **Required runtime bumped to apcore 0.24.0** — `docs/getting-started.md` prerequisites updated from `apcore 0.22.0+` to `apcore 0.24.0+`. Python constraint bumped from `apcore>=0.22.0` to `>=0.24.0`; TypeScript from `apcore-js>=0.22.0` to `>=0.24.0`. Rust SDK updated to `apcore = "0.23"` — apcore 0.24.0 is not yet published to crates.io; a follow-up patch will advance it to `"0.24"` once indexed. Toolkit's stable surface (`ModuleAnnotations`, `Registry`, `ModuleExample`, `parse_docstring`, `Module`, `Context`, `errors`, `annotationsFromJSON`/`annotationsToJSON`) is unchanged by the 0.22 → 0.24 delta; all three SDKs pass full test suites without code changes.

  **apcore 0.23.0 changes *in scope* for the toolkit:**

  - None. apcore 0.23.0 adds AI error-recovery metadata to `ModuleError` (the `user_fixable` / `ai_guidance` policy) and rewrites `CircuitBreakerMiddleware` to a rolling-window model. Neither touches the toolkit's stable surface — the toolkit does not instantiate `CircuitBreakerMiddleware` or construct `ModuleError` with recovery metadata fields.

  **apcore 0.24.0 changes *in scope* for the toolkit:**

  - None. apcore 0.24.0 scopes `ToggleState` to per-`APCore`-instance and adds conformance coverage for agent governance and toggle isolation. The toolkit does not use `ToggleState`, `APCore`, or `ACL` directly. The TypeScript `Registry.unregister()` drain-state fix (A-D-001) is downstream of `RegistryWriter`'s path — `RegistryWriter` only calls `register`, not `unregister`.

  **apcore 0.23.0 changes *out of scope* for the toolkit:**
  - `CircuitBreakerMiddleware` rewritten to rolling-window error-rate model (breaking constructor: `failure_threshold`/`success_threshold` removed, `open_threshold`/`window_size`/`min_samples` added). Toolkit does not use circuit-breaker middleware.
  - `A2ASubscriber` no longer retries 4xx responses. Toolkit does not subscribe to apcore events.
  - DLQ `original_event` now nests `module_id`/`timestamp` under `metadata`. Toolkit does not emit apcore events.
  - AI error-recovery metadata (`user_fixable`, `ai_guidance`) populated in `ModuleError` at the framework level. Toolkit re-raises `ModuleError` from `RegistryWriter` but does not construct them with recovery metadata.

  **apcore 0.24.0 changes *out of scope* for the toolkit:**
  - Per-instance `ToggleState` isolation (all three SDKs, #71). Toolkit does not construct `APCore` instances.
  - `Registry.unregister()` drain-state fix (TypeScript A-D-001). Toolkit's `RegistryWriter.write()` calls `register` only.
  - Sensitive-key redaction recurses into arrays (TypeScript/Rust A-D-003). Toolkit does not call redaction utilities.
  - Env value coercion and namespace fallback alignment (TypeScript A-D-008, Rust A-D-007/009). Toolkit does not read apcore `Config`.
  - Before-middleware double `on_error` fire (TypeScript A-D-011), ACL `removeRule(null)` conditions fix (A-D-016), `error details` key casing (`camelCase` → `snake_case` A-D-019). Toolkit does not install middleware or manage ACL rules.
  - Rust: schema type coercion enabled by default; `SchemaValidator` returns coerced value. Toolkit conformance fixtures (`format_csv`, `format_jsonl`, `display_resolve`) do not exercise schema validation.
  - Rust: registry event delivery-or-DLQ guarantee (A-D-013), middleware `on_error` exact-once fix (A-D-010/012), mid-stream error recovery (A-D-015), `APCore.on()`/`events()` shared event bus fix (D1-011).

### Documentation

- **`docs/getting-started.md`** — prerequisites updated from `apcore 0.22.0+` to `apcore 0.24.0+` across all three language tabs.
- **`README.md`** — SDK badges updated to 0.8.1; apcore badge updated to 0.24.0+; version compatibility snapshot updated to 0.24.0 / apcore-toolkit 0.8.1 (dated 2026-06-12).

## [0.8.0] - 2026-05-28

Aligned release across Python, TypeScript, and Rust. Bumps the required apcore runtime to 0.22.0 and versions all three SDKs to 0.8.0. No changes to toolkit API surface or conformance fixtures.

### Changed

- **Required runtime bumped to apcore 0.22.0** — `docs/getting-started.md` prerequisites updated from `apcore 0.21.0+` to `apcore 0.22.0+`. Rust SDK dependency updated from `apcore = "0.21"` (caret: `>=0.21.0, <0.22.0`) to `apcore = "0.22"` — the prior caret constraint excluded 0.22.0 entirely, making this a required fix for users on the latest apcore. Python and TypeScript constraints bumped from `apcore>=0.21.0` / `apcore-js>=0.21.0` to `>=0.22.0` for consistency. Toolkit's stable surface (`ModuleAnnotations`, `Registry`, `ModuleExample`, `parse_docstring`, `Module`, `Context`, `errors`, `annotationsFromJSON`/`annotationsToJSON`) is unchanged by the 0.21 → 0.22 delta; all three SDKs pass full test suites without code changes.

  **apcore 0.22.0 changes *in scope* for the toolkit:**

  - **`Registry.register()` concurrent same-ID rejection (D-65)** — 0.22.0 rejects the second concurrent caller immediately with `InvalidInputError(code=DUPLICATE_MODULE_ID)` instead of resolving via lock-ordering. `RegistryWriter.write()` registers modules sequentially and is unaffected under normal usage; callers who concurrently invoke `write()` on overlapping module sets will now see deterministic immediate rejection instead of a race outcome. The `idempotent: false` property note in `docs/features/output-writers.md` updated to reflect the 0.22.0 behavior.

  - **Registration Ordering Invariants (D-65)** — modules are now guaranteed not to become visible to `registry.get()` / `registry.list()` until all `on_load` callbacks have completed. `RegistryWriter.write()` returns only after all registrations finish; callers can now rely on modules being immediately discoverable on return.

  - **`StreamingModule` explicit interface** — apcore 0.22.0 promotes `StreamingModule` from duck-typed to an explicit Protocol / interface / trait. `RegistryWriter` now detects and handles streaming targets across all three SDKs:
    - **Python / TypeScript** — if the resolved target has an async `stream()` method, `RegistryWriter._to_function_module` / `_toFunctionModule` creates a streaming adapter (`_StreamingFunctionModule` / `StreamingFunctionModule`) that satisfies `isinstance(m, StreamingModule)` / `isStreamingModule(m)` without the duck-typing deprecation warning. The adapter's execute path delegates to the target's `execute()` method (or callable form); the stream path delegates to `target.stream()`.
    - **Rust** — new `StreamingHandlerFactory` type and `RegistryWriter::with_streaming_handler_factory(factory)` builder. When the factory returns `Some(stream_fn)` for a target, `RegistryWriter` registers a `StreamingFunctionModule` that implements both `Module` and `StreamingModule` (`as_streaming()` returns `Some(self)`).
    - **Fallback (all SDKs)** — if `annotations.streaming=True` but no streaming implementation is found (no async `stream()` method / factory returns `None`), `RegistryWriter` logs a WARNING and clears the `streaming` flag to prevent `StreamingInterfaceError`. The module is registered as a non-streaming `FunctionModule`.
    - Three new streaming unit tests per SDK. `docs/features/output-writers.md` `Contract: RegistryWriter.write` streaming admonition updated from a limitation warning to usage documentation.

  **apcore 0.22.0 changes *out of scope* for the toolkit** (not consumed by the toolkit's stable surface):
  - `Context.create()` signature unified — `executor` and `caller_id` parameters removed, `cancel_token` promoted as first-class parameter (D-24). Toolkit does not call `Context.create()`.
  - Canonical event name renames — `apcore.module.registered` → `apcore.registry.module_registered`, `apcore.module.unregistered` → `apcore.registry.module_unregistered`, `apcore.error.threshold_exceeded` → `apcore.health.error_threshold_exceeded`, `apcore.latency.threshold_exceeded` → `apcore.health.latency_threshold_exceeded`. Toolkit does not emit or subscribe to apcore events.
  - `TaskStore` fully-async requirement (D-17), `cancel()` real-interrupt requirement (D-18), `call_with_trace` error semantics (D-19), cancellation short-circuit (D-20), cancel token two-point check (D-21), `MiddlewareChainError` unwrap (D-22), shallow-copy `get_status` / `list_tasks` (D-23).
  - Rust `TraceParent` gains `tracestate: Vec<(String, String)>` field; `ContextBuilder::tracestate()` removed. Toolkit does not construct `TraceParent` or use `ContextBuilder`.
  - `A2ASubscriber` now retries 3× on 5xx/timeout by default; reserved namespace query `Config.reserved_namespaces()`; Duplicate Middleware Detection normative SHOULD.

### Documentation

- **`docs/features/output-writers.md`** — `Contract: RegistryWriter.write` updated: `idempotent` property corrected to reflect 0.22.0 immediate `DUPLICATE_MODULE_ID` rejection; new `StreamingModule` warning admonition; new Registration Ordering tip admonition.
- **`docs/getting-started.md`** — prerequisites updated from `apcore 0.21.0+` to `apcore 0.22.0+` across all three language tabs.

## [0.7.0] - 2026-05-12

Aligned release across Python, TypeScript, and Rust. Adds **byte-equivalent tabular formatters** (`format_csv`, `format_jsonl`) for cross-SDK consistency, with shared conformance corpus and updated formatting spec. Lifts a class of bugs that previously affected every apcore-cli SDK's `--format csv` / `--format jsonl` implementation. Includes the post-audit cross-SDK contract reconciliation pass (D10 contract-parity findings).

### Added

- **`format_csv(rows, *, bom=False)` and `format_jsonl(rows)`** — byte-equivalent tabular data formatters in `apcore_toolkit.formatting.tabular` (Python) / `src/formatting/tabular.ts` (TypeScript) / `src/formatting/tabular.rs` (Rust). Re-exported at the top-level package / crate root. All three SDKs emit identical bytes for the same input, asserted via the new shared conformance corpus at `conformance/fixtures/format_csv.json` (15 cases) and `format_jsonl.json` (8 cases).
- **Tabular Formats** section in `docs/features/formatting.md` (~90 lines) — full contract for header derivation (union of keys, insertion-order), cell serialization (canonical compact JSON for nested values, whole-number floats as integers matching JS `JSON.stringify`, NaN/Inf → empty cell / null), RFC 4180 CRLF for CSV / LF for JSONL, BOM option, number portability constraints (`Number.MAX_SAFE_INTEGER`), and the explicit YAML deferral note.

### Changed

- **`apcore-toolkit-rust`** — `serde_json` `preserve_order` feature enabled. Required for canonical insertion-order key emission, matching Python `dict` and JS object key ordering. Transitively affects all `serde_json::Map` instances in the dependency tree; downstream code that relied on alphabetical iteration must re-sort explicitly (the toolkit's `render_annotations_table` already does so to honour the `formatting.md` alphabetical contract).

### Why

Per-SDK reimplementations of csv/jsonl had accumulated divergence over time: apcore-cli-python emitted Python repr `{'k': 'v'}` for nested values (invalid JSON); apcore-cli-typescript derived headers from `Object.keys(rows[0])` only (silent data loss on heterogeneous rows — surfaced via aisee-cli); apcore-cli-rust used `\n` instead of CRLF. The spec MUST language couldn't enforce conformance on downstream consumers (e.g. aisee-cli) that reimplemented their own emission. Lifting csv/jsonl into the toolkit replaces those independent failure modes with a single conformance-tested reference. See `apcore-cli/docs/tech-design.md` ADR-09 for the full tier-split rationale (byte-equivalent toolkit-delegated vs SDK-native presentation vs trivial stdlib).

### Notes

- **YAML is intentionally not in this tier yet.** Each idiomatic YAML library (PyYAML, js-yaml, serde_yaml_ng) emits different forms even for identical input. Byte-equivalence requires a custom emitter, which is deferred. YAML remains SDK-native (Tier 2) and may differ across languages.
- Integers exceeding `Number.MAX_SAFE_INTEGER` (`2^53 - 1`) are not portable across SDKs; callers should serialize them as JSON strings.

### Downstream impact

Downstream adapters (`apcore-cli`, `apcore-mcp`, `apcore-a2a`, and product CLIs like `aisee-cli`) should delete any local csv/jsonl emission logic and import `format_csv` / `format_jsonl` from the toolkit. The shared conformance corpus enforces that every downstream produces identical output going forward.

### Spec — cross-SDK contract reconciliation (post-audit)

The 2026-05-12 cross-SDK audit (`/apcore-skills:audit --scope toolkit`) flagged six D10 contract-parity divergences between spec text and the three implementations. Spec is the authority for cross-SDK behaviour, so the following spec sections were updated to either narrow the contract to match uniform implementation behaviour, or document an intentional language-specific divergence:

- **`docs/features/binding-loader.md` § Loose-mode wrong-type policy** (D10-001) — added a new subsection declaring the cross-SDK policy for non-required wrong-type optional fields (`input_schema`, `output_schema`, `tags`): strict mode raises `BindingLoadError`; loose mode warns and coerces to the empty default. All three SDKs follow this policy after the Python implementation was adjusted (see `apcore-toolkit-python` 0.7.0 entry).
- **`docs/features/output-writers.md` § Contract: get_writer** (D10-002, D10-003) — clarified that canonical formats (`"yaml"`, `"python"`/`"typescript"`, `"registry"`) are matched **case-sensitively** in all three SDKs. The HTTP-proxy aliases (`"http-proxy"`, `"http_proxy"`, `"httpproxy"`) are now declared as a **cross-SDK guarantee** — accepted by Python, TypeScript, and Rust. Rust additionally matches the alias set case-insensitively.
- **`docs/features/openapi.md` § Contract: resolve_ref** (D10-004) — added a "Language-specific hardening" note documenting the TypeScript-only `__proto__` / `constructor` / `prototype` JSON-pointer-segment guard (`PROTO_DENY_LIST` in `binding-parser.ts`). Python and Rust are not vulnerable to the same prototype-pollution attack and walk those segments like any other key.
- **`docs/features/formatting.md` § Contract: format_module / format_modules** (D10-005) — clarified that Rust's typed `ModuleStyle` / `SchemaStyle` / `GroupBy` enums make the unknown-style branch compile-time-impossible. `Err(FormatError)` covers only runtime schema-shape failures (e.g. `FormatError::SchemaNotObject`), not unknown style values.
- **`docs/features/display-overlay.md` § Contract: DisplayResolver.resolve** (D10-006) — narrowed the `### Errors` block from "invalid binding format or conflicting resolution options" to "MCP alias validation only", matching the uniform implementation behaviour across all three SDKs (warn-and-continue on invalid binding-data shape; raise only on alias collisions or shadowing).


## [0.6.0] - 2026-05-07

Aligned release across Python, TypeScript, and Rust. Adds surface-aware formatters (LLM markdown / SKILL.md / CLI table-row / JSON), declares the canonical HTTP-method → annotations mapping, and bumps the required apcore runtime to 0.20.0.

### Changed

- **Required runtime bumped to apcore 0.21.0** — `README.md` badge and `docs/getting-started.md` prerequisites updated from `apcore 0.19.0+` to `apcore 0.21.0+`. Toolkit only consumes apcore's stable surface (`ModuleAnnotations`, `Registry`, `ModuleExample`, `parse_docstring`, `Module`, `Context`, `errors`, `jsonSchemaToTypeBox`, `annotationsFromJSON`/`annotationsToJSON`); none of those touched the 0.19 → 0.21 changes (which were focused on `OverridesStore`, `SubscriberFactory`, `StepMiddleware`, schema/system-modules hardening, plus `CircuitOpenError` → `CircuitBreakerOpenError` rename and `TaskStore.put` → `save` rename — all out of toolkit's reach). All three SDKs continue to pass full test suites (Python 585 / TypeScript 490 / Rust 380).

### Added

- **Surface-aware formatters** (#13) — `format_module`, `format_schema`, `format_modules` for rendering `ScannedModule` and JSON Schema for specific consumer surfaces. Four styles: `markdown` (LLM context), `skill` (drop-in `.claude/skills/<id>/SKILL.md` or `.gemini/skills/<id>/SKILL.md` body with minimal `name` + `description` frontmatter — no vendor-specific extensions), `table-row` (CLI listing), `json` (programmatic). Replaces the ad-hoc `to_markdown(asdict(module))` pattern downstream surfaces were using. Spec: `docs/features/formatting.md`.
- **Annotation-table cross-SDK alignment** (#13 follow-up) — `format_module(style="markdown" | "skill")` now emits the `## Behavior` fact table identically across Python / TypeScript / Rust: only fields that differ from `ModuleAnnotations` defaults are listed, rows are sorted alphabetically by key, and bool values render as lowercase `true` / `false`. The `## Behavior` section is omitted entirely when every annotation field equals its default. This closes the byte-equality gap surfaced by the original cross-SDK verification step.

### Changed

- **`infer_annotations_from_method` canonical mapping** (#11) — spec now declares `HEAD` and `OPTIONS` MUST map to `readonly=true` (without `cacheable=true`), aligning with RFC 9110 §9.2 safe-method semantics. Closes a Python/TypeScript ↔ Rust divergence where Rust already returned `readonly=true` for these methods while Python and TypeScript returned default annotations. Spec: `docs/features/scanning.md`.

### Fixed

- **`HTTPProxyRegistryWriter` doc inconsistency** (#12) — `docs/features/output-writers.md` heading, Contract block, and `get_writer()` factory table now correctly list TypeScript alongside Python and Rust. `apcore-toolkit-typescript` has shipped `HTTPProxyRegistryWriter` since the `http-proxy` writer feature commit; the toolkit docs had not caught up. `docs/features/overview.md` SDK Parity table was already correct. The TypeScript build needs only Node 18+ for the global `fetch`; no extra install is required.
- **`Contract: get_writer` Inputs alignment** (sync follow-up) — `docs/features/output-writers.md` `Contract: get_writer` Inputs paragraph for TypeScript was still asserting "`http-proxy` is not available in TypeScript" (residue from the original #12 misdirection); corrected to list `"http-proxy"` / `"http_proxy"` and the `options.baseUrl` requirement. Rust paragraph clarified to surface `Err(OutputFormatError::Unknown)` rather than the imprecise "returns `None`".
- **`HTTPProxyRegistryWriter` constructor unit divergence** (sync follow-up) — added an admonition under the constructor docs noting that Python uses `timeout: float = 60.0` (seconds), Rust uses `timeout_secs: f64` (seconds, validated non-zero finite), and TypeScript uses `timeoutMs: number = 60_000` (milliseconds, default 60 s). Default behavior is identical; cross-language porters MUST translate units when passing an explicit value.

## [0.5.0] - 2026-04-21

Aligned release across Python, TypeScript, and Rust. Tracks apcore 0.19.0 features (expanded `ModuleAnnotations`, `display` field, declarative config spec).

### Added

- **`BindingLoader`** (three SDKs) — parses `.binding.yaml` files back into `ScannedModule` objects. Pure-data reader: no target import, no Registry side effects. Distinct from `apcore.BindingLoader` which does both. Enables verification, merging, diffing, and round-trip workflows.
  - Loose mode (default): only `module_id + target` required.
  - Strict mode: additionally requires `input_schema + output_schema`.
  - `spec_version` validated; missing/unsupported values WARN but do not fail.
  - Errors: `BindingLoadError` with `file_path`, `module_id`, `missing_fields`, `reason` (Rust: 7-variant `thiserror` enum — adds `FileTooLarge` (16 MiB per-file cap) and `TooManyFiles` (10,000-files-per-directory cap) safety variants).
- **`ScannedModule.display`** (three SDKs) — new optional top-level field holding the sparse display overlay for binding YAML persistence. Distinct from `metadata["display"]` (resolved form produced by `DisplayResolver`).
- **New feature doc**: `docs/features/binding-loader.md`.
- **`docs/features/display-overlay.md`** — new section explaining the sparse-overlay vs resolved-display distinction, and the round-trip flow.
- **`docs/features/output-writers.md`** — notes on conditional `display:` emission, `spec_version` stamping, and a Rust example.
- **`docs/features/overview.md`** — Binding Loader entry; tri-language parity note.
- **`mkdocs.yml`** nav — adds Binding Loader, Display Overlay, Convention Scanning entries.

### Changed

- **`YAMLWriter`** — emits top-level `display:` key only when `ScannedModule.display` is non-empty. Rust writer refactored from `json!` macro to `serde_json::Map` to support conditional keys.
- **`serializers.module_to_dict / moduleToDict / module_to_value`** — include `display` field.
- **Python `AIEnhancer._build_prompt`** — confidence template now built dynamically from `gaps`. When `annotations` is in gaps, requests per-field confidence for every validator key (`annotations.readonly`, `annotations.streaming`, `annotations.cache_ttl`, …). Previously hard-coded to `{"description": 0.0, "documentation": 0.0}`, which caused annotation-field confidence lookups to fall back to `0.0` and fail the threshold check — annotation enhancement silently never took effect.

### Dependencies

- **`apcore >= 0.19.0` / `apcore-js >= 0.19.0`** — picks up the expanded `ModuleAnnotations` (12 fields: adds `streaming`, `cacheable`, `cache_ttl`, `cache_key_fields`, `paginated`, `pagination_style`, `extra`), `FunctionModule.display`, and new binding/schema error types. No toolkit type changes were needed for annotations — reflection (Python), serde (Rust), and `annotationsFromJSON` (TypeScript) propagate new fields automatically.

### Fixed (Rust)

- **`output::registry_writer` / `http_proxy_writer`** — fixed pre-existing `ModuleDescriptor` initialization errors following the apcore 0.19.0 upgrade: adds required `display` field; updates `annotations` to `Option<ModuleAnnotations>`; fills all descriptor fields in `http_proxy_writer` (previously partial).

### Tests

| SDK | Δ tests | Total |
|-----|---------|-------|
| Python | +34 | 440 |
| TypeScript | +29 | 320 |
| Rust | +26 | 304 |

All round-trip tests (`YAMLWriter.write` → `BindingLoader.load`) verify end-to-end preservation of `display`, `annotations` (12 fields), `metadata`, `examples`, and schemas.

### Hardening (post-review)

A code-forge:review pass on the 0.5.0 delta produced a short list of cross-SDK inconsistencies; they are fixed as part of this same release:

- **Silent drop of malformed `display` values** — Python, TypeScript, and Rust all used to swallow non-mapping overlays without warning. All three SDKs now emit a WARN with the offending module_id and return `None`/`null`, matching the behaviour of `annotations`/`examples` parsers.
- **`ScannedModule.display` defensive copy** — Python's `YAMLWriter` used to emit a bare reference (`binding["display"] = module.display`); it now deep-copies, matching TypeScript's `structuredClone` and Rust's `.clone()`.
- **Python `ScannedModule.display` field position** — moved from the middle of the dataclass to the end, preserving positional construction compatibility with pre-0.5.0 callers.
- **Python `BindingLoader` polish** — `load()` gained `recursive: bool = False`; `read_text` forces UTF-8.
- **TypeScript `BindingLoader` polish** — `_parseExamples` uses `structuredClone`; `fs.statSync` failures distinguish `ENOENT` from other `errno` codes.
- **Rust `BindingLoader` polish** — `read_dir` per-entry errors are now surfaced as `BindingLoadError::FileRead` (previously silently discarded); `MissingFields`/`InvalidStructure` `Display` no longer leak `Some(...)` / `None` debug wrappers.

### Hardening (cross-SDK sync — post-audit)

A later cross-SDK sync pass surfaced semantic divergences between the three loaders/writers/scanners. These are fixed in 0.5.0 as well, bringing the shipped SDKs to behavioural parity where the spec now documents it:

- **`BindingLoader` strict-mode wrong-type rejection (Python, TypeScript)** — a required field is now rejected when absent, `null`, or of the wrong type (e.g. `module_id: 42`, `target: true`, empty-string `module_id`). Previously Python/TypeScript silently coerced wrong-type scalars via `str(value)`/`String(value)`, while Rust already rejected them; the same YAML now behaves identically in all three SDKs. The error reason widens from "missing required fields" / "missing or null required fields" to **"missing or invalid required fields"**, matching the Rust loader.
- **`BindingLoader` defensive deep-copy (Python, TypeScript)** — Python previously did a shallow `dict(raw_input_schema)` / `list(raw_tags)`; TypeScript's `_asRecord` returned a fresh outer `{}` but shared nested refs. Both now deep-clone `input_schema`, `output_schema`, and `metadata` (`copy.deepcopy` in Python, `structuredClone` in TypeScript). Rust's `Value.clone` was already deep. Caller mutation of a loaded `ScannedModule.input_schema.properties.id.type` no longer leaks back into the parsed YAML source graph.
- **`BaseScanner.deduplicate_ids` collision pre-scan (Python)** — Python's `deduplicate_ids` now pre-computes the set of original `module_id`s and bumps the suffix counter past any collision, matching the TypeScript and Rust algorithms. Input `[a, a, a_2]` now yields `[a, a_3, a_2]` instead of the previous buggy `[a, a_2, a_2]`.
- **`YAMLWriter` atomic writes (Python)** — Python now writes each binding file to `<name>.<pid>.tmp`, `fsync`s, then `os.replace`s onto the final path, matching the TypeScript (`tmp + renameSync`) and Rust (`tmp + sync_all + rename`) writers. A crash mid-write no longer leaves a partial YAML that `BindingLoader` fails to parse. A pre-write check also refuses to overwrite a symlink at the target path.
- **`resolve_target.allowed_prefixes` (Python)** — Python's `resolve_target` now accepts an optional `allowed_prefixes: list[str] | None` argument (forwarded from `RegistryWriter.write(..., allowed_prefixes=...)`). When set, imports outside the allowlist raise `PermissionError` *before* `importlib.import_module` runs. Mitigates arbitrary-code-execution via forged binding files (e.g. `target: "os:system"`). Parity with the TypeScript SDK's `allowedPrefixes` option, adapted to Python's module-name import model. Rust does not need this because `resolve_target` is parse-only and the `HandlerFactory` is the security boundary.

### Documentation

- **`flattenParams` removed from TypeScript docs + README** — the Python `flatten_pydantic_params` continues to ship, but TypeScript's advertised `flattenParams(func, zodSchema)` was never actually exported and its wrapper semantics are a no-op in TypeScript (native object-argument idiom already flat). `docs/features/pydantic.md` is now Python-only with an admonition explaining the TypeScript/Rust rationale.
- **`HTTPProxyRegistryWriter` docs corrected** — spec stripped `dry_run`/`verify`/`verifiers` from the `write` Contract (Python and Rust impls do not accept them); availability widened from "Python only" to "Python (always) and Rust (with `http-proxy` Cargo feature)".
- **Cross-SDK async contract for `RegistryWriter.write`** — spec now documents `async: false` for Python/Rust and `async: true` for TypeScript (TypeScript must `await resolveTarget` internally).
- **`run_verifier_chain` / `runVerifierChain`** — added a public Contract block in `docs/features/output-writers.md` (was public in all three SDKs but undocumented).
- **Binding Loader: Rust safety caps** — new "Safety Caps (Rust only)" section documenting the 16 MiB per-file and 10,000 files-per-directory limits. Rust `BindingLoadError` variant count corrected from 5 to 7 (adds `FileTooLarge`, `TooManyFiles`).
- **`ScannedModule.Fields`** — removed spurious `spec_version` row (it is a YAML document header stamped by `YAMLWriter`, not a field on the object); added `suggested_alias` (present in all three SDKs but previously undocumented).
- **`filter_modules` Contract errors** — corrected Python entry: raises `ValueError` (wrapping `re.error` via `from exc`), not `re.error` directly; TypeScript `SyntaxError`, Rust `regex::Error` unchanged.
- **Tri-language scope** — `docs/scope.md` updated from "dual-language" (Python + TypeScript) to tri-language parity. `docs/getting-started.md` gained a Rust prerequisites + installation tab. `docs/index.md` org URL corrected (`aipartnerup` → `aiperceivable`) and Quick Start link redirected to `getting-started.md`.
- **Stale TypeScript examples fixed** — `docs/features/openapi.md` (`new ScannedModule` → `createScannedModule`) and `docs/ai-enhancement.md` (`writer.write(modules, { outputDir })` → `writer.write(modules, "./bindings")`). Stale `apcore >= 0.14.0` in `getting-started.md` bumped to `0.19.0+`.
- **Rust README** — `Scanner` trait references corrected to the actual `BaseScanner<App>` (5 replacements in Core Modules table and two examples).

## [0.4.0] - 2026-03-23

### Added

- **`DisplayResolver`** (`apcore_toolkit.display`) — sparse binding.yaml display overlay (§5.13). Merges per-surface presentation fields (alias, description, guidance, tags, documentation) into `ScannedModule.metadata["display"]` for downstream CLI/MCP/A2A surfaces.
  - Resolution chain per field: surface-specific override > `display` default > binding-level field > scanner value.
  - `resolve(modules, *, binding_path=..., binding_data=...)` — accepts pre-parsed dict or a path to a `.binding.yaml` file / directory of `*.binding.yaml` files. `binding_data` takes precedence over `binding_path`.
  - MCP alias auto-sanitization: replaces characters outside `[a-zA-Z0-9_-]` with `_`; prepends `_` if result starts with a digit.
  - MCP alias hard limit: raises `ValueError` if sanitized alias exceeds 64 characters.
  - CLI alias validation: warns and falls back to `display.alias` when user-explicitly-set alias does not match `^[a-z][a-z0-9_-]*$` (module_id fallback always accepted without warning).
  - `suggested_alias` in `ScannedModule.metadata` (emitted by `simplify_ids=True` scanner) used as fallback when no `display.alias` is set.
  - Match-count logging: `INFO` for match count, `WARNING` when binding map loaded but zero modules matched.
- **New feature spec**: `docs/features/display-overlay.md`

### Tests

- 30 new tests in `tests/test_display_resolver.py` covering: no-binding fallthrough, alias-only overlay, surface-specific overrides, MCP sanitization, MCP 64-char limit, `suggested_alias` fallback, sparse overlay (10 modules / 1 binding), tags resolution, `binding_path` file and directory loading, guidance chain, CLI invalid alias warning and fallback, `binding_data` vs `binding_path` precedence.

### Added (Convention Module Discovery — §5.14)

- **`ConventionScanner`** — scans a `commands/` directory of plain Python files for public functions and converts them to `ScannedModule` instances with schema inferred from PEP 484 type annotations.
  - Module ID: `{file_prefix}.{function_name}` with `MODULE_PREFIX` override.
  - Description from first line of docstring (`"(no description)"` fallback).
  - `input_schema` / `output_schema` inferred from type hints.
  - `CLI_GROUP` and `TAGS` module-level constants stored in metadata.
  - `include` / `exclude` regex filters on module IDs.
- **New feature spec**: `docs/features/convention-scanning.md`

### Tests (Convention Module Discovery)

- 15 new tests in `tests/test_convention_scanner.py`.

---

## [0.3.1] - 2026-03-22

### Changed
- Rebrand: aipartnerup → aiperceivable

## [0.3.0] - 2026-03-19

### Added

- `deep_resolve_refs` / `deepResolveRefs` documented in OpenAPI Integration page
  with three-level resolution explanation
- `HTTPProxyRegistryWriter` section in Output Writers page with features,
  installation instructions, and usage example
- `"http-proxy"` format in `get_writer()` factory table with Language column
- `HTTPProxyRegistryWriter` and `enrich_schema_descriptions` added to README
  Core Modules table
- `modules_to_dicts()` / `modulesToDicts()` added to Serialization Utilities table
- "Choosing a Writer" table updated with `HTTPProxyRegistryWriter` row

### Fixed

- `resolve_schema` description corrected from "recursively resolves" to
  "single level only" — the recursive resolver is `deep_resolve_refs`
- Reference Resolution section rewritten with accurate three-level explanation
- Duplicate `## Reference Resolution` heading removed

---

## [0.2.0] - 2026-03-12

### Added

- `TypeScriptWriter` to Core Modules tables in README and docs index
- `"typescript"` format to `get_writer()` factory example (TypeScript only)
- `documentation` and `warnings` fields to `ScannedModule` Fields table
- Serialization Utilities section (`annotationsToDict`, `moduleToDict`)
  in Smart Scanning docs
- Changelog page for documentation site

### Fixed

- Updated `WriteResult` table in Output Writers docs with TypeScript naming
  and type columns
- Fixed `ai_guidance` → `metadata.ai_guidance` in AI Enhancement docs
- TypeScript examples: replaced `new ScannedModule({...})` with
  `createScannedModule({...})` (`ScannedModule` is an interface, not a class)
- TypeScript `YAMLWriter` and `TypeScriptWriter` `write()` call signatures
  corrected from `write(modules, { outputDir })` to `write(modules, outputDir, options?)`
- `filterModules` include pattern corrected from `RegExp` literal to `string`
  matching the actual TypeScript signature
- `resolveTarget` example now shows `await` (the function is async)
- `flattenParams` example now shows required `zodSchema` second argument
- OpenAPI import corrected from `"apcore-toolkit/openapi"` to `"apcore-toolkit"`
  (no sub-path export in TypeScript)
- TypeScript Registry import corrected from `"@anthropic/apcore"` to `"apcore-js"`
- Broken changelog links fixed (`docs/changelog.md` → root `CHANGELOG.md`)
- `docs/features/scanning.md` — TypeScript example `include` option type
  corrected to `string`, import updated
- `docs/index.md` — Python-only framing updated; toolkit is dual-language

---

## [0.1.0] - 2026-03-06

Initial release. Extracts shared framework-agnostic logic from `django-apcore`
and `flask-apcore` into a standalone toolkit package.

### Added

- `ScannedModule` dataclass — canonical representation of a scanned endpoint
- `BaseScanner` ABC with `filter_modules()`, `deduplicate_ids()`,
  `infer_annotations_from_method()`, and `extract_docstring()` utilities
- `YAMLWriter` — generates `.binding.yaml` files for `apcore.BindingLoader`
- `PythonWriter` — generates `@module`-decorated Python wrapper files
- `RegistryWriter` — registers modules directly into `apcore.Registry`
- `to_markdown()` — generic dict-to-Markdown conversion with depth control
  and table heuristics
- `flatten_pydantic_params()` — flattens Pydantic model parameters into
  scalar kwargs for MCP tool invocation
- `resolve_target()` — resolves `module.path:qualname` target strings
- `enrich_schema_descriptions()` — merges docstring parameter descriptions
  into JSON Schema properties
- `annotations_to_dict()` / `module_to_dict()` — serialization utilities
- OpenAPI utilities: `resolve_ref()`, `resolve_schema()`,
  `extract_input_schema()`, `extract_output_schema()`
- Output format factory via `get_writer()`
- 150 tests with 94% code coverage

### Dependencies

- apcore >= 0.9.0
- pydantic >= 2.0
- PyYAML >= 6.0
