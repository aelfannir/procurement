# US6 - Vendor-Favorable Discrepancy Above Threshold `ACHAT-2162` (Priority: P2)

Parent: [spec.md](../spec.md) | Requirements: FR-007

As a user comptabilising an invoice where the vendor charges more than the
system amount and the discrepancy exceeds the threshold, the system blocks
comptabilisation.

**Why this priority**: Blocking rule that prevents unauthorized spending —
critical for financial control but depends on the accounting engine
infrastructure being in place (US4/US5).

**Independent Test**: Attempt to comptabilise an invoice with discrepancy
above threshold — verify blocking message and no entries generated.

## Business Rules

- When FRS_TTC (Devise) > SYS_TTC (Devise) AND discrepancy > clinic threshold:
  - Comptabilisation is **blocked**.
  - No accounting entries are generated.
  - The user stays on the invoice for correction.

## Acceptance Scenarios

1. **Given** an invoice with FRS_TTC (12,000) > SYS_TTC (10,000), threshold
   = 1,000, **When** I click Comptabiliser, **Then** no popup appears;
   instead a blocking message: "L'écart de 2 000 (MAD) dépasse le seuil
   autorisé de 1 000 (MAD) pour la clinique [Nom clinique]."
2. **Given** the blocking message, **Then** no accounting entries are
   generated and the invoice remains on the current screen.
3. **Given** the blocking message amounts, **Then** X (Devise) and Y
   (Devise) are displayed in the currency derived from the invoice's
   purchase order currency.
