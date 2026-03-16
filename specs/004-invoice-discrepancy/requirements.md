# Requirements — Invoice Discrepancy Management

Parent: [spec.md](spec.md)

## Functional Requirements

- **FR-001**: Clinic entity MUST have 3 new fields: `discrepancyThreshold` (decimal 18,2), `clinicFavorableAccount` (relation to Account), `vendorFavorableAccount` (relation to Account).
- **FR-002**: All 3 discrepancy fields are ALWAYS mandatory on clinic save. Clinic save is blocked if any field is missing, with a field-specific validation message. When multiple fields are missing, all messages are displayed simultaneously. The blocking at accounting time (US8) is a safety net for legacy clinics created before this feature.
- **FR-003**: Invoice screen MUST display: system amount, vendor amount, discrepancy value (signed), discrepancy direction, and motif (conditional — read-only, visible only after comptabilisation).
- **FR-004**: All displayed amounts MUST include the (Devise) label derived from the clinic's country.
- **FR-005**: Confirmation popup MUST appear for any non-zero discrepancy before comptabilisation (unless a blocking rule already prevents it).
- **FR-006**: Reason field MUST be mandatory for vendor-favorable discrepancies, optional for clinic-favorable.
- **FR-007**: Vendor-favorable discrepancies exceeding the threshold MUST be blocked with a specific message (no popup). Discrepancy exactly equal to threshold is authorized.
- **FR-008**: Missing/incomplete clinic config MUST block comptabilisation with a field-specific message.
- **FR-009**: Accounting entries MUST balance (total debits = total credits).
- **FR-010**: Vendor account (4411/4481) MUST always use FRS_TTC.
- **FR-011**: Discrepancy confirmation data MUST be auditable (see Audit Data Structure below).
- **FR-012**: Zero-discrepancy invoices MUST follow the existing standard flow unchanged.
- **FR-013**: Payment schedule amount MUST use FRS_TTC (Devise).
- **FR-014**: Accounting entries tab MUST display the generated lines per the applied rule.
- **FR-015**: Bulk comptabilisation MUST exclude invoices with non-zero discrepancies (individual confirmation required). The result MUST report: "X comptabilisée(s), Y ignorée(s) — écart détecté, confirmation individuelle requise." Invoices blocked by other rules (missing config, invalid accounts) are also excluded and reported separately.
- **FR-016**: If the discrepancy account configured on the clinic is no longer usable at comptabilisation time (deleted, inactive, or not selectable), comptabilisation MUST be blocked. No popup, no entries generated. Message: "Le compte d'écart de la clinique [Nom clinique] est invalide. Merci de mettre à jour le paramétrage avant comptabilisation."

## Non-Functional Requirements

- **NFR-001**: Comptabilisation MUST be atomic — concurrent attempts on the same invoice must be serialized (optimistic lock or similar).

## Audit Data Structure

For any invoice with a discrepancy, the system MUST persist enough data to
reconstruct the decision and the rule applied. A new audit entry is appended
on each comptabilisation; previous entries are preserved (append-only).

| Category | Data |
|----------|------|
| Context identifiers | Invoice, clinic, user, date and time of action |
| Amounts | SYS_TTC (Devise), FRS_TTC (Devise), Discrepancy (Devise), authorized threshold (Devise) |
| Qualification | Discrepancy direction + applied scenario |
| Config used | Clinic-favorable account or vendor-favorable account (whichever applies) |
| User decision | Confirmation / cancellation, with optional motif |
| Result | Comptabilisé / blocked / cancelled |

**Storage**: Audit data is stored as structured fields on the Invoice entity
(no separate audit entity). Fields: `discrepancyMotif`, `discrepancyDecision`,
`discrepancyDecisionAt`, `discrepancyDecisionBy`, `discrepancyScenario`.

## Key Entities

- **Clinic** (existing): Extended with `discrepancyThreshold`, `clinicFavorableAccount`, `vendorFavorableAccount`
- **Invoice** (existing): Already has `amountDifference` and `calculatedTotalInclTax` — these map to the discrepancy and SYS_TTC concepts. Extended with audit fields above.
- **Account** (existing): Referenced by the two new clinic account fields
