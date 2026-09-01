<div align="center">
  <img src="./apcore-mcp-logo.svg" alt="apcore-mcp logo" width="200"/>
</div>

# apcore-mcp

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Python Version](https://img.shields.io/badge/python-3.11%2B-blue)](https://github.com/aiperceivable/apcore-mcp-python)
[![TypeScript Version](https://img.shields.io/badge/TypeScript-Node_18%2B-blue)](https://github.com/aiperceivable/apcore-mcp-typescript)
[![Rust Version](https://img.shields.io/badge/Rust-1.75%2B-blue)](https://github.com/aiperceivable/apcore-mcp-rust)

> **Build once, invoke by Code or AI.**
> Same modules, same pipeline — an AI caller gets no shortcut.

**apcore-mcp** turns any [apcore](https://github.com/aiperceivable/apcore)-based project into an [MCP Server](https://modelcontextprotocol.io/) and [OpenAI tool](https://platform.openai.com/docs/guides/function-calling) provider — with **zero code changes** to your existing project.

```
┌──────────────────┐
│  django-apcore   │  ← your existing apcore project (unchanged)
│  nestjs-apcore   │
│  tiptap-apcore   │
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

---

## Installation

=== "🐍 Python"

    ```bash
    pip install apcore-mcp
    ```
    *Requires Python 3.11+. The optional `[markdown]` extra adds Markdown tool descriptions.*

=== "📘 TypeScript"

    ```bash
    npm install apcore-mcp
    ```
    *Requires Node.js 18+. apcore-toolkit is an optional peer for Markdown tool descriptions.*

=== "🦀 Rust"

    ```bash
    cargo add apcore-mcp
    ```
    *Requires Rust 1.75+.*

---

## Quick Start

### 1. Serve your modules via MCP

=== "🐍 Python"

    ```python
    from apcore_mcp import APCoreMCP

    mcp = APCoreMCP("./extensions")

    # Launch as MCP Server over stdio
    mcp.serve()

    # Or with HTTP + Explorer UI
    mcp.serve(transport="streamable-http", port=8000, explorer=True)
    ```

=== "📘 TypeScript"

    ```typescript
    import { serve } from "apcore-mcp";

    // Launch MCP server over stdio
    await serve("./extensions");

    // Launch over Streamable HTTP with Explorer UI
    await serve("./extensions", {
      transport: "streamable-http",
      port: 8000,
      explorer: true,
    });
    ```

=== "🦀 Rust"

    ```rust
    use apcore_mcp::APCoreMCP;
    use apcore::{Config, Executor};
    use std::sync::Arc;

    // Rust cannot discover modules from a directory — see the note below.
    // Register your modules, then hand the builder an Executor.
    let registry = build_my_registry();
    let executor = Arc::new(Executor::new(registry, Config::default()));

    let mcp = APCoreMCP::builder()
        .backend(executor)
        .build()?;
    mcp.serve()?;
    ```

    !!! warning "Rust does not support directory auto-discovery yet"
        `.backend("./extensions")` and the CLI's `--extensions-dir` compile and run, but they build
        an **empty registry** with a warning — apcore's public Rust API has no runtime directory
        discovery. The server starts and serves zero user modules (only the `__apcore_*` meta-tools).
        Use `BackendSource::Registry` — a `Registry`/`Executor` you construct in code — which is
        fully supported. Tracked in the CHANGELOG under *Known limitations (Rust)*; the Python and
        TypeScript paths above are unaffected.

=== "💻 CLI"

    ```bash
    # stdio (default)
    apcore-mcp --extensions-dir ./extensions

    # Streamable HTTP with Explorer UI
    apcore-mcp --extensions-dir ./extensions --transport streamable-http --port 8000 --explorer
    ```

    *The Python and TypeScript CLIs discover modules from the directory. The **Rust** CLI does not —
    `--extensions-dir` builds an empty registry there; see the Rust tab above.*

### 2. Export as OpenAI Tools

=== "🐍 Python"

    ```python
    from apcore_mcp import APCoreMCP

    mcp = APCoreMCP("./extensions")
    tools = mcp.to_openai_tools()
    ```

=== "📘 TypeScript"

    ```typescript
    import { APCoreMCP } from "apcore-mcp";

    const mcp = new APCoreMCP("./extensions");
    const tools = mcp.toOpenaiTools();
    ```

=== "🦀 Rust"

    ```rust
    use apcore_mcp::APCoreMCP;

    let mcp = APCoreMCP::builder()
        .backend(executor)
        .build()?;
    //                          embed_annotations, strict
    let tools = mcp.to_openai_tools(false,             true)?;
    ```

!!! note "`strict` does not default the same way in all three"
    Written exactly as above, the three tabs produce **different** OpenAI schemas from the same
    registry: Python's `to_openai_tools()` defaults `strict=False` and emits permissive schemas,
    while TypeScript's `toOpenaiTools()` defaults to `true` and the Rust call passes `true`
    positionally — both emit strict ones (`additionalProperties: false`, every property forced into
    `required`, nullable types wrapped, `"strict": true` on the function). OpenAI's function-calling
    treats the two shapes differently. **Pass `strict` explicitly** in anything portable:
    `mcp.to_openai_tools(strict=True)` / `mcp.toOpenaiTools({ strict: true })`.

---

## Key Features

- **🚀 Zero Intrusion**: Your apcore project needs no code changes, no imports, and no extra dependencies.
- **🔍 Auto-discovery**: Point to an extensions directory, and everything is automatically discovered and exposed *(Python and TypeScript; the Rust bridge needs a `Registry` you build in code — see the Quick Start note)*.
- **🌐 Triple Transport**: Supports `stdio` (for local LLMs), `Streamable HTTP`, and `SSE`.
- **🛠️ Tool Explorer**: Browser-based UI to browse schemas and test tools interactively (like Swagger UI for MCP).
- **🛡️ Security**: Built-in JWT authentication, PEM key support, and runtime approval elicitation.
- **🤖 AI Optimized**: Enriched metadata (`x-when-to-use`), error sanitization with AI guidance, and strict mode for OpenAI.
- **🔄 Dynamic**: Reflects module registrations/unregistrations at runtime without restarting.

---

## Architecture

apcore-mcp acts as a protocol-specific adapter on top of the apcore Registry, mapping its metadata to MCP and OpenAI standards:

| apcore Concept | MCP Mapping | OpenAI Mapping |
|----------------|-------------|----------------|
| `module_id` | Tool name, **verbatim** — dots and all (`image.resize`); overridable via `display.mcp.alias` | `name` (dash-normalized: `image-resize`) |
| `description` | Tool description | `description` |
| `input_schema` | `inputSchema` | `parameters` |
| `annotations` | `ToolAnnotations` hints | Description suffixes (optional) |
| `metadata` | `_meta` fields | — |

---

## Features & Specifications

The project is architected as a set of modular features, each with its own specification to ensure consistency across languages:

- **[Feature Overview](docs/features/overview.md)** — Implementation roadmap & dependencies
- [Schema Converter](docs/features/schema-converter.md) — Reference resolution for AI protocols
- [Annotation Mapper](docs/features/annotation-mapper.md) — Behavioral hint translation
- [Execution Router](docs/features/execution-router.md) — Dispatcher for the 11-step pipeline
- [Error Mapper](docs/features/error-mapper.md) — Protocol-compliant error feedback
- [MCP Server Factory](docs/features/mcp-server-factory.md) — Core server builder
- [OpenAI Converter](docs/features/openai-converter.md) — Tool exporter for OpenAI
- [Extension Bridge](docs/features/extension-bridge.md) — Wires apcore ExtensionManager into MCP pipeline
- [Transport Manager](docs/features/transport-manager.md) — Stdio/HTTP connectivity
- [Registry Listener](docs/features/registry-listener.md) — Hot-reloading capabilities
- [JWT Authenticator](docs/features/jwt-authenticator.md) — Bearer token security
- [Approval Handler](docs/features/approval-handler.md) — Human-in-the-loop elicitation
- [Async Task Bridge](docs/features/async-task-bridge.md) — Routes async-hinted modules to apcore's AsyncTaskManager
- [Explorer UI](docs/features/explorer-ui.md) — Interactive dev dashboard
- [Markdown](docs/features/markdown.md) — Rich Markdown tool descriptions via apcore-toolkit
- [System Management Extension](docs/features/system-management-extension.md) — `com.aiperceivable/management` MCP extension (Phase A)

---

## Documentation

- **[Full Documentation Site](https://aiperceivable.github.io/apcore-mcp/)**
- **[Getting Started Guide](docs/getting-started.md)** — Installation and basic setup
- [PRD](docs/prd-apcore-mcp.md)
- [SRS](docs/srs-apcore-mcp.md)
- [Tech Design](docs/tech-design-apcore-mcp.md)

---

## Implementations

| Language | Repository | Package | Status |
|----------|-----------|---------|--------|
| Python | [apcore-mcp-python](https://github.com/aiperceivable/apcore-mcp-python) | `pip install apcore-mcp` | ✅ Available |
| TypeScript | [apcore-mcp-typescript](https://github.com/aiperceivable/apcore-mcp-typescript) | `npm install apcore-mcp` | ✅ Available |
| Rust | [apcore-mcp-rust](https://github.com/aiperceivable/apcore-mcp-rust) | `cargo add apcore-mcp` | ✅ Available |
| Go | apcore-mcp-go | — | Planned |

## License

This project is licensed under the **Apache License 2.0**. See the [LICENSE](LICENSE) file for details.
