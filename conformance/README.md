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
| `acl_config.json` | Config Bus `mcp.acl` loading | 6 | How a raw `mcp.acl` JSON value becomes an ACL — null/empty means no ACL, `default_effect`, and rule ordering. |
| `middleware_config.json` | Config Bus `mcp.middleware` loading | 6 | How a raw `mcp.middleware` array becomes a middleware chain — supported types, per-type defaults, and order preservation. |

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
