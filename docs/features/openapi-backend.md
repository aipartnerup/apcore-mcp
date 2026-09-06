---
description: "OpenAPI Backend feature spec: a third apcore-mcp backend source that turns an OpenAPI 3.0/3.1 document into MCP tools via apcore-toolkit's OpenAPIScanner and HTTPProxyRegistryWriter, including module_id/tool-name mapping, the annotation and approval gap, ACL target stability, the collision preflight, the path-typed spec key, and SSRF boundaries."
---

# OpenAPI Backend

> Feature spec for code-forge implementation planning.
> Source: apcore-toolkit 0.11.0 OpenAPI Scanner (`docs/features/openapi-scanner.md` § Phase 3 —
> Consumers), extended to apcore-mcp.
> Created: 2026-09-05

## Purpose

apcore-mcp turns an apcore project into an MCP server. The OpenAPI Backend widens the input side of
that sentence: point the bridge at an **OpenAPI 3.0/3.1 document** and every operation in it becomes
an MCP tool, proxied over HTTP to the API that published the document — with no apcore project on
the other end at all.

The whole path already exists in shipped code. apcore-toolkit 0.11.0 added `OpenAPIScanner`, which
turns a document into `ScannedModule`s with a byte-identical `module_id` derivation across all three
SDKs; `HTTPProxyRegistryWriter` has shipped since 0.7 and registers those modules as executable HTTP
proxies. What is missing is the last link: a bridge entry point that composes the two into a
`Registry` and hands it to the machinery apcore-mcp already has. This feature is that link and
nothing more — it introduces no scanning logic, no schema conversion, and no new execution path.

```
load_spec(url|path) → OpenAPIScanner.scan() → HTTPProxyRegistryWriter.write() → Registry
                                                                                  │
                                              existing bridge, unchanged ─────────┘
                                              (Schema Converter → Annotation Mapper →
                                               Activation → ACL → Approval → Executor)
```

## Scope

**Included:**

- A backend-source entry point per SDK that takes a spec location and returns a populated
  `Registry`, plus a convenience constructor on `APCoreMCP`.
- The `mcp.openapi` Config Bus section and the corresponding CLI flags.
- Mapping rules from a derived `module_id` to an MCP tool name and an OpenAI function name.
- Startup reporting of scan warnings and write failures.
- The governance consequences of scanned modules — annotations, approval, ACL target stability.

**Excluded:**

- The scanner and the writer themselves. Both live in apcore-toolkit; their behaviour, the
  `module_id` derivation algorithm, and the `metadata` execution contract (`http_method` /
  `url_path`) are specified in
  [`apcore-toolkit/docs/features/openapi-scanner.md`](https://github.com/aiperceivable/apcore-toolkit/blob/main/docs/features/openapi-scanner.md)
  and pinned by that repository's 24-case conformance corpus. This spec does not restate them.
- Swagger 2.0. `OpenAPIScanner.scan` refuses anything that is not `3.0.x` / `3.1.x`.
- Obtaining credentials for the upstream API. The writer's `auth_header_factory` hook is where a
  token arrives; producing one is the deployment's problem (apcore-toolkit's `device-auth.md` is a
  proposal, not a shipped feature).
- Generating source code. Static mode persists a reviewable `.binding.yaml`, not stubs.

## Core Responsibilities

1. **Compose, do not reimplement — with one named exception.** The entry point calls `load_spec`
   (or accepts an already-parsed document), `OpenAPIScanner.scan`, and
   `HTTPProxyRegistryWriter.write`, in that order, and returns the registry. Any behaviour
   difference between the three bridges that is not a difference in the toolkit is a defect. The
   exception is the [module-ID projection](#module_id-tool-name), which the bridge owns because
   the scanner's output alphabet is wider than apcore's registry accepts.
2. **Surface what the scan found.** Every `ScannedModule.warnings` entry and every failed
   `WriteResult` MUST reach the operator at startup — WARNING for a scan warning, ERROR for a write
   failure — naming the module ID. A spec with an unresolvable `$ref` produces a tool with an empty
   schema, and an LLM will call it anyway.
3. **Refuse to start on an empty scan only when it was asked for.** A document that yields zero
   modules logs a WARNING and starts (consistent with FR-SERVER-006's zero-module behaviour for an
   extensions directory). A document that fails to load or parse is fatal, and so is a module-ID
   collision against an existing registry — see [ID collisions](#id-collisions-are-a-startup-failure)
   for which failures are fatal and which are per-module.
4. **Namespace by default in mixed deployments.** When an OpenAPI backend is combined with any
   other source, a `prefix` is required — see [ID collisions](#id-collisions-are-a-startup-failure).

---

## Backend sources

| Source | Python | TypeScript | Rust |
|---|---|---|---|
| Extensions directory | ✅ | ✅ | ⚠️ builds an empty registry ([Getting Started §2](../getting-started.md)) |
| `Registry` / `Executor` built in code | ✅ | ✅ | ✅ |
| **OpenAPI document** | ✅ | ✅ | ✅ |

!!! success "The first backend source with full Rust parity"
    Directory auto-discovery is Python/TypeScript-only because apcore's public Rust API has no
    runtime directory discovery. An OpenAPI document needs no discoverer — it is parsed, scanned
    and written through code paths all three toolkits ship — so the Rust bridge reaches feature
    parity here without the caveat that qualifies every `--extensions-dir` sentence in this
    documentation.

---

## `module_id` → tool name

`derive_module_id(path, method, operation)` is byte-identical across the three toolkits and is the
only thing that decides a tool's name:

1. a non-empty `operationId`, sanitized — every character outside `[A-Za-z0-9_.-]` becomes `_`,
   runs of `.` collapse to one, leading/trailing `.` and `_` are stripped;
2. otherwise the path segments (brace-stripped) joined with `.`, plus the method, lowercased and
   sanitized the same way — `/users/{user_id}` + `get` → `users.user_id.get`;
3. otherwise `root.<method>` for an empty path.

…and that output is **not a legal apcore module ID**.

!!! danger "The scanner's alphabet is wider than apcore's, and the canonical Petstore is in the gap"
    `derive_module_id` sanitizes to `[A-Za-z0-9_.-]`. apcore's registry accepts only
    `^[a-z][a-z0-9_]*(\.[a-z][a-z0-9_]*)*$` — *"lowercase, digits, underscores, dots only; no
    hyphens"* — and enforces it at `Registry.register` **and** again at `Executor.call`.

    Measured against apcore 0.30.0 and apcore-toolkit 0.11.1: of nine realistic operation shapes,
    **only two register without repair.** The canonical Swagger Petstore — `listPets`,
    `createPets`, `showPetById` — is entirely in the rejected set. Run end-to-end, its operations
    scan cleanly and then fail registration as per-module `WriteResult`s, leaving an **empty
    registry**. Under this spec's own "log at ERROR and keep going" rule that is a server which
    starts successfully, advertises nothing, and reports one ERROR line per operation.

    This is an upstream contract gap as much as a bridge concern: `OpenAPIScanner` and
    `HTTPProxyRegistryWriter` are documented as an end-to-end pair, and the pair cannot serve the
    reference specification the scanner was verified against — that verification asserted
    byte-identical `ScannedModule` output across SDKs, which never exercised registrability. **It
    should be reported to apcore-toolkit**, where the repair belongs long-term. It lives here now
    because the bridge cannot ship a backend that serves nothing, and because moving it upstream is
    a breaking change to a byte-identical corpus one release old.

**The projection.** Before registration the bridge maps every derived ID into apcore's alphabet:

1. lowercase it;
2. replace `-` with `_`;
3. if every dot-separated segment now matches `^[a-z][a-z0-9_]*$`, use it — otherwise **skip the
   operation** with a WARNING naming the derived ID and the offending segment.

Steps 1-2 are mechanical and lossless up to case. Step 3 is where the bridge stops: repairing a
segment that does not begin with a letter (`/v1/2fa` → `v1.2fa.post`) means **inventing** a
character, which is a naming decision the bridge must not make silently. Skipping rather than
failing, because one unnameable operation must not cost the whole server — and the operator has
`derive_module_id` and `transform_module` to name it themselves.

The projection runs **last, after any caller-supplied `transform_module`**, so the invariant *every
registered module ID is apcore-legal* holds unconditionally; a caller hook that already produces a
legal ID sees it as a no-op. It also runs **before** the scanner's `deduplicate_ids`, because
lowercasing can *create* a collision that the document did not have (`listPets` and `listpets`).

Only then does the bridge apply its **existing, unchanged** mapping: `module_id` becomes the MCP
tool name verbatim (dots and all), overridable per module via `display.mcp.alias`, and is
dash-normalized for OpenAI function names.

| Spec input | Scanner emits | Projected `module_id` | MCP tool | OpenAI function |
|---|---|---|---|---|
| `operationId: listPets` | `listPets` ✗ | `listpets` | `listpets` | `listpets` |
| `GET /pet-store/items` | `pet-store.items.get` ✗ | `pet_store.items.get` | `pet_store.items.get` | `pet_store-items-get` |
| `GET /users/{user_id}` | `users.user_id.get` ✓ | unchanged | `users.user_id.get` | `users-user_id-get` |
| `POST /` | `root.post` ✓ | unchanged | `root.post` | `root-post` |
| `POST /v1/2fa` | `v1.2fa.post` ✗ | **unprojectable** | — skipped with a WARNING — | |

✗ = rejected by apcore's registry as emitted. Note the OpenAI column on row 2: dash-normalization
replaces the **dots** only, so the underscore substituted in step 2 survives.

### ID collisions are a startup failure

`OpenAPIScanner` deduplicates IDs **within one scan** by appending `_2`, `_3`, … and recording a
warning on the renamed module. It knows nothing about modules already in the registry. Since apcore
0.22.0 a duplicate registration is rejected immediately with
`InvalidInputError(code=DUPLICATE_MODULE_ID)` rather than resolved by lock ordering, so an OpenAPI
operation whose derived ID collides with a discovered extension module fails the write.

**A prefix reduces the probability of a collision; it does not eliminate one.** Nothing stops an
extensions directory from already containing a `petstore.*` module, and a duplicate reaches the
writer as a **failed `WriteResult`**, not an exception — the toolkit's writers report per-module
outcomes rather than aborting. Under the "log at ERROR and keep going" rule below, that would leave
the server running with a **partial registry**: some operations served, one silently absent, and a
tool the agent was told about missing from `tools/list`.

Two requirements, and they are not the same mechanism:

1. **`prefix` is mandatory in a mixed-source deployment.** When the OpenAPI backend is combined
   with any other backend source, the bridge MUST refuse to start without one. When it is the only
   source, `prefix` is optional. The prefix is applied by the scanner (`base_path_prefix`) and
   yields `petstore.users.user_id.get`.
2. **A full-set collision preflight runs before the first write, and a collision is fatal.** After
   the scan and before `HTTPProxyRegistryWriter.write`, the bridge MUST intersect the complete set
   of derived module IDs against the IDs already in the target registry and, on a non-empty
   intersection, fail startup naming **every** colliding ID — not the first. Nothing is written.
   The set is fully known at that point, the check is a set intersection, and making it atomic is
   what keeps "collision" from degrading into "partial registry".

!!! info "Which failures are fatal, and why they are not the same class"
    | Failure | Detected | Outcome |
    |---|---|---|
    | Derived ID collides with a module already in the registry | Preflight, before any write | **Fatal**, atomic, all colliding IDs named. Nothing registers |
    | Two operations in **one document** derive the same ID | Inside the scanner | Renamed `_2`, `_3`, … with a warning on the module. Never reaches the preflight |
    | One module's schema or proxy construction fails | `WriteResult` | ERROR logged naming the module; the rest still register |

    The distinction is whether the deployment is **coherent**. A colliding ID means the operator's
    two sources disagree about what a name means, and there is no correct answer for the bridge to
    pick — serving either one silently is a worse outcome than refusing. A module that fails to
    build is a defect in one operation, and dropping it costs one tool.

    This is a **bridge** decision, not a writer behaviour: apcore-toolkit's writers return
    per-module `WriteResult`s by contract and do not distinguish these cases, so the escalation has
    to happen here.

---

## Governance

This is the part of the feature that is not mechanical. An OpenAPI document is a description of an
API's *shape*; it says almost nothing about the *consequences* of calling it, and the bridge must
not let the resulting silence read as safety.

### Every scanned module arrives with `requires_approval = false`

The toolkit infers annotations from the HTTP method alone (RFC 9110 §9.2 safe-method semantics):

| Method | Inferred annotations | MCP `ToolAnnotations` |
|---|---|---|
| `GET` | `readonly`, `cacheable` | `readOnlyHint: true` |
| `HEAD`, `OPTIONS` | `readonly` | `readOnlyHint: true` |
| `PUT` | `idempotent` | `idempotentHint: true` |
| `DELETE` | `destructive` | `destructiveHint: true` |
| **`POST`, `PATCH`, `TRACE`** | **none** | all hints false |

`open_world` defaults to `true`, which is correct — these modules do call an external system — so
`openWorldHint: true` is right without anyone setting it.

`requires_approval` is **never** inferred, for any method. A `POST /charges` that moves money is
annotated exactly like a `POST /echo`, and the approval gate does not fire for either.

!!! danger "The approval gate is silent on an OpenAPI backend unless you configure it"
    The scanner cannot know which operations are consequential, and guessing from the method would
    be wrong in both directions — most `POST`s are harmless, some `GET`s are not. So it declines to
    guess, and the module-level declaration stays `false`.

    Three ways to close it, in increasing order of precision:

    1. **An ACL rule carrying `approval: required`** (apcore 0.28.0, PROTOCOL_SPEC §6.1.6) — puts a
       matching call to a human even though the rule's `effect` is `allow`. Targets a pattern, so
       it scales to "every write operation" if the IDs are named consistently. See
       [ACL Builder](./acl-builder.md).
    2. **`gate_destructive` on the `ExecutionPolicy`** — catches `DELETE` operations, which are the
       only ones the toolkit marks `destructive`. Catches no `POST`.
    3. **A `transform_module` hook** on the scan — the operator's own callable, which can set
       `requires_approval` per operation from anything in the document (a `x-requires-approval`
       vendor extension, a tag, the path). Precise, and the only option that puts the annotation on
       the module itself.

    The bridge **MUST** log a WARNING at startup when an OpenAPI backend registers any module whose
    method is in `{POST, PUT, PATCH, DELETE}` and no approval path is demonstrable — the same class
    of warning as the unprotected-control-surface guard, and, importantly, on the same predicate.

!!! danger "An attached ACL is not proof that anything will ask for approval"
    apcore's own `GovernanceState.unprotected_control_surface` states the rule this warning has to
    follow: it *"reports the **absence of a gate**, never the presence of protection: a wired ACL
    that permits every call still yields `False`."* An earlier draft of this spec suppressed the
    warning whenever an ACL was attached. That was wrong in exactly the way apcore's accessor
    warns about — `default_effect: allow`, an ACL whose `targets` never match these modules, or an
    ACL that allows without carrying `approval: required` anywhere, each leaves every write
    operation ungated while satisfying "an ACL is attached".

    **The warning is suppressed only by:**

    - no module with a method in `{POST, PUT, PATCH, DELETE}` in the scanned set; or
    - every such module declaring `requires_approval` itself — reachable only through a
      `transform_module` hook, since the scanner never infers it; or
    - an explicit operator acknowledgement, `mcp.openapi.acknowledge_unapproved_writes: true`,
      which is a recorded decision rather than an inference.

    **It is never suppressed by `governance_state().acl_configured`.** When an ACL *is* attached and
    at least one rule carries `approval: required`, the message is **downgraded in wording** — an
    approval path exists; whether its `targets` cover these modules is a match-relation question —
    but still emitted. That is the same boundary [ACL Builder tier 2](./acl-builder.md#tier-2-rules-that-load-and-protect-nothing)
    draws: the shape of a rule is decidable, what it matches is not, and a predicate that cannot be
    closed must not be used to silence a safety warning.

    **It is escalated** when `governance_state().builtin_approval_gate_wired` is `false` — under the
    `internal`, `testing` and `minimal` strategies the approval gate is not in the pipeline at all,
    so even a module-level `requires_approval` would not fire, and neither would an ACL rule's
    `approval: required`.

### ACL targets and the stability of a derived ID

An ACL `targets` pattern is matched against the `module_id`. On an extensions directory the operator
writes both; on an OpenAPI backend, **the upstream API's authors decide the left-hand side**. A
document that renames `deleteUser` to `removeUser` silently changes the module ID, and a
`targets: ["deleteUser"]` deny rule stops matching — which under `default_effect: allow` is a
fail-open of exactly the shape apcore#112 closed inside the ACL itself.

**Required guidance, and the default the bridge should make easy:**

- Set `prefix` and write the catch-all rule against the prefix, not against operation names:

  ```yaml
  mcp:
    openapi:
      spec: "https://api.example.com/openapi.json"
      prefix: petstore
    acl:
      default_effect: deny
      rules:
        - callers: ["@external"]
          targets: ["petstore.pets.get", "petstore.pets.pet_id.get"]
          effect: allow
          conditions: { identity_types: ["human"] }
        - callers: ["@external"]
          targets: ["petstore.*"]
          effect: deny
  ```

  A prefixed catch-all deny holds no matter what the upstream renames. An allow-list of operation
  names fails **closed** when an ID changes — the safe direction — which is why `default_effect:
  deny` plus a prefixed catch-all is the recommended shape and not merely a stylistic preference.
- **`targets: []` is no longer a way to say "everything".** apcore 0.29.0 rejects it at every door.
  Write `["*"]`. See [ACL Builder § Pattern-array shape closure](./acl-builder.md#pattern-array-shape-closure-apcore-0290).
- Static mode (below) removes the moving part entirely: the IDs are frozen in a reviewable file and
  a rename shows up as a diff in a pull request.

---

## Dynamic and static modes

**Dynamic (default).** The spec is fetched and scanned at every process start. Zero redeploy when
the API changes; one fetch and parse per start; a network dependency in the startup path.

**Static.** Scan once at build time into `.binding.yaml` (apcore-toolkit's `YAMLWriter`), load it at
runtime with **apcore-toolkit's** `BindingLoader` and write it through the same
`HTTPProxyRegistryWriter`. No network dependency at start, no parse cost, and the generated bindings
are reviewable artifacts — an API contract change becomes visible to a human before it ships, and
the module IDs an ACL rule depends on stop moving on their own.

!!! warning "Two different classes are named `BindingLoader`"
    `apcore_toolkit.BindingLoader` and `apcore.BindingLoader` are unrelated implementations that
    share a name. This spec means the **toolkit's**, which takes its file path from the caller.

    apcore 0.30.0 gave *apcore's* `BindingLoader` a normative directory-resolution contract
    (PROTOCOL_SPEC §5.12.6): invoked without an explicit directory it MUST resolve from
    `bindings.dir` under §9.2 precedence — `APCORE_BINDINGS_DIR` > configuration file > `./bindings`
    — match against `bindings.pattern` (default `*.binding.yaml`) through the same chain, raise
    rather than return empty when the resolved directory is absent, and never auto-scan at
    initialisation. **None of that governs the toolkit's class**, which imports only
    `ModuleAnnotations` and `ModuleExample` from apcore and has no `bindings.dir` awareness. A
    deployment that sets `APCORE_BINDINGS_DIR` expecting it to steer this path will find it
    ignored.

Both writers and the loader already ship. The bridge supports static mode by accepting an
already-parsed document or a pre-built `Registry`; it needs no new machinery for it.

---

## Configuration

### Config Bus (`mcp.openapi`)

```yaml
mcp:
  openapi:
    spec: "https://api.example.com/openapi.json"   # URL or local path — required; path-typed
    base_url: "https://api.example.com"            # defaults to servers[0].url from the document
    prefix: petstore                               # base_path_prefix; required in mixed deployments
    include: "pets.*"                              # scanner include filter
    exclude: "*.internal.*"                        # scanner exclude filter
    include_deprecated: false                      # default true
    acknowledge_unapproved_writes: false           # default false; see Governance below
    timeout: 30.0                                  # spec fetch timeout, seconds
    headers:                                       # sent with the spec fetch only
      X-Api-Key: "${PETSTORE_SPEC_KEY}"
```

| Key | Default | Notes |
|---|---|---|
| `spec` | — | URL or filesystem path. A URL is taken **verbatim**; a relative path resolves against `Config.project_root`, and a set-but-empty value is discarded. No candidate paths are probed. See [The spec location is a path-typed key](#the-spec-location-is-a-path-typed-key) |
| `base_url` | `servers[0].url` | Where requests go. Required when the document has no usable absolute `servers[0].url` |
| `prefix` | `null` | Prepended to every derived `module_id`. **Required** when another backend source is also configured |
| `include` / `exclude` | `null` | Passed to the scanner's own filters |
| `include_deprecated` | `true` | `false` skips operations marked `deprecated: true` |
| `timeout` | `30.0` | Spec fetch only; not the per-call proxy timeout |
| `headers` | `{}` | Spec fetch only. **Not** forwarded to proxied calls — see below |
| `acknowledge_unapproved_writes` | `false` | Records an explicit operator decision to run write-method operations with no approval path, suppressing the startup warning. The only thing that suppresses it besides having nothing to warn about |

### CLI

| Argument | Default | Description |
|---|---|---|
| `--from-openapi` | — | Spec URL or path. Mutually exclusive with `--extensions-dir` unless `--openapi-prefix` is given |
| `--openapi-base-url` | from document | Base URL for proxied requests |
| `--openapi-prefix` | — | `base_path_prefix` for every derived module ID |
| `--openapi-include` / `--openapi-exclude` | — | Scanner filters |
| `--openapi-header` | — | `Key: Value`, repeatable. Spec fetch only |
| `--openapi-no-deprecated` | off | Skip `deprecated: true` operations |

Precedence is the bridge's existing caller-wins rule: an explicit CLI flag or constructor argument
beats the Config Bus, which beats the default.

### The spec location is a path-typed key

`mcp.openapi.spec` holds either an `http(s)://` URL or a filesystem path, and it is the **first
path-typed key in the `mcp` namespace**. apcore 0.30.0 declared the closed set of path-typed
configuration keys (PROTOCOL_SPEC §9.2.1) and the base a relative one resolves against (§9.2.2) —
but both apply to apcore's own key surface, and neither extends here.

!!! danger "apcore 0.30.0's path protections do not reach a consumer namespace"
    Measured against apcore 0.30.0, not inferred from the specification:

    - `Config.path_typed_keys()` returns a **hardcoded tuple** of apcore's own four keys
      (`extensions.root`, `extensions.roots[]`, `schema.root`, `acl.root`, `bindings.dir`). It never
      consults a namespace registered through `Config.register_namespace`, even though that method
      accepts a `schema` and could carry `"x-apcore-path": true`.
    - The §9.2.1 requirement-5 empty-value discard is gated on that same fixed set, so
      `APCORE_MCP_OPENAPI_SPEC=` is treated as an ordinary override to `""` — a legal relative path
      to every filesystem API, and never the one an operator meant. This is precisely the failure
      apcore closed for `APCORE_ACL_ROOT=`, in a namespace the fix does not cover.

    The bridge therefore owns these rules rather than inheriting them.

**Three requirements.**

1. **Discriminate before resolving.** A value beginning `http://` or `https://` is a URL and is
   used verbatim — never path-resolved, never made absolute. Anything else is a filesystem path.
   The test is the scheme prefix, not a guess from the string's shape.
2. **An empty value is not a path.** A set-but-empty `mcp.openapi.spec` — from
   `APCORE_MCP_OPENAPI_SPEC=` or from `spec: ""` in YAML — MUST be discarded with a WARNING and
   resolution MUST fall through to the next tier, mirroring §9.2.1 requirement 5. It MUST NOT be
   treated as "the working directory", which is what an unguarded path join produces.
3. **A relative path resolves against `Config.project_root`.** Not the process CWD, and not the
   OpenAPI document's own directory. `project_root` is apcore 0.30.0's public accessor
   (`Config.project_root` / `projectRoot()` / `Config::project_root()`), carrying no resolution
   behaviour of its own — it reports the base, and the bridge applies it.

!!! success "A new key gets the destination, not the deprecation cycle"
    apcore's §9.2.2 explicitly **forbids** its own four keys from adopting the project-root rule
    early: they have deployed configurations relying on today's two different bases, so §13.2's
    two-minor deprecation window applies and the 1.x line keeps current semantics exactly.

    `mcp.openapi.spec` is in the opposite position. It has never shipped, so its deployed
    population is empty and there is nothing to deprecate — adopting the target semantics now costs
    nobody a migration and avoids minting a fifth key with a base that is already scheduled to
    change. The prohibition is on changing keys people depend on, not on getting a new one right
    the first time.

    The consequence to state plainly: for the whole 1.x line, `mcp.openapi.spec` and `acl.root`
    resolve a relative value against **different bases** under §9.14 discovery tier 1 — the OpenAPI
    key against the project root, `acl.root` against the configuration file's directory. Under
    tiers 2-5 the two coincide. This is a deliberate, documented consequence of adopting early, not
    an oversight, and it ends when apcore 2.0 moves its own keys to the same base.

### Upstream credentials

`headers` and `--openapi-header` authenticate the **spec fetch**. They are deliberately not reused
for proxied calls: a document is often public while the API behind it is not, and quietly
broadcasting a spec-read key on every tool call would be a privilege escalation the operator never
wrote down.

Credentials for proxied calls arrive through `HTTPProxyRegistryWriter`'s `auth_header_factory`
hook, which is a callable, invoked per request, and therefore the correct place for a rotating
token. It is reachable from the programmatic API only — there is no CLI flag for it, and there
should not be one, because a static bearer token on a command line is the failure mode the hook
exists to avoid.

---

## Security considerations

- **SSRF is the caller's boundary, and the bridge must keep it there.** `load_spec` takes its
  source verbatim and does no allowlisting; the toolkit documents *source* as trusted input. The
  spec location MUST therefore come only from operator-controlled configuration — a CLI flag, the
  Config Bus, or a constructor argument. It MUST NOT be reachable from an MCP request, an Explorer
  UI form field, or any other request-scoped input, in any SDK.
- **External `$ref`s are never fetched.** The scanner records a warning and leaves the reference
  unresolved. A spec that leans on external references produces tools with incomplete schemas —
  which is why the scan warnings are a startup WARNING and not a debug line.
- **Upstream error bodies flow back through [Error Mapper](./error-mapper.md).** The proxy returns
  the upstream API's error message; the bridge's existing sanitization and redaction apply
  unchanged. An upstream that echoes credentials into an error body is not made safe by this
  feature, and the deployment should assume its own redaction rules are the last line.
- **A GET with a body is not the risk; a POST with a query parameter is.** The writer partitions
  inputs by HTTP method — `POST`/`PUT`/`PATCH` send a JSON body, everything else a query string —
  rather than by what the document declares. A query parameter declared on a `POST` will be sent in
  the body. This is a known, documented limitation of the shipped writer, not something the bridge
  can correct.

---

## Interfaces

### Inputs

- **spec source** — `str | Path` (URL or path) or an already-parsed document
  (`dict` / `object` / `serde_json::Value`).
- **options** — base URL, prefix, include/exclude, deprecated handling, headers, timeout,
  `auth_header_factory`, and the scanner's `transform_operation` / `transform_module` /
  `derive_module_id` hooks, passed through verbatim.

### Outputs

- **`Registry`** — populated and ready to hand to `APCoreMCP` / `serve()` as a backend.

### Dependencies

- **apcore-toolkit >= 0.11.1** — `load_spec`, `OpenAPIScanner`, `derive_module_id`,
  `HTTPProxyRegistryWriter`. **0.11.0** is the capability floor: `OpenAPIScanner` does not exist
  below it, and it is where the Rust writer stopped rejecting `HEAD` / `OPTIONS` / `TRACE` before
  any network call — a defect that would have made the Rust path quietly broken for those
  operations. **0.11.1** changes no toolkit API at all; it exists to raise its own apcore floor to
  0.30.0, which is why this bridge's apcore floor moves with it.
- **`httpx` / a fetch implementation** — for the spec fetch and for the proxy itself.
- **apcore >= 0.30.0** — the registration and governance surface (see
  [ACL Builder](./acl-builder.md)), plus `Config.project_root`, which 0.30.0 introduces and which
  `mcp.openapi.spec` resolves a relative path against.

!!! note "Python needs a new extra; TypeScript and Rust do not"
    apcore-toolkit is an optional extra in Python (`apcore-mcp[markdown]`) and an unconditional
    dependency in TypeScript and Rust. This feature adds a Python extra:

    ```toml
    openapi = ["apcore-toolkit[http-proxy]>=0.11.1"]
    ```

    `[http-proxy]` pulls `httpx`, which both `load_spec` and the proxy need. YAML specs need no
    extra — apcore already depends on `pyyaml`. Every import stays lazy, and the entry point MUST
    fail with an actionable message (`install 'apcore-mcp[openapi]'`) rather than an `ImportError`
    traceback, matching how the Markdown extra already degrades.

---

## Contract: openapi_backend

> Build a `Registry` from an OpenAPI document. Names per language: Python
> `openapi_backend(spec, *, base_url=None, ...)`, TypeScript `openapiBackend(spec, options?)`,
> Rust `openapi_backend(spec, options)`.

### Inputs

- spec: str | Path | parsed document, required — a URL, a filesystem path, or an already-parsed
  OpenAPI 3.0/3.1 document. An `http(s)://` value is taken verbatim; a relative filesystem path
  resolves against `Config.project_root`; a set-but-empty value is discarded with a WARNING and
  falls through to the next configuration tier. No candidate paths are probed.
- base_url: string, optional, default `servers[0].url` from the document — where proxied requests go.
- prefix: string, optional, default none — `base_path_prefix` applied to every derived module ID.
- include / exclude: string, optional — scanner filters.
- include_deprecated: bool, optional, default `true`.
- headers: mapping, optional — sent with the spec fetch only.
- timeout: number, optional, default `30.0` — spec fetch timeout in seconds.
- auth_header_factory: callable, optional — invoked per proxied request; forwarded to the writer.
- transform_operation / transform_module / derive_module_id: callable, optional — forwarded to
  `OpenAPIScanner.scan` verbatim, in the toolkit's documented invocation order.

### Errors

- `ValueError` / `Error` / `ConfigError` — the document is not OpenAPI 3.0.x / 3.1.x, is malformed,
  or cannot be parsed. Fatal.
- Transport / IO error from the spec fetch or file read — propagated. Fatal.
- `ValueError` / `Error` / `ConfigError` — `base_url` is absent and the document has no usable
  absolute `servers[0].url`. Fatal, because every proxied call would otherwise resolve against an
  unknown host.
- `ValueError` / `Error` / `ConfigError` — another backend source is configured and `prefix` is
  not. Fatal.
- Missing toolkit (Python only) — raised with an actionable "install `apcore-mcp[openapi]`" message.
- `ValueError` / `Error` / `ConfigError` — one or more derived module IDs collide with IDs already
  in the target registry. Raised by the pre-write preflight, naming every colliding ID, before
  anything is written. Fatal and atomic.
- A failed `WriteResult` for an individual module is **not** an error: it is logged at ERROR,
  naming the module ID, and the remaining modules still register. This covers schema and proxy
  construction failures only — a duplicate ID never reaches it, having been caught by the
  preflight above.

### Returns

- On success: a `Registry` containing one module per surviving operation. Every `ScannedModule` the
  scanner produces carries `metadata.http_method` (uppercase), `metadata.url_path`, and
  `metadata.openapi.*` as the toolkit specifies — but whether that `metadata` survives into the
  *registered descriptor* (`get_definition(id).metadata`) is **not uniform across languages**: Rust's
  writer builds a full `ModuleDescriptor` and preserves it; Python's and TypeScript's writers call the
  2-arg `register(id, module)` form and drop it. See [Known cross-language
  divergences](#known-cross-language-divergences). Scan warnings are logged at WARNING; a zero-module
  result logs a WARNING and returns the empty registry.

### Properties

- async: false (Python, Rust — `load_spec` is synchronous there); true (TypeScript — `loadSpec` is
  async, so `openapiBackend` returns a Promise)
- thread_safe: true (builds and returns a fresh registry)
- pure: false (network / filesystem I/O unless handed a parsed document; logs)
- idempotent: false (a second call against a live spec URL may see a different document)

---

## Contract: APCoreMCP.from_openapi

> Convenience constructor: `openapi_backend(...)` followed by the ordinary `APCoreMCP` construction.
> Names per language: Python `APCoreMCP.from_openapi(spec, **options)`, TypeScript
> `APCoreMCP.fromOpenapi(spec, options?)`, Rust `APCoreMCP::builder().openapi(spec, options)`.

### Inputs

- spec and options: as `openapi_backend` above, plus every option the ordinary `APCoreMCP`
  constructor accepts (name, version, approval mode, strategy, output formatter, …).

### Errors

- As `openapi_backend`, plus every error the ordinary constructor raises.

### Returns

- On success: a constructed `APCoreMCP` whose registry is the scanned one. `serve()`,
  `async_serve()` and `to_openai_tools()` behave exactly as they do for any other backend — this
  constructor adds no behaviour beyond the registry it builds.

### Properties

- async: mirrors `openapi_backend` per language
- thread_safe: true
- pure: false
- idempotent: false

---

## Conformance

New shared fixture: `conformance/fixtures/openapi_backend.json`, contract_version 1.0. **Driven by
all three bridges** as of 0.20.0.

The scanner's own behaviour is already pinned by apcore-toolkit's 24-case `openapi_scan.json`
corpus and its Petstore end-to-end check; this fixture pins only what apcore-mcp adds on top. Its
expectations were computed by running apcore-toolkit 0.11.1's real scanner, not derived by reading
the algorithm. Three sections, each with its own shape:

**`test_cases` — document → modules (9)**

| Case | Asserts |
|---|---|
| `operation_id_projected_to_apcore_id` | `listPets` → `listpets`. The case the projection exists for: unprojected, apcore refuses it and the server serves nothing |
| `path_derived_id_dash_normalized_for_openai` | `users.user_id.get` → MCP verbatim, OpenAI `users-user_id-get` |
| `hyphen_projected_to_underscore` | `pet-store.items.get` → `pet_store.items.get`; OpenAI dash-normalizes the **dots only**, so the substituted underscore survives |
| `unprojectable_segment_skipped_with_warning` | `/v1/2fa` skipped with a WARNING; the rest of the document still registers. The scanner records nothing here, so the warning is the bridge's — an implementation that merely drops the module passes every other case and fails this one |
| `projection_collision_deduplicated` | `listPets` + `listpets` → `listpets`, `listpets_2`. Pins that the projection runs **before** `deduplicate_ids` |
| `prefix_applied_to_every_id` | `prefix` prepends before projection, on both surfaces |
| `get_maps_to_readonly_hint` | `GET` → `readOnlyHint: true`, `openWorldHint: true` |
| `delete_maps_to_destructive_hint` | `DELETE` → `destructiveHint: true` |
| `post_carries_no_behavioral_hint` | `POST` → all four hints at protocol defaults, `requires_approval` false — the gap pinned as a fact, so it cannot be closed accidentally in one language only |

**`config_cases` — how the `spec` value resolves (3)**

| Case | Asserts |
|---|---|
| `spec_url_not_path_resolved` | an `https://` value survives verbatim — never joined to `project_root` |
| `relative_spec_resolves_against_project_root` | a relative path resolves against `Config.project_root`, not CWD, under a tier-1 config outside CWD |
| `empty_spec_value_discarded` | a set-but-empty `spec` is discarded with a WARNING and falls through, never resolving to the project root |

**`error_cases` — fatal configurations (4)**

| Case | Asserts |
|---|---|
| `swagger_2_rejected` | error surfaced from the scanner; only the fragment naming 3.0/3.1 is pinned |
| `no_base_url_anywhere_rejected` | no `base_url` option and no absolute `servers[0].url` |
| `missing_prefix_in_mixed_deployment_rejected` | two backend sources, no `prefix` |
| `id_collision_against_registry_rejected` | the pre-write preflight names **both** seeded collisions and registers nothing — an implementation reporting only the first forces one restart per collision |

Deliberately **not** in the shared fixture: the wording of the toolkit's own scan errors, and the
proxy's request shaping (body-vs-query partitioning, path-parameter encoding). Both belong to
apcore-toolkit's corpus, and pinning them here would pin another repository's text.

## Known cross-language divergences

- **`loadSpec` is async in TypeScript** and synchronous in Python and Rust, so `openapiBackend`
  returns a `Promise` there and a value in the other two. The convenience constructor inherits this.
- **Toolkit dependency shape** — Python extra (`apcore-mcp[openapi]`) vs. unconditional dependency
  in TypeScript and Rust. Only Python has a "toolkit not installed" path, and it must be an
  actionable message rather than an `ImportError`.
- **Rust `load_spec` has a two-function surface** — `load_spec(url)` for the zero-config case and
  `load_spec_with_options(url, LoadSpecOptions)` for headers and timeout. Python and TypeScript
  take keyword options on one function. The bridge's `openapi_backend` presents one surface in all
  three and routes to whichever the toolkit exposes.
- **Registered `metadata` visibility is Rust-only.** apcore-toolkit-rust's `HTTPProxyRegistryWriter`
  builds a full `ModuleDescriptor` (required by apcore-rust's `Registry::register` signature) and
  copies `ScannedModule.metadata` into it, so `get_definition(id).metadata` carries `http_method` /
  `url_path` / `openapi.*` there. apcore-toolkit-python's and apcore-toolkit-typescript's writers
  both call the 2-arg `register(id, module)` form, which never attaches metadata to the registered
  descriptor at all — measured directly, not assumed: TypeScript's constructed module *does* carry a
  `metadata` property, but apcore-js's `Registry.register` never reads it off the instance, only an
  unused explicit parameter. The proxy itself still routes correctly in all three languages — it
  closes over the scanned values directly, independent of what the registry exposes. This is an
  upstream (apcore-toolkit) inconsistency, not something this bridge can fix; `conformance/fixtures/
  openapi_backend.json`'s test cases assert module names and annotations only, not registered
  metadata, for exactly this reason.

## See Also

- [ACL Builder](./acl-builder.md) — writing rules whose `targets` survive an upstream rename.
- [Annotation Mapper](./annotation-mapper.md) — how the inferred annotations reach
  `ToolAnnotations`.
- [Schema Converter](./schema-converter.md) — the scanner pre-resolves internal `$ref`s, so the
  converter sees a flat schema; an unresolvable ref reaches it as an empty one.
- [Error Mapper](./error-mapper.md) — how an upstream HTTP error becomes an MCP error.
- apcore-toolkit `docs/features/openapi-scanner.md` — the scanner's normative specification,
  `module_id` derivation, `metadata` execution contract, and error model.
- apcore-toolkit `docs/features/output-writers.md` § `metadata` — the `http_method` / `url_path`
  contract and the uppercase requirement.
