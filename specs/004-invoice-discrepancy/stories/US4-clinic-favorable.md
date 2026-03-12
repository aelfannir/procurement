# US4 - Clinic-Favorable Discrepancy Accounting `ACHAT-2173` (Priority: P1)

Parent: [spec.md](../spec.md) | Requirements: FR-009, FR-010 | Cross-cutting: [US7](US7-vendor-account-rule.md)

As a user comptabilising an invoice where the vendor charges less than the
system amount, the system generates the correct accounting entries with the
discrepancy posted to the clinic-favorable account.

**Why this priority**: This is the simplest discrepancy scenario — no
threshold check, optional reason — and validates the core accounting entry
generation logic.

**Independent Test**: Comptabilise an invoice with FRS_TTC < SYS_TTC on a
fully configured clinic — verify entries balance and discrepancy account is
correct.

## Business Rules

- When FRS_TTC (Devise) < SYS_TTC (Devise):
  - Receipt lines are debited for SYS_TTC (Devise).
  - Vendor account 4411/4481 is credited for FRS_TTC (Devise).
  - The difference is credited to the **clinic-favorable discrepancy account** (configured on the clinic).
  - **No threshold check** is applied.
- After successful confirmation and entry generation, the invoice status transitions to **Comptabilisé**.

## Acceptance Scenarios

1. **Given** an invoice with FRS_TTC (8,000) < SYS_TTC (10,000) on a clinic
   with complete discrepancy config, **When** I confirm via the popup,
   **Then** accounting entries are:
   - Debit: charge accounts for SYS_TTC (10,000)
   - Credit: vendor account 4411/4481 for FRS_TTC (8,000)
   - Credit: clinic-favorable discrepancy account for 2,000
2. **Given** no threshold check is applied, **Then** any clinic-favorable
   discrepancy amount is accepted (even large amounts).

For vendor account, balance, payment schedule, and status rules: see [US7](US7-vendor-account-rule.md).
