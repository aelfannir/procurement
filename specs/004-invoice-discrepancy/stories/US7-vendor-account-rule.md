# US7 - Vendor Account = FRS_TTC `ACHAT-2147` (Priority: P1, cross-cutting)

Parent: [spec.md](../spec.md) | Requirements: FR-009, FR-010, FR-013, FR-014

As a user comptabilising any invoice, the vendor account (4411/4481) MUST
always be based on the vendor invoice total (FRS_TTC), and the payment
schedule amount MUST also use FRS_TTC.

**Why this priority**: This is a transversal accounting rule that applies
to all discrepancy scenarios and must be enforced engine-wide.

**Independent Test**: Comptabilise invoices across all 3 discrepancy
scenarios — verify vendor account and payment schedule always use FRS_TTC.

## Business Rules

- Vendor account 4411/4481 always uses FRS_TTC (Devise).
- Receipt lines remain debited by their charge type.
- Payment schedule amount MUST use FRS_TTC (Devise).
- The system refuses any unbalanced entry.
- The **accounting entries tab** MUST display the generated lines according to the applied rule.
- After successful comptabilisation, the invoice status transitions to **Comptabilisé**.

## Acceptance Scenarios

1. **Given** any invoice (with or without discrepancy), **When** comptabilised,
   **Then** the vendor account 4411/4481 shows FRS_TTC (Devise).
2. **Given** any comptabilised invoice, **Then** receipt lines are debited
   according to their charge type.
3. **Given** any comptabilised invoice, **Then** the payment schedule amount
   uses FRS_TTC (Devise).
4. **Given** any comptabilised invoice, **Then** the system rejects any
   unbalanced entry.
5. **Given** any comptabilised invoice, **Then** the accounting entries tab
   displays the generated lines per the applied rule.
6. **Given** successful comptabilisation, **Then** the invoice status
   transitions to Comptabilisé.
