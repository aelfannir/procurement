# API Contract: Accounting Endpoints

Parent: [plan.md](../plan.md)

## POST `/custom/accounting/{idInvoice}` — Comptabilise Invoice

**Changes**: Accepts optional `discrepancyMotif` in request body.

### Request

```json
{
  "discrepancyMotif": "string | null"
}
```

- `discrepancyMotif`: Required when discrepancy is vendor-favorable. Optional for clinic-favorable. Ignored when no discrepancy.
- Empty body is valid for zero-discrepancy invoices (backward compatible).

### Response — Success (200)

Existing response structure unchanged. Invoice is now accounted.

### Response — Blocked (422)

Existing violation format:

```json
{
  "status": 422,
  "violations": [
    { "propertyPath": "discrepancy", "message": "L'écart de 2 000 (MAD) dépasse le seuil autorisé de 1 000 (MAD) pour la clinique Clinique A." }
  ]
}
```

**New violation messages**:

| Condition | propertyPath | message |
|-----------|-------------|---------|
| Above threshold | `discrepancy` | "L'écart de X (Devise) dépasse le seuil autorisé de Y (Devise) pour la clinique [Nom]." |
| No config | `discrepancy` | "Aucun paramétrage d'écart n'est défini pour la clinique [Nom]. Veuillez renseigner le seuil autorisé ainsi que les comptes comptables d'écart avant de comptabiliser cette facture." |
| Incomplete config | `discrepancy` | "Le paramétrage d'écart de la clinique [Nom] est incomplet : [champ manquant]." |
| Invalid account | `discrepancy` | "Le compte d'écart de la clinique [Nom] est invalide. Merci de mettre à jour le paramétrage avant comptabilisation." |
| Missing motif | `discrepancyMotif` | "Le motif de comptabilisation de l'écart est obligatoire lorsque l'écart est favorable au fournisseur." |

### ~~Response — Needs Confirmation (409)~~ — Alternative, NOT implemented

> **Decision**: Discrepancy detection is computed client-side from existing invoice data (`amountDifference` + PO currency) for UX speed. The backend validates everything server-side on POST. This 409 round-trip is NOT used.

When a non-zero discrepancy is detected and no motif/confirmation is provided, the backend would return a 409 with discrepancy details for the popup:

```json
{
  "status": 409,
  "type": "discrepancy_confirmation_required",
  "data": {
    "systemAmount": 10000.00,
    "vendorAmount": 10500.00,
    "discrepancy": 500.00,
    "discrepancyAbsolute": 500.00,
    "direction": "vendor_favorable",
    "directionLabel": "Favorable fournisseur",
    "currency": "MAD",
    "motifRequired": true,
    "threshold": 1000.00
  }
}
```

Frontend would use this to populate the popup. User enters motif and re-sends the request with `discrepancyMotif`.

---

## POST `/custom/accounting/bulk` — Bulk Comptabilisation

**Changes**: Response includes discrepancy-excluded invoices.

### Request (unchanged)

```json
{
  "invoiceIds": [1, 2, 3]
}
```

### Response (enhanced)

```json
{
  "totalProcessed": 3,
  "success": 1,
  "failed": 0,
  "excluded": 2,
  "results": [
    { "invoiceId": 1, "status": "success", "responseData": { ... } },
    { "invoiceId": 2, "status": "excluded", "reason": "discrepancy_requires_confirmation" },
    { "invoiceId": 3, "status": "excluded", "reason": "discrepancy_requires_confirmation" }
  ]
}
```

**New status values**: `"excluded"` for discrepancy invoices that require individual confirmation.

---

## Clinic Entity — API Platform Serialization

New fields exposed via existing Clinic API operations:

| Field | Read Groups | Write Groups | Type |
|-------|------------|-------------|------|
| `discrepancyThreshold` | ClinicListing, ClinicDetail | ClinicCreate, ClinicUpdate | number |
| `clinicFavorableAccount` | ClinicDetail | ClinicCreate, ClinicUpdate | IRI (Account) |
| `vendorFavorableAccount` | ClinicDetail | ClinicCreate, ClinicUpdate | IRI (Account) |

---

## Invoice Entity — API Platform Serialization

New audit fields exposed as read-only:

| Field | Read Groups | Write Groups | Type |
|-------|------------|-------------|------|
| `discrepancyMotif` | InvoiceDetail | — | string \| null |
| `discrepancyScenario` | InvoiceDetail | — | string \| null |
| `discrepancyDecisionAt` | InvoiceDetail | — | datetime \| null |
| `discrepancyDecisionBy` | InvoiceDetail | — | IRI (User) \| null |
