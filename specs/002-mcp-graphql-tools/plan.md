# Implementation Plan: MCP Data Tools via GraphQL

**Branch**: `002-mcp-graphql-tools` | **Date**: 2026-02-24 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/002-mcp-graphql-tools/spec.md`

## Summary

Enable AI agents to flexibly query all procurement entities through a single GraphQL MCP tool. API Platform's built-in GraphQL support auto-exposes entities with Query/QueryCollection attributes. One MCP tool (`graphql_query`) accepts any GraphQL query and returns results — covering search, filtering, pagination, nested relations, and introspection. Initial rollout covers 9 core entities with read-only operations; mutations deferred to a future phase.

## Technical Context

**Language/Version**: PHP 8.2 / Symfony 7.x / API Platform 4.x (pre-release MCP support)
**Primary Dependencies**: API Platform GraphQL (`webonyx/graphql-php` ^15.22 already installed), API Platform MCP (`McpTool`)
**Storage**: PostgreSQL (existing, no changes)
**Testing**: PHPUnit with API Platform's ApiTestCase, run via `docker compose exec php php bin/phpunit`
**Target Platform**: Linux server (Docker/FrankenPHP)
**Project Type**: Single (API backend)
**Performance Goals**: GraphQL queries should return within acceptable response times for AI tool usage (< 5s for typical queries)
**Constraints**: Must not break existing REST API behavior, must enforce same security rules as REST
**Scale/Scope**: 9 entities in initial rollout, expandable to 40+ entities by adding attributes

### Research Outcomes (resolved)

All research items resolved in [research.md](research.md):

1. **GraphQL tool execution method** → Direct PHP call via `ExecutorInterface` + `SchemaBuilderInterface` (no HTTP overhead)
2. **AEFilter + GraphQL** → Works out of the box. Entities with `#[ApiFilter(AEFilter::class)]` auto-generate `filter` and `sort` args in GraphQL. Verified empirically with PaymentModality.
3. **QueryParameter + GraphQL** → Contradictory findings between docs and code. Code analysis says HTTP-only; docs say it appears in GraphQL. Needs empirical test during implementation. Fallback: existing `#[ApiFilter]` already works.
4. **Serialization groups** → Use default normalization (no explicit groups), matching existing REST Get/GetCollection behavior. Sensitive fields (passwords) are already excluded.
5. **Security expressions** → Start without explicit security on Query/QueryCollection (matching REST). Firewall-level auth + Doctrine query extensions enforce access control.
6. **Key discovery** → Core entities lack `#[ApiResource]` attribute — they use individual operation attributes only. Explicit `#[Query]`/`#[QueryCollection]` attributes are required for GraphQL.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Pre-Research Check

| Principle | Status | Notes |
|-----------|--------|-------|
| **I. API Platform First** | PASS | GraphQL is a native API Platform feature. MCP tool wraps AP's GraphQL executor. No custom controllers or query builders. |
| **II. Domain-Grouped Organization** | PASS | No new domain grouping needed — entities keep their existing domain structure. One cross-cutting MCP tool serves all domains. |
| **III. Enum as Source of Truth** | PASS | GraphQL schema auto-generates from entity properties including enums. No hardcoded values. |
| **IV. YAGNI** | PASS | One tool instead of 80+. Only read operations for now. Mutations deferred. Initial rollout limited to 9 entities. |
| **V. Lean MCP Layer** | PASS | Adding 1 tool (total ~4 with existing). Well within ~25-30 budget. Reuses AP's existing providers via GraphQL. |
| **VI. Multi-Workspace Awareness** | NEEDS VERIFICATION | GraphQL security must enforce workspace constraints. |

### Post-Design Re-Check

| Principle | Status | Notes |
|-----------|--------|-------|
| **I. API Platform First** | PASS | Uses AP's `ExecutorInterface`, `SchemaBuilderInterface`, `#[Query]`, `#[QueryCollection]`, and existing `#[ApiFilter]`. Zero custom query builders. |
| **II. Domain-Grouped Organization** | PASS | No structural changes to domain organization. |
| **III. Enum as Source of Truth** | PASS | GraphQL schema reflects entity properties including PHP-backed enums dynamically. |
| **IV. YAGNI** | PASS | 1 new MCP tool + attribute additions to 9 entities. No new abstractions. |
| **V. Lean MCP Layer** | PASS | Total tools: 4 (get_app_context, search_products, get_purchase_order_statistics, graphql_query). |
| **VI. Multi-Workspace Awareness** | PASS | Verified: Doctrine query extensions apply to all collection providers including GraphQL. Workspace/clinic filtering is automatic. GraphQL firewall uses same ApiKeyAuthenticator as MCP. |

**Gate Result**: PASS

## Project Structure

### Documentation (this feature)

```text
specs/002-mcp-graphql-tools/
├── plan.md              # This file
├── research.md          # Phase 0 output ✓
├── data-model.md        # Phase 1 output ✓
├── quickstart.md        # Phase 1 output ✓
├── contracts/           # Phase 1 output ✓
│   └── graphql-schema.md
└── tasks.md             # Phase 2 output (/speckit.tasks)
```

### Source Code (repository root)

```text
src/
├── ApiResource/
│   ├── GraphqlQuery.php          # MCP tool definition (McpTool attribute)
│   └── GraphqlSchema.php         # MCP tool: serves domain-scoped SDL schema at runtime
├── Dto/
│   └── GraphqlQueryInput.php     # Input DTO for the MCP tool
├── State/
│   └── GraphqlQueryProcessor.php # Processor: executes GraphQL via AP executor
├── Entity/
│   ├── RegularPurchaseRequest.php  # + #[Query] #[QueryCollection]
│   ├── ServiceRegularizationRequest.php  # + #[Query] #[QueryCollection]
│   ├── PurchaseOrder.php           # + #[Query] #[QueryCollection]
│   ├── Receipt.php                 # + #[Query] #[QueryCollection]
│   ├── RegularInvoice.php          # + #[Query] #[QueryCollection]
│   ├── CreditNote.php              # + #[Query] #[QueryCollection]
│   ├── ReturnOrder.php             # + #[Query] #[QueryCollection]
│   ├── Vendor.php                  # + #[Query] #[QueryCollection]
│   └── Product.php                 # + #[Query] #[QueryCollection]
└── ...

config/
└── packages/
    ├── api_platform.yaml         # + graphql section
    └── security.yaml             # + graphql firewall (already added)

```

**Structure Decision**: Follows existing project structure. MCP tool in `src/ApiResource/`, processor in `src/State/`, input DTO in `src/Dto/`. GraphQL config in existing `api_platform.yaml`. Entity modifications are attribute additions only.

## Complexity Tracking

No constitution violations to justify.
