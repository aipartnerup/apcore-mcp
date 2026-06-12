---
description: "Auto-generated index of all apcore-mcp feature specs with dependencies, release status, and the recommended 13-step implementation execution order from Schema Converter to Explorer UI."
---

# Feature Overview

> Auto-generated index of feature specs for apcore-mcp.
> Updated: 2026-05-14

## Features

| Feature | Description | Dependencies | Status |
|---------|-------------|--------------|--------|
| [Schema Converter](./schema-converter.md) | Resolves JSON Schema references for MCP/OpenAI. | none | released-v0.14.0 |
| [Annotation Mapper](./annotation-mapper.md) | Maps apcore metadata to protocol behavioral hints. | none | released-v0.14.0 |
| [Execution Router](./execution-router.md) | Dispatches tool calls to the apcore Executor pipeline. | error-mapper | released-v0.14.0 |
| [Error Mapper](./error-mapper.md) | Translates exceptions to protocol-compliant errors. | none | released-v0.14.0 |
| [MCP Server Factory](./mcp-server-factory.md) | Builds the low-level MCP server instance. | schema-converter, annotation-mapper, execution-router | released-v0.14.0 |
| [OpenAI Converter](./openai-converter.md) | Exports modules as OpenAI-compatible tool definitions. | schema-converter, annotation-mapper | released-v0.14.0 |
| [Extension Bridge](./extension-bridge.md) | Wires apcore ExtensionManager into the MCP server pipeline. | mcp-server-factory, execution-router | released-v0.14.0 (`apply()` portion deferred — EB-1) |
| [Transport Manager](./transport-manager.md) | Manages stdio and network communication layers. | mcp-server-factory | released-v0.14.0 |
| [Registry Listener](./registry-listener.md) | Enables hot-reloading of tools on registry changes. | mcp-server-factory | released-v0.14.0 |
| [JWT Authenticator](./jwt-authenticator.md) | Secures HTTP transports using bearer tokens. | none | released-v0.14.0 |
| [Approval Handler](./approval-handler.md) | Implements human-in-the-loop confirmation via MCP. | execution-router | released-v0.14.0 |
| [Approval Handler (Phase B)](./approval-phase-b.md) | Storage-backed async approval flow with `StorageBackedApprovalHandler`, `InMemoryApprovalStore`, and `__apcore_approval_check` meta-tool. | approval-handler, execution-router | released-v0.16.0 |
| [Async Task Bridge](./async-task-bridge.md) | Routes async-hinted modules to apcore's AsyncTaskManager and exposes task meta-tools. | execution-router, error-mapper | released-v0.14.0 |
| [Explorer UI](./explorer-ui.md) | Web dashboard for inspecting and testing tools. | mcp-server-factory, execution-router | released-v0.14.0 |
| [Markdown](./markdown.md) | Renders `Tool.description` and OpenAI `function.description` as canonical apcore-toolkit Markdown for richer LLM tool-selection signal (`rich_description` / `richDescription` / `with_rich_description`). | apcore-toolkit 0.6+ (optional) | released-v0.15.0 |

## Execution Order

The implementation should follow this sequence to ensure core logic is stable before layering on transports and UI.

1. **Schema Converter** — Foundational for all protocol-compliant tool definitions.
2. **Annotation Mapper** — Required for both MCP and OpenAI adapters.
3. **Error Mapper** — Essential for clean feedback in both interfaces.
4. **Execution Router** — The bridge to the apcore Executor; depends on error-mapper.
5. **OpenAI Converter** — High-value, zero-dependency feature for immediate utility.
6. **MCP Server Factory** — Combines foundational modules into the core MCP server logic.
7. **Extension Bridge** — Wires apcore ExtensionManager into the factory and Executor before transports are layered on.
8. **Transport Manager** — Enables connectivity (stdio, HTTP) for the server.
9. **Registry Listener** — Adds dynamic capabilities once the base server is stable.
10. **Approval Handler** — Enhances safety for the operational server.
10a. **Approval Handler (Phase B)** — Adds storage-backed async approval polling and the `__apcore_approval_check` meta-tool.
11. **Async Task Bridge** — Adds background task execution via apcore's AsyncTaskManager once approval semantics are in place.
12. **JWT Authenticator** — Secures the server for network deployments.
13. **Explorer UI** — Provides the final development and debugging interface.
