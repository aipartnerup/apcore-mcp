# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.15.0] - 2026-05-09

Cross-SDK release leveraging **apcore 0.21.0 + apcore-toolkit 0.6.0**.
Promotes three new upstream capabilities into MCP-facing surface area
across all three SDKs (`apcore-mcp-python`, `apcore-mcp-rust`,
`apcore-mcp-typescript`). Byte-equivalent meta-tool, error envelope,
and Markdown rendering across languages.

### Changed

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
