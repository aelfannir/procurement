# GraphQL Schema Contract

**Feature**: 002-mcp-graphql-tools
**Date**: 2026-02-24

## MCP Tool Contract

### Tool: `graphql_query`

**Description**: Execute a GraphQL query against the procurement API. Supports queries (reads), introspection, and cursor-based pagination. Use the documentation resources to understand the domain, then use introspection to discover exact field names and filter syntax.

**Input Schema**:
```json
{
  "type": "object",
  "properties": {
    "query": {
      "type": "string",
      "description": "GraphQL query string. Supports queries and introspection."
    },
    "variables": {
      "type": "string",
      "description": "Optional JSON-encoded variables map for parameterized queries."
    }
  },
  "required": ["query"]
}
```

**Output**: Raw GraphQL response (JSON with `data` and/or `errors` fields).

**MCP Annotations**:
- `readOnlyHint`: false (will support mutations in future)
- `idempotentHint`: true
- `openWorldHint`: false

---

## Expected GraphQL Queries (examples for AI)

### Collection query with filter and pagination
```graphql
{
  purchaseOrders(
    filter: [{ property: "status", operator: "EQUALS", value: "validating" }]
    sort: [{ createdAt: "DESC" }]
    first: 10
  ) {
    totalCount
    pageInfo {
      endCursor
      hasNextPage
    }
    edges {
      node {
        id
        ref
        status
        totalHt
        totalTtc
        vendor {
          name
        }
        clinic {
          abbreviation
        }
      }
    }
  }
}
```

### Single item query
```graphql
{
  purchaseOrder(id: "/purchase-orders/123") {
    id
    ref
    status
    createdAt
    totalHt
    totalTtc
    vendor {
      name
      addresses {
        edges {
          node {
            city
            address
          }
        }
      }
    }
    purchaseOrderProducts {
      edges {
        node {
          quantity
          unitPrice
          product {
            code
            designation
          }
        }
      }
    }
  }
}
```

### Introspection query
```graphql
{
  __schema {
    queryType {
      fields {
        name
        args {
          name
          type { name kind }
        }
      }
    }
  }
}
```

### Type introspection
```graphql
{
  __type(name: "PurchaseOrder") {
    name
    fields {
      name
      type { name kind ofType { name } }
    }
  }
}
```

## Error Response Contract

GraphQL errors follow the standard format:
```json
{
  "errors": [
    {
      "message": "Cannot query field \"nonexistent\" on type \"PurchaseOrder\".",
      "locations": [{ "line": 3, "column": 5 }],
      "path": ["purchaseOrder"]
    }
  ]
}
```
