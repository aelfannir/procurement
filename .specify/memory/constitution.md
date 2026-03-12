<!--
Sync Impact Report
===================
Version change: 1.0.0 → 1.0.1
Modified principles: None
Added sections: None
Removed sections: None
Corrections:
  - Branch naming: "NNN-short-name" → "ACHAT-XXX-short-name" (align
    with CLAUDE.md Jira-key convention)
  - Frontend stack: "React" → "React 19" (match CLAUDE.md precision)
Templates requiring updates:
  - .specify/templates/plan-template.md ✅ no changes needed
  - .specify/templates/spec-template.md ✅ no changes needed
  - .specify/templates/tasks-template.md ✅ no changes needed
Follow-up TODOs: None
-->

# APP (AKDITAL Purchasing & Procurement) Constitution — Root Workspace

## Scope

This workspace encompasses both sub-projects:
- **procurement-api/** — Backend (PHP 8.2 / Symfony 7.x / API Platform 4.x / PostgreSQL)
- **procurement-app/** — Frontend (React / TypeScript / Vite)

All specs (API-only, frontend-only, or cross-cutting) live here. Each spec declares a Target field: `api`, `app`, or `both`.

## Core Principles

### I. API Platform First

All API endpoints, MCP tools, and MCP resources MUST be built using
API Platform's native features (attributes, providers, processors,
filters, serialization groups). Custom query builders, controllers,
and manual JSON responses are prohibited unless API Platform provides
no viable alternative. Justification MUST be documented in the spec's
research.md when bypassing API Platform.

### II. Domain-Grouped Organization

Features MUST be organized by business domain (purchase-requests,
purchase-orders, receipts, returns, validation, organization,
products-vendors), not by technical layer. MCP tools and resources
follow the same domain grouping. Each domain artifact MUST be
self-contained — no cross-resource dependencies for basic
comprehension.

### III. Enum as Source of Truth

All status values, type lists, and categorical data MUST be defined
as PHP backed enums in `procurement-api/src/Entity/Enumeration/`. Any surface that
exposes these values (MCP resources, API responses, documentation)
MUST generate them dynamically from `::cases()` at runtime. Hardcoded
enum values in markdown, JSON, or templates are prohibited.

### IV. YAGNI

Every addition MUST serve a current, concrete requirement. No
speculative abstractions, premature generalizations, or "just in case"
code. Three similar lines of code are preferable to a premature
abstraction. Features, helpers, and configuration options MUST NOT be
added until explicitly needed. If removing code is simpler than
maintaining it, remove it.

### V. Lean MCP Layer

MCP integration MUST stay lean: ~25-30 tools organized by business
domain, not one tool per API route. MCP resources provide static
reference documentation; MCP tools provide model-controlled actions
that query or mutate data. Tools MUST reuse API Platform's existing
providers and filters — no parallel data access layer.

### VI. Multi-Workspace Awareness

All features MUST respect the workspace model: Project (no purchase
requests), Exploitation (full procurement flow), Administration
(global config). Clinic status determines workspace assignment.
Features MUST NOT assume a single workspace or bypass workspace
constraints. Validation paths, available operations, and document
flows are workspace-dependent.

## Technical Constraints

### Backend (procurement-api/)
- **Stack**: PHP 8.2, Symfony 7.x, API Platform 4.x, PostgreSQL
- **Runtime**: Docker (FrankenPHP), no local PHP assumed
- **Testing**: PHPUnit with API Platform's ApiTestCase, run via
  `docker compose exec php php bin/phpunit`
- **MCP transport**: Streamable HTTP at `/mcp` endpoint
- **MCP resource pattern**: `ReadResourceResult` + `structuredContent: false`
  (required due to API Platform StructuredContentProcessor bug)
- **MCP tool input**: Use separate input DTO class via `input:` option;
  enum types must be `string` (ObjectMapper doesn't deserialize enums)

### Frontend (procurement-app/)
- **Stack**: React 19, TypeScript, Vite
- **Package manager**: pnpm
- **Testing**: Run via `pnpm test`

### Shared
- **Branching**: Feature branches target `dev`, never `main`
- **Branch naming**: `ACHAT-XXX-short-name` (Jira key + kebab-case)

## Development Workflow

- Cross-cutting feature specs live in `specs/NNN-feature-name/` with
  spec.md, plan.md, research.md, and tasks.md
- Speckit workflow: specify → plan → tasks → implement
- Tasks use checkbox format: `- [ ] T001 [US1] Description`
- Tasks MUST clearly indicate which sub-project they target:
  `- [ ] T001 [US1] [API] Description` or `- [ ] T002 [US1] [APP] Description`
- All PRs MUST have passing tests before merge
- Specs MUST document research decisions with rationale and
  alternatives considered

## Governance

This constitution supersedes conflicting practices in specs, plans,
and implementation. Per-project constitutions remain authoritative
for single-project specs. This root constitution governs cross-cutting
specs only.

Amendments require:

1. A documented rationale for the change
2. Version bump following semver (MAJOR for principle removal/redefinition,
   MINOR for additions, PATCH for clarifications)
3. Sync Impact Report updated in this file's HTML comment header

All specs MUST include a Constitution Check section in plan.md
verifying alignment with these principles. Violations MUST be
justified in research.md or rejected.

Use `CLAUDE.md` for runtime development guidance and
`.specify/memory/constitution.md` for governance principles.

**Version**: 1.0.1 | **Ratified**: 2026-03-02 | **Last Amended**: 2026-03-12
