---
description: "Markdown Helpers feature spec: shared layer rendering rich Tool/function descriptions via apcore-toolkit render_module_markdown behind the rich_description option, with silent fallback if absent."
---

# Markdown Helpers

> Feature spec for code-forge implementation planning.
> Source: extracted from CHANGELOG 0.15.0 (Markdown tool descriptions; F-035-style cross-SDK rollout).
> Created: 2026-05-10

## Purpose

The Markdown helpers provide a shared formatting layer that both the MCP Server Factory and the OpenAI Converter use to render rich, structured tool descriptions through apcore-toolkit's `render_module_markdown`. They expose a small public API per language so callers can opt into Markdown rendering via the `rich_description` / `richDescription` option without depending on apcore-toolkit's internal layout, and so they can prime the renderer cache once at startup (TypeScript) to keep `buildTool` synchronous.

The helpers are intentionally thin: every public symbol either calls into apcore-toolkit when it is available or falls back silently (returning `None`/`null` or the raw `descriptor.description`) when it is not. There is no error path — toolkit unavailability is treated as a deployment choice, not a failure.

## Scope

**Included:**
- `is_available` / `isMarkdownAvailable` — detect at runtime whether apcore-toolkit's Markdown renderer is importable.
- `render_module_markdown(descriptor)` (Python+Rust) / `renderModuleMarkdownSync(descriptor)` (TS) — return rendered Markdown for a module, or `None`/`null` when toolkit is unavailable.
- `descriptor_to_scanned_module(descriptor)` (Python+Rust) — internal-but-public adapter from `ModuleDescriptor` to `ScannedModule` (the toolkit input type).
- `prime_markdown_toolkit()` / `primeMarkdownToolkit()` (TS-primary, no-op in Python+Rust) — pre-load the dynamically-imported renderer module so subsequent calls are synchronous.

**Excluded:**
- Implementation of the Markdown renderer itself (lives in apcore-toolkit; this feature only adapts to it).
- Schema-level rendering (handled by `SchemaConverter` and `OpenAIConverter` separately).
- Caching of rendered Markdown beyond the toolkit's own internal cache.

## Core Responsibilities

1. **Toolkit Detection** — `is_available` checks whether apcore-toolkit's Markdown renderer can be imported in this process. Result MUST be cached after the first successful resolution; subsequent calls do not re-import.
2. **Renderer Adapter** — `render_module_markdown` builds the `ScannedModule` view (`descriptor_to_scanned_module`) and forwards to apcore-toolkit's `render_module(...)` when available.
3. **Prime-on-Startup (TS)** — `primeMarkdownToolkit()` resolves the dynamic `import("apcore-toolkit/markdown")` Promise so that synchronous `renderModuleMarkdownSync(descriptor)` callers see a populated cache.

## Interfaces

### Inputs
- **ModuleDescriptor** — the apcore SDK descriptor type the helper renders.

### Outputs
- **str | None** (Python), **string | null** (TS), **Option<String>** (Rust) — rendered Markdown text, or absence when toolkit unavailable.

### Dependencies
- **apcore-toolkit** (Python: `apcore-toolkit`; TypeScript: `apcore-toolkit`; Rust: `apcore-toolkit` crate) — provides the Markdown renderer.

## Notes

- The helper layer is the only place in apcore-mcp that touches `apcore-toolkit`. This keeps the optional dependency edge auditable and avoids leaking toolkit types into other modules.
- All public helpers MUST be safe to call when toolkit is not installed — never raise, never log at ERROR.

---

## Contract: is_available

> Detect whether apcore-toolkit's Markdown renderer is importable in this process. Result is memoized for the lifetime of the process. Names per language: Python `is_available()`, TypeScript `isMarkdownAvailable()`, Rust `is_available()`.

### Inputs
- (no parameters)

### Errors
- No errors raised — import failures are caught and treated as "not available".

### Returns
- On first call: `bool` — `true` if the toolkit's Markdown module imports successfully, `false` otherwise. Result is cached.
- On subsequent calls: cached `bool`.

### Properties
- async: false
- thread_safe: true (idempotent memoization; concurrent first-call races resolve to the same value)
- pure: false (depends on the runtime module-resolution outcome)
- idempotent: true

---

## Contract: render_module_markdown

> Render a structured Markdown description for an apcore module via apcore-toolkit. Returns `None`/`null` when toolkit is not installed. Names per language: Python `render_module_markdown(descriptor)`, TypeScript `renderModuleMarkdownSync(descriptor)` (sync) and `renderModuleMarkdown(descriptor)` (async — see Notes), Rust `render_module_markdown(descriptor)`.

### Inputs
- descriptor: ModuleDescriptor, required — the module's full descriptor (id, description, metadata, annotations, input_schema, output_schema)

### Errors
- No errors raised — when the toolkit raises during render, the helper logs at WARNING and returns `None`/`null` (callers fall back to `descriptor.description`).

### Returns
- On success (toolkit available): `str | None` (Python) / `string | null` (TS) / `Option<String>` (Rust) — rendered Markdown text
- On toolkit unavailable: `None` / `null` / `None` — caller falls back to raw description
- On render failure: `None` / `null` / `None` (logged WARNING; never raises)

### Properties
- async: false (sync variants — Python, Rust, and TS `renderModuleMarkdownSync`)
- thread_safe: true
- pure: false (depends on toolkit availability)
- idempotent: true

### Notes
- TypeScript only: an async variant `renderModuleMarkdown(descriptor)` exists that awaits the dynamic toolkit import before rendering. Prefer the sync `renderModuleMarkdownSync` after a `primeMarkdownToolkit()` call at startup. (As of 0.15.0 the async variant has zero internal callers and may be removed in a future release — see audit D9-002.)

---

## Contract: descriptor_to_scanned_module

> Internal-but-public adapter that builds the `ScannedModule` value the toolkit consumes from an apcore `ModuleDescriptor`. Names per language: Python `descriptor_to_scanned_module(descriptor)`, Rust `descriptor_to_scanned_module(descriptor)`. Not exposed in TypeScript (the TS path constructs the toolkit input inline).

### Inputs
- descriptor: ModuleDescriptor, required (duck-typed) — must expose `id`, `description`, `metadata`, `annotations`, `input_schema`, `output_schema`

### Errors
- No errors raised — missing optional fields default to empty values:
  - missing `description` → `""`
  - missing `metadata` → `{}` / `Map::new()`
  - missing `annotations` → empty annotation set
  - missing `input_schema` → empty object

### Returns
- On success: `ScannedModule` value with fields populated from the descriptor

### Properties
- async: false
- thread_safe: true
- pure: true
- idempotent: true

### Visibility note
Although `pub` / exported, this helper is intended for internal apcore-mcp use. Demoting it to `pub(crate)` (Rust) / removing from `__all__` (Python) is tracked as D9-013 and will not break the public surface. External consumers should call `render_module_markdown(descriptor)` directly.

---

## Contract: prime_markdown_toolkit

> Pre-load the apcore-toolkit Markdown module so that subsequent synchronous render calls do not pay the dynamic-import cost. **Required call site (TypeScript):** application startup, before any synchronous `buildTool`/`convertDescriptor` invocation that uses `richDescription: true`. **Python and Rust:** the helper is exposed for API parity but is a no-op (toolkit imports are synchronous in those languages and resolve eagerly on first `is_available()` / `render_module_markdown()` call).

### Inputs
- (no parameters)

### Errors
- No errors raised — toolkit-load failures are caught and logged at WARNING. Subsequent `render_module_markdown` calls return `None`/`null` (fallback path).

### Returns
- TypeScript: `Promise<void>` — resolves once the dynamic import has settled (success or fallback)
- Python / Rust: `None` / `()` — synchronous no-op for parity

### Properties
- async: true (TS); false (Python, Rust)
- thread_safe: true (idempotent prime; concurrent calls share the same in-flight Promise in TS)
- pure: false (mutates a module-level cache in TS)
- idempotent: true

### Usage (TypeScript)
```ts
await MCPServerFactory.prepare(); // re-exposed alias of primeMarkdownToolkit
// or
await primeMarkdownToolkit();
// then later:
const tools = factory.buildTools(registry, { richDescription: true });
```
