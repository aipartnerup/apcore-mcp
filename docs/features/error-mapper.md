---
description: "Error Mapper feature spec: translates apcore ModuleError subclasses (validation, ACL, timeout) into MCP CallToolResult errors with AI guidance metadata and sensitive-detail redaction."
---

# Error Mapper

> Feature spec for code-forge implementation planning.
> Source: extracted from apcore-mcp/docs/tech-design-apcore-mcp.md
> Created: 2026-04-06

## Purpose

The Error Mapper is responsible for translating apcore's rich error hierarchy (including schema validation errors, ACL denials, and timeouts) into structured, protocol-compliant error responses for MCP clients and OpenAI's API. It ensures that AI agents receive actionable, machine-readable feedback without leaking sensitive system information.

## Scope

**Included:**
- Mapping all `ModuleError` subclasses to `CallToolResult` error responses.
- Formatting field-level schema validation details for display in the client.
- Redaction of sensitive details (e.g., specific `caller_id` or `trace_id` in ACL denials) for protocol-facing output.
- Attachment of AI guidance metadata (e.g., `retryable`, `aiGuidance`) to error responses.
- Sanitation of unexpected non-module exceptions into generic internal error messages.

**Excluded:**
- Implementation of the `Executor` (where errors originate).
- Protocol-level response wrapping (handled by the `ExecutionRouter`).

## Core Responsibilities

1. **Category Mapping** — Maps specific apcore exceptions (e.g., `ModuleNotFoundError`, `ModuleTimeoutError`) to corresponding protocol-level error codes and formatted messages.
2. **Detail Extraction** — Extracts structured information (e.g., specific fields that failed validation) and converts them into a human-readable, bulleted string.
3. **AI Guidance Injection** — Attaches metadata to the error response to help agents understand whether to retry, what the user should fix, or what the model should adjust.
4. **Information Security** — Replaces internal exceptions and stack traces with sanitized, safe messages (e.g., "Internal error occurred") for external consumption.

## Interfaces

### Inputs
- **Exception Instance** (Execution Router) — The error raised during the execution pipeline.

### Outputs
- **CallToolResult** (MCP SDK) — A protocol result object with `isError=true` and formatted text content.

### Dependencies
- **apcore SDK (language-equivalent: apcore-python / apcore-js / apcore Rust crate)** — Provides the `ModuleError` base class and its specific subclasses.
- **language MCP SDK** (Python: `mcp`; TypeScript: `@modelcontextprotocol/sdk`; Rust: `mcp-sdk`) — Provides the `CallToolResult` and `TextContent` types.

> **Note:** Python and Rust additionally expose `internal_error_response()` and `to_mcp_error_any(error)` for generic-error fallback (EM-6, CHANGELOG 0.14.0). Python exports both `McpErrorFormatter` (canonical) and `MCPErrorFormatter` (alias) per EM-1.

## Data Flow

```mermaid
graph LR
    A[apcore Exception] --> B[Identify Error Type]
    B --> C[Extract Field Details]
    C --> D[Apply AI Guidance]
    D --> E[Sanitize Message]
    E --> F[CallToolResult Output]
```

## Key Behaviors

### Structured Validation Feedback
For `SchemaValidationError`, the mapper generates a multi-line, bulleted summary showing each failed field, its error code, and its human-readable message.

### AI Guidance Metadata
If an error object carries guidance attributes (`retryable`, `ai_guidance`, `user_fixable`, `suggestion`), the mapper extracts these and attaches them to the response, often as a JSON appendix in the error text content to ensure visibility for LLMs.

### Security Sanitization
Only errors derived from `ModuleError` (which are designed to be user-facing) are allowed to pass their messages to the client. All other exceptions result in a generic "Internal error occurred" message, with the original traceback only visible in server-side logs.

## Constraints

- **Consistency**: The same error code from apcore must always result in the same message format and wire key convention.
- **Protocol Requirements**: Every mapped error result must explicitly set `isError=True`.
- **Wire Case**: Use camelCase for keys in the JSON guidance appendix to align with MCP/OpenAI conventions.

## Error Handling

- **Unrecognized Error**: If the mapper receives a `ModuleError` it doesn't specifically know, it returns a generic "Module error: {code}" response using the original code.
- **Extreme Failure**: If the mapping logic itself fails, it reverts to a hardcoded "Internal error occurred" safety response.

## Notes

- This component is critical for the "auto-retry" and "intelligent fix" behaviors in AI agents.
- It leverages apcore's built-in error code registry to stay synchronized with the core framework.

---

## Contract: ErrorMapper.to_mcp_error

### Inputs
- error: per-language input type, required:
  - **Python**: `Exception` — any exception; duck-typed for ModuleError-like (must have `code`, `message`, `details` attributes for the apcore path)
  - **TypeScript**: `unknown` — any value; duck-typed for ModuleError-like; non-matching values fall through to the INTERNAL_ERROR envelope
  - **Rust**: `&apcore::ModuleError` only — strongly typed; for arbitrary `&dyn std::error::Error` use `to_mcp_error_any` (see EM-6 below)

### Errors
- Never raises; all exceptions produce a valid error dict

### Returns
- On success: dict with camelCase top-level keys — `isError: true`, `errorType: str`, `message: str`, `details: dict|None`
- All SDKs return camelCase keys directly — Python uses isError/errorType with no intermediate snake_case conversion
- Additional optional fields: `retryable: bool`, `aiGuidance: str`, `userFixable: bool`, `suggestion: str` — populated from error attributes when present
- `userFixable: true` — TypeScript only (hardcoded at errors.ts:241,273). Python: userFixable is attached only when error.user_fixable is truthy (via _attach_ai_guidance); DependencyNotFoundError does not set user_fixable by default, so Python responses for these codes will NOT include userFixable:true unless apcore sets it explicitly
- Non-ModuleError exceptions → `errorType: "GENERAL_INTERNAL_ERROR"`, `message: "Internal error occurred"`, `details: null` (matches `internal_error_response()`; see EM-6 below)
- ExecutionCancelledError → `errorType: "EXECUTION_CANCELLED"`, `retryable: true`
- CircuitBreakerOpenError → `errorType: "CIRCUIT_BREAKER_OPEN"`, `retryable: true`, `aiGuidance: "Upstream circuit is open; retry after backoff."` (added in 0.15.0; dispatched via the `CIRCUIT_BREAKER_OPEN` code constant)
- On failure: returns hardcoded GENERAL_INTERNAL_ERROR dict (never raises)

### Properties
- async: false
- thread_safe: true
- pure: false
- idempotent: true

---

## Contract: ErrorMapper._handle_apcore_error

### Inputs
- error: Exception, required, validates[duck-typed: has `code`, `message`, `details` attributes], reject_with=INTERNAL_ERROR dict

### Errors
- Never raises; internal errors are converted to sanitized responses

### Returns
- On success: dict with keys `isError: True`, `errorType: str` (from error.code), `message: str`, `details: dict|None`
- CALL_DEPTH_EXCEEDED, CIRCULAR_CALL, CALL_FREQUENCY_EXCEEDED → sanitized "Internal error occurred" message
- ACL_DENIED → sanitized "Access denied", details: null
- SCHEMA_VALIDATION_ERROR → multi-line bulleted message with field-level errors; details preserved
- APPROVAL_PENDING → details narrowed to `{"approvalId": ...}` (camelCase)
- APPROVAL_TIMEOUT → `retryable: true`
- AI guidance fields (retryable, aiGuidance, userFixable, suggestion) attached from error attributes when present and not already set
- On failure: returns safe fallback dict (never raises)

### Properties
- async: false
- thread_safe: true
- pure: false
- idempotent: true

---

## Contract: ErrorMapper.internal_error_response

EM-6 generic-error fallback. Python and Rust expose this as a free helper alongside the class; TypeScript inlines the same envelope inside `toMcpError(error)` for any non-`ModuleError` input. All three SDKs MUST emit byte-identical envelopes to keep MCP clients' `errorType === "GENERAL_INTERNAL_ERROR"` branch portable.

### Inputs
- (none)

### Errors
- Never raises.

### Returns
- On success (Python): `dict` with keys `isError: True`, `errorType: "GENERAL_INTERNAL_ERROR"`, `message: "Internal error occurred"`, `details: None`
- On success (Rust): `serde_json::Value` (object) with the same keys/values
- On success (TS): equivalent object inlined inside `toMcpError(error)` for non-`ModuleError` inputs
- camelCase wire keys; `errorType` is the exact string `"GENERAL_INTERNAL_ERROR"`; `details` is JSON null

### Properties
- async: false
- thread_safe: true
- pure: true
- idempotent: true

---

## Contract: ErrorMapper.to_mcp_error_any

EM-6 generic-error fallback for arbitrary error inputs. Python signature `to_mcp_error_any(error: Exception)` and Rust signature `ErrorMapper::to_mcp_error_any<E: std::error::Error>(error: E)` both accept any error-like value and return the `internal_error_response()` envelope unchanged — they NEVER inspect the error's contents. TypeScript covers the same surface inside `toMcpError(error: unknown)` when the input fails the `ModuleError` shape check.

### Inputs
- error: any error-like value (Python `Exception`, Rust any `std::error::Error`, TS `unknown`) — required, no validation; the function deliberately ignores the contents to avoid leaking server-side detail

### Errors
- Never raises.

### Returns
- On success: identical envelope to `internal_error_response()` — `isError: true`, `errorType: "GENERAL_INTERNAL_ERROR"`, `message: "Internal error occurred"`, `details: null`
- The original error's class name, message, traceback, or details are NEVER surfaced on the wire (security: avoid leaking internal state to MCP clients)

### Properties
- async: false
- thread_safe: true
- pure: true
- idempotent: true

---

## Contract: CIRCUIT_BREAKER_OPEN error-code constant

> Public error-code constant introduced in 0.15.0. Exposed alongside the existing apcore error codes so SDK callers can branch on `error.errorType === "CIRCUIT_BREAKER_OPEN"` portably across all three languages. Names per language: Python `ERROR_CODES["CIRCUIT_BREAKER_OPEN"]`, TypeScript `ErrorCodes.CIRCUIT_BREAKER_OPEN`, Rust `ApcoreErrorCode::CircuitBreakerOpen` (serializes to the literal string `"CIRCUIT_BREAKER_OPEN"` for wire output).

### Inputs
- (none — constant)

### Errors
- (none)

### Returns
- Python: `str` literal `"CIRCUIT_BREAKER_OPEN"`
- TypeScript: `const` literal `"CIRCUIT_BREAKER_OPEN"` (typed as `ErrorCodes.CIRCUIT_BREAKER_OPEN`)
- Rust: enum variant `ApcoreErrorCode::CircuitBreakerOpen`; `Display` / serde implementation MUST emit the exact wire string `"CIRCUIT_BREAKER_OPEN"`

### Properties
- async: false
- thread_safe: true
- pure: true
- idempotent: true

### Wire format
The `errorType` field of an `to_mcp_error` envelope dispatched from a `CircuitBreakerOpenError` MUST equal this literal string. MCP clients branch on `response.errorType === "CIRCUIT_BREAKER_OPEN"` to apply backoff-and-retry logic before falling through to generic retryable handling.
