---
description: "Schema Converter feature spec: inlines $ref/$defs from apcore Pydantic JSON Schemas into self-contained MCP inputSchema/OpenAI parameters with root object normalization and 32-level cycle detection."
---

# Schema Converter

> Feature spec for code-forge implementation planning.
> Source: extracted from apcore-mcp/docs/tech-design-apcore-mcp.md
> Created: 2026-04-06

## Purpose

The Schema Converter is responsible for transforming apcore JSON Schema definitions (produced by Pydantic) into clean, compatible schemas suitable for MCP `inputSchema` and OpenAI `parameters`. It ensures that all internal references are resolved and that the resulting schema follows the strict requirements of AI tool-use protocols.

## Scope

**Included:**
- Conversion of `input_schema` and `output_schema` from apcore `ModuleDescriptor`.
- Inlining of `$defs` and `$ref` references to create self-contained schemas.
- Guaranteeing that root-level schemas have `type: object`.
- Validation and cycle detection during reference resolution.
- Support for all standard JSON Schema types.

**Excluded:**
- External file reference resolution (only handles local `$defs` references).
- Domain-specific schema validation beyond the protocol requirements.
- Modification of original `ModuleDescriptor` data.

## Core Responsibilities

1. **Reference Inlining** — Recursively resolves and replaces `$ref` nodes with their actual definitions from `$defs` or `definitions`.
2. **Root Normalization** — Ensures every tool input schema has a clear `type: object` and properties mapping, even if the source is empty.
3. **Cycle Detection** — Prevents infinite recursion by tracking visited reference paths and enforcing a maximum recursion depth (32 levels).
4. **Clean-up** — Removes `$defs` and `definitions` keys from the final schema after successful inlining.

## Interfaces

### Inputs
- **ModuleDescriptor** (apcore Registry) — Contains the raw `input_schema` and `output_schema` dicts.

### Outputs
- **JSON Schema Dict** (MCP/OpenAI Converters) — A self-contained, inlined schema dictionary.

### Dependencies
- **apcore SDK (language-equivalent: apcore-python / apcore-js / apcore Rust crate)** — Provides the `ModuleDescriptor` structure and raw schema data.

## Data Flow

```mermaid
graph LR
    A[ModuleDescriptor] --> B[Extract Raw Schema]
    B --> C[Resolve $ref and Inline $defs]
    C --> D[Normalize Root Object]
    D --> E[Inlined Schema Output]
```

## Key Behaviors

### $ref Inlining Algorithm
The converter walks the schema tree recursively. When it encounters a `{"$ref": "#/$defs/Name"}` node, it looks up "Name" in the schema's `$defs` section, deep-copies the definition, recursively resolves any nested refs within that copy, and replaces the `$ref` node with the resolved result.

### Root Object Guarantee
If a schema is empty (`{}`), it is normalized to `{"type": "object", "properties": {}}`. If it lacks a `type` but has `properties`, `type: object` is added.

### Output Schema Handling
`output_schema` is converted for use in structured tool results. Unlike `input_schema`, an empty `output_schema` results in an empty dict `{}` as no input parameters are expected.

## Constraints

- **Recursion Limit**: Maximum of 32 levels of nesting to prevent stack overflow.
- **Bijective Resolution**: Local references must exist in the schema's own `$defs` section.
- **Immutability**: The converter must operate on copies and never mutate the source `ModuleDescriptor` data.

## Error Handling

- **Circular Reference**: Raises `ValueError` with the path of the cycle (e.g., "Circular reference: A -> B -> A").
- **Missing Definition**: Raises `KeyError` if a `$ref` points to a missing key in `$defs`.
- **Max Depth Exceeded**: Raises `ValueError` if the 32-level recursion limit is reached.

## Strict Mode for MCP

The Schema Converter exposes a `strict` option that tightens the converted JSON Schema for MCP clients without breaking MCP's permissive posture.

### Per-SDK Defaults
- Python (`SchemaConverter(strict=True)`) — default `True`.
- TypeScript (`convertInputSchema(descriptor, { strict })`) — option is optional and defaults to `false` at the converter level; the MCP Server Factory opts in with `strict: true` when building tools.
- Rust (`SchemaConverter::convert_input_schema_strict(schema, strict)`) — strict is an explicit parameter; the default constructor path uses strict mode, and the factory invokes the strict variant.

### What Strict Injects
When enabled, strict recursively walks the schema and sets `additionalProperties: false` on every JSON Schema node that is one of:
- `type == "object"`,
- a `type` array containing `"object"`,
- or a schema that declares `properties` without an explicit `type`.

### What Strict Does NOT Do
Unlike apcore's core `to_strict_schema`, the MCP-side strict mode does **not** force every property to be `required` and does **not** rewrite optional fields to be `nullable`. This preserves MCP's permissive posture (optional fields stay optional, absent fields stay absent) while still blocking unknown keys.

### Preservation Rules
User-set `additionalProperties` values (whether `true`, `false`, or a subschema) are always preserved — strict mode never overwrites an existing declaration. Injection only occurs when the key is absent.

### Implementation References
- Python: `src/apcore_mcp/adapters/schema.py` (`_inject_additional_properties_false`).
- TypeScript: `src/adapters/schema.ts` (`ConvertSchemaOptions.strict`).
- Rust: `src/adapters/schema.rs` (`inject_strict`).

## Notes

- This component is critical for compatibility with MCP clients (like Claude Desktop) and OpenAI's API, which often struggle with unresolved JSON Schema references.
- It reuses patterns from apcore's internal `RefResolver` but is optimized for the specific constraints of tool-use protocols.

---

## Contract: SchemaConverter.convert_input_schema

### Inputs
- descriptor: Any, required (duck-typed) — must have `input_schema` attribute containing a dict; NOT a raw schema dict or JSON Value

### Errors
- ValueError — when a circular `$ref` is detected (e.g., "Circular $ref detected: #/$defs/A")
- ValueError — when the 32-level recursion depth limit is exceeded
- KeyError — when a `$ref` points to a missing key in `$defs`

### Returns
- On success: dict[str, Any] — self-contained, inlined JSON Schema; all `$ref` nodes replaced; `$defs` removed from output; root always has `type: "object"`
- Empty schema `{}` → `{"type": "object", "properties": {}, "additionalProperties": false}` (when strict=True)
- Schema with properties but no type → `type: "object"` added
- Deep copy of source — never mutates the original descriptor
- When `strict=True` (default): `additionalProperties: false` injected on every object-typed node that lacks it; existing user-set values preserved

### Properties
- async: false
- thread_safe: true
- pure: true
- idempotent: true

---

## Contract: SchemaConverter._inject_strict

### Inputs
- node: Any, required — JSON Schema node (dict, list, or primitive); non-dict values are no-ops

### Errors
- No exceptions raised

### Returns
- On success: None — in-place mutation: sets `additionalProperties: false` on every object-typed subschema that doesn't already define the key
- Canonical subschema-keyword recursion set (all three SDKs MUST walk all of these): `items`, `additionalProperties`, `not`, `if`, `then`, `else`, `contains`, `propertyNames`, `oneOf`, `anyOf`, `allOf`, `prefixItems`, `properties`, `patternProperties`, `$defs`, `definitions`
- User-set `additionalProperties` values (true, false, or subschema dict) are always preserved — injection only when key is absent
- Object detection: `type == "object"` OR `type` array contains `"object"` OR has `properties` without conflicting non-object scalar type

### Properties
- async: false
- thread_safe: true
- pure: false
- idempotent: true
