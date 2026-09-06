# Cross-language conformance fixtures

These JSON files are the **behavioural contract** shared by the three apcore-mcp bridges. Each one
pins a mapping that must produce byte-identical results in `apcore-mcp-python`,
`apcore-mcp-typescript` and `apcore-mcp-rust`. They are data, not tests: every SDK loads them from
this directory and runs its own assertions against them, so a divergence surfaces as a failing test
in whichever bridge drifted rather than as a review comment.

Fixtures live in `fixtures/`. Nothing else in this directory is normative.

## What each fixture pins

| Fixture | Entry point | Cases | Pins |
|---|---|---|---|
| `tool_mapping.json` | `MCPServerFactory.build_tool(descriptor)` | 2 | apcore Module → MCP `Tool` (SRS §7.1). Basic field mapping, and that `x-` extensions survive into the MCP input schema. |
| `openai_tool_mapping.json` | `OpenAIConverter.convert_descriptor(descriptor, embed_annotations, strict)` | 4 | apcore Module → OpenAI function definition (SRS §7.2). Module-ID normalization (`.` → `-`), `x-llm-description` priority, `x-` stripping, and `mcp_` extras reaching the description. |
| `output_redaction.json` | `ExecutionRouter` output redaction | 2 | F-038. `x-sensitive` fields masked with `***REDACTED***`; unmarked fields left alone under an empty schema. |
| `acl_config.json` | Config Bus `mcp.acl` loading | 11 + 21 error | How a raw `mcp.acl` JSON value becomes an ACL — null/empty means no ACL, `default_effect`, rule ordering, the `approval` key's closed value set, and PROTOCOL_SPEC §6.2.1's closed pattern-array shape with its normative validation order (`contract_version` 1.2). |
| `system_surface.json` | `tools/list`, `resources/list`, `resources/templates/list` | 9 modules (3 tools + 3 resources + 3 templates), plus 6 `not_tools` negatives | Each canonical `system.*` module's exact MCP primitive and its name or URI, including the RFC 6570 `{?period}` suffix on `system.usage.module`. **Not case-shaped** — it is a declarative surface description with `tools` / `not_tools` / `resources` / `resource_templates` / `extension` keys rather than `test_cases`. Added in 0.19.0. |
| `middleware_config.json` | Config Bus `mcp.middleware` loading | 6 + 4 error | How a raw `mcp.middleware` array becomes a middleware chain — supported types, per-type defaults, and order preservation. |
| `openapi_backend.json` | `openapi_backend(spec, …)` | 9 + 3 config + 4 error | A scanner-derived module ID's projection into apcore's legal alphabet and onto the MCP tool name and OpenAI function name; HTTP-method → `ToolAnnotations`; the path-typed handling of `spec` (URL verbatim, relative resolved against `project_root`, empty discarded); and the four fatal configurations. See [`features/openapi-backend.md`](../docs/features/openapi-backend.md#conformance). |

### Driven by all three bridges

`acl_config.json` at `contract_version` 1.2 and `openapi_backend.json` are driven by
`apcore-mcp-python`, `apcore-mcp-typescript` and `apcore-mcp-rust` as of each bridge's 0.20.0.
Their expectations were computed by running the real upstreams — apcore 0.29.0's `ACLRule` errors
and apcore-toolkit 0.11.1's scanner — rather than transcribed from prose.

`openapi_backend.json` uses three sections rather than the usual two: `test_cases` (document →
modules), `config_cases` (how the `spec` value resolves), `error_cases` (fatal configurations).
Each has its own uniform shape, so a driver dispatches on the section rather than sniffing keys.

**A case must not defeat a driver's own technique.** `not_with_one_operand_accepted` names a literal
caller rather than `*` because the Rust driver probes `default_effect` by calling `check()` with a
pair it expects no rule to match — and `["$not", "banned.*"]` matches almost every module ID, so a
`*` caller would have fired the rule on the probe. Found by that driver failing.

**Mirror policy for shape cases.** Every `acl_config.json` case that pins a pattern-array *shape*
appears twice — once with the shape in `callers`, once in `targets`. §6.2.1 constrains the two
fields identically, and an implementation that validates one and infers the other is precisely the
defect the mirrors exist to catch. apcore's own `acl_pattern_arity.json` carries the same
`*_in_callers_*` / `*_in_targets_*` pairs for the same reason. Cases that pin *ordering* or
*acceptance* need no mirror.

**What a shared error case may pin.** The **bridge's** own prefix (`mcp.acl.rules[0]`) plus, via
`expected_error_names_field`, the **bare** name of the offending field — never a reason phrase
originating in apcore or apcore-toolkit. Measured, not assumed: for one §6.2.1 fault apcore-python
writes *"ACLRule has an invalid 'targets' (PROTOCOL_SPEC §6.2.1): '$not' at index 0 MUST be followed
by exactly one pattern, got 2"*, apcore-js writes *"Rule 0 'targets' has an illegal pattern-array
shape: '$not' at index 0 takes exactly one operand and this carries 2"*, and apcore-rust names the
offending element (`'callers[1]'`) rather than the field. An earlier draft of `acl_config.json` 1.2
pinned fragments read off apcore-python alone; two of six failed against apcore-js, and the quoted
field name then failed against apcore-rust. The bare token is the only spelling all three share.
This is the same rule that has always kept the `deny` + `approval: required` rejection out of the
shared error cases.

## Fixture format

Every file is a single JSON object:

```jsonc
{
  "description": "What this fixture pins, in prose.",
  "entry_point": "The function or method under test.",   // optional
  "spec": "srs-apcore-mcp.md#7.1",                        // optional, the normative source
  "test_cases": [ { "id": "...", "description": "...", /* inputs and expectations */ } ],
  "known_gaps": [ { "id": "...", "severity": "...", "status": "...", "description": "..." } ]
}
```

Input and expectation keys vary by fixture — they mirror the entry point's own parameters — so read
the file before writing a loader for it.

### `known_gaps`

A `known_gaps` entry records a divergence that is **deliberately not pinned** by a test case, because
pinning it would declare one language's behaviour correct when the disagreement lives upstream in
apcore itself. Each entry states what was measured in each language and where the real fix belongs.
Do not "fix" a bridge to match a known gap; fix it upstream, then delete the gap and add a case.

## Adding or changing a fixture

1. A fixture change is a **contract change**. Land it here first, then update all three SDKs — a
   fixture that only one bridge satisfies is worse than no fixture.
2. Keep case `id`s stable; SDK tests reference them in failure messages.
3. Give every case a `description` that says what would be wrong if it failed.
4. Prefer inputs with no incidental ambiguity. `output_redaction.json`'s
   `empty_schema_leaves_unmarked_fields_alone` deliberately uses field names with no secret
   connotation, because the three upstream libraries disagree about secret-sounding names — that
   disagreement is recorded as a `known_gap` instead.

## How each SDK loads them

All three walk up from the test file looking for `apcore-mcp/conformance/fixtures`, so a sibling
checkout of this repo is found automatically, as is the CI layout where the spec repo is checked out
inside the workspace. All three honour the same environment override —
**`APCORE_CONFORMANCE_FIXTURES`**, pointing at a fixtures directory — and each decides for itself
whether a missing directory is a skip (local dev) or a failure (CI):

| SDK | Loader |
|---|---|
| Python | `tests/conformance_fixtures.py` |
| TypeScript | `tests/conformance-fixtures.ts` |
| Rust | `conformance_fixture()` in the `src/server/router.rs` test module |

Rust's conformance assertions are inline `#[cfg(test)]` tests inside `src/`, not files under
`tests/` — a coverage audit that only looks at `tests/` will wrongly conclude Rust skips a fixture.

If you are implementing a fourth bridge, these five fixtures plus
[`docs/examples-spec.md`](../docs/examples-spec.md) are the compliance surface to target.
