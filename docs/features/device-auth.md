---
description: "PROPOSED (not implemented) RFC 8628 Device Authorization Flow client: a deterministic polling state machine, TokenSet lifecycle, and a portable TokenStore protocol — protocol only, no terminal UI."
---

# Device Authorization Flow — V1 Proposal

!!! info "Status: ACCEPTED — design settled, no code written"
    Both blocking prerequisites are resolved (2026-09-08) and every open
    question has a recorded decision, so this document is no longer a
    proposal awaiting a verdict — it is an accepted design awaiting
    implementation capacity. **Nothing ships against it yet.** It is the
    relocated, re-scoped form of a proposal originally filed against
    `apcore-cli` as "Standardized RFC 8628 (Device Authorization Flow) for
    CLI Authentication".

    | | |
    |---|---|
    | **Author** | apcore-toolkit maintainers |
    | **First drafted** | 2026-09-03 |
    | **Accepted** | 2026-09-08 |
    | **Target release** | 0.13.0 (earliest) — deliberately *not* bundled with the 0.12.0 `BindingLoader.pattern` work |
    | **Tracking issue** | [aiperceivable/apcore-toolkit#17](https://github.com/aiperceivable/apcore-toolkit/issues/17) |
    | **Depends on** | [`output-writers.md`](output-writers.md#auth_header_factory-invocation-contract) — the integration point, shipped, and its invocation contract now verified |
    | **Affects** | `apcore-cli-{python,typescript,rust}` (terminal UI half, separate PRs) · `apexe` (credential-baseline note, see [Downstream Impact](#downstream-impact)) |
    | **Requires** | ~~An amendment to [`scope.md`](../scope.md)~~ — **done**, see [Scope Reconciliation](#scope-reconciliation) |

!!! success "The gating artifact exists: `conformance/fixtures/device_auth.json`"
    Phase 1.5 is done. The corpus — **65 cases** — was written, validated
    against an executable reading of this document, and committed **before**
    any SDK started, exactly as `view_model.json` preceded the three
    `TuiViewModel` implementations. That ordering is not ceremony: the
    surfaces implemented first and reconciled later (`csv`/`jsonl` before
    0.7.0) produced three independent bugs, and this is the only
    credential-handling surface in the toolkit, where a cross-SDK divergence
    is a security bug rather than a formatting nit.

    Phase 2 (~2000 LOC across three SDKs) implements against it.

---

## Goal

Ship the **protocol half** of RFC 8628 Device Authorization Grant as a
reusable, framework-agnostic capability: the polling state machine, token
lifecycle (expiry, refresh, persistence), and a portable storage protocol.

The **presentation half** — displaying the user code, opening a browser,
rendering a spinner while polling — stays in `apcore-cli` and is reached
through callbacks. The toolkit never writes to a terminal.

This split is not invented here; it is the one the original issue itself
proposed ("core protocol logic within apcore-toolkit for platform-agnostic
reuse, while keeping the interactive UI components within apcore-cli"). This
document takes that split literally and specifies where the line falls.

## Non-Goals

- **No terminal output.** No `print`, no spinner, no colour, no browser
  launch. The toolkit emits events; the consumer renders them. A library that
  writes to stdout cannot be used by a daemon, a GUI, or a test.
- **No full Authorization Code flow with a loopback listener, and no client
  credentials, in V1.** Device flow is the grant the tracking issue asked for.
  Note that this boundary is **under active review**: an ecosystem survey found
  that headless AI tooling has standardised on manual code entry rather than
  device flow, so a narrow `authorization_code` variant may belong in V1 after
  all — see [Open Question](#open-questions) 5 and
  [Manual code entry](#manual-code-entry).
- **No authorization server.** This is a client. Nothing here issues tokens.
- **No JWT parsing, validation, or `Identity` construction.** See
  [Boundary: this produces tokens, not identities](#boundary-this-produces-tokens-not-identities)
  — this is the most important boundary in the proposal and is easy to get
  wrong.
- **No OS keychain implementation in the toolkit.** A portable file-backed
  store ships; platform keychains are a consumer concern. See
  [Token Storage](#token-storage).
- **No credential entry.** Username/password prompts are precisely what device
  flow exists to avoid.

## Scope Reconciliation

[`scope.md`](../scope.md#not-a-workflow-engine) **used to state** (amended 2026-09-08 per this section; the original wording is kept here so the reasoning below still reads):

> **Not a Workflow Engine** — Orchestration, token management, and multi-agent
> coordination belong to `apflow`, not the toolkit.

Read literally, "token management" excludes this proposal. And the ambiguity is
**genuine, not merely theoretical** — `apflow` demonstrably owns *both* things
that "token" can mean here:

| Meaning | Where it lives in `apflow` |
|---|---|
| **(a) LLM context-token budgeting** — cost governance, budgets, model downgrade chains | `apflow/src/apflow/governance/budget.py` (`TokenBudget`, `BudgetManager`); described as "Token budget management" in `apflow/README.md` and `docs/features/cost-governance.md` |
| **(b) JWT bearer tokens** — a CLI surface literally captioned "Token Management" | `apflow config gen-token` / `verify-token`, plus `apflow/src/apflow/api/auth.py` |

So the sentence cannot be waved away as obviously meaning (a).

The reading this proposal assumes is still **(a)**, for one structural reason:
the phrase appears in a list beside "orchestration" and "multi-agent
coordination" — all workflow-engine concerns — under a heading that reads
*"Not a Workflow Engine"*. Cost governance belongs to that set; credential
acquisition does not.

It is also worth noting that (b) is **not** what this proposal does either:
`apflow`'s JWT surface *mints and verifies* tokens an operator holds the key
for. This proposal *obtains* a token from a third-party authorization server
through a browser-based grant. Those are different sides of the credential
lifecycle, and neither overlaps the other.

**This must nevertheless be resolved explicitly before implementation**, in
whichever direction the maintainers choose:

- If the intent is (a) (expected): amend `scope.md` to say *"LLM
  context-token budgeting"*, removing the collision, and add OAuth credential
  handling to the toolkit's listed responsibilities.
- If the intent genuinely covers OAuth credentials: this proposal is out of
  scope for the toolkit and should be re-homed rather than implemented here.

Flagging this explicitly because a reviewer skimming `scope.md` would
reasonably reject this proposal on its current wording, and because silently
implementing against a contradicting scope document is how boundary documents
rot.

## Boundary: this produces tokens, not identities

The tracking issue's discussion suggests this flow "ultimately needs to
produce" an `apcore` `Identity`. **It does not, and it must not.**

| | Client side (this proposal) | Server side (not this proposal) |
|---|---|---|
| Runs in | CLI / desktop / device | The API being called |
| Holds | `TokenSet` — an opaque bearer credential | The verified claims |
| Produces | An authentication header | An `Identity` |
| Trusts | Nothing; it cannot verify its own token | The signature it verified |

`Identity` is the output of **verifying** a credential, and only the party
holding the verification key can produce one honestly. A client that parses
its own JWT and constructs an `Identity` from the unverified payload has
produced a self-asserted claim wearing the costume of a verified one — the
exact confusion that leads to a downstream consumer trusting attacker-supplied
`roles`.

So the toolkit's device-auth client treats the access token as **opaque**. It
does not decode it. It does not look inside for `exp`, `sub`, or `roles`; it
uses the `expires_in` value from the token response instead. `Identity`
construction stays where it belongs: in the server-side SDKs that validate
incoming requests — which is how the ecosystem already builds it, via
`ContextFactory` and the JWT verification paths in the MCP and A2A SDKs.

!!! warning "Related ecosystem gap, worth its own issue"
    Verifying the above surfaced something adjacent that this proposal does not
    fix and should not silently rely on: `apcore`'s protocol specification
    states that the *governance projection* MUST NOT be accepted from
    caller-supplied input, but carries **no equivalent rule for `identity`** —
    while `Context.deserialize` will happily reconstruct an `Identity`,
    including its `roles`, from unverified wire JSON.

    Nothing in this proposal exploits or depends on that. It is noted here
    because the "clients must not assert identity" principle this section
    relies on is currently a convention rather than a normative rule, and it
    belongs in `apcore`'s protocol spec rather than in a toolkit feature
    document. Recommend filing against `apcore`.

### Consequence for sequencing

The tracking issue raises a sequencing concern: that ecosystem gaps in
role/claim propagation into `Identity.roles` make a device-flow client
"lower-value than it looks", and that this should perhaps wait.

Given the boundary above, that concern is **largely decoupled from this
proposal**. The immediate and complete use case is:

> Obtain a token → put it in whatever authentication header the target API
> expects → call a remote HTTP API that already knows how to validate it.

That works today with any conforming authorization server, and requires
nothing of `Identity` at all. The server-side claim-propagation gaps are real
and worth their own issues, but they gate *server-side authorization
decisions*, not *client-side credential acquisition*. This proposal can ship
independently.

## Motivation

### What the ecosystem has today

Verified against the shipped CLI SDKs, so the gap this fills is concrete rather
than assumed:

| Capability | Status |
|---|---|
| Static API-key auth (`AuthProvider` — sets `Authorization: Bearer <key>`, handles 401/403) | **Shipped**, complete in all three `apcore-cli-*` SDKs |
| Encrypted credential storage (OS keyring with an AES-256-GCM fallback, `config_encryptor`) | **Shipped**, complete in all three, with tests |
| Interactive login, device flow, token refresh, token store | **Absent** — no `login` command, no OAuth code, no refresh logic anywhere in the CLI family |

So the ecosystem can already *carry* a credential; it has no way to
*obtain* one interactively. This proposal fills exactly that gap and nothing
else.

Two consequences worth carrying into the plan:

- **The keychain work is mostly done.** The `KeychainTokenStore` in Phase 3 is
  not a from-scratch integration — `config_encryptor` already solves OS keyring
  access plus an encrypted-file fallback in all three languages. It needs
  adapting to the `TokenStore` protocol, not writing.
- **The injection point already exists too.** `AuthProvider` is already the
  thing that sets the `Authorization` header for CLI requests, so a
  device-flow-backed credential slots into an established seam rather than a
  new one.

### Reality check: device flow's place in the AI ecosystem

A survey of how AI-ecosystem tooling actually authenticates (**2026-09-03**)
returned a result that argues against part of this proposal, and it belongs
here rather than buried in a risk table.

**RFC 8628 is essentially absent from this ecosystem:**

| Surface | Device flow support |
|---|---|
| A major model vendor's public API | Not documented |
| That vendor's own coding CLI | Supports `authorization_code` only. A tracked request notes it **ignores `device_authorization_endpoint` even when a server advertises it** — still unimplemented |
| **MCP authorization specification**, both the 2025-06-18 and 2026-07-28 revisions | **No mention of RFC 8628 at all.** Mandates OAuth 2.1 + PKCE + resource indicators |
| The MCP authorization-extensions repository | No device-flow extension |

**What headless environments actually do today** is one of two things, neither
of which is device flow:

1. **Print the authorization URL, have the user paste the returned code back
   into the terminal.** This is what shipping CLIs do for SSH sessions,
   containers, and WSL — a documented `--no-browser` style fallback.
2. **A long-lived token minted by a setup command** (one observed CLI issues a
   one-year token for CI use).

#### What this means for this proposal

It does **not** invalidate the work. Device flow remains the right answer for
genuinely input-constrained devices, and it is what the tracking issue asked
for. But three conclusions follow, and they change the plan:

- **Device flow must not be positioned as *the* login mechanism for
  `apcore-cli`.** The evidence says the ecosystem's headless default is
  paste-the-code, and a CLI that only offers device flow will hit providers
  that do not implement it.
- **The V1 architecture should be grant-pluggable**, with device flow as the
  first grant rather than as the abstraction itself. The polling state machine,
  `TokenSet`, `TokenStore`, refresh handling, and the provider-compatibility
  surface are all **grant-independent** — they are the reusable core, and
  scoping them as "the device-flow client's internals" would waste them.
- **The paste-the-code fallback should ship in V1**, not later. It is small
  (see [Manual code entry](#manual-code-entry)), it is what the ecosystem
  actually standardised on, and omitting it means the first real consumer has
  to build it themselves.

This is raised as [Open Question](#open-questions) 5, because whether to widen
V1's grant scope is a product decision rather than a technical one.

!!! note "MCP authorization is a separate, larger piece of work"
    The MCP specification mandates a materially different stack: OAuth 2.1,
    mandatory PKCE, **RFC 8707 resource indicators sent unconditionally on both
    the authorization and token requests**, RFC 9728 protected-resource
    metadata with 401-driven discovery, RFC 9207 issuer validation, and — as of
    2026-07-28 — Client ID Metadata Documents replacing the now-deprecated
    dynamic client registration.

    None of that is in scope here, and this proposal should not pretend
    otherwise. It matters because `apcore-mcp` exists and will eventually need
    it, so it deserves **its own proposal**. Two design notes worth carrying
    forward when that happens, both of which affect shared infrastructure:

    - MCP requires credentials to be **keyed by the issuing authorization
      server's `issuer` identifier**, and forbids reusing them when the AS
      changes. This proposal's [store key](#filetokenstore-path) is already
      `issuer + client_id`, so that requirement is satisfied by construction.
    - MCP's discovery is **401-driven**: the client makes an unauthenticated
      request, reads `WWW-Authenticate`, and only then discovers the
      authorization server. Credential acquisition therefore cannot be purely a
      pre-request step; a future MCP grant will need a challenge-handler entry
      point, not just `get_token()`.

### Precedent for this relocation

This is not a fresh re-scoping. `apcore-cli`'s own changelog already records
the decision, when both proposals were deferred out of its v0.8.0 scope:

> Both belong primarily in `apcore-toolkit` (with thin cli-side adapters),
> require their own RFCs… Tracked for v0.9+.

This document, and its sibling [`openapi-scanner.md`](openapi-scanner.md), are
those RFCs.

### The duplication being removed

Every apcore-ecosystem CLI that authenticates re-implements the same six
steps: request a device code, show the user code, poll with backoff, handle
`slow_down`, persist the token, refresh it before expiry. None of these are
hard individually; all of them have sharp edges that are easy to get subtly
wrong, and the failure modes are security-relevant rather than merely
annoying:

| Step | Common defect when hand-rolled |
|---|---|
| Polling | Fixed interval that ignores `slow_down`, getting the client rate-limited or banned |
| Backoff | Retrying immediately on `authorization_pending`, hammering the token endpoint |
| Expiry | Comparing against a local clock with no skew allowance, so a valid token is discarded or an expired one is sent |
| Storage | World-readable file at an ad-hoc path |
| Refresh | Concurrent refreshes from parallel invocations, each invalidating the other's rotated refresh token |
| Logging | Tokens echoed into debug logs |

Centralising this trades six chances to get it wrong per CLI for one shared,
conformance-tested implementation.

### Why the toolkit, not `apcore` core

`apcore` is depended on by every SDK and every framework adapter — axum,
flask, django, fastapi, nestjs, hono. None of them need a device-auth *client*:
a web framework receives tokens, it does not go through a browser-based grant
to obtain them. Putting an HTTP polling client, token storage, and refresh
logic into the package everything depends on imposes that weight on adapters
that will never call it.

`apcore-toolkit` already occupies exactly the right layer: it depends on
`apcore`, and it exists as the shared *capability* layer above it (Scanner,
Verifier, writers, output pipeline). An optional credential-acquisition
capability is the same shape as everything else already here.

### Why not `apcore-cli`

Nothing in RFC 8628 depends on a terminal. The flow is equally applicable to a
desktop app, a TUI, an IDE plugin, or a headless agent that surfaces the user
code some other way. Only the *presentation* is CLI-specific — and that part
does stay in `apcore-cli`.

## Architecture

```
   ┌────────────────────────── apcore-cli (Tier 2, per-SDK) ──────────────────────────┐
   │  spinner · user-code highlighting · `open` browser · "waiting…" copy             │
   └───────────────▲───────────── callbacks ─────────────▲───────────────────────────┘
                   │ on_user_code(...)                   │ on_poll(...)
   ┌───────────────┴─────────────────────────────────────┴───────────────────────────┐
   │                    apcore-toolkit (Tier 1, byte-equivalent)                      │
   │                                                                                  │
   │   DeviceAuthClient ── request_device_code() ──► POST device_authorization_endpoint│
   │         │                                                                        │
   │         └────────── poll_for_token() ─────────► POST token_endpoint (repeatedly) │
   │                          │                                                       │
   │                    [state machine: pending / slow_down / denied / expired]        │
   │                          │                                                       │
   │                          ▼                                                       │
   │                      TokenSet ──► TokenStore (protocol) ──► FileTokenStore        │
   │                          │                                    (portable, 0600)   │
   │                          └──► as_auth_header_factory() ──────────────┐            │
   └────────────────────────────────────────────────────────────────────┼────────────┘
                                                                         ▼
                                              HTTPProxyRegistryWriter(auth_header_factory=…)
                                                          (already shipped)
```

The bottom edge is the point of this design: the output of the flow plugs into
an **integration point that already exists**. `HTTPProxyRegistryWriter` has
accepted a pluggable `auth_header_factory` since it shipped — today consumers
pass `lambda: {"Authorization": "Bearer xxx"}` with a hard-coded token. This
proposal fills that hole with a managed credential, and adds no new integration
surface at all.

## Provider Configuration and Compatibility

RFC 8628 fixes the *shape* of the flow, not the URLs, not the extra parameters
each vendor demands, and not the response encoding. Every authorization server
differs, so **no endpoint may be hard-coded anywhere in the toolkit**. This
section defines the configuration surface that makes the client work against
an arbitrary provider.

### `DeviceAuthConfig`

| Field | Required | Purpose |
|---|---|---|
| `issuer` | one of `issuer` / explicit endpoints | Base URL for [OIDC discovery](#endpoint-discovery). Endpoints are fetched from it when not given explicitly |
| `device_authorization_endpoint` | see above | Explicit URL. **Overrides discovery** when both are present |
| `token_endpoint` | see above | Explicit URL. Overrides discovery |
| `revocation_endpoint` | no | Used by `logout()`; discovered when available. Absent for providers that do not implement RFC 7009 |
| `client_id` | yes | Public client identifier |
| `client_secret` | no | Some providers operate device flow as a *confidential* client. RFC 8628 targets public clients, but this must be supported to work with real deployments |
| `client_auth_method` | no, default `none` | `none` \| `client_secret_post` \| `client_secret_basic`. Which method is used, and **on which endpoints** — see [Client authentication](#client-authentication) |
| `scope` | no, but **often required in practice** | List of scopes; joined with `scope_separator`. RFC 8628 marks `scope` OPTIONAL, yet at least one surveyed provider rejects a device request without it (`invalid_scope` / "No scopes were requested"). Sending a non-empty scope is the safer default |
| `scope_separator` | no, default `" "` | RFC 6749 mandates a space. A minority of providers expect commas |
| `extra_device_params` | no | Additional form fields on the device-authorization request |
| `extra_token_params` | no | Additional form fields on every token request |
| `extra_headers` | no | Additional HTTP headers on both requests |
| `error_aliases` | no | Maps non-standard error identifiers onto the RFC's four (see [Error identifier normalisation](#error-identifier-normalisation)) |
| `field_aliases` | no | Extends the accepted response field-name lists (see [Field-name normalisation](#field-name-normalisation)) |
| `default_interval` | no, default `5` | Fallback when the provider omits `interval` |
| `http_timeout` | no | Per-request timeout, distinct from the flow deadline |
| `transform_request` | no | [Hook](#extension-hooks) — mutate outbound params/headers; cannot change the URL |
| `parse_response` | no | [Hook](#extension-hooks) — custom response decoding; `None` falls back to built-in parsers |
| `classify_error` | no | [Hook](#extension-hooks) — custom error classification; must return an RFC identifier or `None` |
| `http_client` | no | [Hook](#extension-hooks) — SDK-native HTTP client injection (proxies, mTLS, test doubles) |

Everything beyond `client_id` and a way to reach the endpoints has a working
default, so a conforming provider needs three lines of configuration while a
non-conforming one remains reachable without patching the toolkit.

### Endpoint discovery

Passing `issuer` lets the client resolve `device_authorization_endpoint`,
`token_endpoint`, and `revocation_endpoint` from the server's own metadata
document. This is the single most useful accommodation for provider diversity:
three lines of configuration work against any provider publishing metadata,
and nobody has to memorise URL layouts.

!!! info "Standard well-known paths are not vendor hard-coding"
    The two well-known suffixes below are defined by IETF and OpenID Foundation
    specifications and are identical for every provider. Building them in is
    standards conformance. What must never be built in is a *specific
    provider's* endpoint URL — see the warning at the end of this section.

#### Two metadata standards, two different path constructions

This is a genuine and easily-missed incompatibility, not a vendor quirk. The
two specifications build their URL differently for an issuer that has a path
component:

| Standard | Suffix | Construction | Example for issuer `https://auth.example.com/tenant1` |
|---|---|---|---|
| **RFC 8414** (OAuth 2.0 Authorization Server Metadata) | `oauth-authorization-server` | **Inserted** between host and path | `https://auth.example.com/.well-known/oauth-authorization-server/tenant1` |
| **OpenID Connect Discovery 1.0** | `openid-configuration` | **Appended** to the issuer | `https://auth.example.com/tenant1/.well-known/openid-configuration` |

For an issuer with no path component the two collapse to the same shape, which
is why the difference goes unnoticed until a tenant- or realm-scoped issuer
appears — exactly the layout multi-tenant providers use.

#### Resolution order

Following RFC 8414's own guidance for mixed environments, the client tries, in
order, stopping at the first success:

1. RFC 8414 path-insertion with the `oauth-authorization-server` suffix
2. RFC 8414 path-insertion with the `openid-configuration` suffix
3. OpenID Connect Discovery 1.0 appending, i.e. `{issuer}/.well-known/openid-configuration`

Step 3 is the backward-compatibility fallback for servers deployed against the
older OIDC-only convention. Attempting all three is what makes a single
`issuer` value work across providers that picked different conventions.

This ordering matches the one the MCP authorization specification mandates for
its clients, which is worth staying aligned with even though MCP support is
[out of scope here](#reality-check-device-flows-place-in-the-ai-ecosystem).

#### What counts as a successful fetch

!!! danger "HTTP 200 does not mean you found metadata"
    A surveyed provider serves an **HTML single-page application** at one of
    the well-known paths — returning `200 OK` with a full HTML document rather
    than a 404. A client that advances on status code alone treats this as
    success, then fails while parsing, and never tries the remaining
    candidates.

    A candidate counts as successful only when the response is `2xx` **and**
    the body parses as a JSON object **and** it carries the fields being looked
    for. Anything else — including a 200 that is not JSON — moves to the next
    candidate.

Two further validations, both security-relevant:

- **The `issuer` in the document MUST equal the issuer used to build the URL.**
  A mismatch means the document is not authoritative for this issuer, and it is
  rejected rather than used. MCP states this as a hard requirement, and it is
  cheap insurance against a misconfigured or hostile metadata host.
- **Compare issuers as exact strings.** Do **not** normalise before comparing —
  no case folding of scheme or host, no dropping a default port, no adding or
  removing a trailing slash, no percent-encoding normalisation. This is
  counter-intuitive, and worth stating explicitly because a general-purpose
  "normalise URL" helper is exactly the kind of utility an implementer reaches
  for by reflex. Applying one here weakens the check.

#### Capability check

`device_authorization_endpoint` is **OPTIONAL** in RFC 8414 metadata, so a
provider may support device flow without advertising it — and at least one
major provider does exactly that, publishing metadata that omits the endpoint
while fully supporting the grant.

!!! danger "Capability detection must not be strict, or it locks out working providers"
    Two independently-observed realities make a naive check actively harmful:

    1. **A provider lists the grant under a non-standard short name.** One
       surveyed provider advertises `"device_code"` in `grant_types_supported`
       rather than the RFC 8628 URN
       `urn:ietf:params:oauth:grant-type:device_code` — while still requiring
       the full URN in the actual token request. A client matching only the URN
       concludes "unsupported" and refuses to run against a provider that works
       perfectly.
    2. **A provider omits `grant_types_supported` entirely.** Another major
       provider's discovery document has no such field at all, so its absence
       proves nothing.

    Capability detection is therefore **advisory and permissive**: it may
    produce a clearer error message, but it must never be the sole reason to
    refuse to start.

After a successful metadata fetch the client:

- Uses `device_authorization_endpoint` when present.
- When it is absent, requires the endpoint to be configured explicitly and says
  so — an actionable error naming the missing setting, rather than a confusing
  404 much later.
- Treats the grant as **supported** when `grant_types_supported` is absent, or
  when it contains *either* the full URN *or* the bare `device_code` token.
- Emits a **warning, not an error**, when the field is present and matches
  neither form. The flow still proceeds: the metadata may simply be incomplete,
  and the authorization server's own rejection is more authoritative than a
  guess made from its advertisement.

Constraints, mirroring the [I/O separation](openapi-scanner.md#separation-of-io-from-traversal)
the sibling proposal applies to spec loading:

- Discovery is **network I/O and therefore a separate, explicit step**
  (`config.discover()`), never a hidden fetch inside `login()`. A caller that
  supplies endpoints explicitly performs no network access before the flow
  starts.
- The discovery document is **not** trusted to override an explicitly
  configured endpoint. Explicit configuration always wins, so a compromised or
  misconfigured discovery document cannot silently redirect a token request.
- **Discovered endpoints MUST be `https://`.** A discovered `http://` endpoint
  is refused, not warned about. This is the hard rule.
- **A discovered endpoint on a different origin than the issuer is warned about
  and then followed**, not rejected. *(Amended 2026-09-08. This bullet
  previously said such an endpoint "is rejected rather than followed", which
  was both self-contradictory — `SHOULD` and "rejected" cannot both hold — and
  in conflict with conformance case 029. Rejecting breaks real providers, which
  routinely host the token endpoint on a separate host from the issuer. The
  document's authority is already established by two other checks: it was
  fetched from the issuer's own well-known path over TLS, and its `issuer`
  field was compared verbatim. A third, weaker check that breaks conforming
  providers buys nothing those two do not already cover.)*
- Providers that publish no discovery document, or omit the device endpoint
  from it, simply take the explicit-configuration path. Discovery is a
  convenience, never a requirement.

### Client authentication

RFC 8628 is written for public clients, and it is tempting to assume no client
authentication is involved at all. Real deployments contradict this in two ways
worth designing for:

1. **Confidential clients are common.** Some providers require `client_secret`
   in the device flow; one surveyed provider **requires it in the token
   request**, and another permits several authentication methods with
   `client_secret_basic` as the registration default — so a client registered
   without explicitly choosing "no authentication" silently becomes
   confidential.
2. **Authentication may apply to the device authorization request too, not
   only the token request.** This is the easy thing to miss. A provider can
   reject an unauthenticated `/device/authorize` call with `invalid_client`,
   long before any token request happens.

The configuration therefore covers both dimensions — *how* to authenticate and
*where*:

| `client_auth_method` | Behaviour |
|---|---|
| `none` (default) | `client_id` in the form body, no secret. The RFC's public-client assumption |
| `client_secret_post` | `client_id` + `client_secret` in the form body |
| `client_secret_basic` | HTTP Basic header; `client_id` remains in the body. **The client id and secret are form-urlencoded before base64**, per RFC 6749 §2.3.1 — *(specified 2026-09-08)*. Raw concatenation is a real defect, not a style choice: a secret containing `+` arrives at the server as one containing a space. |

!!! note "Which form-encoding variant is deliberately not specified"
    *(Added 2026-09-08, after all three SDKs had landed.)* The languages'
    default helpers disagree on how a space, `*`, and `~` are spelled:

    | Helper | space | `+` | `*` | `~` |
    |---|---|---|---|---|
    | Python `quote_plus` | `+` | `%2B` | `%2A` | `~` |
    | JS `URLSearchParams` | `+` | `%2B` | `*` | `%7E` |
    | JS `encodeURIComponent` | `%20` | `%2B` | `*` | `~` |

    **All three decode identically**, so no server can tell them apart. The
    corpus therefore asserts the *round trip* — form-decoding the body returns
    every parameter unchanged — rather than an exact byte string. Cases 058 and
    061 were written as exact-bytes assertions first and rewritten, because
    pinning one spelling would have forced two SDKs to hand-roll an encoder for
    zero functional gain.

    This is the repository's [ownership
    rule](../reference/conformance.md) applied rather than assumed:
    byte-equivalence is for formats two consumers must compare, and the
    consumer here is a third-party authorization server that decodes. What
    must not vary is that encoding happens at all.

When a secret is configured, it is sent on **both** the device authorization
and token requests unless the provider is known to want otherwise. Providers
ignore credentials they do not require far more gracefully than they accept
requests missing credentials they do.

More exotic schemes — `private_key_jwt`, `client_secret_jwt`, or a proof-of-
possession header such as DPoP — are deliberately **not** enumerated here.
They involve signing, key management, and per-request proof construction, and
belong in [`transform_request`](#extension-hooks), which exists precisely so
that an unanticipated scheme does not require a toolkit release. This is the
first concrete case validating that hook's existence.

### Request body encoding

RFC 6749 §4.1.3 specifies `application/x-www-form-urlencoded` for token
requests, and it is natural to hard-code it. **That breaks against a real
provider**, and in a way no alias list can repair.

Observed: one vendor's **single token endpoint** accepts three different body
encodings depending on what is being requested — the authorization-code
exchange is form-encoded, while the refresh call and the token-exchange call
on that same URL are **JSON**. A client that sends form-encoded refresh
requests there simply fails.

Encoding is therefore configurable **per request kind**, not globally:

| Setting | Default | Notes |
|---|---|---|
| `request_encoding["device"]` | `form` | The device authorization request |
| `request_encoding["token"]` | `form` | Initial token request |
| `request_encoding["refresh"]` | `form` | Refresh — the one most likely to need `json` |
| `request_encoding["revoke"]` | `form` | Revocation |

Values are `form` or `json`. The defaults match the RFC, so a conforming
provider needs no configuration; a divergent one needs one line rather than a
fork.

This is deliberately a separate axis from
[response encoding](#response-encoding-differences). The two are independent:
the same provider can require a JSON request and return a form-encoded
response, and conflating them into one "uses JSON" flag would make that
combination inexpressible.

### Authentication header shape

Once a token is obtained, the header carrying it is **not** universally
`Authorization: Bearer`. Observed in the survey:

| Shape | Used by |
|---|---|
| `Authorization: Bearer <token>` | OAuth generally; MCP mandates it explicitly |
| `x-api-key: <key>` | A major model vendor's API-key credentials |
| `api-key: <key>` | A cloud-hosted variant of another vendor's API |

The sharpest detail: **one vendor accepts both**, and the correct header
depends on *which kind of credential* is held — a static API key goes in
`x-api-key`, while a short-lived federated token goes in
`Authorization: Bearer`. A shipping CLI in that ecosystem selects the header by
credential type, and its documentation states the mapping explicitly.

The design consequence is a small but load-bearing one:

!!! tip "Credential type → header shape must be data, not a code branch"
    A client that hard-codes `headers["Authorization"] = f"Bearer {token}"`
    cannot express any of the other shapes, and a client with an
    `if provider == …` ladder cannot be extended without a release.

    This is why [`as_auth_header_factory()`](#composition-with-httpproxyregistrywriter)
    returns a **complete header mapping** rather than a token string — the
    factory decides both the header name and the value format. The shipped
    `HTTPProxyRegistryWriter` already types this as an arbitrary string map, so
    every shape above is expressible today with no interface change.

Some APIs additionally require **non-credential headers on every request** — a
mandatory dated API-version header, an organisation or workspace selector.
These are not authentication, but they are just as mandatory, and they belong
in `extra_headers` (for the auth requests) and in the proxy writer's own header
configuration (for API calls).

### Response-encoding differences

Not every provider returns JSON by default — some return
`application/x-www-form-urlencoded` unless the request asks otherwise. The
client therefore MUST send `Accept: application/json` on both requests, and
MUST parse a form-encoded response body as a fallback when the response
content type is not JSON. Failing to do this produces a parse error against
providers that are otherwise perfectly conforming.

`extra_headers` covers the remaining cases (a vendor-specific API-version
header, for instance).

### Field-name normalisation

Providers disagree not only on *values* but on the **names of the response
fields themselves**. At least one major provider returns the verification
address as `verification_url` rather than RFC 8628's `verification_uri`, and
another reports quota errors in a field called `error_code` rather than
`error`. A parser bound to the exact RFC spelling silently reads null and
displays a blank URL to the user.

The client therefore reads each field from an **ordered list of accepted
names**, taking the first present:

| Logical field | Accepted names, in order |
|---|---|
| verification address | `verification_uri`, `verification_url` |
| complete address | `verification_uri_complete`, `verification_url_complete` |
| error identifier | `error`, `error_code` |

The standard name is always tried first, so a conforming provider is
unaffected. `field_aliases` in the configuration extends these lists for a
provider nobody anticipated.

Two related parsing rules:

- **Unknown fields MUST be ignored, never rejected.** Providers add proprietary
  fields to both responses — a localised human-readable prompt, a creation
  timestamp — and a strict parser that fails on unrecognised keys breaks
  against a perfectly functional server.
- **`user_code` MUST be passed through verbatim.** It is case-sensitive at some
  providers, and its shape varies — surveyed values include an 8-character
  group with an embedded hyphen and an 8-character group with none, with at
  least one provider embedding the code unmodified into a URL query parameter.
  Implementations MUST NOT upper-case, strip, or re-group it — not when
  storing, not when handing it to `on_user_code`. Presentation is the
  consumer's business, and even there it must remain enterable as given.

#### Where aliasing stops

Field-name aliasing handles *the same structure under different names*. It
cannot handle *a different structure*, and that case is real: one surveyed
provider returns some errors in the OAuth envelope
(`{error, error_description}`) and others — on the very same endpoint — in a
proprietary envelope with an entirely different field set
(`{errorCode, errorSummary, errorLink, errorId, errorCauses}`). Which envelope
arrives cannot be predicted from the HTTP status, because the same status can
carry either.

No declarative alias list expresses "parse this shape when it looks like that
one". This is the boundary where [`parse_response` and
`classify_error`](#extension-hooks) earn their place: an envelope the toolkit
has never seen is a parsing problem the consumer can solve locally, without a
toolkit release and without touching the state machine.

It is also a reason the built-in parser must **fail soft on shape**: if neither
a standard nor an aliased error identifier can be found, the client reports a
protocol error carrying the raw body, rather than crashing on a missing key.

### Error identifier normalisation

The state machine dispatches on the four RFC 8628 identifiers. Real providers
diverge from them more than the RFC's tone suggests — surveyed behaviour
includes one provider that **has no `slow_down` at all**, one that **renames
`access_denied`**, one that **has no `expired_token`** (folding expiry into a
generic `invalid_grant`), and one whose own documentation spells the same error
two different ways. Without normalisation, each of these reaches the "any other
error identifier" row and terminates a flow that should have continued or
should have failed with a meaningful message.

`error_aliases` maps provider identifiers onto standard ones **before**
dispatch:

```python
error_aliases={
    "authorization_declined": "access_denied",   # renamed by one provider
    "token_expired":          "expired_token",   # alternate spelling
    "invalid_grant":          "expired_token",   # only where the provider
                                                 # folds expiry into it
}
```

!!! warning "`invalid_grant` needs a deliberate decision, not a default"
    Mapping `invalid_grant` → `expired_token` is correct for a provider that
    reports device-code expiry that way, and **wrong** everywhere else, where
    `invalid_grant` also covers a malformed or already-redeemed code. It is
    therefore never applied by default — it is opt-in per provider, precisely
    because a blanket mapping would mask genuine errors as benign expiry.

Aliasing is deliberately data, not code: the state machine's logic — and its
conformance corpus — stay defined purely over the four RFC identifiers, while a
consumer adapts to a non-conforming provider without waiting for a toolkit
release. Aliases may only map **onto** the four standard identifiers; they
cannot invent new states.

### Optional fields real providers omit

| Field | When absent | Client behaviour |
|---|---|---|
| `interval` | Common | Falls back to `default_interval` (5) |
| `verification_uri_complete` | **The norm** — only one surveyed provider returns it, and another documents it as explicitly unsupported | `on_user_code` receives null; the consumer displays the plain URI plus the code. Never build a QR-code-only UI that assumes it |
| `expires_in` (device response) | Rare | Falls back to a 15-minute deadline rather than polling forever |
| `refresh_token` (token response) | Common for short-lived scopes | `ensure_valid()` raises `NoCredentialError` at expiry instead of refreshing, requiring a fresh `login()` |
| `expires_in` (token response) | Occasional | Token is treated as non-expiring; see [`is_expired`](#contract-tokensetis_expired) |

Every one of these is a *normal* provider difference rather than an error, and
each has a defined fallback so that a sparse-but-valid response never crashes
the flow.

### Multi-provider configuration

Because the [token store is keyed by issuer + client_id](#filetokenstore-path),
a single user can hold live credentials for several providers at once. A
consumer wanting a "profile" concept (`--profile staging`) builds it by
constructing one `DeviceAuthClient` per configured provider; the toolkit needs
no profile concept of its own.

### Field Evidence

Every extension point above exists because a real, widely-deployed provider
requires it — none is speculative. The table records what was observed during a
survey of major providers' published documentation and live metadata documents
(**surveyed 2026-09-03**), so a reviewer can judge whether each knob earns its
place.

| Observed divergence | Extension point it justifies |
|---|---|
| A provider returns `authorization_pending` as HTTP **428** and `slow_down`/`access_denied` as **403**, not 400 | [Dispatch on body, never status](#response-dispatch-normative) |
| A provider's error set has **no `slow_down`** and renames `access_denied` → `authorization_declined` | `error_aliases` |
| A provider has **no `expired_token`**, folding expiry into `invalid_grant` | `error_aliases`, opt-in only |
| A provider's own docs spell one error two ways (`expired_token` / `token_expired`) | `error_aliases` |
| A provider returns **`verification_url`**, not `verification_uri` | Field-name alias list |
| A provider reports quota errors in **`error_code`**, not `error` | Field-name alias list |
| A provider returns **form-urlencoded by default**, JSON only with an explicit `Accept` | Mandatory `Accept: application/json` + form-encoded parse fallback |
| A provider returns an updated **`interval` inside the `slow_down` body** | [Prefer the returned interval](#backoff-on-slow_down) |
| A provider publishes **no discovery document at all** (404) | Explicit endpoint configuration must remain fully supported |
| A provider publishes discovery but **omits `device_authorization_endpoint`** | Endpoint override with an actionable error |
| A provider advertises the grant as bare **`device_code`**, not the RFC URN | Permissive capability detection |
| A provider's discovery has **no `grant_types_supported` field** | Absence must never imply "unsupported" |
| A provider **requires `client_secret`** in the device token request | `client_secret` |
| A provider requires a **tenant path segment** consistent across both requests | Explicit per-endpoint URLs |
| A second, independent provider also returns `access_denied` as **403** | Confirms status-code dispatch is broadly wrong, not one vendor's quirk |
| A second provider also surfaces device-code expiry as **`invalid_grant`** | `error_aliases`, opt-in — the pattern is common enough to anticipate |
| A provider mixes **two different error envelope structures on one endpoint**, not predictable from the status code | [`parse_response` / `classify_error` hooks](#extension-hooks) — beyond what any alias list can express |
| A provider **rejects a device request with no `scope`**, though RFC 8628 marks it OPTIONAL | Non-empty `scope` as the safer default |
| A provider applies **client authentication to the device authorization request**, not only the token request | [`client_auth_method`](#client-authentication) covering both endpoints |
| A provider can require a **proof-of-possession header** (DPoP) on token requests | `transform_request` hook |
| A provider defines proprietary error codes including a **429** variant | `error_aliases` plus fail-soft parsing |
| A vendor API accepts **two different authentication headers**, chosen by credential *type* (static key vs short-lived federated token) | [Header shape as data](#authentication-header-shape), not a code branch |
| An API mandates a **dated version header** on every request, unrelated to credentials | `extra_headers` as a first-class, non-optional concept |
| AI-ecosystem headless tooling standardised on **manual code entry**, not device flow; a major CLI ignores `device_authorization_endpoint` even when advertised | [Grant pluggability](#grant-pluggability); [Open Question 5](#open-questions) |
| A shipping CLI uses the OS keychain but **degrades to a `0600` file when it is locked** (routine over SSH) | `FileTokenStore` as the specified fallback half, not a lesser option |
| The MCP specification requires credentials **keyed by the issuing AS `issuer`**, forbidding reuse across AS changes | The existing `issuer + client_id` [store key](#filetokenstore-path) — satisfied by construction |
| A vendor's `--device-auth` is a **proprietary JSON protocol**, not RFC 8628: no `device_code`, pending signalled by **HTTP 403/404**, polling returns an authorization code plus a server-supplied PKCE verifier | [`Grant` abstraction](#grant-pluggability) — beyond what aliases or hooks can express |
| One token endpoint uses **form-encoded for code exchange but JSON for refresh** | [`request_encoding` per request kind](#request-body-encoding) |
| A well-known path returns **`200` with an HTML SPA**, not a 404 | Discovery success requires parseable JSON, not just a 2xx |
| Credential-dependent **extra headers** (account selector, compliance flag) accompany the token | `as_auth_header_factory()` returning a header map, never a bare token string |
| Providers return proprietary extra fields (localised prompt, creation timestamp) | Unknown fields ignored, not rejected |
| A provider's `user_code` is case-sensitive with prescribed formatting | Verbatim `user_code` handling |
| `expires_in` ranges from 300 to 1800 seconds across providers | No hard-coded lifetime; server value with a fallback |
| Only one surveyed provider returns `verification_uri_complete`; another documents it as explicitly unsupported | Treated as optional throughout |

!!! danger "This table is evidence, not configuration"
    These observations MUST NOT be turned into built-in provider profiles, and
    **no vendor endpoint URL appears anywhere in this specification or in any
    implementation of it**. Two reasons:

    1. **It would be wrong within a release or two.** Provider behaviour
       changes, and several entries above are already documentation
       inconsistencies rather than stable contracts.
    2. **Per-tenant and per-realm path segments make any fixed URL
       misleading** — the correct value differs per customer, not just per
       vendor.

    Resolve endpoints through discovery where available, and otherwise from
    configuration the operator supplies. A consumer that wants a
    named-provider preset builds it in its own configuration layer, where it
    can be corrected without a toolkit release.

    The survey date is recorded above because this is a snapshot of observed
    behaviour, not a contract any provider has made.

## Extension Hooks

Declarative configuration covers the divergences we surveyed. It cannot cover
the ones we did not — self-hosted gateways, corporate IdPs, and providers not
in the survey. Without an escape hatch, an unanticipated quirk means the
consumer waits for a toolkit release, which is exactly the coupling this
proposal exists to remove.

So hooks are supported. But **where** a hook may run is the whole design.

### The layering rule

This repository's value proposition is that three SDKs behave identically,
asserted by a conformance corpus. An arbitrary callback participating in
protocol decisions destroys that guarantee: the fixtures would no longer
describe what the client does. The rule that follows:

| Layer | Hook? | Why |
|---|---|---|
| **Transport** — how bytes travel | ✅ Open | Proxies, mTLS, custom CAs, retry policy. No protocol semantics. Precedent exists: the TypeScript writer already accepts `fetchImpl` |
| **Serialisation** — bytes ⇄ fields | ✅ Open | Input and output are pure data. A quirk here is a parsing problem, not a state-machine problem |
| **Normalisation** — vendor identifiers ⇄ RFC identifiers | ⚠️ Open, **constrained output** | Must return one of the four RFC identifiers or "no opinion". The hook chooses *which* standard state applies; it cannot invent a new one |
| **Protocol decisions** — dispatch, backoff, deadline, expiry | ❌ **Closed** | This is what the corpus asserts. Opening it means the toolkit no longer guarantees anything, and every consumer re-derives RFC 8628 |
| **Credential construction** — building a `TokenSet` | ❌ **Closed** | A hook that fabricates a `TokenSet` bypasses every validation. Security-critical, zero legitimate need |

The short version: **hooks may change what the client understands, never what
it decides.**

### The hooks

#### `transform_request(kind, params, headers) -> (params, headers)`

Runs immediately before each outbound request. `kind` is `"device"`,
`"token"`, `"refresh"`, or `"revoke"`.

Covers what static `extra_*` config cannot: values computed at call time —
a request signature, a nonce, a rotating tenant hint, a header derived from
runtime state.

**Constraint: the hook receives no URL and cannot change one.** Request
targeting stays with configuration and discovery. A hook able to redirect the
token request is a hook able to exfiltrate credentials, and no legitimate
compatibility need requires it.

#### `parse_response(kind, status, content_type, raw_body) -> mapping | None`

Runs before the built-in parsers. Returning `None` means "no opinion, use the
default" — so a hook can special-case one endpoint and ignore the rest.

Covers response encodings beyond the built-in JSON and form-urlencoded paths.
This is not hypothetical: one surveyed provider also serves **XML** on request,
and enterprise gateways wrap payloads in envelopes.

The returned mapping must use standard field names; field-name aliasing has
already been applied conceptually by the time the state machine reads it.

#### `classify_error(body) -> identifier | None`

Runs after `error_aliases` and before dispatch. Returns one of
`authorization_pending`, `slow_down`, `access_denied`, `expired_token`, or
`None` for "no opinion".

This is the programmatic form of `error_aliases`, for the cases a static map
cannot express — where the decision needs several fields, a nested error
object, or an HTTP status combined with a body field.

!!! danger "The return value is validated, and a bad one is an error"
    Returning anything outside the four identifiers plus `None` is a
    programming error, and implementations MUST reject it loudly rather than
    coercing it or passing it through. Without this check the hook becomes a
    back door into the closed protocol-decision layer — it could invent a fifth
    state the state machine has no defined behaviour for. Conformance case 045
    asserts the rejection.

#### `http_client` injection

An SDK-native client object (`httpx.Client`, a `fetch` implementation, a
`reqwest::Client`). Covers proxies, custom CA bundles, mTLS, connection
pooling, and test doubles.

Idiomatic per language rather than a common signature, matching the precedent
`HTTPProxyRegistryWriter` already sets.

### Hooks and the conformance corpus

Two rules keep hooks from eroding the cross-SDK guarantee:

1. **The corpus runs with no hooks installed.** It asserts default behaviour.
   A consumer installing hooks accepts responsibility for their own consistency
   — a fair trade, since they chose to leave the paved path.
2. **Hook *contracts* are themselves conformance-tested.** Invocation order,
   the meaning of `None`, and rejection of an invalid `classify_error` return
   are asserted, so a hook written against one SDK behaves the same on the
   other two.

### Hooks are an escape hatch, not the intended path

A hook solves one consumer's problem privately. That is its purpose and also
its cost: nothing shared is learned from it.

**When two or more consumers write the same hook, that divergence should be
promoted into declarative configuration** — a new alias list, a new `extra_*`
field, a new parser branch — where it gets a conformance case and everyone
benefits. `field_aliases` and `error_aliases` exist precisely because the
survey found divergences common enough to deserve first-class support; the same
promotion path should stay open afterwards.

Reviewers should treat a proliferation of hooks in downstream code as a signal
that this specification is missing something, not as the system working as
intended.

### Why not simply subclass?

An obvious alternative is to make `DeviceAuthClient` subclassable with
overridable methods. Rejected for three reasons:

- **It exposes everything.** Subclassing puts the protocol-decision layer in
  reach by default, which is exactly the boundary worth keeping closed. Hooks
  make the open surface explicit and finite.
- **It does not port.** Python inheritance, TypeScript class extension, and
  Rust trait implementation have materially different semantics; a
  subclass-based contract cannot be specified once for three SDKs. Function
  values can.
- **It is untestable as a contract.** "Any override of any method" has no
  conformance surface. Four named hooks with defined signatures do.

## The Polling State Machine

This is the part that must be identical across SDKs, and the part best suited
to conformance testing. It is pure logic over a response sequence and a clock.

### Flow

1. **Request.** `POST` to the device authorization endpoint with `client_id`
   and `scope`. Response yields `device_code`, `user_code`,
   `verification_uri`, optional `verification_uri_complete`, `expires_in`, and
   optional `interval`.
2. **Notify.** Invoke `on_user_code` with the display-relevant fields. This is
   the only place the consumer is asked to show something.
3. **Wait.** Sleep `interval` seconds *before* the first poll (RFC 8628 §3.5:
   the client should not poll faster than the interval, and the user has not
   had time to act yet).
4. **Poll.** `POST` to the token endpoint with
   `grant_type=urn:ietf:params:oauth:grant-type:device_code`, `device_code`,
   and `client_id`.
5. **Dispatch** on the response, per the table below.
6. **Terminate** on success, terminal error, or deadline.

### Response dispatch (normative)

!!! danger "Dispatch on the response body, never on the HTTP status code"
    RFC 8628 §3.5 describes error responses as HTTP 400, and it is tempting to
    key the state machine on the status. **Do not.** Surveying live providers
    (see [Field Evidence](#field-evidence)) found one major identity provider
    returning `authorization_pending` as **HTTP 428** and both `slow_down` and
    `access_denied` as **HTTP 403**. A status-code-driven client treats all
    three as fatal and breaks against that provider entirely.

    Implementations MUST parse the body and dispatch on the error identifier.
    The status code is used for exactly one thing: deciding whether the body is
    a success payload (`2xx`) or an error payload (everything else).

| Body | Action | Interval change |
|---|---|---|
| Success payload containing `access_token` | Return `TokenSet`; flow succeeds | — |
| error `authorization_pending` | Continue polling | unchanged |
| error `slow_down` | Continue polling | **see [Backoff](#backoff-on-slow_down)** |
| error `access_denied` | Terminate — user refused | — |
| error `expired_token` | Terminate — device code expired | — |
| Any other error identifier, or unparseable body | Terminate — protocol error | — |
| Transport failure (connection reset, timeout) | Continue polling, counted against the deadline | unchanged |

Dispatch happens **after** [error normalisation](#error-identifier-normalisation),
so a provider's non-standard spelling reaches this table already mapped onto
one of the four rows.

Transport failures are treated as retryable rather than terminal: a dropped
connection mid-flow is common on flaky networks, and the deadline already
bounds the total wait, so retrying cannot loop forever.

### Backoff on `slow_down`

RFC 8628 §3.5 mandates a fixed **+5 seconds**, not a multiplier —
implementations MUST NOT substitute exponential backoff, because the server's
rate limiter is written against the RFC's behaviour.

One refinement, observed in the field: some providers return an updated
`interval` **inside the `slow_down` error response**. When present, that value
is authoritative and is used verbatim; otherwise the client adds 5 seconds.
Preferring the server's own number is strictly better than guessing, and costs
one conditional.

```
new_interval = error_body.interval  if present
               else current_interval + 5
```

### Deadline

Polling stops when `expires_in` seconds have elapsed since the device-code
request, even if the server has not yet returned `expired_token`. Relying
solely on the server's error leaves a client polling indefinitely against a
server that never sends it.

### Determinism and the injected clock

For the state machine to be conformance-testable without real time passing,
both the clock and the sleep function MUST be injectable. Each SDK exposes
these as optional constructor parameters defaulting to the real
implementations:

| SDK | Monotonic clock (deadline) | Sleep | Wall clock (`expires_at`) |
|---|---|---|---|
| Python | `clock: Callable[[], float] = time.monotonic` | `sleep: Callable[[float], None] = time.sleep` | `wall_clock: Callable[[], float] = time.time` |
| TypeScript | `clock?: () => number` | `sleep?: (ms: number) => Promise<void>` | `wallClock?: () => number` |
| Rust | `clock: Box<dyn Fn() -> Instant>` | `sleep: Box<dyn Fn(Duration) -> BoxFuture<'static, ()>>` | `wall_clock: Box<dyn Fn() -> SystemTime>` |

**There are three injection points, not two.** *(Added 2026-09-08; the table
named only two, while conformance case 015 pins a wall-clock `now` of 1000.)*
The two clocks are not interchangeable and the split is the one described under
[`TokenSet`](#tokenset): the **monotonic** clock measures the polling deadline,
so an NTP correction or a laptop suspend cannot make it jump; the **wall**
clock computes `expires_at`, which must survive a process restart and so cannot
be monotonic. An implementation that reuses one for both fails case 015 or
becomes untestable.

#### Poll-loop ordering (normative)

```
loop:
    sleep(interval)
    elapsed += interval
    if elapsed >= deadline:  ->  deadline_exceeded, stop
    poll the token endpoint
    dispatch on the body
```

*(Stated normatively 2026-09-08. The corpus pinned this; the prose did not, and
it is the most likely off-by-one in the whole implementation.)* Checking the
deadline **before** sleeping instead is visually equivalent and wrong: case 023
requires 180 sleeps and 179 polls against a 900-second deadline at a
5-second interval, and the other ordering gives 180 of each. Case 008 fails the
same way.

Elapsed-time measurement MUST use a **monotonic** clock, not wall-clock time,
so that an NTP correction or a laptop suspend/resume mid-flow cannot make the
deadline jump backwards or fire early.

## Grant Pluggability

The [reality check](#reality-check-device-flows-place-in-the-ai-ecosystem)
above argues for separating the reusable core from the device-flow-specific
part. That split is worth drawing explicitly, because it is nearly all core:

| Component | Grant-independent? |
|---|---|
| `TokenSet`, expiry, skew handling | ✅ Reusable by any grant |
| `TokenStore` protocol, `FileTokenStore` | ✅ |
| Refresh with rotation handling | ✅ |
| Provider configuration, discovery, field/error normalisation | ✅ |
| Extension hooks | ✅ |
| Token redaction | ✅ |
| **Polling state machine** | ❌ Device-flow specific |
| **Device authorization request** | ❌ Device-flow specific |

Only two of eight are device-flow-specific. Naming the whole thing
"DeviceAuthClient" therefore under-sells it and makes a second grant look like
a rewrite when it is actually an additional ~100 lines against a shared core.

**Recommendation:** structure the implementation as a `Grant` abstraction with
`DeviceCodeGrant` as its first implementation, even if device flow is the only
grant that ships in V1. The cost is one interface; the alternative is a
refactor the moment a second grant is needed — which the ecosystem evidence
suggests will be soon.

#### The case that settles it: a "device flow" that is not RFC 8628

One major AI vendor's CLI ships a `--device-auth` login. It is **not** RFC 8628
— it is a proprietary protocol that merely solves the same problem:

| Aspect | RFC 8628 | That vendor's protocol |
|---|---|---|
| Request body | form-encoded | **JSON** |
| Response fields | `device_code`, `verification_uri`, `expires_in`, `interval` | `device_auth_id`, `user_code`, `interval` — **no `device_code`, no `verification_uri`, no `expires_in`** |
| "Still pending" signal | `error: "authorization_pending"` in the body | **HTTP 403 / 404**, with no error body to inspect |
| What polling returns on success | The access token | An **authorization code**, plus a **server-supplied PKCE verifier** |
| Completion | Done | A second, standard code-for-token exchange |

Note the third row: pending is expressed **purely by HTTP status**, which
directly contradicts this specification's
[dispatch-on-body rule](#response-dispatch-normative). Both rules are correct
for their own protocol — and that is exactly the point. They cannot coexist
inside one state machine.

This settles the question of how far configuration and hooks can stretch:

- `error_aliases` cannot help — there is no error identifier to alias.
- `field_aliases` cannot help — the fields are not renamed, they are *absent*,
  and a new one (`device_auth_id`) has no RFC counterpart.
- Even `parse_response` and `classify_error` cannot help — the **shape of the
  flow** differs: it polls to obtain a code, not a token, and then performs an
  additional exchange.

A protocol that different is a **different grant**, and pretending otherwise
would mean bending the RFC 8628 state machine until it no longer describes
RFC 8628. With a `Grant` interface, it is an independent implementation reusing
the same `TokenSet`, `TokenStore`, refresh, and provider-configuration layers —
which is precisely the arrangement that makes it cheap.

Whether the toolkit itself should ship such a grant is a separate question
(probably not — it is vendor-specific). What matters is that **a consumer can
add one without forking**, and that requires the interface to exist in V1.

### Manual code entry

The ecosystem's actual headless fallback. Worth specifying precisely, because
"just have the user paste a code" hides a real requirement.

The flow is **not** a simplified device flow. It is `authorization_code`
carried by hand:

1. The client builds an authorization URL and displays it.
2. The user opens it on any device, authenticates, and is shown an
   authorization code.
3. The user pastes that code into the terminal.
4. The client exchanges it at the token endpoint for a `TokenSet`.

No polling, no device endpoint, no `slow_down` handling — steps 1–3 replace the
entire state machine.

!!! warning "PKCE is mandatory here, not optional"
    Because the authorization code travels through a human — and possibly
    through a clipboard, a chat window, or a screenshot — this variant **MUST**
    use PKCE (RFC 7636). Without it, an intercepted code is directly
    redeemable. OAuth 2.1 requires PKCE for all authorization-code flows for
    exactly this reason, and MCP mandates it too.

    This is why manual code entry is **not free**: it needs the
    authorization-endpoint half of OAuth (URL construction, `state`, PKCE
    verifier/challenge generation and storage) that device flow does not.
    Roughly 120 lines per SDK, not 20. Cheap relative to its value, but it must
    be budgeted rather than assumed.

Everything after the code exchange — storage, expiry, refresh, redaction — is
the shared core, unchanged.

## `TokenSet`

The credential produced by a successful flow.

| Field | Type | Notes |
|---|---|---|
| `access_token` | string | Treated as opaque. Never parsed. |
| `token_type` | string | Normalised to `"Bearer"` casing on read; servers vary between `Bearer` and `bearer` |
| `expires_at` | integer (Unix seconds) | Computed as *wall-clock now + `expires_in`* at receipt. Persisted as an absolute instant, not a duration, because a duration is meaningless after a restart |
| `refresh_token` | string \| null | Absent when the server does not issue one |
| `scope` | list of strings | Split on spaces per RFC 6749; empty list when absent |
| `obtained_at` | integer (Unix seconds) | For diagnostics and store-format migration |

!!! note "Two clocks, deliberately"
    The polling deadline uses a **monotonic** clock (correctness under NTP
    adjustment). `expires_at` uses **wall-clock** time, because it must survive
    process restarts, and monotonic values are meaningless across them. This is
    not an inconsistency; the two serve different purposes.

### Expiry check

`is_expired(skew_seconds = 30)` returns true when
`wall_clock_now + skew_seconds >= expires_at`.

The 30-second default skew means a token is treated as expired slightly before
it really is, so a request is not dispatched with a credential that will expire
in flight. Callers on a badly-synchronised clock can widen it.

## Refresh

When `refresh_token` is present, `refresh()` exchanges it for a new `TokenSet`
via `grant_type=refresh_token`.

**Refresh-token rotation is assumed.** OAuth 2.1 and RFC 6749 §10.4 recommend
that servers issue a new refresh token on each use and invalidate the old one.
Implementations MUST therefore:

- Replace the entire stored `TokenSet` — never merge a new access token into an
  old record while keeping the previous refresh token.
- Treat a refresh failure with `invalid_grant` as **terminal**: the refresh
  token is spent or revoked, and the correct response is to discard the stored
  credential and require a fresh login, not to retry.
- **`error_aliases` does not apply on the refresh path.** *(Specified
  2026-09-08.)* The refresh path inspects the **raw, unaliased** identifier, so
  the rule above still fires for a consumer who opted into
  `invalid_grant` → `expired_token` for the device-code path. Without this, the
  one alias the spec explicitly describes would disable the one terminal rule
  that protects the store.

### Concurrency

Two processes refreshing the same rotated refresh token concurrently will race:
one succeeds, the other gets `invalid_grant`, and if the loser then clears the
store, it destroys the winner's freshly written credential.

V1's answer is deliberately modest: `FileTokenStore` performs an **atomic
replace** (write to a temporary file in the same directory, then `rename`), so
a reader never observes a half-written file, and a lost race costs a redundant
re-login rather than a corrupted store. Full cross-process locking is listed in
[Open Questions](#open-questions) Q2 rather than specified here, because
correct advisory locking differs substantially across the three languages and
platforms, and the failure it prevents is recoverable.

## Token Storage

### Why the toolkit does not ship a keychain integration

OS keychain access is the least portable thing in this proposal: macOS
Keychain, Windows Credential Manager, and Linux Secret Service (with no
guaranteed daemon on headless boxes) have different semantics, and the three
SDKs would reach them through three unrelated third-party libraries with
different failure modes. Cross-language behavioural parity — the property this
repository's conformance corpus exists to enforce — is not achievable there.

The honest split:

| Layer | Ships where |
|---|---|
| `TokenStore` **protocol** (`load` / `save` / `clear`) | toolkit — all three SDKs |
| `FileTokenStore` — portable, `0600`, atomic replace | toolkit — all three SDKs |
| `KeychainTokenStore` — OS-native | **consumer** (`apcore-cli`, or an opt-in extra) |

Consumers wanting a keychain implement the protocol; the flow neither knows nor
cares which store it was handed.

!!! important "A keychain store MUST define its degraded path"
    This is not a corner case. A shipping CLI in this ecosystem uses the macOS
    Keychain by default and **falls back to a `0600` file when the Keychain
    refuses the write** — which happens routinely over SSH, where the login
    keychain is locked. Its Linux and Windows paths are plain files to begin
    with.

    So the pattern the ecosystem actually converged on is *keychain when
    available, `0600` file otherwise* — and `FileTokenStore` is exactly the
    fallback half of it, which is a good reason for the toolkit to own that
    half properly rather than treating it as a lesser option.

    Any consumer-supplied `KeychainTokenStore` **MUST** specify what it does
    when the keychain is unavailable or locked. Silently failing to persist is
    the worst outcome: the user appears to log in successfully and is prompted
    again on every invocation, with no indication why.

### `FileTokenStore` path

The default location follows platform convention and MUST be documented
prominently, because other tools need to know about it (see
[Downstream Impact](#downstream-impact)):

| Platform | Path |
|---|---|
| Linux / BSD | `$XDG_CONFIG_HOME/apcore/credentials.json`, falling back to `~/.config/apcore/credentials.json` |
| macOS | `~/.config/apcore/credentials.json` |
| Windows | `%APPDATA%\apcore\credentials.json` |

macOS deliberately uses the XDG-style path rather than
`~/Library/Application Support`, so that a developer's dotfile conventions and
any cross-platform tooling see one location. The file holds a JSON object keyed
by *issuer* + *client_id*, so credentials for multiple authorization servers
coexist without collision.

### File permissions

The file MUST be created with mode `0600` (owner read/write only) on POSIX. It
MUST be created with those permissions **at open time**, not chmod'd afterward
— a create-then-chmod sequence leaves a window in which the file is
world-readable. On Windows, the file inherits the ACL of `%APPDATA%`, which is
already user-scoped.

If an existing credentials file is found with broader permissions, the store
MUST refuse to read it and surface an actionable error rather than silently
using a credential that other local users can read.

## Consumer Callbacks

The toolkit's only channel to a user interface.

| Callback | When | Arguments |
|---|---|---|
| `on_user_code` | Once, after the device-code response | `verification_uri`, `user_code`, `verification_uri_complete` (nullable), `expires_in` |
| `on_poll` | Before each poll attempt | `attempt` (1-based), `interval`, `elapsed` |

Both are optional; omitting them yields a silent, headless flow suitable for
tests and daemons. Neither is permitted to influence protocol behaviour — a
callback that raises MUST NOT abort the flow, because a rendering failure in
the UI layer is not a reason to lose an in-flight authorization. Exceptions
from callbacks are swallowed and recorded as a warning.

`on_user_code` intentionally receives `verification_uri_complete` separately
rather than pre-merged: RFC 8628 §3.3.1 notes it embeds the user code in the
URL and is convenient for QR codes, but consumers should still display the
plain URI and code for manual entry.

## Security Considerations

| Risk | Mitigation |
|---|---|
| **Tokens in logs** | No SDK may log a token, even at debug level. `TokenSet`'s `__repr__` / `toString` / `Debug` MUST redact `access_token` and `refresh_token` to a fixed placeholder. This is a spec requirement, not a suggestion — a leaked debug log is the most common way CLI credentials escape. |
| **World-readable credentials** | `0600` at creation; refuse to read over-permissive files. |
| **`device_code` persistence** | The `device_code` is a short-lived pre-authorization secret and MUST NOT be written to the store — only the resulting `TokenSet` is persisted. |
| **Refresh-token replay** | Rotation assumed; whole record replaced; `invalid_grant` is terminal. |
| **Clock manipulation** | Monotonic clock for the polling deadline; skew allowance on expiry checks. |
| **Endpoint confusion** | Both endpoints MUST be `https://` (except `http://localhost` for development). A plaintext token endpoint is refused at construction time, not at request time. |
| **Open redirect via `verification_uri`** | The URI comes from the authorization server and is displayed to the user. Consumers that auto-open a browser MUST validate the scheme is `https` and SHOULD show the URL before opening it. Specified here because the auto-open lives in `apcore-cli`, which needs the requirement stated somewhere normative. |
| **Token echoed to a proxied API** | `as_auth_header_factory()` returns headers for one configured `base_url`. Consumers MUST NOT reuse a factory across hosts; sending a bearer token to an unintended host leaks it. |

## Downstream Impact

Once this proposal fixes a credential path, tools that maintain
credential-sensitive path baselines need to know about it. `apexe` — which
wraps other CLI tools and protects a list of credential-bearing paths
(`~/.ssh`, `~/.aws`, `~/.netrc`, …) — must add the
[`FileTokenStore` path](#filetokenstore-path) to that list. Otherwise a
governed tool-wrapper ends up with a blind spot for exactly the credential
this proposal introduces.

This is why the storage path is normative and documented rather than left to
each SDK: an undocumented or per-SDK-varying location cannot be added to
anyone's baseline.

## Public API

### Python

```python
from apcore_toolkit.auth import (
    DeviceAuthClient, DeviceAuthConfig, FileTokenStore, TokenSet,
)

# (a) Discovery — endpoints resolved from the issuer's well-known document
config = DeviceAuthConfig(issuer="https://auth.example.com", client_id="apcore-cli",
                          scope=["openid", "api.read"])
config = config.discover()      # explicit network step, never implicit

# (b) Explicit endpoints — for providers without a discovery document.
#     Anything set here takes precedence over a discovered value.
config = DeviceAuthConfig(
    device_authorization_endpoint="https://auth.example.com/device/code",
    token_endpoint="https://auth.example.com/token",
    client_id="apcore-cli",
    scope=["openid", "api.read"],
    # Provider-specific knobs — none of these are required by RFC 8628,
    # and each exists because some real provider needs it:
    extra_device_params={"audience": "https://api.example.com"},
    extra_headers={"X-Api-Version": "2024-01"},
    error_aliases={"pending": "authorization_pending"},
    scope_separator=" ",
    default_interval=5,
)

client = DeviceAuthClient(config, store=FileTokenStore())

# Callbacks receive KEYWORD arguments in Python - see the login contract below.
tokens: TokenSet = client.login(
    on_user_code=lambda *, verification_uri, user_code, **_: print(
        f"Visit {verification_uri} and enter {user_code}"
    ),
)

tokens = client.ensure_valid()          # refreshes if near expiry
headers = client.as_auth_header_factory()
```

### TypeScript

```typescript
import {
  DeviceAuthClient, FileTokenStore, type TokenSet,
} from "apcore-toolkit";

const client = new DeviceAuthClient({
  deviceAuthorizationEndpoint: "https://auth.example.com/device/code",
  tokenEndpoint: "https://auth.example.com/token",
  clientId: "apcore-cli",
  scope: ["openid", "api.read"],
  store: new FileTokenStore(),
});

const tokens: TokenSet = await client.login({
  onUserCode: ({ verificationUri, userCode }) =>
    console.log(`Visit ${verificationUri} and enter ${userCode}`),
});

await client.ensureValid();
const headers = client.asAuthHeaderFactory();
```

### Rust

```rust
use apcore_toolkit::auth::{DeviceAuthClient, DeviceAuthConfig, FileTokenStore, TokenSet};

let client = DeviceAuthClient::new(
    DeviceAuthConfig {
        device_authorization_endpoint: "https://auth.example.com/device/code".into(),
        token_endpoint: "https://auth.example.com/token".into(),
        client_id: "apcore-cli".into(),
        scope: vec!["openid".into(), "api.read".into()],
    },
    Box::new(FileTokenStore::default()),
)?;

let tokens: TokenSet = client.login(Some(&|ev| {
    println!("Visit {} and enter {}", ev.verification_uri, ev.user_code);
})).await?;

client.ensure_valid().await?;
let headers = client.as_auth_header_factory();
```

### Composition with `HTTPProxyRegistryWriter`

```python
writer = HTTPProxyRegistryWriter(
    base_url="https://api.example.com",
    auth_header_factory=client.as_auth_header_factory(),
)
```

`as_auth_header_factory()` returns a **callable**, not a snapshot of headers,
so each request picks up the current token. The factory calls `ensure_valid()`
internally, meaning a long-running process refreshes transparently without the
proxy writer knowing anything about OAuth.

!!! success "Confirmed: the factory is invoked per request"
    Verified against all three shipped implementations — Python, TypeScript,
    and Rust each call `auth_header_factory` **once per request**, inside
    `execute`, not once at construction. Transparent refresh through the
    factory is therefore viable, and no change to the invocation timing is
    needed. (This is currently undocumented in
    [`output-writers.md`](output-writers.md) and should be stated there.)

!!! danger "Blocking: the shipped factory signature is synchronous in all three SDKs"
    All three writers type the factory as a **synchronous** function returning
    a plain string map — Python `Callable[[], dict[str, str]]`, TypeScript
    `() => Record<string, string>`, Rust `Fn() -> HashMap<String, String>`.
    Python and TypeScript additionally raise a `TypeError` if the return value
    is not a plain object, so returning a coroutine or promise fails loudly
    rather than working by accident.

    A token refresh is an HTTP round-trip. In TypeScript and Rust that is
    inherently async, so **a synchronous factory cannot perform one**.
    Transparent refresh is therefore impossible on two of the three SDKs
    without widening the signature. See [Open Questions](#open-questions) Q1
    for the options and the recommendation.

## Conformance Corpus

The fixture lives at
[`conformance/fixtures/device_auth.json`](https://github.com/aiperceivable/apcore-toolkit/blob/main/conformance/fixtures/device_auth.json)
and follows the structure used by `format_csv.json` and `display_resolve.json`
(a single JSON document with `$schema`, `title`, `description`, `version`, and
a `test_cases` array). It ships **65 cases** — the 55 enumerated below, plus ten added during review and
during the three SDK implementations. See [Harness contract](#harness-contract).

### Harness contract

Read these before writing a harness; they are what make the corpus runnable
without a network:

| Convention | Meaning |
|---|---|
| `kind` | One of `poll`, `expiry`, `refresh`, `discovery_url`, `discovery`, `parse`, `request`, `redaction`, `alias_validation`, `hook`. Dispatch on it. |
| `poll_delays[i]` | The sleep performed **before** `token_responses[i]`. The first entry is therefore the initial wait, never `0`. |
| `polls_made` | How many scripted responses were actually consumed. A case scripting more responses than this is asserting the client **stopped**. |
| `repeat_last_response` | When `true`, the final scripted response repeats indefinitely — used by case 023, whose 15-minute deadline would otherwise need 180 literal entries. |
| Wall clock | Pinned to `1000` wherever a case asserts `expires_at` or `obtained_at`. The **polling** clock stays monotonic and advances only by the sleeps. |
| `transport_error: true` | A scripted connection failure rather than an HTTP response. |

**Six cases beyond the 55 below, each closing a way to pass without asserting anything.**

| Case | Why it exists |
|---|---|
| `expiry_017b_not_yet_expired` | The negative of 017. An implementation returning `true` unconditionally passes 017 alone. |
| `provider_025b_alias_onto_standard_accepted` | The negative of 025. One rejecting every alias passes 025 alone. |
| `discovery_056_http_endpoint_refused` | The hard half of the endpoint rule — `http://` is refused. |
| `discovery_057_cross_origin_endpoint_warned_not_refused` | The soft half — a different origin is warned about and followed. Pins the 2026-09-08 amendment that resolved the spec's self-contradiction against case 029. |
| `clientauth_058_basic_urlencodes_before_base64` | Case 049's `cid`/`sec` encode identically under both readings, so it cannot discriminate. This one uses a secret containing `:`, `+` and a space. |
| `refresh_059_aliases_do_not_apply_on_refresh` | An opted-in `invalid_grant` alias must not disable the terminal store-clearing rule. |

**Cases 045 and 046 were rewritten during the first implementation** because
both were vacuous as originally written. 045 supplied no body, so a harness
inventing a *standard* identifier resolved it before the hook was ever reached
and nothing raised; 046 used `access_denied`, which resolves without the hook,
so it passed whatever the hook did. Both now carry a deliberately unresolved
identifier. This is worth recording rather than quietly fixing: a corpus case
that cannot fail is worse than no case, because it reports coverage that does
not exist.

Because the state machine is pure over an injected clock and a scripted
response sequence, **no HTTP mocking is required in any SDK**. Each case
supplies a device-code response and an ordered list of token-endpoint
responses; the expectation is the resulting outcome plus the exact sequence of
poll delays.

```jsonc
{
  "id": "device_auth_poll_004_slow_down",
  "description": "slow_down increments interval by exactly 5 seconds, per RFC 8628 §3.5",
  "input": {
    "device_response": { "device_code": "d", "user_code": "ABCD-EFGH",
                         "verification_uri": "https://e.example/device",
                         "expires_in": 600, "interval": 5 },
    "token_responses": [
      { "status": 400, "body": { "error": "authorization_pending" } },
      { "status": 429, "body": { "error": "slow_down" } },
      { "status": 400, "body": { "error": "authorization_pending" } },
      { "status": 200, "body": { "access_token": "t", "token_type": "Bearer",
                                 "expires_in": 3600 } }
    ]
  },
  "expected": {
    "outcome": "success",
    "poll_delays": [5, 5, 10, 10],
    "final_interval": 10
  }
}
```

| Test case | Purpose |
|---|---|
| `device_auth_poll_001_immediate_success` | Token returned on first poll |
| `device_auth_poll_002_pending_then_success` | `authorization_pending` handled, interval unchanged |
| `device_auth_poll_003_default_interval` | Absent `interval` defaults to 5 |
| `device_auth_poll_004_slow_down` | `+5` increment, not exponential |
| `device_auth_poll_005_repeated_slow_down` | Two `slow_down` responses → interval 15 |
| `device_auth_poll_006_access_denied` | Terminal, no further polls |
| `device_auth_poll_007_expired_token` | Terminal |
| `device_auth_poll_008_deadline_exceeded` | Client stops at `expires_in` without a server error |
| `device_auth_poll_009_initial_wait` | First poll occurs after one interval, not immediately |
| `device_auth_poll_010_transport_error_retried` | Connection failure is retried, deadline still enforced |
| `device_auth_poll_011_unknown_error_terminal` | Unrecognised `error` code terminates |
| `device_auth_poll_012_malformed_body` | Unparseable success body terminates cleanly |
| `device_auth_poll_013_token_type_normalised` | `"bearer"` → `"Bearer"` |
| `device_auth_poll_014_scope_split` | `"openid api.read"` → `["openid", "api.read"]` |
| `device_auth_poll_015_expires_at_computed` | `expires_at == now + expires_in` under a fixed clock |
| `device_auth_expiry_016_skew` | `is_expired` true within the 30s skew window |
| `device_auth_expiry_017_no_expiry` | Absent `expires_in` → never auto-expires |
| `device_auth_refresh_018_rotation` | New refresh token replaces the old record wholesale |
| `device_auth_refresh_019_invalid_grant_terminal` | `invalid_grant` clears the store, no retry |
| `device_auth_redaction_020` | `repr`/`toString`/`Debug` contains neither token value |
| `device_auth_provider_021_error_alias` | A vendor code mapped via `error_aliases` dispatches as its standard equivalent |
| `device_auth_provider_022_form_encoded` | A `application/x-www-form-urlencoded` token response parses correctly |
| `device_auth_provider_023_no_device_expiry` | Device response without `expires_in` falls back to a 15-minute deadline |
| `device_auth_provider_024_scope_separator` | A non-default `scope_separator` is used when encoding the request |
| `device_auth_provider_025_alias_cannot_invent_state` | An alias mapping to a non-RFC code is rejected at construction |
| `device_auth_discovery_026_rfc8414_path_insertion` | Issuer with a path component → suffix inserted between host and path |
| `device_auth_discovery_027_oidc_append` | Same issuer → OIDC fallback appends instead of inserting |
| `device_auth_discovery_028_bare_issuer` | Issuer without a path component → both constructions agree |
| `device_auth_discovery_029_explicit_overrides` | An explicitly configured endpoint is not replaced by a discovered one |
| `device_auth_discovery_030_grant_unrecognised_warns` | `grant_types_supported` matching neither form → warning, flow still proceeds |

Cases 026–028 cover pure URL construction and need no network at all, which is
why the well-known path difference is worth pinning in the corpus: it is the
kind of one-line divergence that would otherwise be reimplemented three ways.

The following cases encode divergences observed in the
[field survey](#field-evidence). Each one is a real provider behaviour that a
naive implementation gets wrong:

| Test case | Purpose |
|---|---|
| `device_auth_compat_031_status_428_pending` | `authorization_pending` at HTTP **428** continues polling |
| `device_auth_compat_032_status_403_slow_down` | `slow_down` at HTTP **403** backs off rather than terminating |
| `device_auth_compat_033_status_403_denied` | `access_denied` at HTTP **403** terminates as denial, not protocol error |
| `device_auth_compat_034_interval_from_body` | `interval` inside a `slow_down` body overrides the `+5` default |
| `device_auth_compat_035_verification_url_alias` | `verification_url` is read as the verification address |
| `device_auth_compat_036_error_code_field` | An error reported in `error_code` is dispatched correctly |
| `device_auth_compat_037_unknown_fields_ignored` | Proprietary extra fields in both responses parse without error |
| `device_auth_compat_038_user_code_verbatim` | A mixed-case, hyphenated `user_code` reaches the callback unmodified |
| `device_auth_compat_039_alias_declined` | `authorization_declined` aliased to `access_denied` terminates as denial |
| `device_auth_compat_040_invalid_grant_not_aliased_by_default` | Without an explicit alias, `invalid_grant` is a protocol error — **not** silently treated as expiry |
| `device_auth_discovery_041_bare_device_code_grant` | `grant_types_supported: ["device_code"]` is accepted as supported |
| `device_auth_discovery_042_no_grant_types_field` | Absent `grant_types_supported` is treated as supported |

Case 040 is the one that protects against over-correction: normalisation makes
the client permissive, and this asserts that permissiveness stops short of
masking real errors as benign.

[Hook](#extension-hooks) contracts are asserted too — not the hooks' contents,
which are the consumer's business, but the guarantees the toolkit makes about
invoking them:

| Test case | Purpose |
|---|---|
| `device_auth_hook_043_transform_request` | Params and headers returned by `transform_request` reach the outbound request |
| `device_auth_hook_044_parse_response_none_falls_back` | Returning `None` uses the built-in parser rather than failing |
| `device_auth_hook_045_classify_error_invalid_return` | A return value outside the four identifiers is **rejected loudly**, not coerced |
| `device_auth_hook_046_classify_error_none_falls_back` | Returning `None` falls through to default classification |
| `device_auth_hook_047_hook_order` | `error_aliases` applies before `classify_error`, which applies before dispatch |
| `device_auth_hook_048_no_hooks_matches_baseline` | With no hooks installed, output is identical to the corresponding hook-free case |

Client authentication and fail-soft parsing:

| Test case | Purpose |
|---|---|
| `device_auth_clientauth_049_basic_both_endpoints` | `client_secret_basic` produces a Basic header on **both** the device and token requests |
| `device_auth_clientauth_050_none_sends_no_secret` | Default `none` sends `client_id` only, with no secret anywhere |
| `device_auth_parse_051_unknown_envelope_soft_fail` | An error body matching no known envelope yields a protocol error **carrying the raw body**, not a crash on a missing key |
| `device_auth_encoding_052_json_refresh` | `request_encoding["refresh"] = "json"` sends a JSON body while the token request stays form-encoded |
| `device_auth_discovery_053_html_200_falls_through` | A `200` HTML response at a well-known path is **not** accepted; the next candidate is tried |
| `device_auth_discovery_054_issuer_mismatch_rejected` | Metadata whose `issuer` differs from the constructed one is rejected |
| `device_auth_discovery_055_issuer_compared_verbatim` | Issuers differing only by trailing slash or case are treated as **different**, not normalised into equality |

The last case is worth stating as conformance rather than convention: token
redaction is a security property, and a property nobody tests is a property
that regresses.

---

## Contract: DeviceAuthClient.login

### Inputs
- `on_user_code` / `onUserCode`: callback, optional — invoked once with `verification_uri`, `user_code`, `verification_uri_complete` (nullable), `expires_in`. **`expires_in` is the *effective* deadline, never null** — the callback receives the number the poll loop actually uses: the server's `expires_in` when it sends one, the 900-second fallback when it does not, and `timeout_seconds` whenever that is shorter than either. One number, and it is the one the flow will really stop at. *(Specified 2026-09-08 after it surfaced as a live three-way divergence: Python passed 900 while TypeScript and Rust passed null.)* A consumer rendering a countdown must be told when the client will really give up; passing null makes every consumer reimplement the fallback, which is the per-CLI duplication this proposal exists to remove. Cases 060, 060b and 060c pin the three inputs. 060c was added last: all three SDKs had correctly clamped the *deadline* with `timeout_seconds` and then reported the unclamped server value to the callback — consistently inconsistent with the sentence above, which is the failure mode a corpus catches only when someone writes the case.
- `on_poll` / `onPoll`: callback, optional — invoked before each poll with `attempt`, `interval`, `elapsed`

!!! note "Callback shape differs per SDK, deliberately"
    The *fields* are identical across SDKs; the *calling convention* is
    idiomatic per language. Python passes them as keyword arguments,
    TypeScript passes a single options object (`{ verificationUri, userCode,
    verificationUriComplete, expiresIn }`), and Rust passes a borrowed event
    struct. Porters must translate the shape, not just the names — the same
    divergence the [`ScannedModule` construction note](scanning.md#contract-scannedmodule)
    already documents for data classes.
- `timeout_seconds` / `timeoutSeconds`: number, optional — hard ceiling independent of the server's `expires_in`; when both apply, the **shorter** wins

### Errors
- `AuthorizationDeniedError` — server returned `access_denied`
- `AuthorizationExpiredError` — device code expired, or the deadline elapsed
- `AuthorizationProtocolError` — unrecognised error code, malformed body, or non-retryable HTTP status
- Transport errors are retried, not raised, until the deadline
- Callback exceptions are **swallowed** and recorded as warnings; they never abort the flow

### Returns
- On success: `TokenSet`, already persisted to the configured store

### Properties
- async: false (Python) / true (TypeScript, Rust)
- pure: false — network I/O, store writes, sleeps
- idempotent: false — each call starts a new authorization
- deterministic: the **state machine** is deterministic given a fixed clock and response sequence; this is what the corpus asserts

---

## Contract: DeviceAuthClient.ensure_valid

### Inputs
- `skew_seconds` / `skewSeconds`: number, optional, default `30`

### Errors
- `NoCredentialError` — nothing in the store; caller must `login()` first
- `RefreshFailedError` — refresh rejected with `invalid_grant`; the store is cleared as a side effect
- Transport errors propagate (unlike during polling, there is no deadline to bound retries)

### Returns
- On success: a `TokenSet` valid for at least `skew_seconds` more

### Properties
- async: false (Python) / true (TypeScript, Rust)
- pure: false — may perform a refresh and write to the store
- idempotent: true when the token is still valid (returns it unchanged, no network call)

---

## Contract: TokenSet.is_expired

### Inputs
- `skew_seconds` / `skewSeconds`: number, optional, default `30`
- `now`: injectable wall-clock, optional — for testing

### Errors
- None raised

### Returns
- boolean — `true` when `now + skew >= expires_at`; `false` when `expires_at` is absent (a token with no stated expiry never auto-expires)

### Properties
- async: false
- pure: true

---

## Contract: TokenStore

### Methods
- `load(key)` → `TokenSet | None` — never raises on a missing store; returns null/None
- `save(key, tokens)` → void — MUST be atomic (write-temp-then-rename) and MUST create with `0600` on POSIX
- `clear(key)` → void — idempotent; clearing an absent key is not an error

### Inputs
- `key`: string, required — canonically `"<issuer>|<client_id>"`, so multiple authorization servers coexist in one store. **When no `issuer` is configured** — the explicit-endpoints path, which is a first-class option — the key falls back to `"<token_endpoint>|<client_id>"`. *(Specified 2026-09-08; previously undefined, and three SDKs inventing three fallbacks would write stores that cannot read each other.)*

### Errors
- `CredentialPermissionError` — an existing file has permissions broader than `0600`; refuse rather than read
- I/O errors propagate

### Properties
- async: false (Python) / true (TypeScript, Rust)
- pure: false
- availability: protocol in all three SDKs; `FileTokenStore` in all three; OS-keychain implementations are consumer-supplied

---

## Contract: Extension Hooks

All hooks are optional; omitting every one yields the fully-specified default
behaviour the conformance corpus asserts.

### Invocation order

Fixed, and conformance-tested:

```
transform_request  →  [HTTP]  →  parse_response  →  field-name aliasing
                                        →  error_aliases  →  classify_error
                                        →  state-machine dispatch
```

!!! important "`classify_error` is consulted only when the identifier is still unresolved"
    *(Clarified 2026-09-08.)* The arrow diagram shows precedence, and precedence
    here also means **skipping**: read the identifier through field aliases,
    apply `error_aliases`, and if the result is one of the four standard
    identifiers, dispatch immediately — `classify_error` is not called at all.
    Only a body whose identifier is still unresolved reaches the hook.

    This matters beyond tidiness. "Always call the hook and let the alias win"
    and "call the hook only when unresolved" produce identical results for a
    pure hook, so both pass the corpus — but they differ observably for a hook
    that raises, logs, or has any other side effect. Three SDKs could each pick
    one and all stay green. Case 047 pins the precedence; this paragraph pins
    the skipping.

### transform_request

- **Inputs:** `kind` (`"device"` | `"token"` | `"refresh"` | `"revoke"`), `params` (mapping), `headers` (mapping)
- **Returns:** the possibly-modified `(params, headers)` pair
- **Errors:** an exception propagates and fails the flow — unlike the observational callbacks, this hook is load-bearing and a failure must not be swallowed
- **Constraints:** receives no URL and cannot influence request targeting

### parse_response

- **Inputs:** `kind`, `status` (integer), `content_type` (string | null), `raw_body` (string)
- **Returns:** a mapping using standard field names, or `None` to fall back to the built-in JSON / form-urlencoded parsers
- **Errors:** propagate

### classify_error

- **Inputs:** `body` (the parsed error mapping)
- **Returns:** exactly one of `"authorization_pending"`, `"slow_down"`, `"access_denied"`, `"expired_token"`, or `None`
- **Errors:** returning any other value MUST raise / throw / return `Err` — it is never coerced, defaulted, or passed through to dispatch

### http_client

- **Inputs:** none — this is an injected object, not a callback
- **Type:** SDK-native (`httpx.Client`, a `fetch` implementation, `reqwest::Client`)
- **Errors:** transport errors surface exactly as they would from the default client, so the retry semantics in the [dispatch table](#response-dispatch-normative) are unchanged

### Properties

- async: follows each SDK's convention — synchronous in Python, may return a promise/future in TypeScript and Rust
- purity: not required; hooks may read external state
- conformance: hook *contracts* (order, `None` semantics, invalid-return rejection) are asserted; hook *contents* are the consumer's responsibility
- guarantee: with no hooks installed, behaviour is byte-identical to the corpus baseline

---

## Migration Plan

### Phase 0 — Resolve the scope question ✅ done (2026-09-08)

[`scope.md`](../scope.md#not-a-workflow-engine) has been amended. "Token
management" no longer appears as an undifferentiated exclusion; the clause now
separates `apflow`'s two token concerns (context-token budgeting, and issuing
`apflow`'s own service JWTs) from third-party credential *acquisition*, and
admits the latter to the toolkit on the narrow grounds that the toolkit already
ships the `auth_header_factory` seam this fills.

### Phase 1 — Verify the writer's factory contract ✅ done (2026-09-08)

Verified against the shipped 0.11.1 sources and recorded in
[`output-writers.md`](output-writers.md#auth_header_factory-invocation-contract).
Two findings:

- **Per request, not at construction.** All three SDKs call the factory from
  inside `execute`, so a rotating credential is picked up without any writer
  change. Transparent refresh is reachable.
- **Synchronous in all three.** The return type is a plain header mapping with
  no promise, future, or coroutine. This is what makes [Open
  Question 1](#open-questions) real rather than hypothetical, and it is now
  answered below.

### Phase 1.5 — Write the conformance corpus first (blocking)

`conformance/fixtures/device_auth.json` is written, reviewed, and committed
**before** any SDK implementation begins. This ordering is not a preference:
`view_model.json` preceded the three `TuiViewModel` implementations and
produced zero divergences, while the surfaces that were implemented first and
reconciled later (`csv`/`jsonl` before 0.7.0) produced three independent bugs.
Credential handling is the worst possible surface on which to repeat the
second pattern.

### Phase 2 — Toolkit implementation

In `apcore-toolkit-{python,typescript,rust}`:

1. `TokenSet`, `TokenStore` protocol, `FileTokenStore`.
2. `DeviceAuthConfig` including the provider-compatibility surface
   (`extra_*`, `error_aliases`, `scope_separator`, `default_interval`) and
   `discover()` for OIDC well-known resolution.
3. `Grant` interface plus `DeviceCodeGrant` — the polling state machine,
   injected clock and sleep, JSON with form-encoded parsing fallback, and
   per-request-kind body encoding.
4. Refresh with rotation handling.
5. [Extension hooks](#extension-hooks) — the four call sites, plus
   `classify_error` return-value validation.
6. `conformance/fixtures/device_auth.json` wired into all three suites.
7. Redaction in `repr` / `toString` / `Debug`.
8. Docs: flip this file's status banner, update `overview.md`, `README.md`,
   `mkdocs.yml`.

### Phase 3 — `apcore-cli` (separate PR per SDK)

1. A `login` command rendering the user code, opening the browser, and
   showing poll progress via the callbacks.
2. A `KeychainTokenStore` **adapting the existing `config_encryptor`** (OS
   keyring + AES-256-GCM fallback, already shipped in all three CLI SDKs) to
   the toolkit's `TokenStore` protocol. This is an adapter, not a new
   integration.
3. Extend the existing `AuthProvider` to source its bearer token from the
   device-auth client when configured, keeping the static API-key path as-is.
4. Wire `as_auth_header_factory()` into the existing HTTP-proxy path.

### Phase 4 — Ecosystem note

Open an issue against `apexe` to add the credentials path to its
credential-baseline list.

## Decisions

*Recorded 2026-09-08. Every question below now has a verdict; the questions and
their option tables are kept verbatim because the reasoning is what makes a
later reversal auditable.*

| # | Question | Decision |
|---|---|---|
| 1 | Async refresh through a synchronous factory | **Option A — widen the factory.** Phase 1 verified the signature is synchronous in all three SDKs, so this is a real constraint. TypeScript widens the return type to `Record` \| `Promise<Record>` and awaits (awaiting a non-promise is a no-op, so every existing caller keeps working); Rust adds a *separate* optional `async_auth_header_factory` field rather than changing the existing one; Python stays synchronous because its `httpx` path already is. Option C stays available as an interim if the writer change cannot land first. |
| 2 | Cross-process refresh locking | **Defer.** V1 relies on atomic replace. The race costs a redundant re-login, which is recoverable; correct advisory locking differs meaningfully in all three languages. Revisit only on an observed failure. |
| 3 | Encrypt `FileTokenStore` at rest | **No.** A key stored next to its ciphertext is theatre. Ship `0600` plus honest documentation; real protection is an OS keychain, which is a consumer concern. |
| 4 | Multi-account support | **Defer.** The store is keyed on the issuer and client id joined by a pipe (`<issuer>` \| `<client_id>`), giving one identity per authorization server. The key format extends without a migration when a profile concept is actually asked for. |
| 5 | V1 grant scope | **Option A — device flow only — with the `Grant` interface in place from the start.** This is a deliberate scope-down from this document's original Option B recommendation; see the note below. |
| 6 | MCP-compliant authorization for `apcore-mcp` | **Its own proposal.** Materially different (OAuth 2.1, mandatory PKCE, RFC 8707 resource indicators, 401-driven RFC 9728 discovery). Nothing here blocks it, and the issuer-keyed store is already compatible. |

### Note on Decision 5 — why A rather than the recommended B

The argument for B is sound and is not disputed: headless AI tooling has
largely standardised on manual code entry, and six of the eight components are
grant-independent, so B is cheaper than it looks. Two things still favour A for
**V1**:

- **B needs PKCE, and PKCE widens the one surface whose risk mitigation depends
  on being narrow.** This document's own risk table rates a credential-handling
  defect as the highest-severity failure class here and mitigates it with
  "narrow surface (one grant type)". Adding an S256 challenge/verifier path in
  the same release removes that mitigation while the corpus is still new.
- **There is no committed consumer yet.** Phase 3 has not started in any
  `apcore-cli` SDK. B is justified by what the *first* consumer will hit;
  until one exists, it is speculative scope on a security surface.

The cost of being wrong is bounded and was made bounded on purpose: because the
`Grant` interface ships in V1, adding `ManualCodeGrant` later is one new
implementation against a stable seam, not a rewrite. **This decision reverses
the moment a consumer names a provider without device-flow support** — that is
the trigger to widen, and it should be recorded on issue #17 when it happens.

## Open Questions

*All six are resolved — see [Decisions](#decisions) for the verdicts. The
questions and their option tables are retained verbatim because the reasoning
is what makes a later reversal auditable.*

1. **How does an async refresh reach a synchronous factory?** This is the one
   genuinely blocking design question. Invocation timing is settled (per
   request, verified), but the signature is synchronous in all three SDKs and
   a refresh is an HTTP round-trip. Four options:

    | Option | Assessment |
    |---|---|
    | **A. Widen the factory to allow an async return** | TypeScript: fully backward-compatible — widen to `Record` \| `Promise<Record>` and `await` it (awaiting a non-promise is a no-op). Rust: add a separate optional `async_auth_header_factory` field rather than changing the existing one, keeping the current API intact. Python stays synchronous, since its `httpx` path already is. |
    | **B. Proactive background refresh** | A timer refreshes ahead of expiry so the factory always returns a fresh cached token. Requires a background task in a CLI process that may live for two seconds; awkward and easy to leak. |
    | **C. Caller invokes `ensure_valid()` explicitly** | Simple and requires no writer change, but abandons transparency — every call site must remember, which is exactly the per-CLI duplication this proposal exists to remove. |
    | **D. Block on the refresh inside a sync factory** | Impossible in TypeScript; in Rust, `block_on` inside an async context deadlocks or panics. Not viable. |

    **Recommendation: A**, because it is backward-compatible in both affected
    SDKs and keeps refresh genuinely transparent. C is an acceptable interim
    if the writer change cannot land first — the client works either way; only
    the ergonomics differ.
2. **Cross-process refresh locking.** V1 relies on atomic replace, accepting a
   redundant re-login when two processes race. **Recommendation: defer.** Add
   file locking only if the race is observed in practice; correct advisory
   locking is meaningfully different in all three languages and the failure it
   prevents is already recoverable.
3. **Should `FileTokenStore` encrypt at rest?** Encryption with a key stored
   next to the ciphertext is theatre; real protection requires an OS keychain,
   which is explicitly a consumer concern. **Recommendation: no — rely on
   `0600` and document it honestly** rather than implying protection that
   isn't there.
4. **Multi-account support.** The store is keyed by issuer + client_id, so one
   identity per authorization server. Multiple simultaneous accounts against
   the *same* server would need a profile concept. **Recommendation: defer
   until asked for**; the key format can be extended without a migration.
5. **What is V1's grant scope?** — *the one genuine product decision here.*
   The original plan was device-flow-only. The
   [ecosystem survey](#reality-check-device-flows-place-in-the-ai-ecosystem)
   found that AI tooling has standardised on something else, which makes that
   scope worth re-examining before code is written rather than after.

    | Option | Cost | Assessment |
    |---|---|---|
    | **A. Device flow only** (original) | Baseline | Delivers exactly what the tracking issue asked for. Risk: the first consumer hits a provider without device-flow support and builds paste-the-code themselves anyway |
    | **B. Device flow + manual code entry** | +~120 LOC/SDK, needs PKCE | Covers both the RFC-conforming providers and the ecosystem's actual headless default |
    | **C. Authorization code + PKCE first, device flow second** | Largest reordering | Best aligned with MCP and with observed vendor behaviour, but abandons the issue's stated request |

    **Recommendation: B**, with the [`Grant` abstraction](#grant-pluggability)
    in place from the start. It honours the tracking issue, matches what
    headless environments actually do, and — because six of the eight
    components are grant-independent — costs far less than it appears. A
    reviewer preferring A should still take the `Grant` interface, since that
    is what keeps C reachable later without a rewrite.

6. **Does `apcore-mcp` need MCP-compliant authorization, and when?** Out of
   scope here, but the requirement is real and materially different (OAuth 2.1,
   mandatory PKCE, unconditional RFC 8707 resource indicators, 401-driven
   RFC 9728 discovery, Client ID Metadata Documents). **Recommendation: its own
   proposal**, informed by whether `apcore-mcp` intends to consume remote
   authenticated MCP servers. Nothing here blocks it, and the `issuer`-keyed
   store is already compatible.

## Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| `scope.md` "token management" reading blocks or reverses the proposal | **Medium-high** | Phase 0 forces the decision before implementation. Explicitly surfaced rather than glossed. |
| The writer's factory contract turns out to be construction-time | Medium | Phase 1 checks it first; the fix is small and local to the writers. |
| Security defect in credential handling — the highest-severity failure class here | Low-medium | Narrow surface (one grant type), opaque tokens, mandated `0600`, redaction as a conformance case, no bespoke crypto. |
| Cross-SDK divergence in polling behaviour | Low-medium | 20-case corpus over an injected clock; no network mocking needed. |
| Keychain absence disappoints users expecting OS-native storage | Medium | Documented explicitly as a consumer extension point with a working portable default, rather than half-implemented in three incompatible ways. |
| Scope creep into a general OAuth library | Medium | Non-Goals enumerate the excluded grants; each addition requires its own proposal and consumer. |
| Server implementations deviating from RFC 8628 | **High — confirmed, not hypothetical.** A survey of major providers found every one of them diverging on at least one axis; see [Field Evidence](#field-evidence) | The whole of [Provider Configuration](#provider-configuration-and-compatibility) exists for this, and each knob is now traceable to an observed divergence rather than to speculation. 12 conformance cases encode the specific behaviours. |
| Over-permissive normalisation masks genuine errors as benign | Medium | `invalid_grant` is never aliased by default; capability detection warns rather than refuses; case 040 asserts that an un-aliased `invalid_grant` still surfaces as a protocol error. |
| The field survey ages and its conclusions drift | **Certain, over time** | The survey is dated and explicitly framed as a snapshot, not a contract. Nothing in the implementation branches on provider identity, so a provider changing behaviour costs a consumer one configuration line — never a toolkit release. |
| [Hooks](#extension-hooks) become the default path, fragmenting behaviour across consumers | Medium | Hooks cannot reach the protocol-decision layer, so even heavy hook use leaves dispatch, backoff, and expiry identical. The promotion path (recurring hook ⇒ declarative config) is written into the spec, and reviewers are told to read hook proliferation as a spec gap. |
| A hook is used to subvert credential handling | Low-medium | The two dangerous capabilities are closed by design: hooks receive no URL and cannot redirect a token request, and no hook can construct a `TokenSet`. `classify_error`'s return value is validated rather than trusted. |
| Hook signatures diverge across SDKs (the `auth_header_factory` problem again) | Medium | Specified up front as sync-in-Python / promise-or-future-permitted in TypeScript and Rust, rather than being discovered late. Contract block states it explicitly. |
| Provider needs an accommodation nobody anticipated | Medium | The escape hatches are data-driven (`extra_device_params`, `extra_token_params`, `extra_headers`, `error_aliases`), so a consumer adapts without waiting for a toolkit release. |
| Endpoint URLs hard-coded somewhere and going stale | Low | Specified as forbidden; no vendor URL table appears in this document, and discovery is the recommended path. |

## Implementation Estimate

| Phase | Component | Estimated LOC | Notes |
|---|---|---|---|
| 0 | `scope.md` amendment | ~10 | Blocking, trivial |
| 1 | `output-writers.md` factory-contract documentation | ~40 | Blocking, docs only |
| 2 | Python — client + config/discovery + TokenSet + store + tests | ~600 | Provider-compatibility surface and discovery add ~150 over a bare RFC client |
| 2 | TypeScript — same | ~600 | |
| 2 | Rust — same | ~700 | Injected async sleep adds trait-object plumbing |
| 2 | Conformance corpus (55 cases) | ~1000 | Shared. Grew after the provider surveys — 16 cases encode real observed divergences, 6 assert hook contracts, 3 cover client authentication and fail-soft parsing |
| 2 | `Grant` interface + `DeviceCodeGrant` split | ~40 each | One interface. Cheap now, a refactor later — see [Grant Pluggability](#grant-pluggability) |
| 2 | *(Optional, [Open Question 5](#open-questions))* Manual code entry + PKCE | ~120 each | Only if V1 scope widens to option B |
| 2 | Hook plumbing (all three SDKs) | ~60 each | Small: four call sites plus return-value validation. The design work was deciding what *not* to expose |
| 2 | Spec doc updates | ~150 | |
| **Phase 2 total** | | **~2000 LOC** | |
| 3 | `apcore-cli` login command + keychain store, per SDK | ~120 each | Consumer-side. Lower than a from-scratch estimate because `config_encryptor` (keyring + AES-GCM) and `AuthProvider` (header injection) already ship — Phase 3 adapts them rather than building them |

## See Also

- [`output-writers.md`](output-writers.md#httpproxyregistrywriter-python-typescript-and-rust) — `auth_header_factory`, the integration point this fills.
- [`openapi-scanner.md`](openapi-scanner.md#end-to-end-composition) — the sibling proposal; together they cover "wrap a remote authenticated API".
- [Scope & Boundaries](../scope.md) — requires the amendment described above.
- [RFC 8628](https://datatracker.ietf.org/doc/html/rfc8628) — OAuth 2.0 Device Authorization Grant.
- [RFC 6749 §10.4](https://datatracker.ietf.org/doc/html/rfc6749#section-10.4) — refresh-token security considerations.
