# Tasks: MCP AI Resources

**Input**: Design documents from `/specs/001-mcp-ai-resources/`
**Prerequisites**: plan.md, spec.md, research.md

**Organization**: Tasks are grouped by user story. Each resource is self-contained with embedded enums (US3) and business rules (US4), so US3/US4 are satisfied incrementally as each resource is implemented.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3, US4)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Validate the resource provider pattern before creating all 8 resources

- [x] T001 Test simplified provider pattern by refactoring `src/ApiResource/WorkflowStatuses.php` — Result: simplified pattern fails (StructuredContentProcessor bug), confirmed `ReadResourceResult` + `structuredContent: false` as the pattern
- [x] T002 Documented confirmed pattern in `specs/001-mcp-ai-resources/research.md`
- [x] T003 Remove debug code (`file_put_contents`) from `tests/McpApiTest.php`

**Checkpoint**: Provider pattern confirmed — all subsequent resources will follow the same pattern

---

## Phase 2: User Story 1 — AI Agent Understands Procurement Workflow (Priority: P1) 🎯 MVP

**Goal**: AI reads lifecycle, PR, and PO resources and can explain the full procurement workflow, statuses, and transitions

**Independent Test**: Ask AI "What happens after a purchase request is validated?" — it should describe PO creation. Ask "What are the possible statuses for a purchase order?" — it should list all 5.

### Implementation

- [x] T004 [US1] Create `src/ApiResource/ProcurementLifecycle.php` — APP overview, 3 workspaces (Project: UC clinics no PR, Exploitation: Operational clinics full flow, Administration: global superadmin), lifecycle flow PR→PO→Receipt→Invoice→Return, multi-currency (clinic→country→currency), how workspaces affect features
- [x] T005 [US1] Create `src/ApiResource/PurchaseRequests.php` — DA vs DP types, isService flag, natures (dynamic from PurchaseRequestNatureEnum), regularization reasons (dynamic from RegularizationReasonEnum), statuses with transitions (dynamic from PurchaseRequestStatusEnum), key fields, Exploitation workspace only
- [x] T006 [US1] Create `src/ApiResource/PurchaseOrders.php` — direct creation in both workspaces, purchase types (dynamic from PurchaseType), statuses with transitions (dynamic from PurchaseOrderStatusEnum), cancellation rules (only from pending/validating), key fields
- [x] T007 [US1] Create `src/ApiResource/ReceiptsAndInvoices.php` — partial receipts, no status workflow for receipts, compliance statuses (dynamic from ComplianceStatusEnum), invoice matching against PO, invoice types (dynamic from InvoiceType), credit notes linked to returns, key fields
- [x] T008 [US1] Create `src/ApiResource/ReturnOrders.php` — post-receipt only, statuses (dynamic from ReturnOrderStatusEnum), vendor accept/refuse flow, credit note linkage (full/partial), key fields
- [x] T009 [US1] Add tests in `tests/McpApiTest.php` — test `resources/list` contains all 5 new resources, test `resources/read` for `resource://app/procurement-lifecycle` returns markdown with workspace descriptions

**Checkpoint**: US1 complete — AI can understand the full procurement workflow by reading 5 resources

---

## Phase 3: User Story 2 — AI Agent Understands Business Entities and Relationships (Priority: P1)

**Goal**: AI reads organization, products/vendors, and validation resources and can explain entity attributes, relationships, and the validation system

**Independent Test**: Ask AI "What is a validation path?" — it should explain the configurable approval workflow. Ask "How are budgets structured?" — it should explain the hierarchy.

### Implementation

- [x] T010 [P] [US2] Create `src/ApiResource/Organization.php` — clinics (status→workspace mapping, ICE, taxId, CNSS), Country→City→Clinic hierarchy, workspaces (Environment entity), SectionCategory tree structure, ProductSection, Budget→BudgetExercise→ProductSectionBudget (soft warning), roles overview (SuperAdmin, Admin, Buyer, etc. with operations vs actions)
- [x] T011 [P] [US2] Create `src/ApiResource/ProductsAndVendors.php` — product types (dynamic from ProductTypeEnum with prefixes), isService flag, product fields (code, designation, vatRate), ProductPricing (vendor-specific), units (dynamic from UnitEnum), vendors (code, name, email, addresses), discount types (dynamic from DiscountTypeEnum)
- [x] T012 [US2] Create `src/ApiResource/ValidationSystem.php` — ValidationPath fields (name, target, environment, clinic, sectionCategory, regularized, minAmount, maxAmount), target types (dynamic from ValidationPathTargetEnum), Step entity (users and/or roles), ValidationRequest statuses (dynamic from ValidationRequestStatusEnum), fallback selection logic for PR (clinic+sectionCategory+regularized → parent category → workspace level) and PO (clinic → workspace), amount range matching
- [x] T013 [US2] Add tests in `tests/McpApiTest.php` — test `resources/list` contains Organization, Products-And-Vendors, Validation-System resources, test `resources/read` for `resource://app/validation-system` returns markdown mentioning fallback logic

**Checkpoint**: US2 complete — AI understands all entities, relationships, and the validation system

---

## Phase 4: User Story 3 & 4 — Status Values and Business Rules (Priority: P2)

**Goal**: Verify all dynamic enum values match PHP definitions (US3) and business rules are documented in each resource (US4). These are already embedded in resources from US1/US2 — this phase validates completeness.

**Independent Test (US3)**: Read each resource and verify enum values match `::cases()` output exactly.
**Independent Test (US4)**: Ask AI "Can I cancel a validated PO?" — it should answer correctly from the PO resource.

### Implementation

- [x] T014 [US3] Review all 8 resources and verify every dynamic enum section uses `::cases()` — no hardcoded values. Check: PurchaseRequestStatusEnum, PurchaseOrderStatusEnum, ReturnOrderStatusEnum, ValidationRequestStatusEnum, PurchaseRequestNatureEnum, RegularizationReasonEnum, PurchaseType, ComplianceStatusEnum, InvoiceType, ProductTypeEnum, UnitEnum, DiscountTypeEnum, ValidationPathTargetEnum, WorkspaceEnum
- [x] T015 [US4] Review all 8 resources and verify business rules are documented: PO cancellation rules, validation path requirements, budget soft warning, receipt partial tracking, return post-receipt constraint, workspace restrictions (no PR in Project)
- [x] T016 [US3] Add test in `tests/McpApiTest.php` — test `resources/read` for `resource://app/purchase-orders` and verify the response text contains all PurchaseOrderStatusEnum values

**Checkpoint**: US3+US4 complete — all enums are dynamic, all business rules are documented

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Cleanup and finalize

- [x] T017 Delete `src/ApiResource/WorkflowStatuses.php` — content migrated to domain resources
- [x] T018 Update `tests/McpApiTest.php` — remove old WorkflowStatuses tests (testMcpResourceList, testMcpResourceRead), update assertions to reflect new resource names
- [x] T019 Run full test suite and fix any failures

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies — validate pattern first
- **Phase 2 (US1)**: Depends on Phase 1 — uses confirmed provider pattern
- **Phase 3 (US2)**: Depends on Phase 1 — can run in parallel with Phase 2
- **Phase 4 (US3+US4)**: Depends on Phase 2 + Phase 3 — validation of existing resources
- **Phase 5 (Polish)**: Depends on Phase 4 — cleanup after all resources exist

### Parallel Opportunities

Phase 2 resources (T004-T008) are independent files but share the same pattern, so they can be implemented in parallel if desired.

Phase 3 resources (T010, T011) are marked [P] — they can be created in parallel. T012 (ValidationSystem) depends on understanding the fallback logic which is self-contained.

### Within Each Phase

- Resources before tests
- Each resource is a single self-contained PHP file
- No cross-resource dependencies

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Validate provider pattern (T001-T003)
2. Complete Phase 2: 5 workflow resources + tests (T004-T009)
3. **STOP and VALIDATE**: AI can explain procurement workflow
4. Deploy/demo if ready

### Full Delivery

1. Phase 1 → Pattern confirmed
2. Phase 2 → US1 complete (workflow understanding)
3. Phase 3 → US2 complete (entity understanding)
4. Phase 4 → US3+US4 validated (enums + rules)
5. Phase 5 → Cleanup, old resources removed

---

## Notes

- All resources use the same pattern: `#[ApiResource]` with `McpResource` in `mcp` config, static `provide()` method
- Content is markdown with dynamic enum sections generated from PHP `::cases()`
- No database access needed — all resources are static reference data
- Each resource file is ~100-300 lines (markdown heredoc + enum generation)
- Total: 19 tasks across 5 phases
