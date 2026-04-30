# Execution Router

> Feature spec for code-forge implementation planning.
> Source: extracted from apcore-mcp/docs/tech-design-apcore-mcp.md
> Created: 2026-04-06

## Purpose

The Execution Router is the central dispatcher that receives tool-call requests from the MCP server and routes them through apcore's `Executor` pipeline. It ensures that every tool call from an AI agent is subject to the same rigorous validation, security (ACL), and middleware rules as any other module invocation in the apcore ecosystem.

## Scope

**Included:**
- Routing tool calls by `module_id` with incoming arguments (dict).
- Integration with apcore's `Executor.call_async()` to handle both sync and async modules.
- Serialization of module output (from dicts) into standard JSON strings for MCP `TextContent`.
- Redaction of sensitive fields in tool output before returning to the agent.
- Optional capturing of pipeline traces for observability.
- Pre-execution validation ("Preflight Check") to verify if a tool call would succeed.

**Excluded:**
- Direct module execution (always delegates to `Executor`).
- Protocol-level message handling (handled by the MCP SDK).

## Core Responsibilities

1. **Dispatcher** — Maps the tool name to the appropriate apcore `module_id` and executes it through the full 11-step pipeline.
2. **Output Normalization** — Converts various module output types into a consistent JSON string format using `json.dumps()` with a `default=str` fallback for non-serializable objects.
3. **Data Protection** — Automatically redacts fields marked as sensitive (`x-sensitive: true`) and keys matching `_secret_*` to prevent accidental leak of credentials to AI agents.
4. **Validation Gateway** — Provides a non-destructive preflight validation path (`Executor.validate()`) that checks ACLs and schemas without executing the actual module.

## Interfaces

### Inputs
- **Tool Name** (MCP Client) — The identifier of the tool to be called (maps to `module_id`).
- **Arguments** (MCP Client) — A dictionary of input values conforming to the tool's `input_schema`.

### Outputs
- **CallToolResult** (MCP SDK) — The final result (success with text content or error).
- **PreflightResult** (Internal/Explorer) — A summary of check results (module lookup, ACL, schema validation) without execution.

### Dependencies
- **apcore SDK (language-equivalent: apcore-python / apcore-js / apcore Rust crate)** — Provides the `Executor`, `Context`, and `Registry`.
- **Error Mapper** — Used to transform execution failures into formatted MCP error responses.

## Data Flow

```mermaid
graph TD
    A[MCP Tool Call] --> B[Execution Router]
    B --> C{Preflight Check?}
    C -- Yes --> D[Executor.validate]
    C -- No --> E[Executor.call_async]
    E --> F[Full 11-Step Pipeline]
    F --> G[Module Output]
    G --> H[Redact Sensitive Fields]
    H --> I[JSON Serialize]
    I --> J[CallToolResult Output]
```

## Key Behaviors

### Full Pipeline Enforcement
The router ensures every call passes through all 11 apcore pipeline steps: context creation, call-chain guard, module lookup, ACL check, approval gate, middleware before, input validation, execution, output validation, middleware after, and final return.

### ExtensionManager Plugin Points
When the server is constructed with an `ExtensionManager` (see `./extension-bridge.md` and `apcore/docs/features/extension-system.md`), the router itself does not register extensions — `ExtensionManager.apply()` runs during factory setup and mutates the underlying `Executor` and `Registry`. Three pipeline steps are the effective observation/mutation sites for extension-driven behavior:

- **Step 2 (module lookup)** — observes the `discoverer` and `module_validator` extension points, wired onto the `Registry` before the router is constructed.
- **Step 5 (middleware before) / Step 10 (middleware after)** — observe user-registered `middleware` extensions layered onto the `Executor`, including the `TracingMiddleware` fed by `span_exporter` extensions. The `acl` and `approval_handler` extensions take effect at Steps 4 and 5 respectively via their dedicated executor slots.

The router's contract with extensions is strictly pass-through: it forwards the current `Context` and arguments into the Executor and never skips pipeline steps, so registered extensions always fire exactly once per tool call.

### Output Redaction
Before the output is sent to the AI agent, the router applies recursive redaction. Any field with `x-sensitive: true` in its schema or any key starting with `_secret_` has its value replaced by `"***REDACTED***"`.

### Async-First Strategy
The router always uses `Executor.call_async()`. This allows the MCP server to remain responsive by offloading synchronous module executions to worker threads via `asyncio.to_thread()` automatically.

## Constraints

- **Thread Safety**: Must handle concurrent tool calls from multiple clients safely.
- **Latency**: The routing layer itself should add minimal overhead (< 5ms) beyond the actual module execution time.
- **Data Integrity**: Must operate on a deep copy of the output if redaction or transformation is required.

## Error Handling

- **Dispatch Failures**: Catches all execution errors (ModuleNotFoundError, ACLDeniedError, etc.) and routes them to the `ErrorMapper` to prevent raw tracebacks from reaching the client.
- **Serialization Failures**: Provides a safe fallback if the module output cannot be converted to JSON.

## Version Hint Negotiation

The Execution Router resolves an optional `version_hint` (semver range) for every tool call and forwards it to `Executor.call(..., version_hint=...)` so that apcore can pin module version resolution. The hint is cross-language and travels on the wire as MCP request metadata.

### Wire Contract
The MCP client carries the hint at `params._meta.apcore.version` (string). Example:
```json
{
  "method": "tools/call",
  "params": {
    "name": "my-module",
    "arguments": { "...": "..." },
    "_meta": { "apcore": { "version": ">=1.2.0" } }
  }
}
```

### Value Format
- Semver range string (e.g., `">=1.0.0"`, `"1.2.x"`, `"^2.0"`).
- Maximum length: 64 characters.
- Allowed charset: `[A-Za-z0-9.\-+_~^>=<* ]`.

### Precedence Order
When multiple sources supply a hint, the router resolves in this order (first match wins):
1. Explicit `extras.version_hint` / `extra.versionHint` — SDK caller-supplied (highest priority).
2. MCP request `_meta.apcore.version` — client-supplied.
3. Module descriptor default `metadata.version_hint` / `metadata.versionHint` (lowest).
4. `None` — no pinning; apcore resolves to the latest matching version.

### Validation
- Values exceeding the length cap or containing characters outside the allowed charset are silently dropped (treated as absent), with a `debug`-level log line for diagnosis.
- Malformed semver ranges are passed through to apcore unchanged; apcore rejects them with the standard `invalid_version_range` error, which is surfaced via the Error Mapper.
- The `_meta` dict is untrusted input from the MCP client; SDKs MUST bound-check and charset-validate before forwarding to the Executor (security note).

### Trace-Mode Caveat
When the request carries `_meta.trace == true`, `version_hint` may be unavailable in some SDKs pending apcore 0.19's `call_with_trace` signature extension. This is a known gap tracked by the Rust SDK TODO (`src/server/router.rs`, see `TODO(apcore>=0.19)`); implementations that cannot forward the hint through the trace path MUST still honor it on the non-trace path.

### Implementation References
- Python: `src/apcore_mcp/server/router.py` (version_hint extraction around the `_meta.apcore.version` branch).
- TypeScript: `src/server/router.ts` (`versionHint` resolution at the extras/`_meta.apcore.version`/descriptor cascade).
- Rust: `src/server/router.rs` (`handle_call` — `version_hint` extraction and `TODO(apcore>=0.19)` streaming-trace gap).

## Cancellation Handling

> **Status (v0.14.x, post-sync):** The `ExecutionRouter.cancel(call_id, reason)` public API now exists in all three SDKs (Python `ExecutionRouter.cancel`, TypeScript `ExecutionRouter.cancel`, Rust `ExecutionRouter::cancel_call`) with end-to-end wiring through `handle_call` (auto-registers a CancelToken per call, releases in a `finally` block / RAII guard, with FIFO-bounded eviction at 4096 entries). The router's cancel layer is now functional for callers that invoke `router.cancel(call_id)` directly.
>
> **MCP `notifications/cancelled` integration (clarification):** The MCP Python and TypeScript SDKs both ALREADY handle inbound `notifications/cancelled` *internally* — Python via `mcp.shared.session._receive_loop` which calls `self._in_flight[cancelled_id].cancel()` (anyio cancel scope), TypeScript via the SDK's built-in `setNotificationHandler(CancelledNotificationSchema, ...)` registered at protocol construction time. So inbound MCP cancel notifications **already** abort the in-flight `tools/call` request task at the SDK level — the executing `handle_call` will see a `CancelledError` / equivalent at its next `.await`. The router-level `cancel(call_id)` API is therefore a *cooperative* additional layer for modules that want to check `ctx.cancel_token.is_cancelled` and exit gracefully without raising; it is orthogonal to MCP-level task cancellation, not a replacement.

### What works today (v0.14.0)

- **Process-level cancellation (stdio):** When the parent MCP client closes the stdio pipe, the transport manager exits the read loop and the server shuts down. In-flight tool calls are abandoned (no cooperative drain). All three SDKs (Python, TypeScript, Rust) behave this way.
- **Async-task cancellation (Rust only):** Rust's `TransportManager` accepts a `cancel_handler` callback (`set_cancel_handler` / `notify_cancel`). At server build time, `apcore_mcp::APCoreMCP` wires this callback to `AsyncTaskBridge::cancel_session_tasks(session_key)` and `AsyncTaskBridge::cancel(task_id)`. When an MCP `notifications/cancelled` message is received, async-task-bridge tasks bound to that session are cooperatively cancelled. **Synchronous** tool calls (those routed through `ExecutionRouter::handle_call`) are NOT cancelled — there is no `CancelToken` map and no `ContextVar` propagation. Python and TypeScript SDKs do not implement this at all.
- **Error mapping for explicit `ExecutionCancelledError`:** if a module itself raises `ExecutionCancelledError` (e.g., on a timeout), the router catches it and forwards to the Error Mapper, which emits MCP error code `EXECUTION_CANCELLED`. This is downstream-only; nothing in the router *initiates* cancellation in response to MCP `notifications/cancelled`.

### What's still missing today

What's still missing today is automatic transport-level wiring of MCP `notifications/cancelled` for sync calls in Python+TS — Rust handles this via `TransportManager::set_cancel_handler` / `notify_cancel`. The cross-SDK ExecutionRouter dispatcher is a 0.15.0 roadmap item.

> **Note:** Rust uses `cancel_call(call_id, reason)` instead of `cancel` due to Rust naming idiom — same semantics.

### Roadmap (planned for 0.15.0)

The following design is preserved verbatim from earlier drafts as the implementation target:

- A per-server `Dict[call_id, CancelToken]` (guarded by a lock). On tool-call entry, the router generates/extracts the MCP `call_id`, creates a fresh `CancelToken`, inserts into the map, and attaches it to the `Context` via `Context.create(cancel_token=token)`. On completion the entry is removed in `finally`.
- An `ExecutionRouter.cancel(call_id, reason)` method that:
  1. Looks up the `CancelToken` for `call_id`.
  2. Calls `token.cancel()` (cooperative signal picked up by the module's next `token.check()`).
  3. Also calls `executor.cancel(call_id)` when the apcore feature is available.
  4. Emits a `mcp.call.cancelled` observability event with the reason.
- `ContextVar` propagation so the active token is visible across `asyncio.to_thread()` boundaries.
- Race handling: before-start tombstones, after-complete no-op-with-debug, idempotent concurrent-cancel.
- Cross-transport parity (stdio, streamable-http, sse) — all transports parse `notifications/cancelled` and forward to `ExecutionRouter.cancel`.

The `EXECUTION_CANCELLED` error code is already reserved in all three SDKs for this purpose; only the dispatcher half of the contract remains to be implemented.

## Notes

- This component is the primary security boundary between the untrusted AI agent and the internal apcore module environment.
- It leverages the `ContextVar` system to preserve identity and tracing across async boundaries.

---

## Contract: ExecutionRouter.handle_call

### Inputs
- tool_name: str, required, validates[non-empty], reject_with=(error content, is_error=True, None)
- arguments: dict[str, Any], required, validates[dict type]
- extra: dict[str, Any] | None, optional — may contain: `progress_token`, `send_notification`, `session`, `identity`, `call_id`, `_meta`, `version_hint`

### Errors
- Returns `(content, is_error=True, trace_id)` with ErrorMapper output — for all execution exceptions (ModuleNotFoundError, ACLDeniedError, etc.)
- Returns `(content, is_error=True, None)` with validation message — when `validate_inputs=True` and schema fails
- Never raises; all exceptions are caught and converted to error tuples

### Returns
- On success: tuple `(content: list[dict], is_error: bool, trace_id: str|None)`
  - `content` is a list of `{"type": "text", "text": str}` dicts
  - `is_error: False` on success, `True` on any execution failure
  - `trace_id` is `context.trace_id` when context is available, else None
- Version hint 3-source cascade: 1) `extra.version_hint`, 2) `extra._meta.apcore.version`, 3) `descriptor.metadata.version_hint`; first non-None wins; passed to `Executor.call_async(version_hint=...)`

### Properties
- async: true
- thread_safe: true
- pure: false
- idempotent: false

---

## Contract: ExecutionRouter.from_executor

### Inputs
- executor: Any, required (duck-typed) — must expose async `call_async(module_id, inputs, context?)` method
- validate_inputs: bool, optional, default=False
- output_formatter: callable | None, optional
- redact_output: bool, optional, default=True
- output_schema_map: dict | None, optional
- trace: bool, optional, default=False
- async_bridge: AsyncTaskBridge | None, optional
- descriptor_lookup: callable | None, optional

### Errors
- No constructor errors (fail-late at call time)

### Returns
- On success: ExecutionRouter instance

### Properties
- async: false
- thread_safe: true
- pure: false
- idempotent: true
