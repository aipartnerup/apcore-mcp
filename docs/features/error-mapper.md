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
- error: Exception, required — any Python exception; duck-typed for ModuleError-like (must have `code`, `message`, `details` attributes for apcore path)

### Errors
- Never raises; all exceptions produce a valid error dict

### Returns
- On success: dict with camelCase top-level keys — `isError: true`, `errorType: str`, `message: str`, `details: dict|None`
- `isError` and `errorType` are camelCase at the wire level; Python dict uses snake_case keys (`is_error`, `error_type`) internally but MCP response layer converts to camelCase
- Additional optional fields: `retryable: bool`, `aiGuidance: str`, `userFixable: bool`, `suggestion: str` — populated from error attributes when present
- `userFixable: true` for codes: DEPENDENCY_NOT_FOUND, DEPENDENCY_VERSION_MISMATCH, VERSION_CONSTRAINT_INVALID, BINDING_* error codes
- Non-ModuleError exceptions → `error_type: "INTERNAL_ERROR"`, `message: "Internal error occurred"`, `details: null`
- ExecutionCancelledError → `error_type: "EXECUTION_CANCELLED"`, `retryable: true`
- On failure: returns hardcoded INTERNAL_ERROR dict (never raises)

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
- On success: dict with keys `is_error: True`, `error_type: str` (from error.code), `message: str`, `details: dict|None`
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
