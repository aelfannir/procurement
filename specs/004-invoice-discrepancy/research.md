# Research: Invoice Discrepancy Management

Parent: [plan.md](plan.md)

## R1 — Custom Controller Justification (Constitution I: API Platform First)

**Decision**: Keep `AccountingController` as custom controller for discrepancy logic.

**Rationale**: The accounting endpoint orchestrates external Sage API calls, multi-step validation, and conditional entry generation — none of which fit API Platform's CRUD pattern. The controller already exists at `POST /accounting/{idInvoice}` and `POST /accounting/bulk`. The discrepancy feature extends this controller, not replaces it.

**Alternatives considered**: API Platform State Processor — rejected because the accounting flow is not a simple entity state change; it involves external API calls, conditional popup flow, and multi-entity writes (invoice + payment terms).

## R2 — Enum Strategy (Constitution III: Enum as Source of Truth)

**Decision**: Create `DiscrepancyDirectionEnum` backed enum.

**Rationale**: The discrepancy direction (Null, ClinicFavorable, VendorFavorable) is a categorical value used in display, logic branching, and audit. Per constitution, this MUST be a PHP backed enum.

**Fields**:
- `DiscrepancyDirectionEnum`: `Null = 'null'`, `ClinicFavorable = 'clinic_favorable'`, `VendorFavorable = 'vendor_favorable'`

**Alternatives considered**: Adding a `DiscrepancyScenarioEnum` for the processing path — rejected per YAGNI. The scenario is derivable from direction + whether blocked/confirmed. The `discrepancyScenario` audit field will store a free string label (e.g., "clinic_favorable", "vendor_within_threshold", "blocked_above_threshold") rather than a separate enum.

## R3 — Audit Storage (FR-011)

**Decision**: Store latest decision state as structured fields on Invoice entity. Rely on existing DH Auditor for historical audit trail.

**Rationale**: The Invoice entity is already marked `#[Audit\Auditable]` — DH Auditor tracks all field changes with user, timestamp, and diffs. Adding structured fields to Invoice gives us queryable current state. The auditor provides append-only history automatically. No separate `InvoiceDiscrepancyLog` entity needed.

**Fields added to Invoice**: `discrepancyMotif` (string), `discrepancyDecisionAt` (datetime), `discrepancyDecisionBy` (User relation), `discrepancyScenario` (string).

**Alternatives considered**: Separate audit log table — rejected per YAGNI; DH Auditor already does this.

## R4 — Vendor Account Amount (FR-010, US7)

**Decision**: No change to vendor account logic. The current implementation already credits vendor account for `totalAmount` (= FRS_TTC).

**Rationale**: Current `AccountingService` builds debit entries from receipt products (= SYS_TTC) and credits vendor account for `totalAmount` (= FRS_TTC). When these differ by > 0.01, it blocks. The discrepancy feature removes the block and adds a balancing entry to a discrepancy account. The vendor account amount is already FRS_TTC.

**Payment schedule**: Currently built from `totalAmount` (= FRS_TTC) via `buildPaymentTerms()`. Already compliant with FR-013. No change needed.

## R5 — Currency Derivation (FR-004)

**Decision**: Currency comes from `invoice->getPurchaseOrder()->getCurrencyCode()`, falling back to 'MAD'. This is already the existing pattern.

**Rationale**: The spec confirms (Devise) = invoice's currency. The current `AccountingService` already uses PO currency code (line 110). For display, the frontend already has access to `purchaseOrder.currencyCode`. No new currency field needed.

## R6 — Invoice amountDifference Field

**Decision**: Use existing `amountDifference` calculated field for display. Do NOT add a new persisted discrepancy field.

**Rationale**: The frontend already has `amountDifference` as a read-only calculated field (`totalAmount - sum(receiptProducts.priceInclTax)`). This IS the discrepancy. The backend calculates the same value in `processInvoiceAccounting()`. No need to duplicate.

**Direction derivation**: `amountDifference > 0` → Vendor-favorable, `< 0` → Clinic-favorable, `= 0` → Null.

## R7 — Clinic Account Relations (FR-001)

**Decision**: Add two ManyToOne relations from Clinic to Account entity for the discrepancy accounts. Add a decimal field for the threshold.

**Rationale**: The Account entity already exists with `code` and `name`. The spec requires dropdown selection from chart of accounts. ManyToOne relations give us entity-level validation and foreign key integrity.

**Note**: Existing ReceiptProduct stores account as a string code, but for clinic config, relations are preferable because: (1) the user selects from a dropdown, (2) we need to validate the account exists and is active at comptabilisation time (FR-016).

## R8 — Frontend Dialog Pattern

**Decision**: Create a new `DiscrepancyConfirmDialog` component using the `Dialog` component (not `ConfirmationModal`).

**Rationale**: The existing `ConfirmationModal` is a simple AlertDialog with no input fields. The discrepancy popup needs a textarea for motif + display of amounts + conditional validation. The `Dialog` component from base-ui is already used in `BulkAccountingModal` and supports flexible layouts.

## R9 — Bulk Comptabilisation (FR-015)

**Decision**: Filter discrepancy invoices client-side before calling bulk endpoint. Backend also validates per-invoice.

**Rationale**: The existing bulk endpoint (`POST /accounting/bulk`) iterates invoices and calls the service for each. The backend will naturally block discrepancy invoices (since they require confirmation). The frontend filters them out pre-request for UX (avoids unnecessary API calls and provides immediate feedback). The result summary already returns `{totalProcessed, success, failed}`.

## R10 — Workspace Impact (Constitution VI: Multi-Workspace Awareness)

**Decision**: Discrepancy config is on Clinic entity, which is workspace-independent. The accounting flow is the same across workspaces. No workspace-specific logic needed.

**Rationale**: Clinics are shared across workspaces. The discrepancy parameters (threshold, accounts) are clinic-level, not workspace-level. The accounting button visibility is already controlled by permissions, which are workspace-dependent. No change to workspace model.
