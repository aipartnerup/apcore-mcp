# Async Task Bridge

> Feature spec for code-forge implementation planning.
> Source: extracted from apcore-mcp/docs/tech-design-apcore-mcp.md (F-043)
> Created: 2026-04-15

## Overview

The Async Task Bridge exposes apcore's `AsyncTaskManager` (see apcore `docs/features/async-tasks.md`) through the MCP protocol so that long-running or fire-and-forget module invocations can be submitted, tracked, cancelled, and harvested by AI agents without blocking the stdio/HTTP transport. Modules whose descriptor carries an async hint (e.g. `metadata.async = true` or `annotations.extra["mcp_async"] = "true"`) are routed to `AsyncTaskManager.submit()` instead of the synchronous `Executor.call_async()` path used by the Execution Router, and the agent receives a lightweight handle (`task_id`) it can poll via dedicated meta-tools.

## Scope

**Included:**
- Detection of async-hinted modules at tool-call dispatch time.
- Routing such calls through `AsyncTaskManager.submit()` and returning an immediate `{"task_id": ..., "status": "pending"}` envelope.
- Four MCP meta-tools (`__apcore_task_submit`, `__apcore_task_status`, `__apcore_task_cancel`, `__apcore_task_list`) that wrap the manager API.
- Progress notifications via MCP `notifications/progress` when the module emits progress events through the execution context.
- Shared lifecycle definitions with apcore (`pending` -> `running` -> `completed` | `failed` | `cancelled`).

**Excluded:**
- Implementation of `AsyncTaskManager` itself (provided by apcore).
- Persistent task storage across server restarts (in-memory only; matches apcore).
- Scheduling or cron semantics; the bridge only surfaces single-shot submissions.

## Core Responsibilities

1. **Async Dispatcher** - Inspect the resolved `ModuleDescriptor`; if it carries an async hint, call `AsyncTaskManager.submit(module_id, inputs, context)` and return the `task_id` envelope. Otherwise, delegate unchanged to the Execution Router.
2. **Meta-Tool Surface** - Register four reserved tool names (`__apcore_task_*`) with the MCP server so clients can submit, query, cancel, and list tasks without calling the manager directly.
3. **Progress Fan-Out** - Subscribe to module-emitted progress events on the execution context and relay them as MCP `notifications/progress` with the task's `progressToken` (when the caller supplied one via `_meta.progressToken`).
4. **Result Retrieval** - On `__apcore_task_status`, if the task is `completed`, include the (redacted, JSON-serialised) result inline; if `failed`, include the error message mapped through the Error Mapper.
5. **Lifecycle Guardrails** - Reject submission with a protocol error when the manager's `max_tasks` cap is reached; surface capacity errors through the Error Mapper.

## Interfaces

### Inputs
- **Tool Name + Arguments** (MCP Client) - Normal `tools/call` requests; the bridge inspects the descriptor to decide async routing.
- **Meta-Tool Arguments** - `task_id` (for status/cancel) or optional `status` filter (for list).

### Outputs
- **Task Envelope** - `{"task_id": str, "status": "pending"}` returned immediately on async submission.
- **TaskInfo Projection** - JSON view of apcore's `TaskInfo` (task_id, module_id, status, timestamps, result, error) returned by status/list.
- **Progress Notification** - `notifications/progress` events with `{progressToken, progress, total?, message?}`.

### Dependencies
- **Execution Router** - Bridge sits in front of the router; forwards sync modules unchanged.
- **Error Mapper** - Translates capacity, not-found, and execution errors into MCP error payloads.
- **apcore `AsyncTaskManager`** - Backing implementation (lifecycle, concurrency, cleanup).

## MCP Surface

### Meta-Tools

| Tool | Arguments | Behavior |
|------|-----------|----------|
| `__apcore_task_submit` | `module_id: str` (required), `arguments?: object` (default `{}`), `version_hint?: str` | Explicit submission path. Returns `{task_id, status: "pending"}`. |
| `__apcore_task_status` | `task_id: str` (required) | Returns the `TaskInfo` projection; includes `result` when `completed`, `error` when `failed`. |
| `__apcore_task_cancel` | `task_id: str` (required) | Calls `AsyncTaskManager.cancel()`. Returns `{task_id, cancelled: bool}`. |
| `__apcore_task_list` | `status?: "pending"\|"running"\|"completed"\|"failed"\|"cancelled"` | Returns `{tasks: TaskInfo[]}`. |

`arguments` is **optional** on `__apcore_task_submit` — callers may submit a module with no inputs. All three SDKs (Python, TypeScript, Rust) advertise schemas with `required: ["module_id"]` only and `additionalProperties: false`.

Meta-tool names are reserved (double-underscore `__apcore_` prefix) and MUST NOT collide with user-registered modules; the MCP Server Factory rejects any module whose id starts with `__apcore_`.

### Progress Notifications

When the caller includes `_meta.progressToken` on the original `tools/call`, the bridge:
1. Stores the mapping `task_id -> progressToken`.
2. Attaches a progress sink to the execution context before handing to `AsyncTaskManager.submit()`.
3. Emits `notifications/progress` for each progress event and one final event on terminal transition.

## Task Lifecycle

```
          submit()                slot acquired             terminal
pending ----------->  running ---------------------->  completed | failed | cancelled
   |                     |
   +-- cancel() ---------+-- cancel() (cooperative via CancelToken)
```

The bridge only projects apcore's lifecycle; it does not add new states. Terminal results are retained until the configured cleanup interval elapses (default: 3600 s, matching `AsyncTaskManager.cleanup()`).

## Configuration

| Key | Default | Description |
|-----|---------|-------------|
| `async.max_concurrent` | `10` | Forwarded to `AsyncTaskManager(max_concurrent=...)`. |
| `async.max_tasks` | `1000` | Forwarded to `AsyncTaskManager(max_tasks=...)`. |
| `async.cleanup_interval_s` | `3600` | Age threshold passed to periodic `cleanup()`. |
| `async.hint_keys` | `["metadata.async", "annotations.extra.mcp_async"]` | Descriptor paths inspected to classify a module as async. |
| `async.meta_tools_enabled` | `true` | When `false`, the four `__apcore_task_*` tools are not registered and async routing is disabled. |

## Error Handling

- **Capacity exceeded** - `AsyncTaskManager.submit()` raises when `max_tasks` is reached; the bridge maps this to an `ASYNC_CAPACITY_EXCEEDED` protocol error via the Error Mapper.
- **Unknown task_id** - `__apcore_task_status` / `_cancel` return an `ASYNC_TASK_NOT_FOUND` error when the manager has no record (including after cleanup eviction).
- **Non-async module on `__apcore_task_submit`** - Returns `ASYNC_MODULE_NOT_ASYNC` to prevent accidental async-wrapping of sync-only modules. Agents are expected to use regular `tools/call` for those.
- **Result retrieval before completion** - `__apcore_task_status` returns the current non-terminal status without a `result` field; `get_result()` is only called for terminal `completed` tasks.
- **Progress sink failures** - Logged at `warning` and swallowed; they never fail the underlying task.
- **Shutdown** - On server shutdown, `AsyncTaskManager.shutdown()` is awaited so pending/running tasks are cancelled deterministically.

## Notes

- The bridge is a thin routing layer; all concurrency, cancellation, and cleanup semantics are delegated to apcore's `AsyncTaskManager`.
- Meta-tool names align with MCP's convention of reserved double-underscore identifiers to avoid polluting the public tool namespace.
- Output of `__apcore_task_status` passes through the same redactor used by the Execution Router so sensitive fields in task results are masked consistently.

---

## Contract: AsyncTaskBridge.submit

> **Trust boundary.** `submit()` is a trusted internal entrypoint reached only after the meta-tool dispatcher (`__apcore_task_submit` handler) has already validated `module_id`. The validation rule "non-empty string, not starting with `__apcore_`" is enforced at the dispatcher (see `## Contract: AsyncTaskBridge._handle_submit_tool`), not at `submit()` itself, so direct callers (tests, internal call sites) bypass it by design. The router uses the bridge's `is_reserved_id()` / `is_async_module_*()` guards before invoking `submit()` for the dynamic-routing path.

### Inputs
- module_id: str, required, **validation deferred to caller** (meta-tool dispatcher in production; direct callers are trusted)
- arguments: dict[str, Any], required
- context: Context | None, optional — identity and trace_parent extracted when present
- progress_token: Any | None, optional — when provided alongside send_notification, the bridge wires a progress sink so module-emitted progress events are fanned out as MCP `notifications/progress`. The installation point is language-specific (see ## Cross-Language Notes):
  - **Python / TypeScript**: progress sink is installed into `context.data[MCP_PROGRESS_KEY]` (`"_mcp_progress"`) **before** calling `AsyncTaskManager.submit`. Modules read the sink via `context.data["_mcp_progress"]`.
  - **Rust**: progress sender is registered on bridge-side maps (`progress_senders[task_id]`) **after** `AsyncTaskManager.submit` returns the task_id. Modules receive progress through a bridge-managed channel keyed by `task_id` (not via `context.data`, since Rust's `Context<Value>` uses `serde_json::Value` and cannot hold async closures).
- send_notification: async callable | None, optional — paired with progress_token for fan-out
- session_key: str | None, optional — when provided, task_id is recorded in _session_tasks for mass-cancel on disconnect

### Errors
- TaskLimitExceededError(code=TASK_LIMIT_EXCEEDED) — when AsyncTaskManager max_tasks cap is reached (retryable: true)
- ValueError — propagated from `AsyncTaskManager.submit` if the underlying executor rejects the module

### Returns
- On success: dict `{"task_id": str, "status": "pending"}` — immediate envelope; task runs in background
- On failure: raises (callers should map through ErrorMapper)

### Properties
- async: true
- thread_safe: true
- pure: false
- idempotent: false

### Cross-Language Notes

The progress-sink installation point differs between Python/TypeScript (in-context) and Rust (bridge-managed channel) because Rust's `Context<Value>` (where the type parameter resolves to `serde_json::Value`) cannot hold async closures. Both contracts produce the same wire-level `notifications/progress` events for consumers — the behavioral contract for AI clients (receiving notifications/progress events keyed by `progressToken`) is identical. The implementation surface differs by language:

| Language     | Install when             | Where                                                | Module reads from                                       |
|--------------|--------------------------|------------------------------------------------------|---------------------------------------------------------|
| Python       | Before `manager.submit`  | `context.data[MCP_PROGRESS_KEY]` (`"_mcp_progress"`) | `context.data["_mcp_progress"]`                         |
| TypeScript   | Before `manager.submit`  | `context.data[MCP_PROGRESS_KEY]` (`"_mcp_progress"`) | `context.data["_mcp_progress"]`                         |
| Rust         | After `manager.submit`   | `progress_senders[task_id]` on the bridge            | Bridge-managed channel keyed by `task_id`               |

This asymmetry is structural, not behavioral: when porting modules between languages, the module-facing API differs (context lookup vs. channel-on-bridge), but the wire-level progress event stream observed by MCP clients is byte-equivalent.

---

## Contract: AsyncTaskBridge.is_async_module

### Inputs
- descriptor: Any, required (duck-typed) — inspects `descriptor.metadata["async"]` and `descriptor.annotations.extra["mcp_async"]`

### Errors
- No exceptions raised; None input returns False

### Returns
- On success: bool — True when `metadata.async == True` (bool) OR `annotations.extra["mcp_async"] == "true"` (string, case-insensitive); False otherwise
- On failure: returns False (never raises)

### Properties
- async: false
- thread_safe: true
- pure: true
- idempotent: true

---

## Contract: AsyncTaskBridge._handle_status_tool

> Private method — invoked internally via `handle_meta_tool()` dispatch, not called directly by external consumers.

### Inputs
- task_id: str, required, validates[non-empty string], reject_with=error tuple

### Errors
- Returns `(content, is_error=True, None)` with `{"error": "ASYNC_TASK_NOT_FOUND", "task_id": ...}` — when task_id unknown or evicted after cleanup
- Returns error tuple — when task_id is empty or missing

### Returns
- On success: tuple `(list[dict], bool, str|None)` — content contains JSON-serialised TaskInfo projection; includes `result` field when status is `completed`; includes `error` field when status is `failed`; result is passed through redactor when available
- On failure: returns error tuple (never raises)

### Properties
- async: false
- thread_safe: true
- pure: false
- idempotent: true

---

## Contract: AsyncTaskBridge._handle_cancel_tool

> Private method — invoked internally via `handle_meta_tool()` dispatch, not called directly by external consumers.

### Inputs
- task_id: str, required, validates[non-empty string], reject_with=error tuple

### Errors
- Returns `(content, is_error=True, None)` with `{"error": "ASYNC_TASK_NOT_FOUND"}` — when task_id unknown

### Returns
- On success: tuple `(list[dict], bool, str|None)` — content contains `{"task_id": str, "cancelled": bool}`; `cancelled` is True when AsyncTaskManager successfully cancelled the task
- On failure: returns error tuple (never raises)

### Properties
- async: true
- thread_safe: true
- pure: false
- idempotent: false

---

## Contract: AsyncTaskBridge.handle_preview

> Handler for the `__apcore_module_preview` meta-tool introduced in 0.15.0. Renders a non-executing preview of a module's input schema, description, and metadata so agents can decide whether to invoke before paying the cost of an actual call.

### Inputs
- module_id: str, required, validates[non-empty string, not starting with `__apcore_`], reject_with=error tuple
- arguments: dict[str, Any] | None, optional — preserved as-is in the preview envelope when null (distinguishing "no arguments supplied" from "empty object supplied")
- version_hint: str | None, optional — forwarded to descriptor lookup

### Errors
- Returns `(content, is_error=True, None)` with `{"error": "PREVIEW_UNAVAILABLE", "module_id": ...}` — when descriptor cannot be resolved OR module does not support preview
- Returns `(content, is_error=True, None)` with `{"error": "MODULE_NOT_FOUND", "module_id": ...}` — when registry has no descriptor for module_id
- Returns error tuple — when module_id is empty or starts with reserved `__apcore_` prefix

### Returns
- On success: tuple `(list[dict], bool, str|None)` — content contains `{"module_id": str, "input_schema": dict, "description": str, "annotations": dict, "arguments": dict | None}`; `arguments` field reflects the input verbatim (null preserved as null, not coerced to empty object)
- On failure: returns error tuple (never raises)

### Properties
- async: true
- thread_safe: true
- pure: false
- idempotent: true

---

## Contract: AsyncTaskBridge.__init__ (executor parameter)

> Constructor parameter introduced in 0.15.0. The bridge previously bound to its manager's default executor; the explicit `executor` kwarg lets callers route async submissions through a different executor (e.g., a sandboxed executor for untrusted modules) while sharing the manager.

### Inputs
- manager: AsyncTaskManager, required
- executor: Executor | None, optional — defaults to `manager.executor` (the manager-bound executor) when omitted; when provided, the bridge calls `manager.submit(..., executor=executor)` for every dispatch and for the `__apcore_module_preview` handler
- redactor: Redactor | None, optional — passed through to `_handle_status_tool` for result masking
- options: AsyncTaskBridgeOptions | None, optional (TS) — `{ executor?, redactor? }` aggregate form for ergonomic construction

### Errors
- TypeError(code=INVALID_ARG) — when manager is None
- TypeError(code=INVALID_ARG) — when executor is provided but does not implement the `Executor` interface (Python: missing `call_async`; TS: missing `callAsync`; Rust: type-checked at compile time)

### Returns
- On success: AsyncTaskBridge instance
- On failure: raises (constructor)

### Properties
- async: false
- thread_safe: false (constructor must be called from a single thread; instance is thread-safe after construction)
- pure: false
- idempotent: false
