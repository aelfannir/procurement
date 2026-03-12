# US3 - Confirmation Popup `ACHAT-2148` (Priority: P1)

Parent: [spec.md](../spec.md) | Requirements: FR-005, FR-006, FR-011

As a user comptabilising an invoice with a non-zero discrepancy, I see a
confirmation popup so I can review the discrepancy before proceeding.

**Why this priority**: The popup is the gatekeeper for all discrepancy
accounting — it enforces user acknowledgment and mandatory reason capture.

**Independent Test**: Click Comptabiliser on an invoice with a discrepancy
— verify popup content, motif validation, and cancel behavior.

## Popup Trigger

The popup appears when `FRS_TTC (Devise) != SYS_TTC (Devise)`,
UNLESS a blocking rule is already triggered (above-threshold or missing config).

## Popup Template (exact wording)

```
Un écart de XX (Devise) a été détecté.
  - Montant facture système : XXX (Devise)
  - Montant facture fournisseur : XXX (Devise)
  - Sens de l'écart : Favorable clinique / Favorable fournisseur

Confirmez-vous la comptabilisation ?

Champ : Motif de comptabilisation de l'écart
Boutons : Confirmer / Annuler
```

## Motif Rules

- **Mandatory** if the discrepancy is vendor-favorable.
- **Optional** if the discrepancy is clinic-favorable.
- If the user clicks Confirmer without a motif when it is mandatory, the
  confirmation is refused with: "Le motif de comptabilisation de l'écart
  est obligatoire lorsque l'écart est favorable au fournisseur."
- When provided, the motif is persisted in the invoice audit fields.

## Acceptance Scenarios

1. **Given** an invoice with FRS_TTC != SYS_TTC and no blocking rule
   triggered, **When** I click Comptabiliser, **Then** the popup appears
   with the exact content template above.
2. **Given** a vendor-favorable discrepancy popup, **When** I click
   Confirmer without entering a motif, **Then** confirmation is refused
   with the mandatory-motif message.
3. **Given** a clinic-favorable discrepancy popup, **When** I confirm
   without a motif, **Then** accounting proceeds (motif is optional).
4. **Given** the popup, **When** I click Annuler, **Then** I return to
   the invoice in edit mode — no accounting entries are generated.
5. **Given** a confirmed discrepancy, **Then** the decision (confirm/cancel),
   motif, user, date, and time are logged in the invoice audit fields.
6. **Given** all displayed amounts in the popup, **Then** they include
   the (Devise) label derived from the clinic's reference country.
