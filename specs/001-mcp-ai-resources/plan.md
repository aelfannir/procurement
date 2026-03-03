# Implementation Plan: MCP AI Resources

**Branch**: `001-mcp-ai-resources` | **Date**: 2026-02-23 | **Spec**: [spec.md](./spec.md)

## Summary

Expose APP (AKDITAL Purchasing & Procurement) documentation as 8 domain-grouped MCP resources. Each resource is self-contained — includes its own statuses, enums, rules, and entity descriptions. Enum values are embedded dynamically from PHP enums (zero drift). All resources are static (no DB), use `text/markdown` format.

## Technical Context

**Language/Version**: PHP 8.2 / Symfony 7.x / API Platform 4.x (pre-release MCP support)
**Primary Dependencies**: API Platform MCP integration (`McpResource`), PHP enums
**Storage**: N/A (static reference data)
**Testing**: PHPUnit with API Platform's `ApiTestCase`
**Target Platform**: Linux server (Streamable HTTP)
**Performance Goals**: Each resource loads in < 1 second
**Scale/Scope**: 8 resources

## Constitution Check

Constitution is a blank template — no gates. Proceeding.

## Design Decisions

### D1: Resource provider pattern

Test simplified pattern first (return string, let API Platform handle `contents` wrapping). Fallback to `ReadResourceResult` if needed. See [research.md](./research.md) R1.

### D2: 8 domain-grouped resources

| # | Resource | URI | Key content |
|---|----------|-----|-------------|
| 1 | Procurement-Lifecycle | `resource://app/procurement-lifecycle` | APP overview, 3 workspaces, clinic↔workspace mapping, full lifecycle flow |
| 2 | Purchase-Requests | `resource://app/purchase-requests` | DA (products) vs DP (services), natures (standard/urgent/regularized), statuses, transitions |
| 3 | Purchase-Orders | `resource://app/purchase-orders` | Direct PO creation (both workspaces), statuses, cancellation rules, purchase types |
| 4 | Receipts-And-Invoices | `resource://app/receipts-and-invoices` | Partial receipts, compliance statuses, invoice matching, credit notes, invoice types |
| 5 | Return-Orders | `resource://app/return-orders` | Post-receipt returns, vendor accept/refuse, credit note linkage, statuses |
| 6 | Organization | `resource://app/organization` | Clinics (status→workspace), sections, section categories (tree), budgets (soft warning), workspaces |
| 7 | Products-And-Vendors | `resource://app/products-and-vendors` | Product types, isService flag, pricings, units, vendors, discount types |
| 8 | Validation-System | `resource://app/validation-system` | Paths, steps, roles/users, target types, amount thresholds, fallback selection logic |

### D3: Enum values embedded per domain

No separate statuses/enums resource. Each resource includes its enums inline, generated dynamically from PHP `::cases()`. Example in Purchase-Orders:

```markdown
## Statuses
- pending: Initial state after creation
- validating: Sent for approval
- validated: Approved
- rejected: Rejected by validator
- canceled: Canceled (only from pending/validating)
```

### D4: All markdown format

All 8 resources use `text/markdown` mimeType. Markdown is natural for AI reading. Dynamic enum sections are generated at runtime in PHP heredoc.

### D5: Content outline per resource

**1. Procurement-Lifecycle** (entry point — AI reads this first)
- What is APP (AKDITAL Purchasing & Procurement)
- The 3 workspaces:
  - **Project**: UnderConstruction clinics, no purchase requests, direct PO → Receipt → Invoice
  - **Exploitation**: Operational clinics, full PR → PO → Receipt → Invoice → Return
  - **Administration**: Global superadmin, system configuration, not tied to clinics
- Lifecycle diagram: PR → PO → Receipt → Invoice → ReturnOrder
- How workspaces affect available features
- Multi-currency: each clinic belongs to a Country with its own currency (e.g., MAD for Morocco, SAR for Saudi Arabia)

**2. Purchase-Requests** (Exploitation workspace only)
- Two types: RegularPurchaseRequest (DA, physical products) vs ServiceRegularizationRequest (DP, services)
- Product isService flag determines which type
- Natures: Standard, Urgent, Regularized (dynamic from PurchaseRequestNatureEnum)
- Regularization reasons: Fire, Flood, UrgentWorks (dynamic from RegularizationReasonEnum)
- Statuses with transitions (dynamic from PurchaseRequestStatusEnum): Draft → Validating → Validated → Processing → Processed, or Rejected/Canceled
- Key fields: requestNumber, vendor, clinic, sectionCategory, products, totals
- Validated total vs submitted total

**3. Purchase-Orders**
- Can be created from PR (Exploitation) or directly (both workspaces)
- Purchase types: ProductsServices, Prestations (dynamic from PurchaseType)
- Statuses with transitions (dynamic from PurchaseOrderStatusEnum): Pending → Validating → Validated → Rejected/Canceled
- Cancellation rule: only from Pending or Validating
- Key fields: orderNumber, vendor, clinic, products, totals (taxIncluded)

**4. Receipts-And-Invoices**
- Receipts: created from validated PO, partial receipts supported (multiple per PO)
- Quantity tracking: ordered vs received
- Compliance statuses (dynamic from ComplianceStatusEnum): None, Undefined, Conform, ConformWithReserve
- Invoices: matched against PO amounts
- Invoice types (dynamic from InvoiceType): Invoice, CreditNote
- Credit notes: linked to return orders
- Key fields: receiptNumber, purchaseOrder, products, attachments

**5. Return-Orders**
- Created after receipt only (returning received goods)
- Statuses (dynamic from ReturnOrderStatusEnum): Draft → Validating → Validated → AcceptedByVendor/RefusedByVendor → CreditNote flow → Closed
- Vendor response: accept or refuse the return
- Credit note: fully or partially received
- Key fields: products, attachment, status flow

**6. Organization**
- Clinics: abbreviation, city, ICE, taxId, CNSS, status (UnderConstruction/Operational)
  - UnderConstruction → Project workspace only
  - Operational → Exploitation workspace only
- Workspace (Environment entity): code, name, linked to users and validation paths
- SectionCategory: tree structure (parent/child), used for validation path scoping
- ProductSection: links products to sections
- Budget → BudgetExercise → ProductSectionBudget: spending control (soft warning, doesn't block)
- Roles: SuperAdmin, Admin, Buyer, BuyerGroup, ResponsableAchat, ResponsableAchatGroup, Reviewer
  - Roles have operations (CRUD permissions) and actions (custom permissions)

**7. Products-And-Vendors**
- Product types (dynamic from ProductTypeEnum): Simple, Combined, SubComponent (with prefixes)
- isService flag: determines PR type (DA vs DP)
- Product fields: code, designation, vatRate, locale
- ProductPricing: vendor-specific pricing
- Units (dynamic from UnitEnum)
- Vendors: code, name, email, addresses
- Discount types (dynamic from DiscountTypeEnum): Percent, Amount
- PaymentModality: payment terms and methods available for purchase orders

**8. Validation-System**
- ValidationPath: defines approval circuit
  - Fields: name, target, environment (workspace), clinic (optional), sectionCategory (optional), regularized flag, minAmount, maxAmount, status (enabled/archived)
  - Target types (dynamic from ValidationPathTargetEnum): PurchaseOrder, PurchaseRequest, ServiceRegularizationRequest
- Step: ordered approval steps within a path
  - Assigned to users and/or roles
- ValidationRequest: one per step when document sent for validation
  - Statuses (dynamic from ValidationRequestStatusEnum): Pending → Validated/Rejected/Reset
  - Tracks: who validated, when, comments on rejection
- **Fallback selection logic** (critical for AI understanding):
  - For RegularPurchaseRequest: try clinic+sectionCategory+regularized → clinic+parentCategory+regularized → workspace+sectionCategory+regularized → workspace+parentCategory+regularized
  - For PurchaseOrder/ServiceRegularization: try clinic → workspace (no clinic)
  - Then filter by amount range (minAmount/maxAmount vs document total)
  - First match wins

## Project Structure

```text
src/ApiResource/
├── ProcurementLifecycle.php      # New
├── PurchaseRequests.php          # New
├── PurchaseOrders.php            # New
├── ReceiptsAndInvoices.php       # New
├── ReturnOrders.php              # New
├── Organization.php              # New
├── ProductsAndVendors.php        # New
├── ValidationSystem.php          # New
└── WorkflowStatuses.php          # Delete after migration

tests/
└── McpApiTest.php                # Extend with resource tests
```

## Implementation Order

1. Test simplified provider pattern on WorkflowStatuses
2. ProcurementLifecycle (entry point)
3. PurchaseRequests
4. PurchaseOrders
5. ReceiptsAndInvoices
6. ReturnOrders
7. Organization
8. ProductsAndVendors
9. ValidationSystem
10. Delete WorkflowStatuses
11. Tests for all resources
12. Cleanup (remove debug code from tests)
