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
