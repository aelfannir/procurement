# Data Model: MCP Data Tools via GraphQL

**Feature**: 002-mcp-graphql-tools
**Date**: 2026-02-24

## Overview

This feature does not introduce new entities. It adds GraphQL read operations (`#[Query]`, `#[QueryCollection]`) to existing entities and creates one new MCP tool (`GraphqlQuery`) with its input DTO.

## New MCP Components

### GraphqlQuery (MCP Tool)

The single MCP tool that accepts and executes GraphQL queries.

| Field | Type | Description |
|-------|------|-------------|
| query | string | GraphQL query string (required) |

**MCP annotations**: `readOnlyHint: false` (will support mutations later), `idempotentHint: true` (queries are idempotent), `openWorldHint: false`

**Tool description** (guided): Includes available entity names, AEFilter syntax (property/operator/value, compound AND/OR), and pagination pattern (first/after). Enough for the AI to construct queries without introspection for common cases.

### GraphqlQueryInput (Input DTO)

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| query | string | yes | The GraphQL query to execute |

### GraphqlSchema (MCP Tool)

Domain-scoped GraphQL schema tool. Returns SDL for a specific business domain topic.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| topic | string | yes | Domain topic: purchase-requests, purchase-orders, receipts-and-invoices, return-orders, products-and-vendors, organization, all |

**MCP annotations**: `readOnlyHint: true`, `idempotentHint: true`, `openWorldHint: false`

**Domain → Type mapping**:

| Topic | GraphQL Types |
|-------|---------------|
| purchase-requests | RegularPurchaseRequest, ServiceRegularizationRequest, PurchaseRequestProduct |
| purchase-orders | PurchaseOrder, PurchaseOrderProduct |
| receipts-and-invoices | Receipt, ReceiptProduct, RegularInvoice, CreditNote |
| return-orders | ReturnOrder, ReturnOrderProduct |
| products-and-vendors | Product, Vendor, VendorAddress |
| organization | Clinic, Country, City, Budget |
| all | Full SDL (all types) |

Injects `SchemaBuilderInterface`, generates full SDL, then filters to return only types matching the requested topic. Zero drift when entities change.

## Entities Receiving GraphQL Attributes (Initial Rollout)

No schema changes — only new attributes on existing entity classes.

### Entities to Add #[Query] + #[QueryCollection]

| Entity | GraphQL Item Query | GraphQL Collection Query | Existing Filters |
|--------|-------------------|-------------------------|------------------|
| PurchaseOrder | `purchaseOrder(id:)` | `purchaseOrders(filter:, sort:, first:, after:)` | AEFilter, OrderFilter |
| RegularPurchaseRequest | `regularPurchaseRequest(id:)` | `regularPurchaseRequests(filter:, sort:, first:, after:)` | AEFilter, OrderFilter |
| ServiceRegularizationRequest | `serviceRegularizationRequest(id:)` | `serviceRegularizationRequests(filter:, sort:, first:, after:)` | AEFilter, OrderFilter |
| Receipt | `receipt(id:)` | `receipts(filter:, sort:, first:, after:)` | AEFilter, OrderFilter |
| RegularInvoice | `regularInvoice(id:)` | `regularInvoices(filter:, sort:, first:, after:)` | AEFilter, OrderFilter |
| CreditNote | `creditNote(id:)` | `creditNotes(filter:, sort:, first:, after:)` | AEFilter, OrderFilter |
| ReturnOrder | `returnOrder(id:)` | `returnOrders(filter:, sort:, first:, after:)` | AEFilter, OrderFilter |
| Vendor | `vendor(id:)` | `vendors(filter:, sort:, first:, after:)` | AEFilter, OrderFilter |
| Product | `product(id:)` | `products(filter:, sort:, first:, after:)` | AEFilter, OrderFilter |

### Filter Arguments (auto-generated from existing #[ApiFilter])

- `filter`: Iterable — passes through to AEFilter (Airtable-style compound/nested filters)
  - PropertyFilter: `{ property, operator, value }` with operators: EQUALS, NOT_EQUALS, GT, GTE, LT, LTE, IN, NOT_IN, IS_NULL, IS_NOT_NULL, CONTAINS, STARTS_WITH, ENDS_WITH
  - CompoundFilter: `{ operator: AND|OR, filters: [...] }` (recursive)
- `sort`: List of `{ field: "ASC"|"DESC" }` — passes through to OrderFilter

### Pagination (cursor-based, auto-generated)

All collection queries support:
- `first` / `after` — forward pagination
- `last` / `before` — backward pagination
- Response includes `totalCount`, `pageInfo { startCursor, endCursor, hasNextPage, hasPreviousPage }`, `edges { cursor, node { ... } }`

## Configuration Changes

### api_platform.yaml additions

```yaml
api_platform:
    graphql:
        enabled: true
        nesting_separator: '__'
        introspection:
            enabled: true
```

### security.yaml additions

```yaml
firewalls:
    graphql:
        pattern: ^/graphql
        stateless: true
        custom_authenticators:
            - App\Security\ApiKeyAuthenticator
```

(Already added during research testing)
