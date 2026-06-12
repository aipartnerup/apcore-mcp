---
description: "Getting-started guide for apcore-mcp: install in Python/TypeScript/Rust and launch an MCP server from an extensions directory via zero-code CLI or programmatic API."
---

# Getting Started

This guide covers how to set up and use **apcore-mcp** to expose your apcore-based modules to AI agents as MCP tools or OpenAI tools.

## 1. Installation

Install **apcore-mcp** alongside your existing apcore project.

=== "Python"

    ```bash
    pip install apcore-mcp
    ```

=== "TypeScript"

    ```bash
    npm install apcore-mcp
    ```

=== "Rust"

    ```bash
    cargo add apcore-mcp
    ```

## 2. Zero-Code CLI Usage

If you have an `extensions/` directory containing your apcore modules, you can launch an MCP server immediately without writing any code.

=== "Python"

    ```bash
    # stdio (default for desktop clients like Claude)
    apcore-mcp --extensions-dir ./extensions

    # HTTP with Tool Explorer UI
    apcore-mcp --extensions-dir ./extensions 
               --transport streamable-http 
               --port 8000 
               --explorer 
               --allow-execute
    ```

=== "TypeScript"

    ```bash
    # stdio (default)
    npx apcore-mcp --extensions-dir ./extensions

    # HTTP with Tool Explorer UI
    npx apcore-mcp --extensions-dir ./extensions 
                   --transport streamable-http 
                   --port 8000 
                   --explorer 
                   --allow-execute
    ```

=== "Rust"

    ```bash
    # stdio (default)
    apcore-mcp --extensions-dir ./extensions

    # HTTP with Tool Explorer UI
    apcore-mcp --extensions-dir ./extensions 
               --transport streamable-http 
               --port 8000 
               --explorer 
               --allow-execute
    ```

Once running in HTTP mode with `--explorer`, visit `http://127.0.0.1:8000/explorer/` to browse and test your tools.

## 3. Programmatic Usage

Integrate **apcore-mcp** directly into your application. The `APCoreMCP` class is the recommended entry point — one object, all capabilities.

=== "Python"

    ```python
    from apcore_mcp import APCoreMCP

    # One line setup — auto-discovers all modules
    # Tool outputs default to raw JSON. To format as Markdown, install
    # `apcore-toolkit` and pass `output_formatter=to_markdown` (see below).
    mcp = APCoreMCP("./extensions")

    # Launch as MCP Server
    mcp.serve()

    # Or with HTTP + Explorer UI
    mcp.serve(transport="streamable-http", port=8000, explorer=True)

    # Or convert to OpenAI tool definitions
    tools = mcp.to_openai_tools()
    # response = openai.chat.completions.create(tools=tools, ...)

    # Opt into Markdown formatting (requires `pip install apcore-toolkit`)
    from apcore_toolkit import to_markdown
    mcp = APCoreMCP("./extensions", output_formatter=to_markdown)
    ```

    You can also pass an existing Registry or Executor:

    ```python
    from apcore import Registry
    from apcore_mcp import APCoreMCP

    registry = Registry(extensions_dir="./extensions")
    registry.discover()
    mcp = APCoreMCP(registry, name="my-server", tags=["public"])
    ```

    <details>
    <summary>Function-based API (still supported)</summary>

    ```python
    from apcore import Registry
    from apcore_mcp import serve, to_openai_tools

    registry = Registry(extensions_dir="./extensions")
    registry.discover()

    serve(registry, transport="stdio")
    tools = to_openai_tools(registry)
    ```
    </details>

=== "TypeScript"

    ```typescript
    import { APCoreMCP } from "apcore-mcp";

    // One line setup — auto-discovers all modules
    // Tool outputs default to raw JSON; pass an `outputFormatter` for custom formatting.
    const mcp = new APCoreMCP("./extensions");

    // Launch as MCP Server
    await mcp.serve();

    // Or with HTTP + Explorer UI
    await mcp.serve({ transport: "streamable-http", port: 8000, explorer: true });

    // Or convert to OpenAI tool definitions
    const tools = mcp.toOpenaiTools();
    ```

    You can also pass an existing Registry or Executor:

    ```typescript
    import { Registry } from "apcore-js";
    import { APCoreMCP } from "apcore-mcp";

    const registry = new Registry({ extensionsDir: "./extensions" });
    await registry.discover();
    const mcp = new APCoreMCP(registry, { name: "my-server", tags: ["public"] });
    ```

    <details>
    <summary>Function-based API (still supported)</summary>

    ```typescript
    import { Registry } from "apcore-js";
    import { serve, toOpenaiTools } from "apcore-mcp";

    const registry = new Registry({ extensionsDir: "./extensions" });
    await registry.discover();

    await serve(registry, { transport: "stdio" });
    const tools = toOpenaiTools(registry);
    ```
    </details>

=== "Rust"

    ```rust
    use std::sync::Arc;
    use apcore::{config::Config, executor::Executor, registry::registry::Registry};
    use apcore_mcp::APCoreMCP;

    // BackendSource::Executor is the functional backend (as of v0.15.0).
    // Construct Registry → Executor first, then hand the Arc<Executor> to APCoreMCP.
    let registry = Registry::new();
    // ... register modules ...
    let executor = Arc::new(Executor::new(registry, Config::default()));

    let mcp = APCoreMCP::builder()
        .backend(executor)
        .build()?;

    // Launch as MCP Server (synchronous — spawns its own Tokio runtime).
    mcp.serve()?;

    // Or convert to OpenAI tool definitions
    let tools = mcp.to_openai_tools(false, true)?;
    ```

    You can also pass an existing Registry or Executor:

    ```rust
    use std::sync::Arc;
    use apcore::registry::registry::Registry;
    use apcore_mcp::APCoreMCP;

    let registry = Arc::new(Registry::new());
    // ... register modules ...
    let mcp = APCoreMCP::builder()
        .backend(registry)
        .name("my-server")
        .tags(vec!["public".into()])
        .build()?;
    ```

## 4. MCP Client Configuration

To use your tools in desktop AI applications, add **apcore-mcp** to their configuration files.

### Claude Desktop

Add this to `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

=== "Python"

    ```json
    {
      "mcpServers": {
        "apcore": {
          "command": "apcore-mcp",
          "args": ["--extensions-dir", "/absolute/path/to/your/extensions"]
        }
      }
    }
    ```

=== "TypeScript"

    ```json
    {
      "mcpServers": {
        "apcore": {
          "command": "npx",
          "args": ["apcore-mcp", "--extensions-dir", "/absolute/path/to/your/extensions"]
        }
      }
    }
    ```

=== "Rust"

    ```json
    {
      "mcpServers": {
        "apcore": {
          "command": "apcore-mcp",
          "args": ["--extensions-dir", "/absolute/path/to/your/extensions"]
        }
      }
    }
    ```

### Cursor / Claude Code / Windsurf

Add a `.mcp.json` or equivalent in your project root:

```json
{
  "mcpServers": {
    "apcore": {
      "command": "apcore-mcp",
      "args": ["--extensions-dir", "./extensions"]
    }
  }
}
```

## Next Steps

- Explore the [Technical Design](tech-design-apcore-mcp.md) for architecture details.
- See how [Approval Mechanisms](tech-design-apcore-mcp.md#612-approval-handler-f-028) work for sensitive tools.
- Learn about [JWT Authentication](tech-design-apcore-mcp.md#611-jwt-authentication-f-027) for remote access.
