# Product Requirements Document: apcore-mcp

| Field       | Value                                                        |
|-------------|--------------------------------------------------------------|
| Title       | apcore-mcp: Automatic MCP Server & OpenAI Tools Bridge       |
| Version     | 1.3                                                          |
| Date        | 2026-02-27                                                   |
| Author      | aipartnerup Product Team                                     |
| Status      | Draft                                                        |
| Reviewers   | apcore Core Maintainers, Community Contributors              |
| License     | Apache 2.0                                                   |

---

## Background

apcore-mcp originated from the observation that apcore modules already carry all the metadata AI agent protocols need — `input_schema`, `output_schema`, `description`, and `annotations` — yet there was no standard way to expose them to MCP clients or OpenAI Function Calling. The idea was validated through competitive analysis (5+ manually-built ComfyUI MCP servers, each duplicating schema work) and demand research (MCP ecosystem growth in 2025-2026, OpenAI Function Calling as the dominant tool-use format). The core insight: since the mapping from apcore metadata to both MCP and OpenAI formats is nearly 1:1, a single adapter package can eliminate all manual tool definition work for the entire apcore ecosystem. This PRD formalizes that validated idea into actionable requirements.

> For the original brainstorming and validation notes, see [`ideas/apcore-mcp.md`](../ideas/apcore-mcp.md).

---

## 1. Executive Summary

**apcore-mcp** is an independent Python adapter package that automatically bridges any apcore Module Registry into both a fully functional MCP (Model Context Protocol) Server and OpenAI-compatible tool definitions. Instead of requiring developers to hand-write MCP tool definitions or OpenAI function schemas for every module, apcore-mcp reads the existing apcore metadata -- `input_schema`, `output_schema`, `description`, and `annotations` -- and generates correct tool definitions at runtime with zero manual effort. The package exposes two primary entry points: `serve(registry)` to launch a standards-compliant MCP Server (supporting stdio, Streamable HTTP, and SSE transports), and `to_openai_tools(registry)` to export an OpenAI-compatible tool list for function calling. By eliminating the manual mapping layer, apcore-mcp reduces tool integration time from approximately 2 hours per module to zero seconds, validates apcore's core claim of "inherent AI compatibility," and serves as the infrastructure layer that gives every `xxx-apcore` project (comfyui-apcore, vnpy-apcore, blender-apcore, etc.) instant MCP and OpenAI capability. This package is the critical multiplier for the apcore ecosystem's adoption in the AI agent landscape.

**Key business metrics impacted:** Developer adoption rate across the apcore ecosystem, time-to-integration for MCP/OpenAI tool exposure, and number of downstream `xxx-apcore` projects with AI agent interoperability.

---

## 2. Problem Statement

### 2.1 Current Pain Point

apcore modules are designed to be schema-driven and AI-perceivable. Every module carries `input_schema` (JSON Schema via Pydantic), `output_schema`, a human-readable `description`, and behavioral `annotations` (readonly, destructive, idempotent, requires_approval, open_world). This metadata is exactly what AI agent protocols -- MCP and OpenAI Function Calling -- require to define callable tools.

However, there is currently **no standard way to expose apcore modules to AI systems**. A developer who wants to make their ComfyUI workflows available to Claude Desktop via MCP must:

1. Manually create an MCP Server from scratch.
2. For each apcore module, hand-write a corresponding MCP tool definition -- duplicating the `input_schema` as `inputSchema`, the `description`, and mapping each annotation to MCP tool annotations.
3. Write execution routing logic that accepts MCP tool calls and dispatches them to the apcore `Executor`.
4. Handle error mapping from apcore's error hierarchy (`SchemaValidationError`, `ACLDeniedError`, `ModuleNotFoundError`, etc.) to MCP error responses.
5. Repeat an equivalent process if they also want OpenAI function calling support.

**Concrete example:** The ComfyUI MCP ecosystem already has 5+ independent server projects, each manually defining 10-50+ tool schemas. Every one of those projects performs the same tedious schema transcription that apcore has already automated at the module level.

### 2.2 Impact of Not Solving

- **Developer friction:** Each module requires approximately 2 hours of manual tool definition work (schema transcription, annotation mapping, error handling, testing). A registry with 20 modules means 40 hours of boilerplate.
- **Error-prone duplication:** Manual schema copying leads to drift between the authoritative apcore schema and the tool definition. A field renamed in the module's `input_schema` but not updated in the MCP tool definition causes silent failures.
- **Ecosystem fragmentation:** Without a standard bridge, each `xxx-apcore` project will build its own ad-hoc MCP integration. The community fragments into incompatible implementations.
- **Unproven core claim:** apcore's value proposition -- "modules are inherently AI-compatible" -- remains theoretical without executable proof. apcore-mcp is that proof.

### 2.3 "What Happens If We Don't Build This?" Analysis

If apcore-mcp is not built:

1. **apcore's AI compatibility claim stays marketing, not engineering.** Competitors can dismiss it as vaporware.
2. **Each xxx-apcore project must independently solve the MCP/OpenAI bridging problem**, leading to 5-10 incompatible implementations within 12 months, each with its own bugs and maintenance burden.
3. **Developer adoption of apcore stalls.** In a 2025-2026 market where MCP support is a table-stakes feature for developer tooling, modules that cannot be called by AI agents are perceived as legacy.
4. **The window of opportunity closes.** The MCP ecosystem is rapidly maturing. First-mover adapter packages will capture developer mindshare. Delaying 6 months means competing against entrenched alternatives.

---

## 3. Target Users & Personas

### 3.1 Primary Persona: Module Developer ("Maya")

| Attribute       | Detail                                                                 |
|-----------------|------------------------------------------------------------------------|
| Name            | Maya                                                                   |
| Role            | apcore module developer building domain-specific extensions            |
| Experience      | 3+ years Python, familiar with Pydantic schemas, new to MCP           |
| Pain Points     | Wants AI agents to call her modules but does not want to learn MCP protocol details; spends hours writing boilerplate tool definitions |
| Goals           | Expose her module registry to Claude Desktop with a single function call; validate that her schemas work correctly in AI contexts |
| Success Metric  | Zero lines of MCP-specific code written; modules callable by Claude within 5 minutes of installing apcore-mcp |

**User journey (before):**
1. Maya writes an apcore module with `input_schema`, `output_schema`, `description`.
2. Maya wants Claude Desktop to use her module.
3. Maya reads MCP documentation (2+ hours).
4. Maya hand-writes an MCP Server, tool definitions, and execution routing (~4 hours for 5 modules).
5. Maya discovers schema drift after a module update. Debugging takes 1+ hour.
6. Maya gives up on OpenAI support because it requires a separate conversion.

**User journey (after):**
1. Maya writes an apcore module (same as before).
2. Maya runs `pip install apcore-mcp`.
3. Maya adds 3 lines of code: import, create registry, call `serve(registry)`.
4. Claude Desktop connects. All modules are available as tools. Annotations are preserved.
5. Maya also calls `to_openai_tools(registry)` to get OpenAI-compatible definitions for her chatbot.
6. When Maya updates a module schema, the MCP tools automatically reflect the change on next restart.

### 3.2 Secondary Persona: xxx-apcore Project Developer ("David")

| Attribute       | Detail                                                                 |
|-----------------|------------------------------------------------------------------------|
| Name            | David                                                                  |
| Role            | Developer building comfyui-apcore (or vnpy-apcore, blender-apcore)    |
| Experience      | 5+ years Python, expert in domain (ComfyUI/VNPy/Blender), familiar with apcore |
| Pain Points     | Needs MCP support for his project but doesn't want to maintain a custom MCP server alongside domain logic |
| Goals           | Add MCP capability to his project by adding apcore-mcp as a dependency; no custom MCP code |
| Success Metric  | All domain modules exposed via MCP with zero MCP-specific code in his project |

### 3.3 Tertiary Persona: AI Agent Builder ("Alex")

| Attribute       | Detail                                                                 |
|-----------------|------------------------------------------------------------------------|
| Name            | Alex                                                                   |
| Role            | AI agent developer integrating external tools into Claude/GPT workflows |
| Experience      | Familiar with MCP clients and OpenAI function calling; not an apcore expert |
| Pain Points     | Needs reliable, well-documented tool servers; tired of hand-crafted MCP servers with incomplete schemas |
| Goals           | Connect to an apcore-mcp server and have all tools "just work" with correct schemas, descriptions, and safety annotations |
| Success Metric  | Tools discovered automatically; input validation errors are clear; destructive operations are flagged |

---

## 4. Market Context

### 4.1 Market Landscape

The AI agent tooling ecosystem is experiencing rapid growth in 2025-2026:

- **MCP (Model Context Protocol):** Introduced by Anthropic, MCP has become the de facto standard for exposing tools to AI assistants. Claude Desktop, Cursor, Windsurf, and a growing list of IDEs and agent frameworks support MCP. The MCP Python SDK is mature and actively maintained.
- **OpenAI Function Calling:** Since GPT-4's launch, OpenAI's function calling (now "tools") format has been adopted by virtually every LLM provider (Google Gemini, Anthropic Claude API, Mistral, Cohere, etc.) as the standard for structured tool use. It is the most widely deployed tool-use format globally.
- **Agent Frameworks:** LangChain, CrewAI, AutoGen, and others are building ecosystems around tool use, all consuming either MCP or OpenAI-format tool definitions.

The market demand is clear: developers need their software callable by AI agents, and the two dominant formats are MCP (for desktop/IDE integration) and OpenAI tools (for API-based agent systems).

### 4.2 Competitive Analysis

| Solution | Approach | Auto-discovery | Schema Reuse | Annotation Support | Dual Format (MCP + OpenAI) | Maintenance Burden |
|----------|----------|:-:|:-:|:-:|:-:|---|
| **Manual MCP Server** (e.g., comfyui-mcp projects) | Hand-write tool definitions per endpoint | No | No | No | No | High -- every schema change requires manual update |
| **FastMCP** (mcp SDK helpers) | Decorator-based tool definition | No -- still requires per-function decorators | No -- schemas defined inline | Partial -- manual annotation | No | Medium -- decorators reduce boilerplate but each tool still needs explicit definition |
| **LangChain Tool Wrappers** | Wrap functions as LangChain tools | No | No | No | Partial -- LangChain converts to OpenAI format | Medium -- tied to LangChain ecosystem |
| **Custom schema-to-MCP scripts** | One-off scripts that read JSON Schema and emit MCP | Partial | Partial | No | No | High -- scripts are project-specific |
| **apcore-mcp (this project)** | Automatic bridge from apcore Registry | **Yes** -- reads Registry at runtime | **Yes** -- 1:1 mapping from apcore schemas | **Yes** -- full annotation mapping | **Yes** -- single package | **Minimal** -- zero per-module code |

### 4.3 Differentiation Summary

apcore-mcp's unique advantage is the **zero-configuration, zero-code bridge** from an existing schema-rich registry to both MCP and OpenAI formats. Every alternative requires per-tool definition work. apcore-mcp requires none because apcore modules already carry all the metadata both protocols need. This is not a generic MCP framework -- it is a purpose-built adapter that leverages apcore's schema system to eliminate the adapter-writing problem entirely.

---

## 5. Product Vision & Strategy

### 5.1 Vision Statement

Any apcore module, in any domain, is instantly callable by any AI agent -- via MCP or OpenAI -- with zero additional code. apcore-mcp is the invisible bridge that makes apcore's "inherently AI-compatible" promise real.

### 5.2 Strategic Alignment with apcore Ecosystem

apcore-mcp fits precisely into the architecture defined by apcore's own SCOPE.md, which explicitly designates MCP/A2A adapters as independent projects:

```
apcore-python (core SDK)
    |
    +-- Provides: Registry, Executor, ModuleDescriptor, ModuleAnnotations, error hierarchy
    |
apcore-mcp (this project)
    |
    +-- Consumes: Registry API, Executor.call() / call_async()
    +-- Produces: MCP Server (via mcp SDK), OpenAI tools list (pure dict conversion)
    |
    +-- MCP path --> Claude Desktop, Cursor, Windsurf, any MCP client
    +-- OpenAI path --> OpenAI API, Azure OpenAI, any OpenAI-compatible platform
```

apcore-mcp is the **first adapter** in the planned family (apcore-a2a is future). Its success validates the adapter architecture pattern and creates a template for future protocol adapters.

### 5.3 Success Metrics (KPIs)

| KPI | Target | Measurement Method | Timeline |
|-----|--------|-------------------|----------|
| **Schema mapping accuracy** | 100% of apcore module fields correctly mapped to MCP and OpenAI formats | Automated test suite with full field coverage | At launch |
| **Integration time** | < 5 minutes from `pip install` to working MCP server | Timed user test with documented quickstart | At launch |
| **Transport coverage** | All 3 transports (stdio, Streamable HTTP, SSE) functional | Integration tests per transport | At launch |
| **Annotation preservation rate** | 100% of apcore annotations correctly mapped to MCP tool annotations | Unit tests for each annotation field | At launch |
| **Downstream adoption** | >= 1 xxx-apcore project (comfyui-apcore) using apcore-mcp for its MCP support | GitHub dependency tracking | Within 4 weeks of launch |
| **Error mapping coverage** | 100% of apcore error types mapped to appropriate MCP error codes | Unit tests for each error type | At launch |
| **MCP client compatibility** | Verified working with Claude Desktop and at least 1 additional MCP client | Manual integration testing | At launch |
| **Test coverage** | >= 90% line coverage on core logic | pytest-cov report | At launch |
| **Package size** | Core logic <= 1,200 lines (excluding tests and docs) | `cloc` measurement | At launch |

---

## 6. Feature Requirements

### P0 -- Must Have (Launch Blockers)

---

#### F-001: Registry-to-MCP Schema Mapping

**Title:** Automatic conversion of apcore Module schemas to MCP Tool definitions

**Description:** Given an apcore `Registry` with registered modules, apcore-mcp iterates over all modules, calls `registry.get_definition(module_id)` to obtain each `ModuleDescriptor`, and converts it into an MCP Tool definition. The mapping is:

| apcore ModuleDescriptor field | MCP Tool field |
|-------------------------------|----------------|
| `module_id`                   | `name`         |
| `description`                 | `description`  |
| `input_schema` (JSON Schema dict) | `inputSchema` |

**User Story:** As an apcore module developer, I want my module's `input_schema` and `description` to automatically appear as an MCP tool definition, so that I never have to write MCP-specific schema code.

**Acceptance Criteria:**
1. Every module registered in the Registry produces exactly one MCP Tool definition.
2. The MCP tool `name` equals the apcore `module_id`.
3. The MCP tool `description` equals the apcore module `description`.
4. The MCP tool `inputSchema` is a valid JSON Schema object equal to the apcore module's `input_schema`.
5. Modules with empty `input_schema` (`{}`) produce an MCP tool with `inputSchema: {"type": "object", "properties": {}}`.
6. Modules with complex/nested `input_schema` (nested objects, arrays, `$defs`) are correctly represented.

**Priority:** P0

---

#### F-002: Annotation-to-MCP Mapping

**Title:** Automatic conversion of apcore ModuleAnnotations to MCP Tool annotations

**Description:** apcore `ModuleAnnotations` fields map to MCP tool annotations as follows:

| apcore annotation       | MCP tool annotation   |
|--------------------------|-----------------------|
| `destructive`            | `destructiveHint`     |
| `readonly`               | `readOnlyHint`        |
| `idempotent`             | `idempotentHint`      |
| `open_world`             | `openWorldHint`       |

The `requires_approval` annotation is not a native MCP annotation but is preserved in a way that MCP clients can consume (e.g., as a custom annotation or via the confirmation flow).

**User Story:** As an AI agent builder, I want MCP tool annotations to accurately reflect the module's behavioral characteristics (destructive, read-only, etc.), so that my AI client can make safe decisions about tool invocation.

**Acceptance Criteria:**
1. An apcore module with `annotations=ModuleAnnotations(readonly=True)` produces an MCP tool with `readOnlyHint=True`.
2. An apcore module with `annotations=ModuleAnnotations(destructive=True)` produces an MCP tool with `destructiveHint=True`.
3. An apcore module with `annotations=ModuleAnnotations(idempotent=True)` produces an MCP tool with `idempotentHint=True`.
4. An apcore module with `annotations=ModuleAnnotations(open_world=True)` produces an MCP tool with `openWorldHint=True`.
5. An apcore module with `annotations=None` (no annotations) produces an MCP tool with default annotation values.
6. The `requires_approval` flag is preserved and accessible to MCP clients.

**Priority:** P0

---

#### F-003: MCP Execution Routing

**Title:** Route MCP tool calls through apcore Executor pipeline

**Description:** When an MCP client invokes a tool, apcore-mcp receives the tool name (= apcore `module_id`) and arguments (= inputs dict), then routes the call through `Executor.call(module_id, inputs)` (or `Executor.call_async(module_id, inputs)` in async contexts). The full apcore pipeline -- ACL enforcement, input validation, middleware before/after, timeout enforcement, output validation -- is executed. The module output dict is returned as the MCP tool result.

**User Story:** As an apcore module developer, I want MCP tool calls to go through the full apcore Executor pipeline (including ACL, validation, and middleware), so that my security and quality guarantees are preserved regardless of how the module is invoked.

**Acceptance Criteria:**
1. A tool call with valid inputs returns the module's output as the MCP result content.
2. A tool call to a non-existent module returns an MCP error with `isError=true` and a message containing "Module not found".
3. A tool call with invalid inputs (schema validation failure) returns an MCP error with details about which fields failed validation.
4. A tool call denied by ACL returns an MCP error indicating access denied.
5. A tool call that exceeds the executor timeout returns an MCP error indicating timeout.
6. Middleware before/after hooks are executed for MCP-originated calls.
7. Both sync and async module `execute()` methods are supported.

**Priority:** P0

---

#### F-004: MCP Error Mapping

**Title:** Map apcore error hierarchy to MCP error responses

**Description:** apcore has a rich error hierarchy (see `apcore.errors`). Each error type must be mapped to an appropriate MCP error response with `isError=true`, a human-readable error message, and structured details.

| apcore Error                 | MCP Error Behavior                                      |
|------------------------------|---------------------------------------------------------|
| `ModuleNotFoundError`        | Tool not found error                                    |
| `SchemaValidationError`      | Invalid params error with field-level details           |
| `ACLDeniedError`             | Permission denied error                                 |
| `ModuleTimeoutError`         | Timeout error with duration                             |
| `InvalidInputError`          | Invalid input error                                     |
| `CallDepthExceededError`     | Internal error (safety limit)                           |
| `CircularCallError`          | Internal error (safety limit)                           |
| `CallFrequencyExceededError` | Internal error (safety limit)                           |
| Any unexpected `Exception`   | Internal error with sanitized message                   |

**User Story:** As an AI agent builder, I want MCP error responses to contain clear, structured information about what went wrong, so that my AI client can recover gracefully or inform the user.

**Acceptance Criteria:**
1. Each apcore error type listed above produces a distinct, identifiable MCP error response.
2. `SchemaValidationError` responses include field-level error details (field name, error code, message).
3. `ACLDeniedError` responses do not leak sensitive security information (no caller_id exposure unless explicitly configured).
4. Unexpected exceptions produce a generic "Internal error" message without leaking stack traces to the MCP client.
5. All error responses set `isError=true` in the MCP result.

**Priority:** P0

---

#### F-005: `serve()` Function -- MCP Server Entry Point

**Title:** One-function MCP Server launch

**Description:** The `serve(registry, ...)` function is the primary entry point for launching an MCP Server. It accepts an apcore `Registry` (or `Executor`) and optional configuration, creates an MCP Server instance using the official `mcp` Python SDK, registers all modules as tools, and starts serving.

```python
from apcore import Registry
from apcore_mcp import serve

registry = Registry(extensions_dir="./extensions")
registry.discover()

# Simplest usage -- stdio transport (default)
serve(registry)
```

**User Story:** As an apcore module developer, I want to start a fully functional MCP Server with a single function call, so that I can expose my modules to AI agents without learning MCP internals.

**Acceptance Criteria:**
1. `serve(registry)` starts an MCP Server using stdio transport by default.
2. `serve(registry, transport="streamable-http", host="127.0.0.1", port=8000)` starts an MCP Server with Streamable HTTP transport.
3. `serve(registry, transport="sse", host="127.0.0.1", port=8000)` starts an MCP Server with SSE transport.
4. The server name and version can be optionally configured via parameters.
5. Passing an `Executor` instance instead of a `Registry` is supported (allowing users to pre-configure ACL, middleware, and timeouts).
6. The function blocks until the server is shut down (consistent with MCP SDK behavior).
7. Calling `serve()` with an empty registry (zero modules) starts a server with zero tools and logs a warning.

**Priority:** P0

---

#### F-006: stdio Transport Support

**Title:** MCP Server over stdio transport

**Description:** stdio is the default MCP transport used by Claude Desktop and most MCP clients. apcore-mcp must support launching the MCP Server with stdio transport, where the server reads from stdin and writes to stdout.

**User Story:** As an AI agent builder, I want to configure Claude Desktop to launch the apcore-mcp server via stdio, so that my apcore modules appear as tools in Claude Desktop.

**Acceptance Criteria:**
1. `serve(registry)` (default) uses stdio transport.
2. The server can be configured in Claude Desktop's `claude_desktop_config.json` as a stdio MCP server.
3. Tools are listed and callable from Claude Desktop.
4. The server handles graceful shutdown when the parent process terminates.

**Priority:** P0

---

#### F-007: Streamable HTTP Transport Support

**Title:** MCP Server over Streamable HTTP transport

**Description:** Streamable HTTP is the recommended network transport for MCP servers that need to be accessed over HTTP. apcore-mcp must support this transport with configurable host and port.

**User Story:** As a developer deploying an MCP server to a remote machine, I want to serve over HTTP so that MCP clients can connect over the network.

**Acceptance Criteria:**
1. `serve(registry, transport="streamable-http", host="0.0.0.0", port=8000)` starts an HTTP-based MCP Server.
2. The server responds to MCP protocol requests over HTTP.
3. The default host is `"127.0.0.1"` and the default port is `8000`.
4. The server handles concurrent connections correctly.

**Priority:** P0

---

#### F-008: `to_openai_tools()` Function -- OpenAI Tools Export

**Title:** Export apcore Registry as OpenAI-compatible tool definitions

**Description:** The `to_openai_tools(registry)` function iterates over all modules in a Registry, converts each to an OpenAI-compatible tool definition dict, and returns a list. The conversion is pure data transformation with no server or SDK dependency.

```python
from apcore import Registry
from apcore_mcp import to_openai_tools

registry = Registry(extensions_dir="./extensions")
registry.discover()

tools = to_openai_tools(registry)
# Returns:
# [
#   {
#     "type": "function",
#     "function": {
#       "name": "image.resize",
#       "description": "Resize an image to the specified dimensions",
#       "parameters": { ... JSON Schema from input_schema ... }
#     }
#   },
#   ...
# ]
```

**User Story:** As an AI agent builder using the OpenAI API, I want to get a list of tool definitions from my apcore Registry that I can pass directly to `openai.chat.completions.create(tools=...)`, so that I can use apcore modules as OpenAI function calls.

**Acceptance Criteria:**
1. `to_openai_tools(registry)` returns a `list[dict]`.
2. Each dict has `"type": "function"` at the top level.
3. Each dict has a `"function"` key containing `"name"`, `"description"`, and `"parameters"`.
4. The `"name"` equals the apcore `module_id`.
5. The `"description"` equals the apcore module `description`.
6. The `"parameters"` is a valid JSON Schema object equal to the apcore module's `input_schema`.
7. The returned list is directly usable with `openai.chat.completions.create(tools=tools)`.
8. An empty registry returns an empty list `[]`.
9. The function has no dependency on the `openai` package (returns plain dicts).

**Priority:** P0

---

#### F-009: CLI Entry Point

**Title:** Command-line interface for launching apcore-mcp server

**Description:** Provide a `python -m apcore_mcp` (or `apcore-mcp`) CLI command that discovers modules from a specified directory and starts the MCP Server. This enables stdio-based usage in MCP client configurations without writing any Python code.

```bash
# Simplest usage
python -m apcore_mcp --extensions-dir ./extensions

# With Streamable HTTP transport
python -m apcore_mcp --extensions-dir ./extensions --transport streamable-http --port 8000

# With SSE transport
python -m apcore_mcp --extensions-dir ./extensions --transport sse --port 8000
```

**User Story:** As an AI agent builder configuring Claude Desktop, I want to point to a command-line executable for the MCP server, so that I can configure it in `claude_desktop_config.json` without writing a Python script.

**Acceptance Criteria:**
1. `python -m apcore_mcp --extensions-dir ./extensions` starts an MCP server over stdio.
2. The `--transport` flag accepts `stdio`, `streamable-http`, and `sse`.
3. The `--host` and `--port` flags configure network transports.
4. The `--help` flag displays usage information.
5. If `--extensions-dir` points to a non-existent directory, the CLI exits with a clear error message and non-zero exit code.
6. If no modules are discovered, the CLI logs a warning and starts with zero tools.

**Priority:** P0

---

### P1 -- Should Have (Important for User Satisfaction)

---

#### F-010: SSE Transport Support (Backward Compatibility)

**Title:** MCP Server over Server-Sent Events transport

**Description:** While SSE transport is deprecated in the MCP SDK in favor of Streamable HTTP, many existing MCP clients still only support SSE. apcore-mcp should support SSE for backward compatibility.

**User Story:** As a developer using an older MCP client that only supports SSE, I want apcore-mcp to serve over SSE transport, so that I can still connect.

**Acceptance Criteria:**
1. `serve(registry, transport="sse", host="0.0.0.0", port=8000)` starts an SSE-based MCP Server.
2. The SSE transport is marked as deprecated in documentation with a recommendation to use Streamable HTTP.
3. The SSE server correctly handles the SSE protocol for tool listing and invocation.

**Priority:** P1

---

#### F-011: OpenAI Annotation Embedding

**Title:** Optionally embed apcore annotations in OpenAI tool descriptions

**Description:** OpenAI's tool format does not have native annotation support. apcore-mcp should optionally embed annotation information (destructive, readonly, requires_approval, etc.) in the tool description string so that safety-aware applications can parse it.

```python
tools = to_openai_tools(registry, embed_annotations=True)
# Tool description becomes:
# "Resize an image to the specified dimensions\n\n[Annotations: destructive=false, readonly=false, idempotent=true]"
```

**User Story:** As an AI agent builder, I want apcore safety annotations to be visible in OpenAI tool descriptions, so that my application can make informed decisions about tool invocation (e.g., requiring human confirmation for destructive operations).

**Acceptance Criteria:**
1. `to_openai_tools(registry, embed_annotations=True)` appends annotation metadata to each tool's description string.
2. `to_openai_tools(registry, embed_annotations=False)` (default) does not modify descriptions.
3. The annotation format is parseable (structured text, not free-form prose).
4. Only non-default annotation values are included to keep descriptions concise.

**Priority:** P1

---

#### F-012: OpenAI Strict Mode Support

**Title:** Support OpenAI Structured Outputs (`strict: true`) in tool definitions

**Description:** OpenAI's Structured Outputs feature allows setting `"strict": true` on tool definitions, which constrains the model to only produce outputs matching the schema exactly. apcore-mcp should support generating strict-mode-compatible tool definitions.

```python
tools = to_openai_tools(registry, strict=True)
# Each tool includes "strict": true in the function definition
```

**User Story:** As an AI agent builder, I want to use OpenAI's strict mode for tool calls, so that the model always produces valid inputs matching my apcore module schemas.

**Acceptance Criteria:**
1. `to_openai_tools(registry, strict=True)` adds `"strict": true` to each function definition.
2. `to_openai_tools(registry, strict=False)` (default) does not add the strict field.
3. When strict mode is enabled, schemas are validated to be strict-mode-compatible (e.g., all properties required, no `additionalProperties`). Incompatible schemas produce a warning.

**Priority:** P1

---

#### F-013: Structured Output Responses

**Title:** Return structured MCP tool results using apcore `output_schema`

**Description:** apcore modules have an `output_schema` that describes the structure of their return value. When an MCP tool call succeeds, apcore-mcp should return the result in a structured format, optionally utilizing the MCP SDK's structured content features if available, and always including the raw output as JSON text content.

**User Story:** As an AI agent builder, I want MCP tool results to include structured data, so that my AI client can parse and use the output programmatically.

**Acceptance Criteria:**
1. Successful tool calls return the module output serialized as a JSON string in the MCP text content.
2. The output is valid JSON that conforms to the module's `output_schema`.
3. Output that includes non-serializable types (e.g., bytes) is handled gracefully with appropriate conversion or error.

**Priority:** P1

---

#### F-014: Executor Passthrough

**Title:** Accept pre-configured Executor instance

**Description:** Advanced users may want to pre-configure an `Executor` with custom ACL rules, middleware, and timeout settings. `serve()` and `to_openai_tools()` should accept either a `Registry` or an `Executor`.

```python
from apcore import Registry, Executor, ACL, ACLRule

registry = Registry(extensions_dir="./extensions")
registry.discover()

acl = ACL(default_policy="deny", rules=[ACLRule(...)])
executor = Executor(registry, acl=acl)

serve(executor)  # Uses the pre-configured executor
```

**User Story:** As an apcore module developer deploying to production, I want to pass a pre-configured Executor with ACL and middleware to the MCP server, so that my security policies are enforced on all tool calls.

**Acceptance Criteria:**
1. `serve(executor)` accepts an `Executor` instance and uses it for all tool call routing.
2. `serve(registry)` creates a default `Executor(registry)` internally.
3. When an `Executor` is passed, the MCP server uses `executor.call()` / `executor.call_async()` for execution.
4. `to_openai_tools(executor)` extracts the registry from the executor and generates tool definitions.

**Priority:** P1

---

#### F-015: Dynamic Tool Registration

**Title:** Reflect registry changes in MCP tool list at runtime

**Description:** When modules are registered or unregistered from the apcore Registry after the MCP server has started, the MCP tool list should update accordingly. This leverages apcore Registry's event system (`registry.on("register", callback)` and `registry.on("unregister", callback)`).

**User Story:** As an apcore module developer, I want to add or remove modules from the registry while the MCP server is running, so that I can hot-reload modules during development without restarting the server.

**Acceptance Criteria:**
1. Registering a new module via `registry.register(id, module)` after `serve()` has started adds the corresponding MCP tool.
2. Unregistering a module via `registry.unregister(id)` after `serve()` has started removes the corresponding MCP tool.
3. MCP clients are notified of tool list changes (via MCP's tool list changed notification if supported).
4. The tool list update is thread-safe.

> **Note -- `dynamic` parameter (Reserved for future implementation):** Both Python and TypeScript implementations of `serve()` accept a `dynamic` boolean parameter (default: `False` in Python, `false` in TypeScript). This parameter is intended to control whether the `RegistryListener` is activated for dynamic tool registration. However, in current releases, the parameter is accepted but **not yet wired** to the `RegistryListener` -- dynamic tool list updates based on Registry events are not yet functional. The parameter is reserved for future implementation and should not be relied upon for production use.

**Priority:** P1

---

#### F-016: Logging and Observability

**Title:** Structured logging for MCP server operations

**Description:** apcore-mcp should use Python's standard `logging` module to log tool calls, errors, and server lifecycle events. Logs should be structured and consistent with apcore's logging patterns.

**User Story:** As an apcore module developer, I want to see clear log output showing which tools are being called and any errors, so that I can debug issues during development.

**Acceptance Criteria:**
1. Server startup logs the number of tools registered and the transport type.
2. Each tool call logs the tool name at DEBUG level.
3. Tool call errors log the error type and message at ERROR level.
4. Log messages use the `apcore_mcp` logger namespace.
5. Log verbosity is controllable via standard Python logging configuration.

**Priority:** P1

---

### P2 -- Nice to Have (v2 Candidates)

---

#### F-017: `to_openai_tools()` with Filtering

**Title:** Filter modules when generating OpenAI tool definitions

**Description:** Allow users to filter which modules are included in the OpenAI tools output by tags or prefix.

```python
tools = to_openai_tools(registry, tags=["image"], prefix="comfyui.")
```

**User Story:** As an AI agent builder, I want to export only a subset of my registry as OpenAI tools, so that I can present a focused set of capabilities to the AI model.

**Acceptance Criteria:**
1. `to_openai_tools(registry, tags=["image"])` returns only tools from modules with the "image" tag.
2. `to_openai_tools(registry, prefix="comfyui.")` returns only tools from modules whose IDs start with "comfyui.".
3. Filters can be combined (tags AND prefix).
4. Passing no filters returns all modules (default behavior).

**Priority:** P2

---

#### F-018: `serve()` with Module Filtering

**Title:** Filter which modules are exposed as MCP tools

**Description:** Allow users to specify which modules from the registry should be exposed as MCP tools, using tags or prefix filters.

```python
serve(registry, tags=["public"], prefix="api.")
```

**User Story:** As an apcore module developer deploying to production, I want to expose only a subset of my modules via MCP, so that internal/admin modules are not accessible to AI agents.

**Acceptance Criteria:**
1. `serve(registry, tags=["public"])` exposes only modules with the "public" tag.
2. `serve(registry, prefix="api.")` exposes only modules whose IDs start with "api.".
3. Filters can be combined.
4. Filtered-out modules are not discoverable via the MCP tool list and calls to them return "tool not found".

**Priority:** P2

---

#### F-019: Health Check Endpoint

**Title:** HTTP health check for network transports

**Description:** When serving over Streamable HTTP or SSE, expose a `/health` endpoint that returns server status, number of registered tools, and uptime.

**User Story:** As a DevOps engineer, I want a health check endpoint for the MCP server, so that I can monitor its availability in production.

**Acceptance Criteria:**
1. `GET /health` returns HTTP 200 with a JSON body containing `status`, `module_count`, and `uptime_seconds`.
2. The endpoint is only available for HTTP-based transports (not stdio).
3. The endpoint does not require authentication.

**Priority:** P2

---

#### F-020: MCP Resource Exposure

**Title:** Expose apcore module documentation as MCP Resources

**Description:** MCP supports "Resources" in addition to Tools. apcore modules with `documentation` fields could be exposed as MCP Resources, allowing AI agents to read module documentation before invoking tools.

**User Story:** As an AI agent, I want to read the full documentation for a tool before calling it, so that I can use it correctly.

**Acceptance Criteria:**
1. Modules with non-empty `documentation` fields are exposed as MCP Resources.
2. Each resource is named `docs://{module_id}` and contains the documentation text.
3. Modules without documentation do not generate resources.

**Priority:** P2

---

#### F-021: Prometheus Metrics Endpoint

**Title:** `/metrics` endpoint for Prometheus scraping

**Description:** When serving over Streamable HTTP or SSE, expose a `/metrics` endpoint that returns Prometheus-format metrics (content type `text/plain; version=0.0.4; charset=utf-8`) when a `MetricsExporter` is configured. Returns HTTP 404 if no metrics exporter is configured.

**Acceptance Criteria:**
1. `GET /metrics` returns HTTP 200 with Prometheus text format when a `MetricsExporter` is configured.
2. `GET /metrics` returns HTTP 404 when no `MetricsExporter` is configured.
3. The endpoint is only available for HTTP-based transports (not stdio).

**Priority:** P2

---

#### F-022: Metrics Collector Parameter

**Title:** `metrics_collector` / `metricsCollector` parameter in `serve()`

**Description:** The `serve()` function accepts an optional `metrics_collector` (Python) / `metricsCollector` (TypeScript) parameter. When provided, the server collects per-tool-call metrics (call counts, error counts, durations) and exposes them via the `/metrics` endpoint (F-021).

**Acceptance Criteria:**
1. `serve(registry, metrics_collector=collector)` enables metrics collection for all tool calls.
2. When no `metrics_collector` is provided, metrics collection is disabled and `/metrics` returns 404.

**Priority:** P2

---

#### F-023: Input Validation Parameter

**Title:** `validate_inputs` parameter in `serve()` for pre-execution schema validation

**Description:** The `serve()` function accepts an optional `validate_inputs` boolean parameter (default: `False`). When enabled, the MCP layer performs JSON Schema validation on tool call inputs before routing to the Executor. This provides early validation feedback at the MCP layer, independent of the Executor's own schema validation middleware.

**Acceptance Criteria:**
1. `serve(registry, validate_inputs=True)` enables pre-execution input validation at the MCP layer.
2. `serve(registry, validate_inputs=False)` (default) delegates all validation to the Executor pipeline.
3. Validation errors are returned as MCP error responses with field-level details.

**Priority:** P2

---

#### F-024: Streaming and Progress Support

**Title:** `executor.stream()` for chunked responses with `notifications/progress`

**Description:** Support streaming/progress notifications for long-running tool calls. When a module execution uses the `executor.stream()` interface, the MCP server sends `notifications/progress` messages to the client with incremental updates, enabling real-time feedback for long-running operations.

**Acceptance Criteria:**
1. Long-running tool calls can emit progress notifications via `notifications/progress`.
2. The final result is delivered as a standard `CallToolResult` after streaming completes.

**Priority:** P2

---

#### F-025: MCPServer Background Wrapper (Python Only)

**Title:** `MCPServer` class for non-blocking server lifecycle management

**Description:** A Python-only `MCPServer` class that wraps `serve()` in a background thread, providing a non-blocking server lifecycle with `start()`, `stop()`, and `wait()` methods. This is useful for embedding an MCP server inside a larger application without blocking the main thread. TypeScript does not need an equivalent since `serve()` is already async and can be managed with standard async patterns.

**Acceptance Criteria:**
1. `MCPServer(registry, **serve_kwargs)` creates a server wrapper without starting it.
2. `server.start()` launches the MCP server in a background thread and returns immediately.
3. `server.stop()` signals the server to shut down gracefully.
4. `server.wait()` blocks until the server has fully stopped.
5. This class is Python-only; TypeScript relies on async `serve()` directly.

**Priority:** P2

#### F-026: MCP Tool Explorer (Optional Explorer UI)

**Title:** Built-in browser UI for inspecting and testing MCP tools

**Description:** An optional, lightweight web UI served alongside the MCP HTTP server that allows developers to browse registered tools, inspect their schemas, and execute them interactively. This is the MCP-layer equivalent of Swagger UI -- it shows exactly what AI clients see after protocol conversion (tool names, input schemas, annotations). The Explorer is disabled by default and only available with HTTP-based transports (Streamable HTTP, SSE). It is **not** a production dashboard; it is a developer debugging aid.

The key motivation is ecosystem scale: as the number of `xxx-apcore` projects grows across multiple languages and frameworks, having the Explorer at the MCP layer avoids N-framework x M-language reimplementation. Each `apcore-mcp-{lang}` implementation provides the Explorer once; all framework extensions (`flask-apcore`, `django-apcore`, `express-apcore`, `gin-apcore`, `spring-apcore`, etc.) benefit automatically.

**User Story:** As a developer building an `xxx-apcore` project, I want to visually inspect the MCP tools my modules expose and test them from a browser, so that I can verify schema mapping and execution without launching an MCP client like Claude Desktop.

**Acceptance Criteria:**
1. When `explorer=True` is passed to `serve()` (or `--explorer` on CLI), the Explorer UI is mounted at `explorer_prefix` (default `/explorer`), e.g., `http://localhost:8000/explorer/`.
2. `GET /explorer/` returns an interactive HTML page listing all registered tools.
3. `GET /explorer/tools` returns a JSON array of tool summaries (name, description, annotations).
4. `GET /explorer/tools/<name>` returns a JSON object with full tool detail including `inputSchema`.
5. `POST /explorer/tools/<name>/call` accepts a JSON body, executes the tool via the Executor pipeline, and returns the result.
6. The `POST /explorer/tools/<name>/call` endpoint respects `allow_execute` (default `False`); returns 403 when disabled.
7. The HTML UI is a single self-contained file (no external CDN dependencies) embedded in the package.
8. Explorer is disabled by default; enabling it requires explicit opt-in.
9. Explorer is only available for HTTP-based transports; ignored for stdio.
10. The HTML/JS UI is language-agnostic and can be shared across all `apcore-mcp-{lang}` implementations.
11. `explorer_prefix` is configurable (default `/explorer`); all Explorer endpoints are mounted under this prefix.
12. Explorer GET endpoints are exempt from JWT authentication. `POST /explorer/tools/<name>/call` enforces auth when authenticator is configured.

**Priority:** P2

---

#### F-027: JWT Authentication for HTTP Transports

**Title:** Optional JWT Bearer token authentication for HTTP-based MCP endpoints

**Description:** When serving over Streamable HTTP or SSE, the server can optionally require a valid JWT Bearer token on every request to `/mcp`. On successful validation, the JWT claims are mapped to an apcore `Identity` and injected into the `Context` passed to the Executor, enabling apcore's existing ACL conditions (`identity_types`, `roles`) to work with external callers. Authentication is pluggable via an `Authenticator` protocol; a built-in `JWTAuthenticator` implementation is provided. Health and metrics endpoints are exempt by default.

This feature bridges the gap between HTTP-level authentication and apcore's module-level authorization. Without it, `Context` is created without `Identity`, making ACL role/type-based conditions ineffective for external HTTP callers.

**User Story:** As a platform engineer deploying an MCP server over HTTP, I want to require JWT authentication so that only authorized clients can call tools, and I want the caller's identity to flow through to apcore's ACL system.

**Acceptance Criteria:**
1. `serve()` accepts an optional `authenticator` parameter. When provided with an HTTP transport, requests without a valid Bearer token receive HTTP 401 with `WWW-Authenticate: Bearer`.
2. `JWTAuthenticator(key, algorithms, audience, issuer, claim_mapping, require_claims)` validates JWT tokens. Token errors (expired, bad signature, missing claims) return `None`, never leak token content.
3. `ClaimMapping` (frozen dataclass) configures how JWT claims map to `Identity` fields: `id_claim="sub"`, `type_claim="type"`, `roles_claim="roles"`, `attrs_claims=None`.
4. On successful authentication, the resulting `Identity` is injected into the `Context` via a `ContextVar` bridge, making `context.identity` available to the Executor and ACL system.
5. `/health`, `/metrics`, and Explorer browsing endpoints (GET) are exempt from authentication by default. Exempt paths are configurable via `exempt_paths`/`exempt_prefixes` parameters.
6. `require_auth=False` (permissive mode) allows unauthenticated requests to proceed without identity, preserving backward compatibility.
7. Non-HTTP ASGI scopes (WebSocket, lifespan) pass through without authentication.
8. CLI provides `--jwt-secret`, `--jwt-algorithm`, `--jwt-audience`, `--jwt-issuer`, `--jwt-key-file` (Python), `--jwt-require-auth`, and `--exempt-paths` flags.
9. `Authenticator` is a `@runtime_checkable` Protocol, allowing custom authentication backends.
10. A JWT validation library is added as a required dependency (e.g., `PyJWT>=2.0` for Python, `jsonwebtoken` for TypeScript, `golang-jwt` for Go).

**Priority:** P2

---

**Feature Count Summary:**

| Priority | Count | Features |
|----------|-------|----------|
| P0       | 9     | F-001 through F-009 |
| P1       | 7     | F-010 through F-016 |
| P2       | 11    | F-017 through F-027 |
| **Total**| **27**|                      |

---

## 7. User Experience

### 7.1 Key User Workflows

#### Workflow 1: Quickstart -- MCP Server via stdio (3 lines of code)

```python
from apcore import Registry
from apcore_mcp import serve

registry = Registry(extensions_dir="./extensions")
registry.discover()
serve(registry)
```

**Expected behavior:** The MCP server starts, discovers all modules, registers them as MCP tools, and listens on stdin/stdout. A Claude Desktop configuration pointing to this script as a command will see all tools.

#### Workflow 2: MCP Server with ACL and HTTP transport

```python
from apcore import Registry, Executor, ACL, ACLRule
from apcore_mcp import serve

registry = Registry(extensions_dir="./extensions")
registry.discover()

acl = ACL(default_policy="deny", rules=[
    ACLRule(caller="*", target="image.*", policy="allow"),
])
executor = Executor(registry, acl=acl)

serve(executor, transport="streamable-http", host="0.0.0.0", port=8000)
```

**Expected behavior:** The MCP server starts on port 8000, only allows calls to modules with IDs starting with "image.", and denies all other calls with an ACL error.

#### Workflow 3: OpenAI Function Calling integration

```python
from openai import OpenAI
from apcore import Registry, Executor
from apcore_mcp import to_openai_tools

registry = Registry(extensions_dir="./extensions")
registry.discover()

tools = to_openai_tools(registry)
executor = Executor(registry)

client = OpenAI()
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Resize my image to 800x600"}],
    tools=tools,
)

# Handle tool calls
for tool_call in response.choices[0].message.tool_calls:
    result = executor.call(tool_call.function.name, json.loads(tool_call.function.arguments))
    # Send result back to the model...
```

**Expected behavior:** The OpenAI model sees all apcore modules as available tools, selects the appropriate one, and the developer routes the call through the apcore Executor.

#### Workflow 4: CLI-based setup for Claude Desktop

`claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "my-apcore-tools": {
      "command": "python",
      "args": ["-m", "apcore_mcp", "--extensions-dir", "/path/to/extensions"]
    }
  }
}
```

**Expected behavior:** Claude Desktop launches the apcore-mcp server as a subprocess, communicates via stdio, and presents all discovered modules as available tools.

### 7.2 API Design Principles

1. **Minimal surface area:** Two primary functions (`serve()`, `to_openai_tools()`), one CLI command. No unnecessary abstractions.
2. **Progressive disclosure:** The simplest usage is one function call. Advanced features (ACL, transport config, filtering) are opt-in via parameters.
3. **Accept both Registry and Executor:** Users who just want to serve modules pass a `Registry`. Users who need ACL/middleware pass a pre-configured `Executor`. The API accepts both.
4. **No surprises:** `serve()` blocks (consistent with server behavior). `to_openai_tools()` is a pure function that returns data (no side effects).
5. **Type-safe:** Full type annotations on all public APIs. Compatible with mypy and pyright.

### 7.3 Error Handling Philosophy

1. **User-facing errors are clear and actionable.** "Module 'image.resize' not found" is better than "KeyError: image.resize".
2. **apcore errors are translated, not swallowed.** Every apcore error type maps to a specific MCP error response. The original error information is preserved in structured form.
3. **Internal errors are sanitized.** Unexpected exceptions produce generic error messages to MCP clients. Full stack traces go to logs only.
4. **Fail fast on configuration errors.** Invalid transport names, unreachable ports, and non-existent extension directories raise exceptions at startup, not at first request.

---

## 8. Technical Considerations

### 8.1 High-Level Architecture

```
+-------------------------------+
|  User's Application / CLI     |
|  (3 lines of code)            |
+-------------------------------+
        |
        v
+-------------------------------+
|  apcore-mcp                   |
|  +-------------------------+  |
|  | serve()                 |  |   +---------------------------+
|  |   - Tool Registration   |------>|  MCP SDK (mcp package)   |
|  |   - Execution Routing   |  |   |  - stdio transport       |
|  |   - Error Mapping       |  |   |  - Streamable HTTP       |
|  +-------------------------+  |   |  - SSE                    |
|  | to_openai_tools()       |  |   +---------------------------+
|  |   - Schema Conversion   |  |
|  |   - Pure dict output    |  |
|  +-------------------------+  |
+-------------------------------+
        |
        v
+-------------------------------+
|  apcore-python SDK            |
|  - Registry (module discovery)|
|  - Executor (call pipeline)   |
|  - ModuleDescriptor (schemas) |
|  - ModuleAnnotations          |
|  - Error hierarchy            |
+-------------------------------+
```

### 8.2 Key Technical Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Use MCP SDK, not raw protocol | Use official `mcp` Python SDK | The MCP SDK handles protocol details, transport negotiation, and message serialization. Reimplementing these is out of scope and error-prone. |
| Async-first execution | Use `Executor.call_async()` for MCP tool handlers | MCP servers typically run in an async event loop. Using `call_async()` avoids blocking the event loop during module execution. |
| Pure dict output for OpenAI | `to_openai_tools()` returns `list[dict]` | No dependency on `openai` package. Users can pass the dicts directly to any OpenAI-compatible API client. Maximum interoperability. |
| Single package, not two | MCP + OpenAI in one package | The OpenAI conversion logic is approximately 20-50 lines. Splitting into a separate package adds packaging overhead without benefit. |
| Registry events for dynamic tools | Subscribe to `registry.on("register/unregister")` | Leverages apcore's existing event system. No polling or manual refresh needed. |
| JWT auth bridges to apcore ACL | Optional `JWTAuthenticator` + `AuthMiddleware` at HTTP layer, Identity flows into Executor ACL | HTTP callers need a way to establish `Identity` for apcore's ACL. JWT at the transport layer feeds into the existing ACL system rather than replacing it. Custom backends via `Authenticator` protocol. |

### 8.3 Dependencies and Risks

| Dependency | Version Constraint | Risk | Mitigation |
|------------|-------------------|------|------------|
| `apcore` (apcore-python) | >= 0.2.0 | API changes in pre-1.0 SDK | apcore API is declared stable; pin to compatible range; test against latest |
| `mcp` (official MCP SDK) | >= 1.0.0 | SDK breaking changes; transport API changes | Pin to compatible range; the SDK is mature; transport API is stable |
| `openai` (optional) | Not required | N/A -- `to_openai_tools()` produces plain dicts | No runtime dependency |
| Python | >= 3.10 | Minimum version for modern type hints and `match` statements | Align with apcore-python's minimum Python version |

### 8.4 Performance Requirements

| Metric | Target | Rationale |
|--------|--------|-----------|
| Tool registration time | < 100ms for 100 modules | Schema conversion is pure dict manipulation; should be near-instant |
| Tool call overhead | < 5ms added latency beyond apcore Executor time | apcore-mcp is a thin adapter; no heavy processing in the routing layer |
| Memory overhead | < 10MB for 100 modules | Tool definitions are small JSON objects; no large data structures |
| Concurrent connections (HTTP) | Handle 10+ simultaneous MCP clients | Delegated to MCP SDK's HTTP server; apcore-mcp adds no bottlenecks |

---

## 9. Scope & Boundaries

### 9.1 In Scope

- Auto-discovery of all modules from an apcore Registry
- Schema mapping: apcore `input_schema`/`output_schema` to MCP `inputSchema` and OpenAI `parameters`
- Annotation mapping: apcore `ModuleAnnotations` to MCP tool annotations
- Execution routing: MCP tool calls to apcore Executor (with ACL, validation, middleware)
- Error mapping: apcore error hierarchy to MCP error responses
- Transport support: stdio, Streamable HTTP, SSE
- `serve()` function for MCP Server launch
- `to_openai_tools()` function for OpenAI tool list export
- CLI entry point (`python -m apcore_mcp`)
- Dynamic tool registration via Registry events
- Structured logging

### 9.2 Out of Scope

- **MCP protocol implementation** -- use the official MCP SDK; apcore-mcp does not reimplement the protocol.
- **Module definition** -- that is apcore-python's responsibility. apcore-mcp only reads module metadata.
- **Domain-specific node/function wrapping** -- that is the job of xxx-apcore projects (comfyui-apcore, vnpy-apcore, etc.).
- **OpenAI API client or agent runtime** -- apcore-mcp exports tool definitions; the user's application handles API calls.
- **A2A (Agent-to-Agent) adapter** -- this is a separate future project (apcore-a2a).
- **OAuth / API key management** -- apcore-mcp provides JWT validation via `JWTAuthenticator` (F-027), but does not implement OAuth flows, API key issuance, or user management. Custom auth backends can be plugged in via the `Authenticator` protocol.
- **Full-featured GUI or production dashboard** -- the optional MCP Tool Explorer (F-026) is a minimal developer debugging UI, not a production monitoring dashboard.
- **Module execution outside the apcore Executor** -- all calls go through the Executor pipeline.

### 9.3 Assumptions

1. The apcore-python SDK API (`Registry`, `Executor`, `ModuleDescriptor`, `ModuleAnnotations`) is stable and will not undergo breaking changes during development.
2. The `mcp` Python SDK provides stable APIs for tool registration, transport configuration, and error handling.
3. MCP clients (Claude Desktop, Cursor) correctly implement the MCP protocol for tool discovery and invocation.
4. apcore modules produce JSON-serializable output dicts from their `execute()` methods.
5. The target deployment environment has Python >= 3.10 installed.

---

## 10. Launch Plan

### 10.1 Phased Rollout

#### Phase 1: Core (Week 1)
- F-001: Registry-to-MCP Schema Mapping
- F-002: Annotation-to-MCP Mapping
- F-003: MCP Execution Routing
- F-004: MCP Error Mapping
- F-005: `serve()` function
- F-006: stdio transport
- F-008: `to_openai_tools()` function
- Unit tests for all schema mapping and error mapping logic
- **Milestone:** A simple demo registry can be served via stdio and tools are callable from an MCP client.

#### Phase 2: Complete (Week 2)
- F-007: Streamable HTTP transport
- F-009: CLI entry point
- F-010: SSE transport
- F-013: Structured output responses
- F-014: Executor passthrough
- F-016: Logging and observability
- Integration tests with Claude Desktop
- **Milestone:** All transports working, CLI usable, Claude Desktop verified.

#### Phase 3: Polish (Week 3)
- F-011: OpenAI annotation embedding
- F-012: OpenAI strict mode support
- F-015: Dynamic tool registration
- Documentation (quickstart guide, API reference, Claude Desktop setup guide)
- Package publishing preparation (pyproject.toml, PyPI metadata)
- **Milestone:** Package ready for v0.1.0 release.

### 10.2 Launch Criteria

The following must all be true before the v0.1.0 release:

1. All P0 features (F-001 through F-009) pass their acceptance criteria.
2. Test coverage >= 90% on core logic (`src/apcore_mcp/`).
3. Verified working with Claude Desktop via stdio transport.
4. Verified working with at least one HTTP-based MCP client via Streamable HTTP transport.
5. `to_openai_tools()` output verified with OpenAI API (`gpt-4o` model).
6. Quickstart documentation exists and is verified by a developer who did not write the code.
7. `pyproject.toml` is complete with correct dependencies, metadata, and entry points.
8. No known P0 or P1 bugs.

### 10.3 Documentation Requirements

| Document | Purpose | Required for Launch? |
|----------|---------|:-------------------:|
| README.md | Project overview, quickstart, installation | Yes |
| Quickstart Guide | Step-by-step: install, discover, serve | Yes |
| API Reference | `serve()`, `to_openai_tools()` signatures and parameters | Yes |
| Claude Desktop Setup Guide | How to configure Claude Desktop to use apcore-mcp | Yes |
| Architecture Overview | Internal design for contributors | No (post-launch) |
| Migration Guide | N/A for v1, needed for future breaking changes | No |

---

## 11. Success Criteria & KPIs

### 11.1 Quantitative Metrics

| Metric | Target | Timeline | How Measured |
|--------|--------|----------|--------------|
| Schema mapping accuracy | 100% field correctness | At launch | Automated test suite: every `ModuleDescriptor` field mapped and verified |
| Time from install to working MCP server | < 5 minutes | At launch | Timed walkthrough with quickstart guide |
| Lines of user code required for basic MCP server | <= 5 lines | At launch | Count lines in quickstart example |
| Package core logic size | 500-1,200 lines | At launch | `cloc src/apcore_mcp/ --exclude-dir=tests` |
| Test coverage | >= 90% line coverage | At launch | `pytest --cov` report |
| P0 feature completion | 9/9 | At launch | Feature checklist |
| MCP client compatibility | >= 2 verified clients | At launch | Manual testing log |
| PyPI release | Published and installable | At launch | `pip install apcore-mcp` succeeds |

### 11.2 Qualitative Success Indicators

- **Developer sentiment:** Early adopters describe the setup as "trivially easy" or "it just works."
- **Ecosystem validation:** At least one xxx-apcore project (comfyui-apcore) adopts apcore-mcp instead of building its own MCP integration.
- **Technical credibility:** The existence of apcore-mcp is cited as evidence of apcore's AI compatibility in blog posts, talks, or documentation.
- **Community contribution:** At least one external contributor submits a PR or issue within 2 months of launch.

---

## 12. Open Questions & Risks

### 12.1 Open Questions

| # | Question | Impact | Decision Needed By |
|---|----------|--------|-------------------|
| OQ-1 | How should apcore modules with complex/nested `output_schema` be serialized in MCP responses? Should we use JSON text content, or leverage MCP's structured content types? | Affects F-013 implementation | Phase 1 end |
| OQ-2 | Should `to_openai_tools()` normalize `module_id` values that contain characters invalid for OpenAI function names (e.g., dots, slashes)? OpenAI function names must match `^[a-zA-Z0-9_-]+$`. | Affects F-008 correctness for all module IDs | Phase 1 start |
| OQ-3 | Should annotations be embedded in OpenAI tool descriptions by default or opt-in? | Affects F-011 default behavior | Phase 2 |
| OQ-4 | Should the `serve()` function support an `async` variant (`serve_async()`) for embedding in existing async applications? | Affects API surface | Phase 2 |
| OQ-5 | How to handle apcore modules with `input_schema` containing `$defs` / `$ref` -- should these be inlined for MCP/OpenAI compatibility? | Affects schema mapping correctness for complex schemas | Phase 1 start |

### 12.2 Risk Matrix

| Risk | Likelihood | Impact | Severity | Mitigation |
|------|:----------:|:------:|:--------:|------------|
| **MCP SDK breaking changes** in transport API | Low | High | Medium | Pin dependency to compatible range; MCP SDK is past 1.0 and stable; monitor changelog |
| **apcore-python API changes** before 1.0 | Low | High | Medium | apcore API is declared stable; coordinate with apcore maintainers; integration tests catch breaks early |
| **Module IDs incompatible with OpenAI function name constraints** (dots in IDs) | High | Medium | Medium | Implement a deterministic normalization function (e.g., replace `.` with `_`) in F-008; provide a reverse-mapping function for dispatching |
| **Complex schemas with $ref not supported by MCP clients** | Medium | Medium | Medium | Implement $ref inlining as part of schema conversion; test with real MCP clients |
| **Async/sync mismatch** between MCP SDK (async) and apcore Executor (sync+async) | Medium | Low | Low | Use `Executor.call_async()` which handles both sync and async modules transparently |
| **SSE transport deprecated** and removed from MCP SDK | Medium | Low | Low | SSE is P1, not P0; if removed, gracefully degrade with a clear error message |
| **Low initial adoption** due to small apcore ecosystem | Medium | Medium | Medium | Ship a compelling demo with mock modules; integrate with comfyui-apcore early; promote via apcore documentation |

### 12.3 Mitigation Strategies

1. **Dependency pinning with flexibility:** Use compatible release specifiers (e.g., `apcore>=0.2.0,<1.0`, `mcp>=1.0.0,<2.0`) to allow minor updates while protecting against breaking changes.
2. **Integration test suite:** Maintain integration tests that run against the latest versions of apcore-python and the MCP SDK in CI. Break-detection within 24 hours.
3. **Module ID normalization:** Design a bijective (reversible) normalization function for module IDs early in Phase 1. Document the normalization rules. Provide `from_openai_name(name)` for reverse lookup.
4. **Schema inlining:** Implement a `$ref` resolver that inlines all references before passing schemas to MCP/OpenAI. Leverage apcore's existing `ref_resolver` module if applicable.
5. **Demo-first development:** Build a self-contained demo with 3-5 mock modules before the core library. Use the demo as both a test fixture and a marketing asset.

---

## 13. Appendix

### 13.1 Glossary

| Term | Definition |
|------|------------|
| **apcore** | A schema-driven module development framework that provides Registry, Executor, and schema validation for modular applications. |
| **apcore-python** | The Python SDK implementation of the apcore framework. Provides `Registry`, `Executor`, `ModuleDescriptor`, `ModuleAnnotations`, and the error hierarchy. |
| **MCP (Model Context Protocol)** | An open protocol introduced by Anthropic for connecting AI assistants to external tools and data sources. Defines a standard for tool discovery, invocation, and result reporting. |
| **MCP Server** | A process that implements the MCP protocol and exposes tools, resources, and/or prompts to MCP clients. |
| **MCP Client** | An AI assistant or application that connects to MCP Servers to discover and invoke tools. Examples: Claude Desktop, Cursor, Windsurf. |
| **MCP Tool** | A callable function exposed by an MCP Server, defined by a name, description, and input schema. |
| **MCP Tool Annotations** | Metadata on an MCP tool that describes behavioral characteristics: `destructiveHint`, `readOnlyHint`, `idempotentHint`, `openWorldHint`. Used by clients for safety decisions. |
| **OpenAI Function Calling / Tools** | OpenAI's mechanism for allowing language models to invoke external functions. Tools are defined with a name, description, and parameters (JSON Schema). |
| **Structured Outputs (strict mode)** | An OpenAI feature where `"strict": true` on a tool definition constrains the model to produce outputs exactly matching the schema. |
| **Registry** | An apcore class that discovers, registers, and provides query access to modules. Central to apcore's module management. |
| **Executor** | An apcore class that orchestrates the module execution pipeline: context creation, safety checks, ACL, validation, middleware, execution, and result return. |
| **ModuleDescriptor** | An apcore dataclass containing a module's full metadata: `module_id`, `name`, `description`, `input_schema`, `output_schema`, `annotations`, `examples`, `tags`, `version`. |
| **ModuleAnnotations** | An apcore dataclass with behavioral flags: `readonly`, `destructive`, `idempotent`, `requires_approval`, `open_world`. |
| **xxx-apcore** | Convention for apcore adapter projects targeting specific domains: `comfyui-apcore`, `vnpy-apcore`, `blender-apcore`, etc. |
| **Transport** | The communication mechanism between MCP client and server. Supported: stdio (stdin/stdout), Streamable HTTP, SSE (Server-Sent Events). |
| **stdio** | A transport where the MCP server reads from standard input and writes to standard output. Used by Claude Desktop for local server processes. |
| **Streamable HTTP** | The recommended HTTP-based MCP transport. Replaces SSE as the primary network transport. |
| **SSE (Server-Sent Events)** | A deprecated MCP transport using HTTP Server-Sent Events. Supported for backward compatibility with older clients. |
| **ACL (Access Control List)** | apcore's built-in mechanism for controlling which callers can invoke which modules. Configured with rules and a default policy. |

### 13.2 References

| Reference | Description |
|-----------|-------------|
| `ideas/apcore-mcp.md` | Original idea document with validation status, schema mappings, and competitive analysis |
| apcore-python source (`/Users/tercel/WorkSpace/aipartnerup/apcore-python/`) | SDK source code: Registry, Executor, ModuleDescriptor, ModuleAnnotations, error hierarchy |
| [MCP Specification](https://modelcontextprotocol.io/) | Official Model Context Protocol specification |
| [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk) | Official Python SDK for building MCP servers and clients |
| [OpenAI Function Calling Documentation](https://platform.openai.com/docs/guides/function-calling) | OpenAI's guide to function calling / tools |
| apcore SCOPE.md | apcore's scope document that designates MCP/A2A adapters as independent projects |

### 13.3 apcore API Surface Used by apcore-mcp

For reference, the following apcore-python APIs are consumed by apcore-mcp:

```python
# Registry (module discovery and query)
registry = Registry(extensions_dir="./extensions")
registry.discover() -> int
registry.list(tags=None, prefix=None) -> list[str]
registry.get(module_id) -> module | None
registry.get_definition(module_id) -> ModuleDescriptor | None
registry.iter() -> Iterator[tuple[str, Any]]
registry.on("register", callback)
registry.on("unregister", callback)

# Executor (execution pipeline)
executor = Executor(registry, middlewares=None, acl=None, config=None)
executor.call(module_id, inputs, context=None) -> dict
executor.call_async(module_id, inputs, context=None) -> dict  # async

# ModuleDescriptor (schema data)
descriptor.module_id: str
descriptor.description: str
descriptor.input_schema: dict[str, Any]   # JSON Schema
descriptor.output_schema: dict[str, Any]  # JSON Schema
descriptor.annotations: ModuleAnnotations | None
descriptor.name: str | None
descriptor.documentation: str | None
descriptor.tags: list[str]
descriptor.version: str
descriptor.examples: list[ModuleExample]

# ModuleAnnotations (behavioral flags)
annotations.readonly: bool
annotations.destructive: bool
annotations.idempotent: bool
annotations.requires_approval: bool
annotations.open_world: bool

# Error hierarchy (all extend ModuleError)
ModuleNotFoundError(module_id)
SchemaValidationError(message, errors)
ACLDeniedError(caller_id, target_id)
ModuleTimeoutError(module_id, timeout_ms)
InvalidInputError(message)
CallDepthExceededError(depth, max_depth, call_chain)
CircularCallError(module_id, call_chain)
CallFrequencyExceededError(module_id, count, max_repeat, call_chain)
```
