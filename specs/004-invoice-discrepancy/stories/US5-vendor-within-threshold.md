# US5 - Vendor-Favorable Discrepancy Within Threshold `ACHAT-2161` (Priority: P1)

Parent: [spec.md](../spec.md) | Requirements: FR-006, FR-009, FR-010 | Cross-cutting: [US7](US7-vendor-account-rule.md)

As a user comptabilising an invoice where the vendor charges more than the
system amount but within the allowed threshold, the system requires a
mandatory reason and generates entries with the vendor-favorable account.

**Why this priority**: Core business rule — vendor-favorable discrepancies
require traceability (mandatory reason) and threshold enforcement.

**Independent Test**: Comptabilise an invoice with FRS_TTC > SYS_TTC but
discrepancy <= threshold — verify reason is required and entries are correct.

## Business Rules

- When FRS_TTC (Devise) > SYS_TTC (Devise) AND discrepancy <= clinic threshold:
  - Receipt lines are debited for SYS_TTC (Devise).
  - The difference is debited to the **vendor-favorable discrepancy account** (configured on the clinic).
  - Vendor account 4411/4481 is credited for FRS_TTC (Devise).
  - Motif is **mandatory**.
- The threshold used is the one defined for the relevant clinic.
- After successful confirmation and entry generation, the invoice status transitions to **Comptabilisé**.

## Acceptance Scenarios

1. **Given** an invoice with FRS_TTC (10,500) > SYS_TTC (10,000), threshold
   = 1,000, **When** I enter a motif and confirm, **Then** accounting entries are:
   - Debit: charge accounts for SYS_TTC (10,000)
   - Debit: vendor-favorable discrepancy account for 500
   - Credit: vendor account 4411/4481 for FRS_TTC (10,500)
2. **Given** successful comptabilisation, **Then** the reason, user, date,
   and time are persisted in the invoice audit fields.

For vendor account, balance, payment schedule, and status rules: see [US7](US7-vendor-account-rule.md).
