<div align="center">
  <img src="./apcore-mcp-logo.svg" alt="apcore-mcp logo" width="200"/>
</div>

# apcore-mcp

Automatic MCP Server & OpenAI Tools Bridge for apcore.

**apcore-mcp** turns any [apcore](https://github.com/aipartnerup/apcore)-based project into an MCP Server and OpenAI tool provider — with **zero code changes** to your existing project.

```
┌──────────────────┐
│  comfyui-apcore  │  ← your existing apcore project (unchanged)
│  robotics-apcore │
│  audio-apcore    │
│  ...             │
└────────┬─────────┘
         │  extensions directory
         ▼
┌──────────────────┐
│    apcore-mcp    │  ← just install & point to extensions dir
└───┬──────────┬───┘
    │          │
    ▼          ▼
  MCP       OpenAI
 Server      Tools
```

## Design Philosophy

- **Zero intrusion** — your apcore project needs no code changes, no imports, no dependencies on apcore-mcp
- **Zero configuration** — point to an extensions directory, everything is auto-discovered
- **Pure adapter** — apcore-mcp reads from the apcore Registry; it never modifies your modules
- **Works with any `xxx-apcore` project** — if it uses the apcore Module Registry, apcore-mcp can serve it

## Features

- **Auto-discovery** — all modules in the extensions directory are found and exposed automatically
- **Three transports** — stdio (default, for desktop clients), Streamable HTTP, and SSE
- **Annotation mapping** — apcore annotations (readonly, destructive, idempotent) map to MCP ToolAnnotations
- **Schema conversion** — JSON Schema `$ref`/`$defs` inlining, strict mode for OpenAI Structured Outputs
- **Error sanitization** — ACL errors and internal errors are sanitized; stack traces are never leaked
- **Dynamic registration** — modules registered/unregistered at runtime are reflected immediately
- **Dual output** — same registry powers both MCP Server and OpenAI tool definitions
- **Tool Inspector** — optional built-in browser UI for inspecting and testing MCP tools (like Swagger UI for MCP)

## How It Works

### Mapping: apcore to MCP

| apcore | MCP |
|--------|-----|
| `module_id` | Tool name |
| `description` | Tool description |
| `input_schema` | `inputSchema` |
| `annotations.readonly` | `ToolAnnotations.readOnlyHint` |
| `annotations.destructive` | `ToolAnnotations.destructiveHint` |
| `annotations.idempotent` | `ToolAnnotations.idempotentHint` |
| `annotations.open_world` | `ToolAnnotations.openWorldHint` |

### Mapping: apcore to OpenAI Tools

| apcore | OpenAI |
|--------|--------|
| `module_id` (`image.resize`) | `name` (`image-resize`) |
| `description` | `description` |
| `input_schema` | `parameters` |

Module IDs with dots are normalized to dashes for OpenAI compatibility (bijective mapping).

### Architecture

```
Your apcore project (unchanged)
    │
    │  extensions directory
    ▼
apcore-mcp (separate process / library call)
    │
    ├── MCP Server path
    │     SchemaConverter + AnnotationMapper
    │       → MCPServerFactory → ExecutionRouter → TransportManager
    │
    └── OpenAI Tools path
          SchemaConverter + AnnotationMapper + IDNormalizer
            → OpenAIConverter → tool definitions
```

## Implementations

| Language | Repository | Package | Status |
|----------|-----------|---------|--------|
| Python | [apcore-mcp-python](https://github.com/aipartnerup/apcore-mcp-python) | `pip install apcore-mcp` |  ✅  v0.1.0 |
| TypeScript | [apcore-mcp-typescript](https://github.com/aipartnerup/apcore-mcp-typescript) | `npm install apcore-mcp` |  ✅  v0.1.0 |
| Go | apcore-mcp-go | — | Planned |

## Specification Documents

- [Product Requirements (PRD)](docs/prd-apcore-mcp.md)
- [Software Requirements (SRS)](docs/srs-apcore-mcp.md)
- [Technical Design](docs/tech-design-apcore-mcp.md)
- [Test Plan](docs/test-plan-apcore-mcp.md)

## License

Apache-2.0
