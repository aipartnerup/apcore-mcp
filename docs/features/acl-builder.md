---
description: "ACL Builder feature spec: constructs an apcore ACL from the mcp.acl Config Bus section, including apcore 0.29.0's PROTOCOL_SPEC §6.2.1 pattern-array shape closure, the bridge/apcore division of validation labour, error-type contract, and the tier-2 never-matches startup diagnostic."
---

# ACL Builder

> Feature spec for code-forge implementation planning.
> Source: extracted from CHANGELOG 0.19.0 (`mcp.acl` `approval` contract) and 0.20.0
> (apcore 0.29.0 pattern-array shape closure, PROTOCOL_SPEC §6.2.1 / spec v1.31.0-v1.33.0).
> Created: 2026-09-05

## Purpose

The ACL Builder turns the declarative `mcp.acl` section of apcore's Config Bus into a live
`apcore.ACL` instance that the bridge attaches to its `Executor`. It is the only route by which an
apcore-mcp deployment gets an access-control policy without writing code: a YAML block in
`apcore.yaml` (or an `acl=` constructor argument) becomes the policy every `tools/call` and
`resources/read` is evaluated against.

It exists because the MCP transport carries no caller identity. Every call arrives with
`caller_id = null`, which apcore normalizes to the synthetic identity `@external` — so `callers`
patterns cannot distinguish a human at a console from an autonomous agent, and the distinction has
to be made in `conditions` against JWT-derived claims. The builder is where that shape is validated
and where a misconfiguration is caught, at startup, before the server accepts its first request.

## Scope

**Included:**

- `build_acl_from_config(acl_config)` / `buildAclFromConfig(aclConfig)` /
  `build_acl_from_config(acl_config)` (Rust) — the single public entry point.
- The `mcp.acl` Config Bus schema: `default_effect`, `rules[]`, and each rule's `callers`,
  `targets`, `effect`, `description`, `conditions`, `approval`.
- Validation the **bridge** owns: Config-Bus-shaped faults that apcore never sees, reported with a
  `mcp.acl.rules[i]` prefix.
- The delegation boundary to apcore's own three doors (`ACL.load`, `ACLRule(...)`,
  `ACL(rules=[...])` / `add_rule`) and the error type each raises.
- The **tier-2 startup diagnostic**: reporting rules that load cleanly and protect nothing.

**Excluded:**

- ACL *evaluation* — matching, first-match-wins ordering, condition handlers, the UNEVALUABLE
  backstop. All of that is apcore's (PROTOCOL_SPEC §6.1-§6.5); the bridge only constructs.
- Discovery of an `acl/` directory. `ACL.discover()` is apcore's; the bridge reads `mcp.acl` only.
- The `system.*` management surface's own shape — see
  [System Management Extension](./system-management-extension.md).
- Authentication. Whether a JWT is present and valid is [JWT Authenticator](./jwt-authenticator.md);
  the claims it produces are what `conditions` matches against.

## Core Responsibilities

1. **Absence is not a policy.** A falsy `acl_config` returns `None` — no ACL is attached. This is
   indistinguishable from "no ACL was ever configured", and with `sys_modules.enabled: true` it
   means every caller reaches every `system.*` module including `system.control.*`. The builder
   MUST NOT invent a default policy; the deployment guidance to always pair `sys_modules.enabled`
   with an `mcp.acl` section belongs in the docs, not in a silent fallback.
2. **Fail loudly at startup.** Every malformed entry raises before `serve()` binds a port. A policy
   that half-loads is worse than one that refuses to load.
3. **Name the rule.** Every bridge-owned message carries the rule's index — `mcp.acl.rules[3]`.
   The index is the operator's only handle on a 20-rule YAML block, and apcore's own constructor
   messages do not carry one (see [Error surface](#error-surface-what-the-operator-actually-sees)).
4. **Delegate the rule semantics.** The bridge does not re-implement apcore's rule model. Where a
   constraint originates in `ACLRule` / `ACL`, the bridge passes the value through and lets apcore
   refuse it, so the two cannot drift.

## The `mcp.acl` schema

```yaml
mcp:
  acl:
    default_effect: deny
    rules:
      # Rule 1 — read-only management surface.
      # MUST precede the catch-all deny: evaluation is first-match-wins.
      - callers: ["@external"]
        targets: ["system.health.*", "system.usage.*", "system.manifest.*"]
        effect: allow
        conditions:
          identity_types: ["human"]
          roles: ["apcore.admin"]
        description: "Console read access to the management surface"

      # Rule 2 — administration. ACL allow is not execution:
      # system.control.* declares requires_approval=true and still passes the approval gate.
      - callers: ["@external"]
        targets: ["system.control.*"]
        effect: allow
        conditions:
          identity_types: ["human"]
          roles: ["apcore.admin"]
        description: "Administration; requires_approval still applies"

      # Rule 3 — catch-all deny. MUST be last.
      - callers: ["@external"]
        targets: ["system.*"]
        effect: deny
        description: "Block all other access to system modules"
```

Two mechanics make the `conditions` blocks load-bearing rather than decorative:

- **`caller_id` is always null over MCP.** apcore normalizes it to `@external` before matching
  `callers`. A `callers` pattern can therefore never separate a human from an agent — both arrive
  as `@external`. Only `conditions`, evaluated against the authenticated JWT's `identity_types` /
  `roles` claims, can.
- **First-match-wins.** The first rule whose `callers` / `targets` / `conditions` all match decides
  the effect; no later rule is consulted. A narrower `allow` MUST precede a broader `deny` that
  would otherwise shadow it.

!!! warning "There is no `sys.` namespace"
    The canonical IDs are `system.health.*` / `system.usage.*` / `system.manifest.*` /
    `system.control.*`. A **deny** rule written against `sys.*` never matches, so it silently
    blocks nothing — the dangerous direction. Fixed in 0.19.0
    (aiperceivable/apcore-mcp#14); repeated here because a copied rule is how it spread.

---

## Pattern-array shape closure (apcore 0.29.0)

apcore 0.29.0 closes the shape of a `callers` / `targets` array at **every entry point** —
file loading, direct construction and runtime insertion alike (PROTOCOL_SPEC §6.2.1, spec
v1.31.0, [apcore#112](https://github.com/aiperceivable/apcore/issues/112)). This is the single
largest change the 0.29.0 floor brings to `mcp.acl`, and it is **breaking for any deployment
carrying one of the affected shapes**.

### Why it changed

Through apcore 0.28.0 a `callers` / `targets` of `[]`, `["$or"]` or `["$not"]` could never match,
and all three SDKs' matchers returned `false` — reading an *arity fault* as a *scope decision*.
The rule was inert: the outcome tracked `default_effect` exactly, and `validate_rules()` reported
nothing.

On an `allow` rule that is merely useless. On a **`deny` rule under `default_effect: allow` it is a
fail-open**: the call the operator wrote the rule to block was permitted, by a rule that loaded
without error and a validator that called it clean. `mcp.acl` reaches this from plain YAML — the
builder's own `callers` / `targets` non-empty check catches `[]`, but nothing caught `["$or"]`,
`["$not"]`, or the wider-than-written multi-operand `$not`.

### What is now rejected

A pattern array is **flat**: index 0 is the only operator position, every later element is a plain
pattern string, the operators do not nest, and there is no precedence. Detection of a reserved
token is by **equality**, never by a `$` prefix — `$orders.*` remains an ordinary pattern and
still loads.

| Shape | Through 0.28.0 | From 0.29.0 |
|---|---|---|
| `["api.*"]`, `["$or", "a", "b"]`, `["$not", "banned.*"]` | accepted | accepted (unchanged) |
| `["$or", "a"]` — one operand | accepted | accepted — `$or` requires *at least* one, not two |
| `[]` | matched nothing; rule inert | **rejected** |
| `["$or"]` | matched nothing; rule inert | **rejected** |
| `["$not"]` | matched nothing; rule inert | **rejected** |
| `["$not", p1, p2, …]` | read as `["$not", p1]`; **wider than written** | **rejected** |
| `["a", ""]` — an empty pattern string | matched nothing | **rejected** |
| `["$or", "$not", "a"]` — reserved token off index 0 | matched a module literally named `$not` | **rejected** |

### Migration

| Written | Meant | Rewrite |
|---|---|---|
| `targets: []` | "everything" | `targets: ["*"]` |
| `targets: []` | "nothing" | delete the rule — a rule that matches nothing is not a rule |
| `callers: ["$or"]` | — | same two readings as above; apply the same rewrite |
| `["$not", p1, p2]` | "not p1" (what it has been doing) | `["$not", p1]` |
| `["$not", p1, p2]` | `NOT (p1 OR p2)` | **no mechanical rewrite** — see below |

!!! danger "The multi-operand `$not` has no automatic migration"
    `["$not", p1]` preserves what the rule has actually been doing, but if `NOT (p1 OR p2)` was
    intended, a leading `deny` rule is **not** equivalent: a non-matching rule lets evaluation
    continue to later rules, and a `deny` ends it. The rewrite has to be done by hand, against
    the rule's position in the list. Migration tooling **MUST NOT** apply it automatically.

### Validation order is now normative

§6.2.1 fixes, for the first time, the order in which a rule bad on more than one axis is refused:

1. **`default_effect` first**, ahead of every rule and ahead of the file-level checks on the
   `rules` collection itself. It is not a rule and has no index.
2. Then, per rule, **rule index dominating**: `effect` → `approval` → `callers` / `targets`.
3. The two pattern fields count as **one axis**: the §6.1.4.1 *type* fault precedes the §6.2.1
   *shape* closure, and `callers` precedes `targets`.

Three implementations produced three different orders before this was written down, and one
produced two different answers through two of its own doors.

!!! bug "The bridge's own order is currently the reverse — this MUST be realigned"
    `build_acl_from_config` validates **`callers` → `targets` → `effect` → `approval`** in all
    three bridges. A rule wrong in both `effect` and `callers` is therefore reported for `callers`
    by the Config Bus door and for `effect` by apcore's — the same file, two different answers,
    depending on which door it reaches first.

    Required change: reorder the per-rule checks in `build_acl_from_config` to
    `effect` → `approval` → `callers` → `targets`, matching §6.2.1. The unknown-key check stays
    ahead of all four (it is a Config-Bus-shape fault with no apcore counterpart), and the
    `default_effect` check stays ahead of the rule loop, where it already is.

    Existing single-fault conformance cases cannot see this; a multi-fault case is added to pin
    it (see [Conformance](#conformance)).

### Tier 2: rules that load and protect nothing

Closing the arities does not exhaust the inert class. `["$not", "*"]` has perfectly legal arity,
exactly one operand, and matches nothing — the identical fail-open, reached through a
well-formed array. apcore reports these through `ACL.validate_rules()` as **findings**: they load,
they change no decision, and they are never rejected, because the predicate cannot be closed
without freezing the pattern language.

The MUST-detect minimum is a criterion, not an enumeration:

- an all-wildcard `$not` operand — `["$not", "*"]`, `["$not", "**"]` — negates a pattern matching
  every module ID, so the rule fires for nothing;
- a `targets` array whose every operand is `@external` — the caller-side sentinel that no module
  ID is ever equal to. Entirely legal in `callers`, which is what it is for.

!!! tip "New required startup diagnostic"
    No bridge calls `validate_rules()` today. From 0.20.0, when `build_acl_from_config` returns an
    ACL, `serve()` / `async_serve()` **MUST** call `validate_rules()` on it once the registry is
    assembled and log each finding at WARNING, naming the rule index and the field.

    It belongs next to the existing unprotected-control-surface warning
    (0.19.0, [MCP Server Factory](./mcp-server-factory.md)) and behaves the same way: prominent,
    non-fatal, and never a startup failure. `validate_rules()` also reports unregistered
    `conditions` handler keys, which is why it is not called at build time — handler registration
    legitimately happens after discovery.

---

## Error surface: what the operator actually sees

The builder constructs `ACLRule(**rule_kwargs)` and then `ACL(rules=..., default_effect=...)`.
Anything apcore refuses is raised from inside those two calls, not from the builder's own checks —
and **apcore's message names the type, not the rule**:

```
ACLRule has an invalid 'targets' (PROTOCOL_SPEC §6.2.1): '$not' at index 0 MUST be followed
by exactly one pattern, got 2. …
```

`where="ACLRule"` is correct for apcore — a rule under construction has no position yet, and
§6.2.1 forbids inventing one. It is useless to an operator holding a 20-rule YAML file. The
arity closure's entire remedy is its message; delivering it without the index throws that away.

!!! danger "The escaping error type is not `ValueError` in Python"
    `ACLRuleError` extends `apcore.errors.ModuleError`, which extends `Exception` — **not**
    `ValueError`. `build_acl_from_config`'s documented contract ("raises `ValueError` on malformed
    entries") is therefore false for every §6.2.1 fault, and a caller with
    `except ValueError:` around startup stops catching a class of misconfiguration it used to
    catch through the builder's own `[]` check.

**Required behaviour (all three SDKs).** `build_acl_from_config` MUST catch `ACLRuleError` around
the `ACLRule(...)` construction and around the `ACL(...)` construction, and re-raise it in the
builder's own error type with the `mcp.acl.rules[i]` prefix — `mcp.acl` (no index) for a fault
raised by the `ACL` constructor itself — preserving apcore's message verbatim after the prefix and
chaining the original as the cause (`raise ... from exc` / `cause` / `source`).

This is the same treatment the Rust bridge already applies (it wraps apcore's error as a Config
error naming `mcp.acl`); Python and TypeScript let it escape raw. The wrap makes the three
consistent and restores the documented error type.

### Division of labour

| Fault | Owner | Why |
|---|---|---|
| `mcp.acl` not a mapping; `rules` not a list; rule not an object; unknown rule key | **bridge** | Config-Bus shape. apcore never sees these — they exist only in the YAML projection. |
| `default_effect` outside `{allow, deny}` | **bridge**, ahead of every rule | apcore checks it too; the bridge's message is the Config-Bus-flavoured one, and §6.2.1 puts it first at every door. |
| `effect` outside `{allow, deny}` | **bridge** | Same. Pinned by `invalid_effect` in the fixture. |
| `approval` outside `{required, not_required}` | **bridge** | Pinned by `approval_unknown_value` (contract 1.1). A typo fails with a Config-Bus message instead of surfacing from deep inside apcore. |
| `callers` / `targets` missing, not a list, or empty | **bridge** | Pre-dates the closure and is pinned by `missing_callers` / `missing_targets`. Overlaps §6.2.1 clause 1 — deliberately kept, because the fixture pins the message. |
| Every other §6.2.1 shape fault — `["$or"]`, `["$not"]`, multi-operand `$not`, empty pattern string, reserved token off index 0 | **apcore**, wrapped by the bridge | One predicate, three languages, one source of truth. A bridge copy is a second value set that drifts — exactly how `effect` came to be checked in one door and not another. |
| `deny` + `approval: required` | **apcore**, wrapped by the bridge | §6.1.6 rule 2. Already delegated; unchanged. |
| Tier-2 never-matches findings | **apcore**, reported by the bridge at startup | Diagnostics, not enforcement (§6.1.3). |

---

!!! warning "A user-level apcore config can put a *foreign* ACL in force (apcore 0.30.0 §9.2.2)"
    The absence case above is not the only way an operator ends up with a policy they did not
    write. apcore 0.30.0 documents, and reproduces, a second one: `acl.root` resolves against the
    **configuration file's** directory, so a config at `~/.config/apcore/config.yaml` carrying
    `acl.root: ./acl` loads its policy from `~/.config/apcore/acl/` into **every project that user
    runs**, while the project's own `./acl/` is ignored. For a default-deny, explicitly-granted
    authorization system that is the inverse of the intent, and it is silent.

    Two consequences for this bridge:

    - `ACL.discover()` returning a non-`None` ACL is **not** evidence that the project's own `acl/`
      directory was read. Under §9.14 discovery tiers 6-7 it may be a different project's.
    - `mcp.acl` is unaffected — it arrives through the Config Bus `mcp` namespace, not through
      `acl.root` discovery — which makes an explicit `mcp.acl` section the *more* predictable of
      the two routes, not merely the more convenient one.

    apcore 0.30.0 changes no behaviour here: §9.2.2 is a deprecation phase and the 1.x line keeps
    the current bases exactly. The repair lands in apcore 2.0, when every relative path-typed value
    resolves against `Config.project_root`. Until then a deployment that cares can read
    `Config.project_root` and compare it to CWD — apcore 0.30.0 exposes it for exactly this.

## Interfaces

### Inputs

- **`acl_config`** — the `mcp.acl` value from the Config Bus, or the `acl=` constructor argument.
  `dict[str, Any] | None` (Python) / `Record<string, unknown> | null` (TypeScript) /
  `Option<&serde_json::Value>` (Rust).

### Outputs

- **`apcore.ACL | None`** / **`ACL | null`** / **`Option<ACL>`** — the constructed policy, or
  absence when no section is configured.

### Dependencies

- **apcore >= 0.30.0** — `ACL`, `ACLRule`, `ACLRuleError`, `ACL.validate_rules`. **0.29.0** is the
  correctness floor for everything on this page: on 0.28.0 the shapes in the table above load
  silently and the deployment believes it has a rule it does not have. **0.30.0** is required
  transitively (apcore-toolkit 0.11.1 pins it) and adds nothing this page depends on — its ACL-side
  effect is documentation only, noted below.

## Notes

- The builder is pure with respect to the registry: it never inspects module IDs, so a `targets`
  pattern that matches no registered module is not an error here. Tier-2 findings are about
  patterns that can match *no legal module ID at all*, which is a different and decidable question.
- ACL `allow` is not execution. A rule allowing `system.control.*` does not bypass the approval
  gate — `requires_approval` still applies, and an ACL rule may itself carry `approval: required`
  to demand one for a call the module does not ask for.

---

## Contract: build_acl_from_config

> Construct an `apcore.ACL` from a Config Bus `mcp.acl` mapping. Names per language: Python
> `build_acl_from_config(acl_config)`, TypeScript `buildAclFromConfig(aclConfig)`, Rust
> `build_acl_from_config(acl_config)`.

### Inputs

- acl_config: mapping | null, required — the `mcp.acl` section. Falsy (null, `{}`) means no ACL.
  - default_effect: string, optional, default `"deny"` — `"allow"` or `"deny"`.
  - rules: array, optional, default `[]` — each entry an object carrying `callers`, `targets`,
    `effect`, and optionally `description`, `conditions`, `approval`. No other key is accepted.

### Errors

- `ValueError` (Python) / `Error` (TypeScript) / `ConfigError` (Rust) — raised for every fault in
  the [division-of-labour table](#division-of-labour), including a §6.2.1 shape fault re-raised
  from `ACLRuleError`. Message is prefixed `mcp.acl.rules[i] ` for a rule-scoped fault and
  `mcp.acl ` for a section-scoped one, and preserves apcore's own text after the prefix when the
  fault originated there.
- The per-rule checks MUST run in §6.2.1 order — unknown keys, then `effect`, `approval`,
  `callers`, `targets` — with `default_effect` judged ahead of the rule loop.
- No error is raised for a rule that loads and matches nothing (tier 2); that is a startup WARNING.

### Returns

- On no section configured (`null` / `{}`): `None` / `null` / `None`.
- On success: an `ACL` carrying the rules in file order, with `default_effect` applied.

### Properties

- async: false
- thread_safe: true (constructs a fresh `ACL`; touches no shared state)
- pure: false (reads the Config Bus; logs at INFO on success)
- idempotent: true

---

## Contract: ACL startup validation

> Report rules that loaded cleanly and can protect nothing. Names per language: the call is
> apcore's `acl.validate_rules()` / `acl.validateRules()` / `acl.validate_rules()`; the bridge-side
> reporting has no separate public symbol and runs inside `serve()` / `async_serve()`.

### Inputs

- (no parameters) — operates on the `ACL` the executor was assembled with. Skipped when no ACL is
  attached.

### Errors

- No errors raised. A findings list is diagnostics, never enforcement (§6.1.3). An implementation
  MUST NOT refuse to start on a finding, and MUST NOT let an exception from `validate_rules()`
  abort startup — catch, log at WARNING, continue, exactly as the unprotected-control-surface
  check does for `governance_state()`.

### Returns

- Nothing. Side effect: one WARNING log line per finding, naming the rule index, the field
  (`callers` / `targets`), and apcore's reason phrase verbatim.

### Properties

- async: false
- thread_safe: true
- pure: false (logs)
- idempotent: true (repeated calls produce the same findings; the ACL is not mutated)

---

## Conformance

Shared fixture: `conformance/fixtures/acl_config.json`, **contract_version 1.1 → 1.2**, driven by
all three bridges as of 0.20.0. 16 new cases (11 `test_cases` and 21 `error_cases` in total). Every
rejection fragment below was verified against apcore 0.29.0's real `ACLRule` errors rather than
transcribed from the specification.

**Accepted (3)** — the boundaries an over-strict implementation gets wrong:

| Case | Asserts |
|---|---|
| `or_with_one_operand_accepted` | `["$or", "a"]` loads — *at least* one operand, not at least two |
| `not_with_one_operand_accepted` | `["$not", "banned.*"]` — the only legal `$not` arity |
| `dollar_prefixed_literal_accepted` | `["$orders.*"]` loads — detection is by equality, not a `$` prefix |

**Rejected (12)** — six shapes × the `callers` / `targets` mirror:

| Shape | Owner | Pinned fragments |
|---|---|---|
| `[]` | **bridge** (its own non-empty check predates the closure) | `mcp.acl.rules[0]`, `must be a non-empty list` |
| `["$or"]` | apcore, wrapped | `mcp.acl.rules[0]`, `at least one pattern` |
| `["$not"]` | apcore, wrapped | `mcp.acl.rules[0]`, `exactly one pattern` |
| `["$not", p1, p2]` | apcore, wrapped | `mcp.acl.rules[0]`, `exactly one pattern` |
| `["api.*", ""]` | apcore, wrapped | `mcp.acl.rules[0]`, `empty string` |
| `["$or", "$not", "a"]` | apcore, wrapped | `mcp.acl.rules[0]`, `reserved token` |

**Ordering (1)** — `effect_reported_before_callers`: a rule wrong on both `effect` and `callers`
must report **`effect`**. It additionally asserts the `callers` message is *absent*, which is what
separates "named the right axis" from "rejected something". Every other error case in the fixture
is single-fault and cannot see the realignment.

Two format additions ride along with the version bump, and are why it is a bump rather than an
append: `expected_error_substrings` (an array — assert every fragment) alongside the existing
single-string `expected_error_substring`, and `must_not_contain`. Without the array a shape case
could pin either the bridge's `mcp.acl.rules[i]` prefix or the axis, not both, and all twelve would
carry identical expectations.

The mirrors are not padding. §6.2.1 constrains `callers` and `targets` identically, and an
implementation that validates one field and infers the other is exactly what they exist to catch —
apcore's own `acl_pattern_arity.json` (41 cases) carries the same pairs for the same reason.

Each `expected_error_substring` pins the **bridge's** prefix and a stable fragment of apcore's
reason, never apcore's full sentence: the wording is apcore's to change, and a shared fixture that
pins it is pinning the wrong repository's text. The same reasoning already keeps `deny` +
`approval: required` out of the shared error cases, and it is also why the §6.1.4.1 **type** fault
is absent — a bare-string `callers` is unrepresentable in apcore-rust's `Vec<String>`, so no door
rejects it and apcore classifies the rule UNEVALUABLE at `check()` time instead.

## Known cross-language divergences

- **Error type.** Python raises `ValueError`, TypeScript `Error`, Rust a `ConfigError` naming
  `mcp.acl`. Rust already wraps apcore's `ACLRuleError`; Python and TypeScript must start to.
- **`description` storage.** Python and TypeScript store it as a required string (empty when
  omitted); Rust as `Option<String>` (`None` when omitted). Conformance cases assert rule counts
  and `default_effect`, not per-field reconstruction.
- **`ACLRule` is `#[non_exhaustive]` in Rust as of apcore 0.29.0.** The Rust bridge must construct
  rules through the builder/constructor path rather than a struct literal. Python and TypeScript
  are unaffected.

## See Also

- [System Management Extension](./system-management-extension.md) — the `system.*` surface these
  rules exist to protect.
- [MCP Server Factory](./mcp-server-factory.md) — the unprotected-control-surface startup warning
  the tier-2 diagnostic sits beside.
- [OpenAPI Backend](./openapi-backend.md) — module IDs derived from an OpenAPI document are what
  `targets` patterns must match; the derivation is stable but not under the operator's control.
- [JWT Authenticator](./jwt-authenticator.md) — the source of the `identity_types` / `roles` claims
  `conditions` matches against.
- apcore PROTOCOL_SPEC §6.1-§6.2 and `conformance/fixtures/acl_pattern_arity.json` (41 cases).
