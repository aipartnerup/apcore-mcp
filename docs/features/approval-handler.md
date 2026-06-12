---
description: "Approval Handler feature spec: ElicitationApprovalHandler implements human-in-the-loop confirmation via MCP elicitation, mapping user accept/reject responses to apcore ApprovalResult."
---

# Approval Handler

> Feature spec for code-forge implementation planning.
> Source: extracted from apcore-mcp/docs/srs-apcore-mcp.md
> Created: 2026-04-06

## Purpose

The Approval Handler implements apcore's "Human-in-the-Loop" safety mechanism by leveraging the Model Context Protocol (MCP) elicitation capability. When a tool call is flagged as requiring approval (e.g., destructive or high-cost operations), the handler interrupts the execution flow and asks the human user for explicit confirmation before proceeding.

## Scope

**Included:**
- `ElicitationApprovalHandler` implementation of the `ApprovalHandler` protocol.
- Integration with the MCP elicitation request (`sampling/createMessage` or tool-specific elicitation).
- Formatting of the approval request (module ID, description, and proposed arguments).
- Mapping of user responses (accept, reject) to `ApprovalResult`.
- Support for CLI flags to configure approval behavior (`elicit`, `auto-approve`, `always-deny`).

**Excluded:**
- Implementation of the `Executor` approval gate (provided by apcore).
- Direct UI interaction (handled by the MCP client, e.g., Claude Desktop).

## Core Responsibilities

1. **Safety Intercept** — Receives the `ApprovalRequest` from the Executor and prepares the protocol-level message for the human user.
2. **Elicitation Bridge** — Calls the MCP elicitation callback (stored in the execution context) and waits asynchronously for the user's response.
3. **Action Mapping** — Translates the user's action (e.g., clicking "Approve" or "Cancel") into the formal `ApprovalResult` (status and optional reason) required by the Executor.
4. **Error Resiliency** — Handles cases where the client does not support elicitation or the user cancels the request by failing the approval safely.

## Interfaces

### Inputs
- **ApprovalRequest** (apcore Executor) — Contains the `module_id`, `description`, and `arguments` of the tool call awaiting approval.

### Outputs
- **ApprovalResult** (apcore Executor) — The decision (`approved` or `rejected`) returned to the pipeline.
- **Elicitation Message** (MCP Client) — A structured request displayed to the user.

### Dependencies
- **apcore SDK (language-equivalent: apcore-python / apcore-js / apcore Rust crate)** — Provides the `ApprovalHandler` protocol and related types.
- **MCP SDK (language-equivalent: mcp Python / @modelcontextprotocol/sdk / mcp-sdk Rust crate)** — Provides the elicitation protocol handlers.

## Data Flow

```mermaid
graph TD
    A[Executor: approval_gate] --> B[ElicitationApprovalHandler]
    B --> C[Extract Elicit Callback from Context]
    C --> D[Format User Message]
    D --> E[Call Client Elicitation]
    E --> F{User Response?}
    F -- Approve --> G[ApprovalResult: approved]
    F -- Deny --> H[ApprovalResult: rejected]
    F -- No Support --> H
    G --> I[Resume Execution]
    H --> J[Return Error to Agent]
```

## Key Behaviors

### Structured User Prompt
The handler generates a human-friendly message that includes the tool's purpose and the exact arguments being passed. This ensures the user has full context before granting approval.

### Fail-Safe Rejection
If the MCP client does not support elicitation, if the connection is lost during the request, or if the user simply cancels, the handler defaults to a "Rejected" result. Security is preserved by never assuming implicit approval.

### CLI Configuration
The system supports global approval modes via the `--approval` CLI flag:
- `elicit`: Always ask the user (default for high-security environments).
- `auto-approve`: Automatically grant approval for all modules (development mode).
- `always-deny`: Block all tools that require approval (maximum lockdown).

## Constraints

- **Non-Blocking Loop**: The handler must not block the entire MCP server while waiting for a single user's response; other clients must remain responsive.
- **Timeout Management**: The handler should respect any execution timeouts configured for the tool call.

## Error Handling

- **Elicitation Failure**: If the protocol message cannot be sent, the handler returns a rejected `ApprovalResult` with a "Communication failed" reason.
- **Malformed Response**: If the client returns an unrecognized response, it is treated as a rejection.

## Notes

- This feature is a primary defense against unintended side effects in autonomous agent workflows.
- It leverages the standard `sampling` or `elicitation` features of the MCP protocol to provide a native-feeling UI for the user.

---

## Contract: ElicitationApprovalHandler.request_approval

### Inputs
- request: ApprovalRequest, required, validates[has `context`, `module_id`, `description`, `arguments` fields], reject_with=ApprovalResult(status="rejected")
- request.context.data[MCP_ELICIT_KEY]: async callable, required per-call (Python / TypeScript) — extracted from context at call time (not constructor), reject_with=ApprovalResult(status="rejected", reason="No elicitation callback available")

> **Rust contract variation**: `ElicitationApprovalHandler::new(elicit)` accepts an `Option<Arc<dyn ElicitCallback>>` at construction. Because Rust's `Context<Value>` (with `serde_json::Value`) cannot store async closures, the handler cannot extract a callback from `request.context.data` per-call (which is always the case for Rust's `serde_json::Value`-typed context). Instead, Rust uses the constructor-injected callback for every `request_approval` call. The rejection semantics ("No context available for elicitation", "No elicitation callback available") are preserved: if construction omitted the callback, the handler returns `ApprovalResult(status="rejected", reason="No elicitation callback available")`.

### Errors
- Returns `ApprovalResult(status="rejected", reason="No context available for elicitation")` — when request.context is None or has no `data`
- Returns `ApprovalResult(status="rejected", reason="No elicitation callback available")` — when MCP_ELICIT_KEY not in context.data
- Returns `ApprovalResult(status="rejected", reason="Elicitation request failed")` — when elicit callback raises any exception
- Returns `ApprovalResult(status="rejected", reason="Elicitation returned no response")` — when elicit callback returns None
- Never raises; all failure modes return a rejected ApprovalResult

### Returns
- On success (action == "accept"): `ApprovalResult(status="approved")`
- On any other action (decline/cancel/any non-accept string): `ApprovalResult(status="rejected", reason="User action: {action}")`
- Arguments are formatted as JSON string in the elicitation message

### Properties
- async: true
- thread_safe: true
- pure: false
- idempotent: false

### Cross-Language Notes

The elicit-callback source differs between Python/TypeScript and Rust:

| Language     | Callback source                                                              |
|--------------|------------------------------------------------------------------------------|
| Python       | `request.context.data[MCP_ELICIT_KEY]` (per-call lookup in execution context)|
| TypeScript   | `request.context.data[MCP_ELICIT_KEY]` (per-call lookup in execution context)|
| Rust         | Constructor-injected `Option<Arc<dyn ElicitCallback>>` (set on handler ctor) |

The wire-level rejection semantics ("No context available for elicitation", "No elicitation callback available", "Elicitation request failed", "Elicitation returned no response") are preserved across all three SDKs. The structural difference exists because Rust's `Context<Value>` is parameterized over `serde_json::Value`, which cannot hold async closures.

---

## Implementation Notes — Failure Containment

The elicit callback's failure containment guarantees differ across language SDKs:

- **Python**: `try/except Exception` covers thrown exceptions and awaited rejections. Does NOT catch `BaseException` subclasses (`KeyboardInterrupt`, `SystemExit`, `asyncio.CancelledError` in Python 3.8+) — these propagate.
- **TypeScript**: `try/catch` covers thrown errors and rejected Promises. JavaScript has no panic semantics distinct from exceptions, so this is the strongest guarantee available on the runtime.
- **Rust**: `futures::FutureExt::catch_unwind` additionally catches `panic!()` across `.await` points, converting them to `ApprovalResult(status="rejected", reason="Elicitation request failed")`. This is the strongest guarantee — even unwinding panics in user-supplied elicit callbacks are contained.

When porting modules between languages, treat panic-safety as Rust-only. JavaScript/TypeScript and Python module authors should not rely on panic-recovery semantics, because their runtimes either do not distinguish panics from exceptions (JS) or let certain `BaseException`s bypass the catch block (Python). See audit finding **D11-108** for the original gap analysis.
