---
description: "com.aiperceivable/management MCP extension (Phase A, unofficial): identifier, capabilities.extensions negotiation shape, resource/tool conventions for the nine canonical system.* modules, ACL/approval interaction, degradation behavior."
---

# System Management Extension (`com.aiperceivable/management`)

> Feature spec for code-forge implementation planning.
> Tracking issue: aiperceivable/apcore-mcp#16 (Phase A only — B and C are separate follow-ups).
> Blocked by / depends on: [`system.* module projection`](./mcp-server-factory.md#system-module-projection-resources-vs-tools) (aiperceivable/apcore-mcp#15).
> Status: Phase A — unofficial extension. Not submitted to `modelcontextprotocol/modelcontextprotocol`.
> Created: 2026-09-01

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** in this document are to be
interpreted as described in RFC 2119.

## Purpose

Publish the `system.*` management surface (health, usage, manifest, control) as a discoverable,
versioned MCP extension so a client can learn — from one field of the `initialize` handshake — that a
server exposes a known-shaped governance surface, instead of inferring it by pattern-matching
`module_id` strings.

This is [SEP-2133](https://modelcontextprotocol.io/seps/2133-extensions) **Phase A**: an *unofficial*
extension, which SEP-2133 states "may be introduced and governed by developers outside the MCP
organization" with no approval required. Phase B (Working Group discussion) and Phase C (formal SEP
submission) are tracked separately and are out of scope for this document.

## Identifier

`com.aiperceivable/management`

Format is `{vendor-prefix}/{extension-name}` per SEP-2133, where the vendor prefix is a reversed
domain the author controls. `aiperceivable.com` is the project domain, and every project in the
ecosystem documents under `<project>.aiperceivable.com`. SEP-2133 requires a *new* identifier for any
breaking change to this extension's shape, so this prefix and name **MUST** be treated as stable from
first publication — a future breaking revision ships under a new identifier, never a version bump of
this one.

## Design constraint — the extension is metadata, not a gate

An adapter **MUST NOT** let declaring or omitting this extension change what a client can reach.

- A client that does not declare `com.aiperceivable/management` in its own capabilities **MUST NOT**
  lose access to anything: every `system.*` resource and tool **MUST** remain reachable through
  ordinary `resources/list` / `resources/read` / `tools/list` / `tools/call`, subject only to
  Activation (§6.6.3 layer 1), ACL (layer 2) and Approval (layer 3) as defined by
  [PROTOCOL_SPEC §6.6.3](https://github.com/aiperceivable/apcore/blob/main/PROTOCOL_SPEC.md).
- An adapter **MUST NOT** introduce a permission switch keyed off whether the *caller* declared this
  extension. PROTOCOL_SPEC §6.6.3 prohibits adapters from inventing permission mechanisms beyond
  Activation / ACL / Approval; a capability-gated access difference would be exactly that — a shadow
  permission system that can diverge from the ACL's own decision.
- Declaring the extension only tells the client *that the shapes below are known and stable*. It is
  pure discovery.

## Negotiation

Advertised in the `initialize` response's `capabilities.extensions` map, keyed by identifier:

```json
{
  "capabilities": {
    "tools": { "listChanged": true },
    "resources": { "listChanged": true },
    "extensions": {
      "com.aiperceivable/management": {
        "surfaces": ["health", "usage", "manifest", "control"],
        "protocolVersion": "1.30.0"
      }
    }
  }
}
```

### `surfaces`

An array drawn from the four-element set `["health", "usage", "manifest", "control"]`, reflecting only
what is **actually reachable** on this server instance:

| Element | Present iff |
|---|---|
| `"health"` | `system.health.summary` or `system.health.module` is registered (`sys_modules.enabled = true`) |
| `"usage"` | `system.usage.summary` or `system.usage.module` is registered |
| `"manifest"` | `system.manifest.full` or `system.manifest.module` is registered |
| `"control"` | any `system.control.*` module is registered (`sys_modules.enabled = true` **and** `sys_modules.events.enabled = true`) |

An adapter **MUST NOT** advertise the extension at all — the whole `"com.aiperceivable/management"` key
**MUST** be absent from `capabilities.extensions` — when `surfaces` would be empty (i.e.
`sys_modules.enabled = false`, the default). An empty `surfaces` array is never sent; the key is
either present with ≥1 element or not present.

This mirrors [`GovernanceState`](https://github.com/aiperceivable/apcore) (`Executor.governanceState()`
/ `governance_state()`), specifically `controlModulesRegistered` for `"control"` and a per-prefix scan
of the registry for `"health"` / `"usage"` / `"manifest"` (`GovernanceState.readModulesRegistered` is
their disjunction and is not fine-grained enough on its own to populate `surfaces`).

### `protocolVersion`

The [PROTOCOL_SPEC](https://github.com/aiperceivable/apcore/blob/main/PROTOCOL_SPEC.md) version this
adapter's `system.*` projection was written against (currently `1.30.0`). A client MAY use this to
decide whether it understands the resource-URI and tool-annotation conventions below; an adapter
**MUST** update this string when it adopts a newer PROTOCOL_SPEC's `§6.6` changes.

## Resource and tool conventions for the nine canonical modules

This section restates, for the extension's own reference-implementation contract, the classification
[`system.* module projection`](./mcp-server-factory.md#system-module-projection-resources-vs-tools)
already establishes normatively. A client that recognizes this extension can rely on the following
shapes without probing:

| Module | MCP primitive | URI / name |
|---|---|---|
| `system.health.summary` | resource | `apcore://system.health.summary` |
| `system.health.module` | resource template | `apcore://system.health.module/{module_id}` |
| `system.usage.summary` | resource | `apcore://system.usage.summary{?period}` |
| `system.usage.module` | resource template | `apcore://system.usage.module/{module_id}{?period}` |
| `system.manifest.full` | resource | `apcore://system.manifest.full` |
| `system.manifest.module` | resource template | `apcore://system.manifest.module/{module_id}` |
| `system.control.update_config` | tool | `system.control.update_config` |
| `system.control.reload_module` | tool | `system.control.reload_module` |
| `system.control.toggle_feature` | tool | `system.control.toggle_feature` |

A `resources/read` on any `apcore://system.*` URI **MUST** be dispatched through the same execution
path (Activation → ACL → Approval → Executor pipeline) as a `tools/call` on the corresponding
module — an adapter **MUST NOT** add a second, resource-only invocation path that bypasses ACL or
audit logging. Concretely, an adapter routes the parsed URI's module ID and arguments through its own
tool-call dispatcher (e.g. `ExecutionRouter.handleCall` / `handle_call`), not through a raw
registry/module invocation.

## Interaction with ACL and Approval

Nothing here changes ACL or Approval semantics. The three read-only groups (`health`, `usage`,
`manifest`) are read-only, no-side-effect modules per PROTOCOL_SPEC §6.6.2 and are still subject to
ACL evaluation on every `resources/read`, exactly as they would be on a `tools/call` were they
projected as tools. The `control` group keeps its `requiresApproval` annotation and passes through the
Approval gate identically whether it is discovered via `tools/list` or invoked directly by a client
that already knows its name from this extension's contract.

Declaring this extension does **not** imply an ACL rule exists that permits reaching any of these
modules — see [`acl-builder`](../../README.md) rule-template guidance
(aiperceivable/apcore-mcp#14) for the ACL configuration that actually gates this surface.

## Degradation behavior

Per SEP-2133's graceful-degradation requirement, a client that does not understand this extension
simply does not see the `com.aiperceivable/management` key in `capabilities.extensions` and falls back
to core MCP discovery (`resources/list`, `resources/templates/list`, `tools/list`) to find the same
modules. No fallback code path is required on the client side because there is no behavior to fall
back *from* — the extension adds no protocol requirement, only a discovery shortcut.

## Security considerations

- This extension's presence or absence carries **no** authorization meaning, per the Design Constraint
  above. Treating it as one is a spec violation of this document, not a supported configuration.
- Advertising `"control"` in `surfaces` is a statement that `system.control.*` modules are
  *registered*, not that they are *protected*. Whether an unauthenticated or unauthorized caller can
  actually invoke one is entirely a function of the ACL/Approval layers. See the adapter-level startup
  warning for an unprotected control surface (aiperceivable/apcore-mcp#15(b),
  `GovernanceState.unprotectedControlSurface`) — that warning is orthogonal to this extension and
  fires regardless of whether any client ever negotiates it.
- `protocolVersion` is informational only; an adapter **MUST NOT** change its actual ACL/Approval
  enforcement based on a client's declared understanding (or lack thereof) of this extension.

## Acceptance (Phase A)

- [ ] All three adapters (Python, TypeScript, Rust) advertise `com.aiperceivable/management` in
      `initialize` when and only when `surfaces` would be non-empty.
- [ ] A client that does not declare the extension still reaches every management resource and tool
      (regression test — proves the extension is not acting as a gate).
- [ ] `surfaces` accurately reflects only the four groups actually registered on the running instance.

Phases B (Working Group discussion) and C (formal Extensions Track SEP against
`modelcontextprotocol/modelcontextprotocol`) are tracked by their own follow-ups when they start; see
aiperceivable/apcore-mcp#16 for the full three-phase plan and its gating requirements (a reference
implementation inside an official MCP SDK, and named Extension Maintainers, before SEP review can
begin).
