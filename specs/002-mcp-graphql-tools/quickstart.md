# Quickstart: MCP Data Tools via GraphQL

**Feature**: 002-mcp-graphql-tools
**Date**: 2026-02-24

## Prerequisites

- Docker environment running (`docker compose up -d`)
- `webonyx/graphql-php` already installed (composer.json)
- Existing 8 MCP resources + 3 MCP tools on branch `001-mcp-ai-resources`

## Implementation Steps

### Step 1: Enable GraphQL in API Platform config

Add GraphQL section to `config/packages/api_platform.yaml`:
```yaml
api_platform:
    graphql:
        enabled: true
        nesting_separator: '__'
        introspection:
            enabled: true
```

### Step 2: Add GraphQL firewall (already done during research)

In `config/packages/security.yaml`, the `graphql` firewall with `ApiKeyAuthenticator` was added before the catch-all `api` firewall.

### Step 3: Add Query/QueryCollection attributes to 9 core entities

For each entity, add:
```php
use ApiPlatform\Metadata\GraphQl\Query;
use ApiPlatform\Metadata\GraphQl\QueryCollection;

#[Query]
#[QueryCollection]
class PurchaseOrder { ... }
```

Since these entities lack `#[ApiResource]`, the explicit GraphQL attributes are required.

### Step 4: Create the MCP tool

- `src/ApiResource/GraphqlQuery.php` — McpTool definition
- `src/Dto/GraphqlQueryInput.php` — Input DTO (query + variables)
- `src/State/GraphqlQueryProcessor.php` — Injects `ExecutorInterface` + `SchemaBuilderInterface`, calls `executeQuery()` directly

### Step 5: Test

```bash
# Test GraphQL endpoint directly
docker compose exec php curl -s -X POST http://localhost/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <api-key>" \
  -d '{"query":"{ purchaseOrders(first: 5) { totalCount edges { node { id ref status } } } }"}'

# Test MCP tool via MCP protocol
# (use Claude Desktop or MCP Inspector connected to /mcp endpoint)

# Run PHPUnit tests
docker compose exec php php bin/phpunit tests/Api/GraphqlQueryToolTest.php
```

## Key Architecture Decisions

1. **Direct PHP execution** — MCP processor calls `ExecutorInterface::executeQuery()` directly, no HTTP overhead
2. **Existing filters auto-work** — `#[ApiFilter(AEFilter::class)]` generates `filter` and `sort` args in GraphQL
3. **No new serialization groups** — GraphQL uses default normalization (same as REST Get/GetCollection)
4. **Cursor-based pagination** — API Platform's default for GraphQL (Relay Connection spec)
5. **IRI identifiers** — Single-item queries use IRIs (`/purchase-orders/123`), matching JSON-LD convention

## Files to Create/Modify

| Action | File | Description |
|--------|------|-------------|
| Modify | `config/packages/api_platform.yaml` | Add GraphQL config |
| Done | `config/packages/security.yaml` | GraphQL firewall (already added) |
| Modify | `src/Entity/PurchaseOrder.php` | Add Query + QueryCollection |
| Modify | `src/Entity/RegularPurchaseRequest.php` | Add Query + QueryCollection |
| Modify | `src/Entity/ServiceRegularizationRequest.php` | Add Query + QueryCollection |
| Modify | `src/Entity/Receipt.php` | Add Query + QueryCollection |
| Modify | `src/Entity/RegularInvoice.php` | Add Query + QueryCollection |
| Modify | `src/Entity/CreditNote.php` | Add Query + QueryCollection |
| Modify | `src/Entity/ReturnOrder.php` | Add Query + QueryCollection |
| Modify | `src/Entity/Vendor.php` | Add Query + QueryCollection |
| Modify | `src/Entity/Product.php` | Add Query + QueryCollection |
| Create | `src/ApiResource/GraphqlQuery.php` | MCP tool definition |
| Create | `src/Dto/GraphqlQueryInput.php` | Input DTO |
| Create | `src/State/GraphqlQueryProcessor.php` | Processor |
| Create | `tests/Api/GraphqlQueryToolTest.php` | Integration tests |
