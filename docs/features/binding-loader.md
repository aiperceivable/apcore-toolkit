---
description: "Pure-data BindingLoader parses .binding.yaml back into ScannedModule objects (inverse of YAMLWriter), with strict/loose modes for validation and round-trip."
---

# Binding Loader

The `BindingLoader` parses `.binding.yaml` files back into `ScannedModule` objects — the inverse of [`YAMLWriter`](output-writers.md#yamlwriter). Available in **Python**, **TypeScript**, and **Rust**.

## Overview

`YAMLWriter` emits binding files from `ScannedModule` objects (write path). `BindingLoader` reads them back (read path), enabling:

- **Validation** — re-scan + load, diff to catch drift between code and committed bindings.
- **Merging** — load a curated binding file, merge with a fresh scan, re-emit.
- **Round-trip** — edit `display`, `annotations`, `metadata` manually in YAML and preserve them across regenerations.
- **Inspection tooling** — build CLIs, dashboards, linters that consume binding files without running the underlying code.

Unlike `apcore.BindingLoader` (which `import_module`s the `target` and registers a runtime `FunctionModule`), toolkit's `BindingLoader` is **pure data**: it does not import code, does not call the target, and does not touch the apcore Registry. Result is a list of plain `ScannedModule` values.

## `BindingLoader`

**Python module**: `apcore_toolkit.binding_loader` (also re-exported from `apcore_toolkit`)
**TypeScript module**: `apcore-toolkit`
**Rust module**: `apcore_toolkit::binding_loader` (also re-exported from `apcore_toolkit`)

### Methods

| Method | Description |
|--------|-------------|
| `load(path, *, strict=False, recursive=False, pattern="*.binding.yaml")` | Load a single binding file, or every file in a directory whose **name** matches `pattern`. Pass `recursive=True` (Python) / a `true` third argument (TypeScript) / `recursive: true` (Rust) to descend into subdirectories; `pattern` still matches file names only, at every depth. See [Pattern Matching](#pattern-matching). |
| `load_data(data, *, strict=False)` | Load pre-parsed YAML data (a `{"bindings": [...]}` dict). |

Both return `list[ScannedModule]` (or `ScannedModule[]` / `Vec<ScannedModule>`).

**Directory loading is all-or-nothing.** When given a directory, the loader walks files in lexicographic order; the first malformed file raises `BindingLoadError` and discards every previously-parsed file. Callers that need best-effort aggregation (e.g., a linter) should iterate files themselves and invoke `load` per file with their own exception handling.

### Modes

| Mode | Required fields | Use case |
|------|-----------------|----------|
| **loose** (`strict=False`, default) | `module_id`, `target` | Verification, merging, tooling — permissive parsing with sensible defaults. |
| **strict** (`strict=True`) | `module_id`, `target`, `input_schema`, `output_schema` | CI pipelines, production builds — fail fast on incomplete bindings. |

Missing optional fields fall back to `ScannedModule` dataclass defaults (empty schemas, empty tags, `version="1.0.0"`, `display=None`, etc.).

#### Loose-mode wrong-type policy

When a non-required field (`input_schema`, `output_schema`, `tags`) is present but holds the wrong type — e.g., `input_schema: 42` or `tags: "single-string"` — the policy is mode-dependent:

| Mode | Behaviour |
|------|-----------|
| **strict** (`strict=True`) | Raise `BindingLoadError` (Python) / `Err(BindingLoadError)` (Rust) / `throw BindingLoadError` (TypeScript). |
| **loose** (`strict=False`, default) | Log a warning naming the offending field and entry, then coerce the field to its empty default (`{}` for schemas, `[]` for tags). Cross-SDK guarantee — Python, Rust, and TypeScript all warn-and-coerce in loose mode. |

The loose-mode behaviour is intentional: callers running with `strict=False` have explicitly opted into permissive parsing, and a single wrong-type optional field should not abort scanning of an otherwise valid binding file.

Required fields (`module_id`, `target`) are always validated and reject wrong-type or empty-string values regardless of mode.

## Pattern Matching

*Added in 0.12.0. Tracking issue: [aiperceivable/apcore-toolkit#18](https://github.com/aiperceivable/apcore-toolkit/issues/18).*

apcore 0.30 made `bindings.pattern` a canonical configuration key
(`schemas/defaults.schema.json`, default `"*.binding.yaml"`). Before 0.12.0 the
toolkit loader hardcoded that default and had no parameter through which a
caller could honour a configured value, so any consumer that needs the
loader's **return value** — rather than apcore's registration side effect —
silently ignored the key.

`load` now takes the resolved pattern as an argument.

### The loader takes a value; it does not read `Config`

This is deliberate and is the whole reason a parameter was chosen over
config-awareness. `BindingLoader` is the *pure-data* half of the ecosystem's
binding story: it does not import `target`, does not touch a `Registry`, and
does not depend on `apcore.Config`. Resolution — the environment > file >
default precedence chain of PROTOCOL_SPEC §9.2 — belongs to the caller, which
is the layer that actually holds a `Config`.

```python
# The caller resolves; the loader matches.
pattern = config.get("bindings.pattern") or "*.binding.yaml"
modules = loader.load(bindings_dir, pattern=pattern)
```

This mirrors `apcore`'s own `load_binding_dir_with_config(dir, pattern, config)`,
which likewise accepts an explicit `pattern` that outranks the configured one.

### Supported syntax

`pattern` is matched against the **file name only** — never a directory
component, never the full path. Two metacharacters are recognised:

| Token | Meaning |
|---|---|
| `*` | Zero or more characters, including `.` |
| `?` | Exactly one character |
| anything else | A literal, including `[`, `]`, `{`, `}`, `!`, `^`, `-` |

Everything about this matcher is fixed across the three SDKs and asserted by
[`conformance/fixtures/binding_pattern.json`](../reference/conformance.md).

- **Character classes are not supported.** `[ab].binding.yaml` matches a file
  literally named `[ab].binding.yaml`, nothing else. POSIX classes, ranges, and
  the two incompatible negation spellings (`[!x]` vs `[^x]`) are excluded
  precisely because they are where language glob implementations diverge.
- **Brace expansion is not supported.** `{a,b}.yaml` is a literal.
- **Matching is case-sensitive on every platform**, including macOS and
  Windows. The matcher never case-folds. (A case-insensitive *filesystem* can
  still hand the loader a name whose case differs from what was written on
  disk; that is the OS's behaviour, not the matcher's.)
- **Leading-dot files are matched normally.** `*.binding.yaml` matches
  `.hidden.binding.yaml`. Shell globs traditionally exclude dotfiles; this
  matcher does not, which preserves the pre-0.12.0 behaviour of all three SDKs.

### Normative matching algorithm

Three independent implementations converge only if the algorithm is specified,
not just the syntax. Implement exactly this — the standard two-pointer glob
match with single-star backtracking:

```text
match(pattern, name):
    p = 0; n = 0                  # cursors into pattern and name
    star = -1; mark = 0           # last '*' seen, and where to resume the name

    while n < len(name):
        if p < len(pattern) and pattern[p] == '?':
            p += 1; n += 1
        elif p < len(pattern) and pattern[p] == '*':
            star = p; mark = n; p += 1        # consume zero chars for now
        elif p < len(pattern) and pattern[p] == name[n]:
            p += 1; n += 1
        elif star >= 0:
            p = star + 1; mark += 1; n = mark  # let the last '*' eat one more
        else:
            return false

    while p < len(pattern) and pattern[p] == '*':
        p += 1                    # trailing stars may match nothing

    return p == len(pattern)
```

Three properties this pins down, each of which a hand-rolled matcher gets wrong
in a different language:

- **Bounded time — no exponential blowup.** Backtracking resumes only from the
  most recent `*`, giving O(len(pattern) × len(name)) worst case. The obvious
  recursive "try every split point" matcher is *exponential* on inputs like
  `*a*a*a*a*b` against a long run of `a`s, and `pattern` arrives from
  configuration, which is not always held to the same trust level as code.
- **Comparison is over Unicode code points, not bytes or UTF-16 units.**
  Rust iterates `char`s, Python iterates `str`. **TypeScript must not index the
  string directly** — `"…"[i]` yields UTF-16 code units, so a single astral
  character would be consumed by two `?`s. Convert once with `Array.from(name)`
  / `[...name]` and index the resulting array.
- **No normalization, no case folding.** Code points are compared for equality
  as-is. Two names that are canonically equivalent but differently composed
  (NFC vs NFD — routine on macOS) are **not** equal here. The matcher does not
  apply Unicode normalization, and neither should the caller silently: a name
  comes from the filesystem in whatever form the filesystem stores it.

### Every string is a valid pattern

*Changed in 0.12.0, to track apcore 0.31.0.*

`load` **never raises on `pattern` for syntactic reasons.** `a[b`, `{x,y}`,
`**`, an empty string, and a value containing `/` or `\` are all valid
patterns whose brackets, braces, extra star and separators are literals. A
pattern matching no file yields no modules, which is not an error in itself.

This is apcore's rule, adopted verbatim: `PROTOCOL_SPEC` §9.2.3 Algorithm A25
requirement 2, and §5.12.6 clause 6 stating it for `bindings.pattern`
specifically.

!!! note "This reverses the 0.12.0 behaviour, and the reason is worth recording"
    0.12.0 rejected an empty pattern and any pattern containing `/` or `\`,
    which made `**/*.binding.yaml` — the shape a caller reaches for first when
    they want recursion — a clear error pointing at `recursive=True` instead of
    a silently empty result. That was the better diagnostic, and it is gone.

    It was given up because the whole reason this parameter exists is to let a
    caller honour apcore's `bindings.pattern` (see the section opening). A
    caller that resolves the key from `Config` and hands the same string to
    both components must get the same answer from both; under 0.12.0, apcore
    matched nothing and the toolkit raised. Two components reading one
    configuration key and disagreeing is precisely the failure
    [#18](https://github.com/aiperceivable/apcore-toolkit/issues/18) was filed
    to close, and keeping a nicer error message at the price of reopening it
    was the wrong trade.

    The diagnostic now lives here rather than in an exception: **if a pattern
    selects nothing and it contains `/`, you probably wanted `recursive=True`.**

`\` deserves its own mention because it is the one that looks like a bug: A25
requirement 4 names it a literal, so on a filesystem where a filename may
contain a backslash, `sub\*.binding.yaml` genuinely matches `sub\x.binding.yaml`.
It is not a path separator here.

### Composition with `recursive`

The two parameters are orthogonal, and this is the answer to the question
issue #18 left open:

| Parameter | Governs |
|---|---|
| `recursive` | **Which directories are traversed** — the immediate directory, or the whole tree |
| `pattern` | **Which file names match**, at whatever depth traversal reached |

The caller never writes a `**/` prefix, and the loader never synthesises one.
`load(dir, recursive=True, pattern="*.binding.yaml")` matches
`dir/a.binding.yaml` and `dir/nested/deep/b.binding.yaml` alike.

Before 0.12.0 Python composed `"**/" + pattern` internally while Rust and
TypeScript suffix-matched against a flat traversal. Threading a caller-supplied
pattern through those two shapes unchanged would have produced three different
answers for `recursive=True`; specifying `pattern` as a name matcher removes
the composition question entirely.

### Ignored for single files

When `path` names a file, `pattern` is ignored for **matching**. A caller that
explicitly names one file has already made the selection; the loader does not
second-guess it, and `load("odd-name.yaml")` still works.

*0.12.0 additionally validated the pattern here, ahead of the file/directory
check, so that a malformed one raised even for a single-file path. That was
dropped before release — validation is gone entirely — see
[Every string is a valid pattern](#every-string-is-a-valid-pattern) — so there
is nothing left to order, and "ignored" now simply means ignored.*

### Directories are never candidates

A directory whose *name* matches the pattern is skipped, at every depth, in
both the recursive and non-recursive branches. It is not selected and then
failed on at read time.

This is easy to get wrong in exactly the way that produces a three-way
divergence: `Path.glob` and a bare `readdir` both yield directories, so an
implementation that filters on name alone will hand a directory to its YAML
reader — one SDK surfacing `EISDIR`, another an empty list, a third a parse
error. Guard on file type during traversal.

**Test the target, not the link.** The file-type check follows symlinks: a
symlink whose name matches and whose target is a regular file **is** selected.
Do not implement the guard with a non-following check — Rust's
`DirEntry::file_type()` and Node's `Dirent.isFile()` both report on the link
itself, so a guard built on either silently drops every symlinked binding file.
That is a data-loss-shaped regression with no error, and all three SDKs
included symlinked files before the guard existed. Use a following stat
(`Path::is_file`, `fs.statSync(...).isFile()`, `Path.is_file()`); a broken
symlink fails the stat and is skipped like any non-file.

**Following file symlinks is not following directory symlinks.** Traversal
still does not descend into a symlinked directory — that is where cycles and
tree-escape live, and the existing `walkdir(follow_links(false))` policy stays.
A symlink to a directory is therefore neither selected nor traversed. Cases 040
and 041 pin both halves.

### Ordering and caps are unchanged

Matched files are still sorted lexicographically by path before parsing, the
directory load is still all-or-nothing, and the safety caps still apply to the
matched set (see [Safety Caps](#safety-caps-rust-only)). `pattern` narrows
*which* files are read; it changes nothing about how they are read.

**Sort by code point, case-sensitively, on every platform.** Fixture case 035
pins this deliberately. Rust's `PathBuf` ordering and JavaScript's default
string sort are already code-point order everywhere. Python's `sorted()` over
`Path` objects is **not** — it compares `_str_normcase`, which case-folds on
Windows, so `["M.binding.yaml", "a.binding.yaml"]` would come back reversed
there. Python must sort with an explicit `key=str`. This is a pre-existing
cross-platform divergence that `pattern` did not introduce and that case 035
surfaces; sorting by the path string costs nothing on POSIX and removes it.

### Upstream note: apcore's own three SDKs disagree here

Worth recording, because a reader will reasonably ask why the toolkit does not
simply copy apcore's matcher. As of apcore 0.30, apcore's three
`load_binding_dir` implementations interpret `bindings.pattern` three different
ways:

| SDK | Implementation | `data*.yaml` vs `data1.yaml` |
|---|---|---|
| Python | `Path.glob(pattern)` — full glob, character classes included | matches |
| Rust | `pattern.strip_prefix('*')` then `ends_with(suffix)` | no match |
| TypeScript | `pattern.replace('*', '')` then `endsWith(suffix)` | no match |

All three agree on the default `*.binding.yaml` and on nothing else — the same
latent-divergence shape issue #18 filed against the toolkit, one layer up. The
toolkit therefore specifies its own matcher and pins it with a fixture rather
than inheriting an accident. This is filed upstream separately; the toolkit
does not wait on it.

## Field Mapping

| YAML key | `ScannedModule` field | Strict required | Loose default |
|----------|-----------------------|-----------------|---------------|
| `module_id` | `module_id` | ✓ (always required) | — |
| `target` | `target` | ✓ (always required) | — |
| `description` | `description` | — | `""` |
| `documentation` | `documentation` | — | `None` |
| `tags` | `tags` | — | `[]` |
| `version` | `version` | — | `"1.0.0"` |
| `annotations` | `annotations` | — | `None` (parsed via `ModuleAnnotations.from_dict` when present) |
| `examples` | `examples` | — | `[]` (malformed entries skipped with warning) |
| `metadata` | `metadata` | — | `{}` |
| `input_schema` | `input_schema` | ✓ strict | `{}` |
| `output_schema` | `output_schema` | ✓ strict | `{}` |
| `display` | `display` | — | `None` |
| `suggested_alias` | `suggested_alias` | — | `None` |
| `warnings` | `warnings` | — | `[]` |

## `spec_version` Handling

The top-level `spec_version` field is advisory:

| State | Behaviour |
|-------|-----------|
| Missing | Warn and assume `"1.0"`. |
| `"1.0"` | Silent. |
| Anything else | Warn and proceed best-effort (forward compatibility). |

## Errors

`BindingLoadError` is raised on:

- Path not found.
- Malformed YAML (parse error).
- Top-level document not a mapping; `bindings` key missing or not a list; entry not a mapping.
- Missing required fields per selected mode.

The error carries `file_path`/`module_id`/`missing_fields` plus a human-readable `reason` in Python; TypeScript exposes the same data as camelCase fields `filePath`/`moduleId`/`missingFields`/`reason` on `BindingLoadError extends Error`.

In Rust the error is an enum (`thiserror`-derived) with 7 variants — `PathNotFound`, `FileRead`, `YamlParse`, `MissingFields`, `InvalidStructure`, `FileTooLarge`, `TooManyFiles` — carrying per-variant payloads; callers pattern-match to recover structured information. The final two are safety caps introduced in 0.5.0 (see [Safety Caps](#safety-caps-rust-only) below). An `InvalidPattern` variant existed in an unreleased draft of this work and was withdrawn before 0.12.0 shipped: a pattern can no longer be invalid (see [Every string is a valid pattern](#every-string-is-a-valid-pattern)).

## Safety Caps (Rust only)

The Rust loader enforces two defensive caps that surface as structured errors rather than unbounded resource use on untrusted input:

| Cap | Default | Error variant | Payload |
|-----|---------|---------------|---------|
| Max file size | 16 MiB | `FileTooLarge` | `{ path, size, max }` |
| Max files per directory scan | 10,000 | `TooManyFiles` | `{ path, max }` |

Python and TypeScript loaders do not currently enforce these caps. Callers in those SDKs that load untrusted directories should pre-validate file counts and sizes. Cross-SDK alignment for these caps is tracked as an open item — see the 0.5.0 changelog and cross-SDK sync reports.

## Code Examples

=== "Python"

    ```python
    from apcore_toolkit import BindingLoader, BindingLoadError

    loader = BindingLoader()

    # Load a directory
    modules = loader.load("./bindings")

    # Honour a configured bindings.pattern — the caller resolves, the loader matches
    modules = loader.load(
        "./bindings",
        recursive=True,
        pattern=config.get("bindings.pattern") or "*.binding.yaml",
    )

    # Load a single file in strict mode
    try:
        modules = loader.load("users.binding.yaml", strict=True)
    except BindingLoadError as exc:
        print(exc.missing_fields)

    # Load pre-parsed data
    modules = loader.load_data({
        "spec_version": "1.0",
        "bindings": [{"module_id": "x.y", "target": "pkg:f"}],
    })
    ```

=== "TypeScript"

    ```typescript
    import { BindingLoader, BindingLoadError } from "apcore-toolkit";

    const loader = new BindingLoader();

    // Load a directory
    const modules = loader.load("./bindings");

    // Honour a configured bindings.pattern
    const configured = loader.load("./bindings", false, true, "*.binding.yaml");

    // Strict mode
    try {
      const strictModules = loader.load("users.binding.yaml", true);
    } catch (exc) {
      if (exc instanceof BindingLoadError) console.log(exc.missingFields);
    }

    // Pre-parsed data
    loader.loadData({
      spec_version: "1.0",
      bindings: [{ module_id: "x.y", target: "pkg:f" }],
    });
    ```

=== "Rust"

    ```rust
    use apcore_toolkit::{BindingLoader, BindingLoadError};
    use std::path::Path;

    let loader = BindingLoader::new();
    let modules = loader.load(Path::new("./bindings"), false, false)?;

    // Honour a configured bindings.pattern
    let configured = loader.load_with_pattern(
        Path::new("./bindings"),
        false,
        true,
        Some("*.binding.yaml"),
    )?;

    match loader.load(Path::new("users.binding.yaml"), true, false) {
        Ok(mods) => { /* ... */ }
        Err(BindingLoadError::MissingFields { missing_fields, .. }) => {
            println!("{missing_fields:?}");
        }
        Err(e) => return Err(e.into()),
    }
    ```

## Contract: BindingLoader.load_data

### Inputs
- `data`: dict, required — pre-parsed YAML content. Must be a dict with a `"bindings"` key containing a list of binding entries (e.g., `{"bindings": [{...}, {...}]}`). A bare list is rejected with `BindingLoadError`.
- `strict`: bool, optional, default=false — if true, raises `BindingLoadError` on missing required fields (`input_schema`, `output_schema`)

### Errors
- `BindingLoadError` (Python raises, TypeScript throws, Rust returns `Err`) — top-level value is not a mapping, missing `bindings` key, invalid entry structure, or strict-mode violation

### Returns
- On success: `list[ScannedModule]` / `ScannedModule[]` / `Vec<ScannedModule>`

### Properties
- async: false
- pure: true (no filesystem access — operates on already-parsed data)
- thread_safe: true

---

## Contract: BindingLoadError

### Inputs
N/A — this is an exception class, not a callable function.

### Errors
N/A — exception classes are not called and do not raise secondary exceptions.

### Returns
N/A — exception classes are not called and do not return values.

### Fields (Python / TypeScript)
- `file_path` / `filePath`: string | None — path to the `.binding.yaml` file that triggered the error (if applicable)
- `module_id` / `moduleId`: string | None — module ID of the entry that failed (if applicable)
- `missing_fields` / `missingFields`: list[str] / string[] — field names missing in strict mode (empty list for non-strict errors)
- `reason`: string — human-readable description of the error

### Cross-SDK Shape

| SDK | Type | Shape |
|-----|------|-------|
| Python | `class BindingLoadError(Exception)` | Single class with all 4 fields as attributes |
| TypeScript | `class BindingLoadError extends Error` | Same 4 fields as camelCase properties |
| Rust | `enum BindingLoadError` (`thiserror`) | 7 variants: `PathNotFound { path }`, `FileRead { path, source }`, `YamlParse { path, source }`, `MissingFields { path: Option<String>, module_id: Option<String>, missing_fields: Vec<String> }`, `InvalidStructure { path: Option<String>, reason: String }`, `FileTooLarge { path, size, max }`, `TooManyFiles { path, max }` |

Rust callers pattern-match on the variant to recover structured information. Python/TypeScript callers access fields directly.

### Properties
- Python: a plain `class BindingLoadError(Exception)` — a data-carrying exception, not a dataclass; instantiated and raised by `BindingLoader`/`ConventionScanner` internals, never constructed by callers for their own use
- TypeScript: a plain `class BindingLoadError extends Error` — same shape and the same caller relationship as Python's
- Rust: a `thiserror`-derived `enum`, not an exception — errors are returned via `Result<_, BindingLoadError>` rather than thrown; each of the 7 variants carries only the fields relevant to that failure mode, so `missing_fields` (for example) exists only on the `MissingFields` variant rather than as an always-present-but-often-empty field the way Python/TypeScript's single-class shape requires
- immutable once constructed, in all three SDKs
- this Python/TypeScript-vs-Rust shape difference (one class with 4 always-present fields vs. a 7-variant enum) is intentional, not a divergence to reconcile — see [Cross-SDK Shape](#cross-sdk-shape) above

---

## Round-Trip Guarantee

`YAMLWriter.write` followed by `BindingLoader.load` preserves every persisted field: `module_id`, `target`, `description`, `documentation`, `tags`, `version`, `annotations` (including the 12 `ModuleAnnotations` fields such as `streaming`, `cache_ttl`), `examples`, `metadata`, `input_schema`, `output_schema`, and `display`. This is covered by round-trip tests in all three SDKs.

Fields that `YAMLWriter` does not emit (e.g., `warnings`) are not preserved — those default to empty on load.

---

## Contract: BindingLoader.load

### Inputs
- `path`: string or Path, required — path to a `.binding.yaml` file OR a directory containing `.binding.yaml` files
- `strict`: bool, optional, default=false — if true, raises on any malformed binding entry
- `recursive`: bool, optional, default=false — a positional boolean in TypeScript, **not** a `BindingLoadOptions` object (that type belongs to `loadData` / `parseBindingDocument`, never to `load`) — when `true`, all three SDKs walk subdirectories recursively (Python via `Path.rglob`, TypeScript via `_collectRecursive`, Rust via `walkdir::WalkDir`). Governs traversal depth only.
- `pattern`: string, optional, default=`"*.binding.yaml"` — matched against each candidate's **file name**. Ignored when `path` is a file. See [Pattern Matching](#pattern-matching) for the supported syntax and the validation rules. Signatures differ by SDK only in how an optional argument is idiomatically expressed:

    | SDK | Signature |
    |---|---|
    | Python | `load(path, *, strict=False, recursive=False, pattern="*.binding.yaml")` |
    | TypeScript | `load(filePath, strict?, recursive?, pattern?)` |
    | Rust | `load(&self, path, strict, recursive)` unchanged; `load_with_pattern(&self, path, strict, recursive, pattern: Option<&str>)` added |

    Rust gains a second method rather than a fourth parameter because adding one would break every existing caller; `load` delegates to `load_with_pattern(..., None)`. This is the same two-tier shape `apcore` uses for `load_binding_dir` / `load_binding_dir_with_config`.

### Errors
- `BindingLoadError` / `BindingLoadError` (Python raises, Rust returns `Err`) — path not found, YAML parse failure, or strict mode violation
- `BindingLoadError::FileRead` (Rust) — any OS/IO error on the *root* path; per-entry errors during recursive traversal are governed by the policy below
- `BindingLoadError` (Python) — OS errors on the root path wrapping `IOError`/`OSError` raise immediately

### Recursive scan error handling

When `recursive=true` and the scan encounters a per-entry I/O error
(e.g., `EACCES` / `EPERM` on a subdirectory or unreadable file), the canonical
behavior is **best-effort**: emit a warning, skip the unreadable entry, and
continue traversing.

| SDK        | Current behavior                                                                                 | Aligned? |
|------------|--------------------------------------------------------------------------------------------------|----------|
| TypeScript | Best-effort: warn + skip + continue                                                              | ✓        |
| Python     | Best-effort by default — `Path.glob` skips inaccessible subdirectories on most platforms         | partial — platform-dependent |
| Rust       | Currently fail-fast (`BindingLoadError::FileRead`) — pending alignment with best-effort policy   | ✗ pending |

`BindingLoadError` is still raised when the *root* path is inaccessible or
missing — the best-effort policy only applies to per-entry errors encountered
during recursive traversal.

### Returns
- On success: `list[ScannedModule]` / `ScannedModule[]` / `Vec<ScannedModule>` — all modules loaded from the file or directory
- On directory with no `.binding.yaml` files: returns empty list (not an error)

### Properties
- async: false
- pure: false (reads filesystem)
- thread_safe: true (no shared state)

---

## TypeScript-only Extensions

### BindingParser (TypeScript only)

The TypeScript implementation splits binding loading into two classes for browser/edge runtime compatibility:

- **`BindingParser`** — runtime-neutral in-memory parser. Accepts **pre-parsed** binding data (e.g. the result of `yaml.load()`, or a JSON response already parsed via `fetch()` + `.json()`) and returns `ScannedModule[]` directly. Has no filesystem dependency and does **not** parse YAML text itself — YAML-string parsing only happens in `BindingLoader.load()` (Node-only, via `js-yaml`), one layer up. Available via both the main and browser entry points.
- **`parseBindingDocument(raw, options?, filePath?): ScannedModule[]`** — standalone function wrapping `BindingParser.loadData` for callers that don't need class instantiation; the optional `filePath` is embedded in any `BindingLoadError` thrown, for error context. Available via both the main and browser entry points.
- **`BindingLoader`** — extends `BindingParser`, adding the Node.js filesystem-reading entry point (`load(path, strict?, recursive?)`) that reads a file, parses it as YAML via `js-yaml`, and delegates to the inherited parsing logic. Available via the main entry point only.

There is no `BindingDocument` type — both `BindingParser.loadData` and `parseBindingDocument` return `ScannedModule[]` directly, with no intermediate representation.

### Contract: BindingParser.loadData (TypeScript only)

#### Inputs
- `data`: pre-parsed object, required — must be a mapping with a `bindings` key holding an array of binding entries (e.g. `{ bindings: [...] }`). **Not** a raw YAML/JSON string — the caller parses that first.
- `options`: `BindingLoadOptions`, optional — `{ strict?: boolean }` (see the shared `strict`/loose-mode rules above; `recursive` is ignored here, it only applies to `BindingLoader.load`).

#### Errors
- `BindingLoadError` (TypeScript throws) — the top-level value is not a mapping, the `bindings` key is missing or not an array, an entry is missing required fields, or strict-mode validation fails.

#### Returns
- On success: `ScannedModule[]`.
- On failure: throws `BindingLoadError`.

#### Properties
- async: false
- pure: true (no filesystem access — operates purely on the input value)
- thread_safe: true

### Contract: parseBindingDocument (TypeScript only)

A standalone function wrapping `BindingParser.loadData` for callers that don't need to retain a parser instance. Behaviour, errors, and properties are identical to `BindingParser.loadData` above, plus an optional `filePath` for error context.

#### Inputs
- `raw`: pre-parsed object, required — see `BindingParser.loadData`.
- `options`: `BindingLoadOptions`, optional — see `BindingParser.loadData`.
- `filePath`: string or `null`, optional, default `null` — embedded in any `BindingLoadError` thrown for this call, so a caller that knows which file/endpoint a document came from can get that context in the error.

#### Errors
- `BindingLoadError` (TypeScript throws) — see `BindingParser.loadData`.

#### Returns
- On success: `ScannedModule[]`.
- On failure: throws `BindingLoadError`.

#### Properties
- async: false
- pure: true
- thread_safe: true

### Browser / Edge Runtime Subpath

The package exports a `apcore-toolkit/browser` subpath that includes only:
- `BindingParser` and `parseBindingDocument` (filesystem-free)
- All verifiers, formatters, and display utilities that have no Node.js dependencies

Import from the browser subpath in edge runtimes:

```typescript
import { BindingParser } from 'apcore-toolkit/browser';
```

Python and Rust do not have an equivalent browser/edge runtime entry point — these are platform-specific concerns only relevant in the TypeScript/JavaScript ecosystem.

---

## Comparison with `apcore.BindingLoader`

| Aspect | `apcore-toolkit` `BindingLoader` | `apcore` `BindingLoader` |
|--------|-----------------------------------|---------------------------|
| Returns | `list[ScannedModule]` (data) | `list[FunctionModule]` (runtime) |
| Imports `target` | No | Yes (via `importlib.import_module`) |
| Registers modules | No | Yes (mutates a `Registry`) |
| Required fields | `module_id`, `target` (loose) | `module_id`, `target` (always) |
| Use case | Tooling, CI, merging, diffing | Runtime module loading |

Use the toolkit loader when you need to **inspect or transform** binding files. Use `apcore.BindingLoader` when you need to **execute** the modules they describe.
