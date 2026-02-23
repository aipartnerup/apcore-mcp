# Test Plan & Test Cases: apcore-mcp

| Field       | Value                                                        |
|-------------|--------------------------------------------------------------|
| Title       | apcore-mcp Test Plan & Test Cases                            |
| Version     | 1.0                                                          |
| Date        | 2026-02-15                                                   |
| Author      | aipartnerup QA Team                                          |
| Status      | Draft                                                        |
| PRD Ref     | `docs/prd-apcore-mcp.md` v1.0                               |
| Tech Design | `docs/tech-design-apcore-mcp.md` v1.0                       |
| License     | Apache 2.0                                                   |

---

## 1. Test Plan Overview

### 1.1 Purpose and Scope

This document defines the comprehensive test plan and test cases for **apcore-mcp**, the automatic MCP Server and OpenAI Tools Bridge for the apcore ecosystem. As a greenfield project developed under TDD strict mode, this test plan establishes the testing standard before any implementation code is written. Every test case defined here will be implemented as executable pytest code prior to the corresponding production code.

The scope covers all 20 PRD features (F-001 through F-020) across 9 architectural components: Schema Converter, Annotation Mapper, Execution Router, Error Mapper, MCP Server Factory, OpenAI Converter, Transport Manager, CLI Module, and Dynamic Registry Listener. Testing spans five levels: unit, integration, end-to-end, performance, and security.

### 1.2 Test Objectives

1. Validate 100% schema mapping accuracy between apcore ModuleDescriptor fields and MCP/OpenAI tool definitions.
2. Verify all 8 apcore error types plus unexpected exceptions are correctly mapped to MCP error responses.
3. Confirm all 3 transport types (stdio, Streamable HTTP, SSE) function correctly.
4. Ensure annotation preservation rate of 100% (all 5 apcore annotation fields mapped).
5. Validate performance targets: <100ms for 100-module registration, <5ms tool call overhead, <10MB memory for 100 modules.
6. Confirm security guarantees: no sensitive data leakage, ACL enforcement, error sanitization.

### 1.3 Quality Goals

| Metric                | Target   |
|-----------------------|----------|
| Line coverage         | >= 90%   |
| P0 feature test pass  | 100%     |
| P1 feature test pass  | >= 95%   |
| P2 feature test pass  | >= 90%   |
| Unit test pass rate   | 100%     |
| Integration test pass | 100%     |
| Performance benchmarks| All pass |

---

## 2. Test Strategy

### 2.1 Test Levels and Pyramid Distribution

| Level       | Target % | Approx Count | Description                                      |
|-------------|----------|---------------|--------------------------------------------------|
| Unit        | 60%      | ~95           | Individual component behavior in isolation        |
| Integration | 25%      | ~20           | Multi-component workflows, end-to-end data flows  |
| E2E         | 10%      | ~8            | Full server lifecycle with real MCP client         |
| Performance | 3%       | ~7            | Benchmarks, stress tests, memory profiling         |
| Security    | 2%       | ~6            | ACL enforcement, error sanitization, input fuzzing |

### 2.2 Test Frameworks and Tools

| Tool              | Purpose                                      |
|-------------------|----------------------------------------------|
| pytest >= 7.0     | Test runner, fixtures, parametrize            |
| pytest-asyncio    | Async test support (asyncio_mode = "auto")   |
| pytest-cov >= 4.0 | Line coverage measurement and enforcement    |
| pytest-benchmark  | Performance benchmarking (tool call overhead) |
| unittest.mock     | MagicMock, AsyncMock for apcore/MCP SDK mocks |
| tracemalloc       | Memory profiling for performance tests        |

### 2.3 Mock Strategy

| Component Under Test | What to Mock                     | What to Use Real             |
|----------------------|----------------------------------|------------------------------|
| SchemaConverter      | Nothing                          | Real: pure dict transform    |
| AnnotationMapper     | Nothing                          | Real: pure function          |
| ErrorMapper          | Nothing                          | Real: pure function          |
| ModuleIDNormalizer   | Nothing                          | Real: pure function          |
| ExecutionRouter      | Mock: Executor.call_async()      | Real: ErrorMapper            |
| MCPServerFactory     | Mock: mcp.server.lowlevel.Server | Real: SchemaConverter, AnnotationMapper |
| OpenAIConverter      | Mock: Registry                   | Real: SchemaConverter, AnnotationMapper, IDNormalizer |
| TransportManager     | Mock: Server, stdio_server       | Real: validation logic       |
| CLI Module           | Mock: Registry, serve()          | Real: argparse               |
| RegistryListener     | Mock: Registry.on(), Factory     | Real: internal tools dict    |
| Integration tests    | Mock: module execute() only      | Real: all apcore-mcp components |
| E2E tests            | Nothing                          | Real: full stack             |

### 2.4 Fixtures Strategy

Shared fixtures defined in `tests/conftest.py`:

- `sample_annotations`: ModuleAnnotations with non-default values
- `sample_descriptor`: ModuleDescriptor for "image.resize" with full schema
- `descriptor_with_refs`: ModuleDescriptor with $defs/$ref in input_schema
- `descriptor_empty_schema`: ModuleDescriptor with empty input_schema
- `descriptor_no_annotations`: ModuleDescriptor with annotations=None
- `mock_registry`: MagicMock(spec=Registry) returning sample descriptors
- `mock_executor`: MagicMock(spec=Executor) with AsyncMock call_async
- `multi_module_registry`: Registry mock with 5 diverse modules
- `large_registry`: Registry mock with 100 modules for performance tests

---

## 3. Test Environment

### 3.1 Python Version Requirements

| Version | CI Matrix | Notes                          |
|---------|-----------|--------------------------------|
| 3.10    | Yes       | Minimum supported              |
| 3.11    | Yes       | Secondary                      |
| 3.12    | Yes       | apcore-python dev version      |
| 3.13    | Yes       | Latest stable                  |

### 3.2 Dependency Versions

| Package        | Version Constraint   |
|----------------|---------------------|
| apcore         | >= 0.2.0, < 1.0     |
| mcp            | >= 1.0.0, < 2.0     |
| pytest         | >= 7.0              |
| pytest-asyncio | >= 0.21             |
| pytest-cov     | >= 4.0              |
| pytest-benchmark | >= 4.0             |

### 3.3 Test Configuration (pyproject.toml)

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
markers = [
    "unit: Unit tests",
    "integration: Integration tests",
    "e2e: End-to-end tests",
    "performance: Performance benchmark tests",
    "security: Security tests",
    "slow: Tests that take > 5 seconds",
]

[tool.coverage.run]
source = ["src/apcore_mcp"]
omit = ["src/apcore_mcp/__main__.py"]

[tool.coverage.report]
fail_under = 90
show_missing = true
```

### 3.4 CI/CD Integration Plan

| Stage          | Trigger             | Tests Run                        |
|----------------|---------------------|----------------------------------|
| Pre-commit     | git commit          | Unit tests (fast)                |
| PR validation  | Pull request        | Unit + Integration + Security    |
| Nightly        | Cron (daily)        | All tests including E2E + Perf   |
| Release gate   | Tag push (v*)       | Full suite, coverage enforcement |

---

## 4. Feature-to-Test Traceability Matrix

| PRD Feature | Description                        | Priority | Test Case IDs                                                    |
|-------------|------------------------------------|----------|------------------------------------------------------------------|
| F-001       | Registry-to-MCP Schema Mapping     | P0       | TC-SCHEMA-001 to TC-SCHEMA-012, TC-INT-001                      |
| F-002       | Annotation-to-MCP Mapping          | P0       | TC-ANNOT-001 to TC-ANNOT-010, TC-INT-001                        |
| F-003       | MCP Execution Routing              | P0       | TC-EXEC-001 to TC-EXEC-012, TC-INT-001, TC-INT-003              |
| F-004       | MCP Error Mapping                  | P0       | TC-ERROR-001 to TC-ERROR-011, TC-INT-003                        |
| F-005       | serve() Function                   | P0       | TC-SERVER-001 to TC-SERVER-010, TC-INT-001, TC-E2E-001          |
| F-006       | stdio Transport                    | P0       | TC-TRANSPORT-001 to TC-TRANSPORT-003, TC-INT-002, TC-E2E-001   |
| F-007       | Streamable HTTP Transport          | P0       | TC-TRANSPORT-004 to TC-TRANSPORT-006, TC-INT-002, TC-E2E-002   |
| F-008       | to_openai_tools() Function         | P0       | TC-OPENAI-001 to TC-OPENAI-012, TC-INT-004                     |
| F-009       | CLI Entry Point                    | P0       | TC-CLI-001 to TC-CLI-010, TC-E2E-001                            |
| F-010       | SSE Transport                      | P1       | TC-TRANSPORT-007 to TC-TRANSPORT-009, TC-INT-002                |
| F-011       | OpenAI Annotation Embedding        | P1       | TC-OPENAI-005, TC-OPENAI-006                                    |
| F-012       | OpenAI Strict Mode                 | P1       | TC-OPENAI-007, TC-OPENAI-008, TC-OPENAI-009                    |
| F-013       | Structured Output Responses        | P1       | TC-EXEC-001, TC-EXEC-008                                        |
| F-014       | Executor Passthrough               | P1       | TC-SERVER-003, TC-INT-005                                        |
| F-015       | Dynamic Tool Registration          | P1       | TC-DYNAMIC-001 to TC-DYNAMIC-007, TC-INT-006, TC-E2E-003       |
| F-016       | Logging and Observability          | P1       | TC-SERVER-009, TC-EXEC-006                                       |
| F-017       | to_openai_tools() Filtering        | P2       | TC-OPENAI-010, TC-OPENAI-011                                    |
| F-018       | serve() Module Filtering           | P2       | TC-SERVER-007, TC-SERVER-008                                     |
| F-019       | Health Check Endpoint              | P2       | TC-E2E-004                                                       |
| F-020       | MCP Resource Exposure              | P2       | TC-E2E-005                                                       |

---

## 5. Test Cases by Component

### 5.1 Schema Converter (TC-SCHEMA-xxx)

**Test File:** `tests/unit/adapters/test_schema.py`

---

#### TC-SCHEMA-001: Convert simple schema without $ref

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** SchemaConverter instance created
- **Test Data:**
  ```python
  input_schema = {
      "type": "object",
      "title": "ImageResizeInput",
      "properties": {
          "width": {"type": "integer", "description": "Target width in pixels"},
          "height": {"type": "integer", "description": "Target height in pixels"},
          "format": {"type": "string", "default": "png", "enum": ["png", "jpg", "webp"]}
      },
      "required": ["width", "height"]
  }
  descriptor = ModuleDescriptor(
      module_id="image.resize",
      description="Resize an image",
      input_schema=input_schema,
      output_schema={}
  )
  ```
- **Steps:**
  1. Create SchemaConverter instance.
  2. Call `converter.convert_input_schema(descriptor)`.
  3. Assert the returned dict equals the input_schema exactly (no transformation needed).
- **Expected Result:** Returned dict is identical to the input `input_schema` dict. All properties, types, required fields, and enums are preserved.
- **Traceability:** F-001

---

#### TC-SCHEMA-002: Convert schema with single-level $ref inlining

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** SchemaConverter instance created
- **Test Data:**
  ```python
  input_schema = {
      "type": "object",
      "properties": {
          "workflow_name": {"type": "string"},
          "parameters": {"$ref": "#/$defs/WorkflowParams"}
      },
      "required": ["workflow_name", "parameters"],
      "$defs": {
          "WorkflowParams": {
              "type": "object",
              "properties": {
                  "seed": {"type": "integer", "default": 42},
                  "steps": {"type": "integer", "default": 20}
              }
          }
      }
  }
  ```
- **Steps:**
  1. Create SchemaConverter instance.
  2. Call `converter.convert_input_schema(descriptor)`.
  3. Assert `$defs` key is not present in result.
  4. Assert `parameters` property contains the inlined object definition.
- **Expected Result:**
  ```python
  {
      "type": "object",
      "properties": {
          "workflow_name": {"type": "string"},
          "parameters": {
              "type": "object",
              "properties": {
                  "seed": {"type": "integer", "default": 42},
                  "steps": {"type": "integer", "default": 20}
              }
          }
      },
      "required": ["workflow_name", "parameters"]
  }
  ```
- **Traceability:** F-001

---

#### TC-SCHEMA-003: Convert schema with nested $ref (A references B)

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** SchemaConverter instance created
- **Test Data:**
  ```python
  input_schema = {
      "type": "object",
      "properties": {
          "config": {"$ref": "#/$defs/Config"}
      },
      "required": ["config"],
      "$defs": {
          "Config": {
              "type": "object",
              "properties": {
                  "output": {"$ref": "#/$defs/OutputSettings"}
              }
          },
          "OutputSettings": {
              "type": "object",
              "properties": {
                  "format": {"type": "string"},
                  "quality": {"type": "integer"}
              }
          }
      }
  }
  ```
- **Steps:**
  1. Call `converter.convert_input_schema(descriptor)`.
  2. Assert `$defs` is removed.
  3. Assert nested `OutputSettings` is inlined inside `Config.properties.output`.
- **Expected Result:** Both `$ref` nodes are resolved. `config.properties.output` contains `{"type": "object", "properties": {"format": {"type": "string"}, "quality": {"type": "integer"}}}`. No `$defs` key in result.
- **Traceability:** F-001

---

#### TC-SCHEMA-004: Detect circular $ref and raise ValueError

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** SchemaConverter instance created
- **Test Data:**
  ```python
  input_schema = {
      "type": "object",
      "properties": {
          "node": {"$ref": "#/$defs/TreeNode"}
      },
      "$defs": {
          "TreeNode": {
              "type": "object",
              "properties": {
                  "value": {"type": "string"},
                  "child": {"$ref": "#/$defs/TreeNode"}
              }
          }
      }
  }
  ```
- **Steps:**
  1. Call `converter.convert_input_schema(descriptor)`.
  2. Assert `ValueError` is raised.
- **Expected Result:** `ValueError` is raised with a message indicating circular `$ref` detected.
- **Traceability:** F-001

---

#### TC-SCHEMA-005: Convert empty input_schema to valid object schema

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** SchemaConverter instance created
- **Test Data:**
  ```python
  descriptor = ModuleDescriptor(
      module_id="system.ping",
      description="Health check",
      input_schema={},
      output_schema={}
  )
  ```
- **Steps:**
  1. Call `converter.convert_input_schema(descriptor)`.
  2. Assert result has `"type": "object"` and `"properties": {}`.
- **Expected Result:** `{"type": "object", "properties": {}}`.
- **Traceability:** F-001 (AC5)

---

#### TC-SCHEMA-006: Strip $defs when no $ref references exist

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** SchemaConverter instance created
- **Test Data:**
  ```python
  input_schema = {
      "type": "object",
      "properties": {
          "name": {"type": "string"}
      },
      "$defs": {
          "UnusedModel": {
              "type": "object",
              "properties": {"x": {"type": "integer"}}
          }
      }
  }
  ```
- **Steps:**
  1. Call `converter.convert_input_schema(descriptor)`.
  2. Assert `$defs` key is not present in result.
  3. Assert `properties.name` is preserved.
- **Expected Result:** `{"type": "object", "properties": {"name": {"type": "string"}}}`.
- **Traceability:** F-001

---

#### TC-SCHEMA-007: Convert schema with array items containing $ref

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** SchemaConverter instance created
- **Test Data:**
  ```python
  input_schema = {
      "type": "object",
      "properties": {
          "tags": {
              "type": "array",
              "items": {"$ref": "#/$defs/Tag"}
          }
      },
      "$defs": {
          "Tag": {
              "type": "object",
              "properties": {
                  "key": {"type": "string"},
                  "value": {"type": "string"}
              }
          }
      }
  }
  ```
- **Steps:**
  1. Call `converter.convert_input_schema(descriptor)`.
  2. Assert `tags.items` contains the inlined Tag object.
- **Expected Result:** `tags.items` equals `{"type": "object", "properties": {"key": {"type": "string"}, "value": {"type": "string"}}}`. No `$defs`.
- **Traceability:** F-001

---

#### TC-SCHEMA-008: Convert schema with oneOf containing $ref

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** SchemaConverter instance created
- **Test Data:**
  ```python
  input_schema = {
      "type": "object",
      "properties": {
          "source": {
              "oneOf": [
                  {"$ref": "#/$defs/FileSource"},
                  {"$ref": "#/$defs/URLSource"}
              ]
          }
      },
      "$defs": {
          "FileSource": {"type": "object", "properties": {"path": {"type": "string"}}},
          "URLSource": {"type": "object", "properties": {"url": {"type": "string", "format": "uri"}}}
      }
  }
  ```
- **Steps:**
  1. Call `converter.convert_input_schema(descriptor)`.
  2. Assert `source.oneOf` contains two inlined objects.
- **Expected Result:** `source.oneOf[0]` equals `{"type": "object", "properties": {"path": {"type": "string"}}}`. `source.oneOf[1]` equals `{"type": "object", "properties": {"url": {"type": "string", "format": "uri"}}}`. No `$defs`.
- **Traceability:** F-001

---

#### TC-SCHEMA-009: Ensure root type is object when missing

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** SchemaConverter instance created
- **Test Data:**
  ```python
  input_schema = {
      "properties": {
          "name": {"type": "string"}
      },
      "required": ["name"]
  }
  ```
- **Steps:**
  1. Call `converter.convert_input_schema(descriptor)`.
  2. Assert result has `"type": "object"`.
- **Expected Result:** Result includes `"type": "object"` alongside existing `properties` and `required`.
- **Traceability:** F-001

---

#### TC-SCHEMA-010: Handle maximum nesting depth (32 levels)

- **Priority:** P2
- **Type:** Unit
- **Preconditions:** SchemaConverter instance created
- **Test Data:** Programmatically generated schema with 33 levels of nested $ref:
  ```python
  defs = {}
  for i in range(33):
      name = f"Level{i}"
      next_name = f"Level{i+1}"
      if i < 32:
          defs[name] = {"type": "object", "properties": {"child": {"$ref": f"#/$defs/{next_name}"}}}
      else:
          defs[name] = {"type": "object", "properties": {"value": {"type": "string"}}}
  input_schema = {
      "type": "object",
      "properties": {"root": {"$ref": "#/$defs/Level0"}},
      "$defs": defs
  }
  ```
- **Steps:**
  1. Call `converter.convert_input_schema(descriptor)`.
  2. Assert `ValueError` is raised due to depth exceeding 32.
- **Expected Result:** `ValueError` raised indicating maximum recursion depth exceeded.
- **Traceability:** F-001

---

#### TC-SCHEMA-011: Preserve Unicode in property descriptions

- **Priority:** P2
- **Type:** Unit
- **Preconditions:** SchemaConverter instance created
- **Test Data:**
  ```python
  input_schema = {
      "type": "object",
      "properties": {
          "name": {"type": "string", "description": "username"},
          "greeting": {"type": "string", "description": "Bonjour, comment allez-vous?"}
      }
  }
  ```
- **Steps:**
  1. Call `converter.convert_input_schema(descriptor)`.
  2. Assert Unicode descriptions are preserved exactly.
- **Expected Result:** `properties.name.description` equals `"username"`. `properties.greeting.description` equals `"Bonjour, comment allez-vous?"`.
- **Traceability:** F-001

---

#### TC-SCHEMA-012: Handle very large schema with 50+ properties

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** SchemaConverter instance created
- **Test Data:** Programmatically generated schema:
  ```python
  properties = {f"field_{i}": {"type": "string", "description": f"Field number {i}"} for i in range(50)}
  input_schema = {"type": "object", "properties": properties, "required": list(properties.keys())}
  ```
- **Steps:**
  1. Call `converter.convert_input_schema(descriptor)`.
  2. Assert result has exactly 50 properties.
  3. Assert all 50 fields are in `required`.
- **Expected Result:** All 50 properties preserved with correct types and descriptions. All 50 field names present in `required` array.
- **Traceability:** F-001

---

### 5.2 Annotation Mapper (TC-ANNOT-xxx)

**Test File:** `tests/unit/adapters/test_annotations.py`

---

#### TC-ANNOT-001: Map readonly=True to readOnlyHint=True

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** AnnotationMapper instance created
- **Test Data:**
  ```python
  annotations = ModuleAnnotations(readonly=True, destructive=False, idempotent=False, requires_approval=False, open_world=True)
  ```
- **Steps:**
  1. Call `mapper.to_mcp_annotations(annotations)`.
  2. Assert `result.read_only_hint` is `True`.
- **Expected Result:** `ToolAnnotations(read_only_hint=True, destructive_hint=False, idempotent_hint=False, open_world_hint=True)`.
- **Traceability:** F-002 (AC1)

---

#### TC-ANNOT-002: Map destructive=True to destructiveHint=True

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** AnnotationMapper instance created
- **Test Data:**
  ```python
  annotations = ModuleAnnotations(readonly=False, destructive=True, idempotent=False, requires_approval=False, open_world=True)
  ```
- **Steps:**
  1. Call `mapper.to_mcp_annotations(annotations)`.
  2. Assert `result.destructive_hint` is `True`.
- **Expected Result:** `destructive_hint=True`, all others at their respective values.
- **Traceability:** F-002 (AC2)

---

#### TC-ANNOT-003: Map idempotent=True to idempotentHint=True

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** AnnotationMapper instance created
- **Test Data:**
  ```python
  annotations = ModuleAnnotations(readonly=False, destructive=False, idempotent=True, requires_approval=False, open_world=True)
  ```
- **Steps:**
  1. Call `mapper.to_mcp_annotations(annotations)`.
  2. Assert `result.idempotent_hint` is `True`.
- **Expected Result:** `idempotent_hint=True`.
- **Traceability:** F-002 (AC3)

---

#### TC-ANNOT-004: Map open_world=False to openWorldHint=False

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** AnnotationMapper instance created
- **Test Data:**
  ```python
  annotations = ModuleAnnotations(readonly=False, destructive=False, idempotent=False, requires_approval=False, open_world=False)
  ```
- **Steps:**
  1. Call `mapper.to_mcp_annotations(annotations)`.
  2. Assert `result.open_world_hint` is `False`.
- **Expected Result:** `open_world_hint=False`.
- **Traceability:** F-002 (AC4)

---

#### TC-ANNOT-005: Map None annotations to MCP defaults

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** AnnotationMapper instance created
- **Test Data:** `annotations = None`
- **Steps:**
  1. Call `mapper.to_mcp_annotations(None)`.
  2. Assert result uses MCP default values.
- **Expected Result:** `ToolAnnotations(read_only_hint=False, destructive_hint=False, idempotent_hint=False, open_world_hint=True)`.
- **Traceability:** F-002 (AC5)

---

#### TC-ANNOT-006: Map all annotations set simultaneously

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** AnnotationMapper instance created
- **Test Data:**
  ```python
  annotations = ModuleAnnotations(readonly=True, destructive=True, idempotent=True, requires_approval=True, open_world=False)
  ```
- **Steps:**
  1. Call `mapper.to_mcp_annotations(annotations)`.
  2. Assert all four MCP hint fields match.
- **Expected Result:** `ToolAnnotations(read_only_hint=True, destructive_hint=True, idempotent_hint=True, open_world_hint=False)`.
- **Traceability:** F-002

---

#### TC-ANNOT-007: requires_approval flag is preserved

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** AnnotationMapper instance created
- **Test Data:**
  ```python
  annotations = ModuleAnnotations(readonly=False, destructive=False, idempotent=False, requires_approval=True, open_world=True)
  ```
- **Steps:**
  1. Call `mapper.has_requires_approval(annotations)`.
  2. Assert returns `True`.
- **Expected Result:** `True`.
- **Traceability:** F-002 (AC6)

---

#### TC-ANNOT-008: requires_approval returns False for None annotations

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** AnnotationMapper instance created
- **Test Data:** `annotations = None`
- **Steps:**
  1. Call `mapper.has_requires_approval(None)`.
  2. Assert returns `False`.
- **Expected Result:** `False`.
- **Traceability:** F-002 (AC6)

---

#### TC-ANNOT-009: Description suffix includes only non-default values

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** AnnotationMapper instance created
- **Test Data:**
  ```python
  annotations = ModuleAnnotations(readonly=False, destructive=True, idempotent=True, requires_approval=False, open_world=True)
  ```
- **Steps:**
  1. Call `mapper.to_description_suffix(annotations)`.
  2. Assert result contains `destructive=true` and `idempotent=true`.
  3. Assert result does NOT contain `readonly=false` or `open_world=true` (defaults).
- **Expected Result:** `"\n\n[Annotations: destructive=true, idempotent=true]"`.
- **Traceability:** F-011

---

#### TC-ANNOT-010: Description suffix is empty for None annotations

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** AnnotationMapper instance created
- **Test Data:** `annotations = None`
- **Steps:**
  1. Call `mapper.to_description_suffix(None)`.
  2. Assert returns empty string.
- **Expected Result:** `""`.
- **Traceability:** F-011

---

### 5.3 Execution Router (TC-EXEC-xxx)

**Test File:** `tests/unit/server/test_router.py`

---

#### TC-EXEC-001: Successful tool call returns JSON output

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ExecutionRouter with mock Executor; `call_async` returns `{"status": "ok", "path": "/out/resized.png"}`
- **Test Data:**
  ```python
  tool_name = "image.resize"
  arguments = {"width": 800, "height": 600}
  ```
- **Steps:**
  1. Call `await router.handle_call("image.resize", {"width": 800, "height": 600})`.
  2. Assert `result.isError` is `False`.
  3. Parse `result.content[0].text` as JSON.
  4. Assert parsed JSON equals `{"status": "ok", "path": "/out/resized.png"}`.
- **Expected Result:** `CallToolResult` with `isError=False`, content contains `TextContent` with text `'{"status": "ok", "path": "/out/resized.png"}'`.
- **Traceability:** F-003 (AC1), F-013

---

#### TC-EXEC-002: Non-existent module returns error with module_id

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ExecutionRouter with mock Executor; `call_async` raises `ModuleNotFoundError("unknown.module")`
- **Test Data:**
  ```python
  tool_name = "unknown.module"
  arguments = {}
  ```
- **Steps:**
  1. Call `await router.handle_call("unknown.module", {})`.
  2. Assert `result.isError` is `True`.
  3. Assert `result.content[0].text` contains `"Module not found: unknown.module"`.
- **Expected Result:** `CallToolResult(isError=True)` with text `"Module not found: unknown.module"`.
- **Traceability:** F-003 (AC2), F-004

---

#### TC-EXEC-003: Schema validation failure returns field-level errors

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ExecutionRouter with mock Executor; `call_async` raises `SchemaValidationError("Validation failed", errors=[{"field": "width", "code": "int_type", "message": "Input should be a valid integer"}])`
- **Test Data:**
  ```python
  tool_name = "image.resize"
  arguments = {"width": "not_a_number", "height": 600}
  ```
- **Steps:**
  1. Call `await router.handle_call("image.resize", {"width": "not_a_number", "height": 600})`.
  2. Assert `result.isError` is `True`.
  3. Assert `result.content[0].text` contains `"Input validation failed"`.
  4. Assert text contains `"width"` field name.
- **Expected Result:** `CallToolResult(isError=True)` with text `"Input validation failed:\n- width: Input should be a valid integer (int_type)"`.
- **Traceability:** F-003 (AC3), F-004

---

#### TC-EXEC-004: ACL denied returns access denied without caller_id

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ExecutionRouter with mock Executor; `call_async` raises `ACLDeniedError("mcp_client", "image.resize")`
- **Test Data:**
  ```python
  tool_name = "image.resize"
  arguments = {"width": 800, "height": 600}
  ```
- **Steps:**
  1. Call `await router.handle_call("image.resize", {"width": 800, "height": 600})`.
  2. Assert `result.isError` is `True`.
  3. Assert `result.content[0].text` equals `"Access denied"`.
  4. Assert text does NOT contain `"mcp_client"`.
- **Expected Result:** `CallToolResult(isError=True)` with text exactly `"Access denied"`. No caller identity leaked.
- **Traceability:** F-003 (AC4), F-004

---

#### TC-EXEC-005: Timeout returns error with duration

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ExecutionRouter with mock Executor; `call_async` raises `ModuleTimeoutError("image.resize", 5000)`
- **Test Data:**
  ```python
  tool_name = "image.resize"
  arguments = {"width": 800, "height": 600}
  ```
- **Steps:**
  1. Call `await router.handle_call("image.resize", {"width": 800, "height": 600})`.
  2. Assert `result.isError` is `True`.
  3. Assert text contains `"timed out"` and `"5000ms"`.
- **Expected Result:** `CallToolResult(isError=True)` with text `"Module timed out after 5000ms"`.
- **Traceability:** F-003 (AC5), F-004

---

#### TC-EXEC-006: Unexpected exception returns sanitized error and logs traceback

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ExecutionRouter with mock Executor; `call_async` raises `RuntimeError("disk full")`
- **Test Data:**
  ```python
  tool_name = "image.resize"
  arguments = {"width": 800, "height": 600}
  ```
- **Steps:**
  1. Call `await router.handle_call(...)` with caplog fixture.
  2. Assert `result.isError` is `True`.
  3. Assert `result.content[0].text` equals `"Internal error occurred"`.
  4. Assert text does NOT contain `"disk full"`.
  5. Assert `"disk full"` appears in ERROR-level log output.
- **Expected Result:** Client receives `"Internal error occurred"`. Server logs full traceback at ERROR level with `"disk full"` message.
- **Traceability:** F-003, F-004, F-016

---

#### TC-EXEC-007: Empty arguments passed to executor

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** ExecutionRouter with mock Executor; `call_async` returns `{"status": "pong"}`
- **Test Data:**
  ```python
  tool_name = "system.ping"
  arguments = {}
  ```
- **Steps:**
  1. Call `await router.handle_call("system.ping", {})`.
  2. Assert mock Executor's `call_async` was called with `("system.ping", {})`.
  3. Assert `result.isError` is `False`.
- **Expected Result:** Empty dict passed through to Executor. Success result returned.
- **Traceability:** F-003

---

#### TC-EXEC-008: Non-serializable output uses default=str fallback

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** ExecutionRouter with mock Executor; `call_async` returns `{"timestamp": datetime(2026, 1, 15, 10, 30, 0), "path": PosixPath("/tmp/out.png")}`
- **Test Data:**
  ```python
  from datetime import datetime
  from pathlib import PosixPath
  tool_name = "image.resize"
  arguments = {"width": 800, "height": 600}
  ```
- **Steps:**
  1. Call `await router.handle_call(...)`.
  2. Assert `result.isError` is `False`.
  3. Parse `result.content[0].text` as JSON.
  4. Assert `timestamp` field is a string representation.
  5. Assert `path` field is `"/tmp/out.png"`.
- **Expected Result:** `json.dumps(output, default=str)` converts datetime and Path to strings. Output is valid JSON.
- **Traceability:** F-013

---

#### TC-EXEC-009: Concurrent tool calls handled correctly

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** ExecutionRouter with mock Executor; `call_async` returns different values based on module_id using side_effect
- **Test Data:**
  ```python
  async def mock_call(module_id, inputs, context=None):
      await asyncio.sleep(0.01)
      return {"module": module_id, "result": "ok"}
  executor.call_async = AsyncMock(side_effect=mock_call)
  ```
- **Steps:**
  1. Launch 10 concurrent `router.handle_call()` calls via `asyncio.gather()`.
  2. Assert all 10 results have `isError=False`.
  3. Assert each result contains the correct module_id.
- **Expected Result:** All 10 calls succeed independently. No cross-contamination between concurrent call results.
- **Traceability:** F-003

---

#### TC-EXEC-010: Module returning None handled gracefully

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** ExecutionRouter with mock Executor; `call_async` returns `None`
- **Test Data:**
  ```python
  tool_name = "system.shutdown"
  arguments = {}
  ```
- **Steps:**
  1. Call `await router.handle_call("system.shutdown", {})`.
  2. Assert `result.isError` is `False`.
  3. Assert content text is `"null"` or `"{}"`.
- **Expected Result:** `CallToolResult` with `isError=False`. Output serialized gracefully (null JSON or empty object).
- **Traceability:** F-003, F-013

---

#### TC-EXEC-011: Executor passthrough -- call_async receives exact arguments

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** ExecutionRouter with mock Executor
- **Test Data:**
  ```python
  tool_name = "text.translate"
  arguments = {"text": "Hello world", "target_lang": "es", "options": {"formal": True}}
  ```
- **Steps:**
  1. Call `await router.handle_call("text.translate", arguments)`.
  2. Assert `executor.call_async.assert_called_once_with("text.translate", {"text": "Hello world", "target_lang": "es", "options": {"formal": True}})`.
- **Expected Result:** Executor receives the exact tool_name and arguments dict without modification.
- **Traceability:** F-003, F-014

---

#### TC-EXEC-012: InvalidInputError returns descriptive message

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ExecutionRouter with mock Executor; `call_async` raises `InvalidInputError("Missing required field: width")`
- **Test Data:**
  ```python
  tool_name = "image.resize"
  arguments = {"height": 600}
  ```
- **Steps:**
  1. Call `await router.handle_call("image.resize", {"height": 600})`.
  2. Assert `result.isError` is `True`.
  3. Assert text contains `"Invalid input: Missing required field: width"`.
- **Expected Result:** `CallToolResult(isError=True)` with text `"Invalid input: Missing required field: width"`.
- **Traceability:** F-004

---

### 5.4 Error Mapper (TC-ERROR-xxx)

**Test File:** `tests/unit/adapters/test_errors.py`

---

#### TC-ERROR-001: ModuleNotFoundError mapping

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ErrorMapper instance created
- **Test Data:** `error = ModuleNotFoundError("image.resize")`
- **Steps:**
  1. Call `mapper.to_mcp_error(error)`.
  2. Assert `result.isError` is `True`.
  3. Assert text equals `"Module not found: image.resize"`.
- **Expected Result:** `CallToolResult(isError=True, content=[TextContent(text="Module not found: image.resize")])`.
- **Traceability:** F-004

---

#### TC-ERROR-002: SchemaValidationError with single field

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ErrorMapper instance created
- **Test Data:**
  ```python
  error = SchemaValidationError(
      "Validation failed",
      errors=[{"field": "width", "code": "int_type", "message": "Input should be a valid integer"}]
  )
  ```
- **Steps:**
  1. Call `mapper.to_mcp_error(error)`.
  2. Assert text starts with `"Input validation failed:"`.
  3. Assert text contains `"width: Input should be a valid integer (int_type)"`.
- **Expected Result:** `"Input validation failed:\n- width: Input should be a valid integer (int_type)"`.
- **Traceability:** F-004

---

#### TC-ERROR-003: SchemaValidationError with multiple fields

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ErrorMapper instance created
- **Test Data:**
  ```python
  error = SchemaValidationError(
      "Validation failed",
      errors=[
          {"field": "width", "code": "int_type", "message": "Input should be a valid integer"},
          {"field": "format", "code": "enum", "message": "Input should be 'png', 'jpg' or 'webp'"}
      ]
  )
  ```
- **Steps:**
  1. Call `mapper.to_mcp_error(error)`.
  2. Assert text contains both field error lines.
- **Expected Result:** Multi-line text with both `width` and `format` errors listed.
- **Traceability:** F-004

---

#### TC-ERROR-004: ACLDeniedError does not leak caller_id

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ErrorMapper instance created
- **Test Data:** `error = ACLDeniedError("secret_user_123", "image.resize")`
- **Steps:**
  1. Call `mapper.to_mcp_error(error)`.
  2. Assert text equals `"Access denied"`.
  3. Assert `"secret_user_123"` does NOT appear in text.
  4. Assert `"image.resize"` does NOT appear in text.
- **Expected Result:** Exactly `"Access denied"` -- no sensitive information.
- **Traceability:** F-004 (AC3)

---

#### TC-ERROR-005: ModuleTimeoutError includes duration

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ErrorMapper instance created
- **Test Data:** `error = ModuleTimeoutError("image.resize", 30000)`
- **Steps:**
  1. Call `mapper.to_mcp_error(error)`.
  2. Assert text equals `"Module timed out after 30000ms"`.
- **Expected Result:** `"Module timed out after 30000ms"`.
- **Traceability:** F-004

---

#### TC-ERROR-006: InvalidInputError preserves message

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ErrorMapper instance created
- **Test Data:** `error = InvalidInputError("Arguments must be a JSON object")`
- **Steps:**
  1. Call `mapper.to_mcp_error(error)`.
  2. Assert text equals `"Invalid input: Arguments must be a JSON object"`.
- **Expected Result:** `"Invalid input: Arguments must be a JSON object"`.
- **Traceability:** F-004

---

#### TC-ERROR-007: CallDepthExceededError returns generic message

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ErrorMapper instance created
- **Test Data:** `error = CallDepthExceededError(depth=10, max_depth=5, call_chain=["a", "b", "c"])`
- **Steps:**
  1. Call `mapper.to_mcp_error(error)`.
  2. Assert text equals `"Call depth limit exceeded"`.
  3. Assert text does NOT contain the call chain.
- **Expected Result:** `"Call depth limit exceeded"`. No internal call chain exposed.
- **Traceability:** F-004

---

#### TC-ERROR-008: CircularCallError returns generic message

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ErrorMapper instance created
- **Test Data:** `error = CircularCallError(module_id="a.module", call_chain=["a.module", "b.module", "a.module"])`
- **Steps:**
  1. Call `mapper.to_mcp_error(error)`.
  2. Assert text equals `"Circular call detected"`.
- **Expected Result:** `"Circular call detected"`. No call chain exposed.
- **Traceability:** F-004

---

#### TC-ERROR-009: CallFrequencyExceededError returns generic message

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ErrorMapper instance created
- **Test Data:** `error = CallFrequencyExceededError(module_id="spam.module", count=100, max_repeat=10, call_chain=["spam.module"])`
- **Steps:**
  1. Call `mapper.to_mcp_error(error)`.
  2. Assert text equals `"Call frequency limit exceeded"`.
- **Expected Result:** `"Call frequency limit exceeded"`.
- **Traceability:** F-004

---

#### TC-ERROR-010: Unexpected exception returns sanitized message

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ErrorMapper instance created
- **Test Data:** `error = RuntimeError("Connection to database failed at 10.0.0.5:5432")`
- **Steps:**
  1. Call `mapper.to_mcp_error(error)`.
  2. Assert text equals `"Internal error occurred"`.
  3. Assert `"10.0.0.5"` does NOT appear in text.
  4. Assert `"database"` does NOT appear in text.
- **Expected Result:** Exactly `"Internal error occurred"`. No internal details leaked.
- **Traceability:** F-004 (AC4)

---

#### TC-ERROR-011: All error results have isError=True

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ErrorMapper instance created
- **Test Data:** One instance of each error type:
  ```python
  errors = [
      ModuleNotFoundError("x"),
      SchemaValidationError("v", errors=[]),
      ACLDeniedError("c", "t"),
      ModuleTimeoutError("x", 1000),
      InvalidInputError("bad"),
      CallDepthExceededError(5, 3, []),
      CircularCallError("x", []),
      CallFrequencyExceededError("x", 10, 5, []),
      RuntimeError("unexpected"),
  ]
  ```
- **Steps:**
  1. For each error, call `mapper.to_mcp_error(error)`.
  2. Assert `result.isError` is `True` for every result.
- **Expected Result:** All 9 results have `isError=True`.
- **Traceability:** F-004 (AC5)

---

### 5.5 MCP Server Factory (TC-SERVER-xxx)

**Test File:** `tests/unit/server/test_factory.py`

---

#### TC-SERVER-001: Build single tool from ModuleDescriptor

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** MCPServerFactory instance created
- **Test Data:**
  ```python
  descriptor = ModuleDescriptor(
      module_id="image.resize",
      description="Resize an image to the specified dimensions",
      input_schema={
          "type": "object",
          "properties": {
              "width": {"type": "integer"},
              "height": {"type": "integer"}
          },
          "required": ["width", "height"]
      },
      output_schema={},
      annotations=ModuleAnnotations(readonly=False, destructive=False, idempotent=True, requires_approval=False, open_world=True)
  )
  ```
- **Steps:**
  1. Call `factory.build_tool(descriptor)`.
  2. Assert `tool.name` equals `"image.resize"`.
  3. Assert `tool.description` equals `"Resize an image to the specified dimensions"`.
  4. Assert `tool.inputSchema` has `type: object` with `width` and `height` properties.
  5. Assert `tool.annotations.idempotent_hint` is `True`.
- **Expected Result:** `types.Tool` with name, description, inputSchema, and annotations all correctly mapped from the descriptor.
- **Traceability:** F-001, F-002, F-005

---

#### TC-SERVER-002: Build tools from registry with multiple modules

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** MCPServerFactory instance, mock Registry returning 3 module IDs
- **Test Data:** Mock registry with modules `["image.resize", "text.summarize", "system.ping"]`, each returning a valid ModuleDescriptor.
- **Steps:**
  1. Call `factory.build_tools(registry)`.
  2. Assert returned list has length 3.
  3. Assert tool names match the module IDs.
- **Expected Result:** List of 3 `types.Tool` objects with correct names.
- **Traceability:** F-001, F-005

---

#### TC-SERVER-003: serve() accepts Executor instance

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** Mock Executor with `.registry` property returning mock Registry
- **Test Data:**
  ```python
  executor = MagicMock(spec=Executor)
  executor.registry = mock_registry
  ```
- **Steps:**
  1. Call `serve(executor)` (mocking transport to avoid blocking).
  2. Assert Executor's registry is used for tool building.
  3. Assert Executor is used for call routing.
- **Expected Result:** serve() extracts registry from executor and uses executor for call_async routing.
- **Traceability:** F-005 (AC5), F-014

---

#### TC-SERVER-004: Empty registry produces empty tool list with warning

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** MCPServerFactory instance, mock Registry with `list()` returning `[]`
- **Test Data:** `registry.list.return_value = []`
- **Steps:**
  1. Call `factory.build_tools(registry)` with caplog fixture.
  2. Assert returned list is empty.
  3. Assert WARNING log contains "No modules registered" or similar.
- **Expected Result:** Empty list `[]` returned. Warning logged.
- **Traceability:** F-005 (AC7)

---

#### TC-SERVER-005: Tool with None annotations uses defaults

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** MCPServerFactory instance
- **Test Data:**
  ```python
  descriptor = ModuleDescriptor(
      module_id="system.ping",
      description="Ping",
      input_schema={},
      output_schema={},
      annotations=None
  )
  ```
- **Steps:**
  1. Call `factory.build_tool(descriptor)`.
  2. Assert `tool.annotations.read_only_hint` is `False`.
  3. Assert `tool.annotations.open_world_hint` is `True`.
- **Expected Result:** Tool uses MCP default annotation values.
- **Traceability:** F-002 (AC5)

---

#### TC-SERVER-006: Create server with custom name and version

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** MCPServerFactory instance
- **Test Data:** `name="my-tools"`, `version="2.0.0"`
- **Steps:**
  1. Call `factory.create_server(name="my-tools", version="2.0.0")`.
  2. Assert server is created (returns Server instance).
- **Expected Result:** Server instance created with custom name and version.
- **Traceability:** F-005 (AC4)

---

#### TC-SERVER-007: Build tools with tag filter

- **Priority:** P2
- **Type:** Unit
- **Preconditions:** MCPServerFactory, mock Registry where `list(tags=["image"])` returns `["image.resize"]`
- **Test Data:** `tags=["image"]`
- **Steps:**
  1. Call `factory.build_tools(registry, tags=["image"])`.
  2. Assert `registry.list` was called with `tags=["image"]`.
  3. Assert only `"image.resize"` tool is in result.
- **Expected Result:** Filtered list with 1 tool.
- **Traceability:** F-018

---

#### TC-SERVER-008: Build tools with prefix filter

- **Priority:** P2
- **Type:** Unit
- **Preconditions:** MCPServerFactory, mock Registry where `list(prefix="image.")` returns `["image.resize", "image.crop"]`
- **Test Data:** `prefix="image."`
- **Steps:**
  1. Call `factory.build_tools(registry, prefix="image.")`.
  2. Assert only image-prefixed tools returned.
- **Expected Result:** List with 2 tools: `image.resize` and `image.crop`.
- **Traceability:** F-018

---

#### TC-SERVER-009: Server startup logs tool count and transport

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** Mock serve() internals with caplog
- **Test Data:** Registry with 5 modules, transport="stdio"
- **Steps:**
  1. Trigger server startup logging.
  2. Assert INFO log contains tool count (e.g., "5 tools registered").
  3. Assert INFO log contains transport type (e.g., "stdio").
- **Expected Result:** Log message like `"Starting apcore-mcp server with 5 tools via stdio transport"`.
- **Traceability:** F-016

---

#### TC-SERVER-010: serve() rejects invalid transport name

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** None
- **Test Data:** `transport="websocket"`
- **Steps:**
  1. Call `serve(registry, transport="websocket")`.
  2. Assert `ValueError` is raised.
- **Expected Result:** `ValueError` with message indicating `"websocket"` is not a valid transport. Allowed: `stdio`, `streamable-http`, `sse`.
- **Traceability:** F-005

---

### 5.6 OpenAI Converter (TC-OPENAI-xxx)

**Test File:** `tests/unit/converters/test_openai.py`

---

#### TC-OPENAI-001: Basic conversion produces correct structure

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** OpenAIConverter instance, mock Registry with one module
- **Test Data:**
  ```python
  descriptor = ModuleDescriptor(
      module_id="image.resize",
      description="Resize an image to the specified dimensions",
      input_schema={"type": "object", "properties": {"width": {"type": "integer"}, "height": {"type": "integer"}}, "required": ["width", "height"]},
      output_schema={}
  )
  ```
- **Steps:**
  1. Call `converter.convert_descriptor(descriptor)`.
  2. Assert result has `"type": "function"`.
  3. Assert `result["function"]["name"]` equals `"image-resize"` (dots normalized).
  4. Assert `result["function"]["description"]` equals the original description.
  5. Assert `result["function"]["parameters"]` equals the input_schema.
- **Expected Result:**
  ```python
  {
      "type": "function",
      "function": {
          "name": "image-resize",
          "description": "Resize an image to the specified dimensions",
          "parameters": {"type": "object", "properties": {"width": {"type": "integer"}, "height": {"type": "integer"}}, "required": ["width", "height"]}
      }
  }
  ```
- **Traceability:** F-008 (AC1, AC2, AC3, AC4, AC5, AC6)

---

#### TC-OPENAI-002: Module ID normalization replaces dots with hyphens

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ModuleIDNormalizer instance
- **Test Data:**
  ```python
  test_cases = [
      ("image.resize", "image-resize"),
      ("comfyui.workflow.execute", "comfyui-workflow-execute"),
      ("simple", "simple"),
      ("a.b.c.d.e", "a-b-c-d-e"),
  ]
  ```
- **Steps:**
  1. For each (input, expected), call `normalizer.normalize(input)`.
  2. Assert result equals expected.
- **Expected Result:** All dot-separated module IDs converted to hyphen format. IDs without dots are unchanged.
- **Traceability:** F-008

---

#### TC-OPENAI-003: Module ID denormalization reverses normalization

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** ModuleIDNormalizer instance
- **Test Data:**
  ```python
  test_cases = [
      ("image-resize", "image.resize"),
      ("comfyui-workflow-execute", "comfyui.workflow.execute"),
      ("simple", "simple"),
  ]
  ```
- **Steps:**
  1. For each (input, expected), call `normalizer.denormalize(input)`.
  2. Assert result equals expected.
- **Expected Result:** Hyphens converted back to dots.
- **Traceability:** F-008

---

#### TC-OPENAI-004: Empty registry returns empty list

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** OpenAIConverter, mock Registry with `list()` returning `[]`
- **Test Data:** Empty registry
- **Steps:**
  1. Call `converter.convert_registry(registry)`.
  2. Assert result is `[]`.
- **Expected Result:** Empty list `[]`.
- **Traceability:** F-008 (AC8)

---

#### TC-OPENAI-005: embed_annotations=True appends annotation suffix

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** OpenAIConverter, descriptor with `destructive=True, idempotent=True`
- **Test Data:**
  ```python
  descriptor = ModuleDescriptor(
      module_id="file.delete",
      description="Delete a file",
      input_schema={"type": "object", "properties": {"path": {"type": "string"}}, "required": ["path"]},
      output_schema={},
      annotations=ModuleAnnotations(readonly=False, destructive=True, idempotent=True, requires_approval=True, open_world=True)
  )
  ```
- **Steps:**
  1. Call `converter.convert_descriptor(descriptor, embed_annotations=True)`.
  2. Assert `result["function"]["description"]` contains `"Delete a file"`.
  3. Assert description contains `"[Annotations:"`.
  4. Assert description contains `"destructive=true"` and `"idempotent=true"` and `"requires_approval=true"`.
- **Expected Result:** Description equals `"Delete a file\n\n[Annotations: destructive=true, idempotent=true, requires_approval=true]"`.
- **Traceability:** F-011 (AC1, AC3, AC4)

---

#### TC-OPENAI-006: embed_annotations=False does not modify description

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** OpenAIConverter, descriptor with annotations
- **Test Data:** Same descriptor as TC-OPENAI-005
- **Steps:**
  1. Call `converter.convert_descriptor(descriptor, embed_annotations=False)`.
  2. Assert description equals exactly `"Delete a file"` with no suffix.
- **Expected Result:** `"Delete a file"` -- unmodified.
- **Traceability:** F-011 (AC2)

---

#### TC-OPENAI-007: strict=True adds strict field and modifies schema

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** OpenAIConverter
- **Test Data:**
  ```python
  descriptor = ModuleDescriptor(
      module_id="image.resize",
      description="Resize",
      input_schema={
          "type": "object",
          "properties": {
              "width": {"type": "integer"},
              "height": {"type": "integer"},
              "format": {"type": "string", "default": "png"}
          },
          "required": ["width", "height"]
      },
      output_schema={}
  )
  ```
- **Steps:**
  1. Call `converter.convert_descriptor(descriptor, strict=True)`.
  2. Assert `result["function"]["strict"]` is `True`.
  3. Assert `parameters["additionalProperties"]` is `False`.
  4. Assert `"format"` is in `parameters["required"]`.
  5. Assert `parameters["properties"]["format"]["type"]` includes `"null"` (nullable).
  6. Assert `"default"` key is not in `parameters["properties"]["format"]`.
- **Expected Result:** `strict: true` added. Schema has `additionalProperties: false`, all properties required, optional `format` becomes nullable, `default` removed.
- **Traceability:** F-012 (AC1, AC3)

---

#### TC-OPENAI-008: strict=False does not add strict field

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** OpenAIConverter
- **Test Data:** Same descriptor as TC-OPENAI-007
- **Steps:**
  1. Call `converter.convert_descriptor(descriptor, strict=False)`.
  2. Assert `"strict"` key is NOT in `result["function"]`.
  3. Assert schema is unmodified.
- **Expected Result:** No `strict` key. Schema preserves `default` values and original `required` list.
- **Traceability:** F-012 (AC2)

---

#### TC-OPENAI-009: Output directly usable with OpenAI API format

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** OpenAIConverter, registry with 2 modules
- **Test Data:** Two descriptors: `image.resize` and `text.summarize`
- **Steps:**
  1. Call `converter.convert_registry(registry)`.
  2. Assert result is a `list`.
  3. Assert length is 2.
  4. For each item, assert `item["type"] == "function"`.
  5. For each item, assert `"name"` in `item["function"]`.
  6. For each item, assert `"description"` in `item["function"]`.
  7. For each item, assert `"parameters"` in `item["function"]`.
- **Expected Result:** List of 2 dicts, each with `type: "function"` and `function` containing `name`, `description`, `parameters`. Directly passable to `openai.chat.completions.create(tools=...)`.
- **Traceability:** F-008 (AC7, AC9)

---

#### TC-OPENAI-010: Tag filtering returns only matching modules

- **Priority:** P2
- **Type:** Unit
- **Preconditions:** OpenAIConverter, registry with `list(tags=["image"])` returning `["image.resize"]`
- **Test Data:** `tags=["image"]`
- **Steps:**
  1. Call `converter.convert_registry(registry, tags=["image"])`.
  2. Assert result has 1 item with name `"image-resize"`.
- **Expected Result:** Single tool in list.
- **Traceability:** F-017

---

#### TC-OPENAI-011: Prefix filtering returns only matching modules

- **Priority:** P2
- **Type:** Unit
- **Preconditions:** OpenAIConverter, registry with `list(prefix="comfyui.")` returning `["comfyui.workflow"]`
- **Test Data:** `prefix="comfyui."`
- **Steps:**
  1. Call `converter.convert_registry(registry, prefix="comfyui.")`.
  2. Assert result has 1 item.
- **Expected Result:** Single tool in list with name `"comfyui-workflow"`.
- **Traceability:** F-017

---

#### TC-OPENAI-012: No dependency on openai package

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** None
- **Test Data:** N/A
- **Steps:**
  1. Import `apcore_mcp.converters.openai` module.
  2. Assert no `import openai` in the module source.
  3. Call `to_openai_tools(registry)`.
  4. Assert result is `list[dict]` (plain Python types only).
- **Expected Result:** Function works without `openai` package installed. Returns plain dicts.
- **Traceability:** F-008 (AC9)

---

### 5.7 Transport Manager (TC-TRANSPORT-xxx)

**Test File:** `tests/unit/server/test_transport.py`

---

#### TC-TRANSPORT-001: stdio transport starts successfully

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** TransportManager instance, mock Server, mock stdio_server
- **Test Data:** Default parameters (no host/port needed for stdio)
- **Steps:**
  1. Mock `mcp.server.stdio.stdio_server` context manager.
  2. Call `await transport.run_stdio(server, init_options)`.
  3. Assert `stdio_server` was called.
  4. Assert `server.run` was called with read_stream and write_stream.
- **Expected Result:** stdio transport lifecycle initiated correctly.
- **Traceability:** F-006

---

#### TC-TRANSPORT-002: stdio transport handles graceful shutdown

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** TransportManager instance, mock Server
- **Test Data:** N/A
- **Steps:**
  1. Start stdio transport.
  2. Simulate read stream closing (EOF).
  3. Assert no exception raised and method returns cleanly.
- **Expected Result:** Clean shutdown without exceptions.
- **Traceability:** F-006 (AC4)

---

#### TC-TRANSPORT-003: stdio is the default transport

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** None
- **Test Data:** `serve(registry)` called without transport parameter
- **Steps:**
  1. Verify `serve()` function signature has `transport: str = "stdio"`.
  2. Mock internals to verify stdio transport is selected.
- **Expected Result:** Default transport is `"stdio"`.
- **Traceability:** F-005 (AC1), F-006

---

#### TC-TRANSPORT-004: Streamable HTTP transport starts with host and port

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** TransportManager instance, mock Server
- **Test Data:** `host="127.0.0.1"`, `port=8000`
- **Steps:**
  1. Call `await transport.run_streamable_http(server, init_options, host="127.0.0.1", port=8000)`.
  2. Assert HTTP server started on the specified host and port.
- **Expected Result:** HTTP transport started on 127.0.0.1:8000.
- **Traceability:** F-007 (AC1, AC3)

---

#### TC-TRANSPORT-005: Invalid port raises ValueError

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** TransportManager instance
- **Test Data:** `port=0`, `port=65536`, `port=-1`
- **Steps:**
  1. For each invalid port, call `transport.run_streamable_http(server, init_options, port=port)`.
  2. Assert `ValueError` is raised each time.
- **Expected Result:** `ValueError` for port 0, 65536, and -1.
- **Traceability:** F-007

---

#### TC-TRANSPORT-006: Empty host raises ValueError

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** TransportManager instance
- **Test Data:** `host=""`
- **Steps:**
  1. Call `transport.run_streamable_http(server, init_options, host="", port=8000)`.
  2. Assert `ValueError` is raised.
- **Expected Result:** `ValueError` indicating host must not be empty.
- **Traceability:** F-007

---

#### TC-TRANSPORT-007: SSE transport starts successfully

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** TransportManager instance, mock Server
- **Test Data:** `host="127.0.0.1"`, `port=8000`
- **Steps:**
  1. Call `await transport.run_sse(server, init_options, host="127.0.0.1", port=8000)`.
  2. Assert SSE server started.
- **Expected Result:** SSE transport lifecycle initiated.
- **Traceability:** F-010

---

#### TC-TRANSPORT-008: SSE transport logs deprecation warning

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** TransportManager with caplog
- **Test Data:** N/A
- **Steps:**
  1. Call `await transport.run_sse(...)`.
  2. Assert WARNING log contains "deprecated" or "SSE".
- **Expected Result:** Deprecation warning logged.
- **Traceability:** F-010 (AC2)

---

#### TC-TRANSPORT-009: Default host and port values

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** Inspect TransportManager API
- **Test Data:** N/A
- **Steps:**
  1. Assert `run_streamable_http` default `host` is `"127.0.0.1"`.
  2. Assert `run_streamable_http` default `port` is `8000`.
  3. Assert same defaults for `run_sse`.
- **Expected Result:** Default host `"127.0.0.1"`, default port `8000`.
- **Traceability:** F-007 (AC3)

---

### 5.8 CLI Module (TC-CLI-xxx)

**Test File:** `tests/unit/test_cli.py`

---

#### TC-CLI-001: Basic usage with --extensions-dir

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** Mock Registry.discover(), mock serve()
- **Test Data:** `args = ["--extensions-dir", "/tmp/test_extensions"]` (directory exists)
- **Steps:**
  1. Mock `os.path.isdir` to return `True` for `/tmp/test_extensions`.
  2. Run CLI `main()` with args.
  3. Assert Registry was created with `extensions_dir="/tmp/test_extensions"`.
  4. Assert `registry.discover()` was called.
  5. Assert `serve(registry)` was called.
- **Expected Result:** Registry created, discover() called, serve() called with default transport.
- **Traceability:** F-009 (AC1)

---

#### TC-CLI-002: --transport flag accepts all valid values

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** Mock serve()
- **Test Data:**
  ```python
  transports = ["stdio", "streamable-http", "sse"]
  ```
- **Steps:**
  1. For each transport, run CLI with `["--extensions-dir", "/tmp/ext", "--transport", transport]`.
  2. Assert `serve()` was called with `transport=transport`.
- **Expected Result:** All three transport values accepted without error.
- **Traceability:** F-009 (AC2)

---

#### TC-CLI-003: --host and --port flags configure network transports

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** Mock serve()
- **Test Data:** `args = ["--extensions-dir", "/tmp/ext", "--transport", "streamable-http", "--host", "0.0.0.0", "--port", "9000"]`
- **Steps:**
  1. Run CLI main() with args.
  2. Assert `serve()` called with `host="0.0.0.0"` and `port=9000`.
- **Expected Result:** serve() receives custom host and port.
- **Traceability:** F-009 (AC3)

---

#### TC-CLI-004: --help displays usage information

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** None
- **Test Data:** `args = ["--help"]`
- **Steps:**
  1. Run CLI main() with `["--help"]`.
  2. Capture stdout.
  3. Assert output contains `"--extensions-dir"`, `"--transport"`, `"--host"`, `"--port"`.
- **Expected Result:** Help text displayed with all flag descriptions. SystemExit(0) raised.
- **Traceability:** F-009 (AC4)

---

#### TC-CLI-005: Non-existent extensions-dir exits with error

- **Priority:** P0
- **Type:** Unit
- **Preconditions:** None
- **Test Data:** `args = ["--extensions-dir", "/nonexistent/path/to/extensions"]`
- **Steps:**
  1. Run CLI main() with args.
  2. Assert SystemExit with code 1.
  3. Assert stderr contains error message about directory not existing.
- **Expected Result:** Exit code 1 with clear error message.
- **Traceability:** F-009 (AC5)

---

#### TC-CLI-006: Invalid transport flag exits with error

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** None
- **Test Data:** `args = ["--extensions-dir", "/tmp/ext", "--transport", "websocket"]`
- **Steps:**
  1. Run CLI main() with args.
  2. Assert SystemExit with code 2 (argparse error).
- **Expected Result:** Exit code 2 with argparse error message.
- **Traceability:** F-009

---

#### TC-CLI-007: No modules discovered logs warning but starts server

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** Mock Registry.discover() returning 0, mock serve()
- **Test Data:** `args = ["--extensions-dir", "/tmp/empty_dir"]`
- **Steps:**
  1. Mock `registry.discover()` to return 0.
  2. Run CLI main() with args and caplog.
  3. Assert WARNING log about no modules discovered.
  4. Assert `serve()` was still called.
- **Expected Result:** Warning logged, server starts with zero tools.
- **Traceability:** F-009 (AC6)

---

#### TC-CLI-008: --log-level flag sets logging level

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** Mock serve()
- **Test Data:** `args = ["--extensions-dir", "/tmp/ext", "--log-level", "DEBUG"]`
- **Steps:**
  1. Run CLI main() with args.
  2. Assert logging level for `apcore_mcp` logger set to DEBUG.
- **Expected Result:** Logger configured at DEBUG level.
- **Traceability:** F-009, F-016

---

#### TC-CLI-009: --name and --version flags set server identity

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** Mock serve()
- **Test Data:** `args = ["--extensions-dir", "/tmp/ext", "--name", "my-server", "--version", "3.0.0"]`
- **Steps:**
  1. Run CLI main() with args.
  2. Assert `serve()` called with `name="my-server"` and `version="3.0.0"`.
- **Expected Result:** Custom server name and version passed to serve().
- **Traceability:** F-009

---

#### TC-CLI-010: Invalid port exits with error

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** None
- **Test Data:** `args = ["--extensions-dir", "/tmp/ext", "--transport", "streamable-http", "--port", "99999"]`
- **Steps:**
  1. Run CLI main() with args.
  2. Assert SystemExit with code 1.
- **Expected Result:** Exit code 1 with error about invalid port.
- **Traceability:** F-009

---

### 5.9 Dynamic Registry Listener (TC-DYNAMIC-xxx)

**Test File:** `tests/unit/server/test_listener.py`

---

#### TC-DYNAMIC-001: Register new module adds tool to list

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** RegistryListener started with mock Registry and MCPServerFactory
- **Test Data:**
  ```python
  module_id = "new.tool"
  descriptor = ModuleDescriptor(module_id="new.tool", description="New tool", input_schema={}, output_schema={})
  factory.build_tool.return_value = types.Tool(name="new.tool", description="New tool", inputSchema={"type": "object", "properties": {}})
  ```
- **Steps:**
  1. Simulate registry `register` event: call `listener._on_register("new.tool", mock_module)`.
  2. Assert `factory.build_tool` was called.
  3. Assert `listener.tools` dict contains `"new.tool"`.
- **Expected Result:** Tool added to listener's internal tools dict.
- **Traceability:** F-015 (AC1)

---

#### TC-DYNAMIC-002: Unregister module removes tool from list

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** RegistryListener with one existing tool `"old.tool"` in tools dict
- **Test Data:** `module_id = "old.tool"`
- **Steps:**
  1. Pre-populate `listener._tools["old.tool"] = mock_tool`.
  2. Call `listener._on_unregister("old.tool", mock_module)`.
  3. Assert `"old.tool"` is NOT in `listener.tools`.
- **Expected Result:** Tool removed from tools dict.
- **Traceability:** F-015 (AC2)

---

#### TC-DYNAMIC-003: Register when get_definition returns None

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** RegistryListener, mock Registry with `get_definition` returning `None`
- **Test Data:** `module_id = "ghost.tool"`
- **Steps:**
  1. Call `listener._on_register("ghost.tool", mock_module)`.
  2. Assert `"ghost.tool"` is NOT in `listener.tools`.
  3. Assert warning logged.
- **Expected Result:** Tool NOT added. Warning logged. No crash.
- **Traceability:** F-015

---

#### TC-DYNAMIC-004: Unregister non-existent module is silent

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** RegistryListener with empty tools dict
- **Test Data:** `module_id = "nonexistent.tool"`
- **Steps:**
  1. Call `listener._on_unregister("nonexistent.tool", mock_module)`.
  2. Assert no exception raised.
- **Expected Result:** No exception. Silent no-op.
- **Traceability:** F-015

---

#### TC-DYNAMIC-005: Start registers event callbacks

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** RegistryListener, mock Registry
- **Test Data:** N/A
- **Steps:**
  1. Call `listener.start()`.
  2. Assert `registry.on("register", ...)` was called.
  3. Assert `registry.on("unregister", ...)` was called.
- **Expected Result:** Both event callbacks registered on the Registry.
- **Traceability:** F-015

---

#### TC-DYNAMIC-006: Stop causes callbacks to no-op

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** RegistryListener started
- **Test Data:** N/A
- **Steps:**
  1. Call `listener.start()`.
  2. Call `listener.stop()`.
  3. Simulate register event.
  4. Assert tools dict is unchanged.
- **Expected Result:** After stop(), register events are ignored.
- **Traceability:** F-015

---

#### TC-DYNAMIC-007: Concurrent register and unregister are thread-safe

- **Priority:** P1
- **Type:** Unit
- **Preconditions:** RegistryListener started
- **Test Data:** 50 register events and 50 unregister events fired concurrently from different threads
- **Steps:**
  1. Launch 50 threads each calling `_on_register(f"tool.{i}", mock)`.
  2. Launch 50 threads each calling `_on_unregister(f"tool.{i}", mock)`.
  3. Join all threads.
  4. Assert no exceptions raised.
  5. Assert tools dict is in a consistent state (no partial entries).
- **Expected Result:** No race conditions, no exceptions. Tools dict is consistent.
- **Traceability:** F-015 (AC4)

---

## 6. Integration Test Cases (TC-INT-xxx)

**Test Directory:** `tests/integration/`

---

#### TC-INT-001: Full MCP flow -- Registry to tool list to tool call to result

- **Priority:** P0
- **Type:** Integration
- **Preconditions:** Real SchemaConverter, AnnotationMapper, ErrorMapper, MCPServerFactory, ExecutionRouter. Mock Executor with `call_async` returning `{"status": "ok"}`.
- **Test Data:**
  ```python
  descriptor = ModuleDescriptor(
      module_id="image.resize",
      description="Resize an image",
      input_schema={"type": "object", "properties": {"width": {"type": "integer"}}, "required": ["width"]},
      output_schema={},
      annotations=ModuleAnnotations(readonly=False, destructive=False, idempotent=True, requires_approval=False, open_world=True)
  )
  mock_registry.list.return_value = ["image.resize"]
  mock_registry.get_definition.return_value = descriptor
  ```
- **Steps:**
  1. Create MCPServerFactory and build tools from registry.
  2. Assert tool list has 1 tool with name `"image.resize"`.
  3. Create ExecutionRouter with mock executor.
  4. Call `router.handle_call("image.resize", {"width": 800})`.
  5. Assert result is success with `{"status": "ok"}`.
- **Expected Result:** End-to-end flow: schema converted, tool built, call routed, result returned.
- **Traceability:** F-001, F-002, F-003, F-005

---

#### TC-INT-002: Same flow works over all three transports

- **Priority:** P0
- **Type:** Integration
- **Preconditions:** Full apcore-mcp stack with mock module
- **Test Data:** Transport values: `["stdio", "streamable-http", "sse"]`
- **Steps:**
  1. For each transport, start server (with timeout/background).
  2. Connect MCP client (mock or SDK client).
  3. List tools.
  4. Call a tool.
  5. Assert result matches expected.
  6. Shut down server.
- **Expected Result:** Identical behavior across all three transports.
- **Traceability:** F-006, F-007, F-010

---

#### TC-INT-003: Error flow -- MCP client calls non-existent tool

- **Priority:** P0
- **Type:** Integration
- **Preconditions:** Server running with mock registry (1 module: `image.resize`)
- **Test Data:**
  ```python
  tool_name = "nonexistent.tool"
  arguments = {"key": "value"}
  ```
- **Steps:**
  1. Build tools from registry (only `image.resize`).
  2. Call `router.handle_call("nonexistent.tool", {"key": "value"})`.
  3. Assert `result.isError` is `True`.
  4. Assert text contains `"Module not found: nonexistent.tool"`.
- **Expected Result:** Error result with module not found message.
- **Traceability:** F-003 (AC2), F-004

---

#### TC-INT-004: to_openai_tools() roundtrip -- format matches OpenAI spec

- **Priority:** P0
- **Type:** Integration
- **Preconditions:** Registry with 3 modules of varying complexity
- **Test Data:** Three descriptors with simple, nested ($ref), and empty schemas.
- **Steps:**
  1. Call `to_openai_tools(registry)`.
  2. Assert result is a list of 3 dicts.
  3. Validate each dict against OpenAI tool schema structure.
  4. Assert all module IDs are normalized (no dots).
  5. Assert all parameters are valid JSON Schema.
- **Expected Result:** Output is valid OpenAI tools format. All $refs are inlined. All module IDs are normalized.
- **Traceability:** F-008

---

#### TC-INT-005: Executor passthrough with ACL enforcement

- **Priority:** P1
- **Type:** Integration
- **Preconditions:** Real Executor with ACL configured, real Registry with 2 modules
- **Test Data:**
  ```python
  # Module "public.tool" -- allowed
  # Module "private.tool" -- denied by ACL
  executor.call_async("public.tool", {}) -> {"result": "ok"}
  executor.call_async("private.tool", {}) -> raises ACLDeniedError
  ```
- **Steps:**
  1. Create ExecutionRouter with real Executor.
  2. Call `router.handle_call("public.tool", {})` -- assert success.
  3. Call `router.handle_call("private.tool", {})` -- assert `isError=True` with `"Access denied"`.
- **Expected Result:** ACL enforcement works through the full stack.
- **Traceability:** F-014, F-003 (AC4)

---

#### TC-INT-006: Dynamic registration -- add module while server components are running

- **Priority:** P1
- **Type:** Integration
- **Preconditions:** RegistryListener started, MCPServerFactory, mock Registry
- **Test Data:**
  ```python
  new_descriptor = ModuleDescriptor(
      module_id="new.module",
      description="Dynamically added module",
      input_schema={"type": "object", "properties": {"x": {"type": "integer"}}},
      output_schema={}
  )
  ```
- **Steps:**
  1. Start with empty tool list.
  2. Simulate registry register event for "new.module".
  3. Assert `listener.tools` now contains `"new.module"`.
  4. Build tool list from listener -- assert 1 tool present.
- **Expected Result:** New tool appears in the tool list after dynamic registration.
- **Traceability:** F-015

---

## 7. E2E Test Cases (TC-E2E-xxx)

**Test Directory:** `tests/e2e/`

---

#### TC-E2E-001: CLI server startup, MCP client connection, tool execution, shutdown

- **Priority:** P0
- **Type:** E2E
- **Preconditions:** Test extensions directory with at least 1 mock apcore module, MCP client library available
- **Test Data:**
  ```python
  extensions_dir = "/tmp/test_extensions"  # contains a simple mock module
  cli_args = ["python", "-m", "apcore_mcp", "--extensions-dir", extensions_dir]
  ```
- **Steps:**
  1. Create a temp directory with a mock apcore module.
  2. Start `python -m apcore_mcp --extensions-dir <temp_dir>` as a subprocess.
  3. Connect MCP client via stdio (reading subprocess stdout, writing to stdin).
  4. Send `tools/list` request.
  5. Assert response contains the mock module as a tool.
  6. Send `tools/call` request for the mock module.
  7. Assert response contains expected output.
  8. Terminate subprocess (SIGTERM).
  9. Assert clean shutdown (exit code 0).
- **Expected Result:** Full lifecycle works: startup, discovery, connection, tool listing, tool call, shutdown.
- **Traceability:** F-005, F-006, F-009

---

#### TC-E2E-002: HTTP transport server with tool execution

- **Priority:** P0
- **Type:** E2E
- **Preconditions:** Test extensions directory, available port
- **Test Data:**
  ```python
  cli_args = ["python", "-m", "apcore_mcp", "--extensions-dir", extensions_dir,
              "--transport", "streamable-http", "--port", "18765"]
  ```
- **Steps:**
  1. Start server subprocess with streamable-http transport.
  2. Wait for server to be ready (poll health or retry connect).
  3. Connect MCP client via HTTP to `http://127.0.0.1:18765`.
  4. List tools.
  5. Call a tool.
  6. Assert success response.
  7. Shut down server.
- **Expected Result:** Full HTTP transport lifecycle works.
- **Traceability:** F-007, F-009

---

#### TC-E2E-003: Hot-reload -- add module while server is running

- **Priority:** P1
- **Type:** E2E
- **Preconditions:** Server running with initial modules, programmatic access to Registry
- **Test Data:** Initial: 2 modules. After add: 3 modules.
- **Steps:**
  1. Start server with 2 modules.
  2. Connect MCP client. List tools -- assert 2 tools.
  3. Register a new module to the registry.
  4. Wait for tool list change notification (or re-list tools).
  5. List tools -- assert 3 tools.
  6. Call the new tool -- assert success.
- **Expected Result:** Dynamically added module appears as a callable tool.
- **Traceability:** F-015

---

#### TC-E2E-004: Multi-module registry with mixed annotations

- **Priority:** P1
- **Type:** E2E
- **Preconditions:** Registry with modules having varied annotation combinations
- **Test Data:**
  ```python
  modules = [
      ("reader.get", ModuleAnnotations(readonly=True, destructive=False, idempotent=True, requires_approval=False, open_world=False)),
      ("writer.delete", ModuleAnnotations(readonly=False, destructive=True, idempotent=False, requires_approval=True, open_world=True)),
      ("worker.process", None),  # No annotations
  ]
  ```
- **Steps:**
  1. Start server with 3 modules.
  2. Connect MCP client and list tools.
  3. For `reader.get`: assert `readOnlyHint=True`, `idempotentHint=True`.
  4. For `writer.delete`: assert `destructiveHint=True`.
  5. For `worker.process`: assert default annotation values.
- **Expected Result:** All annotation combinations correctly mapped per module.
- **Traceability:** F-002

---

#### TC-E2E-005: Health check endpoint on HTTP transport

- **Priority:** P2
- **Type:** E2E
- **Preconditions:** Server running with streamable-http transport, 3 modules registered
- **Test Data:** N/A
- **Steps:**
  1. Start server on streamable-http.
  2. Send GET request to `http://127.0.0.1:<port>/health`.
  3. Assert HTTP 200 response.
  4. Parse JSON body.
  5. Assert `status` field present.
  6. Assert `tools_count` equals 3.
  7. Assert `uptime_seconds` is a positive number.
- **Expected Result:** JSON body `{"status": "ok", "tools_count": 3, "uptime_seconds": <positive float>}`.
- **Traceability:** F-019

---

#### TC-E2E-006: MCP Resource exposure for documented modules

- **Priority:** P2
- **Type:** E2E
- **Preconditions:** Server with one module having `documentation` field and one without
- **Test Data:**
  ```python
  doc_module = ModuleDescriptor(module_id="api.info", description="Info", documentation="Full API documentation text here.", input_schema={}, output_schema={})
  no_doc_module = ModuleDescriptor(module_id="api.ping", description="Ping", documentation=None, input_schema={}, output_schema={})
  ```
- **Steps:**
  1. Start server with both modules.
  2. Connect MCP client and list resources.
  3. Assert resource `docs://api.info` exists with documentation text.
  4. Assert no resource for `docs://api.ping`.
- **Expected Result:** Only modules with non-empty documentation are exposed as MCP Resources.
- **Traceability:** F-020

---

#### TC-E2E-007: Claude Desktop configuration integration (manual verification)

- **Priority:** P0
- **Type:** E2E (Manual)
- **Preconditions:** Claude Desktop installed, apcore-mcp installed
- **Test Data:**
  ```json
  {
      "mcpServers": {
          "apcore-test": {
              "command": "python",
              "args": ["-m", "apcore_mcp", "--extensions-dir", "/path/to/test/extensions"]
          }
      }
  }
  ```
- **Steps:**
  1. Create test extensions directory with 2 mock modules.
  2. Add configuration to Claude Desktop `claude_desktop_config.json`.
  3. Restart Claude Desktop.
  4. Open a new conversation.
  5. Verify tools appear in Claude's tool list.
  6. Invoke a tool via natural language prompt.
  7. Verify tool result is returned correctly.
- **Expected Result:** Claude Desktop discovers and can invoke apcore modules as tools.
- **Traceability:** F-006

---

#### TC-E2E-008: Verify server with zero modules starts with warning

- **Priority:** P1
- **Type:** E2E
- **Preconditions:** Empty extensions directory
- **Test Data:** `args = ["--extensions-dir", "/tmp/empty_extensions"]`
- **Steps:**
  1. Create empty temp directory.
  2. Start server via CLI.
  3. Connect MCP client.
  4. List tools -- assert empty list.
  5. Assert server logs contain warning about no modules.
- **Expected Result:** Server starts successfully with 0 tools and logs a warning.
- **Traceability:** F-005 (AC7), F-009 (AC6)

---

## 8. Performance Test Cases (TC-PERF-xxx)

**Test File:** `tests/performance/test_benchmarks.py`

---

#### TC-PERF-001: Schema conversion for 100 modules under 100ms

- **Priority:** P0
- **Type:** Performance
- **Preconditions:** SchemaConverter, MCPServerFactory, 100 mock ModuleDescriptors
- **Test Data:** 100 ModuleDescriptors with schemas of 5-10 properties each, including 20% with $ref nodes.
- **Steps:**
  1. Create 100 ModuleDescriptor instances programmatically.
  2. Start timer.
  3. Call `factory.build_tools(registry)` where registry returns all 100.
  4. Stop timer.
  5. Assert elapsed time < 100ms.
- **Expected Result:** 100-module tool building completes in under 100ms.
- **Traceability:** F-001, F-005

---

#### TC-PERF-002: Tool call routing overhead under 5ms

- **Priority:** P0
- **Type:** Performance
- **Preconditions:** ExecutionRouter with mock Executor. `call_async` returns instantly (no-op).
- **Test Data:** `tool_name = "test.tool"`, `arguments = {"key": "value"}`
- **Steps:**
  1. Mock `executor.call_async` to return `{"result": "ok"}` with zero processing time.
  2. Run `router.handle_call(...)` 1000 times.
  3. Compute average time per call (total / 1000).
  4. Assert average overhead < 5ms.
- **Expected Result:** Average routing overhead (handle_call minus executor call) is under 5ms.
- **Traceability:** F-003

---

#### TC-PERF-003: Memory overhead under 10MB for 100 modules

- **Priority:** P0
- **Type:** Performance
- **Preconditions:** 100 ModuleDescriptors with realistic schemas (10 properties each)
- **Test Data:** 100 programmatically generated descriptors.
- **Steps:**
  1. Start `tracemalloc`.
  2. Take snapshot_before.
  3. Call `factory.build_tools(registry)` with 100 modules.
  4. Take snapshot_after.
  5. Compute memory difference.
  6. Assert difference < 10MB (10 * 1024 * 1024 bytes).
- **Expected Result:** Memory increase from building 100 tool definitions is under 10MB.
- **Traceability:** F-005

---

#### TC-PERF-004: 10 concurrent tool calls handled correctly

- **Priority:** P1
- **Type:** Performance
- **Preconditions:** ExecutionRouter with mock Executor. `call_async` has 10ms simulated delay.
- **Test Data:** 10 different tool calls with distinct arguments.
- **Steps:**
  1. Launch 10 concurrent `router.handle_call()` tasks via `asyncio.gather()`.
  2. Assert all 10 complete within 500ms (parallel, not sequential).
  3. Assert all 10 results are successful.
  4. Assert no result cross-contamination.
- **Expected Result:** All 10 calls succeed in parallel. Total time is near 10ms (parallel), not 100ms (sequential).
- **Traceability:** F-003, F-007 (AC4)

---

#### TC-PERF-005: Large schema with 50+ properties converts correctly

- **Priority:** P1
- **Type:** Performance
- **Preconditions:** SchemaConverter instance
- **Test Data:** Schema with 50 properties, 5 nested objects, 3 $ref nodes:
  ```python
  properties = {f"field_{i}": {"type": "string"} for i in range(50)}
  properties["nested_obj"] = {"$ref": "#/$defs/NestedConfig"}
  input_schema = {
      "type": "object",
      "properties": properties,
      "required": [f"field_{i}" for i in range(10)],
      "$defs": {
          "NestedConfig": {"type": "object", "properties": {"x": {"type": "integer"}, "y": {"type": "integer"}}}
      }
  }
  ```
- **Steps:**
  1. Start timer.
  2. Call `converter.convert_input_schema(descriptor)`.
  3. Stop timer.
  4. Assert elapsed time < 50ms.
  5. Assert result has 51 properties (50 + nested_obj inlined).
  6. Assert no `$defs` in result.
- **Expected Result:** Large schema converted correctly and quickly (under 50ms).
- **Traceability:** F-001

---

#### TC-PERF-006: to_openai_tools for 100 modules under 200ms

- **Priority:** P1
- **Type:** Performance
- **Preconditions:** 100 ModuleDescriptors, OpenAIConverter
- **Test Data:** 100 modules with varied schemas.
- **Steps:**
  1. Start timer.
  2. Call `to_openai_tools(registry)`.
  3. Stop timer.
  4. Assert elapsed < 200ms.
  5. Assert result has 100 items.
- **Expected Result:** 100-module OpenAI conversion under 200ms.
- **Traceability:** F-008

---

#### TC-PERF-007: Memory scales linearly with module count

- **Priority:** P2
- **Type:** Performance
- **Preconditions:** tracemalloc, varying module counts
- **Test Data:** Module counts: 10, 50, 100, 500
- **Steps:**
  1. For each count N, measure memory used by `build_tools(registry_with_N_modules)`.
  2. Compute memory per tool (total / N).
  3. Assert memory per tool is roughly constant (< 100KB per tool).
  4. Assert total for 500 modules < 50MB.
- **Expected Result:** Linear scaling. Per-tool memory is approximately constant.
- **Traceability:** F-005

---

## 9. Security Test Cases (TC-SEC-xxx)

**Test File:** `tests/security/test_security.py`

---

#### TC-SEC-001: ACL denied calls do not leak caller identity

- **Priority:** P0
- **Type:** Security
- **Preconditions:** ExecutionRouter with mock Executor raising `ACLDeniedError`
- **Test Data:**
  ```python
  error = ACLDeniedError(caller_id="admin_user_42", target_id="secret.module")
  ```
- **Steps:**
  1. Configure mock executor to raise the error.
  2. Call `router.handle_call("secret.module", {})`.
  3. Assert `result.content[0].text` does NOT contain `"admin_user_42"`.
  4. Assert text does NOT contain `"secret.module"`.
  5. Assert text equals exactly `"Access denied"`.
- **Expected Result:** No sensitive information in the error response.
- **Traceability:** F-004 (AC3)

---

#### TC-SEC-002: Unexpected exceptions do not leak stack traces

- **Priority:** P0
- **Type:** Security
- **Preconditions:** ExecutionRouter with mock Executor raising various exceptions
- **Test Data:**
  ```python
  exceptions = [
      RuntimeError("Connection refused to postgres://admin:secret@db:5432/prod"),
      FileNotFoundError("/etc/shadow"),
      PermissionError("Cannot access /root/.ssh/id_rsa"),
      ValueError("Invalid token: eyJhbGciOiJIUzI1NiJ9..."),
  ]
  ```
- **Steps:**
  1. For each exception, call `router.handle_call(...)`.
  2. Assert `result.content[0].text` equals `"Internal error occurred"`.
  3. Assert response text does NOT contain any of: `"postgres"`, `"secret"`, `"/etc/shadow"`, `"/root/.ssh"`, `"eyJhbG"`.
- **Expected Result:** All sensitive details are stripped. Only `"Internal error occurred"` returned.
- **Traceability:** F-004 (AC4)

---

#### TC-SEC-003: Malformed JSON arguments handled safely

- **Priority:** P1
- **Type:** Security
- **Preconditions:** ExecutionRouter with mock Executor that raises SchemaValidationError for bad input
- **Test Data:**
  ```python
  malformed_inputs = [
      {"width": "<script>alert(1)</script>"},
      {"width": "'; DROP TABLE users; --"},
      {"width": "{{7*7}}"},
      {"width": "\x00\x01\x02"},
  ]
  ```
- **Steps:**
  1. For each input, call `router.handle_call("image.resize", input)`.
  2. Assert either schema validation error or success (depending on type coercion).
  3. Assert no unhandled exception.
  4. Assert response does NOT echo back the malicious input unescaped.
- **Expected Result:** All malformed inputs handled gracefully. No injection or crash.
- **Traceability:** F-003, F-004

---

#### TC-SEC-004: Oversized input handled gracefully

- **Priority:** P1
- **Type:** Security
- **Preconditions:** ExecutionRouter with mock Executor
- **Test Data:**
  ```python
  arguments = {"data": "A" * 10_000_000}  # 10MB string
  ```
- **Steps:**
  1. Call `router.handle_call("data.process", arguments)`.
  2. Assert no memory exhaustion crash.
  3. Assert either success or error result returned.
- **Expected Result:** Handled without crash. Executor's validation may reject or accept.
- **Traceability:** F-003

---

#### TC-SEC-005: Error mapper never exposes call chain details

- **Priority:** P0
- **Type:** Security
- **Preconditions:** ErrorMapper
- **Test Data:**
  ```python
  errors_with_chains = [
      CallDepthExceededError(depth=10, max_depth=5, call_chain=["module.a", "module.b", "module.c", "module.d"]),
      CircularCallError(module_id="module.x", call_chain=["module.x", "module.y", "module.x"]),
      CallFrequencyExceededError(module_id="spam", count=100, max_repeat=10, call_chain=["spam", "spam"]),
  ]
  ```
- **Steps:**
  1. For each error, call `mapper.to_mcp_error(error)`.
  2. Assert result text does NOT contain `"module.a"`, `"module.b"`, `"module.x"`, `"module.y"`.
  3. Assert text does NOT contain `"call_chain"`.
- **Expected Result:** Call chain internals never exposed in MCP responses.
- **Traceability:** F-004

---

#### TC-SEC-006: Default HTTP host is localhost only

- **Priority:** P1
- **Type:** Security
- **Preconditions:** Inspect serve() and TransportManager defaults
- **Test Data:** N/A
- **Steps:**
  1. Assert `serve()` default `host` parameter is `"127.0.0.1"`.
  2. Assert `TransportManager.run_streamable_http` default `host` is `"127.0.0.1"`.
  3. Assert `TransportManager.run_sse` default `host` is `"127.0.0.1"`.
- **Expected Result:** All defaults bind to localhost only. No accidental network exposure.
- **Traceability:** F-007 (AC3)

---

## 10. Test Data Specification

### 10.1 Sample ModuleDescriptor Instances

#### Simple Descriptor (image.resize)

```python
SIMPLE_DESCRIPTOR = ModuleDescriptor(
    module_id="image.resize",
    name="Image Resize",
    description="Resize an image to the specified dimensions",
    documentation="Resizes the input image to the target width and height using bicubic interpolation.",
    input_schema={
        "type": "object",
        "title": "ImageResizeInput",
        "properties": {
            "width": {"type": "integer", "description": "Target width in pixels"},
            "height": {"type": "integer", "description": "Target height in pixels"},
            "format": {"type": "string", "default": "png", "enum": ["png", "jpg", "webp"]}
        },
        "required": ["width", "height"]
    },
    output_schema={
        "type": "object",
        "properties": {
            "status": {"type": "string"},
            "path": {"type": "string"}
        },
        "required": ["status", "path"]
    },
    version="1.0.0",
    tags=["image", "transform"],
    annotations=ModuleAnnotations(
        readonly=False,
        destructive=False,
        idempotent=True,
        requires_approval=False,
        open_world=True
    ),
    examples=[]
)
```

#### Complex Descriptor with $ref (workflow.execute)

```python
COMPLEX_DESCRIPTOR = ModuleDescriptor(
    module_id="workflow.execute",
    name="Execute Workflow",
    description="Execute a workflow with parameters",
    documentation=None,
    input_schema={
        "type": "object",
        "properties": {
            "workflow_name": {"type": "string"},
            "parameters": {"$ref": "#/$defs/WorkflowParams"}
        },
        "required": ["workflow_name", "parameters"],
        "$defs": {
            "WorkflowParams": {
                "type": "object",
                "properties": {
                    "seed": {"type": "integer", "default": 42},
                    "steps": {"type": "integer", "default": 20}
                }
            }
        }
    },
    output_schema={},
    version="2.0.0",
    tags=["workflow"],
    annotations=ModuleAnnotations(
        readonly=False,
        destructive=True,
        idempotent=False,
        requires_approval=True,
        open_world=True
    ),
    examples=[]
)
```

#### Edge Case -- Empty Schema (system.ping)

```python
EMPTY_SCHEMA_DESCRIPTOR = ModuleDescriptor(
    module_id="system.ping",
    name="System Ping",
    description="Health check endpoint",
    documentation=None,
    input_schema={},
    output_schema={"type": "object", "properties": {"status": {"type": "string"}}},
    version="1.0.0",
    tags=["system"],
    annotations=None,
    examples=[]
)
```

#### Edge Case -- Destructive with Approval (file.delete)

```python
DESTRUCTIVE_DESCRIPTOR = ModuleDescriptor(
    module_id="file.delete",
    name="File Delete",
    description="Permanently delete a file",
    documentation="Deletes the specified file from disk. This action is irreversible.",
    input_schema={
        "type": "object",
        "properties": {
            "path": {"type": "string", "description": "File path to delete"},
            "force": {"type": "boolean", "default": False}
        },
        "required": ["path"]
    },
    output_schema={"type": "object", "properties": {"deleted": {"type": "boolean"}}},
    version="1.0.0",
    tags=["file", "danger"],
    annotations=ModuleAnnotations(
        readonly=False,
        destructive=True,
        idempotent=True,
        requires_approval=True,
        open_world=False
    ),
    examples=[]
)
```

#### Edge Case -- Read-Only (data.query)

```python
READONLY_DESCRIPTOR = ModuleDescriptor(
    module_id="data.query",
    name="Data Query",
    description="Query data from the database",
    documentation=None,
    input_schema={
        "type": "object",
        "properties": {
            "table": {"type": "string"},
            "limit": {"type": "integer", "default": 100}
        },
        "required": ["table"]
    },
    output_schema={},
    version="1.0.0",
    tags=["data", "read"],
    annotations=ModuleAnnotations(
        readonly=True,
        destructive=False,
        idempotent=True,
        requires_approval=False,
        open_world=False
    ),
    examples=[]
)
```

### 10.2 Expected MCP Tool Definition (for image.resize)

```python
EXPECTED_MCP_TOOL = types.Tool(
    name="image.resize",
    description="Resize an image to the specified dimensions",
    inputSchema={
        "type": "object",
        "title": "ImageResizeInput",
        "properties": {
            "width": {"type": "integer", "description": "Target width in pixels"},
            "height": {"type": "integer", "description": "Target height in pixels"},
            "format": {"type": "string", "default": "png", "enum": ["png", "jpg", "webp"]}
        },
        "required": ["width", "height"]
    },
    annotations=ToolAnnotations(
        read_only_hint=False,
        destructive_hint=False,
        idempotent_hint=True,
        open_world_hint=True
    )
)
```

### 10.3 Expected OpenAI Tool Definition (for image.resize)

```python
EXPECTED_OPENAI_TOOL = {
    "type": "function",
    "function": {
        "name": "image-resize",
        "description": "Resize an image to the specified dimensions",
        "parameters": {
            "type": "object",
            "title": "ImageResizeInput",
            "properties": {
                "width": {"type": "integer", "description": "Target width in pixels"},
                "height": {"type": "integer", "description": "Target height in pixels"},
                "format": {"type": "string", "default": "png", "enum": ["png", "jpg", "webp"]}
            },
            "required": ["width", "height"]
        }
    }
}
```

### 10.4 Expected OpenAI Tool with strict=True (for image.resize)

```python
EXPECTED_OPENAI_STRICT_TOOL = {
    "type": "function",
    "function": {
        "name": "image-resize",
        "description": "Resize an image to the specified dimensions",
        "parameters": {
            "type": "object",
            "properties": {
                "width": {"type": "integer", "description": "Target width in pixels"},
                "height": {"type": "integer", "description": "Target height in pixels"},
                "format": {"type": ["string", "null"], "enum": ["png", "jpg", "webp", None]}
            },
            "required": ["width", "height", "format"],
            "additionalProperties": False
        },
        "strict": True
    }
}
```

---

## 11. Test Execution Plan

### Phase 1 (Week 1): Core Adapters

| Day   | Activity                                                          | Test Cases                              |
|-------|-------------------------------------------------------------------|-----------------------------------------|
| Day 1 | SchemaConverter unit tests                                        | TC-SCHEMA-001 to TC-SCHEMA-012          |
| Day 2 | AnnotationMapper unit tests                                       | TC-ANNOT-001 to TC-ANNOT-010            |
| Day 3 | ErrorMapper unit tests                                            | TC-ERROR-001 to TC-ERROR-011            |
| Day 4 | OpenAIConverter unit tests + ModuleIDNormalizer                   | TC-OPENAI-001 to TC-OPENAI-012          |
| Day 5 | Review, fix failures, achieve 90% coverage for adapters/converters| All Phase 1 tests pass                   |

**Phase 1 Exit Criteria:** All 45 adapter/converter unit tests pass. >= 90% coverage on `adapters/` and `converters/`.

### Phase 2 (Week 2): Server Components + Integration

| Day   | Activity                                                          | Test Cases                              |
|-------|-------------------------------------------------------------------|-----------------------------------------|
| Day 1 | ExecutionRouter unit tests                                        | TC-EXEC-001 to TC-EXEC-012             |
| Day 2 | MCPServerFactory unit tests                                       | TC-SERVER-001 to TC-SERVER-010          |
| Day 3 | TransportManager + CLI unit tests                                 | TC-TRANSPORT-001 to TC-TRANSPORT-009, TC-CLI-001 to TC-CLI-010 |
| Day 4 | RegistryListener unit tests                                       | TC-DYNAMIC-001 to TC-DYNAMIC-007        |
| Day 5 | Integration tests                                                 | TC-INT-001 to TC-INT-006                |

**Phase 2 Exit Criteria:** All unit + integration tests pass. >= 90% coverage on `server/`. Integration flows verified.

### Phase 3 (Week 3): E2E + Performance + Security + Polish

| Day   | Activity                                                          | Test Cases                              |
|-------|-------------------------------------------------------------------|-----------------------------------------|
| Day 1 | E2E tests (stdio, HTTP)                                           | TC-E2E-001 to TC-E2E-004               |
| Day 2 | E2E tests (dynamic, annotations, resources)                      | TC-E2E-005 to TC-E2E-008               |
| Day 3 | Performance tests                                                 | TC-PERF-001 to TC-PERF-007             |
| Day 4 | Security tests                                                    | TC-SEC-001 to TC-SEC-006                |
| Day 5 | Claude Desktop manual verification, regression, coverage report   | TC-E2E-007 (manual), full regression    |

**Phase 3 Exit Criteria:** All automated tests pass. Performance benchmarks met. Security tests pass. Manual Claude Desktop verification documented. >= 90% total line coverage.

---

## 12. Quality Gates

### 12.1 Definition of Done -- Per Phase

| Phase   | Criteria                                                                                    |
|---------|---------------------------------------------------------------------------------------------|
| Phase 1 | All TC-SCHEMA, TC-ANNOT, TC-ERROR, TC-OPENAI tests pass; >= 90% coverage on adapters/converters |
| Phase 2 | All TC-EXEC, TC-SERVER, TC-TRANSPORT, TC-CLI, TC-DYNAMIC, TC-INT tests pass; >= 90% coverage on server/ |
| Phase 3 | All TC-E2E, TC-PERF, TC-SEC tests pass; >= 90% total coverage; Claude Desktop verified     |

### 12.2 Release Criteria

1. **All P0 test cases pass** (100% pass rate).
2. **All P1 test cases pass** (>= 95% pass rate; any failures documented with workarounds).
3. **Line coverage >= 90%** on `src/apcore_mcp/` (measured by pytest-cov).
4. **All performance benchmarks met** (TC-PERF-001 through TC-PERF-005).
5. **All security tests pass** (TC-SEC-001 through TC-SEC-006).
6. **Claude Desktop integration manually verified** (TC-E2E-007).
7. **No known P0 or P1 bugs** at release time.
8. **Zero test flakiness** -- all tests pass 3 consecutive runs.

### 12.3 Regression Strategy

| Event                        | Regression Suite                                       |
|------------------------------|--------------------------------------------------------|
| Every commit                 | Unit tests (fast, < 30 seconds)                        |
| Every pull request           | Unit + Integration + Security (< 3 minutes)           |
| Nightly CI                   | Full suite: Unit + Integration + E2E + Perf + Security |
| Dependency update (apcore)   | Full suite + manual MCP client check                   |
| Dependency update (mcp SDK)  | Full suite + manual Claude Desktop check               |
| Pre-release                  | Full suite 3x + Claude Desktop manual verification     |

---

## 13. Appendix

### 13.1 Glossary

| Term                    | Definition                                                          |
|-------------------------|---------------------------------------------------------------------|
| TDD                     | Test-Driven Development -- tests written before implementation      |
| MCP                     | Model Context Protocol -- Anthropic's tool integration protocol     |
| SUT                     | System Under Test                                                   |
| Mock                    | Test double that simulates behavior of a dependency                 |
| Fixture                 | Reusable test setup/teardown code managed by pytest                 |
| P0/P1/P2                | Priority tiers: P0 = must have, P1 = should have, P2 = nice to have|
| Coverage                | Percentage of source code lines executed during test runs           |
| Flaky test              | Test that intermittently passes/fails without code changes          |
| Regression              | Re-running existing tests to detect newly introduced bugs           |

### 13.2 Test Case ID Naming Convention

```
TC-<COMPONENT>-<NNN>

Components:
  SCHEMA    = Schema Converter (adapters/schema.py)
  ANNOT     = Annotation Mapper (adapters/annotations.py)
  EXEC      = Execution Router (server/router.py)
  ERROR     = Error Mapper (adapters/errors.py)
  SERVER    = MCP Server Factory (server/factory.py)
  OPENAI    = OpenAI Converter (converters/openai.py)
  TRANSPORT = Transport Manager (server/transport.py)
  CLI       = CLI Module (__main__.py)
  DYNAMIC   = Dynamic Registry Listener (server/listener.py)
  INT       = Integration tests
  E2E       = End-to-end tests
  PERF      = Performance tests
  SEC       = Security tests

NNN = Zero-padded sequential number (001, 002, ...)
```

### 13.3 References

| Reference                                | Description                                            |
|------------------------------------------|--------------------------------------------------------|
| `docs/prd-apcore-mcp.md`                | Product Requirements Document v1.0                     |
| `docs/tech-design-apcore-mcp.md`        | Technical Design Document v1.0                         |
| IEEE 829                                 | Standard for Software Test Documentation               |
| ISTQB Foundation Level Syllabus          | International Software Testing Qualifications Board    |
| Google Testing Blog                      | Best practices for test strategy and test pyramid      |
| pytest documentation                     | https://docs.pytest.org/                               |
| pytest-asyncio documentation             | https://pytest-asyncio.readthedocs.io/                 |
| MCP Specification                        | https://modelcontextprotocol.io/                       |
| OpenAI Function Calling Docs             | https://platform.openai.com/docs/guides/function-calling|

---

*End of Test Plan & Test Cases Document*
