# Feature Specification: MCP Data Tools via GraphQL

**Feature Branch**: `002-mcp-graphql-tools`
**Created**: 2026-02-24
**Status**: Draft
**Target**: api
**Input**: User description: "MCP data tools via GraphQL — enable AI to flexibly query all procurement entities through a single GraphQL MCP tool"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - AI Agent Queries Any Entity Flexibly (Priority: P1)

An AI agent uses a single GraphQL MCP tool to query any procurement entity — products, vendors, purchase orders, purchase requests, receipts, invoices, return orders, clinics, and more. The agent constructs GraphQL queries selecting exactly the fields and relations it needs, applying filters, and controlling pagination. It answers natural language questions like "Show me all pending purchase orders for Clinic Casablanca" or "Which vendors have the most unpaid invoices?"

**Why this priority**: Flexible data access is the foundation for all AI interactions. One tool covering all entities means the AI can answer any data question without waiting for per-entity tools to be built.

**Independent Test**: Can be tested by asking the AI "List all purchase orders in validating status with their vendor names and amounts" — the agent should construct a GraphQL query with the right filters and fields.

**Acceptance Scenarios**:

1. **Given** an AI agent connected to the MCP server, **When** it sends a GraphQL query filtering purchase orders by status "validating", **Then** it receives matching purchase orders with only the requested fields
2. **Given** an AI agent, **When** it queries vendors with nested address data, **Then** it receives the complete nested response in a single query
3. **Given** an AI agent, **When** it queries with cursor-based pagination (first/after), **Then** it receives the requested page with `pageInfo` (endCursor, hasNextPage) and `totalCount`
4. **Given** an AI agent, **When** it sends a malformed query, **Then** it receives a descriptive GraphQL error message it can interpret and correct

---

### User Story 2 - AI Agent Discovers Available Data Schema (Priority: P1)

An AI agent discovers what entities, fields, and filters are available by combining existing MCP documentation resources (domain knowledge) with GraphQL introspection (exact field names and types). This lets the AI construct correct queries without hardcoded knowledge.

**Why this priority**: Without schema discovery, the AI would need to guess field names and filter syntax. Introspection makes the tool self-documenting and resilient to schema changes.

**Independent Test**: Can be tested by asking the AI "What fields does a purchase order have?" — the agent should use introspection or documentation resources to provide an accurate answer.

**Acceptance Scenarios**:

1. **Given** an AI agent, **When** it sends a GraphQL introspection query, **Then** it receives the full schema with entity types, fields, and available filters
2. **Given** an AI agent that has read the documentation resources, **When** it needs to query receipts, **Then** it can combine domain knowledge (what a receipt means) with introspection (exact field names) to construct a correct query
3. **Given** a schema change (new field added to an entity), **When** the AI introspects, **Then** it immediately sees the new field without any MCP tool updates

---

### User Story 3 - AI Agent Performs Cross-Entity Analysis (Priority: P2)

An AI agent answers complex questions that span multiple entities by composing multiple GraphQL queries or using nested queries. For example: "Compare spending by clinic this quarter" or "Which purchase requests are stuck in validation the longest?" The AI does the analysis — the tool just provides flexible data access.

**Why this priority**: The real value of an AI assistant is answering questions that require combining data from multiple sources. GraphQL's nested query capability makes this efficient.

**Independent Test**: Can be tested by asking "What's the total spending per clinic this month?" — the agent should query purchase orders with nested clinic data and compute totals from the results.

**Acceptance Scenarios**:

1. **Given** an AI agent, **When** asked about spending by clinic, **Then** it queries purchase orders with nested clinic data and computes totals from the results
2. **Given** an AI agent, **When** asked to compare receipts against orders for a vendor, **Then** it queries both entities and presents a comparison
3. **Given** an AI agent, **When** a question requires data from multiple entity types, **Then** it makes multiple focused GraphQL queries rather than one massive query

---

### Edge Cases

- **Query depth/complexity limits**: API Platform enforces `max_query_depth: 20` and `max_query_complexity: 500` by default. Exceeding these returns a descriptive GraphQL error the AI can interpret and adjust (e.g., reduce nesting or split into multiple queries).
- **Permission-restricted fields**: Firewall-level auth (ApiKeyAuthenticator) and Doctrine query extensions enforce access control. Unauthorized queries return empty results or auth errors — no field-level leakage.
- **Schema too large for AI context**: Solved by the `graphql_schema` tool (FR-004c) which returns domain-scoped SDL subsets instead of the full 50+ entity schema.
- **IRI-based identifiers**: GraphQL single-item queries use IRIs (e.g., `/purchase-orders/89`) following API Platform's Global Object Identification spec. The AI learns this pattern from collection query results which include `id` fields as IRIs.
- **camelCase field names**: GraphQL uses camelCase (matching PHP entity properties) by default. The tool description and `graphql_schema` output use the same naming. No conversion needed.

## Requirements *(mandatory)*

### Functional Requirements

#### GraphQL MCP Tool

- **FR-001**: System MUST expose a single MCP tool (`graphql_query`) that accepts a GraphQL query string, executes it via API Platform's GraphQL executor directly in PHP, and returns the result
- **FR-002**: The tool MUST support query operations (reads) and GraphQL introspection queries (targeted type introspection via `__type` and `__schema`)
- **FR-003**: The tool MUST return GraphQL responses as-is (data + errors) so the AI can interpret both successful results and error messages
- **FR-004**: The tool MUST support mutations when the API is ready for write operations — using the same single tool for all data interactions
- **FR-004b**: The tool description MUST include a guided summary: (1) all 9 entity collection query names (e.g., `purchaseOrders`, `vendors`), (2) AEFilter syntax with operator list (EQUALS, NOT_EQUALS, GT, GTE, LT, LTE, IN, NOT_IN, IS_NULL, IS_NOT_NULL, CONTAINS, STARTS_WITH, ENDS_WITH) and compound filter pattern (AND/OR), (3) cursor-based pagination args (`first`, `after`, `last`, `before`) with response shape (`totalCount`, `pageInfo`, `edges.node`), (4) single-item query pattern using IRI id (e.g., `purchaseOrder(id: "/purchase-orders/123")`)

#### GraphQL Schema Tool

- **FR-004c**: System MUST expose an MCP tool (`graphql_schema`) that returns the GraphQL schema in SDL format for a given business domain topic, generated at runtime from API Platform's SchemaBuilder
- **FR-004d**: The tool MUST accept a `topic` parameter matching the same domain grouping as the existing `get_app_context` tool (purchase-requests, purchase-orders, receipts-and-invoices, return-orders, products-and-vendors, organization, all)
- **FR-004e**: Each topic MUST return only the GraphQL types relevant to that domain — not the full 50+ entity schema
- **FR-004f**: The schema MUST auto-update when entities or attributes change — no static files or manual maintenance

#### GraphQL Configuration

- **FR-005**: System MUST enable API Platform's GraphQL support with Query and QueryCollection attributes added directly to entity classes — consistent with the existing pattern of direct REST operation attributes (#[Get], #[GetCollection], etc.)
- **FR-006**: Initial rollout MUST add Query and QueryCollection attributes on the core procurement entities: RegularPurchaseRequest, ServiceRegularizationRequest, PurchaseOrder, Receipt, RegularInvoice, CreditNote, ReturnOrder, Vendor, Product
- **FR-007**: Additional entities (Clinic, Category, SectionCategory, Budget, ValidationPath, etc.) SHOULD be exposed incrementally by adding the same attributes
- **FR-008**: GraphQL operations MUST reuse existing `#[ApiFilter]` declarations (AEFilter, OrderFilter) which auto-generate `filter` and `sort` arguments in GraphQL. QueryParameter compatibility with GraphQL is unverified (see research.md R3) and SHOULD be tested empirically during implementation as a potential future improvement
- **FR-009**: Pagination MUST use cursor-based pagination (API Platform's default for GraphQL, following the Relay Connection spec)
- **FR-010**: Nesting separator MUST be configured (e.g., `__`) to avoid ambiguity between nested property filters and underscore-containing field names

#### Serialization & Field Exposure

- **FR-011**: GraphQL operations MUST use default serialization (no explicit normalizationContext), matching existing REST Get/GetCollection behavior. Dedicated serialization groups MAY be added per-entity later if granular field control is needed (see research.md R4)
- **FR-012**: Sensitive fields (passwords, tokens, API keys, internal hashes) MUST NOT be exposed in GraphQL serialization groups
- **FR-013**: GraphQL serialization groups SHOULD be as permissive as practical for read operations — the goal is flexibility, not restriction. Only security-sensitive fields should be excluded
- **FR-014**: Associations (relations between entities) MUST be accessible in GraphQL to enable nested queries (e.g., purchase order → vendor → addresses)

#### Security

- **FR-015**: GraphQL Query and QueryCollection attributes MUST match the security behavior of their REST equivalents. Since current REST Get/GetCollection operations have no explicit security expressions (access control is handled by firewall + Doctrine query extensions), GraphQL operations start without explicit `security:` expressions. Explicit security SHOULD be added when REST operations add them (see research.md R5)
- **FR-016**: GraphQL security expressions SHOULD match the REST security behavior for equivalent operations (same access rules)
- **FR-017**: Association-level security SHOULD be configured on sensitive relations to prevent traversal attacks
- **FR-018**: Introspection MUST remain enabled so the AI can discover the schema

### Key Entities (Initial Rollout)

All entities already exposed via API Platform REST operations can be made available through GraphQL. The initial rollout focuses on:

- **RegularPurchaseRequest**: Product lines, requester, clinic, section category, budget link. Statuses: Draft → Validating → Validated → Rejected.
- **ServiceRegularizationRequest**: Service-based requests. Same lifecycle as regular purchase requests.
- **PurchaseOrder**: Product lines, vendor, payment modality, delivery terms. Statuses: Pending → Validating → Validated → Canceled.
- **Receipt**: Receipt lines with quantities received vs ordered. Compliance status tracking.
- **RegularInvoice**: Financial document matched to receipts. Accounting status.
- **CreditNote**: Credit document linked to return orders.
- **ReturnOrder**: Return reason, product lines, vendor accept/refuse flow. 10 statuses.
- **Vendor**: Addresses, payment terms, active/inactive status.
- **Product**: Code, designation, unit, pricing, section, type classification.

## Assumptions

- API Platform's GraphQL support exposes entities that have Query/QueryCollection attributes — same pattern as REST operations, no custom resolvers needed for standard reads
- The AI agent operates within the same security context as the authenticated user — GraphQL does not provide elevated access
- GraphQL uses IRI-based identifiers (e.g., `/purchase-orders/89`) following the Global Object Identification spec — the AI can learn this from documentation resources and from collection query results (`@id` fields in JSON-LD responses)
- GraphQL operations will use QueryParameter for filters and sorting — independent from the existing REST ApiFilter definitions, allowing GraphQL-specific filter configuration
- The existing AEFilter (Airtable-style compound/nested filter) has not been tested with GraphQL — compatibility needs to be researched during planning
- The existing 8 MCP documentation resources provide domain context (what entities mean, business rules, workflows) while GraphQL introspection provides technical context (field names, types, filters)
- Property names in GraphQL will follow the default convention (no conversion) — camelCase as defined in entity properties
- Write operations (mutations) are out of scope for this phase but the tool and GraphQL configuration support them for future extension
- The `search_products` and `get_purchase_order_statistics` existing MCP tools remain available alongside the GraphQL tool — they may be deprecated later once GraphQL covers their use cases

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: An AI agent can query any of the 9 initial entities through a single GraphQL MCP tool — no per-entity tools needed
- **SC-002**: An AI agent can answer user questions about live procurement data by composing GraphQL queries from documentation context and introspection
- **SC-003**: Permission-restricted queries enforce the same access rules as the REST API — no data leakage across workspaces or clinics
- **SC-004**: The AI can discover new entities and fields through introspection without requiring MCP tool updates
- **SC-005**: Adding a new entity to GraphQL requires only adding Query/QueryCollection attributes to the entity class — zero MCP tool changes
- **SC-006**: Sensitive fields (passwords, tokens) are never returned in GraphQL responses regardless of the query
