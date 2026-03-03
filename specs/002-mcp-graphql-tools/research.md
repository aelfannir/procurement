# Research: MCP Data Tools via GraphQL

**Feature**: 002-mcp-graphql-tools
**Date**: 2026-02-24

## R1: GraphQL Tool Execution Method

**Decision**: Call the GraphQL executor directly in PHP via injected services.

**Rationale**: API Platform exposes two injectable services:
- `api_platform.graphql.executor` (`ExecutorInterface`) — wraps `graphql-php`'s `GraphQL::executeQuery()`
- `api_platform.graphql.schema_builder` (`SchemaBuilderInterface`) — builds the GraphQL schema from AP metadata

The MCP processor can inject both and execute queries with zero HTTP overhead:

```php
$schema = $this->schemaBuilder->getSchema();
$result = $this->executor->executeQuery($schema, $query, null, null, $variables);
return $result->toArray();
```

The executor returns `GraphQL\Executor\ExecutionResult` with `$data` and `$errors` arrays — perfect for returning as MCP tool output.

**Alternatives considered**:
- Internal HTTP call to `/graphql`: Works but adds unnecessary HTTP overhead (request serialization, routing, response parsing). No benefit since the executor is directly injectable.

**Configuration available on executor**:
- `max_query_complexity`: Default 500
- `max_query_depth`: Default 20
- `introspection.enabled`: Default true

---

## R2: AEFilter + GraphQL Compatibility

**Decision**: AEFilter works with GraphQL out of the box. No changes needed.

**Rationale**: AEFilter extends `ApiPlatform\Doctrine\Orm\Filter\AbstractFilter` (standard Doctrine ORM filter). GraphQL's `FieldsBuilder` converts `operation->getFilters()` (ApiFilter list) into GraphQL input args. At query time, GraphQL's `ReadProvider` converts args to `context['filters']`, then Doctrine's `FilterExtension` applies them — identical to REST.

**AEFilter capabilities available in GraphQL**:
- PropertyFilter: `{ property, operator, value }` with operators: EQUALS, NOT_EQUALS, GT, GTE, LT, LTE, IN, NOT_IN, IS_NULL, IS_NOT_NULL, CONTAINS, STARTS_WITH, ENDS_WITH
- CompoundFilter: `{ operator: AND|OR, filters: [...] }` (recursive nesting)
- Nested property filters via LEFT JOIN (e.g., `clinic.abbreviation`)
- Property value substitution: values wrapped in `{propertyName}` reference other entity properties

**Important**: All 9 core entities already have `#[ApiFilter(AEFilter::class)]` and `#[ApiFilter(OrderFilter::class)]` — these will automatically become available as GraphQL query arguments.

---

## R3: QueryParameter + GraphQL Compatibility

**Decision**: Needs empirical testing. Code analysis and documentation are contradictory.

**Code analysis findings**: QueryParameter appears HTTP-REST only:
- `ParameterExtension` (which processes QueryParameter) only runs in the HTTP Doctrine ORM flow
- GraphQL's `ReadProvider` does NOT process `operation->getParameters()` — it only converts GraphQL args to filters
- Zero references to QueryParameter in `vendor/api-platform/core/src/GraphQl/`

**Documentation findings**: API Platform docs state:
- "When parameters are enabled, they automatically appear in the Hydra, OpenAPI and GraphQL documentations"
- "It is strongly recommended to configure your filters via the parameters attribute using QueryParameter"

**Empirical test**: Pending. Need to add `#[Query]`/`#[QueryCollection]` with QueryParameter to a test entity and verify if filtering works at query execution time (not just schema generation).

**Current working approach**: Entities with `#[ApiFilter(AEFilter::class)]` and `#[ApiResource]` attribute DO get `filter` and `sort` args in GraphQL and they work at execution time. Verified empirically with PaymentModality (78 records returned via cursor pagination).

**Key discovery**: Core entities (PurchaseOrder, Receipt, etc.) do NOT have `#[ApiResource]` — they use individual operation attributes only. This is why they don't auto-generate GraphQL operations. Adding explicit `#[Query]`/`#[QueryCollection]` attributes is required.

---

## R3b: GraphQL Auto-Generation Requires #[ApiResource]

**Decision**: Entities without `#[ApiResource]` need explicit GraphQL attributes.

**Rationale**: Empirical testing revealed:
- Entities WITH `#[ApiResource]` (PaymentModality, User) → auto-generate GraphQL Query + QueryCollection
- Entities WITHOUT `#[ApiResource]` (PurchaseOrder, Receipt, Vendor, Product, all 9 core entities) → only appear as single-item queries in GraphQL (from Get operations), no collection queries

**Impact**: Adding `#[Query]` and `#[QueryCollection]` to each core entity is the correct approach — consistent with spec FR-005/FR-006. The `#[ApiFilter]` on these entities should automatically generate `filter`/`sort` args once the GraphQL operations exist.

---

## R4: Serialization Groups Strategy

**Decision**: Use default serialization (no explicit normalizationContext) for GraphQL Query/QueryCollection, matching the existing REST Get/GetCollection behavior.

**Rationale**: Investigation of the 9 core entities shows that most Get/GetCollection operations use no explicit normalizationContext — they rely on default serialization. Only specialized operations (print, export, update, duplicate) define explicit groups. Following the same pattern for GraphQL means:
- Maximum field exposure for AI flexibility (FR-013)
- Consistent behavior between REST and GraphQL reads
- No new serialization groups to create or maintain

**Sensitive field analysis**:
- `User::$password` — No #[Groups] assigned, correctly excluded from all serialization
- `User::$plainPassword` — Only in denormalization groups (input), never in normalization (output)
- No API keys, tokens, or secrets found in the 9 core entities
- No sensitive data exposure risk with default serialization

**If granular control is needed later**: Define GraphQL-specific normalizationContext on the Query/QueryCollection attributes. This can be done per-entity without affecting REST.

---

## R5: Security Expressions for GraphQL Operations

**Decision**: Start with minimal security on GraphQL operations (matching REST), add workspace-level security via existing mechanisms.

**Rationale**: Current REST entities have minimal explicit security:
- Only `RegularInvoice` has security expressions: `security: "object.isAccounted() === false"` on Put and Delete (business rule, not access control)
- No Get/GetCollection operations have security expressions
- Access control is handled at the firewall/route level, not per-operation

For GraphQL:
- Query and QueryCollection attributes can start without explicit `security:` — matching REST behavior
- The GraphQL endpoint (`/graphql`) will be behind the same firewall as REST
- Workspace/clinic filtering is handled by Doctrine query extensions (applied automatically to both REST and GraphQL)
- Association-level security (`ApiProperty(security: ...)`) can be added later for traversal protection

**Constitution VI (Multi-Workspace Awareness)**: VERIFIED — workspace constraints are enforced via Doctrine extensions that apply to all query paths, including GraphQL. No additional configuration needed.

---

## R6: GraphQL Configuration Requirements

**Decision**: Enable GraphQL in `api_platform.yaml` with sensible defaults.

**Configuration needed**:
```yaml
api_platform:
    graphql:
        enabled: true
        nesting_separator: '__'       # Avoid underscore ambiguity
        max_query_depth: 20           # Default, sufficient for nested queries
        max_query_complexity: 500     # Default, adjustable if needed
        introspection:
            enabled: true             # Required for AI schema discovery
```

**GraphQL route**: Automatically registered at `/graphql` when enabled. The `graphql.php` routing file in AP vendor handles this.

**Dependency**: `webonyx/graphql-php` ^15.22 is already in composer.json — no new packages needed.
