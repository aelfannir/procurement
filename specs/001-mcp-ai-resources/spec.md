# Feature Specification: MCP AI Resources - Application Documentation

**Feature Branch**: `001-mcp-ai-resources`
**Created**: 2026-02-23
**Status**: Draft
**Target**: api
**Input**: User description: "Write all app documentation as MCP resources so AI agents understand how the app works"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - AI Agent Understands Procurement Workflow (Priority: P1)

An AI agent connects to the procurement MCP server and reads available resources to understand the complete procurement lifecycle: how purchase requests become purchase orders, how receipts and invoices are processed, and how return orders work. The agent can then answer user questions about workflow status, next steps, and business rules without needing additional context.

**Why this priority**: Without understanding the core workflow, the AI cannot assist users with any procurement task. This is the foundational context every AI interaction depends on.

**Independent Test**: Can be tested by connecting an AI agent, reading the workflow resource, and asking "What happens after a purchase request is validated?" — the agent should accurately describe the purchase order creation step.

**Acceptance Scenarios**:

1. **Given** an AI agent connected to the MCP server, **When** it reads the workflow documentation resource, **Then** it receives a complete description of the Purchase Request → Purchase Order → Receipt → Invoice lifecycle
2. **Given** an AI agent with workflow context, **When** asked "What are the possible statuses for a purchase order?", **Then** it correctly lists: pending, validating, validated, rejected, canceled
3. **Given** an AI agent with workflow context, **When** asked "How does validation work?", **Then** it explains the validation path, steps, and approval/rejection flow

---

### User Story 2 - AI Agent Understands Business Entities and Relationships (Priority: P1)

An AI agent reads resources describing the key business entities (vendors, products, clinics, categories, sections, budgets) and their relationships. It can then help users navigate the system, explain what fields mean, and understand how entities connect.

**Why this priority**: Entity knowledge is required alongside workflow knowledge for the AI to be useful. An agent that knows the workflow but not what a "clinic" or "section category" is cannot help users effectively.

**Independent Test**: Can be tested by asking the AI "What information does a vendor have?" or "How are products organized?" — the agent should describe the entity attributes and relationships correctly.

**Acceptance Scenarios**:

1. **Given** an AI agent with entity documentation, **When** asked "What is a validation path?", **Then** it explains: a configurable approval workflow with steps, amount thresholds, target type, and clinic/section scope
2. **Given** an AI agent with entity documentation, **When** asked "How are budgets structured?", **Then** it explains the Budget → BudgetExercise → ProductSectionBudget hierarchy and its role in spending control
3. **Given** an AI agent with entity documentation, **When** asked about a clinic, **Then** it explains clinic attributes including status (UnderConstruction/Operational), tax identifiers, and workspace implications

---

### User Story 3 - AI Agent Knows All Status Values and Transitions (Priority: P2)

An AI agent reads resources containing all enumeration values used across the system — statuses, types, categories, and classification codes. When users ask about allowed values or next steps, the AI provides accurate, up-to-date answers.

**Why this priority**: Status values and enumerations change over time. Exposing them as live resources ensures the AI always has current data rather than stale training knowledge.

**Independent Test**: Can be tested by reading the statuses resource and verifying all enum values match the application's actual enum definitions.

**Acceptance Scenarios**:

1. **Given** an AI agent reading the statuses resource, **When** asked "What statuses can a return order have?", **Then** it lists all 10 ReturnOrder statuses accurately
2. **Given** an AI agent with enum knowledge, **When** asked "What types of purchase requests exist?", **Then** it distinguishes between Regular Purchase Requests and Service Regularization Requests with their prefixes (DA, DP)
3. **Given** an AI agent with enum knowledge, **When** asked "What are the workspace types?", **Then** it explains project, exploitation, and administration workspaces and how they relate to clinic status

---

### User Story 4 - AI Agent Understands Business Rules and Constraints (Priority: P2)

An AI agent reads resources documenting the business rules that govern the procurement system: validation path selection criteria, budget constraints, required fields, and compliance rules. It can proactively warn users about potential issues.

**Why this priority**: Business rules are the most complex part of the system and the area where users need the most help. Without this knowledge, the AI can describe the workflow but cannot guide users through it effectively.

**Independent Test**: Can be tested by asking the AI "What do I need before I can validate a purchase order?" — it should list the validation path requirement, amount thresholds, and assigned validators.

**Acceptance Scenarios**:

1. **Given** an AI agent with business rules context, **When** asked about validation requirements, **Then** it explains that a ValidationPath must exist matching the target type, amount range, clinic, and section category
2. **Given** an AI agent with business rules context, **When** asked "Can I cancel a validated purchase order?", **Then** it correctly states that only pending or validating orders can be canceled
3. **Given** an AI agent with business rules context, **When** asked about budget control, **Then** it explains how ProductSectionBudget allocations work within BudgetExercise periods

---

### Edge Cases

- What happens when enum values are added or removed from the application? Resources should reflect current values dynamically.
- How does the AI handle questions about features that span multiple resources (e.g., "walk me through creating a purchase order from start to finish")?
- What if the AI reads a resource that references entities described in another resource it hasn't read yet?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST expose a workflow documentation resource describing APP (AKDITAL Purchasing & Procurement), the 3 workspaces (Project, Exploitation, Administration) with their constraints, multi-currency support, and the complete procurement lifecycle (Purchase Request → Purchase Order → Receipt → Invoice → Return Order)
- **FR-002**: System MUST expose status values and their allowed transitions for each workflow entity (PurchaseRequest, PurchaseOrder, ReturnOrder, ValidationRequest), embedded within each entity's domain resource
- **FR-003**: System MUST expose a resource describing the validation system: validation paths, steps, roles, amount thresholds, and target types
- **FR-004**: System MUST expose a resource documenting key business entities: Vendor, Product, Clinic, Category, ProductSection, SectionCategory, Budget, PaymentModality
- **FR-005**: System MUST expose a resource documenting business rules and constraints: required fields for transitions, cancellation rules, compliance statuses, receipt/invoice tracking
- **FR-006**: System MUST expose all classification enumerations (purchase types, natures, workspace types, product types, invoice types, measurement units) embedded within each relevant domain resource
- **FR-007**: Each resource MUST be readable by any MCP-compatible AI agent via the `resources/read` method
- **FR-008**: Each resource MUST be listed in the `resources/list` response with a clear name, description, and URI
- **FR-009**: Status and enumeration resources MUST reflect the current application values (not hardcoded documentation)
- **FR-010**: Resources MUST be readable without database access (static reference data)

### Key Entities

- **MCP Resource**: A read-only content unit exposed to AI agents via the MCP protocol. Has a URI, name, description, MIME type, and text content.
- **Workflow Status**: The set of possible states for each procurement entity and the allowed transitions between them.
- **Business Rule**: A constraint or requirement that governs when and how procurement operations can be performed.
- **Enumeration**: A fixed set of values used for classification (statuses, types, categories).

## Assumptions

- All documentation resources are static (no database queries needed) — they describe the system's structure and rules, not live data
- Resources are organized by business domain (purchase requests, purchase orders, receipts, etc.) so each resource is self-contained — AI reads one resource and gets statuses, rules, and entity descriptions for that domain
- The provider pattern (static method returning content, API Platform handles `contents` wrapping) will be validated during implementation; fallback to `ReadResourceResult` if needed
- Content is written in plain language optimized for AI comprehension, not as formal API documentation
- All resources use markdown format with dynamic enum values embedded inline (generated from PHP enums at runtime)

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: An AI agent reading all documentation resources can correctly answer 90% of questions about the procurement workflow without additional context
- **SC-002**: All enumeration/status resources return values that exactly match the application's current enum definitions (zero drift)
- **SC-003**: A new AI agent can go from zero knowledge to understanding the full procurement system by reading fewer than 10 resources
- **SC-004**: Each resource loads in under 1 second (static content, no database queries)
- **SC-005**: An AI agent can explain the complete lifecycle of any procurement entity (purchase request, purchase order, receipt, invoice, return order) after reading the workflow resource
