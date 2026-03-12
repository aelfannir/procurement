# US1 - Clinic Discrepancy Configuration `ACHAT-2172` (Priority: P1)

Parent: [spec.md](../spec.md) | Requirements: FR-001, FR-002

As a clinic administrator, I configure discrepancy parameters on the clinic
form so that invoice accounting rules are enforced per-clinic.

**Why this priority**: Without configuration, no discrepancy processing can
happen. This is the foundation for all other stories.

**Independent Test**: Create/edit a clinic, fill the discrepancy fields,
save — verify fields persist and validation enforces completeness.

## Data to Configure

| Field | Type / Format | Required | Description |
|-------|--------------|----------|-------------|
| Seuil d'écart TTC autorisé | Decimal positive (18,2) | Yes | Threshold for vendor-favorable discrepancies only |
| Compte écart favorable à la clinique | Account reference (dropdown) | Yes | Account used when FRS_TTC < SYS_TTC |
| Compte écart favorable au fournisseur | Account reference (dropdown) | Yes | Account used when FRS_TTC > SYS_TTC |

## Business Rules

- The discrepancy config is NOT a standalone screen — it is integrated into the existing clinic form.
- Parameters are available on both clinic **creation** and **modification**.
- Currency is NOT configurable in this block — it is derived automatically from the clinic's reference country.
- One discrepancy configuration per clinic (no duplicates).
- Account references are selected from the chart of accounts — no hard-coded account numbers.
- Threshold can be 0 (meaning: no vendor-favorable discrepancy is tolerated).
- If a clinic-level config exists, it takes priority over any generic configuration.

## Acceptance Scenarios

1. **Given** a clinic form in creation or edit mode, **When** I fill all 3
   discrepancy fields (threshold, clinic-favorable account, vendor-favorable
   account), **Then** the clinic saves successfully.
2. **Given** a clinic form, **When** I leave the threshold empty but fill
   both accounts, **Then** save is blocked with: "Le paramétrage d'écart de
   la clinique [Nom] est incomplet : seuil d'écart TTC autorisé manquant."
3. **Given** a clinic form, **When** I leave the clinic-favorable account
   empty, **Then** save is blocked with: "Le paramétrage d'écart de la
   clinique [Nom] est incomplet : compte d'écart favorable à la clinique manquant."
4. **Given** a clinic form, **When** I leave the vendor-favorable account
   empty, **Then** save is blocked with: "Le paramétrage d'écart de la
   clinique [Nom] est incomplet : compte d'écart favorable au fournisseur manquant."
5. **Given** a clinic with threshold = 0, **When** I save, **Then** it
   succeeds (threshold 0 means no vendor-favorable discrepancy is allowed).
6. **Given** a clinic form with all 3 discrepancy fields empty, **When** I
   save, **Then** save is blocked with the first missing field message
   (seuil d'écart TTC autorisé manquant).
