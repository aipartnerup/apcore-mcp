---
description: "Complete apcore-mcp configuration reference covering CLI arguments (--extensions-dir, --transport, --jwt-secret, --explorer) and the APCoreMCP programmatic API options."
---

# Configuration Reference

This page provides a detailed reference for all configuration options available in **apcore-mcp**, whether you are using the CLI or the programmatic API.

## CLI Arguments

The CLI allows you to launch an MCP server by pointing to an extensions directory.

| Argument | Default | Description |
|---|---|---|
| `--extensions-dir` | *(required)* | Path to apcore extensions directory. **Rust:** accepted, but builds an empty registry — see [Getting Started §2](getting-started.md) |
| `--transport` | `stdio` | Transport protocol: `stdio`, `streamable-http`, or `sse` |
| `--host` | `127.0.0.1` | Host for HTTP-based transports |
| `--port` | `8000` | Port for HTTP-based transports (1-65535) |
| `--name` | `apcore-mcp` | MCP server name (appears in client UI) |
| `--version` | package version | MCP server version string |
| `--log-level` | `INFO` | Logging level: `DEBUG`, `INFO`, `WARNING`, `ERROR` |
| `--explorer` | off | Enable the browser-based Tool Explorer UI (HTTP only) |
| `--explorer-prefix` | `/explorer` | Mount path for the Explorer UI |
| `--allow-execute` | off | Allow tool execution from the explorer UI |
| `--jwt-secret` | — | JWT secret key for Bearer token authentication |
| `--jwt-key-file` | — | Path to a PEM key file, for asymmetric algorithms |
| `--jwt-algorithm` | `HS256` | Signing algorithm to accept |
| `--jwt-audience` | — | Expected `aud` claim |
| `--jwt-issuer` | — | Expected `iss` claim |
| `--jwt-require-auth` | `true` | Require authentication. **The permissive form differs per language** — Python: `--no-jwt-require-auth`; TypeScript: `--jwt-permissive`; Rust: `--jwt-require-auth false` (it takes a value). `--no-jwt-require-auth` is a Python-only spelling |
| `--exempt-paths` | `/health,/metrics` | Comma-separated paths exempt from auth. Passing this **replaces** the default set rather than adding to it. `/usage` is deliberately not exempt |
| `--approval` | `off` | Approval mode: `elicit`, `auto-approve`, `always-deny`, `off` |
| `--strategy` | executor default | Execution strategy: `standard`, `internal`, `testing`, `performance`, `minimal` |
| `--output-format` | `json` | Built-in output format: `json`, `csv`, `jsonl` |
| `--observability` | off | Enable metrics + usage middleware and expose `/metrics` and `/usage` |
| `--async` / `--no-async` | on | Enable the Async Task Bridge meta-tools *(TypeScript spelling; Python and Rust enable it unconditionally)* |

## Config Bus (`apcore.yaml` / environment)

apcore-mcp registers an `mcp` namespace on apcore's Config Bus, so the same settings can come from
`apcore.yaml`, from environment variables prefixed `APCORE_MCP_`, or from the CLI. **Precedence is
caller-wins:** an explicit `serve()` argument or CLI flag beats the Config Bus, which beats the
built-in default.

```yaml
# apcore.yaml
mcp:
  transport: streamable-http
  host: 0.0.0.0
  port: 8000
  explorer: true
  require_auth: true
```

Equivalently: `APCORE_MCP_TRANSPORT=streamable-http`, `APCORE_MCP_PORT=8000`, and so on —
uppercase the key and prefix it with `APCORE_MCP_`.

| Key | Default | Notes |
|---|---|---|
| `transport` | `stdio` | `stdio`, `streamable-http`, `sse` |
| `host` | `127.0.0.1` | HTTP transports only |
| `port` | `8000` | HTTP transports only |
| `name` | `apcore-mcp` | Server name shown to clients |
| `log_level` | `null` | `DEBUG`, `INFO`, `WARNING`, `ERROR` |
| `validate_inputs` | `false` | Validate arguments against the input schema before dispatch |
| `explorer` | `false` | Enable the Tool Explorer UI |
| `explorer_prefix` | `/explorer` | Mount path for the Explorer |
| `require_auth` | `true` | Require authentication on HTTP transports |
| `middleware` | `[]` | Declarative middleware list; each entry is `{type: ..., ...kwargs}` |
| `acl` | `null` | Declarative ACL — `{default_effect: "deny"\|"allow", rules: [...]}`; `null` means allow all |
| `output_format` | `json` | **Rust only.** `json`, `csv`, `jsonl`. Python and TypeScript do not read this key from the Config Bus — set the formatter programmatically or with `--output-format` there |

The `mcp.pipeline` section is read separately to build a custom execution strategy; see the
`strategy` parameter and the Technical Design's YAML pipeline configuration section.

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
        rich_description=False,      # v0.15+: render Tool.description as
                                     # apcore-toolkit Markdown when toolkit
                                     # is installed (`pip install apcore-mcp[markdown]`).
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
      richDescription: false,     // v0.15+: render Tool.description as
                                  // apcore-toolkit Markdown when toolkit
                                  // is installed (declared under
                                  // `optionalDependencies`). Requires
                                  // `await MCPServerFactory.prepare()` at startup.
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
        // v0.15+: render Tool.description as apcore-toolkit Markdown.
        // Equivalent to Python's `rich_description=True` / TS `richDescription: true`.
        // .with_rich_description(true) is also available on MCPServerFactory directly
        // for callers wiring the factory manually.
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

    // Construct the executor backend (only BackendSource::Executor is functional as of v0.15.0).
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

!!! warning "`strict` has a different default in each language"
    Python defaults to `False`, TypeScript to `true`, and Rust takes it positionally with no default.
    Omitting it therefore gives you permissive schemas from Python and strict ones from the other
    two, for the same registry. Pass it explicitly.

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
      strict: true,                 // DEFAULT is true here — unlike Python
      tags: [],                     // Filter modules by tags
      prefix: "",                   // Filter modules by ID prefix
    });
    ```

=== "Rust"

    ```rust
    use apcore_mcp::{to_openai_tools, OpenAIToolsConfig};

    // The free function takes a config struct; strict has no implicit default.
    let tools = to_openai_tools(executor, OpenAIToolsConfig {
        embed_annotations: false,
        strict: true,
        tags: None,
        prefix: None,
    })?;

    // The method on APCoreMCP takes the two flags positionally instead:
    //   mcp.to_openai_tools(embed_annotations, strict)
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

    **As of v0.15.0**, Markdown-rendered tool *descriptions* are a first-class
    feature again — opt in via `rich_description=True` (Python),
    `richDescription: true` (TypeScript), or
    `MCPServerFactory::with_rich_description(true)` (Rust). When apcore-toolkit
    is installed (declared under `optionalDependencies` in TS / behind the
    `[markdown]` extra in Python / linked at build time in Rust), tool
    descriptions are rendered as canonical apcore-toolkit Markdown so LLMs
    pack more decision-relevant signal per token. The `output_formatter` /
    `outputFormatter` option above is unrelated — it targets module *output*,
    not tool description rendering.

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

### Phase A — inline elicitation

The handler asks the caller to approve, in-band, and blocks until the answer arrives. Suitable for
interactive MCP clients.

=== "Python"

    ```python
    from apcore_mcp import serve
    from apcore_mcp.adapters import ElicitationApprovalHandler

    handler = ElicitationApprovalHandler()
    serve(registry, approval_handler=handler)   # serve() is synchronous in Python — do not await it
    ```

=== "TypeScript"

    ```typescript
    import { serve, ElicitationApprovalHandler } from "apcore-mcp";

    const handler = new ElicitationApprovalHandler();
    await serve(registry, { approvalHandler: handler });
    ```

=== "Rust"

    ```rust
    use apcore_mcp::{APCoreMCP, ElicitationApprovalHandler};
    use std::sync::Arc;

    let mcp = APCoreMCP::builder()
        .backend("./extensions")
        .approval_handler(Arc::new(ElicitationApprovalHandler::new(None)))
        .build()?;
    mcp.serve()?;
    ```

### Phase B — storage-backed approvals

Phase A needs the caller to stay connected while a human decides. Phase B does not: the call returns
immediately with an `APPROVAL_PENDING` response carrying an `approval_id`, the decision is recorded
out-of-band, and the agent polls until it resolves. Pass an `approval_store` — and optionally an
`approval_notify` callback that fires when a request is created, so you can push it to Slack, email,
or a review queue.

=== "Python"

    ```python
    from apcore_mcp import serve, InMemoryApprovalStore

    store = InMemoryApprovalStore()

    # The callback is awaited — it must be async, or return an awaitable.
    async def notify(approval_id: str, module_id: str, arguments: dict) -> None:
        print(f"approval {approval_id} requested for {module_id}")

    serve(registry, approval_store=store, approval_notify=notify)
    ```

=== "TypeScript"

    ```typescript
    import { serve, InMemoryApprovalStore } from "apcore-mcp";

    const store = new InMemoryApprovalStore();

    await serve(registry, {
      approvalStore: store,
      // Returns Promise<void> — declare it async even when the body is synchronous.
      approvalNotify: async (approvalId, moduleId, args) => {
        console.error(`approval ${approvalId} requested for ${moduleId}`);
      },
    });
    ```

=== "Rust"

    ```rust
    use apcore_mcp::{APCoreMCP, InMemoryApprovalStore};
    use std::sync::Arc;

    let mcp = APCoreMCP::builder()
        .backend("./extensions")
        .approval_store(Arc::new(InMemoryApprovalStore::default()))
        .approval_notify(|approval_id, module_id, _args| {
            eprintln!("approval {approval_id} requested for {module_id}");
        })
        .build()?;
    mcp.serve()?;
    ```

**The agent-side flow.** When a store is configured the bridge also exposes a meta-tool,
`__apcore_approval_check`:

1. The agent calls the gated tool. It comes back with an `APPROVAL_PENDING` response containing an
   `approval_id`.
2. A human resolves the request out-of-band — `store.resolve(approval_id, approved=True)`, or
   whatever your backend does.
3. The agent polls `__apcore_approval_check({"approval_id": "..."})`, which returns
   `{approval_id, status, reason?}` where `status` is `pending`, `approved`, or `rejected`.
4. Once `approved`, the agent retries the original tool call with `_meta.approvalId` set to that
   `approval_id`.

`InMemoryApprovalStore` is process-local and non-durable: pending records expire, resolved records
are kept only briefly, and everything is lost on restart. It is the right default for a single
process and the wrong one for anything horizontally scaled — implement the `ApprovalStore` interface
against Redis or a database for that. See
[Approval Handler (Phase B)](features/approval-phase-b.md) for the full contract, including the
Redis-backed pseudocode.

## Tool Explorer

The Tool Explorer is a built-in UI that allows you to interactively test your MCP tools. It is only available when using `streamable-http` or `sse` transports.

- **URL:** `http://<host>:<port>/explorer/`
- **Execution:** Set `allow_execute=True` (Python) or `allowExecute: true` (TypeScript) to enable the "Call Tool" button.
- **Auth:** If JWT is enabled, the UI provides a field to enter your Bearer token.
