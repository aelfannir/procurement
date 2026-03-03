# Tasks: MCP Data Tools via GraphQL

**Input**: Design documents from `/specs/002-mcp-graphql-tools/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/graphql-schema.md

**Organization**: Tasks are grouped by user story to enable independent implementation and testing.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2)
- Exact file paths included in descriptions

---

## Phase 1: Setup

**Purpose**: Enable GraphQL in API Platform configuration

- [x] T001 Add GraphQL configuration section (enabled, nesting_separator, introspection) to config/packages/api_platform.yaml

**Checkpoint**: GraphQL endpoint at `/graphql` responds (introspection works, no entity types yet beyond auto-generated ones)

---

## Phase 2: Foundational (Entity GraphQL Attributes)

**Purpose**: Add `#[Query]` and `#[QueryCollection]` attributes to all 9 core entities. Existing `#[ApiFilter(AEFilter::class)]` and `#[ApiFilter(OrderFilter::class)]` will auto-generate `filter` and `sort` args in GraphQL.

**CRITICAL**: Must complete before US1/US2 tools can return meaningful results

- [x] T002 [P] Add `#[Query]` and `#[QueryCollection]` attributes to src/Entity/PurchaseOrder.php
- [x] T003 [P] Add `#[Query]` and `#[QueryCollection]` attributes to src/Entity/RegularPurchaseRequest.php
- [x] T004 [P] Add `#[Query]` and `#[QueryCollection]` attributes to src/Entity/ServiceRegularizationRequest.php
- [x] T005 [P] Add `#[Query]` and `#[QueryCollection]` attributes to src/Entity/Receipt.php
- [x] T006 [P] Add `#[Query]` and `#[QueryCollection]` attributes to src/Entity/RegularInvoice.php
- [x] T007 [P] Add `#[Query]` and `#[QueryCollection]` attributes to src/Entity/CreditNote.php
- [x] T008 [P] Add `#[Query]` and `#[QueryCollection]` attributes to src/Entity/ReturnOrder.php
- [x] T009 [P] Add `#[Query]` and `#[QueryCollection]` attributes to src/Entity/Vendor.php
- [x] T010 [P] Add `#[Query]` and `#[QueryCollection]` attributes to src/Entity/Product.php
- [x] T011 Verify GraphQL schema generation: run `docker compose exec php bin/console api:graphql:export` and confirm all 9 entity types appear with filter/sort args

**Note**: Sub-entities (PurchaseOrderProduct, PurchaseRequestProduct, ReceiptProduct, ReturnOrderProduct, VendorAddress) do NOT need explicit `#[Query]`/`#[QueryCollection]` — they are auto-exposed via associations from parent entities in GraphQL nested queries.

**Checkpoint**: All 9 entities queryable via `/graphql` endpoint with filters, sorting, and cursor-based pagination

---

## Phase 3: User Story 1 - AI Agent Queries Any Entity Flexibly (Priority: P1)

**Goal**: Single MCP tool (`graphql_query`) that accepts a GraphQL query string, executes it via API Platform's GraphQL executor directly in PHP, and returns results

**Independent Test**: Send a GraphQL query filtering purchase orders by status via MCP tool and receive matching results with only requested fields

### Implementation for User Story 1

- [x] T012 [US1] Create input DTO with `query` property (string, required) in src/Dto/GraphqlQueryInput.php
- [x] T013 [US1] Create GraphQL query processor that injects `ExecutorInterface` + `SchemaBuilderInterface`, executes query, and returns `data`/`errors` response in src/State/GraphqlQueryProcessor.php
- [x] T014 [US1] Create MCP tool definition with `McpTool` attribute, guided tool description (entity names, AEFilter syntax, pagination pattern), annotations (readOnlyHint: true, idempotentHint: true, openWorldHint: false) in src/ApiResource/GraphqlQuery.php
- [x] T015 [US1] Smoke test: container compiles, processor wired with SchemaBuilder + Executor services, tool registered in MCP

**Checkpoint**: AI agent can query any of the 9 entities through the `graphql_query` MCP tool with filters, pagination, nested relations, and introspection

---

## Phase 4: User Story 2 - AI Agent Discovers Available Data Schema (Priority: P1)

**Goal**: Domain-scoped GraphQL schema tool that returns SDL for a specific business domain topic, generated at runtime from API Platform's SchemaBuilder

**Independent Test**: Call `graphql_schema` tool with topic "purchase-orders" and receive SDL containing only PurchaseOrder and PurchaseOrderProduct types

### Implementation for User Story 2

- [x] T016 [US2] Create input DTO with `topic` property (string, required) in src/Dto/GraphqlSchemaInput.php
- [x] T017 [US2] Create schema processor that injects `SchemaBuilderInterface`, generates full SDL via `SchemaPrinter::doPrint()`, then filters types by topic mapping in src/State/GraphqlSchemaProcessor.php
- [x] T018 [US2] Create MCP tool definition with `McpTool` attribute, description listing available topics and domain-to-type mapping, annotations (readOnlyHint: true, idempotentHint: true, openWorldHint: false) in src/ApiResource/GraphqlSchema.php
- [x] T019 [US2] Smoke test: container compiles, processor wired with SchemaBuilder, tool registered in MCP

**Checkpoint**: AI agent can discover schema for any domain topic without loading the full 50+ entity schema

---

## Phase 5: User Story 3 - AI Agent Performs Cross-Entity Analysis (Priority: P2)

**Goal**: Validate that the tools from US1 + US2 enable cross-entity analysis (composing multiple queries, using nested relations)

**Independent Test**: Execute a nested query fetching purchase orders with vendor and clinic data, then a separate query for receipts — verify both return correct nested structures

### Implementation for User Story 3

- [x] T020 [US3] Validate nested relation queries work: PurchaseOrder type includes vendor (Vendor), clinic (Clinic), products (PurchaseOrderProduct) with nested relations in schema
- [x] T021 [US3] Validate cross-entity composition: all 9 core entities expose full fields with nested relations, enabling multi-query analysis

**Checkpoint**: AI agent can answer complex questions spanning multiple entities by composing GraphQL queries

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Final validation and refinements

- [x] T022 Refine graphql_query tool description based on actual schema field names and tested filter patterns in src/ApiResource/GraphqlQuery.php
- [x] T023 Schema validates: all 9 core entities + 39 supporting entities generate correct GraphQL types with fields, relations, and pagination
- [x] T024 Error handling: GraphQL executor includes max_query_depth: 20 and max_query_complexity: 500 validation rules (configured in Executor constructor)
- [x] T025 Security verified: User type only exposes `id` (no password/email/roles), sensitive fields absent from GraphQL schema

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 — BLOCKS all user stories
- **US1 (Phase 3)**: Depends on Phase 2 — core tool
- **US2 (Phase 4)**: Depends on Phase 2 — can run in parallel with US1
- **US3 (Phase 5)**: Depends on Phase 3 (needs graphql_query tool working)
- **Polish (Phase 6)**: Depends on US1 + US2 complete

### User Story Dependencies

- **US1 (P1)**: Can start after Foundational (Phase 2) — no dependency on other stories
- **US2 (P1)**: Can start after Foundational (Phase 2) — independent of US1
- **US3 (P2)**: Depends on US1 completion (needs working graphql_query tool)

### Parallel Opportunities

- T002-T010: All 9 entity attribute tasks can run in parallel (different files)
- T012-T014 (US1) and T016-T018 (US2): Can run in parallel (different files, independent tools)

---

## Parallel Example: Phase 2 (Entity Attributes)

```bash
# All 9 entity modifications can run in parallel:
T002: Add Query/QueryCollection to PurchaseOrder.php
T003: Add Query/QueryCollection to RegularPurchaseRequest.php
T004: Add Query/QueryCollection to ServiceRegularizationRequest.php
T005: Add Query/QueryCollection to Receipt.php
T006: Add Query/QueryCollection to RegularInvoice.php
T007: Add Query/QueryCollection to CreditNote.php
T008: Add Query/QueryCollection to ReturnOrder.php
T009: Add Query/QueryCollection to Vendor.php
T010: Add Query/QueryCollection to Product.php
```

## Parallel Example: US1 + US2

```bash
# US1 and US2 tool creation can run in parallel:
T012-T014: GraphqlQuery tool (DTO, processor, tool definition)
T016-T018: GraphqlSchema tool (DTO, processor, tool definition)
```

---

## Implementation Strategy

### MVP First (US1 Only)

1. Complete Phase 1: Setup (config)
2. Complete Phase 2: Foundational (9 entity attributes)
3. Complete Phase 3: US1 (graphql_query tool)
4. **STOP and VALIDATE**: AI can query all 9 entities with filters, pagination, nested relations
5. This alone delivers 80% of the feature value

### Incremental Delivery

1. Setup + Foundational -> GraphQL endpoint live with 9 entities
2. Add US1 (graphql_query) -> AI can query any entity flexibly (MVP!)
3. Add US2 (graphql_schema) -> AI can discover schema by domain topic
4. Add US3 (validation) -> Confirm cross-entity analysis works
5. Polish -> Refine descriptions, run full validation

---

## Notes

- Security: graphql firewall already added to config/packages/security.yaml during research
- AEFilter auto-generates `filter` and `sort` args — no custom filter configuration needed
- Default serialization (no explicit groups) matches REST Get/GetCollection behavior
- Cursor-based pagination is API Platform's default for GraphQL (Relay Connection spec)
- IRI identifiers used for single-item queries (e.g., `/purchase-orders/123`)
