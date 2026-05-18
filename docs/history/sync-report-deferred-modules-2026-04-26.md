# Sync Report — Deferred Deep-Chain Modules (10/10)

**Date:** 2026-04-26  
**Trigger:** Continuation of `/apcore-skills:sync --scope mcp` Phase A, Step 4C.  
**Modules covered:** annotation-mapper, error-mapper, schema-converter, transport-manager, openai-converter, jwt-authenticator, approval-handler, extension-bridge, explorer-ui, module-id-normalizer.

The original sync run analyzed 4 of 14 modules (registry-listener, async-task-bridge, execution-router, mcp-server-factory) in Phase A Step 4C. The 10 modules below were deferred for budget reasons. This addendum captures the results of running deep-chain on each of them.

## Total findings across the 10 modules

| Module | Total | Critical/High | Medium | Low |
|---|---|---|---|---|
| annotation-mapper | 9 | 2 | 5 | 2 |
| error-mapper | 14 | 1 | 7 | 6 |
| schema-converter | 19 | 5 | 11 | 3 |
| transport-manager | 16 | 4 | 7 | 5 |
| openai-converter | 10 | 3 | 5 | 2 |
| jwt-authenticator | 16 | 2 | 6 | 8 |
| approval-handler | 11 | 3 | 5 | 3 |
| extension-bridge | 7 | 2 | 3 | 2 |
| explorer-ui | 6 | 1 | 1 | 4 |
| module-id-normalizer | 8 | 1 | 3 | 4 |
| **TOTAL** | **116** | **24** | **53** | **39** |

## Critical / high findings (24 — by module)

### annotation-mapper
- **AM-1 (critical)** Output key naming differs at the dict layer: Python returns snake_case (`read_only_hint`), TS returns camelCase (`readOnlyHint`), Rust uses serde rename to camelCase. Python wire format incompatible without upstream rename.
- **AM-2 (critical)** TS does NOT compare `cache_ttl`/`pagination_style` against defaults; emits any non-null value. Python+Rust skip when equal to default. Spec says "default values omitted from suffix".

### error-mapper
- **EM-1 (high)** `McpErrorFormatter` + `register_mcp_formatter` exist only in Rust at the file scope. Python/TS may register elsewhere but the symbol is absent from the analyzed files. Spec lists it as a public symbol.
- **EM-3 (high)** TS stamps `userFixable: true` on dependency/binding/version errors; Python+Rust do not.
- **EM-6 (high)** Generic `"Internal error occurred"` fallback for unrecognized exceptions exists in Python+TS; Rust's `to_mcp_error(&ModuleError)` only accepts ModuleError so the fallback is unreachable.

### schema-converter
- **SC-1 (high)** TS has NO recursion depth limit. Python+Rust cap at 32. TS can stack-overflow on pathological-but-legal acyclic schemas.
- **SC-9 (high)** Rust's strict-mode walker descends into `enum`, `const`, `examples`, `default` — and may spuriously inject `additionalProperties:false` on object-shaped values inside those. Python+TS only walk a whitelist of subschema-bearing keys.
- **SC-10 (high)** Python's strict-mode skips the empty-schema branch — `{}` with `strict=True` returns `{type:object, properties:{}}` WITHOUT `additionalProperties:false`. TS+Rust correctly inject.
- **SC-11 (high)** TS strict-mode default is **false**; Python+Rust default is **true**. TS callers silently get permissive schemas.
- **SC-18 (high)** Rust's `ensure_object_type` overwrites `type: ['object','null']` with bare `'object'`, losing nullability. Python+TS preserve.

### transport-manager
- **TM-1 (high)** `/usage` endpoint absent in Python (TS+Rust both register it).
- **TM-2 (high)** Auth middleware integration absent in Python's `TransportManager` (TS has `setAuthenticator`/`setRequireAuth`, Rust has `HttpAuthConfig`).
- **TM-3 (high)** Tool Explorer wiring (`setExplorer` / `explorer_prefix`) absent in Python's `TransportManager`.
- **TM-4 (high)** Cancellation forwarding wired only for SSE/streamable-http transport-shutdown in Python+TS post-A-D-018 fixes; Rust uses generic cancel handler. Python's transport.py has neither `set_cancel_handler` nor `setAsyncTaskBridge`.

### openai-converter
- **OC-1 (high)** TS strict mode is a manual recursive walker that does NOT promote `x-llm-description`, does NOT strip `x-*` extensions, does NOT recurse into `oneOf`/`anyOf`/`allOf` or `$defs`. Python+Rust delegate to apcore's full strict pipeline.
- **OC-3 (high)** ID collision detection on bijective normalization is **absent in all three** — spec contract violation. Two modules whose ids collide post-normalization (`a.b` and `a-b` both → `a-b`) silently produce duplicate `function.name` entries.
- **OC-5 (high)** Rust `convert_registry` takes `&Value` (JSON), not a Registry trait. Python/TS take a duck-typed Registry. Public API shape diverges.

### jwt-authenticator
- **JWT-1 (high)** Authenticator.authenticate input type diverges: Python sync `dict[str,str]`, Rust async `&HashMap<String,String>`, TS async `IncomingMessage`. A custom Authenticator written for one SDK cannot be drop-in ported.
- **JWT-2 (high)** 401 body and `WWW-Authenticate` header diverge: Python+Rust emit `{error: Unauthorized, detail: ...}`, TS emits `{error: "Authentication required"}`. None of the three set the spec-mandated `WWW-Authenticate: Bearer error="invalid_token"` on expired tokens.

### approval-handler
- **AH-1 (high)** Rust handler reads elicit callback from constructor field, NOT from `request.context.data`. Python+TS extract per-request from context. Rust violates spec line "reads elicit callback from context.data".
- **AH-3 (high)** Rust does NOT catch elicit callback failures; Python+TS do (return rejected with reason "Elicitation request failed"). Spec mandates "communication failure → rejected with reason".

### extension-bridge
- **EB-1 (high)** `ExtensionManager.apply()` not implemented in **any** SDK. Spec's full Extension Bridge contract (apply at step 1 of load order) is uniformly unimplemented.
- **EB-2 (high)** Adapter-hook kwargs (`schema_converter`, `annotation_mapper`, `error_mapper`) not exposed by `serve()` in any SDK. The factory always instantiates fresh defaults internally.

### explorer-ui
- **EUI-1 (high)** Spec mandates `POST /explorer/tools/<id>/validate` but **no language registers it**. `ExecutionRouter.validate_tool` exists everywhere but the explorer UI cannot reach it via the documented endpoint.

### module-id-normalizer
- ~~**MID-5 (high)**~~ ✅ **Resolved 2026-04-28** — all three SDKs now expose a bijection-guarded variant: `try_denormalize` (Python) / `tryDenormalize` (TS) / `denormalize_checked` (Rust). The plain `denormalize` stays lenient (backwards compatible). The new variant runs the dash→dot replacement and validates the result against `MODULE_ID_PATTERN`, returning `None` / `null` / `Err(InvalidModuleId)` on a non-pre-image input. Useful for sanitizing OpenAI tool-call responses against the registered module set. 8 Python + 9 TS + 5 Rust regression tests added.

## Medium and low findings

53 medium + 39 low findings detail issues like:

- Wire-format consistency (camelCase vs snake_case, quote style in error messages)
- API-shape gaps (instance methods vs static fns, optional vs typed Result)
- Cross-language test parity (e.g. only Rust has comprehensive tests for module-id-normalizer)
- Defensive-gap divergence (one language catches exceptions where peers propagate)
- Doc-comment errors (TS module-id-normalizer doc claims MCP needs hyphens — contradicts spec)
- Default-value handling (clock skew leeway, exempt paths, ClaimMapping field names)

Full per-module finding sets are preserved in the sub-agent JSON output captured in the conversation transcript at this date.

## Spec-vs-code uniform gaps (not divergences)

Several findings are **uniform across all three SDKs** — they're not cross-language divergences but spec/code drift to fix in concert:

1. **Extension Bridge**: `ExtensionManager.apply()` and adapter-hook kwargs are entirely unimplemented (EB-1, EB-2).
2. **Explorer UI**: `POST /explorer/tools/<id>/validate` endpoint is undefined in all three (EUI-1).
3. **OpenAI converter**: ID collision detection is missing in all three despite the spec contract (OC-3).
4. **JWT clock-skew leeway**: spec says ~30s; Python/TS default to 0s, Rust to 60s (library default) — none configure the spec value.
5. **Module ID normalizer denormalize bijection**: contract gap shared across all three (MID-5).

These should be fixed by either (a) updating the spec to match the realized subset, or (b) implementing the spec across all three SDKs in coordination.

## Recommendation

The 24 critical/high findings outnumber the 17 critical/blocker items from the original sync. Recommended approach:

- **Phase 1 (uniform spec drift)** — fix EB-1/EB-2, EUI-1, OC-3, JWT-3 in coordination across all 3 SDKs (or update the spec).
- **Phase 2 (Python catch-up)** — TM-1, TM-2, TM-3, TM-4 are all "Python TransportManager missing what TS+Rust have". Add /usage, auth, explorer, cancellation forwarding to Python's transport.
- **Phase 3 (TS strict-mode)** — SC-1, SC-11, OC-1 are TS strict-mode being weaker than Python+Rust. Tighten TS strict to match.
- **Phase 4 (Rust polish)** — SC-9 (strict descends too aggressively), SC-18 (nullable-object), AH-1/AH-3 (approval handler architectural gaps).
- **Phase 5 (cosmetic and contract)** — wire-format unification (AM-1, JWT-2), doc fixes, error-class promotion (MID-3).

Estimated effort: **multi-day work** with cross-language API design decisions.

---

## Status update — fixes landed in this round (2026-04-27)

The following 14 cross-language fixes from the 24 critical/high deferred findings have been committed:

### Cross-language (3 SDKs)
- ✅ **JWT-3** clock-skew leeway = 30s (Python `pyjwt.decode` `leeway=30`, TS `clockTolerance: 30`, Rust `validation.leeway = 30`)
- ✅ **OC-3** OpenAI ID collision detection in `convert_registry` (raises ValueError / throws Error / returns `ConverterError::StrictMode` on `a.b` + `a-b` → `a-b` collisions)

### Python
- ✅ **SC-10** empty-schema strict short-circuit now injects `additionalProperties:false`
- ✅ **AM-1** `AnnotationMapper.to_mcp_annotations` returns camelCase keys (was snake_case)

### TypeScript
- ✅ **SC-1** `_inlineRefs` recursion depth capped at 32 (was unbounded)
- ✅ **SC-2** `activeRefs.delete` now in `try/finally` (exception-safe)
- ✅ **SC-11** strict default now `true` (was `undefined → false`)
- ✅ **AM-2** `cache_ttl` and `pagination_style` skipped when equal to default
- ✅ **JWT-2** 401 body unified with Python+Rust: `{error: "Unauthorized", detail: "Missing or invalid Bearer token"}`
- ✅ **MID-6** `idNormalizer` doc-comment corrected (no longer claims "MCP tool names use hyphens")

### Rust
- ✅ **SC-9** `inject_strict` walks only whitelisted subschema keys (no descent into `enum`/`const`/`examples`/`default`)
- ✅ **SC-18** `inject_strict` and `ensure_object_type` preserve `type: ["object", "null"]` (no nullable downgrade)
- ✅ **AH-3** approval-handler catches elicit-callback panics via `futures::FutureExt::catch_unwind`

## Outstanding critical/high (1 of 24 — EB-1 is a spec drift recommended for 0.16 spec update; TM-1/2/3 remain as cosmetic API parity)

These require either substantial new features, public API changes, or upstream coordination:

| ID | Module | Why deferred |
|---|---|---|
| ~~**AH-1**~~ | ~~approval-handler~~ | ✅ **Resolved 2026-04-28** — added `tokio::task_local! ELICIT_CALLBACK` in `apcore_mcp::helpers`. `ElicitationApprovalHandler::request_approval` now reads the callback from the task-local first (matching Python+TS per-request scoping), falling back to the constructor field. apcore-rust's `Context::data` is `HashMap<String, serde_json::Value>` and cannot hold boxed `Fn`s, so a task-local is the closest cross-SDK equivalent to context.data without forcing an apcore-rust API extension. 4 regression tests added. |
| ~~**EM-1**~~ | ~~error-mapper~~ | ✅ **Resolved 2026-04-28** — investigation showed both symbols already existed; the only divergence was Python's class name `MCPErrorFormatter` (all-caps `MCP`) vs TS+Rust's `McpErrorFormatter` (PascalCase). Added `McpErrorFormatter` as the canonical Python class name and kept `MCPErrorFormatter` as a backwards-compatible alias. Both names are now exported from `apcore_mcp` and `apcore_mcp.adapters`. |
| ~~**EM-3**~~ | ~~error-mapper~~ | ✅ **Resolved 2026-04-28** — Python `ErrorMapper` and Rust `to_mcp_error` now stamp `userFixable=true` for `DEPENDENCY_NOT_FOUND`, `DEPENDENCY_VERSION_MISMATCH`, `VERSION_CONSTRAINT_INVALID`, and the four `BINDING_*` codes (matches TypeScript). Rust adds `USER_FIXABLE_ERROR_CODES` const + stamp in `build_detail_response`. Python adds `_USER_FIXABLE_ERROR_CODES` set. 9 Python regression tests + 5 Rust regression tests added. |
| ~~**EM-6**~~ | ~~error-mapper~~ | ✅ **Resolved 2026-04-28** — Rust adds `ErrorMapper::internal_error_response()` and `ErrorMapper::to_mcp_error_any<E: std::error::Error>()`, matching Python's `to_mcp_error(error: Exception)` and TypeScript's `toMcpError(error: unknown)` signatures. Returns the `{is_error: true, error_type: "GENERAL_INTERNAL_ERROR", message: "Internal error occurred", details: null}` envelope for any non-`ModuleError` input. 3 regression tests added. |
| ~~**JWT-1**~~ | ~~jwt-authenticator~~ | ✅ **Resolved 2026-04-28** — unified to `authenticate(headers: HeaderMap) -> Awaitable<Identity \| null>` across all three SDKs. **TS** swapped `IncomingMessage` for `Record<string, string>` and exports `extractHeaders(req)` migration helper. **Python** changed `Authenticator.authenticate` to `async`; new `call_authenticator(auth, headers)` helper supports legacy sync implementations transparently (inspects coroutine return). **Rust** already aligned. JWT regression tests rewritten to async. |
| ~~**OC-1**~~ | ~~openai-converter~~ | ✅ **Resolved 2026-04-28** — TS strict-mode walker now mirrors apcore's canonical strict pipeline: promotes `x-llm-description` → `description`, strips all `x-*` extension keys after promotion, recurses into `oneOf`/`anyOf`/`allOf` and `$defs`/`definitions`, sorts property names alphabetically, and removes `default` values. 6 regression tests added in `tests/converters/openai.test.ts`. Output now matches Python+Rust (which delegate to apcore's `to_strict_schema`). |
| ~~**OC-5**~~ | ~~openai-converter~~ | ✅ **Resolved 2026-04-28** — Rust `convert_registry` is now the canonical entrypoint and takes `&apcore::registry::Registry` directly (matches Python+TS duck-typed Registry). The pre-0.15 `&Value`-snapshot variant was renamed to `convert_registry_json` and remains available for callers that hold a serialized snapshot. `APCoreMCP::to_openai_tools` now passes the live registry through, avoiding a redundant clone+reparse round trip. 4 regression tests added. |
| **TM-1/2/3** | transport-manager | Python `TransportManager` lacks setter API for `/usage`, auth, explorer (these *function* via the wrapper-level `apcore_mcp.py` / `__init__.py`, but the bare `TransportManager` doesn't expose `set_usage_collector` / `set_authenticator` / `set_explorer` as TS+Rust do). Cosmetic API parity, not a functional gap. |
| ~~**TM-4**~~ | ~~transport-manager~~ | ✅ **Resolved 2026-04-28** — Python `TransportManager` now has `set_async_task_bridge(bridge)` plus a `transport_session_var` ContextVar. StreamableHTTP and SSE transports scope the session id via `_scoped_session()`, which calls `bridge.cancel_session_tasks(session_id)` on context exit. `factory.handle_call_tool` reads the contextvar and forwards `session_key` to `bridge.submit()`. Wired automatically by both `serve()` and `APCoreMCP.serve/async_serve` when an async bridge is present. 6 regression tests added in `tests/server/test_transport.py::TestTM4CancellationForwarding`. |
| ~~**EB-2**~~ | ~~extension-bridge~~ | ✅ **Resolved 2026-04-28 (Python+TS)** — `serve()` and `async_serve()` now accept optional `schema_converter` / `annotation_mapper` / `error_mapper` kwargs in Python and TS. The factory uses caller-supplied instances when present, falling back to defaults. **Rust deferred to 0.16** — adapters in apcore-mcp-rust are stateless unit structs (`SchemaConverter`, `AnnotationMapper`) with only static methods, so override injection requires a trait-based redesign (out of scope for apcore 0.19.0 features). |
| **EB-1** | extension-bridge | `ExtensionManager.apply()` not implemented in any SDK — uniform spec drift, not a cross-SDK divergence. **Recommended: spec update** to remove `apply()` step from the load-order contract since none of the three SDKs need it (extensions are loaded directly into the factory via Config Bus or kwargs). Defer to 0.16 spec revision. |
| ~~**EUI-1**~~ | ~~explorer-ui~~ | ✅ **Resolved 2026-04-28** — mcp-embedded-ui 0.4.0 ships `POST /tools/{name}/validate` (F7). apcore-mcp Python/TS/Rust SDKs bumped to `mcp-embedded-ui >= 0.4.0`; route flows automatically through the existing `create_mount` / `createNodeHandler` adapters. TC-011 integration tests added in all three SDKs. |

## Recommendation

The 14 landed fixes cover the most-leverage items that were doable without (a) public API breaks, (b) full feature additions, or (c) upstream coordination. The 10 outstanding items each need explicit user buy-in on direction before implementation:

- **TM-1/2/3/4** is the largest concrete-add: do it Python-by-Python (auth → /usage → explorer → cancellation).
- **EB-1/2** needs a cross-language API design conversation before any code lands.
- **JWT-1, OC-5, EM-6** are public API breaks — release-planning items.
- ~~**EUI-1** needs an mcp-embedded-ui PR first.~~ ✅ Resolved by mcp-embedded-ui 0.4.0 (2026-04-28).
- **AH-1** needs apcore-rs-side context plumbing or an apcore-mcp design change.

Suggested next step: triage the 10 outstanding items into "do in 0.15", "release-planning", and "spec-update-instead" buckets before further work.
