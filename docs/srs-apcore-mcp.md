# Software Requirements Specification: apcore-mcp

| Field       | Value                                                                    |
|-------------|--------------------------------------------------------------------------|
| Title       | apcore-mcp: Automatic MCP Server & OpenAI Tools Bridge                   |
| Document    | Software Requirements Specification (SRS)                                |
| Version     | 1.0                                                                      |
| Date        | 2026-02-15                                                               |
| Author      | aipartnerup Engineering Team                                             |
| Status      | Draft                                                                    |
| PRD Ref     | `docs/prd-apcore-mcp.md` v1.0                                           |
| Tech Design | `docs/tech-design-apcore-mcp.md` v1.0                                   |
| Standard    | IEEE 830 / ISO/IEC/IEEE 29148                                            |

---

## Revision History

| Version | Date       | Author                      | Description                  |
|---------|------------|------------------------------|------------------------------|
| 1.0     | 2026-02-15 | aipartnerup Engineering Team | Initial draft                |

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Overall Description](#2-overall-description)
3. [Specific Requirements -- Functional Requirements](#3-specific-requirements--functional-requirements)
4. [Specific Requirements -- Non-Functional Requirements](#4-specific-requirements--non-functional-requirements)
5. [Use Cases](#5-use-cases)
6. [CRUD Matrix](#6-crud-matrix)
7. [Data Dictionary](#7-data-dictionary)
8. [Interface Requirements](#8-interface-requirements)
9. [Traceability Matrix](#9-traceability-matrix)
10. [Appendix](#10-appendix)

---

## 1. Introduction

### 1.1 Purpose

This Software Requirements Specification defines the complete functional and non-functional requirements for **apcore-mcp**, an independent Python adapter package that automatically bridges any apcore Module Registry into both a fully functional MCP (Model Context Protocol) Server and OpenAI-compatible tool definitions. This document formalizes 20 of 25 features from the upstream PRD (F-001 through F-020) into traceable, testable requirements organized by component module. P2 features F-021 through F-025 are deferred to a future SRS revision. It serves as the authoritative reference for implementation, testing, and acceptance of apcore-mcp v0.1.0.

The intended audience includes software engineers implementing apcore-mcp, QA engineers writing test plans, and project stakeholders evaluating feature completeness.

> **Note:** FR-STREAM-001, FR-EXT-001, FR-EXT-002, and FR-EXT-003 are defined in this document but not yet included in the traceability matrix (Section 9). The effective FR count is 82.

### 1.2 Scope

apcore-mcp provides two primary capabilities:

1. **MCP Server Bridge**: A `serve(registry)` function that launches a standards-compliant MCP Server exposing all apcore modules as MCP tools, supporting stdio, Streamable HTTP, and SSE transports.
2. **OpenAI Tools Export**: A `to_openai_tools(registry)` function that returns a list of OpenAI-compatible tool definition dicts for use with OpenAI Function Calling APIs.

The system reads apcore module metadata (`input_schema`, `output_schema`, `description`, `annotations`) and generates correct tool definitions at runtime with zero per-module code. All MCP tool calls are routed through the apcore `Executor` pipeline, preserving ACL enforcement, input validation, middleware execution, and timeout enforcement.

apcore-mcp does NOT reimplement the MCP protocol (it uses the official `mcp` Python SDK), does NOT define modules (that is apcore-python's responsibility), does NOT implement domain-specific wrapping (that is the job of `xxx-apcore` projects), and does NOT build an OpenAI API client or agent runtime.

### 1.3 Definitions, Acronyms, and Abbreviations

| Term | Definition |
|------|------------|
| **apcore** | A schema-driven module development framework providing Registry, Executor, and schema validation for modular applications. |
| **apcore-python** | The Python SDK implementation of the apcore framework. Provides `Registry`, `Executor`, `ModuleDescriptor`, `ModuleAnnotations`, and the error hierarchy. |
| **MCP** | Model Context Protocol -- an open protocol introduced by Anthropic for connecting AI assistants to external tools and data sources. |
| **MCP Server** | A process that implements the MCP protocol and exposes tools, resources, and/or prompts to MCP clients. |
| **MCP Client** | An AI assistant or application that connects to MCP Servers to discover and invoke tools. Examples: Claude Desktop, Cursor, Windsurf. |
| **MCP Tool** | A callable function exposed by an MCP Server, defined by a name, description, and input schema. |
| **MCP Tool Annotations** | Metadata on an MCP tool describing behavioral characteristics: `destructiveHint`, `readOnlyHint`, `idempotentHint`, `openWorldHint`. |
| **OpenAI Function Calling / Tools** | OpenAI's mechanism for allowing language models to invoke external functions, defined with name, description, and parameters (JSON Schema). |
| **Strict Mode** | An OpenAI feature where `"strict": true` on a tool definition constrains the model to produce outputs exactly matching the schema. Requires `additionalProperties: false` and all properties marked required. |
| **Registry** | An apcore class that discovers, registers, and provides query access to modules. |
| **Executor** | An apcore class that orchestrates the module execution pipeline: context creation, safety checks, ACL, validation, middleware, execution, and result return. |
| **ModuleDescriptor** | An apcore dataclass containing a module's full metadata: `module_id`, `name`, `description`, `input_schema`, `output_schema`, `annotations`, `examples`, `tags`, `version`. |
| **ModuleAnnotations** | An apcore frozen dataclass with behavioral flags: `readonly` (bool, default False), `destructive` (bool, default False), `idempotent` (bool, default False), `requires_approval` (bool, default False), `open_world` (bool, default True). |
| **ModuleError** | Base exception class for all apcore framework errors. Contains `code`, `message`, `details`, `cause`, `trace_id`, and `timestamp` fields. |
| **Transport** | The communication mechanism between MCP client and server: stdio, Streamable HTTP, or SSE. |
| **stdio** | A transport where the MCP server reads from stdin and writes to stdout. |
| **Streamable HTTP** | The recommended HTTP-based MCP transport, replacing SSE as the primary network transport. |
| **SSE** | Server-Sent Events -- a deprecated MCP transport supported for backward compatibility. |
| **ACL** | Access Control List -- apcore's built-in mechanism for controlling which callers can invoke which modules. |
| **$ref inlining** | The process of resolving JSON Schema `$ref` references by substituting the referenced definition in-place and removing the `$defs` section. |
| **Module ID normalization** | Converting apcore module IDs (dot-notation, e.g., `image.resize`) to OpenAI-compatible function names (e.g., `image-resize`) by replacing `.` with `-`. |
| **xxx-apcore** | Convention for apcore adapter projects targeting specific domains: `comfyui-apcore`, `vnpy-apcore`, `blender-apcore`, etc. |
| **CallToolResult** | MCP SDK type representing the result of a tool call, containing `content` (list of content items) and `isError` (boolean). |
| **TextContent** | MCP SDK type for text-based content items within a CallToolResult. |
| **ToolAnnotations** | MCP SDK type for behavioral hints on tools: `read_only_hint`, `destructive_hint`, `idempotent_hint`, `open_world_hint`. |
| **JSON Schema** | A vocabulary for annotating and validating JSON documents (draft 2020-12 or compatible). |

### 1.4 References

| ID   | Reference | Description |
|------|-----------|-------------|
| REF-01 | `docs/prd-apcore-mcp.md` v1.0 | Product Requirements Document -- primary input for this SRS |
| REF-02 | `docs/tech-design-apcore-mcp.md` v1.0 | Technical Design Document -- architecture and API details |
| REF-03 | `ideas/apcore-mcp.md` | Original idea document with schema mapping and architecture context |
| REF-04 | apcore-python source (`apcore` package) | SDK source: Registry, Executor, ModuleDescriptor, ModuleAnnotations, error hierarchy |
| REF-05 | [MCP Specification](https://modelcontextprotocol.io/) | Official Model Context Protocol specification |
| REF-06 | [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk) | Official Python SDK for MCP servers and clients |
| REF-07 | [OpenAI Function Calling](https://platform.openai.com/docs/guides/function-calling) | OpenAI's guide to function calling / tools |
| REF-08 | IEEE 830-1998 | IEEE Recommended Practice for SRS |
| REF-09 | ISO/IEC/IEEE 29148:2018 | Systems and software engineering -- Life cycle processes -- Requirements engineering |

### 1.5 Overview

Section 2 provides the overall product description, including ecosystem context, user characteristics, constraints, and assumptions. Section 3 specifies all functional requirements organized by component module. Section 4 specifies non-functional requirements for performance, security, reliability, maintainability, compatibility, and portability. Section 5 defines use cases for major workflows. Sections 6-8 provide the CRUD matrix, data dictionary, and interface requirements. Section 9 contains the bidirectional traceability matrix. Section 10 is the appendix with glossary and API surface references.

---

## 2. Overall Description

### 2.1 Product Perspective

apcore-mcp is a thin adapter layer that sits between the apcore-python SDK and the AI agent ecosystem. It occupies a specific architectural position defined by apcore's SCOPE.md:

```
apcore-python (core SDK)
    |
    +-- Provides: Registry, Executor, ModuleDescriptor, ModuleAnnotations, error hierarchy
    |
apcore-mcp (this project)
    |
    +-- Consumes: Registry API, Executor.call() / call_async()
    +-- Produces: MCP Server (via mcp SDK), OpenAI tools list (pure dict conversion)
    |
    +-- MCP path --> Claude Desktop, Cursor, Windsurf, any MCP client
    +-- OpenAI path --> OpenAI API, Azure OpenAI, any OpenAI-compatible platform
```

apcore-mcp is the first adapter in a planned family (apcore-a2a is future). It depends on the `apcore` package for module metadata and execution, and on the `mcp` package for MCP protocol implementation. It has no dependency on the `openai` package at runtime.

### 2.2 Product Functions (High-Level Summary)

1. **Schema Conversion**: Convert apcore `input_schema` (JSON Schema with potential `$defs`/`$ref`) to clean schemas for MCP and OpenAI consumption.
2. **Annotation Mapping**: Map apcore `ModuleAnnotations` to MCP `ToolAnnotations` and optionally embed in OpenAI tool descriptions.
3. **MCP Server Launch**: Create and start an MCP Server via `serve()` with configurable transport (stdio, Streamable HTTP, SSE).
4. **Execution Routing**: Route MCP tool calls through the apcore `Executor` pipeline with full ACL, validation, and middleware.
5. **Error Mapping**: Translate apcore error hierarchy to structured MCP error responses.
6. **OpenAI Export**: Export registry modules as OpenAI-compatible tool definition dicts via `to_openai_tools()`.
7. **CLI Entry Point**: Provide `python -m apcore_mcp` for launching the server without writing Python code.
8. **Dynamic Registration**: Reflect runtime registry changes in the MCP tool list.
9. **Logging**: Structured logging for tool calls, errors, and server lifecycle events.

### 2.3 User Characteristics

| Persona | Role | Experience | Primary Interaction |
|---------|------|------------|---------------------|
| **Maya** (Module Developer) | Writes apcore modules, wants AI agent integration | 3+ years Python, familiar with Pydantic, new to MCP | Calls `serve(registry)` or `to_openai_tools(registry)` |
| **David** (xxx-apcore Project Dev) | Builds domain-specific apcore projects (comfyui-apcore, etc.) | 5+ years Python, expert in domain, familiar with apcore | Adds apcore-mcp as dependency, calls `serve()` with pre-configured Executor |
| **Alex** (AI Agent Builder) | Integrates external tools into Claude/GPT workflows | Familiar with MCP clients and OpenAI function calling | Connects MCP client to apcore-mcp server, or receives OpenAI tool dicts |

### 2.4 Constraints

| ID | Constraint | Rationale |
|----|-----------|-----------|
| C-01 | Must use official `mcp` Python SDK for MCP protocol implementation | MCP protocol is complex; reimplementation is out of scope and error-prone |
| C-02 | Must use apcore `Executor` for all tool call routing | ACL, validation, and middleware guarantees must be preserved |
| C-03 | Core logic must not exceed 1,200 lines (excluding tests and docs) | Thin adapter design principle; complexity belongs in apcore-python and mcp SDK |
| C-04 | Python >= 3.10 required | Aligns with apcore-python minimum; needed for modern type hints and `match` |
| C-05 | No custom authentication layer | apcore Executor's ACL is the sole access control mechanism |
| C-06 | `to_openai_tools()` must have zero runtime dependency on `openai` package | Maximum interoperability; returns plain dicts |

### 2.5 Assumptions and Dependencies

| ID | Assumption/Dependency | Type |
|----|----------------------|------|
| A-01 | apcore-python SDK API (`Registry`, `Executor`, `ModuleDescriptor`, `ModuleAnnotations`) is stable and will not undergo breaking changes during development | Assumption |
| A-02 | The `mcp` Python SDK (>= 1.0.0) provides stable APIs for tool registration, transport configuration, and error handling | Assumption |
| A-03 | MCP clients (Claude Desktop, Cursor) correctly implement the MCP protocol for tool discovery and invocation | Assumption |
| A-04 | apcore modules produce JSON-serializable output dicts from their `execute()` methods | Assumption |
| A-05 | Python >= 3.10 is available in the target deployment environment | Assumption |
| D-01 | `apcore` package >= 0.2.0 | Dependency |
| D-02 | `mcp` package >= 1.0.0 | Dependency |
| D-03 | `openai` package (optional, not required at runtime) | Dependency |

---

## 3. Specific Requirements -- Functional Requirements

### 3.1 FR-SCHEMA: Schema Conversion Requirements

---

#### FR-SCHEMA-001: Convert ModuleDescriptor input_schema to MCP inputSchema

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-001 |
| **Title** | Convert apcore input_schema to MCP-compatible inputSchema |
| **Priority** | P0 |
| **Traces to** | F-001 |

**Description:** The `SchemaConverter.convert_input_schema()` method shall accept a `ModuleDescriptor` instance and return a JSON Schema dict suitable for use as an MCP Tool's `inputSchema`. The returned schema shall be a valid JSON Schema object with `"type": "object"` at the root level.

**Input/Trigger:** A `ModuleDescriptor` with a populated `input_schema` dict (as produced by Pydantic `model_json_schema()`).

**Expected Output:**
- A `dict[str, Any]` representing a valid JSON Schema.
- The root-level `"type"` key shall equal `"object"`.
- All `$ref` references within the schema shall be inlined (replaced with the referenced definition content).
- The `$defs` key (or `definitions` key) shall be removed from the returned schema.
- The `properties` key shall be preserved with all field definitions intact.

**Boundary Conditions:**
- Empty `input_schema` (`{}`): Return `{"type": "object", "properties": {}}`.
- Schema with no `"type"` key but with `"properties"` key: Add `"type": "object"` to the returned schema.
- Schema with `"title"` key: Preserve the `"title"` key in the returned schema.
- Schema with `"required"` key: Preserve the `"required"` array in the returned schema.

**Error Conditions:**
- Circular `$ref` (e.g., definition A references definition B which references definition A): Raise `ValueError` with a message identifying the circular reference path.
- `$ref` pointing to a non-existent definition key: Raise `KeyError` with the missing definition name.
- Recursion depth exceeding 32 levels during `$ref` resolution: Raise `ValueError` indicating maximum recursion depth exceeded.

---

#### FR-SCHEMA-002: Inline $defs/$ref references in input_schema

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-002 |
| **Title** | Resolve and inline all $defs-local $ref references |
| **Priority** | P0 |
| **Traces to** | F-001 |

**Description:** The `SchemaConverter._inline_refs()` method shall resolve all `$ref` nodes that reference `$defs`-local definitions (format: `"$ref": "#/$defs/DefinitionName"`) by deep-copying the referenced definition and substituting it in place of the `$ref` node. After inlining, the `$defs` key shall be removed from the root schema.

**Input/Trigger:** A JSON Schema dict containing `$defs` and `$ref` nodes, as produced by Pydantic v2's `model_json_schema()`.

**Expected Output:**
- A new dict (original not mutated) with all `$ref` nodes replaced by their resolved definitions.
- Nested `$ref` within definitions shall also be resolved recursively.
- The `$defs` key shall not appear in the returned dict.
- The `definitions` key (legacy format) shall also be handled and removed if present.

**Boundary Conditions:**
- Schema with zero `$ref` nodes: Return the schema unchanged (minus any empty `$defs`).
- Schema with deeply nested `$ref` (up to 32 levels): Resolve all levels.
- Schema with `$ref` in array items (`"items": {"$ref": "#/$defs/X"}`): Resolve correctly.
- Schema with `$ref` in `oneOf`, `anyOf`, `allOf` arrays: Resolve each branch.
- Schema with the same `$ref` used in multiple locations: Each location receives its own deep copy.

**Error Conditions:**
- Circular `$ref` chain: Raise `ValueError` with the cycle path (e.g., "Circular reference: A -> B -> A").
- Resolution depth exceeding 32: Raise `ValueError`.

---

#### FR-SCHEMA-003: Ensure root schema type is "object"

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-003 |
| **Title** | Guarantee root-level type: object in converted schemas |
| **Priority** | P0 |
| **Traces to** | F-001 |

**Description:** The `SchemaConverter._ensure_object_type()` method shall verify that the root schema has `"type": "object"`. If the schema is empty or lacks a `"type"` key, it shall add the appropriate keys.

**Input/Trigger:** A JSON Schema dict (post-`$ref` inlining).

**Expected Output:**
- If schema is `{}`: Return `{"type": "object", "properties": {}}`.
- If schema has `"properties"` but no `"type"`: Return schema with `"type": "object"` added.
- If schema already has `"type": "object"`: Return schema unchanged.

**Boundary Conditions:**
- Schema with `"type": "object"` and `"properties": {}`: Return as-is.
- Schema with `"type": "object"` and populated `"properties"`: Return as-is.

**Error Conditions:** None. This method always produces a valid result.

---

#### FR-SCHEMA-004: Convert ModuleDescriptor input_schema to OpenAI parameters

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-004 |
| **Title** | Convert apcore input_schema to OpenAI-compatible parameters schema |
| **Priority** | P0 |
| **Traces to** | F-008 |

**Description:** The schema conversion for OpenAI `parameters` shall apply the same `$ref` inlining and `type: object` guarantee as MCP conversion (FR-SCHEMA-001 through FR-SCHEMA-003). The resulting schema shall be usable as the `"parameters"` value in an OpenAI tool definition.

**Input/Trigger:** A `ModuleDescriptor` with a populated `input_schema` dict.

**Expected Output:** A JSON Schema dict identical to the MCP `inputSchema` output (same inlining and normalization rules apply).

**Boundary Conditions:** Same as FR-SCHEMA-001.

**Error Conditions:** Same as FR-SCHEMA-001.

---

#### FR-SCHEMA-005: Convert output_schema for structured MCP responses

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-005 |
| **Title** | Convert apcore output_schema for MCP structured output |
| **Priority** | P1 |
| **Traces to** | F-013 |

**Description:** The `SchemaConverter.convert_output_schema()` method shall accept a `ModuleDescriptor` and return a cleaned JSON Schema dict for the output. The same `$ref` inlining logic shall apply. If the `output_schema` is empty (`{}`), the method shall return an empty dict `{}`.

**Input/Trigger:** A `ModuleDescriptor` with `output_schema` dict.

**Expected Output:**
- A JSON Schema dict with all `$ref` inlined and `$defs` removed.
- Empty dict `{}` if the input `output_schema` is empty.

**Boundary Conditions:**
- Empty `output_schema` (`{}`): Return `{}`.
- Complex `output_schema` with `$defs`/`$ref`: Inline all references.

**Error Conditions:**
- Circular `$ref`: Raise `ValueError`.

---

#### FR-SCHEMA-006: Handle all JSON Schema types in conversion

| Field | Value |
|-------|-------|
| **ID** | FR-SCHEMA-006 |
| **Title** | Support all JSON Schema types during schema conversion |
| **Priority** | P0 |
| **Traces to** | F-001 |

**Description:** The schema converter shall correctly preserve and pass through all JSON Schema types in property definitions: `"string"`, `"integer"`, `"number"`, `"boolean"`, `"array"`, `"object"`, and `"null"`. Union types (e.g., `"type": ["string", "null"]`) shall also be preserved.

**Input/Trigger:** A JSON Schema dict containing properties of any valid JSON Schema type.

**Expected Output:** All type declarations shall be preserved exactly in the output schema. No type information shall be lost or transformed.

**Boundary Conditions:**
- Property with `"type": "array"` and `"items"` subschema: Preserve both.
- Property with `"type": "object"` and nested `"properties"`: Preserve the full nested structure.
- Property with `"enum"` values: Preserve the enum array.
- Property with `"const"` value: Preserve.
- Property with `"format"` (e.g., `"date-time"`, `"email"`): Preserve.
- Property with `"default"` value: Preserve (for MCP; OpenAI strict mode strips defaults per FR-OPENAI-003).
- Property with `"description"`: Preserve.

**Error Conditions:** None. All valid JSON Schema types shall be passed through.

---

### 3.2 FR-ANNOT: Annotation Mapping Requirements

---

#### FR-ANNOT-001: Map destructive annotation to MCP destructiveHint

| Field | Value |
|-------|-------|
| **ID** | FR-ANNOT-001 |
| **Title** | Map apcore destructive annotation to MCP destructiveHint |
| **Priority** | P0 |
| **Traces to** | F-002 |

**Description:** When `ModuleAnnotations.destructive` is `True`, the `AnnotationMapper.to_mcp_annotations()` method shall return a `ToolAnnotations` instance with `destructive_hint=True`. When `False`, it shall return `destructive_hint=False`.

**Input/Trigger:** A `ModuleAnnotations` instance with `destructive` field set.

**Expected Output:** `ToolAnnotations(destructive_hint=<value>)` where `<value>` equals the input `destructive` value.

**Boundary Conditions:**
- `destructive=True`: `destructive_hint=True`.
- `destructive=False`: `destructive_hint=False`.

**Error Conditions:** None.

---

#### FR-ANNOT-002: Map readonly annotation to MCP readOnlyHint

| Field | Value |
|-------|-------|
| **ID** | FR-ANNOT-002 |
| **Title** | Map apcore readonly annotation to MCP readOnlyHint |
| **Priority** | P0 |
| **Traces to** | F-002 |

**Description:** When `ModuleAnnotations.readonly` is `True`, the `AnnotationMapper.to_mcp_annotations()` method shall return a `ToolAnnotations` instance with `read_only_hint=True`. When `False`, it shall return `read_only_hint=False`.

**Input/Trigger:** A `ModuleAnnotations` instance with `readonly` field set.

**Expected Output:** `ToolAnnotations(read_only_hint=<value>)` where `<value>` equals the input `readonly` value.

**Boundary Conditions:**
- `readonly=True`: `read_only_hint=True`.
- `readonly=False`: `read_only_hint=False`.

**Error Conditions:** None.

---

#### FR-ANNOT-003: Map idempotent annotation to MCP idempotentHint

| Field | Value |
|-------|-------|
| **ID** | FR-ANNOT-003 |
| **Title** | Map apcore idempotent annotation to MCP idempotentHint |
| **Priority** | P0 |
| **Traces to** | F-002 |

**Description:** When `ModuleAnnotations.idempotent` is `True`, the `AnnotationMapper.to_mcp_annotations()` method shall return a `ToolAnnotations` instance with `idempotent_hint=True`. When `False`, it shall return `idempotent_hint=False`.

**Input/Trigger:** A `ModuleAnnotations` instance with `idempotent` field set.

**Expected Output:** `ToolAnnotations(idempotent_hint=<value>)` where `<value>` equals the input `idempotent` value.

**Boundary Conditions:**
- `idempotent=True`: `idempotent_hint=True`.
- `idempotent=False`: `idempotent_hint=False`.

**Error Conditions:** None.

---

#### FR-ANNOT-004: Map open_world annotation to MCP openWorldHint

| Field | Value |
|-------|-------|
| **ID** | FR-ANNOT-004 |
| **Title** | Map apcore open_world annotation to MCP openWorldHint |
| **Priority** | P0 |
| **Traces to** | F-002 |

**Description:** When `ModuleAnnotations.open_world` is `True`, the `AnnotationMapper.to_mcp_annotations()` method shall return a `ToolAnnotations` instance with `open_world_hint=True`. When `False`, it shall return `open_world_hint=False`.

**Input/Trigger:** A `ModuleAnnotations` instance with `open_world` field set.

**Expected Output:** `ToolAnnotations(open_world_hint=<value>)` where `<value>` equals the input `open_world` value.

**Boundary Conditions:**
- `open_world=True` (default): `open_world_hint=True`.
- `open_world=False`: `open_world_hint=False`.

**Error Conditions:** None.

---

#### FR-ANNOT-005: Default MCP annotations when ModuleAnnotations is None

| Field | Value |
|-------|-------|
| **ID** | FR-ANNOT-005 |
| **Title** | Return MCP default annotations when apcore annotations are None |
| **Priority** | P0 |
| **Traces to** | F-002 |

**Description:** When the `annotations` parameter is `None`, the `AnnotationMapper.to_mcp_annotations()` method shall return a `ToolAnnotations` instance with all fields set to their MCP defaults: `read_only_hint=False`, `destructive_hint=False`, `idempotent_hint=False`, `open_world_hint=True`.

**Input/Trigger:** `annotations=None`.

**Expected Output:** `ToolAnnotations(read_only_hint=False, destructive_hint=False, idempotent_hint=False, open_world_hint=True)`.

**Boundary Conditions:** None (single case: input is `None`).

**Error Conditions:** None.

---

#### FR-ANNOT-006: Preserve requires_approval for MCP clients

| Field | Value |
|-------|-------|
| **ID** | FR-ANNOT-006 |
| **Title** | Preserve requires_approval annotation for MCP client consumption |
| **Priority** | P0 |
| **Traces to** | F-002 |

**Description:** The `AnnotationMapper.has_requires_approval()` method shall return `True` when `ModuleAnnotations.requires_approval` is `True`, and `False` when annotations are `None` or `requires_approval` is `False`. This value shall be accessible to MCP server handlers to enable confirmation flows.

**Input/Trigger:** A `ModuleAnnotations` instance or `None`.

**Expected Output:**
- `annotations=None`: Return `False`.
- `annotations.requires_approval=True`: Return `True`.
- `annotations.requires_approval=False`: Return `False`.

**Boundary Conditions:** None.

**Error Conditions:** None.

---

#### FR-ANNOT-007: Generate parseable annotation suffix for OpenAI descriptions

| Field | Value |
|-------|-------|
| **ID** | FR-ANNOT-007 |
| **Title** | Generate annotation suffix string for OpenAI tool descriptions |
| **Priority** | P1 |
| **Traces to** | F-011 |

**Description:** The `AnnotationMapper.to_description_suffix()` method shall generate a structured, parseable string containing annotation metadata for embedding in OpenAI tool descriptions. Only annotation fields that differ from their defaults shall be included.

**Input/Trigger:** A `ModuleAnnotations` instance or `None`.

**Expected Output:**
- `annotations=None`: Return empty string `""`.
- All annotations at default values: Return empty string `""`.
- Non-default values present: Return string in format `"\n\n[Annotations: <field>=<value>, ...]"`. Example: `"\n\n[Annotations: readonly=true, idempotent=true]"`.

**Default values (excluded when matching):**
- `readonly=False`
- `destructive=False`
- `idempotent=False`
- `requires_approval=False`
- `open_world=True`

**Boundary Conditions:**
- Single non-default field: Return suffix with one field (e.g., `"\n\n[Annotations: destructive=true]"`).
- All five fields non-default: Return suffix with all five fields listed.

**Error Conditions:** None.

---

### 3.3 FR-EXEC: Execution Routing Requirements

---

#### FR-EXEC-001: Route MCP tool call to Executor.call_async

| Field | Value |
|-------|-------|
| **ID** | FR-EXEC-001 |
| **Title** | Route MCP tool calls through Executor.call_async() |
| **Priority** | P0 |
| **Traces to** | F-003 |

**Description:** The `ExecutionRouter.handle_call()` async method shall receive a tool name (string) and arguments (dict), invoke `Executor.call_async(tool_name, arguments)`, and return the result as a `CallToolResult`. The tool name equals the apcore `module_id`.

**Input/Trigger:** MCP `tools/call` request with `name` (string) and `arguments` (dict).

**Expected Output:** On success, a `CallToolResult` with `isError=False` and `content` containing a `TextContent` item whose `text` is the JSON-serialized module output dict.

**Boundary Conditions:**
- Empty arguments dict `{}`: Pass `{}` to `Executor.call_async()`.
- Module output containing nested objects: Serialize the full output dict as JSON.

**Error Conditions:** See FR-ERROR-001 through FR-ERROR-009 for error handling.

---

#### FR-EXEC-002: Serialize module output as JSON text content

| Field | Value |
|-------|-------|
| **ID** | FR-EXEC-002 |
| **Title** | Serialize successful module output to JSON string in MCP TextContent |
| **Priority** | P0 |
| **Traces to** | F-003, F-013 |

**Description:** On successful execution, the `ExecutionRouter` shall serialize the module output dict to a JSON string using `json.dumps()` with `default=str` to handle non-serializable types (e.g., `datetime`, `Path`, `UUID`). The JSON string shall be wrapped in a `TextContent(type="text", text=<json_string>)` within the `CallToolResult`.

**Input/Trigger:** A `dict[str, Any]` returned by `Executor.call_async()`.

**Expected Output:** `CallToolResult(content=[TextContent(type="text", text=<json_string>)], isError=False)`.

**Boundary Conditions:**
- Empty output dict `{}`: Serialize as `"{}"`.
- Output containing `datetime` objects: Convert to string via `default=str`.
- Output containing `Path` objects: Convert to string via `default=str`.
- Output containing `bytes`: Convert to string via `default=str`.

**Error Conditions:**
- If serialization fails even with `default=str`: Return a `CallToolResult` with `isError=True` and message "Failed to serialize module output".

---

#### FR-EXEC-003: Preserve full Executor pipeline for MCP-originated calls

| Field | Value |
|-------|-------|
| **ID** | FR-EXEC-003 |
| **Title** | Execute full apcore pipeline for all MCP tool calls |
| **Priority** | P0 |
| **Traces to** | F-003 |

**Description:** Every MCP tool call routed through `ExecutionRouter.handle_call()` shall pass through the complete apcore Executor 10-step pipeline: (1) context creation, (2) safety checks (call depth, circular call, frequency), (3) module lookup, (4) ACL enforcement, (5) input validation, (6) middleware before hooks, (7) module execution, (8) output validation, (9) middleware after hooks, (10) result return. No step shall be bypassed.

**Input/Trigger:** Any MCP `tools/call` request.

**Expected Output:** The Executor pipeline executes in full. The final output (post-middleware) is returned.

**Boundary Conditions:**
- Module with no middleware: Steps 6 and 9 execute with zero middlewares (no-op).
- Module with no ACL: Step 4 is skipped (Executor behavior when `acl=None`).
- Module with no input_schema: Step 5 is skipped (Executor behavior).

**Error Conditions:** Any pipeline step may raise errors mapped per FR-ERROR-001 through FR-ERROR-009.

---

#### FR-EXEC-004: Support both sync and async module execute methods

| Field | Value |
|-------|-------|
| **ID** | FR-EXEC-004 |
| **Title** | Support sync and async module execution via Executor.call_async() |
| **Priority** | P0 |
| **Traces to** | F-003 |

**Description:** The `ExecutionRouter` shall always use `Executor.call_async()`, which natively handles both synchronous and asynchronous module `execute()` methods. For sync modules, the Executor uses `asyncio.to_thread()` to avoid blocking the event loop. No additional bridging logic is required in apcore-mcp.

**Input/Trigger:** A tool call to any module (sync or async).

**Expected Output:** The module executes correctly regardless of whether its `execute()` method is sync or async.

**Boundary Conditions:**
- Sync module: Executed via `asyncio.to_thread()` by the Executor.
- Async module: Executed directly as a coroutine by the Executor.

**Error Conditions:** None specific to this requirement.

---

#### FR-EXEC-005: Accept pre-configured Executor instance

| Field | Value |
|-------|-------|
| **ID** | FR-EXEC-005 |
| **Title** | Accept Executor instance for tool call routing in serve() |
| **Priority** | P1 |
| **Traces to** | F-014 |

**Description:** The `serve()` function shall accept either a `Registry` or an `Executor` as its first argument. When an `Executor` is passed, `serve()` shall use that Executor for all tool call routing (preserving its ACL, middleware, and timeout configuration). When a `Registry` is passed, `serve()` shall create a default `Executor(registry)` internally.

**Input/Trigger:**
- `serve(executor)` where `executor` is an `Executor` instance.
- `serve(registry)` where `registry` is a `Registry` instance.

**Expected Output:**
- With Executor: The Executor's `registry` property is used for tool discovery; the Executor is used for `call_async()`.
- With Registry: A new `Executor(registry)` is created with default configuration.

**Boundary Conditions:**
- Executor with ACL: ACL rules are enforced on all MCP tool calls.
- Executor with custom middleware: Middleware before/after hooks execute on all MCP tool calls.
- Executor with custom timeout: Timeout configuration applies to all MCP tool calls.

**Error Conditions:**
- First argument is neither `Registry` nor `Executor`: Raise `TypeError` with message "Expected Registry or Executor instance, got <type>".

---

#### FR-EXEC-006: Accept pre-configured Executor in to_openai_tools

| Field | Value |
|-------|-------|
| **ID** | FR-EXEC-006 |
| **Title** | Accept Executor instance in to_openai_tools() for registry extraction |
| **Priority** | P1 |
| **Traces to** | F-014 |

**Description:** The `to_openai_tools()` function shall accept either a `Registry` or an `Executor` as its first argument. When an `Executor` is passed, the function shall extract the Registry via `executor.registry` for module enumeration and schema reading.

**Input/Trigger:**
- `to_openai_tools(executor)` where `executor` is an `Executor` instance.
- `to_openai_tools(registry)` where `registry` is a `Registry` instance.

**Expected Output:** Same list of OpenAI tool definition dicts regardless of whether Registry or Executor was passed (the source Registry is the same).

**Boundary Conditions:** None additional.

**Error Conditions:**
- First argument is neither `Registry` nor `Executor`: Raise `TypeError`.

---

#### FR-EXEC-007: ExecutionRouter never propagates exceptions

| Field | Value |
|-------|-------|
| **ID** | FR-EXEC-007 |
| **Title** | handle_call() catches all exceptions and returns CallToolResult |
| **Priority** | P0 |
| **Traces to** | F-003, F-004 |

**Description:** The `ExecutionRouter.handle_call()` method shall never propagate exceptions to the MCP Server layer. All exceptions -- whether apcore `ModuleError` subclasses or unexpected `Exception` instances -- shall be caught and converted to a `CallToolResult` with `isError=True` via the `ErrorMapper`.

**Input/Trigger:** Any exception raised during `Executor.call_async()` execution.

**Expected Output:** A `CallToolResult` with `isError=True` and appropriate error message (see FR-ERROR section).

**Boundary Conditions:** None.

**Error Conditions:** None (this requirement defines how errors are handled).

---

### 3.4 FR-ERROR: Error Mapping Requirements

---

#### FR-ERROR-001: Map ModuleNotFoundError to MCP error response

| Field | Value |
|-------|-------|
| **ID** | FR-ERROR-001 |
| **Title** | Map apcore ModuleNotFoundError to MCP tool-not-found error |
| **Priority** | P0 |
| **Traces to** | F-004 |

**Description:** When `Executor.call_async()` raises `ModuleNotFoundError`, the `ErrorMapper.to_mcp_error()` method shall return a `CallToolResult` with `isError=True` and text content `"Module not found: {module_id}"` where `{module_id}` is the value from `error.details["module_id"]`.

**Input/Trigger:** `ModuleNotFoundError(module_id="image.resize")`.

**Expected Output:** `CallToolResult(content=[TextContent(type="text", text="Module not found: image.resize")], isError=True)`.

**Boundary Conditions:**
- Module ID with dots: Preserve dots in error message (e.g., "Module not found: comfyui.workflow.execute").
- Module ID as empty string: Display "Module not found: ".

**Error Conditions:** None.

---

#### FR-ERROR-002: Map SchemaValidationError to MCP error with field details

| Field | Value |
|-------|-------|
| **ID** | FR-ERROR-002 |
| **Title** | Map apcore SchemaValidationError to MCP error with field-level details |
| **Priority** | P0 |
| **Traces to** | F-004 |

**Description:** When `Executor.call_async()` raises `SchemaValidationError`, the `ErrorMapper` shall return a `CallToolResult` with `isError=True` and text content formatted as:
```
Input validation failed:
- {field}: {message} ({code})
- {field}: {message} ({code})
```
where each line corresponds to an error entry from `error.details["errors"]`. Each error entry is a dict with keys `"field"`, `"code"`, and `"message"`.

**Input/Trigger:** `SchemaValidationError(message="Input validation failed", errors=[{"field": "width", "code": "int_type", "message": "Input should be a valid integer"}])`.

**Expected Output:** `CallToolResult(content=[TextContent(type="text", text="Input validation failed:\n- width: Input should be a valid integer (int_type)")], isError=True)`.

**Boundary Conditions:**
- Single validation error: One bullet line.
- Multiple validation errors: Multiple bullet lines, one per error.
- Empty errors list: Return text `"Input validation failed"` without bullet lines.
- Nested field path (e.g., `"parameters.width"`): Preserve the full dotted path.

**Error Conditions:** None.

---

#### FR-ERROR-003: Map ACLDeniedError to MCP error without sensitive data

| Field | Value |
|-------|-------|
| **ID** | FR-ERROR-003 |
| **Title** | Map apcore ACLDeniedError to sanitized MCP access denied error |
| **Priority** | P0 |
| **Traces to** | F-004 |

**Description:** When `Executor.call_async()` raises `ACLDeniedError`, the `ErrorMapper` shall return a `CallToolResult` with `isError=True` and text content `"Access denied"`. The `caller_id` and `target_id` from the error details shall NOT be included in the MCP response to prevent leaking security-sensitive information.

**Input/Trigger:** `ACLDeniedError(caller_id="mcp_client_123", target_id="admin.delete_all")`.

**Expected Output:** `CallToolResult(content=[TextContent(type="text", text="Access denied")], isError=True)`.

**Boundary Conditions:**
- `caller_id=None`: Message remains "Access denied".
- Any `target_id` value: Message remains "Access denied".

**Error Conditions:** None.

---

#### FR-ERROR-004: Map ModuleTimeoutError to MCP error with duration

| Field | Value |
|-------|-------|
| **ID** | FR-ERROR-004 |
| **Title** | Map apcore ModuleTimeoutError to MCP timeout error with duration |
| **Priority** | P0 |
| **Traces to** | F-004 |

**Description:** When `Executor.call_async()` raises `ModuleTimeoutError`, the `ErrorMapper` shall return a `CallToolResult` with `isError=True` and text content `"Module timed out after {timeout_ms}ms"` where `{timeout_ms}` is from `error.details["timeout_ms"]`.

**Input/Trigger:** `ModuleTimeoutError(module_id="slow.module", timeout_ms=30000)`.

**Expected Output:** `CallToolResult(content=[TextContent(type="text", text="Module timed out after 30000ms")], isError=True)`.

**Boundary Conditions:**
- `timeout_ms=0`: Display "Module timed out after 0ms".
- Large timeout (e.g., 60000): Display "Module timed out after 60000ms".

**Error Conditions:** None.

---

#### FR-ERROR-005: Map InvalidInputError to MCP error

| Field | Value |
|-------|-------|
| **ID** | FR-ERROR-005 |
| **Title** | Map apcore InvalidInputError to MCP invalid input error |
| **Priority** | P0 |
| **Traces to** | F-004 |

**Description:** When `Executor.call_async()` raises `InvalidInputError`, the `ErrorMapper` shall return a `CallToolResult` with `isError=True` and text content `"Invalid input: {message}"` where `{message}` is `error.message`.

**Input/Trigger:** `InvalidInputError(message="module_id must be a non-empty string")`.

**Expected Output:** `CallToolResult(content=[TextContent(type="text", text="Invalid input: module_id must be a non-empty string")], isError=True)`.

**Boundary Conditions:** None.

**Error Conditions:** None.

---

#### FR-ERROR-006: Map CallDepthExceededError to MCP error

| Field | Value |
|-------|-------|
| **ID** | FR-ERROR-006 |
| **Title** | Map apcore CallDepthExceededError to MCP internal safety error |
| **Priority** | P0 |
| **Traces to** | F-004 |

**Description:** When `Executor.call_async()` raises `CallDepthExceededError`, the `ErrorMapper` shall return a `CallToolResult` with `isError=True` and text content `"Call depth limit exceeded"`. The call chain details shall NOT be included in the MCP response.

**Input/Trigger:** `CallDepthExceededError(depth=33, max_depth=32, call_chain=[...])`.

**Expected Output:** `CallToolResult(content=[TextContent(type="text", text="Call depth limit exceeded")], isError=True)`.

**Boundary Conditions:** None.

**Error Conditions:** None.

---

#### FR-ERROR-007: Map CircularCallError to MCP error

| Field | Value |
|-------|-------|
| **ID** | FR-ERROR-007 |
| **Title** | Map apcore CircularCallError to MCP internal safety error |
| **Priority** | P0 |
| **Traces to** | F-004 |

**Description:** When `Executor.call_async()` raises `CircularCallError`, the `ErrorMapper` shall return a `CallToolResult` with `isError=True` and text content `"Circular call detected"`. The module_id and call chain details shall NOT be included in the MCP response.

**Input/Trigger:** `CircularCallError(module_id="a.module", call_chain=["a.module", "b.module", "a.module"])`.

**Expected Output:** `CallToolResult(content=[TextContent(type="text", text="Circular call detected")], isError=True)`.

**Boundary Conditions:** None.

**Error Conditions:** None.

---

#### FR-ERROR-008: Map CallFrequencyExceededError to MCP error

| Field | Value |
|-------|-------|
| **ID** | FR-ERROR-008 |
| **Title** | Map apcore CallFrequencyExceededError to MCP internal safety error |
| **Priority** | P0 |
| **Traces to** | F-004 |

**Description:** When `Executor.call_async()` raises `CallFrequencyExceededError`, the `ErrorMapper` shall return a `CallToolResult` with `isError=True` and text content `"Call frequency limit exceeded"`.

**Input/Trigger:** `CallFrequencyExceededError(module_id="spammy.module", count=4, max_repeat=3, call_chain=[...])`.

**Expected Output:** `CallToolResult(content=[TextContent(type="text", text="Call frequency limit exceeded")], isError=True)`.

**Boundary Conditions:** None.

**Error Conditions:** None.

---

#### FR-ERROR-009: Map unexpected exceptions to sanitized MCP error

| Field | Value |
|-------|-------|
| **ID** | FR-ERROR-009 |
| **Title** | Map unexpected exceptions to generic MCP internal error |
| **Priority** | P0 |
| **Traces to** | F-004 |

**Description:** When `Executor.call_async()` raises any exception that is NOT a subclass of `ModuleError`, the `ErrorMapper` shall return a `CallToolResult` with `isError=True` and text content `"Internal error occurred"`. The original exception message, class name, and stack trace shall NOT be included in the MCP response. The full exception shall be logged at ERROR level via the `apcore_mcp` logger with the complete stack trace for debugging.

**Input/Trigger:** Any `Exception` not inheriting from `ModuleError` (e.g., `RuntimeError("disk full")`, `KeyError("missing_key")`).

**Expected Output:** `CallToolResult(content=[TextContent(type="text", text="Internal error occurred")], isError=True)`.

**Boundary Conditions:**
- `RuntimeError` with message: Response contains only "Internal error occurred".
- `TypeError` with message: Response contains only "Internal error occurred".
- `OSError` with message: Response contains only "Internal error occurred".

**Error Conditions:** None (this requirement defines the catch-all error path).

---

#### FR-ERROR-010: Map unrecognized ModuleError subclasses to MCP error

| Field | Value |
|-------|-------|
| **ID** | FR-ERROR-010 |
| **Title** | Map unrecognized ModuleError subclasses to generic module error |
| **Priority** | P0 |
| **Traces to** | F-004 |

**Description:** When `Executor.call_async()` raises a `ModuleError` subclass not explicitly mapped by FR-ERROR-001 through FR-ERROR-008 (e.g., future error types added to apcore), the `ErrorMapper` shall return a `CallToolResult` with `isError=True` and text content `"Module error: {error.code}"`.

**Input/Trigger:** A `ModuleError` subclass not in the explicit mapping table (e.g., `ConfigError`, `SchemaNotFoundError`).

**Expected Output:** `CallToolResult(content=[TextContent(type="text", text="Module error: {code}")], isError=True)` where `{code}` is the `error.code` attribute.

**Boundary Conditions:** None.

**Error Conditions:** None.

---

#### FR-ERROR-011: All MCP error responses set isError=True

| Field | Value |
|-------|-------|
| **ID** | FR-ERROR-011 |
| **Title** | Every error response sets isError=True in CallToolResult |
| **Priority** | P0 |
| **Traces to** | F-004 |

**Description:** Every `CallToolResult` produced by the `ErrorMapper.to_mcp_error()` method shall have `isError=True`. No error response shall ever have `isError=False`.

**Input/Trigger:** Any exception passed to `ErrorMapper.to_mcp_error()`.

**Expected Output:** `CallToolResult` with `isError=True`.

**Boundary Conditions:** None.

**Error Conditions:** None.

---

### 3.5 FR-SERVER: MCP Server Requirements

---

#### FR-SERVER-001: serve() launches MCP Server with stdio transport by default

| Field | Value |
|-------|-------|
| **ID** | FR-SERVER-001 |
| **Title** | serve() starts MCP Server with stdio transport as default |
| **Priority** | P0 |
| **Traces to** | F-005, F-006 |

**Description:** Calling `serve(registry)` with no `transport` parameter shall start an MCP Server using stdio transport (reading from stdin, writing to stdout). The default transport value shall be `"stdio"`.

**Input/Trigger:** `serve(registry)` or `serve(registry, transport="stdio")`.

**Expected Output:** An MCP Server starts, reads MCP protocol messages from stdin, writes responses to stdout, and blocks until the connection is closed.

**Boundary Conditions:**
- stdin/stdout redirected (e.g., by Claude Desktop subprocess): Server communicates via the redirected streams.

**Error Conditions:**
- stdio streams cannot be opened: Raise `RuntimeError`.

---

#### FR-SERVER-002: serve() launches MCP Server with Streamable HTTP transport

| Field | Value |
|-------|-------|
| **ID** | FR-SERVER-002 |
| **Title** | serve() starts MCP Server with Streamable HTTP transport |
| **Priority** | P0 |
| **Traces to** | F-005, F-007 |

**Description:** Calling `serve(registry, transport="streamable-http", host=<host>, port=<port>)` shall start an MCP Server listening on the specified host and port using the MCP Streamable HTTP transport protocol.

**Input/Trigger:** `serve(registry, transport="streamable-http", host="0.0.0.0", port=8000)`.

**Expected Output:** An HTTP server starts on the specified host:port, accepts MCP protocol requests over HTTP, and blocks until shut down.

**Boundary Conditions:**
- `host="127.0.0.1"` (default): Server listens only on localhost.
- `host="0.0.0.0"`: Server listens on all network interfaces.
- `port=8000` (default): Server listens on port 8000.
- `port=1` (minimum): Server attempts to bind to port 1.
- `port=65535` (maximum): Server attempts to bind to port 65535.

**Error Conditions:**
- Port already in use: Raise `OSError`.
- Port outside range [1, 65535]: Raise `ValueError` with message "Port must be between 1 and 65535, got {port}".
- Host is empty string: Raise `ValueError` with message "Host must not be empty".

---

#### FR-SERVER-003: serve() launches MCP Server with SSE transport

| Field | Value |
|-------|-------|
| **ID** | FR-SERVER-003 |
| **Title** | serve() starts MCP Server with SSE transport (deprecated) |
| **Priority** | P1 |
| **Traces to** | F-005, F-010 |

**Description:** Calling `serve(registry, transport="sse", host=<host>, port=<port>)` shall start an MCP Server using the Server-Sent Events transport for backward compatibility with older MCP clients. A deprecation warning shall be logged at WARNING level on startup.

**Input/Trigger:** `serve(registry, transport="sse", host="0.0.0.0", port=8000)`.

**Expected Output:** An SSE-based MCP Server starts on the specified host:port. A WARNING-level log message "SSE transport is deprecated; use streamable-http instead" is emitted.

**Boundary Conditions:** Same host/port boundaries as FR-SERVER-002.

**Error Conditions:** Same as FR-SERVER-002.

---

#### FR-SERVER-004: serve() configures server name and version

| Field | Value |
|-------|-------|
| **ID** | FR-SERVER-004 |
| **Title** | serve() accepts name and version parameters |
| **Priority** | P0 |
| **Traces to** | F-005 |

**Description:** The `serve()` function shall accept optional `name` (str, default `"apcore-mcp"`) and `version` (str or None, default None) parameters. These values shall be reported to MCP clients during protocol initialization. When `version` is `None`, the package version from `apcore_mcp.__version__` shall be used.

**Input/Trigger:** `serve(registry, name="my-tools", version="2.0.0")`.

**Expected Output:** The MCP Server reports `name="my-tools"` and `version="2.0.0"` during client initialization.

**Boundary Conditions:**
- `name` max length: 255 characters.
- `name` minimum: 1 character (non-empty).
- `version=None`: Uses `apcore_mcp.__version__`.
- `version=""`: Raises `ValueError`.

**Error Conditions:**
- Empty `name` (`""`): Raise `ValueError` with message "name must not be empty".
- `name` exceeding 255 characters: Raise `ValueError` with message "name must not exceed 255 characters".
- Empty `version` (`""`): Raise `ValueError` with message "version must not be empty".

---

#### FR-SERVER-005: serve() blocks until server shutdown

| Field | Value |
|-------|-------|
| **ID** | FR-SERVER-005 |
| **Title** | serve() blocks the calling thread until server shuts down |
| **Priority** | P0 |
| **Traces to** | F-005 |

**Description:** The `serve()` function shall block the calling thread/coroutine until the MCP Server is shut down (via SIGINT, SIGTERM, parent process exit for stdio, or explicit shutdown). Control shall not return to the caller until the server has fully stopped.

**Input/Trigger:** `serve(registry)` call.

**Expected Output:** Function blocks. Returns `None` when server shuts down.

**Boundary Conditions:**
- SIGINT received: Graceful shutdown, function returns.
- SIGTERM received: Graceful shutdown, function returns.
- stdio parent process exits: Connection closed, function returns.

**Error Conditions:** None.

---

#### FR-SERVER-006: serve() with empty registry starts server with zero tools

| Field | Value |
|-------|-------|
| **ID** | FR-SERVER-006 |
| **Title** | serve() starts server with zero tools for empty registry and logs warning |
| **Priority** | P0 |
| **Traces to** | F-005 |

**Description:** When `serve()` is called with a `Registry` containing zero registered modules, the MCP Server shall start with an empty tool list. A WARNING-level log message shall be emitted: "No modules registered; server starting with zero tools".

**Input/Trigger:** `serve(registry)` where `registry.count == 0`.

**Expected Output:** Server starts. MCP clients that call `tools/list` receive an empty list `[]`. Warning logged.

**Boundary Conditions:** None.

**Error Conditions:** None.

---

#### FR-SERVER-007: serve() validates transport parameter

| Field | Value |
|-------|-------|
| **ID** | FR-SERVER-007 |
| **Title** | serve() rejects unknown transport values |
| **Priority** | P0 |
| **Traces to** | F-005 |

**Description:** The `serve()` function shall validate the `transport` parameter against the allowed values: `"stdio"`, `"streamable-http"`, `"sse"`. Comparison shall be case-insensitive (normalized to lowercase). Any other value shall raise `ValueError`.

**Input/Trigger:** `serve(registry, transport="websocket")`.

**Expected Output:** `ValueError` raised with message "Unknown transport: 'websocket'. Must be one of: stdio, streamable-http, sse".

**Boundary Conditions:**
- `transport="STDIO"`: Accepted (case-insensitive).
- `transport="Streamable-HTTP"`: Accepted (case-insensitive).
- `transport=""`: Raises `ValueError`.
- `transport="http"`: Raises `ValueError` (not an exact match).

**Error Conditions:**
- Unknown transport value: Raise `ValueError`.

---

#### FR-SERVER-008: serve() validates tags parameter

| Field | Value |
|-------|-------|
| **ID** | FR-SERVER-008 |
| **Title** | serve() validates tags filter parameter |
| **Priority** | P1 -- Implemented |
| **Traces to** | F-018 |

**Description:** The `serve()` function shall accept an optional `tags` parameter (`list[str] | None`, default `None`). When provided, only modules with ALL specified tags shall be exposed as MCP tools. Each tag string must be non-empty.

**Input/Trigger:** `serve(registry, tags=["public", "stable"])`.

**Expected Output:** Only modules possessing both tags "public" and "stable" appear in the MCP tool list.

**Boundary Conditions:**
- `tags=None`: All modules exposed (no filtering).
- `tags=[]`: All modules exposed (empty filter matches all).
- `tags=["nonexistent"]`: Zero tools exposed.

**Error Conditions:**
- Any tag is empty string: Raise `ValueError` with message "Tag values must not be empty".

---

#### FR-SERVER-009: serve() validates prefix parameter

| Field | Value |
|-------|-------|
| **ID** | FR-SERVER-009 |
| **Title** | serve() validates prefix filter parameter |
| **Priority** | P1 -- Implemented |
| **Traces to** | F-018 |

**Description:** The `serve()` function shall accept an optional `prefix` parameter (`str | None`, default `None`). When provided, only modules whose `module_id` starts with the prefix shall be exposed as MCP tools.

**Input/Trigger:** `serve(registry, prefix="api.")`.

**Expected Output:** Only modules whose IDs start with "api." appear in the MCP tool list.

**Boundary Conditions:**
- `prefix=None`: All modules exposed.
- `prefix="x."`: Only modules with IDs starting with "x." are exposed.
- `prefix="nonexistent."`: Zero tools exposed.

**Error Conditions:**
- `prefix=""` (empty string): Raise `ValueError` with message "prefix must not be empty".

---

#### FR-SERVER-010: serve() configures log level

| Field | Value |
|-------|-------|
| **ID** | FR-SERVER-010 |
| **Title** | serve() accepts log_level parameter |
| **Priority** | P1 -- Implemented |
| **Traces to** | F-016 |

**Description:** The `serve()` function shall accept an optional `log_level` parameter (`str | None`, default `None`). When provided, it shall configure logging via `logging.basicConfig(level=...)` to set the log level. Log level configuration is also available via the CLI `--log-level` option.

**Input/Trigger:** `serve(registry, log_level="DEBUG")`.

**Expected Output:** The `apcore_mcp` logger is configured at DEBUG level.

**Boundary Conditions:**
- `log_level=None`: No change to logging configuration.
- `log_level="DEBUG"`: DEBUG level set.
- `log_level="INFO"`: INFO level set.
- `log_level="WARNING"`: WARNING level set.
- `log_level="ERROR"`: ERROR level set.
- Case-insensitive: `"debug"` accepted.

**Error Conditions:**
- Unknown log level: Raise `ValueError` with message "Unknown log level: '{level}'. Must be one of: DEBUG, INFO, WARNING, ERROR".

---

#### FR-SERVER-011: MCP Server registers tools from all registry modules

| Field | Value |
|-------|-------|
| **ID** | FR-SERVER-011 |
| **Title** | MCP Server tool list contains one tool per registered module |
| **Priority** | P0 |
| **Traces to** | F-001, F-005 |

**Description:** When the MCP Server starts, it shall iterate over all modules in the Registry (respecting any tag/prefix filters), obtain each module's `ModuleDescriptor` via `registry.get_definition(module_id)`, build an MCP `Tool` object from it, and register all tools. An MCP client sending a `tools/list` request shall receive a list with exactly one tool per eligible module.

**Input/Trigger:** MCP `tools/list` request.

**Expected Output:** A list of `Tool` objects. Each tool has:
- `name` equal to the apcore `module_id`.
- `description` equal to the apcore module `description`.
- `inputSchema` equal to the converted input schema (per FR-SCHEMA-001).
- `annotations` equal to the mapped annotations (per FR-ANNOT-001 through FR-ANNOT-006).

**Boundary Conditions:**
- Registry with 1 module: Tool list has 1 entry.
- Registry with 100 modules: Tool list has 100 entries.
- `get_definition()` returns `None` for a module_id (race condition): Skip that module, log warning.

**Error Conditions:**
- Schema conversion fails for one module: Skip that module, log warning, register remaining modules.

---

### 3.6 FR-TRANSPORT: Transport Requirements

---

#### FR-TRANSPORT-001: stdio transport reads from stdin and writes to stdout

| Field | Value |
|-------|-------|
| **ID** | FR-TRANSPORT-001 |
| **Title** | stdio transport uses stdin/stdout for MCP communication |
| **Priority** | P0 |
| **Traces to** | F-006 |

**Description:** When transport is `"stdio"`, the MCP Server shall read MCP protocol messages from standard input and write responses to standard output. This enables Claude Desktop and other MCP clients to launch the server as a subprocess and communicate via pipes.

**Input/Trigger:** `serve(registry, transport="stdio")`.

**Expected Output:** Server reads from `sys.stdin` and writes to `sys.stdout` using MCP protocol framing.

**Boundary Conditions:**
- stdin is a pipe (subprocess): Normal operation.
- stdin is a TTY (interactive terminal): Normal operation (for debugging).

**Error Conditions:**
- stdin stream closed unexpectedly: Server shuts down gracefully.

---

#### FR-TRANSPORT-002: stdio transport handles graceful shutdown on parent exit

| Field | Value |
|-------|-------|
| **ID** | FR-TRANSPORT-002 |
| **Title** | stdio server shuts down when parent process terminates |
| **Priority** | P0 |
| **Traces to** | F-006 |

**Description:** When the parent process (e.g., Claude Desktop) terminates or closes the stdio pipes, the MCP Server shall detect the closed connection and shut down gracefully without raising unhandled exceptions.

**Input/Trigger:** Parent process exits or closes stdin.

**Expected Output:** Server shuts down. `serve()` returns normally.

**Boundary Conditions:** None.

**Error Conditions:** None.

---

#### FR-TRANSPORT-003: Streamable HTTP transport binds to configurable host and port

| Field | Value |
|-------|-------|
| **ID** | FR-TRANSPORT-003 |
| **Title** | Streamable HTTP transport accepts host and port configuration |
| **Priority** | P0 |
| **Traces to** | F-007 |

**Description:** The Streamable HTTP transport shall bind to the address specified by the `host` parameter (default `"127.0.0.1"`) and the port specified by the `port` parameter (default `8000`).

**Input/Trigger:** `serve(registry, transport="streamable-http", host="0.0.0.0", port=9000)`.

**Expected Output:** HTTP server listens on `0.0.0.0:9000`.

**Boundary Conditions:**
- `host="127.0.0.1"`: Localhost only.
- `host="0.0.0.0"`: All interfaces.
- `port=1`: Minimum valid port.
- `port=65535`: Maximum valid port.

**Error Conditions:**
- Port in use: `OSError` propagated.
- Port < 1 or > 65535: `ValueError` raised (per FR-SERVER-002).

---

#### FR-TRANSPORT-004: SSE transport marked as deprecated

| Field | Value |
|-------|-------|
| **ID** | FR-TRANSPORT-004 |
| **Title** | SSE transport emits deprecation warning on startup |
| **Priority** | P1 |
| **Traces to** | F-010 |

**Description:** When the SSE transport is selected, the server shall log a WARNING-level message: "SSE transport is deprecated; use streamable-http instead". The SSE transport shall otherwise function identically to Streamable HTTP in terms of host/port configuration.

**Input/Trigger:** `serve(registry, transport="sse")`.

**Expected Output:** Warning logged. SSE server starts normally.

**Boundary Conditions:** Same as FR-TRANSPORT-003.

**Error Conditions:** Same as FR-TRANSPORT-003.

---

#### FR-TRANSPORT-005: host and port parameters ignored for stdio transport

| Field | Value |
|-------|-------|
| **ID** | FR-TRANSPORT-005 |
| **Title** | host and port are ignored when transport is stdio |
| **Priority** | P0 |
| **Traces to** | F-006 |

**Description:** When `transport="stdio"`, the `host` and `port` parameters shall be ignored without raising errors, even if explicitly provided.

**Input/Trigger:** `serve(registry, transport="stdio", host="0.0.0.0", port=9000)`.

**Expected Output:** stdio transport starts. host and port values are silently ignored.

**Boundary Conditions:** None.

**Error Conditions:** None.

---

### 3.7 FR-OPENAI: OpenAI Converter Requirements

---

#### FR-OPENAI-001: to_openai_tools() returns list of OpenAI tool definition dicts

| Field | Value |
|-------|-------|
| **ID** | FR-OPENAI-001 |
| **Title** | to_openai_tools() returns list of correctly structured OpenAI tool dicts |
| **Priority** | P0 |
| **Traces to** | F-008 |

**Description:** The `to_openai_tools()` function shall return a `list[dict[str, Any]]` where each dict represents one OpenAI tool definition. Each dict shall have the structure:
```json
{
  "type": "function",
  "function": {
    "name": "<normalized_module_id>",
    "description": "<module_description>",
    "parameters": { <JSON Schema from input_schema> }
  }
}
```

**Input/Trigger:** `to_openai_tools(registry)`.

**Expected Output:** A list with one dict per registered module.

**Boundary Conditions:**
- Empty registry: Return `[]`.
- Registry with 1 module: Return list with 1 dict.
- Registry with 100 modules: Return list with 100 dicts.

**Error Conditions:**
- Schema conversion fails for one module: Skip that module, log warning, include remaining modules.

---

#### FR-OPENAI-002: Normalize module IDs for OpenAI function name compatibility

| Field | Value |
|-------|-------|
| **ID** | FR-OPENAI-002 |
| **Title** | Replace dots with hyphens in OpenAI function names |
| **Priority** | P0 |
| **Traces to** | F-008 |

**Description:** The `ModuleIDNormalizer.normalize()` method shall replace all `.` (dot) characters in apcore module IDs with `-` (hyphen) to comply with OpenAI function name constraints (`^[a-zA-Z0-9_-]+$`). The `denormalize()` method shall reverse this transformation.

**Input/Trigger:** `normalize("image.resize")`.

**Expected Output:** `"image-resize"`.

**Boundary Conditions:**
- Module ID with no dots (e.g., `"simple"`): Return unchanged.
- Module ID with multiple dots (e.g., `"comfyui.workflow.execute"`): Return `"comfyui-workflow-execute"`.
- Module ID with existing underscores (e.g., `"my_module.resize"`): Return `"my_module-resize"`.
- Module ID with hyphens (e.g., `"my-module.resize"`): Reject with validation error. Hyphens are prohibited in canonical module IDs per PROTOCOL_SPEC (reserved for dot-to-hyphen normalization).

**Error Conditions:**
- Module ID containing hyphens: Raise validation error (hyphens are reserved for MCP/OpenAI tool name normalization).

---

#### FR-OPENAI-003: Apply OpenAI strict mode to tool definitions

| Field | Value |
|-------|-------|
| **ID** | FR-OPENAI-003 |
| **Title** | to_openai_tools() with strict=True applies strict mode schema rules |
| **Priority** | P1 |
| **Traces to** | F-012 |

**Description:** When `to_openai_tools(registry, strict=True)` is called, each tool definition shall include `"strict": true` in the `"function"` dict, and the schema shall be transformed to meet OpenAI strict mode requirements:
1. Set `"additionalProperties": false` on all object types.
2. Make all properties required (add all property names to `"required"` array).
3. Optional properties (those not in the original `"required"`) become nullable (`"type": [<original>, "null"]`).
4. Strip `x-*` extension fields.
5. Remove `"default"` values.
6. Recurse into nested objects, array items, and `oneOf`/`anyOf`/`allOf` branches.

**Input/Trigger:** `to_openai_tools(registry, strict=True)`.

**Expected Output:** Each tool dict has `"function": {"strict": true, ...}` and the `"parameters"` schema follows strict mode rules.

**Boundary Conditions:**
- `strict=False` (default): No `"strict"` key in function dict. Schema not transformed.
- Schema already has `"additionalProperties": false`: No change needed for that property.
- Schema with no optional properties: All properties already required; no nullable conversion needed.
- Schema with deeply nested objects: Strict mode rules applied recursively.

**Error Conditions:**
- Schema uses `"additionalProperties": true` intentionally: Log WARNING that schema may be incompatible with strict mode.

---

#### FR-OPENAI-004: Embed annotations in OpenAI tool descriptions

| Field | Value |
|-------|-------|
| **ID** | FR-OPENAI-004 |
| **Title** | to_openai_tools() with embed_annotations=True appends annotation suffix |
| **Priority** | P1 |
| **Traces to** | F-011 |

**Description:** When `to_openai_tools(registry, embed_annotations=True)` is called, each tool's `"description"` shall have the annotation suffix (from FR-ANNOT-007) appended. When `embed_annotations=False` (default), descriptions shall not be modified.

**Input/Trigger:** `to_openai_tools(registry, embed_annotations=True)`.

**Expected Output:** Each tool's description becomes `"{original_description}\n\n[Annotations: ...]"` for modules with non-default annotations.

**Boundary Conditions:**
- Module with all default annotations: Description unchanged (suffix is empty string).
- Module with `annotations=None`: Description unchanged.
- `embed_annotations=False`: Descriptions never modified.

**Error Conditions:** None.

---

#### FR-OPENAI-005: to_openai_tools() has zero dependency on openai package

| Field | Value |
|-------|-------|
| **ID** | FR-OPENAI-005 |
| **Title** | to_openai_tools() produces plain dicts without openai package dependency |
| **Priority** | P0 |
| **Traces to** | F-008 |

**Description:** The `to_openai_tools()` function shall return plain Python `dict` objects. It shall not import or depend on the `openai` package at runtime. The returned dicts shall be directly passable to `openai.chat.completions.create(tools=...)` without transformation.

**Input/Trigger:** `to_openai_tools(registry)`.

**Expected Output:** `list[dict[str, Any]]` containing only Python built-in types (dict, list, str, int, float, bool, None).

**Boundary Conditions:** None.

**Error Conditions:** None.

---

#### FR-OPENAI-006: to_openai_tools() with tag filtering

| Field | Value |
|-------|-------|
| **ID** | FR-OPENAI-006 |
| **Title** | to_openai_tools() filters modules by tags |
| **Priority** | P2 |
| **Traces to** | F-017 |

**Description:** When `to_openai_tools(registry, tags=["image"])` is called, only modules possessing all specified tags shall be included in the returned list. Tag filtering shall use `Registry.list(tags=tags)`.

**Input/Trigger:** `to_openai_tools(registry, tags=["image"])`.

**Expected Output:** List containing only tool dicts for modules with the "image" tag.

**Boundary Conditions:**
- `tags=None`: All modules included.
- `tags=["nonexistent"]`: Empty list returned.
- Multiple tags: Modules must have ALL tags.

**Error Conditions:**
- Empty tag string in list: Raise `ValueError`.

---

#### FR-OPENAI-007: to_openai_tools() with prefix filtering

| Field | Value |
|-------|-------|
| **ID** | FR-OPENAI-007 |
| **Title** | to_openai_tools() filters modules by ID prefix |
| **Priority** | P2 |
| **Traces to** | F-017 |

**Description:** When `to_openai_tools(registry, prefix="comfyui.")` is called, only modules whose `module_id` starts with the specified prefix shall be included.

**Input/Trigger:** `to_openai_tools(registry, prefix="comfyui.")`.

**Expected Output:** List containing only tool dicts for modules whose IDs start with "comfyui.".

**Boundary Conditions:**
- `prefix=None`: All modules included.
- `prefix="nonexistent."`: Empty list returned.
- Combined with tags: Both filters applied (intersection).

**Error Conditions:**
- Empty prefix string: Raise `ValueError`.

---

### 3.8 FR-CLI: CLI Requirements

---

#### FR-CLI-001: python -m apcore_mcp starts MCP server

| Field | Value |
|-------|-------|
| **ID** | FR-CLI-001 |
| **Title** | CLI entry point starts MCP server from extensions directory |
| **Priority** | P0 |
| **Traces to** | F-009 |

**Description:** Running `python -m apcore_mcp --extensions-dir ./extensions` shall create a `Registry`, call `registry.discover()` to load modules from the specified directory, and call `serve(registry)` with the configured transport.

**Input/Trigger:** `python -m apcore_mcp --extensions-dir ./extensions`.

**Expected Output:** MCP Server starts with stdio transport (default), all discovered modules registered as tools.

**Boundary Conditions:**
- Extensions directory with 0 modules: Server starts with zero tools, warning logged (per FR-SERVER-006).
- Extensions directory with 100 modules: All modules registered.

**Error Conditions:**
- `--extensions-dir` not provided: argparse error, exit code 2.

---

#### FR-CLI-002: --extensions-dir flag specifies extensions directory

| Field | Value |
|-------|-------|
| **ID** | FR-CLI-002 |
| **Title** | CLI --extensions-dir flag specifies module discovery directory |
| **Priority** | P0 |
| **Traces to** | F-009 |

**Description:** The `--extensions-dir` flag shall specify the filesystem path to the apcore extensions directory. The path must exist and be a readable directory.

**Input/Trigger:** `python -m apcore_mcp --extensions-dir /path/to/extensions`.

**Expected Output:** Registry created with `extensions_dir="/path/to/extensions"`.

**Boundary Conditions:**
- Absolute path: Accepted.
- Relative path: Accepted (resolved relative to CWD).
- Path with spaces: Accepted (shell quoting responsibility).

**Error Conditions:**
- Path does not exist: Print error "Error: extensions directory does not exist: {path}" to stderr, exit code 1.
- Path is a file (not directory): Print error "Error: extensions path is not a directory: {path}" to stderr, exit code 1.

---

#### FR-CLI-003: --transport flag selects transport protocol

| Field | Value |
|-------|-------|
| **ID** | FR-CLI-003 |
| **Title** | CLI --transport flag selects MCP transport |
| **Priority** | P0 |
| **Traces to** | F-009 |

**Description:** The `--transport` flag shall accept values `stdio`, `streamable-http`, and `sse`. Default is `stdio`.

**Input/Trigger:** `python -m apcore_mcp --extensions-dir ./ext --transport streamable-http`.

**Expected Output:** Server starts with Streamable HTTP transport.

**Boundary Conditions:**
- `--transport` not specified: Default to `stdio`.
- `--transport stdio`: stdio transport.
- `--transport streamable-http`: Streamable HTTP transport.
- `--transport sse`: SSE transport.

**Error Conditions:**
- Unknown transport value: argparse error with choices message, exit code 2.

---

#### FR-CLI-004: --host and --port flags configure network transports

| Field | Value |
|-------|-------|
| **ID** | FR-CLI-004 |
| **Title** | CLI --host and --port flags configure bind address |
| **Priority** | P0 |
| **Traces to** | F-009 |

**Description:** The `--host` flag (default `"127.0.0.1"`) and `--port` flag (default `8000`) shall configure the bind address for HTTP-based transports. They are ignored for stdio transport.

**Input/Trigger:** `python -m apcore_mcp --extensions-dir ./ext --transport streamable-http --host 0.0.0.0 --port 9000`.

**Expected Output:** Server starts on `0.0.0.0:9000`.

**Boundary Conditions:**
- `--port 1`: Minimum valid port.
- `--port 65535`: Maximum valid port.

**Error Conditions:**
- `--port 0`: Print error "Error: port must be between 1 and 65535" to stderr, exit code 1.
- `--port 70000`: Print error "Error: port must be between 1 and 65535" to stderr, exit code 1.
- `--port abc` (non-integer): argparse error, exit code 2.

---

#### FR-CLI-005: --name and --version flags configure server identity

| Field | Value |
|-------|-------|
| **ID** | FR-CLI-005 |
| **Title** | CLI --name and --version flags configure MCP server identity |
| **Priority** | P0 |
| **Traces to** | F-009 |

**Description:** The `--name` flag (default `"apcore-mcp"`, max 255 chars) and `--version` flag (default: package version) shall configure the server name and version reported to MCP clients.

**Input/Trigger:** `python -m apcore_mcp --extensions-dir ./ext --name "my-tools" --version "1.0.0"`.

**Expected Output:** Server reports name="my-tools" and version="1.0.0".

**Boundary Conditions:**
- `--name` not specified: Default `"apcore-mcp"`.
- `--version` not specified: Default from package version.

**Error Conditions:**
- `--name ""`: Print error "Error: server name must not be empty" to stderr, exit code 1.
- `--name` exceeding 255 chars: Print error, exit code 1.

---

#### FR-CLI-006: --log-level flag configures logging verbosity

| Field | Value |
|-------|-------|
| **ID** | FR-CLI-006 |
| **Title** | CLI --log-level flag sets logging level |
| **Priority** | P1 |
| **Traces to** | F-009, F-016 |

**Description:** The `--log-level` flag shall accept values `DEBUG`, `INFO`, `WARNING`, `ERROR` (default `INFO`).

**Input/Trigger:** `python -m apcore_mcp --extensions-dir ./ext --log-level DEBUG`.

**Expected Output:** Logging configured at DEBUG level.

**Boundary Conditions:**
- Not specified: Default `INFO`.

**Error Conditions:**
- Unknown level: argparse error with choices message, exit code 2.

---

#### FR-CLI-007: --help flag displays usage information

| Field | Value |
|-------|-------|
| **ID** | FR-CLI-007 |
| **Title** | CLI --help flag displays complete usage information |
| **Priority** | P0 |
| **Traces to** | F-009 |

**Description:** The `--help` flag shall display usage information including all available flags, their types, defaults, and descriptions, then exit with code 0.

**Input/Trigger:** `python -m apcore_mcp --help`.

**Expected Output:** Usage text printed to stdout, exit code 0.

**Boundary Conditions:** None.

**Error Conditions:** None.

---

#### FR-CLI-008: CLI exit codes

| Field | Value |
|-------|-------|
| **ID** | FR-CLI-008 |
| **Title** | CLI uses specific exit codes for different failure modes |
| **Priority** | P0 |
| **Traces to** | F-009 |

**Description:** The CLI shall use the following exit codes:
- `0`: Normal shutdown (SIGINT, SIGTERM, or parent process exit).
- `1`: Invalid arguments (non-existent directory, invalid port range, empty name).
- `2`: Startup failure (port in use, permission denied) or argparse error.

**Input/Trigger:** Various failure scenarios.

**Expected Output:** Appropriate exit code.

**Boundary Conditions:** None.

**Error Conditions:** None.

---

### 3.9 FR-DYNAMIC: Dynamic Registration Requirements

---

#### FR-DYNAMIC-001: Reflect new module registration in MCP tool list

| Field | Value |
|-------|-------|
| **ID** | FR-DYNAMIC-001 |
| **Title** | New module registration adds MCP tool at runtime |
| **Priority** | P1 |
| **Traces to** | F-015 |

**Description:** When a module is registered via `registry.register(module_id, module)` after `serve()` has started, the `RegistryListener` shall detect the `"register"` event, obtain the `ModuleDescriptor`, build an MCP `Tool` object, and add it to the active tool list. Subsequent `tools/list` requests from MCP clients shall include the new tool.

**Input/Trigger:** `registry.register("new.tool", module_instance)` while server is running.

**Expected Output:** The MCP tool list includes `"new.tool"` on the next `tools/list` request.

**Boundary Conditions:**
- Module registered before server starts: Included in initial tool list (not this requirement).
- Multiple modules registered in rapid succession: All are added.

**Error Conditions:**
- `get_definition()` returns `None` (race condition): Log warning, do not add tool, do not crash server.
- `build_tool()` raises `ValueError`: Log warning, do not add tool, do not crash server.

---

#### FR-DYNAMIC-002: Reflect module unregistration in MCP tool list

| Field | Value |
|-------|-------|
| **ID** | FR-DYNAMIC-002 |
| **Title** | Module unregistration removes MCP tool at runtime |
| **Priority** | P1 |
| **Traces to** | F-015 |

**Description:** When a module is unregistered via `registry.unregister(module_id)` after `serve()` has started, the `RegistryListener` shall detect the `"unregister"` event and remove the corresponding tool from the active tool list. Subsequent `tools/list` requests shall not include the removed tool. Tool calls to the removed tool shall return a "Module not found" error.

**Input/Trigger:** `registry.unregister("old.tool")` while server is running.

**Expected Output:** The MCP tool list no longer includes `"old.tool"`.

**Boundary Conditions:**
- Unregister a module_id not in the tools dict: Silently ignore.

**Error Conditions:** None.

---

#### FR-DYNAMIC-003: Send MCP tool list changed notification

| Field | Value |
|-------|-------|
| **ID** | FR-DYNAMIC-003 |
| **Title** | Notify MCP clients of tool list changes |
| **Priority** | P1 |
| **Traces to** | F-015 |

**Description:** After a tool is added or removed from the active tool list, the `RegistryListener` shall trigger an MCP `notifications/tools/list_changed` notification to connected clients (if the MCP SDK supports this capability). Clients receiving this notification shall re-fetch the tool list.

**Input/Trigger:** Tool added or removed from the active list.

**Expected Output:** MCP `notifications/tools/list_changed` sent to connected clients.

**Boundary Conditions:**
- MCP client does not support the notification: Notification is sent; client ignores it (no error).
- No clients connected: Notification is a no-op.

**Error Conditions:**
- Notification send failure: Log warning, do not crash server.

---

#### FR-DYNAMIC-004: Thread-safe tool list updates

| Field | Value |
|-------|-------|
| **ID** | FR-DYNAMIC-004 |
| **Title** | Tool list updates are thread-safe |
| **Priority** | P1 |
| **Traces to** | F-015 |

**Description:** The internal tools dictionary in `RegistryListener` shall be protected by a `threading.Lock`. All reads (from `list_tools` handler on the async event loop thread) and writes (from Registry callbacks on potentially different threads) shall acquire the lock before accessing the dictionary.

**Input/Trigger:** Concurrent tool list read (MCP `tools/list` request) and write (registry event callback).

**Expected Output:** No data corruption. Reads always see a consistent snapshot.

**Boundary Conditions:**
- High-frequency register/unregister: All operations serialized via lock.

**Error Conditions:** None.

---

### 3.10 FR-LOG: Logging Requirements

---

#### FR-LOG-001: Log tool count and transport on server startup

| Field | Value |
|-------|-------|
| **ID** | FR-LOG-001 |
| **Title** | Log registered tool count and transport type at server startup |
| **Priority** | P1 |
| **Traces to** | F-016 |

**Description:** When the MCP Server starts, a log message at INFO level shall be emitted with the format: "apcore-mcp server started: {count} tools registered, transport={transport}".

**Input/Trigger:** `serve()` completes server initialization.

**Expected Output:** INFO log: "apcore-mcp server started: 5 tools registered, transport=stdio".

**Boundary Conditions:**
- Zero tools: "apcore-mcp server started: 0 tools registered, transport=stdio".

**Error Conditions:** None.

---

#### FR-LOG-002: Log each tool call at DEBUG level

| Field | Value |
|-------|-------|
| **ID** | FR-LOG-002 |
| **Title** | Log tool call name at DEBUG level |
| **Priority** | P1 |
| **Traces to** | F-016 |

**Description:** When a tool call is received by the `ExecutionRouter`, a DEBUG-level log message shall be emitted with format: "Tool call: {tool_name}".

**Input/Trigger:** MCP `tools/call` request received.

**Expected Output:** DEBUG log: "Tool call: image.resize".

**Boundary Conditions:** None.

**Error Conditions:** None.

---

#### FR-LOG-003: Log tool call errors at ERROR level

| Field | Value |
|-------|-------|
| **ID** | FR-LOG-003 |
| **Title** | Log tool call errors with type and message at ERROR level |
| **Priority** | P1 |
| **Traces to** | F-016 |

**Description:** When a tool call results in an error, an ERROR-level log message shall be emitted with format: "Tool call error: {tool_name} - {error_type}: {error_message}". For unexpected exceptions (non-`ModuleError`), the full stack trace shall also be logged.

**Input/Trigger:** Any error during tool execution.

**Expected Output:** ERROR log: "Tool call error: image.resize - SchemaValidationError: Input validation failed".

**Boundary Conditions:**
- `ModuleError` subclass: Log error type and message.
- Non-`ModuleError` exception: Log error type, message, AND full stack trace.

**Error Conditions:** None.

---

#### FR-LOG-004: Use apcore_mcp logger namespace

| Field | Value |
|-------|-------|
| **ID** | FR-LOG-004 |
| **Title** | All log messages use the apcore_mcp logger namespace |
| **Priority** | P1 |
| **Traces to** | F-016 |

**Description:** All log messages emitted by apcore-mcp shall use loggers under the `apcore_mcp` namespace (e.g., `apcore_mcp`, `apcore_mcp.server.router`, `apcore_mcp.adapters.errors`). This enables users to configure log levels for apcore-mcp independently of other packages.

**Input/Trigger:** Any logging operation within apcore-mcp.

**Expected Output:** Logger name starts with `apcore_mcp`.

**Boundary Conditions:** None.

**Error Conditions:** None.

---

#### FR-LOG-005: Log verbosity controllable via standard Python logging

| Field | Value |
|-------|-------|
| **ID** | FR-LOG-005 |
| **Title** | Logging verbosity configurable via Python logging module |
| **Priority** | P1 |
| **Traces to** | F-016 |

**Description:** apcore-mcp shall not configure any log handlers or formatters by default (following Python library best practice). Users shall be able to control logging verbosity by configuring the `apcore_mcp` logger via standard Python `logging` module APIs. The `serve()` function's `log_level` parameter (FR-SERVER-010) provides a convenience shortcut.

**Input/Trigger:** `logging.getLogger("apcore_mcp").setLevel(logging.DEBUG)`.

**Expected Output:** All DEBUG and above messages from apcore-mcp are emitted.

**Boundary Conditions:** None.

**Error Conditions:** None.

---

### 3.11 FR-FILTER: Module Filtering Requirements

---

#### FR-FILTER-001: Filter MCP tools by tags in serve()

| Field | Value |
|-------|-------|
| **ID** | FR-FILTER-001 |
| **Title** | serve() exposes only modules matching tag filter |
| **Priority** | P2 |
| **Traces to** | F-018 |

**Description:** When `serve(registry, tags=["public"])` is called, only modules possessing ALL specified tags shall be registered as MCP tools. Modules not matching the filter shall not appear in `tools/list` responses and calls to them shall return "Module not found" errors.

**Input/Trigger:** `serve(registry, tags=["public"])`.

**Expected Output:** Tool list contains only modules with "public" tag.

**Boundary Conditions:**
- No modules match: Zero tools registered, warning logged.
- All modules match: All tools registered.

**Error Conditions:** None.

---

#### FR-FILTER-002: Filter MCP tools by prefix in serve()

| Field | Value |
|-------|-------|
| **ID** | FR-FILTER-002 |
| **Title** | serve() exposes only modules matching prefix filter |
| **Priority** | P2 |
| **Traces to** | F-018 |

**Description:** When `serve(registry, prefix="api.")` is called, only modules whose `module_id` starts with `"api."` shall be registered as MCP tools.

**Input/Trigger:** `serve(registry, prefix="api.")`.

**Expected Output:** Tool list contains only modules with IDs starting with "api.".

**Boundary Conditions:**
- No modules match: Zero tools, warning logged.
- Prefix matches all: All tools registered.

**Error Conditions:** None.

---

#### FR-FILTER-003: Combine tag and prefix filters

| Field | Value |
|-------|-------|
| **ID** | FR-FILTER-003 |
| **Title** | Tag and prefix filters combine as intersection |
| **Priority** | P2 |
| **Traces to** | F-017, F-018 |

**Description:** When both `tags` and `prefix` parameters are provided (to `serve()` or `to_openai_tools()`), the filters shall be combined as a logical AND (intersection). Only modules matching BOTH the tag filter AND the prefix filter shall be included.

**Input/Trigger:** `serve(registry, tags=["public"], prefix="api.")`.

**Expected Output:** Only modules with "public" tag AND IDs starting with "api.".

**Boundary Conditions:**
- One filter matches no modules: Result is empty.
- Both filters match different sets: Result is their intersection.

**Error Conditions:** None.

---

### 3.12 FR-HEALTH: Health Check Requirements

---

#### FR-HEALTH-001: Expose /health endpoint on HTTP transports

| Field | Value |
|-------|-------|
| **ID** | FR-HEALTH-001 |
| **Title** | HTTP health check endpoint returns server status |
| **Priority** | P2 |
| **Traces to** | F-019 |

**Description:** When the MCP Server runs with Streamable HTTP or SSE transport, a `GET /health` endpoint shall be available that returns HTTP 200 with a JSON body containing:
- `"status"`: string, always `"ok"`.
- `"module_count"`: integer, number of currently registered tools.
- `"uptime_seconds"`: float, seconds since server start.

**Input/Trigger:** HTTP `GET /health` request.

**Expected Output:** HTTP 200, `Content-Type: application/json`, body: `{"status": "ok", "module_count": 5, "uptime_seconds": 123.45}`.

**Boundary Conditions:**
- Server just started (< 1 second uptime): `uptime_seconds` is a small positive float.
- Zero tools registered: `module_count=0`.
- Transport is stdio: Endpoint not available (not applicable).

**Error Conditions:** None. The health endpoint shall always return HTTP 200 if the server is running.

---

#### FR-HEALTH-002: Health endpoint requires no authentication

| Field | Value |
|-------|-------|
| **ID** | FR-HEALTH-002 |
| **Title** | Health endpoint accessible without authentication |
| **Priority** | P2 |
| **Traces to** | F-019 |

**Description:** The `/health` endpoint shall not require any authentication or authorization. It shall be accessible to any HTTP client.

**Input/Trigger:** Unauthenticated `GET /health` request.

**Expected Output:** HTTP 200 response (same as FR-HEALTH-001).

**Boundary Conditions:** None.

**Error Conditions:** None.

---

### 3.13 FR-RESOURCE: MCP Resource Requirements

---

#### FR-RESOURCE-001: Expose module documentation as MCP Resources

| Field | Value |
|-------|-------|
| **ID** | FR-RESOURCE-001 |
| **Title** | Modules with documentation exposed as MCP Resources |
| **Priority** | P2 |
| **Traces to** | F-020 |

**Description:** Modules with a non-empty `documentation` field in their `ModuleDescriptor` shall be exposed as MCP Resources. Each resource shall be named `docs://{module_id}` and contain the documentation text as plain text content.

**Input/Trigger:** MCP `resources/list` or `resources/read` request.

**Expected Output:**
- `resources/list`: Returns list of resources, one per module with documentation.
- `resources/read` for `docs://image.resize`: Returns the documentation text.

**Boundary Conditions:**
- Module with `documentation=None`: No resource generated.
- Module with `documentation=""` (empty string): No resource generated.
- Module with non-empty documentation: Resource generated with URI `docs://{module_id}`.

**Error Conditions:**
- Read request for non-existent resource URI: Return MCP error "Resource not found".

---

#### FR-RESOURCE-002: Modules without documentation do not generate resources

| Field | Value |
|-------|-------|
| **ID** | FR-RESOURCE-002 |
| **Title** | No MCP Resource for modules lacking documentation |
| **Priority** | P2 |
| **Traces to** | F-020 |

**Description:** Modules whose `ModuleDescriptor.documentation` is `None` or empty string `""` shall NOT produce an MCP Resource entry. Only modules with substantive documentation content shall appear in the resource list.

**Input/Trigger:** Module with `documentation=None` or `documentation=""`.

**Expected Output:** No resource entry for this module in `resources/list` response.

**Boundary Conditions:** None.

**Error Conditions:** None.

---

#### FR-STREAM-001: Streaming tool execution via notifications/progress

| Field | Value |
|-------|-------|
| **ID** | FR-STREAM-001 |
| **Title** | Streaming tool execution via notifications/progress |
| **Priority** | P1 |

**Description:** When ALL of these conditions are met:
1. The Executor implements a `stream()` method
2. The client provides `_meta.progressToken` in the `tools/call` request
3. The module descriptor has `annotations.streaming = true`

Then the server SHALL:
1. Call `executor.stream(toolName, args)` instead of `executor.call(toolName, args)`
2. For each yielded chunk, send `notifications/progress` with:
   - `progressToken`: the client-provided token
   - `progress`: monotonically increasing integer (1, 2, 3, ...)
   - `message`: JSON-serialized chunk content
3. After iteration completes, return a standard `CallToolResult` with the accumulated complete result

When ANY condition is not met, fall back to normal atomic `executor.call()`.

**Boundary Conditions:**
- `progressToken` not provided: Normal atomic execution.
- Executor has no `stream()`: Normal atomic execution.
- Stream yields zero chunks: Return empty accumulated result `{}`.
- Stream throws mid-iteration: Map error via `ErrorMapper`, return error result.

---

### 3.14 FR-EXT: Extension Helpers

Extension helpers provide convenient wrappers for MCP-specific protocol features that module implementations can use during tool execution. These helpers abstract MCP protocol details so that module code does not need to interact with the MCP SDK directly.

---

#### FR-EXT-001: report_progress() sends MCP progress notifications

| Field | Value |
|-------|-------|
| **ID** | FR-EXT-001 |
| **Title** | report_progress() sends MCP progress notifications to the client |
| **Priority** | P1 |
| **Traces to** | F-005 |

**Description:** The `report_progress(context, progress, total=None, message=None)` helper function shall send an MCP `notifications/progress` notification to the connected client. The `context` parameter is the MCP server request context (made available to module execution via the apcore context pipeline). The `progress` parameter is a numeric value indicating current progress. The optional `total` parameter indicates the total expected value (enabling percentage calculation). The optional `message` parameter provides a human-readable progress description.

This function is a no-op (silently does nothing) when invoked outside an MCP context (e.g., when the module is called directly via `Executor.call()` without an MCP server).

**Signature:**
```python
async def report_progress(
    context: Any,
    progress: float,
    total: float | None = None,
    message: str | None = None,
) -> None: ...
```

**Input/Trigger:** Module implementation calls `await report_progress(context, 50, total=100, message="Processing image...")` during execution.

**Expected Output:** MCP `notifications/progress` notification sent to the client with `progress=50`, `total=100`, and `message="Processing image..."`.

**Boundary Conditions:**
- `total=None`: Progress notification sent without total (client cannot compute percentage).
- `message=None`: Progress notification sent without message text.
- No active MCP context (e.g., direct Executor call): No-op, no error raised.
- `progress` > `total`: Allowed (client may clamp or ignore).

**Error Conditions:**
- Silently no-ops when context has no MCP progress callback. No validation on progress value.

---

#### FR-EXT-002: elicit() solicits user input via MCP elicitation

| Field | Value |
|-------|-------|
| **ID** | FR-EXT-002 |
| **Title** | elicit() solicits user input from the MCP client |
| **Priority** | P1 |
| **Traces to** | F-005 |

**Description:** The `elicit(context, message, requested_schema=None)` helper function shall send an MCP elicitation request to the connected client, soliciting user input. The `context` parameter is the MCP server request context. The `message` parameter is a human-readable prompt displayed to the user. The optional `requested_schema` parameter is a JSON Schema dict describing the expected shape of the user's response.

The function returns the client's elicitation response. If the MCP client does not support elicitation or the user declines, the function returns `None`.

Returns `None` when called outside MCP context (graceful no-op).

**Signature:**
```python
async def elicit(
    context: Any,
    message: str,
    requested_schema: dict[str, Any] | None = None,
) -> Any | None: ...
```

**Input/Trigger:** Module implementation calls `await elicit(context, "Please confirm deletion", {"type": "object", "properties": {"confirmed": {"type": "boolean"}}})` during execution.

**Expected Output:** MCP elicitation request sent to client. Returns the user's response dict (e.g., `{"confirmed": True}`) or `None` if declined/unsupported.

**Boundary Conditions:**
- `requested_schema=None`: Client presents a free-form input prompt.
- Client does not support elicitation: Returns `None`.
- User declines or cancels: Returns `None`.
- Empty `message` (`""`): Allowed. No validation is performed on the message parameter.

**Error Conditions:**
- Called outside MCP context: Returns `None` (graceful no-op).

---

#### FR-EXT-003: MCP_PROGRESS_KEY and MCP_ELICIT_KEY constants

| Field | Value |
|-------|-------|
| **ID** | FR-EXT-003 |
| **Title** | Module-level constants for MCP context keys |
| **Priority** | P1 |
| **Traces to** | F-005 |

**Description:** The `apcore_mcp` package shall export two string constants used as keys for storing MCP protocol objects in the apcore execution context:

- `MCP_PROGRESS_KEY` -- The context key under which the MCP progress reporting capability is stored. Used internally by `report_progress()` to retrieve the progress notification sender from the execution context.
- `MCP_ELICIT_KEY` -- The context key under which the MCP elicitation capability is stored. Used internally by `elicit()` to retrieve the elicitation sender from the execution context.

These constants allow advanced users to access the underlying MCP protocol objects directly when the helper functions do not cover their use case, while providing a stable, documented key name.

**Values:**
```python
MCP_PROGRESS_KEY: str = "_mcp_progress"
MCP_ELICIT_KEY: str = "_mcp_elicit"
```

**Boundary Conditions:**
- Constants are module-level and immutable (plain `str`).
- Importing these constants does not require an active MCP server.

**Error Conditions:** None.

---

### 3.15 FR-EXPLORER: MCP Tool Explorer Requirements

The MCP Tool Explorer is an optional, built-in browser UI that allows developers to browse, inspect, and test MCP tools. It is only available with HTTP-based transports and must be explicitly enabled.

---

#### FR-EXPLORER-001: Serve Explorer UI when explorer=True

| Field | Value |
|-------|-------|
| **ID** | FR-EXPLORER-001 |
| **Title** | Serve Explorer HTML UI at `explorer_prefix` when enabled |
| **Priority** | P2 |
| **Traces to** | F-026 |

**Description:** When `explorer=True` is passed to `serve()` (or `--explorer` on CLI) and the transport is HTTP-based (Streamable HTTP or SSE), the server SHALL mount a self-contained HTML page at `GET /explorer/` (or the configured `explorer_prefix`) that lists all registered tools with their descriptions and annotations. The `explorer_prefix` parameter is configurable (default `/explorer`).

**Input/Trigger:** `serve(registry, transport="streamable-http", explorer=True)`

**Expected Output:** `GET /explorer/` returns HTTP 200 with `Content-Type: text/html` containing the Explorer UI.

**Boundary Conditions:**
- `explorer=True` with stdio transport: Explorer is silently ignored (no HTTP server).
- `explorer=False` (default): `GET /explorer/` is not mounted; server behaves as before.

---

#### FR-EXPLORER-002: JSON tool list endpoint

| Field | Value |
|-------|-------|
| **ID** | FR-EXPLORER-002 |
| **Title** | GET /explorer/tools returns JSON array of tool summaries |
| **Priority** | P2 |
| **Traces to** | F-026 |

**Description:** When the Explorer is enabled, `GET /explorer/tools` SHALL return a JSON array where each element contains `name` (string), `description` (string), and `annotations` (object with hint booleans).

**Expected Output:**
```json
[
  {
    "name": "image-resize",
    "description": "Resize an image to specified dimensions",
    "annotations": {
      "readOnlyHint": false,
      "destructiveHint": false,
      "idempotentHint": true,
      "openWorldHint": true
    }
  }
]
```

---

#### FR-EXPLORER-003: JSON tool detail endpoint

| Field | Value |
|-------|-------|
| **ID** | FR-EXPLORER-003 |
| **Title** | GET /explorer/tools/<name> returns full tool detail with inputSchema |
| **Priority** | P2 |
| **Traces to** | F-026 |

**Description:** When the Explorer is enabled, `GET /explorer/tools/<name>` SHALL return a JSON object with full tool metadata including `name`, `description`, `annotations`, and `inputSchema`. Returns HTTP 404 if the tool name is not found.

---

#### FR-EXPLORER-004: Tool execution endpoint with allow_execute guard

| Field | Value |
|-------|-------|
| **ID** | FR-EXPLORER-004 |
| **Title** | POST /explorer/tools/<name>/call executes a tool via the Executor pipeline |
| **Priority** | P2 |
| **Traces to** | F-026 |

**Description:** When the Explorer is enabled and `allow_execute=True`, `POST /explorer/tools/<name>/call` SHALL accept a JSON body as tool input, execute the tool through the full Executor pipeline (ACL, validation, middleware), and return the result as JSON. When `allow_execute=False` (default), this endpoint SHALL return HTTP 403.

**Expected Output (success):**
```json
{"result": {"width": 800, "height": 600}}
```

**Error Conditions:**
- `allow_execute=False`: HTTP 403 with `{"error": "Tool execution is disabled"}`.
- Tool not found: HTTP 404 with `{"error": "Tool 'foo' not found"}`.
- Validation failure: HTTP 400 with `{"error": "Input validation failed: ..."}`.
- Execution error: HTTP 500 with `{"error": "..."}`.

---

#### FR-EXPLORER-005: Self-contained HTML asset with no external dependencies

| Field | Value |
|-------|-------|
| **ID** | FR-EXPLORER-005 |
| **Title** | Explorer HTML/JS is a single self-contained file |
| **Priority** | P2 |
| **Traces to** | F-026 |

**Description:** The Explorer UI SHALL be a single HTML file with inline CSS and JavaScript, with no external CDN dependencies. This file is language-agnostic and can be shared across all `apcore-mcp-{lang}` implementations. Each language implementation embeds this file as a static asset in its package.

---

#### FR-EXPLORER-006: Configurable explorer_prefix parameter

| Field | Value |
|-------|-------|
| **ID** | FR-EXPLORER-006 |
| **Title** | `explorer_prefix` parameter configurable with default `/explorer` |
| **Priority** | P2 |
| **Traces to** | F-026 |

**Description:** The `serve()` function SHALL accept an `explorer_prefix` parameter (default `/explorer`) that controls the URL prefix under which all Explorer endpoints are mounted. When set to a custom value (e.g., `/custom`), endpoints SHALL be available at `GET /custom/`, `GET /custom/tools`, `GET /custom/tools/<name>`, and `POST /custom/tools/<name>/call`.

**Input/Trigger:** `serve(registry, transport="streamable-http", explorer=True, explorer_prefix="/custom")`

**Expected Output:** All Explorer endpoints are mounted under `/custom/` instead of the default `/explorer/`.

**Boundary Conditions:**
- Default value is `/explorer`.
- Prefix must start with `/`.
- Trailing slash is normalized (both `/explorer` and `/explorer/` are accepted).

---

## 4. Specific Requirements -- Non-Functional Requirements

### 4.1 NFR-PERF: Performance Requirements

---

#### NFR-PERF-001: Schema conversion time for 100 modules

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-001 |
| **Title** | Schema conversion completes within 100ms for 100 modules |
| **Target** | < 100 milliseconds |
| **Measurement** | Benchmark: time `MCPServerFactory.build_tools(registry)` with a registry of 100 modules, each with a 10-property input schema |
| **Traces to** | F-001 |

**Description:** Converting 100 apcore module schemas to MCP Tool definitions shall complete in less than 100 milliseconds on a standard development machine (4-core CPU, 16GB RAM).

---

#### NFR-PERF-002: Tool call routing overhead

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-002 |
| **Title** | Tool call routing adds less than 5ms overhead |
| **Target** | < 5 milliseconds added beyond Executor.call_async() time |
| **Measurement** | Benchmark: measure time from `handle_call()` entry to return, subtract time spent in `Executor.call_async()`. Average over 1000 calls. |
| **Traces to** | F-003 |

**Description:** The apcore-mcp routing layer (argument parsing, executor dispatch, output serialization, error mapping) shall add less than 5 milliseconds of latency beyond the time spent in the apcore Executor pipeline.

---

#### NFR-PERF-003: Memory overhead for 100 tools

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-003 |
| **Title** | Memory overhead below 10MB for 100 registered tools |
| **Target** | < 10 MB |
| **Measurement** | Memory profiling: measure process memory before and after registering 100 tools with typical schemas |
| **Traces to** | F-001, F-005 |

**Description:** The memory consumed by 100 MCP Tool definition objects (including their JSON Schema dicts) shall not exceed 10 megabytes.

---

#### NFR-PERF-004: Concurrent HTTP connection handling

| Field | Value |
|-------|-------|
| **ID** | NFR-PERF-004 |
| **Title** | Support 10+ simultaneous MCP client connections over HTTP |
| **Target** | >= 10 concurrent connections |
| **Measurement** | Load test: 10 concurrent MCP clients each sending tool calls simultaneously; all receive correct responses |
| **Traces to** | F-007 |

**Description:** When using Streamable HTTP transport, the server shall handle at least 10 simultaneous MCP client connections without errors or response degradation. Concurrency handling is delegated to the MCP SDK's HTTP server implementation.

---

### 4.2 NFR-SEC: Security Requirements

---

#### NFR-SEC-001: ACL enforcement delegated to apcore Executor

| Field | Value |
|-------|-------|
| **ID** | NFR-SEC-001 |
| **Title** | All access control enforced via apcore Executor ACL |
| **Target** | 100% of tool calls pass through Executor ACL check |
| **Measurement** | Code review: verify no code path bypasses Executor.call_async() for tool execution |
| **Traces to** | F-003, F-014 |

**Description:** apcore-mcp shall not implement any custom authentication or authorization mechanism. All access control shall be delegated to the apcore Executor's built-in ACL. Every tool call shall pass through `Executor.call_async()` which performs ACL checking at step 4 of its pipeline.

---

#### NFR-SEC-002: Error responses do not leak sensitive data

| Field | Value |
|-------|-------|
| **ID** | NFR-SEC-002 |
| **Title** | No stack traces, caller IDs, or internal paths in MCP error responses |
| **Target** | Zero sensitive data in any error response |
| **Measurement** | Security review: test all error paths and verify response content. Automated tests asserting absence of stack traces, file paths, caller IDs. |
| **Traces to** | F-004 |

**Description:** MCP error responses shall not contain: (a) Python stack traces, (b) internal file paths, (c) `caller_id` values from ACL errors, (d) exception class names for unexpected errors, (e) any data marked `x-sensitive` in apcore schemas. Full error details shall only appear in server-side logs.

---

#### NFR-SEC-003: No sensitive data in MCP tool definitions

| Field | Value |
|-------|-------|
| **ID** | NFR-SEC-003 |
| **Title** | Tool definitions do not expose sensitive schema metadata |
| **Target** | Zero `x-sensitive` markers exposed to MCP clients |
| **Measurement** | Review: verify tool `inputSchema` does not include implementation-private schema extensions |
| **Traces to** | F-001 |

**Description:** MCP Tool `inputSchema` and OpenAI `parameters` shall faithfully reproduce the JSON Schema from apcore modules. The `x-sensitive` extension on individual fields is metadata for server-side redaction; it may appear in schemas (as it describes the field's nature) but the actual sensitive data values shall never appear in error messages or logs.

---

### 4.3 NFR-REL: Reliability Requirements

---

#### NFR-REL-001: Graceful shutdown on SIGINT/SIGTERM

| Field | Value |
|-------|-------|
| **ID** | NFR-REL-001 |
| **Title** | Server shuts down gracefully on termination signals |
| **Target** | Clean shutdown within 5 seconds of signal receipt |
| **Measurement** | Integration test: send SIGINT to running server, verify `serve()` returns within 5 seconds and no resources are leaked |
| **Traces to** | F-005, F-006 |

**Description:** Upon receiving SIGINT or SIGTERM, the MCP Server shall stop accepting new connections, complete any in-flight tool calls (up to 5-second grace period), and shut down the transport cleanly. The `serve()` function shall return normally.

---

#### NFR-REL-002: Single module failure does not prevent server start

| Field | Value |
|-------|-------|
| **ID** | NFR-REL-002 |
| **Title** | Malformed module schema does not block other tool registrations |
| **Target** | Server starts with N-1 tools when 1 of N modules has invalid schema |
| **Measurement** | Unit test: register 5 modules, one with circular $ref; verify server starts with 4 tools |
| **Traces to** | F-001, F-005 |

**Description:** If schema conversion fails for one module during server startup (e.g., circular `$ref`), that module shall be skipped with a WARNING log, and the remaining modules shall be registered as tools. The server shall not fail to start due to a single module's schema error.

---

#### NFR-REL-003: Thread-safe tool list access

| Field | Value |
|-------|-------|
| **ID** | NFR-REL-003 |
| **Title** | Concurrent tool list access does not cause data corruption |
| **Target** | Zero data races under concurrent access |
| **Measurement** | Stress test: concurrent tool calls and registry modifications; verify no exceptions or inconsistent states |
| **Traces to** | F-015 |

**Description:** The tools dictionary in `RegistryListener` shall be protected by a `threading.Lock` to prevent data races between the async event loop thread (reading tools for `tools/list`) and Registry callback threads (writing tools on `register`/`unregister`).

---

### 4.4 NFR-MAINT: Maintainability Requirements

---

#### NFR-MAINT-001: Core logic size limit

| Field | Value |
|-------|-------|
| **ID** | NFR-MAINT-001 |
| **Title** | Core logic under 1,200 lines of code |
| **Target** | 500-1,200 lines (excluding tests and documentation) |
| **Measurement** | `cloc src/apcore_mcp/ --exclude-dir=tests` |
| **Traces to** | PRD Section 5.3 |

**Description:** The total lines of code in `src/apcore_mcp/` (excluding tests and documentation) shall not exceed 1,200 lines. This enforces the thin adapter design principle.

---

#### NFR-MAINT-002: Test coverage target

| Field | Value |
|-------|-------|
| **ID** | NFR-MAINT-002 |
| **Title** | Test coverage at or above 90% |
| **Target** | >= 90% line coverage |
| **Measurement** | `pytest --cov=apcore_mcp --cov-report=term` |
| **Traces to** | PRD Section 5.3 |

**Description:** Automated tests shall achieve at least 90% line coverage on all code in `src/apcore_mcp/`. Coverage shall be measured using `pytest-cov`.

---

#### NFR-MAINT-003: Full type annotations on public APIs

| Field | Value |
|-------|-------|
| **ID** | NFR-MAINT-003 |
| **Title** | All public functions and methods have complete type annotations |
| **Target** | 100% of public APIs annotated |
| **Measurement** | `mypy src/apcore_mcp/ --strict` passes with zero errors |
| **Traces to** | PRD Section 7.2 |

**Description:** All public functions (`serve()`, `to_openai_tools()`), all public class methods, and all public class attributes shall have complete Python type annotations compatible with `mypy --strict` and `pyright`.

---

#### NFR-MAINT-004: Logging standards

| Field | Value |
|-------|-------|
| **ID** | NFR-MAINT-004 |
| **Title** | Consistent logging format and namespace usage |
| **Target** | All modules use `apcore_mcp.*` logger namespace |
| **Measurement** | Code review: verify every `logging.getLogger()` call uses `__name__` or explicit `apcore_mcp.*` name |
| **Traces to** | F-016 |

**Description:** Every Python module in apcore-mcp shall obtain its logger via `logging.getLogger(__name__)`, which produces logger names under the `apcore_mcp` namespace. No module shall use the root logger or a logger outside the `apcore_mcp` hierarchy.

---

### 4.5 NFR-COMPAT: Compatibility Requirements

---

#### NFR-COMPAT-001: Python version compatibility

| Field | Value |
|-------|-------|
| **ID** | NFR-COMPAT-001 |
| **Title** | Compatible with Python 3.10 and above |
| **Target** | Python >= 3.10 |
| **Measurement** | CI test matrix: Python 3.10, 3.11, 3.12, 3.13 |
| **Traces to** | PRD Section 8.3 |

**Description:** apcore-mcp shall be compatible with Python 3.10, 3.11, 3.12, and 3.13. The `pyproject.toml` shall declare `requires-python = ">=3.10"`.

---

#### NFR-COMPAT-002: apcore-python version compatibility

| Field | Value |
|-------|-------|
| **ID** | NFR-COMPAT-002 |
| **Title** | Compatible with apcore-python >= 0.2.0 |
| **Target** | apcore >= 0.2.0, < 1.0 |
| **Measurement** | Integration tests against latest apcore-python release |
| **Traces to** | PRD Section 8.3 |

**Description:** apcore-mcp shall declare a dependency on `apcore>=0.2.0,<1.0` and shall be tested against the latest release within that range.

---

#### NFR-COMPAT-003: MCP SDK version compatibility

| Field | Value |
|-------|-------|
| **ID** | NFR-COMPAT-003 |
| **Title** | Compatible with MCP Python SDK >= 1.0.0 |
| **Target** | mcp >= 1.0.0, < 2.0 |
| **Measurement** | Integration tests against latest mcp SDK release |
| **Traces to** | PRD Section 8.3 |

**Description:** apcore-mcp shall declare a dependency on `mcp>=1.0.0,<2.0` and shall be tested against the latest release within that range.

---

#### NFR-COMPAT-004: MCP client compatibility

| Field | Value |
|-------|-------|
| **ID** | NFR-COMPAT-004 |
| **Title** | Verified with Claude Desktop and at least one additional MCP client |
| **Target** | >= 2 verified MCP clients |
| **Measurement** | Manual integration testing with Claude Desktop and Cursor (or another MCP client) |
| **Traces to** | PRD Section 5.3 |

**Description:** apcore-mcp shall be verified working with Claude Desktop (stdio transport) and at least one additional MCP client (e.g., Cursor, Windsurf) before v0.1.0 release.

---

### 4.6 NFR-PORT: Portability Requirements

---

#### NFR-PORT-001: OS compatibility

| Field | Value |
|-------|-------|
| **ID** | NFR-PORT-001 |
| **Title** | Compatible with macOS, Linux, and Windows |
| **Target** | All three major desktop platforms |
| **Measurement** | CI test matrix: macOS (latest), Ubuntu (latest), Windows (latest) |
| **Traces to** | General portability requirement |

**Description:** apcore-mcp shall function correctly on macOS, Linux, and Windows. No platform-specific system calls or dependencies shall be used.

---

#### NFR-PORT-002: No platform-specific dependencies

| Field | Value |
|-------|-------|
| **ID** | NFR-PORT-002 |
| **Title** | Zero platform-specific package dependencies |
| **Target** | All dependencies are pure Python or cross-platform |
| **Measurement** | Review `pyproject.toml` dependencies for platform markers |
| **Traces to** | General portability requirement |

**Description:** All runtime dependencies of apcore-mcp (`apcore`, `mcp`) shall be cross-platform Python packages. No platform-conditional dependencies shall be declared.

---

## 5. Use Cases

### UC-001: Start MCP Server via serve()

| Field | Value |
|-------|-------|
| **Use Case ID** | UC-001 |
| **Title** | Start MCP Server via Python serve() call |
| **Primary Actor** | Module Developer (Maya) |
| **Traces to FRs** | FR-SERVER-001, FR-SERVER-004, FR-SERVER-005, FR-SERVER-006, FR-SERVER-011, FR-SCHEMA-001, FR-ANNOT-001 through FR-ANNOT-006 |

**Preconditions:**
1. apcore-mcp is installed (`pip install apcore-mcp`).
2. An apcore `Registry` exists with at least one registered module.

**Main Success Scenario:**
1. Developer creates a `Registry` and calls `registry.discover()`.
2. Developer calls `serve(registry)`.
3. System validates the `registry` parameter is a `Registry` instance.
4. System creates a default `Executor(registry)`.
5. System iterates over all modules via `registry.list()`.
6. For each module, system calls `registry.get_definition(module_id)` and builds an MCP `Tool` via `SchemaConverter` and `AnnotationMapper`.
7. System creates an MCP Server with name `"apcore-mcp"` and package version.
8. System registers `list_tools` and `call_tool` handlers on the server.
9. System starts stdio transport.
10. System logs: "apcore-mcp server started: N tools registered, transport=stdio".
11. System blocks until shutdown signal received.
12. System returns `None`.

**Alternative Flows:**
- **A1: HTTP transport**: At step 2, developer calls `serve(registry, transport="streamable-http", port=9000)`. Steps 9-10 use Streamable HTTP transport instead of stdio.
- **A2: Executor passthrough**: At step 2, developer passes an `Executor` instance. Step 4 is skipped; the provided Executor is used directly.
- **A3: Custom server name**: At step 2, developer passes `name="my-tools"`. Step 7 uses the custom name.

**Exception Flows:**
- **E1: Invalid parameter type**: At step 3, the parameter is neither `Registry` nor `Executor`. System raises `TypeError`. Flow terminates.
- **E2: Invalid transport**: At step 2, `transport="websocket"`. System raises `ValueError`. Flow terminates.
- **E3: Port in use**: At step 9 (HTTP transport), the port is occupied. System raises `OSError`. Flow terminates.
- **E4: Schema conversion failure**: At step 6, one module has circular `$ref`. System logs warning, skips that module, continues with remaining modules.

**Postconditions:**
1. MCP Server was running and accepting connections during its lifetime.
2. All valid modules were registered as MCP tools.
3. Server has shut down cleanly.

---

### UC-002: MCP Client discovers tools

| Field | Value |
|-------|-------|
| **Use Case ID** | UC-002 |
| **Title** | MCP Client discovers available tools |
| **Primary Actor** | AI Agent Builder (Alex) |
| **Traces to FRs** | FR-SERVER-011, FR-SCHEMA-001, FR-ANNOT-001 through FR-ANNOT-006 |

**Preconditions:**
1. MCP Server is running (UC-001 completed successfully).
2. An MCP client is connected to the server.

**Main Success Scenario:**
1. MCP client sends `tools/list` request.
2. Server invokes `list_tools` handler.
3. Handler returns the list of `Tool` objects built during server startup.
4. Client receives a list where each tool has `name`, `description`, `inputSchema`, and `annotations`.
5. Client displays the tools to the user or AI model.

**Alternative Flows:**
- **A1: Empty registry**: At step 3, the list is empty. Client receives `[]`.
- **A2: After dynamic registration**: The tool list includes tools added after server start (per FR-DYNAMIC-001).

**Exception Flows:** None. The `tools/list` operation does not fail under normal conditions.

**Postconditions:**
1. Client has an accurate list of available tools with schemas and annotations.

---

### UC-003: MCP Client calls a tool (success)

| Field | Value |
|-------|-------|
| **Use Case ID** | UC-003 |
| **Title** | MCP Client successfully invokes a tool |
| **Primary Actor** | AI Agent Builder (Alex) |
| **Traces to FRs** | FR-EXEC-001, FR-EXEC-002, FR-EXEC-003, FR-EXEC-004 |

**Preconditions:**
1. MCP Server is running with at least one tool registered.
2. An MCP client has discovered the tools (UC-002).

**Main Success Scenario:**
1. MCP client sends `tools/call` request with `name="image.resize"` and `arguments={"width": 800, "height": 600}`.
2. Server invokes `call_tool` handler.
3. Handler delegates to `ExecutionRouter.handle_call("image.resize", {"width": 800, "height": 600})`.
4. Router calls `Executor.call_async("image.resize", {"width": 800, "height": 600})`.
5. Executor executes the full 10-step pipeline (context, safety, lookup, ACL, validation, middleware before, execute, output validation, middleware after, return).
6. Module returns `{"status": "ok", "path": "/out/resized.png"}`.
7. Router serializes output to JSON: `'{"status":"ok","path":"/out/resized.png"}'`.
8. Router returns `CallToolResult(content=[TextContent(type="text", text=<json>)], isError=False)`.
9. Client receives successful result.

**Alternative Flows:**
- **A1: Async module**: At step 5, the module's `execute()` is a coroutine. Executor calls it directly (no thread bridging needed).
- **A2: Empty arguments**: At step 1, `arguments={}`. Router passes empty dict to Executor.

**Exception Flows:** See UC-004.

**Postconditions:**
1. Module executed successfully with full pipeline.
2. Client received JSON-serialized output.

---

### UC-004: MCP Client calls a tool (error)

| Field | Value |
|-------|-------|
| **Use Case ID** | UC-004 |
| **Title** | MCP Client tool call returns an error |
| **Primary Actor** | AI Agent Builder (Alex) |
| **Traces to FRs** | FR-ERROR-001 through FR-ERROR-011, FR-EXEC-007 |

**Preconditions:**
1. MCP Server is running.
2. An MCP client sends a tool call that triggers an error condition.

**Main Success Scenario (SchemaValidationError):**
1. Client sends `tools/call` with `name="image.resize"` and `arguments={"width": "not_a_number"}`.
2. Router calls `Executor.call_async()`.
3. Executor raises `SchemaValidationError` at pipeline step 5.
4. Router catches the error, delegates to `ErrorMapper.to_mcp_error()`.
5. ErrorMapper formats field-level errors: "Input validation failed:\n- width: Input should be a valid integer (int_type)".
6. Router returns `CallToolResult(isError=True, content=[TextContent(text=<error_message>)])`.
7. Client receives error response.

**Alternative Flows:**
- **A1: ModuleNotFoundError**: Client calls non-existent tool `"foo.bar"`. Response: "Module not found: foo.bar".
- **A2: ACLDeniedError**: Executor ACL denies the call. Response: "Access denied" (no caller_id leaked).
- **A3: ModuleTimeoutError**: Module exceeds timeout. Response: "Module timed out after 30000ms".
- **A4: Unexpected Exception**: Module raises `RuntimeError`. Response: "Internal error occurred". Full trace logged at ERROR level.

**Exception Flows:** None. All errors are caught and returned as `CallToolResult`.

**Postconditions:**
1. Error response sent to client with `isError=True`.
2. For unexpected errors, full stack trace logged server-side.
3. No sensitive data leaked to client.

---

### UC-005: Export OpenAI tools

| Field | Value |
|-------|-------|
| **Use Case ID** | UC-005 |
| **Title** | Export apcore Registry as OpenAI tool definitions |
| **Primary Actor** | Module Developer (Maya) |
| **Traces to FRs** | FR-OPENAI-001 through FR-OPENAI-005, FR-SCHEMA-004 |

**Preconditions:**
1. apcore-mcp is installed.
2. An apcore `Registry` exists with registered modules.

**Main Success Scenario:**
1. Developer calls `to_openai_tools(registry)`.
2. System validates the parameter is a `Registry` or `Executor`.
3. System iterates over all modules via `registry.list()`.
4. For each module, system obtains `ModuleDescriptor` and converts to OpenAI dict.
5. Module ID is normalized (dots replaced with `-`).
6. Schema is converted (same $ref inlining as MCP path).
7. System returns `list[dict]`.
8. Developer passes result to `openai.chat.completions.create(tools=tools)`.

**Alternative Flows:**
- **A1: With annotations**: Developer calls `to_openai_tools(registry, embed_annotations=True)`. Annotation suffix appended to descriptions.
- **A2: With strict mode**: Developer calls `to_openai_tools(registry, strict=True)`. Schemas transformed for strict mode, `"strict": true` added.
- **A3: With filtering**: Developer calls `to_openai_tools(registry, tags=["image"], prefix="comfyui.")`. Only matching modules included.

**Exception Flows:**
- **E1: Invalid parameter type**: `TypeError` raised.
- **E2: Schema conversion failure for one module**: Module skipped with warning, others included.

**Postconditions:**
1. Developer has a list of OpenAI-compatible tool dicts.
2. Dicts are directly usable with OpenAI API.

---

### UC-006: Start MCP Server via CLI

| Field | Value |
|-------|-------|
| **Use Case ID** | UC-006 |
| **Title** | Start MCP Server via command-line interface |
| **Primary Actor** | AI Agent Builder (Alex) |
| **Traces to FRs** | FR-CLI-001 through FR-CLI-008 |

**Preconditions:**
1. apcore-mcp is installed.
2. An extensions directory exists with apcore module files.

**Main Success Scenario:**
1. User runs `python -m apcore_mcp --extensions-dir ./extensions`.
2. CLI parses arguments.
3. CLI validates `--extensions-dir` exists and is a directory.
4. CLI creates `Registry(extensions_dir="./extensions")`.
5. CLI calls `registry.discover()`.
6. CLI calls `serve(registry, transport="stdio")`.
7. Server starts and blocks.
8. User terminates with Ctrl+C.
9. Server shuts down, CLI exits with code 0.

**Alternative Flows:**
- **A1: HTTP transport**: User adds `--transport streamable-http --port 9000`. Server starts on HTTP.
- **A2: Custom name and log level**: User adds `--name "my-tools" --log-level DEBUG`.

**Exception Flows:**
- **E1: Directory not found**: CLI prints error to stderr, exits code 1.
- **E2: Port in use**: Server fails to start, CLI exits code 2.
- **E3: Invalid arguments**: argparse prints error, CLI exits code 2.
- **E4: No modules discovered**: Warning logged, server starts with zero tools.

**Postconditions:**
1. Server ran and shut down cleanly.
2. Exit code reflects the outcome.

---

### UC-007: Dynamic module registration

| Field | Value |
|-------|-------|
| **Use Case ID** | UC-007 |
| **Title** | Add module to registry while MCP server is running |
| **Primary Actor** | Module Developer (Maya) |
| **Traces to FRs** | FR-DYNAMIC-001 through FR-DYNAMIC-004 |

**Preconditions:**
1. MCP Server is running (UC-001 completed).
2. An MCP client is connected.

**Main Success Scenario:**
1. Developer calls `registry.register("new.tool", module_instance)`.
2. Registry emits "register" event.
3. `RegistryListener._on_register()` callback fires.
4. Listener obtains `ModuleDescriptor` via `registry.get_definition("new.tool")`.
5. Listener builds MCP `Tool` via `MCPServerFactory.build_tool()`.
6. Listener adds tool to internal tools dict (under lock).
7. Listener triggers `notifications/tools/list_changed`.
8. MCP client re-fetches tool list and sees "new.tool".

**Alternative Flows:**
- **A1: Unregister module**: Developer calls `registry.unregister("old.tool")`. Listener removes tool from dict, sends notification.

**Exception Flows:**
- **E1: get_definition returns None**: Listener logs warning, does not add tool.
- **E2: build_tool raises ValueError**: Listener logs warning, does not add tool.

**Postconditions:**
1. MCP tool list updated to reflect registry state.
2. MCP client notified of change.

---

### UC-008: OpenAI tools with strict mode

| Field | Value |
|-------|-------|
| **Use Case ID** | UC-008 |
| **Title** | Export OpenAI tool definitions with strict mode enabled |
| **Primary Actor** | AI Agent Builder (Alex) |
| **Traces to FRs** | FR-OPENAI-003 |

**Preconditions:**
1. apcore-mcp is installed.
2. Registry has modules with JSON Schema input schemas.

**Main Success Scenario:**
1. Developer calls `to_openai_tools(registry, strict=True)`.
2. System iterates over modules.
3. For each module, system converts schema and applies strict mode transformations:
   a. Sets `additionalProperties: false` on all object types.
   b. Makes all properties required.
   c. Converts optional properties to nullable types.
   d. Strips `x-*` extensions and `default` values.
4. System adds `"strict": true` to each function definition dict.
5. System returns the list.
6. Developer uses result with `openai.chat.completions.create(tools=tools)`.
7. OpenAI model produces outputs exactly matching the schema.

**Alternative Flows:**
- **A1: strict=False**: No strict mode transformations applied. No `"strict"` key in output.

**Exception Flows:**
- **E1: Schema incompatible with strict mode**: WARNING logged (e.g., "Schema for module 'x' uses additionalProperties: true, which is incompatible with strict mode"). Tool is still included with transformations applied.

**Postconditions:**
1. All tool definitions have `"strict": true`.
2. Schemas are strict-mode-compliant.

---

## 6. CRUD Matrix

| Entity | Create | Read | Update | Delete |
|--------|--------|------|--------|--------|
| **MCP Tool** | Created from `ModuleDescriptor` during `serve()` startup (FR-SERVER-011) or dynamic registration (FR-DYNAMIC-001) | Listed by MCP clients via `tools/list` (UC-002) | Updated when module is re-registered after unregister (FR-DYNAMIC-001/002) | Removed on module unregister (FR-DYNAMIC-002) |
| **OpenAI Tool Definition** | Created by `to_openai_tools()` on each call (FR-OPENAI-001) | Returned to caller as list[dict] (UC-005) | N/A (stateless, re-created on each call) | N/A (garbage collected after caller discards) |
| **MCP Server** | Created by `serve()` (FR-SERVER-001/002/003) | N/A | N/A | Destroyed on shutdown (FR-SERVER-005, NFR-REL-001) |
| **Registry Listener** | Created during `serve()` initialization (FR-DYNAMIC-001) | N/A | N/A | Stopped on server shutdown |
| **CallToolResult** | Created per tool call (FR-EXEC-001/002 or FR-ERROR-*) | Returned to MCP client (UC-003/004) | N/A | N/A (transient per request) |
| **Health Response** | Created per /health request (FR-HEALTH-001) | Returned to HTTP client | N/A | N/A (transient per request) |
| **MCP Resource** | Created from module documentation (FR-RESOURCE-001) | Read by MCP client via `resources/read` | Updated if module documentation changes (dynamic) | Removed if module unregistered |

---

## 7. Data Dictionary

### 7.1 MCP Tool Definition

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| `name` | `str` | `ModuleDescriptor.module_id` | MCP tool name, equals the apcore module ID (dot notation preserved) |
| `description` | `str` | `ModuleDescriptor.description` | Human-readable tool description |
| `inputSchema` | `dict[str, Any]` | `ModuleDescriptor.input_schema` after $ref inlining | JSON Schema for tool input parameters |
| `outputSchema` | `dict[str, Any]` | `ModuleDescriptor.output_schema` after $ref inlining | JSON Schema for tool output (optional) |
| `annotations` | `ToolAnnotations` | `ModuleDescriptor.annotations` via AnnotationMapper | Behavioral hints: `read_only_hint`, `destructive_hint`, `idempotent_hint`, `open_world_hint` |

### 7.2 OpenAI Tool Definition

| Field | Type | Source | Description |
|-------|------|--------|-------------|
| `type` | `str` | Constant | Always `"function"` |
| `function.name` | `str` | `ModuleDescriptor.module_id` via ModuleIDNormalizer | Normalized module ID (dots replaced with `-`) |
| `function.description` | `str` | `ModuleDescriptor.description` (+ optional annotation suffix) | Tool description, optionally with annotation metadata appended |
| `function.parameters` | `dict[str, Any]` | `ModuleDescriptor.input_schema` after $ref inlining | JSON Schema for function parameters |
| `function.strict` | `bool` | Only present when `strict=True` | OpenAI strict mode flag (only included when enabled) |

### 7.3 MCP CallToolResult (Success)

| Field | Type | Value | Description |
|-------|------|-------|-------------|
| `content` | `list[TextContent]` | `[TextContent(type="text", text=<json_output>)]` | JSON-serialized module output |
| `isError` | `bool` | `False` | Indicates success |

### 7.4 MCP CallToolResult (Error)

| Field | Type | Value | Description |
|-------|------|-------|-------------|
| `content` | `list[TextContent]` | `[TextContent(type="text", text=<error_message>)]` | Human-readable error message |
| `isError` | `bool` | `True` | Indicates error |

### 7.5 Error Message Format by Error Type

| apcore Error | Message Format | Example |
|-------------|----------------|---------|
| `ModuleNotFoundError` | `"Module not found: {module_id}"` | `"Module not found: image.resize"` |
| `SchemaValidationError` | `"Input validation failed:\n- {field}: {message} ({code})"` | `"Input validation failed:\n- width: Input should be a valid integer (int_type)"` |
| `ACLDeniedError` | `"Access denied"` | `"Access denied"` |
| `ModuleTimeoutError` | `"Module timed out after {timeout_ms}ms"` | `"Module timed out after 30000ms"` |
| `InvalidInputError` | `"Invalid input: {message}"` | `"Invalid input: module_id must be non-empty"` |
| `CallDepthExceededError` | `"Call depth limit exceeded"` | `"Call depth limit exceeded"` |
| `CircularCallError` | `"Circular call detected"` | `"Circular call detected"` |
| `CallFrequencyExceededError` | `"Call frequency limit exceeded"` | `"Call frequency limit exceeded"` |
| Other `ModuleError` | `"Module error: {code}"` | `"Module error: CONFIG_INVALID"` |
| Non-`ModuleError` Exception | `"Internal error occurred"` | `"Internal error occurred"` |

### 7.6 Health Check Response

| Field | Type | Description |
|-------|------|-------------|
| `status` | `str` | Always `"ok"` |
| `module_count` | `int` | Number of currently registered tools |
| `uptime_seconds` | `float` | Seconds since server start |

### 7.7 CLI Argument Specification

| Argument | Type | Required | Default | Constraints | Description |
|----------|------|----------|---------|-------------|-------------|
| `--extensions-dir` | `str` (path) | Yes | N/A | Must exist, must be directory | Path to apcore extensions directory |
| `--transport` | `str` (choice) | No | `"stdio"` | One of: `stdio`, `streamable-http`, `sse` | MCP transport protocol |
| `--host` | `str` | No | `"127.0.0.1"` | Non-empty | Bind address for HTTP transports |
| `--port` | `int` | No | `8000` | 1-65535 | Bind port for HTTP transports |
| `--name` | `str` | No | `"apcore-mcp"` | Non-empty, max 255 chars | Server name |
| `--version` | `str` | No | Package version | Non-empty | Server version |
| `--log-level` | `str` (choice) | No | `"INFO"` | One of: `DEBUG`, `INFO`, `WARNING`, `ERROR` | Log verbosity |
| `--help` | flag | No | N/A | N/A | Show help and exit |

### 7.8 serve() Parameter Specification

| Parameter | Type | Required | Default | Constraints | Description |
|-----------|------|----------|---------|-------------|-------------|
| `registry_or_executor` | `Registry \| Executor` | Yes | N/A | Must be instance of Registry or Executor | Module source and execution engine |
| `transport` | `str` | No | `"stdio"` | One of: `"stdio"`, `"streamable-http"`, `"sse"` (case-insensitive) | Transport protocol |
| `host` | `str` | No | `"127.0.0.1"` | Non-empty | Bind address (ignored for stdio) |
| `port` | `int` | No | `8000` | 1-65535 | Bind port (ignored for stdio) |
| `name` | `str` | No | `"apcore-mcp"` | Non-empty, max 255 chars | Server name for MCP clients |
| `version` | `str \| None` | No | `None` | Non-empty if provided | Server version (None = package version) |
| `on_startup` | `Callable[[], None] \| None` | No | `None` | Must be callable or None | Optional async callback invoked after server starts |
| `on_shutdown` | `Callable[[], None] \| None` | No | `None` | Must be callable or None | Optional async callback invoked before server stops |
| `dynamic` | `bool` | No | `False` | N/A | Enable dynamic module discovery at runtime |
| `validate_inputs` | `bool` | No | `False` | N/A | Enable input schema validation |
| `tags` | `list[str] \| None` | No | `None` | Each tag non-empty | Tag filter for module selection |
| `prefix` | `str \| None` | No | `None` | Non-empty if provided | Prefix filter for module selection |
| `log_level` | `str \| None` | No | `None` | One of: `"DEBUG"`, `"INFO"`, `"WARNING"`, `"ERROR"` | Log level for apcore_mcp logger |

### 7.9 to_openai_tools() Parameter Specification

| Parameter | Type | Required | Default | Constraints | Description |
|-----------|------|----------|---------|-------------|-------------|
| `registry_or_executor` | `Registry \| Executor` | Yes | N/A | Must be instance of Registry or Executor | Module source |
| `embed_annotations` | `bool` | No | `False` | N/A | Append annotation metadata to descriptions |
| `strict` | `bool` | No | `False` | N/A | Enable OpenAI strict mode |
| `tags` | `list[str] \| None` | No | `None` | Each tag non-empty | Tag filter |
| `prefix` | `str \| None` | No | `None` | Non-empty if provided | Prefix filter |

---

## 8. Interface Requirements

### 8.1 Python API Interface

#### 8.1.1 serve() Function

**Module:** `apcore_mcp`

**Signature:**
```python
def serve(
    registry_or_executor: Registry | Executor,
    *,
    transport: str = "stdio",
    host: str = "127.0.0.1",
    port: int = 8000,
    name: str = "apcore-mcp",
    version: str | None = None,
    on_startup: Callable[[], None] | None = None,
    on_shutdown: Callable[[], None] | None = None,
    tags: list[str] | None = None,
    prefix: str | None = None,
    log_level: str | None = None,
    dynamic: bool = False,
    validate_inputs: bool = False,
) -> None: ...
```

**Behavior:** Blocks until server shutdown. Returns `None`.

**Exceptions:** `TypeError`, `ValueError`, `OSError`.

#### 8.1.2 to_openai_tools() Function

**Module:** `apcore_mcp`

**Signature:**
```python
def to_openai_tools(
    registry_or_executor: Registry | Executor,
    *,
    embed_annotations: bool = False,
    strict: bool = False,
    tags: list[str] | None = None,
    prefix: str | None = None,
) -> list[dict[str, Any]]: ...
```

**Behavior:** Pure function. No side effects. Returns list of dicts.

**Exceptions:** `TypeError`, `ValueError`.

#### 8.1.3 report_progress() Function

**Module:** `apcore_mcp`

**Signature:**
```python
async def report_progress(
    context: Any,
    progress: float,
    total: float | None = None,
    message: str | None = None,
) -> None: ...
```

**Behavior:** Sends an MCP `notifications/progress` notification. Silently no-ops when context has no MCP progress callback. No validation on progress value.

**Exceptions:** None.

#### 8.1.4 elicit() Function

**Module:** `apcore_mcp`

**Signature:**
```python
async def elicit(
    context: Any,
    message: str,
    requested_schema: dict[str, Any] | None = None,
) -> Any | None: ...
```

**Behavior:** Sends an MCP elicitation request and returns the user's response, or `None` if declined/unsupported.

**Exceptions:** None. Returns `None` when called outside MCP context (graceful no-op).

#### 8.1.5 Extension Helper Constants

**Module:** `apcore_mcp`

```python
MCP_PROGRESS_KEY: str = "_mcp_progress"
MCP_ELICIT_KEY: str = "_mcp_elicit"
```

**Behavior:** Module-level string constants. No side effects on import.

### 8.2 CLI Interface

**Entry points:**
- `python -m apcore_mcp [OPTIONS]`
- `apcore-mcp [OPTIONS]` (via `pyproject.toml` script entry point)

**Arguments:** See Section 7.7.

**Exit codes:** 0 (success), 1 (invalid arguments), 2 (startup failure / argparse error).

### 8.3 MCP Protocol Interface

#### 8.3.1 Tool Listing

**MCP Method:** `tools/list`

**Request:** No parameters.

**Response:** List of `Tool` objects, each with `name`, `description`, `inputSchema`, and `annotations`.

#### 8.3.2 Tool Calling

**MCP Method:** `tools/call`

**Request:**
- `name`: string -- tool name (= apcore module_id).
- `arguments`: object -- tool arguments dict.

**Response:** `CallToolResult` with `content` (list of `TextContent`) and `isError` (boolean).

#### 8.3.3 Tool List Changed Notification

**MCP Method:** `notifications/tools/list_changed`

**Direction:** Server -> Client.

**Trigger:** Module registered or unregistered from the Registry at runtime.

#### 8.3.4 Resource Listing (P2)

**MCP Method:** `resources/list`

**Response:** List of resources with URI pattern `docs://{module_id}`.

#### 8.3.5 Resource Reading (P2)

**MCP Method:** `resources/read`

**Request:** Resource URI.

**Response:** Documentation text content.

### 8.4 apcore SDK Interface (Consumed APIs)

#### 8.4.1 Registry API

| Method / Property | Used By | Purpose |
|-------------------|---------|---------|
| `Registry(extensions_dir=...)` | CLI module | Create registry for module discovery |
| `registry.discover()` | CLI module | Discover and register modules from directory |
| `registry.list(tags=None, prefix=None)` | MCPServerFactory, OpenAIConverter | Get list of module IDs with optional filtering |
| `registry.get_definition(module_id)` | MCPServerFactory, OpenAIConverter, RegistryListener | Get ModuleDescriptor for schema and annotation data |
| `registry.on("register", callback)` | RegistryListener | Subscribe to module registration events |
| `registry.on("unregister", callback)` | RegistryListener | Subscribe to module unregistration events |
| `registry.count` | Logging, health check | Get number of registered modules |

#### 8.4.2 Executor API

| Method / Property | Used By | Purpose |
|-------------------|---------|---------|
| `Executor(registry, ...)` | serve() (when Registry passed) | Create default executor |
| `executor.call_async(module_id, inputs, context=None)` | ExecutionRouter | Execute tool calls through full pipeline |
| `executor.registry` | serve(), to_openai_tools() (when Executor passed) | Extract registry for tool discovery |

#### 8.4.3 ModuleDescriptor Fields

| Field | Type | Used For |
|-------|------|----------|
| `module_id` | `str` | MCP tool name, OpenAI function name (normalized) |
| `description` | `str` | MCP tool description, OpenAI function description |
| `input_schema` | `dict[str, Any]` | MCP inputSchema, OpenAI parameters |
| `output_schema` | `dict[str, Any]` | MCP structured output |
| `annotations` | `ModuleAnnotations \| None` | MCP tool annotations, OpenAI annotation suffix |
| `documentation` | `str \| None` | MCP Resource content |
| `tags` | `list[str]` | Module filtering |

#### 8.4.4 Error Hierarchy

| Error Class | Error Code | Mapped To |
|-------------|-----------|-----------|
| `ModuleError` (base) | varies | Generic module error |
| `ModuleNotFoundError` | `MODULE_NOT_FOUND` | "Module not found: {id}" |
| `SchemaValidationError` | `SCHEMA_VALIDATION_ERROR` | Field-level error details |
| `ACLDeniedError` | `ACL_DENIED` | "Access denied" |
| `ModuleTimeoutError` | `MODULE_TIMEOUT` | "Module timed out after {ms}ms" |
| `InvalidInputError` | `GENERAL_INVALID_INPUT` | "Invalid input: {message}" |
| `CallDepthExceededError` | `CALL_DEPTH_EXCEEDED` | "Call depth limit exceeded" |
| `CircularCallError` | `CIRCULAR_CALL` | "Circular call detected" |
| `CallFrequencyExceededError` | `CALL_FREQUENCY_EXCEEDED` | "Call frequency limit exceeded" |

---

## 9. Traceability Matrix

### 9.1 PRD Feature to FR Traceability

| PRD Feature | Description | FRs | NFRs | Use Cases |
|-------------|-------------|-----|------|-----------|
| F-001 | Registry-to-MCP Schema Mapping | FR-SCHEMA-001, FR-SCHEMA-002, FR-SCHEMA-003, FR-SCHEMA-006, FR-SERVER-011 | NFR-PERF-001, NFR-PERF-003 | UC-001, UC-002 |
| F-002 | Annotation-to-MCP Mapping | FR-ANNOT-001, FR-ANNOT-002, FR-ANNOT-003, FR-ANNOT-004, FR-ANNOT-005, FR-ANNOT-006 | -- | UC-001, UC-002 |
| F-003 | MCP Execution Routing | FR-EXEC-001, FR-EXEC-002, FR-EXEC-003, FR-EXEC-004, FR-EXEC-007 | NFR-PERF-002, NFR-SEC-001 | UC-003, UC-004 |
| F-004 | MCP Error Mapping | FR-ERROR-001 through FR-ERROR-011 | NFR-SEC-002 | UC-004 |
| F-005 | serve() Function | FR-SERVER-001 through FR-SERVER-011 | NFR-REL-001, NFR-REL-002 | UC-001 |
| F-006 | stdio Transport | FR-TRANSPORT-001, FR-TRANSPORT-002, FR-TRANSPORT-005 | -- | UC-001, UC-006 |
| F-007 | Streamable HTTP Transport | FR-TRANSPORT-003, FR-SERVER-002 | NFR-PERF-004 | UC-001 |
| F-008 | to_openai_tools() Function | FR-OPENAI-001, FR-OPENAI-002, FR-OPENAI-005, FR-SCHEMA-004 | -- | UC-005 |
| F-009 | CLI Entry Point | FR-CLI-001 through FR-CLI-008 | -- | UC-006 |
| F-010 | SSE Transport (Backward Compat) | FR-SERVER-003, FR-TRANSPORT-004 | -- | UC-001 |
| F-011 | OpenAI Annotation Embedding | FR-ANNOT-007, FR-OPENAI-004 | -- | UC-005 |
| F-012 | OpenAI Strict Mode | FR-OPENAI-003 | -- | UC-008 |
| F-013 | Structured Output Responses | FR-SCHEMA-005, FR-EXEC-002 | -- | UC-003 |
| F-014 | Executor Passthrough | FR-EXEC-005, FR-EXEC-006 | NFR-SEC-001 | UC-001, UC-005 |
| F-015 | Dynamic Tool Registration | FR-DYNAMIC-001 through FR-DYNAMIC-004 | NFR-REL-003 | UC-007 |
| F-016 | Logging and Observability | FR-LOG-001 through FR-LOG-005, FR-SERVER-010 | NFR-MAINT-004 | UC-001 |
| F-017 | to_openai_tools() Filtering | FR-OPENAI-006, FR-OPENAI-007, FR-FILTER-003 | -- | UC-005 |
| F-018 | serve() Module Filtering | FR-SERVER-008, FR-SERVER-009, FR-FILTER-001, FR-FILTER-002, FR-FILTER-003 | -- | UC-001 |
| F-019 | Health Check Endpoint | FR-HEALTH-001, FR-HEALTH-002 | -- | -- |
| F-020 | MCP Resource Exposure | FR-RESOURCE-001, FR-RESOURCE-002 | -- | -- |
| F-026 | MCP Tool Explorer | FR-EXPLORER-001 through FR-EXPLORER-006 | -- | -- |

### 9.2 FR to PRD Feature Reverse Traceability

| FR ID | PRD Feature(s) |
|-------|----------------|
| FR-SCHEMA-001 | F-001 |
| FR-SCHEMA-002 | F-001 |
| FR-SCHEMA-003 | F-001 |
| FR-SCHEMA-004 | F-008 |
| FR-SCHEMA-005 | F-013 |
| FR-SCHEMA-006 | F-001 |
| FR-ANNOT-001 | F-002 |
| FR-ANNOT-002 | F-002 |
| FR-ANNOT-003 | F-002 |
| FR-ANNOT-004 | F-002 |
| FR-ANNOT-005 | F-002 |
| FR-ANNOT-006 | F-002 |
| FR-ANNOT-007 | F-011 |
| FR-EXEC-001 | F-003 |
| FR-EXEC-002 | F-003, F-013 |
| FR-EXEC-003 | F-003 |
| FR-EXEC-004 | F-003 |
| FR-EXEC-005 | F-014 |
| FR-EXEC-006 | F-014 |
| FR-EXEC-007 | F-003, F-004 |
| FR-ERROR-001 | F-004 |
| FR-ERROR-002 | F-004 |
| FR-ERROR-003 | F-004 |
| FR-ERROR-004 | F-004 |
| FR-ERROR-005 | F-004 |
| FR-ERROR-006 | F-004 |
| FR-ERROR-007 | F-004 |
| FR-ERROR-008 | F-004 |
| FR-ERROR-009 | F-004 |
| FR-ERROR-010 | F-004 |
| FR-ERROR-011 | F-004 |
| FR-SERVER-001 | F-005, F-006 |
| FR-SERVER-002 | F-005, F-007 |
| FR-SERVER-003 | F-005, F-010 |
| FR-SERVER-004 | F-005 |
| FR-SERVER-005 | F-005 |
| FR-SERVER-006 | F-005 |
| FR-SERVER-007 | F-005 |
| FR-SERVER-008 | F-018 |
| FR-SERVER-009 | F-018 |
| FR-SERVER-010 | F-016 |
| FR-SERVER-011 | F-001, F-005 |
| FR-TRANSPORT-001 | F-006 |
| FR-TRANSPORT-002 | F-006 |
| FR-TRANSPORT-003 | F-007 |
| FR-TRANSPORT-004 | F-010 |
| FR-TRANSPORT-005 | F-006 |
| FR-OPENAI-001 | F-008 |
| FR-OPENAI-002 | F-008 |
| FR-OPENAI-003 | F-012 |
| FR-OPENAI-004 | F-011 |
| FR-OPENAI-005 | F-008 |
| FR-OPENAI-006 | F-017 |
| FR-OPENAI-007 | F-017 |
| FR-CLI-001 | F-009 |
| FR-CLI-002 | F-009 |
| FR-CLI-003 | F-009 |
| FR-CLI-004 | F-009 |
| FR-CLI-005 | F-009 |
| FR-CLI-006 | F-009, F-016 |
| FR-CLI-007 | F-009 |
| FR-CLI-008 | F-009 |
| FR-DYNAMIC-001 | F-015 |
| FR-DYNAMIC-002 | F-015 |
| FR-DYNAMIC-003 | F-015 |
| FR-DYNAMIC-004 | F-015 |
| FR-LOG-001 | F-016 |
| FR-LOG-002 | F-016 |
| FR-LOG-003 | F-016 |
| FR-LOG-004 | F-016 |
| FR-LOG-005 | F-016 |
| FR-FILTER-001 | F-018 |
| FR-FILTER-002 | F-018 |
| FR-FILTER-003 | F-017, F-018 |
| FR-HEALTH-001 | F-019 |
| FR-HEALTH-002 | F-019 |
| FR-RESOURCE-001 | F-020 |
| FR-RESOURCE-002 | F-020 |
| FR-EXPLORER-001 | F-026 |
| FR-EXPLORER-002 | F-026 |
| FR-EXPLORER-003 | F-026 |
| FR-EXPLORER-004 | F-026 |
| FR-EXPLORER-005 | F-026 |
| FR-EXPLORER-006 | F-026 |

---

## 10. Appendix

### 10.1 Glossary

See Section 1.3 for complete definitions of all terms, acronyms, and abbreviations used in this document.

### 10.2 References

See Section 1.4 for all referenced documents and specifications.

### 10.3 apcore API Surface Used by apcore-mcp

The following apcore-python APIs are consumed by apcore-mcp. Verified against source code at `/Users/tercel/WorkSpace/aipartnerup/apcore-python/src/apcore/`.

```python
# Registry (apcore.registry.registry.Registry)
registry = Registry(extensions_dir="./extensions")
registry.discover() -> int
registry.list(tags=None, prefix=None) -> list[str]
registry.get(module_id) -> module | None
registry.get_definition(module_id) -> ModuleDescriptor | None
registry.iter() -> Iterator[tuple[str, Any]]
registry.on("register", callback)  # callback(module_id: str, module: Any)
registry.on("unregister", callback)
registry.count -> int
registry.module_ids -> list[str]

# Executor (apcore.executor.Executor)
executor = Executor(registry, middlewares=None, acl=None, config=None)
executor.call(module_id, inputs, context=None) -> dict
executor.call_async(module_id, inputs, context=None) -> dict  # async
executor.registry -> Registry

# ModuleDescriptor (apcore.registry.types.ModuleDescriptor)
@dataclass
class ModuleDescriptor:
    module_id: str
    name: str | None
    description: str
    documentation: str | None
    input_schema: dict[str, Any]       # JSON Schema from Pydantic model_json_schema()
    output_schema: dict[str, Any]      # JSON Schema
    version: str = "1.0.0"
    tags: list[str] = field(default_factory=list)
    annotations: ModuleAnnotations | None = None
    examples: list[ModuleExample] = field(default_factory=list)
    metadata: dict[str, Any] = field(default_factory=dict)

# ModuleAnnotations (apcore.module.ModuleAnnotations)
@dataclass(frozen=True)
class ModuleAnnotations:
    readonly: bool = False
    destructive: bool = False
    idempotent: bool = False
    requires_approval: bool = False
    open_world: bool = True

# Error hierarchy (apcore.errors)
# Base: ModuleError(code, message, details, cause, trace_id)
ModuleNotFoundError(module_id: str)                         # code="MODULE_NOT_FOUND"
SchemaValidationError(message, errors: list[dict])          # code="SCHEMA_VALIDATION_ERROR"
ACLDeniedError(caller_id: str | None, target_id: str)      # code="ACL_DENIED"
ModuleTimeoutError(module_id: str, timeout_ms: int)         # code="MODULE_TIMEOUT"
InvalidInputError(message: str)                             # code="GENERAL_INVALID_INPUT"
CallDepthExceededError(depth, max_depth, call_chain)        # code="CALL_DEPTH_EXCEEDED"
CircularCallError(module_id, call_chain)                    # code="CIRCULAR_CALL"
CallFrequencyExceededError(module_id, count, max_repeat, call_chain)  # code="CALL_FREQUENCY_EXCEEDED"
```

### 10.4 Requirement Counts Summary

| Category | Count |
|----------|-------|
| FR-SCHEMA | 6 |
| FR-ANNOT | 7 |
| FR-EXEC | 7 |
| FR-ERROR | 11 |
| FR-SERVER | 11 |
| FR-TRANSPORT | 5 |
| FR-OPENAI | 7 |
| FR-CLI | 8 |
| FR-DYNAMIC | 4 |
| FR-LOG | 5 |
| FR-FILTER | 3 |
| FR-HEALTH | 2 |
| FR-RESOURCE | 2 |
| FR-EXPLORER | 6 |
| **Total FRs** | **84** |
| NFR-PERF | 4 |
| NFR-SEC | 3 |
| NFR-REL | 3 |
| NFR-MAINT | 4 |
| NFR-COMPAT | 4 |
| NFR-PORT | 2 |
| **Total NFRs** | **20** |
| **Grand Total** | **104** |

---

*End of Software Requirements Specification*
