# Configuration Reference

This page provides a detailed reference for all configuration options available in **apcore-mcp**, whether you are using the CLI or the programmatic API.

## CLI Arguments

The CLI allows you to launch an MCP server by pointing to an extensions directory.

| Argument | Default | Description |
|---|---|---|
| `--extensions-dir` | *(required)* | Path to apcore extensions directory |
| `--transport` | `stdio` | Transport protocol: `stdio`, `streamable-http`, or `sse` |
| `--host` | `127.0.0.1` | Host for HTTP-based transports |
| `--port` | `8000` | Port for HTTP-based transports (1-65535) |
| `--name` | `apcore-mcp` | MCP server name (appears in client UI) |
| `--version` | package version | MCP server version string |
| `--log-level` | `INFO` | Logging level: `DEBUG`, `INFO`, `WARNING`, `ERROR` |
| `--explorer` | off | Enable the browser-based Tool Explorer UI (HTTP only) |
| `--allow-execute` | off | Allow tool execution from the explorer UI |
| `--jwt-secret` | — | JWT secret key for Bearer token authentication |
| `--jwt-require-auth` | `true` | Require authentication (use `--no-jwt-require-auth` for permissive mode) |

## Programmatic API

### `APCoreMCP` (recommended)

The unified entry point — configure once, use everywhere.

=== "Python"

    ```python
    from apcore_mcp import APCoreMCP

    mcp = APCoreMCP(
        "./extensions",              # path, Registry, or Executor
        name="apcore-mcp",          # server name
        version=None,                # defaults to package version
        tags=None,                   # filter modules by tags
        prefix=None,                 # filter modules by ID prefix
        log_level=None,              # logging level
        validate_inputs=False,       # validate inputs against JSON Schema
        output_formatter=None,       # default: None (raw JSON). To opt into Markdown
                                     # formatting, install apcore-toolkit and pass
                                     # `output_formatter=to_markdown`.
        metrics_collector=None,      # MetricsExporter for /metrics
        authenticator=None,          # JWTAuthenticator instance
        require_auth=True,           # False = permissive mode
        exempt_paths=None,           # paths that bypass auth
        approval_handler=None,       # approval handler
    )

    # Launch as MCP server (blocking)
    mcp.serve(
        transport="stdio",           # "stdio" | "streamable-http" | "sse"
        host="127.0.0.1",
        port=8000,
        explorer=False,              # Enable Explorer UI
        allow_execute=False,         # Enable "Try it" in UI
    )

    # Export as OpenAI tools
    tools = mcp.to_openai_tools(strict=True, embed_annotations=False)

    # Embed into ASGI app
    async with mcp.async_serve(explorer=True) as app:
        ...

    # Inspect
    mcp.tools       # list of module IDs
    mcp.registry    # underlying Registry
    mcp.executor    # underlying Executor
    ```

=== "TypeScript"

    ```typescript
    import { APCoreMCP } from "apcore-mcp";

    const mcp = new APCoreMCP("./extensions", {
      name: "apcore-mcp",
      tags: undefined,
      prefix: undefined,
      logLevel: undefined,
      validateInputs: false,
      outputFormatter: undefined, // default: undefined - raw JSON
      authenticator: undefined,
      requireAuth: true,
    });

    // Launch as MCP server (blocking)
    await mcp.serve({
      transport: "stdio",
      host: "127.0.0.1",
      port: 8000,
      explorer: false,
      allowExecute: false,
    });

    // Export as OpenAI tools
    const tools = mcp.toOpenaiTools({ strict: true });

    // Inspect
    mcp.tools       // list of module IDs
    mcp.registry    // underlying Registry
    mcp.executor    // underlying Executor
    ```

=== "Rust"

    ```rust
    use apcore_mcp::{APCoreMCP, ServeOptions};

    let mcp = APCoreMCP::builder()
        .backend("./extensions")             // path, Arc<Registry>, or Arc<Executor>
        .name("apcore-mcp")                  // server name
        .version("1.0.0")                    // defaults to crate version
        .tags(vec!["public".into()])         // filter modules by tags
        .prefix("image")                     // filter modules by ID prefix
        .transport("streamable-http")        // "stdio" | "streamable-http" | "sse"
        .host("127.0.0.1")                  // host for HTTP transports
        .port(8000)                          // port for HTTP transports
        .validate_inputs(true)               // validate inputs against schemas
        .authenticator(auth)                 // Authenticator for JWT/token auth (HTTP only)
        .metrics_collector(collector)         // MetricsExporter for /metrics endpoint
        .output_formatter(formatter)         // custom result formatting
        .approval_handler(handler)           // approval handler for runtime approval
        .build()?;

    // Launch as MCP server (synchronous, blocking; spawns its own Tokio runtime).
    // Use `serve_with_options(ServeOptions { ... })` to pass on_startup/on_shutdown
    // hooks or the explorer config.
    mcp.serve()?;

    // Export as OpenAI tools
    let tools = mcp.to_openai_tools(false, true)?;

    // Inspect
    let tool_names = mcp.tools();     // list of module IDs
    let registry = mcp.registry();    // underlying Registry
    let executor = mcp.executor();    // underlying Executor
    ```

### `serve()` (function-based)

The function-based API is still fully supported for users who prefer it.

=== "Python"

    ```python
    from apcore_mcp import serve

    serve(
        registry_or_executor,
        transport="stdio",           # "stdio" | "streamable-http" | "sse"
        host="127.0.0.1",
        port=8000,
        name="apcore-mcp",
        explorer=False,              # Enable Explorer UI
        allow_execute=False,         # Enable "Try it" in UI
        validate_inputs=False,       # Validate inputs against JSON Schema
        log_level="INFO",
        authenticator=None,          # JWTAuthenticator instance
    )
    ```

=== "TypeScript"

    ```typescript
    import { serve } from "apcore-mcp";

    await serve(registryOrExecutor, {
      transport: "stdio",            // "stdio" | "streamable-http" | "sse"
      host: "127.0.0.1",
      port: 8000,
      name: "apcore-mcp",
      explorer: false,               // Enable Explorer UI
      allowExecute: false,           // Enable "Try it" in UI
      validateInputs: false,         // Validate inputs against JSON Schema
      logLevel: "INFO",
      authenticator: undefined,      // JWTAuthenticator instance
    });
    ```

=== "Rust"

    ```rust
    use std::sync::Arc;
    use apcore::{config::Config, executor::Executor};
    use apcore_mcp::{serve, ServeConfig};

    // Construct the executor backend (only BackendSource::Executor is functional in v0.14.0).
    let registry = Registry::new();
    // ... register modules ...
    let executor = Arc::new(Executor::new(registry, Config::default()));

    serve(executor, ServeConfig {
        transport: "streamable-http".into(),
        host: "127.0.0.1".into(),
        port: 8000,
        name: "apcore-mcp".into(),
        ..Default::default()
    })?;
    ```

### `async_serve()` Options

The `async_serve` function returns an embeddable ASGI application (Starlette) for mounting within a larger server. It accepts the same options as `serve()` except `transport`, `host`, and `port`.

=== "Python"

    ```python
    from apcore_mcp import async_serve

    async with async_serve(
        registry_or_executor,
        name="apcore-mcp",
        explorer=True,
        allow_execute=True,
        authenticator=None,
        approval_handler=None,
        output_formatter=None,       # Callable[[dict], str] or None (default: json.dumps)
    ) as app:
        # Mount `app` in your ASGI server (e.g., uvicorn)
        ...
    ```

=== "TypeScript"

    ```typescript
    import { asyncServe } from "apcore-mcp";

    const { handler, close } = await asyncServe(registryOrExecutor, {
      name: "apcore-mcp",
      explorer: true,
      allowExecute: true,
      authenticator: undefined,
      approvalHandler: undefined,
      outputFormatter: undefined,    // (result: object) => string, or undefined
    });
    // Use `handler` with http.createServer(), call `close()` when done
    ```

### `to_openai_tools()` Options

Converts apcore modules into OpenAI-compatible tool definitions.

=== "Python"

    ```python
    from apcore_mcp import to_openai_tools

    tools = to_openai_tools(
        registry_or_executor,
        embed_annotations=False,    # Append annotation hints to descriptions
        strict=False,               # Enable OpenAI Structured Outputs strict mode
        tags=None,                  # Filter modules by tags (list of strings)
        prefix=None,                # Filter modules by ID prefix
    )
    ```

=== "TypeScript"

    ```typescript
    import { toOpenaiTools } from "apcore-mcp";

    const tools = toOpenaiTools(registryOrExecutor, {
      embedAnnotations: false,      // Append annotation hints to descriptions
      strict: false,                // Enable OpenAI Structured Outputs strict mode
      tags: [],                     // Filter modules by tags
      prefix: "",                   // Filter modules by ID prefix
    });
    ```

## Output Formatting

By default, both `APCoreMCP` and the function-based `serve()` API leave tool output as raw JSON (`output_formatter=None`). To opt into Markdown formatting, install `apcore-toolkit` separately and pass `to_markdown` as the formatter.

**Behavior:**

- Dict results → formatted via `output_formatter` (default: `None` = raw JSON)
- Non-dict results → serialized with `json.dumps`
- If the formatter raises an error → falls back to `json.dumps` silently

=== "Python"

    ```python
    # Default: raw JSON (no formatter installed)
    mcp = APCoreMCP("./extensions")

    # Opt into Markdown via apcore-toolkit (optional dependency)
    # Install with: pip install apcore-toolkit
    from apcore_toolkit import to_markdown
    mcp = APCoreMCP("./extensions", output_formatter=to_markdown)

    # Custom formatter
    def my_formatter(result: dict) -> str:
        return yaml.dump(result)

    mcp = APCoreMCP("./extensions", output_formatter=my_formatter)
    ```

=== "TypeScript"

    ```typescript
    // Default: raw JSON
    const mcp = new APCoreMCP("./extensions");

    // Custom formatter (no built-in Markdown formatter — provide your own)
    const mcp = new APCoreMCP("./extensions", {
      outputFormatter: (result) => yaml.dump(result),
    });
    ```

!!! note
    Pre-0.10.0 builds defaulted `APCoreMCP` to Markdown via apcore-toolkit auto-wiring. CHANGELOG 0.10.0 removed apcore-toolkit as a required dependency and changed the default to `None` (raw JSON). Users wanting Markdown should install apcore-toolkit explicitly and pass `to_markdown`.

## Authentication (JWT)

For HTTP-based transports, you can secure your endpoints using JWT Bearer tokens.

=== "Python"

    ```python
    from apcore_mcp import APCoreMCP
    from apcore_mcp.auth import JWTAuthenticator

    auth = JWTAuthenticator(key="your-secret-key")
    mcp = APCoreMCP("./extensions", authenticator=auth)
    mcp.serve(transport="streamable-http")
    ```

=== "TypeScript"

    ```typescript
    import { APCoreMCP, JWTAuthenticator } from "apcore-mcp";

    const authenticator = new JWTAuthenticator({
      key: "your-secret-key",
    });
    const mcp = new APCoreMCP("./extensions", { authenticator });
    await mcp.serve({ transport: "streamable-http" });
    ```

## Approval Mechanism

You can gate destructive or sensitive tool calls behind user approval using the `--approval` CLI flag or the `approval_handler` parameter.

**CLI modes:**

| Mode | Behavior |
|------|----------|
| `elicit` | Prompts user via MCP elicitation (default for interactive clients) |
| `auto-approve` | Approves all requests automatically |
| `always-deny` | Denies all approval requests |
| `off` | Disables approval checks entirely |

=== "Python"

    ```python
    from apcore_mcp import serve
    from apcore_mcp.adapters import ElicitationApprovalHandler

    handler = ElicitationApprovalHandler()
    await serve(registry, approval_handler=handler)
    ```

=== "TypeScript"

    ```typescript
    import { serve, ElicitationApprovalHandler } from "apcore-mcp";

    const handler = new ElicitationApprovalHandler();
    await serve(registry, { approvalHandler: handler });
    ```

## Tool Explorer

The Tool Explorer is a built-in UI that allows you to interactively test your MCP tools. It is only available when using `streamable-http` or `sse` transports.

- **URL:** `http://<host>:<port>/explorer/`
- **Execution:** Set `allow_execute=True` (Python) or `allowExecute: true` (TypeScript) to enable the "Call Tool" button.
- **Auth:** If JWT is enabled, the UI provides a field to enter your Bearer token.
