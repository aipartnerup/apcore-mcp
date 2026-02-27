# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.4.0] - 2026-02-27

### Added
- JWT Authentication feature specification (F-027) across PRD, SRS, tech design, and test plan
- `Authenticator` protocol for pluggable authentication backends
- `JWTAuthenticator` with `ClaimMapping` for JWT Bearer token validation
- `AuthMiddleware` ASGI middleware with `ContextVar` bridge for Identity injection
- `auth/` package added to package structure (protocol.py, jwt.py, middleware.py)
- CLI flags: `--jwt-secret`, `--jwt-algorithm`, `--jwt-audience`, `--jwt-issuer`
- `PyJWT>=2.0` added as required dependency
- 47 test cases specified: 22 unit (JWT), 12 unit (middleware), 13 integration

### Changed
- Updated NG-06 non-goal: from "no auth" to "no OAuth/API key management" (JWT bridges to ACL)
- Updated C-05 constraint: authentication bridges to apcore ACL, not replaces it
- Updated security threat model with JWT-specific entries
- Updated `serve()` API with `authenticator` parameter

## [0.3.0] - 2026-02-25

### Added
- MCP Tool Explorer feature for browsing and testing tools (PRD, SRS, test plan)
- Examples specification for apcore-mcp cross-language standard (`docs/examples-spec.md`)
- Tool Explorer UI screenshot (`apcore-mcp-explorer-ui.png`)

### Changed
- Updated response key from `output` to `result` in SRS and test plan

## [0.2.0] - 2026-02-23

### Added
- Documentation deployment workflow via GitHub Actions
- Project logo (`apcore-mcp-logo.svg`)
- MkDocs configuration for documentation site (`mkdocs.yml`)
- Streaming bridge specification added to SRS and technical design documents
- Background context and Python package availability notes in README

### Changed
- Expanded PRD to cover 25 features with detailed metrics and validation parameters
- Updated SRS to reflect increased feature count and deferred feature notes
- Revised technical design documentation to align with updated feature set and naming conventions
- Updated test plan to validate new naming conventions (module IDs using hyphens instead of double underscores)
- Enhanced test cases to ensure compliance with updated API specifications
- Updated README with TypeScript repository link and current package status

## [0.1.0] - 2026-02-15

### Added
- Initial project setup for apcore-mcp
- Core concept: automatic bridging of apcore modules to MCP Server and OpenAI Tools
- Product Requirements Document (PRD)
- Software Requirements Specification (SRS)
- Technical Design Document
- Test Plan
- Initial README
