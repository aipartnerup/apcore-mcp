# Idea: apcore-mcp

> apcore Module Registry to MCP Server & OpenAI Tools automatic bridge

## Status: Validated

## Problem Statement

apcore modules are inherently AI-perceivable (with `input_schema`, `output_schema`, `description`, `annotations`), but there is no standard way to expose them to AI systems via MCP (Model Context Protocol) or OpenAI Function Calling. Developers would need to manually write MCP tool definitions or OpenAI tool schemas for each module, losing the automation benefits of apcore's schema system.

## Proposed Solution

An independent adapter package that automatically converts any apcore Module Registry into a fully functional MCP Server, and also exports OpenAI-compatible tool definitions. One package bridges the entire apcore ecosystem to the AI agent world — both MCP clients (Claude, Cursor, Windsurf) and OpenAI-compatible platforms.

```python
from apcore import Registry
from apcore_mcp import serve, to_openai_tools

registry = Registry(extensions_dir="./extensions")
registry.discover()

# MCP Server mode — expose all modules as MCP Tools
serve(registry)

# OpenAI Tools mode — export as OpenAI function calling format
tools = to_openai_tools(registry)
# [{"type": "function", "name": "...", "description": "...", "parameters": {...}}, ...]
```

## Core Value Proposition

- **Zero manual tool definitions**: apcore schemas auto-map to MCP `inputSchema` and OpenAI `parameters`
- **Dual output**: one package serves both MCP and OpenAI ecosystems
- **Annotation preservation**: apcore `annotations` (destructive, readonly, requires_approval, idempotent, open_world) map to MCP tool annotations for AI safety
- **Ecosystem multiplier**: every `xxx-apcore` project (comfyui-apcore, vnpy-apcore, blender-apcore, ...) instantly gains MCP + OpenAI capability via this one package
- **Executable proof** of apcore's core claim: "modules inherently MCP/OpenAI compatible"

## Key Schema Mapping

### apcore → MCP

```
apcore Module                    →  MCP Tool
─────────────────────────────────────────────────
module_id                        →  tool name
description                      →  tool description
input_schema (JSON Schema)       →  inputSchema
output_schema (JSON Schema)      →  (structured response)
annotations.destructive          →  tool annotation: destructive
annotations.readonly             →  tool annotation: readOnlyHint
annotations.requires_approval    →  confirmation flow
annotations.open_world           →  tool annotation: openWorldHint
annotations.idempotent           →  tool annotation: idempotentHint
```

### apcore → OpenAI Tools

```
apcore Module                    →  OpenAI Tool
─────────────────────────────────────────────────
(wrapper)                        →  type: "function"
module_id                        →  name
description                      →  description
input_schema (JSON Schema)       →  parameters
output_schema (JSON Schema)      →  (parsed from response)
annotations.requires_approval    →  (application-level confirmation)
```

Note: OpenAI tools format does not have native annotation support. Annotations like `destructive`, `readonly`, etc. can be optionally embedded in the tool description or returned as metadata for the application layer to handle.

## Architecture Position

Defined by apcore SCOPE.md:

- apcore core: "MCP/A2A Adapters → Won't Do → Independent adapter projects"
- apcore-mcp: independent adapter that uses the MCP SDK internally
- Does NOT reimplement MCP protocol; bridges apcore's registry to the existing MCP SDK
- OpenAI tools output is pure schema conversion (no server needed — OpenAI uses REST API)

```
apcore-python (core)
    ↓ provides Registry + Module + Schema
apcore-mcp (this project)
    ├── MCP Server path:
    │     ↓ uses MCP SDK (mcp Python package)
    │   MCP Server (stdio / Streamable HTTP transport)
    │     ↓
    │   Claude / Cursor / Windsurf / any MCP client
    │
    └── OpenAI Tools path:
          ↓ pure schema conversion (no SDK dependency)
        OpenAI-compatible tools list
          ↓
        OpenAI API / any OpenAI-compatible platform
```

## Scope

### In Scope

- Auto-discovery of all modules from an apcore Registry
- **MCP output**:
  - Schema mapping: apcore input_schema/output_schema → MCP inputSchema
  - Annotation mapping: apcore annotations → MCP tool annotations
  - Execution routing: MCP tool calls → apcore Executor (with ACL, validation, middleware)
  - Error mapping: apcore errors → MCP error responses
  - Transport support: stdio and Streamable HTTP
  - Dynamic tool registration (hot-reload when registry changes)
- **OpenAI Tools output**:
  - Schema conversion: apcore input_schema → OpenAI `parameters`
  - `to_openai_tools(registry)` returns a list of OpenAI-compatible tool definitions
  - Optional: annotation embedding in tool descriptions for safety-aware applications

### Out of Scope

- MCP protocol implementation (use existing MCP SDK)
- Module definition (that's apcore-python's job)
- Domain-specific node/function wrapping (that's xxx-apcore projects' job)
- OpenAI API client / agent runtime (that's the user's application)
- A2A adapter (separate project: apcore-a2a, future)

## Dependencies

- `apcore` (apcore-python SDK)
- `mcp` (official MCP Python SDK)
- `openai` (optional — only needed if type references are used; `to_openai_tools()` itself produces plain dicts)

## Target Users

1. **apcore module developers** who want their modules callable by AI agents
2. **xxx-apcore project developers** (comfyui-apcore, vnpy-apcore, etc.) who need MCP or OpenAI tools output
3. **AI agent builders** who want to use apcore-powered tools in Claude/Cursor/OpenAI workflows

## Competitive Analysis

| Approach | How it works | Limitation |
|----------|-------------|-----------|
| Manual MCP server | Hand-write tool definitions | Tedious, error-prone, no validation |
| Manual OpenAI tools | Hand-write function schemas | Duplicated effort, no standardization |
| Existing MCP frameworks | Build tools from scratch | No schema reuse, no standardization |
| **apcore-mcp** | Auto-bridge from apcore Registry | Requires apcore modules (by design) |

## Demand Validation

- MCP ecosystem is booming (2025-2026), with 5+ "best MCP servers" roundup articles
- OpenAI Function Calling is the most widely adopted tool-use format (GPT-4, GPT-4o, o1, o3)
- Existing ComfyUI MCP servers (5+ projects) all manually define tools — proving the need for automation
- apcore already enforces the exact metadata both MCP and OpenAI need (schema + description)
- The mapping is nearly 1:1 for both formats, meaning technical risk is very low
- Supporting both formats in one package maximizes adoption surface

## Development Estimate

- **Scope**: ~500-1200 lines of core logic + tests + docs (OpenAI tools conversion adds ~100-200 lines)
- **Timeline**: ~2-3 weeks
- **Priority**: Build FIRST (before comfyui-apcore), as it is the infrastructure layer

## Success Criteria

1. Any apcore Registry can be served as MCP Server with one function call
2. Any apcore Registry can be exported as OpenAI tools list with one function call
3. All module schemas correctly map to MCP tool definitions
4. All module schemas correctly map to OpenAI tool definitions
5. All apcore annotations correctly map to MCP tool annotations
6. Execution goes through full apcore pipeline (ACL, validation, middleware)
7. Works with Claude Desktop, Cursor, and other MCP clients
8. OpenAI tools output works with OpenAI API and compatible platforms

## Relationship to Other Projects

- **apcore** (protocol spec): defines the standard this project implements against
- **apcore-python**: provides the Registry/Executor API this project consumes
- **comfyui-apcore**: first consumer of this project (ComfyUI nodes → apcore → MCP / OpenAI tools)
- **apcore-a2a** (future): sibling adapter for Agent-to-Agent protocol

## Open Questions

1. How to handle apcore modules with complex/nested output_schema in MCP responses?
2. Should dynamic tool registration be supported (add/remove tools at runtime)?
3. Should annotations be embedded in OpenAI tool descriptions by default, or opt-in?
4. Should `to_openai_tools()` also support OpenAI's `strict: true` mode (Structured Outputs)?

## Resolved Questions

1. ~~Should this also support OpenAI Function Calling format?~~ **Yes** — included in this project. The format difference is minimal (~10-20 lines of conversion logic), not worth a separate package.
2. ~~SSE transport: needed for v1?~~ **No** — SSE is deprecated in MCP SDK. Use stdio + Streamable HTTP instead.
