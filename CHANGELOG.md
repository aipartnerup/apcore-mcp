# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).


## [0.19.0] - 2026-09-01

Spec-contract release, driven by apcore 0.28.0 and three open issues on `system.*` management
modules (aiperceivable/apcore-mcp#14, #15, #16).

### Added — contract

- **System Management Extension (`com.aiperceivable/management`), Phase A only** — new spec document
  [`docs/features/system-management-extension.md`](docs/features/system-management-extension.md).
  Defines an unofficial ([SEP-2133](https://modelcontextprotocol.io/seps/2133-extensions)) MCP
  extension a conforming adapter advertises in `capabilities.extensions` during `initialize`, if and
  only if at least one `system.*` module is reachable. The extension is pure discovery metadata: a
  client that does not declare it **MUST** still reach every management resource and tool, subject
  only to Activation/ACL/Approval — PROTOCOL_SPEC §6.6.3 already forbids adapters from inventing a
  permission switch beyond those three layers, and a capability-gated access difference would be
  exactly that. Resolves aiperceivable/apcore-mcp#16 Phase A (Phases B and C — Working Group
  discussion and formal SEP submission — are out of scope here and tracked by their own follow-ups).

### Changed — contract

- **`system.health.*` / `system.usage.*` / `system.manifest.*` now project as MCP resources, not
  tools; `system.control.*` is unchanged.** `docs/features/mcp-server-factory.md`'s "Bijective
  Mapping" constraint previously had no exception and described all nine `system.*` modules as tools.
  Classification is by `module_id` prefix only, per PROTOCOL_SPEC §6.6.2 — an adapter **SHOULD NOT**
  introduce its own classification mechanism. The three summary/full modules are static resources
  (`apcore://system.health.summary`, `apcore://system.usage.summary{?period}`,
  `apcore://system.manifest.full`); the three per-module modules are resource templates
  (`apcore://system.{health,manifest}.module/{module_id}`,
  `apcore://system.usage.module/{module_id}{?period}`). A `resources/read` on any of these **MUST**
  dispatch through the same Activation → ACL → Approval → Executor pipeline a `tools/call` would use —
  never a second, resource-only path that bypasses ACL or the audit trail. Resolves
  aiperceivable/apcore-mcp#15(a); tracked per-language in apcore-mcp-typescript#9,
  apcore-mcp-python#8, apcore-mcp-rust#6.
- **New required startup guard: the unprotected-control-surface warning.** `docs/features/mcp-server-factory.md`
  now requires `serve()` / `async_serve()` to call `Executor.governanceState()` /
  `governance_state()` once the executor is fully assembled and, when
  `.unprotectedControlSurface` / `.unprotected_control_surface` is `true`, emit a prominent
  (not fatal) startup warning naming the specific gaps and the configuration that closes each one.
  This accessor was itself blocked on aiperceivable/apcore#97 until apcore 0.28.0 shipped it; the
  warning was the reason #15(b) tracked that dependency instead of re-deriving the predicate from
  executor internals.

### Fixed — spec chain vs. implementation

- **ACL rule template documented a `sys.*` namespace that was never registered.** The canonical IDs
  are `system.health.*` / `system.usage.*` / `system.manifest.*` / `system.control.*` — there is no
  `sys.` prefix. A copied **deny** rule using the wrong namespace never matches, so it silently fails
  to block anything, which is the dangerous direction. Fixes aiperceivable/apcore-mcp#14; the
  corrected three-rule template (read-only allow, control allow, catch-all deny) and the mechanism
  notes about `@external` normalization and JWT-derived `identity_types`/`roles` ship in all three
  adapters' `acl_builder` doc comments.

### Added — conformance

- **`conformance/fixtures/acl_config.json` gained `approval` contract cases (`contract_version` 1.0 → 1.1).** The `approval` rule key (apcore 0.28.0, PROTOCOL_SPEC §6.1.6) shipped in all three bridges with nothing pinning its accepted-value set, and all three implemented it independently: Python accepted apcore's full set (`required` / `not_required`), while Rust and TypeScript each narrowed it to `required` only, on the reasoning that `mcp.acl` should have exactly one spelling for "no approval requirement here". That made those two bridges stricter than the schema they bridge — a rule that loads from apcore's own `acl/` directory failed at startup when carried through the Config Bus — so the same `mcp.acl` YAML started a Python bridge and refused to start a Rust or TypeScript one. Three cases now pin the contract: `approval: required` accepted, `approval: not_required` accepted, and a value outside the closed set rejected. The `deny` + `approval: required` rejection is deliberately left to per-language tests: it originates in apcore's own constructor and its message wording differs per language, so a shared `expected_error_substring` would be pinning apcore's phrasing rather than the bridge's behavior.
- **New shared fixture `conformance/fixtures/system_surface.json`.** Pins the nine canonical
  `system.*` modules' exact MCP primitive (tool / resource / resource template) and name/URI,
  operationalizing #15's "byte-identical `tools/list`, `resources/list`, `resources/templates/list`
  across TypeScript, Python and Rust" acceptance criterion as a regression fixture each adapter's
  own test suite drives, rather than a one-time manual comparison. Finding this fixture useful
  immediately: driving it against the three adapters caught `system.usage.module`'s resource
  template missing the RFC 6570 `{?period}` query-expansion suffix in the Python and Rust
  implementations (`apcore-mcp-typescript` had it right) — exactly the class of divergence this
  fixture exists to catch. See each adapter's own CHANGELOG for the fix.

### No SDK behaviour asserted by this document alone

As with 0.18.0, this document records the contract; conformance is the per-language implementation
tracked in the sub-issues above. See each adapter's own CHANGELOG for what actually shipped at its
current version.

## [0.18.0] - 2026-08-19

Spec-correctness release. A full cross-language sync round audited every feature spec and the
PRD/SRS/tech-design/test-plan chain against all three SDKs at 0.17.2 and found the spec layer had
drifted from the shipped implementations in both directions — requirements describing signatures no
SDK has, and specs still describing behaviour that had already been fixed in code. This release
realigns the documentation with what the bridges actually do, resolves the contradictions the audit
surfaced, and settles one genuine contract question.

**No SDK release accompanies this one.** Nothing here changes Python or TypeScript behaviour. The
one item that obligates an implementation change is the `mcp_` extras separator below, which Rust
must adopt.

### Changed — contract

- **`mcp_` extras separator is now normatively `\n`, not `\n\n`.** The doc chain disagreed with
  itself: `features/annotation-mapper.md` specified `\n\n` while SRS FR-EXTRAANNOT-001 and tech
  design §6.2 specified `\n`. Each side had followers — Rust implemented `\n\n`, Python and
  TypeScript implemented `\n` — so the code divergence was downstream of a spec divergence. `\n`
  wins: it was already the majority on both the doc side and the implementation side. For
  `extra = {"mcp_a": "1", "mcp_b": "2"}` with one non-default flag the correct suffix is
  `"\n\n[Annotations: readonly=true]\na: 1\nb: 2"`.
  **Action required (Rust):** `to_description_suffix` (`src/adapters/annotations.rs`) must join the
  extras with `\n` inside the annotations paragraph rather than emitting each as its own
  `\n\n`-separated section, and the tests pinning the old shape move with it. Python and
  TypeScript are already conformant. `annotation-mapper.md` also now requires **ordinal** (code
  point) sorting of the extras keys rather than locale collation, which varies by host.

### Fixed — spec chain vs. implementation

- **Output redaction with an empty `output_schema` (security-relevant).** Tech design §6.16 stated
  redaction returns "the original output unchanged if output_schema is None or `{}`", and SRS
  FR-REDACT-001's boundary conditions agreed — directly contradicting FR-REDACT-002,
  TC-REDACT-002, and the measured behaviour recorded in `conformance/fixtures/output_redaction.json`,
  all of which require `_secret_*` prefixed keys to be masked regardless of schema markings. An
  implementer following the tech design would have returned `_secret_api_key` in plaintext for every
  module without an output schema — the common case, and a direct NFR-SEC-004 violation. Both
  statements corrected, and the two distinct cases are now spelled out separately: an empty `{}`
  schema is still passed to `redact_sensitive()` (prefix rule applies), whereas a tool with no
  registered schema at all skips redaction entirely.
- **`ExecutionRouter.handle_call` return type.** SRS §3.3, tech design §6.3 and test plan §5.3 all
  specified `CallToolResult`; all three SDKs return the tuple `(content, is_error, trace_id)` and
  the MCP SDK server layer materialises the `CallToolResult` a layer above. TC-EXEC-001 as written
  could not pass against any SDK. The SRS glossary now names both levels ("call-result tuple",
  "error response dict"), every signature-level assertion was corrected, and the test plan carries
  an explicit unpacking note.
- **`ErrorMapper.to_mcp_error` / `AnnotationMapper.to_mcp_annotations` return types.** Fourteen
  FR-ERROR requirements and five FR-ANNOT requirements specified MCP SDK objects
  (`CallToolResult`, `ToolAnnotations`); both return plain camelCase dicts. The SRS also carried no
  `errorType` field at all in its error model. Corrected, including the canonical
  `GENERAL_INTERNAL_ERROR` code and the fact that **the AI-guidance keys are camelCase on the
  wire** (`retryable` / `aiGuidance` / `userFixable` / `suggestion`) even though the apcore-side
  source attributes are snake_case.
- **MCP tool names are not sanitized.** Tech design ADR-03 stated MCP tool names come from
  `display.mcp.alias` "pre-sanitized by `DisplayResolver`" with `module_id` as a fallback
  "auto-sanitized: dots → underscores". No `DisplayResolver` exists in any SDK, no sanitization is
  applied, and the `module_id` is exposed verbatim — `image.resize` is the MCP tool name. ADR-03's
  `.` → `-` normalization is **OpenAI-only** and remains correct; its scope is now stated.
- **The display overlay was documented against a key no SDK reads.**
  `features/mcp-server-factory.md`'s Display Precedence named
  `descriptor.annotations.extra["display_overlay"]`. The real path is `descriptor.display["mcp"]`
  with `descriptor.metadata["display"]["mcp"]` as a compatibility fallback, carrying `alias`,
  `description` and `guidance`. Rewritten, including the `"\n\nGuidance: {text}"` append and the
  precedence of an operator-typed `description` over `rich_description` rendering.
- **`serve()` / `async_serve()` had three mutually exclusive signatures of record** — SRS §7.8
  listed 13 parameters, SRS §8.1.1 listed 21, tech design §7.1 listed 30. The real surface is 37
  (`serve`) and 32 (`async_serve`). SRS §8.1.1 is now the complete, authoritative signature;
  tech design §7.1 matches it; SRS §7.8 is explicitly labelled a partial constraints table;
  `features/mcp-server-factory.md` groups the same set by concern. Also corrected along the way:
  `exempt_paths` is a **set**, not a list, and defaults to `{"/health", "/metrics"}` rather than
  empty; the parameter is `middleware` (singular), never `middlewares`; and `metrics_collector`
  accepts a bool.
- **`AsyncTaskBridge.is_async(descriptor)`** (tech design §6.15) exists in no SDK and in no other
  document — the method is `is_async_module`.
- **`PreflightResult` is not a bridge return type.** Four documents specified
  `ExecutionRouter.validate_tool(...) -> PreflightResult`. `PreflightResult` is an upstream apcore
  type; all three bridges project it to a plain `{valid, checks, requires_approval}` dict and none
  re-exports it. Also recorded: Python's `validate_tool` is synchronous while TypeScript's and
  Rust's are async, and all three report `valid: true` for an executor with no `validate()` method.
- **The standalone output redactor was never built.** Tech design §6.16 specified
  `src/apcore_mcp/server/redactor.py` exposing `redact_tool_output(output, output_schema)`, and
  five test-plan steps called it. Redaction is a private `ExecutionRouter` method
  (`_maybe_redact` / `_maybeRedact` / `redact_output`) that resolves the schema from the router's
  own map. Both documents now describe the shipped shape.
- **Version floors were stale in both directions.** Tech design's `pyproject.toml` sample declared
  `requires-python = ">=3.10"` (real: `>=3.11`, matching the SRS, the PRD and the README) and
  `apcore>=0.17.1,<1.0`; SRS NFR-COMPAT-002 declared `apcore >= 0.19.0`. All three SDKs pin
  **0.27.0** — `apcore>=0.27.0`, `apcore-js>=0.27.0`, `apcore = "0.27"` — alongside
  `apcore-toolkit>=0.10.0` and `mcp-embedded-ui>=0.4.0`. Also recorded: apcore-toolkit is an
  optional `[markdown]` extra in Python but a hard dependency in TypeScript and Rust, which is
  intentional per-language behaviour and not drift.
- **F-045 and F-046 were named differently in different documents.** The SRS traceability table
  called them "Observability Auto-Wiring" and "Cross-SDK Cancel Dispatcher"; the PRD, the tech
  design, and the SRS's own §10.3 body call them "Decorator Metadata Mapping" and "Custom
  Middleware Injection". The traceability table was the outlier and now matches.
- **The test plan's own coverage-gap note was wrong about four features.** F-037, F-039, F-040 and
  F-041 were listed as pending while §5.11, §5.13, §5.14 and §5.15 fully define their test cases.
  The note now lists only the six features that genuinely have no cases (F-033, F-034, F-035,
  F-044, F-045, F-046).
- **Feature specs that had fallen behind fixes already made in code:**
  `schema-converter.md` still documented TypeScript's converter-level `strict` default as `false`
  (it is `true` since the [SC-11] alignment); `execution-router.md` still documented Rust's
  `redact_output` as defaulting to `false` (it is `true` in all three, and a `false` default would
  have made Rust the one bridge leaking `x-sensitive` output); `error-mapper.md` still described
  `userFixable` as "TypeScript only, hardcoded" (all three now read it off the error object).
- **Broken cross-references.** Two links and two SRS table-of-contents anchors did not resolve;
  `mkdocs build --strict` now passes clean.

### Changed — status corrections

- **Extension Bridge (F-042 / EB-2) is documented as unimplemented in all three SDKs.**
  `features/extension-bridge.md` claimed adapter-hook injection was "functional in
  Python+TypeScript", and `features/mcp-server-factory.md` documented the three hooks as accepted
  `serve()` keyword arguments. Neither is true and neither ever was: Python's `serve()` takes 37
  keyword parameters and `MCPServerFactory.__init__` takes exactly `strict` and `rich_description`,
  with none of `extensions` / `schema_converter` / `annotation_mapper` / `error_mapper` in either;
  TypeScript rejects the same options at compile time (`tsc` TS2353) and silently drops them for a
  JavaScript caller. There is **no supported way to replace the built-in SchemaConverter,
  AnnotationMapper or ErrorMapper** on any entry point. Consequently test plan §9f's TC-EXTMGR-002
  through TC-EXTMGR-006 are unwritable and are marked DEFERRED; only TC-EXTMGR-001, the
  default-adapter case, remains active. EB-1 (`ExtensionManager.apply()`) stays withdrawn.
- **The Registry Listener's Client Notification Bridge is documented as unimplemented.** No SDK
  contains a call site that sends `notifications/tools/list_changed` — emission, the 100 ms
  debounce window, per-session ACL filtering and HTTP fan-out are all design targets. All three
  nonetheless advertise `tools: { listChanged: true }` in the initialize response, so a client that
  trusts the capability waits for a refresh signal that never arrives. The listener's internal half
  does work: the register/unregister subscription rebuilds the active tool collection, so
  `tools/list` returns the correct set when the client asks again. Treat dynamic registration as
  poll-only until the bridge lands.
- **`async.*` were never Config Bus keys.** `features/async-task-bridge.md` presented five
  `async.*` settings as configuration. They are `serve()` parameters — `async_max_concurrent`,
  `async_max_tasks`, `async_tasks` — and two of the five documented keys exist nowhere:
  `async.cleanup_interval_s` is read by no SDK, and the async-hint paths are hardcoded
  (`metadata.async` truthy, or `annotations.extra["mcp_async"] == "true"`) rather than configurable
  via `async.hint_keys`.
- **Markdown helper reachability.** `features/markdown.md` now records what a package consumer can
  actually reach: TypeScript's `isMarkdownAvailable` / `primeMarkdownToolkit` /
  `renderModuleMarkdownSync` are exported from `src/markdown.ts` but not re-exported by `index.ts`,
  and `package.json` declares `exports` as `"."` only, so none of them are importable from the
  published package; Python has no `prime_markdown_toolkit` symbol at all; and Rust's
  `is_available()` returns a constant `true` because the toolkit is a compile-time dependency
  there, so the spec's "toolkit unavailable" row cannot arise. Rust's
  `render_module_markdown(descriptor, display)` also takes a second argument the spec omitted.

### Added

- **Approval Phase B has user-facing documentation for the first time.** The feature shipped in
  0.16.0 and 0.17.0 made it reachable from `serve()`, but `approval_store` / `approval_notify` /
  `InMemoryApprovalStore` / `__apcore_approval_check` appeared in **zero** user-facing pages across
  the whole ecosystem — a reader going README → getting-started → configuration never learned it
  existed. `docs/configuration.md` now splits Approval into Phase A (inline elicitation) and
  Phase B (storage-backed), with all three language tabs for each, the four-step agent polling flow,
  and an explicit warning that `InMemoryApprovalStore` is process-local and non-durable. The notify
  callback is documented as **awaited** in Python and returning `Promise<void>` in TypeScript, which
  the previous examples would have got wrong.
- **Config Bus reference** (`docs/configuration.md`) — the `mcp` namespace was documented nowhere.
  All twelve keys, their defaults, the `APCORE_MCP_*` environment mapping, and the caller-wins
  precedence rule. `output_format` is flagged as **Rust-only**: Python and TypeScript do not read it
  from the Config Bus, so the same `apcore.yaml` produces CSV output from Rust and JSON from the
  other two.
- **`conformance/README.md`** — the five fixtures are the cross-language behavioural contract but
  had no index, no format documentation and no entry point for a new SDK author. Now documents what
  each fixture pins, the file format, the meaning of `known_gaps`, the shared
  `APCORE_CONFORMANCE_FIXTURES` override, and each SDK's loader — including that Rust's assertions
  are inline `#[cfg(test)]` tests inside `src/`, so a coverage audit that only inspects `tests/`
  will wrongly conclude Rust skips a fixture. Linked from `examples-spec.md`.
- **Rust and TypeScript example layouts in `docs/examples-spec.md`.** Cargo requires each example
  to be its own target, so the canonical tree cannot be reproduced literally in Rust; the equivalent
  layout is now specified as conformant rather than left looking like a deviation. The TypeScript
  note records that `binding_demo/` has no `extensions/` directory because it uses the `module()`
  factory — `binding.yaml` files are Python-only.
- **Cross-language `strict` divergence is documented at every place it bites.** Written exactly as
  the Quick Start shows, the three `to_openai_tools` tabs produce different OpenAI schemas from the
  same registry: Python defaults `strict=False` and emits permissive schemas, TypeScript defaults to
  `true`, and the documented Rust call passes `true`. Now called out in the README, in
  `docs/configuration.md`, and in SRS §7.9, with the advice to pass `strict` explicitly in portable
  code.
- **The Rust directory-discovery limitation is now visible where users hit it.** CHANGELOG 0.17.0
  recorded that `BackendSource::ExtensionsDir` builds an empty registry, but the README Quick Start
  (Rust tab, CLI tab, and the Auto-discovery feature bullet), `docs/index.md`, and
  `docs/getting-started.md` §2 and §4 all still presented the zero-code path as working for Rust —
  a first run yields a server with no user tools. Every one of those now carries the caveat, and the
  README's Rust example was rewritten to the `Registry` → `Executor` → `.backend(...)` pattern that
  actually works.
- `docs/examples-spec.md` — the normative cross-language examples standard, carrying the compliance
  checklist every new `apcore-mcp-{lang}` implementation must satisfy — was **not in the MkDocs
  nav** and therefore not published to the docs site, and not included in the generated
  `llms-full.txt`. Added to the Specifications section.
- `llms.txt` — Features listed 9 of the 16 feature specs; the missing seven (Error Mapper, Extension
  Bridge, Registry Listener, Approval Handler, Approval Handler Phase B, Async Task Bridge,
  Markdown) are added, along with the Test Plan and Examples Standard under Specifications. Two
  claims that apcore-mcp "can aggregate upstream MCP servers as a gateway" were removed: no such
  feature exists in any spec, PRD feature, or SDK source, and `llms.txt` exists specifically to feed
  machine consumers.
- `docs/index.md` now carries a banner recording that the deploy workflow replaces it with the
  repo-root README, so the published Home page is the README and edits to `index.md` never reach the
  site.

### Known limitations (Rust)

Both carried forward from 0.17.0 and re-verified against 0.17.2:

- `strategy` overrides apply on the trace execution path only; the non-trace `call()` path still
  runs the executor's own strategy.
- `BackendSource::ExtensionsDir` cannot honor runtime directory discovery through apcore's public
  API and builds an empty registry with a warning; `BackendSource::Registry` is fully supported.
  This limitation is now surfaced in the README, `docs/index.md` and `docs/getting-started.md`
  rather than only here.


## [0.17.0] - 2026-06-23

Cross-SDK maintenance release resolving an extended audit of the serve/embed
entry points and the Phase B approval chain, and lifting all three SDKs onto
apcore 0.25 / apcore-toolkit 0.9.1. Released as `apcore-mcp-{python,rust,typescript}` v0.17.0.

### Fixed — cross-SDK contract

- **`serve()` / `async_serve()` now expose the Phase B approval inputs**
  (`approval_store` / `approval_notify` in Python/Rust, `approvalStore` /
  `approvalNotify` in TypeScript) in all three SDKs. Previously these were only
  reachable by constructing `APCoreMCP` directly, so the documented Phase B flow
  could not be driven from the top-level entry point.
- **Rust Phase B approval is now actually wired end-to-end.** In 0.16.x the Rust
  `StorageBackedApprovalHandler` was stored but never dispatched, the
  `__apcore_approval_check` meta-tool was never advertised, and the TTL sweep
  never ran — so the Phase B contract in `docs/features/approval-phase-b.md` was
  effectively unimplemented on Rust despite the A-002 builder methods existing.
  It is now live: bridge dispatch, meta-tool advertisement, and sweep all run.
- **Framework-integration server parity.** The non-blocking embeddable server
  (`MCPServer` in Python/Rust) now exposes the same capability surface as the
  blocking `serve()` — output formatter, strategy, observability, output
  redaction, trace, explorer branding, approval, middleware/ACL, and dynamic
  registration. The Rust `MCPServer::start()` and `async_serve()` paths now
  actually register handlers and serve `/mcp` (previously `start()` only awaited
  shutdown and `async_serve()` returned a Router with no `/mcp` route).

### Changed

- All three SDKs raised their dependency floors to **apcore 0.25** /
  **apcore-toolkit 0.9.1** (drop-in; no consumed API changed). The Rust upgrade
  was unblocked by apcore-toolkit 0.9.1 lifting its own `apcore` floor to 0.25.

### Known limitations (Rust)

- `strategy` overrides apply on the trace execution path only; the non-trace
  `call()` path still runs the executor's own strategy.
- `BackendSource::ExtensionsDir` cannot honor runtime directory discovery through
  apcore's public API and builds an empty registry with a warning;
  `BackendSource::Registry` is fully supported.


## [0.16.0] - 2026-06-12

Cross-SDK spec release adding the **Approval Phase B** storage-backed approval handler and a full cross-language sync pass covering all 16 MCP feature modules.

### Added

- **Approval Phase B** (`docs/features/approval-phase-b.md`) — `StorageBackedApprovalHandler` implements long-lived, out-of-band human approval via a persistent `ApprovalStore`. Complements Phase A (`ElicitationApprovalHandler`) for flows where the approver is not present at the terminal (automated pipelines, mobile approvals, team workflows). Key additions across all three SDKs:
  - `StorageBackedApprovalHandler(store, notify_callback?)` — always returns `ApprovalResult(status="pending")` and fires the notify callback fire-and-forget.
  - `ApprovalStore` protocol / trait / interface — `save_pending`, `get_result`, `resolve`.
  - `InMemoryApprovalStore` — default in-process store with separate `resolved_ttl` / `pending_ttl` / `sweep_interval` / `max_records` knobs.
  - `__apcore_approval_check` meta-tool — sixth reserved meta-tool alongside the five `__apcore_task_*` / preview tools. Lets the AI agent poll for out-of-band approval without a full module call.
  - `APCoreMCP` gains `approval_store` / `approvalStore` constructor parameter (Python, TypeScript) that automatically wraps the store in a `StorageBackedApprovalHandler` and registers `__apcore_approval_check`. See `docs/features/approval-phase-b.md`.

### Cross-language sync — full-scope mcp round (2026-06-12)

Full pass across all 16 feature-spec modules against `apcore-mcp-{python,rust,typescript}` v0.16.0. See `sync-report-apcore-mcp-2026-06-12.md` for the full checklist.

**Critical findings (implementation gaps against spec):**

- **A-001 (Rust) — OC-5 rename not applied**: `convert_registry` in `OpenAIConverter` still takes `&serde_json::Value` (old JSON-snapshot path). The `&Registry` canonical variant was added as `convert_registry_apcore` — wrong name. CHANGELOG 0.14.0 migration said `convert_registry` → Registry and `convert_registry_json` → old snapshot; that rename was not executed. Rust callers following the migration guide call `convert_registry(&registry)` and get a type error. Fix: rename `convert_registry_apcore` → `convert_registry`; rename old `convert_registry` → `convert_registry_json`.
- **A-002 (Rust) — `APCoreMCPBuilder` missing `approval_store()` / `approval_notify()`**: `StorageBackedApprovalHandler`, `InMemoryApprovalStore`, and `ApprovalStore` are all exported from `apcore-mcp-rust`, but the builder wires only `approval_handler(Arc<dyn ApprovalHandler>)`. Python/TS accept `approval_store=` / `approvalStore=` directly. The usage example in `approval-phase-b.md` for Rust does not compile. Fix: add `approval_store(Arc<dyn ApprovalStore>)` and `approval_notify(callback)` builder methods.
- **A-003 (Python) — `McpErrorFormatter` alias never exported (EM-1 regression)**: CHANGELOG 0.14.0 EM-1 documents "both names exported from `apcore_mcp` and `apcore_mcp.adapters`" as shipped. The class in `formatter.py` is named `MCPErrorFormatter` only; no `McpErrorFormatter = MCPErrorFormatter` alias exists. `from apcore_mcp import McpErrorFormatter` raises `ImportError` in Python at v0.16.0. Fix: add alias in `formatter.py` and both `__init__.py` export lists.

**Warning findings:**

- **A-004 (Rust) — `RegistryListener.start()` takes `(registry, factory)`**: spec contract says "No parameters". Python/TS take the registry and factory in the constructor then call `start()` with no args; Rust has `new()` with no args and `start(registry, factory)`. Cross-language documentation and usage examples are misaligned.
- **A-005 (Rust) — `AsyncTaskBridge` constructor takes `executor`, not `manager`**: spec `__init__` contract lists `manager: AsyncTaskManager` as the required input. Rust `new(executor)` constructs the manager internally. Documenting this as a Rust-specific structural idiom is tracked; spec contract table needs updating.
- **A-006 (Rust) — `is_async_module` decomposed signature**: spec says `is_async_module(descriptor)` (one duck-typed param); Rust exposes `is_async_module(metadata, annotations_extra)` (two params). `is_async_module_descriptor(descriptor)` also exists as a one-param alternative. Spec should document the two forms or consolidate.

**Doc consistency (Phase B):**

- **B-001**: `approval-phase-b.md` not listed in `mkdocs.yml` navigation — unreachable from the published docs site.
- **B-002**: `approval-phase-b.md` claims Rust builder supports `approval_store` (snake_case) — it doesn't pending A-002.
- **B-003**: `overview.md` feature table does not list Approval Phase B or `__apcore_approval_check`.
- **B-004**: CHANGELOG 0.14.0 EM-1 entry claims Python `McpErrorFormatter` is exported — this is inaccurate; see A-003.

#### Deferred to a future release

- **A-001 fix (Rust)** — `convert_registry` rename; `convert_registry_json` preservation.
- **A-002 fix (Rust)** — `APCoreMCPBuilder::approval_store` / `approval_notify` builder methods.
- **A-003 fix (Python)** — `McpErrorFormatter` alias and `__all__` update.
- **B-001 fix** — add `approval-phase-b.md` to `mkdocs.yml` nav.

---

## [0.15.0] - 2026-05-14

Cross-SDK release leveraging **apcore 0.21.0 + apcore-toolkit 0.6.0**.
Promotes three new upstream capabilities into MCP-facing surface area
across all three SDKs (`apcore-mcp-python`, `apcore-mcp-rust`,
`apcore-mcp-typescript`). Byte-equivalent meta-tool, error envelope,
and Markdown rendering across languages.

### Changed
- Spec: `AsyncTaskBridge.submit` — document Rust's bridge-managed progress channel (`progress_senders[task_id]` registered after `manager.submit`) as a language-specific variation of the Python/TypeScript in-context (`context.data[MCP_PROGRESS_KEY]`) contract. Wire-level `notifications/progress` events are unchanged across SDKs (D10-001).
- Spec: `ElicitationApprovalHandler` — document Rust's constructor-injected `Option<Arc<dyn ElicitCallback>>` contract as a language-specific variation of the Python/TypeScript per-call `request.context.data[MCP_ELICIT_KEY]` lookup. Rejection semantics are preserved across SDKs (D10-002).
- Spec: `JWTAuthenticator.authenticate` — `type` claim normalization: empty string `""` is treated as a missing claim and falls back to the default `"user"`, aligning Python's truthy-fallback semantics with TypeScript and Rust (D11-107).
- Spec: Approval handler — document panic-guard cross-language asymmetry: Python `try/except Exception` (BaseException propagates), TypeScript `try/catch` (no panic distinction), Rust `futures::FutureExt::catch_unwind` (catches `panic!()` across `.await`). Treat panic-safety as Rust-only when porting modules (D11-108).
- **Dependency bumps**: `apcore >= 0.21.0` (Python/Rust), `apcore-js >= 0.21.1` (TS), `apcore-toolkit >= 0.6.0` (Python/Rust) / `>= 0.6.1` (TS).
- **Rust BREAKING**: `AsyncTaskBridge::{submit, cancel, cancel_session_tasks, handle_meta_tool, shutdown}` are now `async fn` — propagates upstream apcore 0.20+ async signatures (D10-003 / D10-004). Sync transport-layer cancel handlers now `tokio::spawn` the cancel call as fire-and-forget.

### Added

- **`__apcore_module_preview` meta-tool (PROTOCOL_SPEC §5.6 / §12.8)** — fifth reserved meta-tool alongside the four `__apcore_task_*` ones. Drives `executor.validate(module_id, inputs, context)` and returns a structured `{valid, requires_approval, predicted_changes, checks}` envelope WITHOUT executing the module. Lets AI orchestrators answer *"what would change in the world if I called this?"* before invoking destructive or stateful modules. `arguments: null` and missing `arguments` are preserved verbatim — the calling business decides whether null is a valid input. Structurally-impossible shapes (arrays, scalars) return a typed validation error. Cross-SDK wire-equivalent across Python, Rust, and TypeScript.
- **`rich_description` / `richDescription` option** on `MCPServerFactory` (Python, Rust, TS) and `OpenAIConverter` (Python, Rust, TS) — renders `Tool.description` / OpenAI `function.description` as canonical apcore-toolkit Markdown (`format_module(style="markdown")`) instead of the plain one-line description. Includes title, description, parameters list, returns list, behavior table (only fields differing from defaults — toolkit 0.6 alignment), tags, and examples. **LLMs select tools primarily from this string; Markdown packs more decision-relevant signal per token.** Display-overlay `mcp.description` overrides still win first. Falls back to the plain description with a one-shot WARN log when `apcore-toolkit` is unavailable (Python optional `[markdown]` extra; TS optional peer dep; Rust mandatory).
- **`markdown` module** in all 3 SDKs — public helpers for descriptor → ScannedModule adapter and direct Markdown rendering. TS adds a `primeMarkdownToolkit()` async loader so synchronous code paths can render after startup priming.
- **`CIRCUIT_BREAKER_OPEN` error mapping** (apcore 0.20 sync alignment A-001) — `ErrorMapper` now dispatches `CircuitBreakerOpenError` (Python) / `ApcoreErrorCode::CircuitBreakerOpen` (Rust) / `ErrorCodes.CIRCUIT_BREAKER_OPEN` (TS) to a `retryable: true` envelope with `aiGuidance` mirrored from the apcore error class. Added to constants tables in all 3 SDKs.
- **Rust-specific**: new `ConvertOptions` struct + `OpenAIConverter::*_with_options` method family (backwards-compatible additive API); `markdown` module re-exported from `lib.rs`; new public `json_entry_to_scanned_module(module_id, &Value) -> ScannedModule` adapter that lets the duck-typed JSON registry path drive the same Markdown rendering as the `&Registry` path.

### Fixed

- **Rust `ApprovalResult` / `ApprovalRequest`** — adapted to apcore 0.21's `#[non_exhaustive]` annotations via the `let mut x = X::default(); x.field = ...; x` construction pattern.
- **Rust binding-error match** — removed deleted `ApcoreErrorCode::BindingPolicyViolation` variant (apcore 0.21 dropped it; the wire string remains supported via the constants table for backward-compat with legacy emitters).
- **Cross-SDK `arguments: null` preservation** — preview handler now forwards null/missing `arguments` verbatim to `executor.validate()` instead of silently coercing to `{}`. Lets the calling business decide whether null is a valid module input.

### Tests

- Python: **771 passed** (was 758, +13 net)
- Rust: **843 passed** (was 828, +15 net)
- TypeScript: **549 passed** (was 534, +15 net)

## [0.14.0] - 2026-05-01

### Added

- **2 new features (F-042, F-043)** leveraging apcore 0.19.0 + apcore-toolkit 0.5.0:
  - **Extension Bridge** (F-042, P1) — formalizes `ExtensionManager → MCPServerFactory` wiring; centralizes resolution precedence (kwarg > ExtensionManager > built-in default) for `schema_converter`, `annotation_mapper`, `error_mapper`, and the load-order policy (extensions first, built-in middleware second). See `docs/features/extension-bridge.md`.
  - **Async Task Bridge** (F-043, P1) — routes async-hinted modules (`metadata.async == true` or `annotations.extra.mcp_async == "true"`) to apcore's `AsyncTaskManager.submit()` via four reserved meta-tools (`__apcore_task_submit`, `__apcore_task_status`, `__apcore_task_cancel`, `__apcore_task_list`). Progress fan-out via `_meta.progressToken`. See `docs/features/async-task-bridge.md`.
- **W3C Trace Context propagation** — inbound `_meta.traceparent` is parsed into the apcore `Context.trace_parent`; outbound responses carry a freshly-minted `_meta.traceparent` so MCP clients can correlate trace chains across module boundaries (Python only at v0.14.0; TypeScript and Rust parse inbound but do not yet inject outbound — tracked in cross-language sync report).
- **Observability auto-wiring** — `serve(observability=True)` / `APCoreMCP(observability=True)` instantiate `MetricsCollector` + `MetricsMiddleware` and `UsageCollector` + `UsageMiddleware` on the Executor and expose `/{explorer_prefix}/api/usage` (and `/api/usage/{module_id}`) endpoints. CLI flag `--observability` toggles the same.
- **isinstance-based error dispatch** — `ErrorMapper` dispatches `TaskLimitExceededError`, `DependencyNotFoundError`, and `DependencyVersionMismatchError` via class checks against apcore 0.19 error classes (no longer duck-typed by code).
- **Expanded `ModuleAnnotations` surfacing** — `cache_ttl`, `cache_key_fields`, and `pagination_style` now appear in the description annotation block when non-default; aligns with apcore 0.19's 12-field `ModuleAnnotations`.
- **4 new error code mappings** — `DEPENDENCY_NOT_FOUND`, `DEPENDENCY_VERSION_MISMATCH`, `TASK_LIMIT_EXCEEDED` (with `retryable: True`), `VERSION_CONSTRAINT_INVALID`.

### Changed

- **Dependency bump**: requires `apcore >= 0.19.0` (was `>= 0.17.1`) for `AsyncTaskManager`, `TraceContext`, `Context.create(trace_parent=...)`, the 12-field `ModuleAnnotations`, and the new dependency/binding error classes.
- **New optional dependency**: `apcore-toolkit >= 0.5.0` — provides `BindingLoader`/`BindingParser` and `ScannedModule.display` for consumers loading `.binding.yaml` files. Not wired by apcore-mcp itself; available transitively for downstream callers.
- `ExecutionRouter.handle_call` response `content` item type widened from `list[dict[str, str]]` to `list[dict[str, Any]]` to carry the optional `_meta` field. Translated to MCP `TextContent.meta` on the wire.
- `MCPServerFactory.register_handlers` gains optional `async_bridge` and `descriptor_lookup` kwargs (Python). Backward-compatible: when omitted, behavior is unchanged.
- Updated PRD to v1.8 (46 features), SRS to v2.0, Tech Design to v1.9, Test Plan to v1.8.
- Feature count: P0=9, P1=15, P2=22, Total=46.

### Cross-language sync — deferred-modules round (2026-04-28)

Follow-up sweep across the 10 modules deferred from the original 0.14.0 sync. Each item below is a cross-SDK divergence or contract gap closed inside this release; together they reduce the outstanding critical/high backlog from 24 to 1 (EB-1 spec drift, recommended for spec revision).

- **EUI-1 — `/validate` endpoint**: `mcp-embedded-ui` minimum bumped to `0.4.0` across all three SDKs. The new `POST /tools/{name}/validate` route flows automatically through the existing `create_mount` / `createNodeHandler` adapters; schema-only validation, ungated by `allow_execute` / authenticator, never invokes the router's `handle_call`. TC-011 integration tests added in all three SDKs.
- **TM-4 (Python) — transport-disconnect cancellation forwarding**: `TransportManager.set_async_task_bridge(bridge)` matches TS `setAsyncTaskBridge` and Rust `set_cancel_handler`. The transport scopes a per-connection session id via the new `transport_session_var` ContextVar; `factory.handle_call_tool` forwards it as `session_key` to `bridge.submit(...)`, and on transport teardown the manager calls `bridge.cancel_session_tasks(session_id)`. Wired automatically by `serve()`, `async_serve()`, and `APCoreMCP.serve` / `async_serve` when an async bridge is present. 6 regression tests added.
- **EM-1 (Python) — `McpErrorFormatter` canonical class name** with backwards-compatible `MCPErrorFormatter` alias for cross-SDK API parity. Both names are exported from `apcore_mcp` and `apcore_mcp.adapters`.
- **EM-3 (Python+Rust) — hardcoded `userFixable=true`** for dependency / binding / version-constraint error codes (matches TS). Affects `DEPENDENCY_NOT_FOUND`, `DEPENDENCY_VERSION_MISMATCH`, `VERSION_CONSTRAINT_INVALID`, `BINDING_SCHEMA_INFERENCE_FAILED`, `BINDING_SCHEMA_MODE_CONFLICT`, `BINDING_STRICT_SCHEMA_INCOMPATIBLE`, `BINDING_POLICY_VIOLATION`. apcore 0.19's error classes don't yet set `user_fixable=true` themselves, so the bridge stamps the hint to give MCP clients a consistent self-healing signal. 9 Python + 5 Rust regression tests added.
- **EM-6 (Rust) — generic-error fallback**: `ErrorMapper::internal_error_response()` and `ErrorMapper::to_mcp_error_any<E: std::error::Error>()` return the `{is_error:true, error_type:"GENERAL_INTERNAL_ERROR", message:"Internal error occurred", details:null}` envelope for any non-`ModuleError` input — matches Python's `to_mcp_error(error: Exception)` and TypeScript's `toMcpError(error: unknown)`. 3 regression tests added.
- **MID-5 — bijection-guarded denormalize across all 3 SDKs**: new variants `try_denormalize` (Python) / `tryDenormalize` (TS) / `denormalize_checked` (Rust) run the dash→dot replacement and validate the result against `MODULE_ID_PATTERN`, returning `None` / `null` / `Err(InvalidModuleId)` if the input was not produced by `normalize`. Plain `denormalize` stays lenient (backwards compatible). Useful for sanitizing OpenAI tool-call responses against the registered module set. 8 Python + 9 TS + 5 Rust regression tests added.
- **OC-1 (TypeScript) — strict-mode walker parity with Python+Rust**: the TS strict-mode pipeline now mirrors apcore's canonical `to_strict_schema`: promotes `x-llm-description` → `description`, strips all `x-*` extension keys after promotion, recurses into `oneOf` / `anyOf` / `allOf` and `$defs` / `definitions`, sorts property names alphabetically, and removes `default` values. Output now matches Python+Rust (which delegate to apcore directly). 6 regression tests added.
- **AH-1 (Rust) — per-request elicit callback via task-local**: added `tokio::task_local! ELICIT_CALLBACK` in `apcore_mcp::helpers`. `ElicitationApprovalHandler::request_approval` now resolves the callback from the task-local first (matching Python+TS, which read it from `context.data`), with the constructor field as a fallback. apcore-rust's `Context::data` cannot hold boxed `Fn`s, so a task-local is the closest cross-SDK equivalent without changing apcore. 4 regression tests added.
- **EB-2 (Python + TypeScript) — adapter-hook kwargs in `serve()` / `async_serve()`**: pass `schema_converter`, `annotation_mapper`, and `error_mapper` to override the factory's built-in adapters. Useful for extensions that customize JSON-Schema strictness, the annotation wire format, or error formatting. Rust deferred to a future release because its adapters are stateless unit structs and require a trait-based redesign first.
- **JWT-2 (Python) — case-insensitive `Authorization` header lookup.** `JWTAuthenticator.authenticate` now tries both `headers["authorization"]` and `headers["Authorization"]`. ASGI lower-cases header names but direct callers (tests, hooks, manual invocations) may pass the capitalised form; RFC 7230 §3.2 mandates case-insensitive header names. Matches TS+Rust behaviour. 1 regression test.
- **AM-L1 — F-041 annotation extras format unification across all 3 SDKs.** `mcp_*` extras now appear after the `[Annotations: ...]` block as `<stripped-key>: <value>` lines separated by single newlines (matches the wire format that TypeScript was already emitting). Pre-fix Python emitted each extra as its own section separated by `\n\n`; pre-fix Rust inlined them into the `[Annotations: ...]` block as `mcp_key=value`. 1 regression test in each SDK.

#### Breaking changes (cross-SDK API unification)

- **JWT-1 — `Authenticator.authenticate` signature unified.** All three SDKs now use `authenticate(headers: HeaderMap) -> Awaitable<Identity | null>`.
  - **Python**: `authenticate` is now `async`. Existing sync implementations continue to work via the new `apcore_mcp.auth.protocol.call_authenticator(auth, headers)` helper, which inspects the return value and awaits if it's a coroutine. Tests for `JWTAuthenticator` are now `async def`.
  - **TypeScript**: `Authenticator.authenticate` now takes `Record<string, string>` instead of `IncomingMessage`. Use the new `extractHeaders(req)` helper (re-exported from the package root) to flatten a Node `IncomingMessage` before calling.
  - **Rust**: no change (already aligned).
- **OC-5 — Rust `convert_registry` signature.** The canonical Rust entrypoint now takes `&apcore::registry::Registry` directly (matching Python+TS duck-typed Registry input). The pre-fix `&Value`-snapshot variant is preserved as `convert_registry_json` for callers that hold a serialized snapshot.

#### Migration

This release contains several behaviour-changing fixes. None of the runtime changes are silent — each is observable via type errors or test failures during upgrade.

##### Python

- **`Authenticator.authenticate` is now `async`.** Existing sync implementations continue to work via the bridge:
  ```python
  # Before (sync):
  class MyAuth:
      def authenticate(self, headers: dict[str, str]) -> Identity | None: ...

  # After (async — preferred):
  class MyAuth:
      async def authenticate(self, headers: dict[str, str]) -> Identity | None: ...

  # Or keep sync — middleware uses `call_authenticator(auth, headers)` which
  # awaits coroutines and falls through for sync returns.
  ```
  Tests calling `authenticator.authenticate(...)` directly must be `async def` and `await` the call.
- **`AnnotationMapper.to_mcp_annotations` returns camelCase keys** (`readOnlyHint`, `destructiveHint`, …). Previously snake_case (`read_only_hint`). Anyone consuming the dict directly must rename keys.
- **`MCPErrorFormatter` ↔ `McpErrorFormatter`** — the new PascalCase form is preferred (matches TS+Rust). Both names are exported.
- **`JWTAuthenticator` clock-skew leeway** is now 30 s (was 0 s). Tokens that were silently rejected ±30 s of expiry/nbf will now validate. If you tested with `exp = now() - 10 s`, expand to `- 60 s` so the token stays expired.

##### TypeScript

- **`Authenticator.authenticate` takes `Record<string, string>` instead of `IncomingMessage`.** Use the new helper to bridge:
  ```ts
  // Before:
  authenticator.authenticate(req);

  // After:
  import { extractHeaders } from "apcore-mcp";
  authenticator.authenticate(extractHeaders(req));
  ```
- **OpenAI strict mode default is `true`** (was `undefined → false`). Existing callers that relied on permissive output must opt out explicitly: `convertRegistry(registry, { strict: false })`.
- **Schema-converter `_inlineRefs` recursion is capped at 32**. Pathological deeply-recursive schemas now throw instead of stack-overflowing.
- **401 body is unified to `{error: "Unauthorized", detail: "..."}`.** Pre-fix TS returned `{error: "Authentication required"}`.

##### Rust

- **`OpenAIConverter::convert_registry` now takes `&apcore::registry::Registry`** (was `&serde_json::Value`). Callers that hold a serialized snapshot should switch to `convert_registry_json`:
  ```rust
  // Live registry path (preferred, matches Python+TS):
  converter.convert_registry(&registry, false, false, None, None)?;

  // Or keep using a JSON snapshot:
  converter.convert_registry_json(&value, false, false, None, None)?;
  ```
- **`register_mcp_formatter`** is the canonical function name (was briefly `register_mcp_error_formatter` during 0.14 dev). The release ships only the unified name.
- **Strict-schema walker preserves `type: ["object", "null"]`** (no longer downgrades to bare `"object"`) and stops descending into `enum` / `const` / `examples` / `default`. Schemas that exploited the over-aggressive descent will see less inserted `additionalProperties: false`.

#### Deferred to a future release

- **A-D-012 (Rust) — canonical strict-schema sourcing via `Registry::export_schema_strict`**: requires a future apcore release (currently committed locally as `62706be` but not yet on crates.io). 0.14.0 ships with the local-`SchemaConverter` fallback as the canonical path; behaviour is identical, the upgrade is purely about delegating to apcore upstream.
- **EB-1 — `ExtensionManager.apply()`**: spec drift, not a cross-SDK divergence — recommended for spec revision since none of the three SDKs need the `apply()` step (extensions load via Config Bus / kwargs).
- **EB-2 (Rust) — adapter-hook injection**: blocked on stateless unit structs; needs adapter trait redesign.
- **TM-1/2/3 (Python TransportManager setter API parity)**: cosmetic, the underlying functionality (auth, explorer, /usage) already works through the wrapper layer.

---

## [0.13.0] - 2026-04-06

### Added

- **6 new features (F-036 through F-041)** fully leveraging apcore 0.17.1:
  - **Pipeline Strategy Selection** (F-036, P1) — `serve(strategy="standard"|"internal"|"testing"|"performance"|"minimal")` parameter, CLI `--strategy` flag, and Config Bus `mcp.pipeline.strategy`.
  - **Pipeline Observability** (F-037, P2) — `serve(trace=True)` enables `call_async_with_trace()`; `PipelineTrace` data in `_meta.trace` response, Explorer, and MetricsCollector.
  - **Tool Output Redaction** (F-038, P1) — `redact_sensitive(output, output_schema)` applied by default before serialization. Fields with `x-sensitive` or `_secret_*` keys replaced with `"***REDACTED***"`.
  - **Tool Preflight Validation** (F-039, P2) — `ExecutionRouter.validate_tool()` and Explorer `POST /validate` endpoint for dry-run via `Executor.validate()`.
  - **YAML Pipeline Configuration** (F-040, P2) — Config Bus `mcp.pipeline` section for declarative pipeline customization via `build_strategy_from_config()`.
  - **Annotation Metadata Passthrough** (F-041, P2) — `ModuleAnnotations.extra` keys prefixed with `mcp_` flow to tool descriptions and Explorer.
- **4 new error mappings** — `ConfigEnvMapConflictError`, `PipelineAbortError`, `StepNotFoundError`, `VersionIncompatibleError`.
- **RegistryListener wired to `serve(dynamic=True)`** — dynamic tool registration now operational.

### Changed

- **Dependency bump**: requires `apcore >= 0.17.1` (was `>= 0.15.1`) for Pipeline v2 delegation, step metadata, YAML pipeline configuration, `build_minimal_strategy()`, `requires`/`provides` on BaseStep, and sensitive field redaction.
- **Pipeline v2 alignment** — 11-step pipeline with `call_chain_guard` (renamed from `safety_check`), middleware before input validation.
- Updated PRD to v1.7 (41 features), SRS to v1.8 (127 requirements), Tech Design to v1.7, Test Plan to v1.6.
- Feature count: P0=9, P1=11, P2=21, Total=41.

---

## [0.12.0] - 2026-03-31

### Added

- **Config Bus namespace registration** (F-033) — apcore-mcp registers an `mcp` namespace with the apcore Config Bus (`Config.register_namespace("mcp", ...)`) using `APCORE_MCP` as the env prefix. MCP-specific configuration (transport, host, port, auth, explorer) can now be managed in a unified `apcore.yaml` file. The adapter reads logging defaults from the `observability` namespace.
- **Error Formatter Registry integration** (F-034) — apcore-mcp registers an MCP-specific `ErrorFormatter` with apcore's Error Formatter Registry (§8.8), formalizing camelCase wire keys and MCP error code sanitization into the shared protocol.
- **Dot-namespaced event type constants** (F-035) — Added `APCORE_EVENTS` constants (TypeScript) with canonical event type names from apcore 0.15.0 (`apcore.module.toggled`, `apcore.config.updated`, `apcore.module.reloaded`, `apcore.health.recovered`). No existing event subscriptions were changed — the `RegistryListener` uses callback-based `registry.on("register")` which is unaffected.
- **6 new error code mappings** — `CONFIG_NAMESPACE_DUPLICATE`, `CONFIG_NAMESPACE_RESERVED`, `CONFIG_ENV_PREFIX_CONFLICT`, `CONFIG_MOUNT_ERROR`, `CONFIG_BIND_ERROR`, `ERROR_FORMATTER_DUPLICATE` added to ErrorMapper.

### Changed

- Dependency bump: requires `apcore >= 0.15.1` (was `>= 0.13.0`) for Config Bus (§9.4), Error Formatter Registry (§8.8), simplified env prefix convention, and dot-namespaced event types (§9.16).
- Updated PRD to v1.5 (35 features, was 32), SRS to v1.6, Tech Design to v1.5.
- Feature count: P2 increased from 15 to 18 (added F-033, F-034, F-035).

---

## [0.11.0] - 2026-03-23

### Added

- **Display overlay in `build_tool()`** (§5.13) — MCP tool name, description, and guidance now sourced from `metadata["display"]["mcp"]` when present.
  - Tool name: `metadata["display"]["mcp"]["alias"]` (pre-sanitized by `DisplayResolver`, already `[a-zA-Z_][a-zA-Z0-9_-]*` and ≤ 64 chars).
  - Tool description: `metadata["display"]["mcp"]["description"]`, with `guidance` appended as `\n\nGuidance: <text>` when set.
  - Falls back to raw `descriptor.module_id` / `descriptor.description` when no display overlay is present.
- Updated `tech-design-apcore-mcp.md` — `build_tool()` mapping updated to reflect display overlay source for tool name and description.

### Changed

- Dependency bump: requires `apcore-toolkit >= 0.4.0` for `DisplayResolver`.

### Tests

- `TestBuildToolDisplayOverlay` (6 tests): MCP alias used as tool name, MCP description used, guidance appended to description, surface-specific override wins over default, fallback to scanner values when no overlay.

---

## [0.10.1] - 2026-03-22

### Changed
- Rebrand: aipartnerup → aiperceivable

## [0.10.0] - 2026-03-14

### Added
- `async_serve()` / `asyncServe()` function specification for embedding MCP in larger ASGI/HTTP applications
- `ExecutionCancelledError` mapping with `retryable=True` AI guidance
- `output_formatter` parameter for customizable tool output (e.g., Markdown via apcore-toolkit)
- Deep merge streaming accumulation with depth limit (32) for chunked responses
- New annotation fields in ModuleAnnotations: `streaming`, `cacheable`, `paginated`, `cache_ttl`, `cache_key_fields`, `pagination_style`
- FR-SERVER-012: async_serve() embeddable ASGI application
- FR-ERROR-012: ExecutionCancelledError mapping

### Changed
- Updated Python and TypeScript implementation versions to v0.10.0
- Default `output_formatter` changed to `None` (raw JSON) in both implementations
- Dependency bump to `apcore>=0.13.0` / `apcore-js>=0.13.0` for new annotation fields (`cacheable`, `paginated`)
- Annotation description suffix now includes `cacheable` and `paginated` when set
- Updated SRS to v1.5 with full `serve()` signature including all parameters added since v0.5.0 (explorer, auth, approval, metrics, output_formatter)
- Bumped minimum Python version from >= 3.10 to >= 3.11 (aligns with apcore-python pyproject.toml)
- Bumped minimum apcore dependency from >= 0.2.0 to >= 0.13.0 (required for new annotation fields and ExecutionCancelledError)
- Requirement count increased from 104 to 106 (86 FRs + 20 NFRs)

### Removed
- `apcore-toolkit` is no longer a required dependency in the Python implementation

## [0.9.0] - 2026-03-06

### Changed
- Replaced the anemone logo in `apcore-mcp-logo.svg` with a new jellyfish mascot; moved the original anemone design to `apcore-mcp-image.svg`

## [0.8.0] - 2026-03-02

### Added
- Approval system feature specification (F-028): runtime approval via `ElicitationApprovalHandler`, `--approval` CLI flag with modes `elicit`, `auto-approve`, `always-deny`, `off`
- Approval error codes: `APPROVAL_DENIED`, `APPROVAL_TIMEOUT`, `APPROVAL_PENDING`
- Enhanced error responses with AI guidance fields (`retryable`, `ai_guidance`/`aiGuidance`, `user_fixable`/`userFixable`, `suggestion`)
- AI intent metadata in tool descriptions (`x-when-to-use`, `x-when-not-to-use`, `x-common-mistakes`, `x-workflow-hints`)
- Streaming annotation in description suffixes
- Comprehensive developer documentation site using MkDocs
- Unified "Getting Started" guide and "Configuration Reference" for Python and TypeScript

### Changed
- Cross-language naming convention documented: Python uses snake_case (`ai_guidance`), TypeScript uses camelCase (`aiGuidance`) for AI guidance fields

## [0.7.0] - 2026-02-28

### Added
- JWT Authentication feature specification (F-027) — key file support, configurable strictness, path exemptions, audit logging
- `Authenticator` protocol for pluggable authentication backends
- `JWTAuthenticator` with `ClaimMapping` for JWT Bearer token validation
- `AuthMiddleware` ASGI middleware with `ContextVar` bridge for Identity injection
- CLI flags for JWT: `--jwt-secret`, `--jwt-algorithm`, `--jwt-audience`, `--jwt-issuer`

### Changed
- Updated security threat model with JWT-specific entries
- Updated `serve()` API with `authenticator` parameter

## [0.6.0] - 2026-02-25

### Added
- Streaming bridge specification — progress notifications, chunk accumulation, fallback to non-streaming
- Elicitation support specification — user input requests during module execution
- Dynamic tool registration specification

## [0.5.1] - 2026-02-25

### Changed
- Renamed "Inspector" to "Explorer" across the entire specification and documentation (PRD, SRS, design, and test plans).

## [0.5.0] - 2026-02-24

### Added
- MCP Tool Explorer specification (F-026) — browser-based UI for inspecting and testing tools
- Examples specification — cross-language standard for demo modules

## [0.4.0] - 2026-02-23

### Added
- Resource handlers specification for serving documentation via MCP
- Prometheus metrics specification (`/metrics` endpoint)
- CI/CD workflow specification for GitHub Actions

## [0.3.0] - 2026-02-22

### Added
- Input validation specification (F-010) — pre-execution validation via `Executor.validate()`
- `Context` and trace ID passback specification for request tracing
- Updated response key from `output` to `result` in SRS and test plan

## [0.2.0] - 2026-02-20

### Added
- MCPServer framework integration specification — non-blocking server wrapper and lifecycle hooks
- Health endpoint specification for HTTP-based transports
- Project logo (`apcore-mcp-logo.svg`) and MkDocs configuration (`mkdocs.yml`)
- Expanded PRD to cover 25 initial features with detailed metrics

## [0.1.0] - 2026-02-15

### Added
- Initial project setup for apcore-mcp
- Core concept: automatic bridging of apcore modules to MCP Server and OpenAI Tools
- Initial Product Requirements Document (PRD), SRS, Technical Design, and Test Plan
