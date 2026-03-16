# Feature Specification: Invoice Discrepancy Management

**Jira Epic**: `ACHAT-2145`
**Feature Branch**: `ACHAT-2145-invoice-discrepancy`
**Created**: 2026-03-12
**Status**: Draft
**Target**: both
**Input**: PO functional specification "Gestion des écarts de facture APP vs FRS" v1.0 (2026-03-11)

## Context

The current `AccountingService` applies a hard-coded 0.01 tolerance on the
discrepancy between the system-calculated invoice total (from receipt products)
and the vendor invoice total. Any discrepancy above 0.01 blocks
comptabilisation entirely with "Ecart de montant entre la facture fournisseur
et la facture système. Comptabilisation impossible."

This feature replaces that binary block with a clinic-level parameterized
discrepancy management system that:
- Authorizes comptabilisation of clinic-favorable discrepancies (no threshold), subject to user confirmation
- Authorizes comptabilisation of vendor-favorable discrepancies within a configured threshold, subject to user confirmation with mandatory motif
- Blocks vendor-favorable discrepancies above the threshold
- Posts the discrepancy to configurable accounting accounts

## Business Definitions

| Term | Definition |
|------|-----------|
| SYS_TTC (Devise) | System-calculated total incl. tax from receipt products attached to the invoice |
| FRS_TTC (Devise) | Vendor invoice total incl. tax (the `totalAmount` field) |
| Discrepancy (Devise) | `FRS_TTC - SYS_TTC` (signed value) |
| Clinic-favorable | FRS_TTC < SYS_TTC (vendor charges less than expected) |
| Vendor-favorable | FRS_TTC > SYS_TTC (vendor charges more than expected) |
| Threshold | Max vendor-favorable discrepancy allowed, per clinic |
| (Devise) | The invoice's currency code (MAD, SAR, etc.). All amounts, thresholds, and comparisons use the invoice's currency. The clinic's reference country currency is only a default fallback when no invoice context exists. |

## Jira Story Mapping

| Jira Key | Jira Summary | Spec Story |
|----------|-------------|------------|
| ACHAT-2172 | Paramétrer les règles de gestion des écarts au niveau de la clinique | [US1](stories/US1-clinic-config.md) |
| ACHAT-2150 | Affichage synthétique de l'écart dans l'écran facture | [US2](stories/US2-display.md) |
| ACHAT-2148 | Demander une confirmation utilisateur en cas d'écart | [US3](stories/US3-popup.md) |
| ACHAT-2173 | Comptabiliser automatiquement un écart favorable à la clinique | [US4](stories/US4-clinic-favorable.md) |
| ACHAT-2161 | Comptabiliser automatiquement un écart favorable au fournisseur dans le seuil autorisé | [US5](stories/US5-vendor-within-threshold.md) |
| ACHAT-2162 | Bloquer la comptabilisation si l'écart favorable au fournisseur dépasse le seuil autorisé | [US6](stories/US6-vendor-above-threshold.md) |
| ACHAT-2147 | Comptabiliser le compte fournisseur sur la base de la facture fournisseur | [US7](stories/US7-vendor-account-rule.md) |

**Not in Jira** (regression safety / implied by other stories):
- [US8](stories/US8-missing-config.md) - Blocking on Missing/Incomplete Clinic Config (covered by ACHAT-2162 acceptance criteria)
- [US9](stories/US9-zero-discrepancy.md) - Zero Discrepancy Pass-Through

## Processing Order

The accounting engine MUST follow this strict 9-step execution order to avoid
ambiguous behavior. No step can be skipped or reordered.

| Step | Processing |
|------|-----------|
| 1. Calculate amounts | Compute SYS_TTC (Devise), FRS_TTC (Devise), Discrepancy (Devise), and discrepancy direction. |
| 2. Zero discrepancy | If Discrepancy = 0, apply the existing standard accounting flow. Stop. |
| 3. Verify config | If a discrepancy exists, verify the clinic has a discrepancy configuration. |
| 4. Blocking checks | Check config completeness, account validity, and threshold breach (vendor-favorable only). |
| 5. Show popup | The popup appears ONLY if the invoice remains accountable after business rules. |
| 6. Validate motif | If motif is required and absent, refuse confirmation. |
| 7. Generate entries | Generate the accounting entry for the applicable scenario. |
| 8. Balance check | Reject any unbalanced entry. |
| 9. Update status | If all checks pass, transition invoice to Comptabilisé. |

## Edge Cases

- What if the clinic's country currency changes after discrepancy config is set? Currency is derived from the clinic's country — no manual currency field to go stale.
- What if threshold is 0 and discrepancy is vendor-favorable? Any vendor-favorable discrepancy (even 0.01) is blocked.
- What if the discrepancy account configured on the clinic is no longer usable at comptabilisation time (deleted, inactive, or not selectable)? Comptabilisation is blocked. No popup, no entries generated. Message: "Le compte d'écart de la clinique [Nom clinique] est invalide. Merci de mettre à jour le paramétrage avant comptabilisation."
- What if an invoice is décomptabilisé and re-comptabilisé with a different discrepancy? A new audit entry is appended; the previous entry is preserved.
- What about bulk comptabilisation? Invoices with non-zero discrepancies are excluded from bulk processing (they require individual confirmation via popup). The bulk result reports: "X comptabilisée(s), Y ignorée(s) — écart détecté, confirmation individuelle requise." Invoices blocked by other rules (missing config, invalid accounts) are also excluded and reported separately.

## File Index

- [requirements.md](requirements.md) — Functional requirements, audit structure, key entities
- [success-criteria.md](success-criteria.md) — CT01-CT09, CA01-CA10, measurable outcomes
- [stories/](stories/) — Individual user story files (US1-US9)
