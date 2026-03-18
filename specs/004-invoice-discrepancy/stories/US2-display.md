# US2 - Discrepancy Display on Invoice Screen `ACHAT-2150` (Priority: P1)

Parent: [spec.md](../spec.md) | Requirements: FR-003, FR-004

As a user viewing an invoice, I see a clear summary of the discrepancy
between system and vendor amounts so I understand the situation before
comptabilisation.

**Why this priority**: Users need visibility into discrepancies to make
informed decisions. This is prerequisite UX for the confirmation flow.

**Independent Test**: Open an invoice where SYS_TTC != FRS_TTC — verify
all fields are displayed with correct values.

## Data to Display

| Field | Type | Format | Required | Description |
|-------|------|--------|----------|-------------|
| Montant système | Calculated | Amount (18,2) + Devise | Yes | Total TTC from receipts |
| Montant fournisseur | Displayed | Amount (18,2) + Devise | Yes | Total TTC from vendor invoice |
| Écart | Calculated | Signed amount (18,2) + Devise | Yes | FRS_TTC - SYS_TTC |
| Sens de l'écart | Calculated | Enum | Yes | Nul / Favorable clinique / Favorable fournisseur |
| Couleur indicateur | UI | Badge / color | No | Visual indicator for discrepancy status |
| Motif | Text | Free text | Conditional | Read-only. Shows the motif entered during comptabilisation (US3). Hidden before comptabilisation, hidden when no discrepancy. |

**Visual indicator**: A badge, color, or equivalent marker MAY complement the
information, as long as it respects the existing UI ergonomics and does not
reduce readability.

## Acceptance Scenarios

1. **Given** an invoice where FRS_TTC = 10,000 and SYS_TTC = 10,500,
   **When** I view the invoice, **Then** I see: System amount = 10,500
   (Devise), Vendor amount = 10,000 (Devise), Discrepancy = -500
   (Devise), Direction = "Favorable clinique".
2. **Given** an invoice where FRS_TTC = SYS_TTC, **When** I view the
   invoice, **Then** discrepancy shows 0 and direction shows "Nul".
3. **Given** an invoice where FRS_TTC > SYS_TTC, **When** I view the
   invoice, **Then** direction shows "Favorable fournisseur".
4. **Given** any invoice, **When** I view it, **Then** all amounts display
   with the invoice's (Devise) label (from PO currency, e.g., "MAD", "SAR").
5. **Given** an invoice with a previously comptabilised discrepancy,
   **When** I view it, **Then** the motif field shows the saved reason.
6. **Given** an invoice with no discrepancy, **When** I view it, **Then**
   the motif field is hidden.
