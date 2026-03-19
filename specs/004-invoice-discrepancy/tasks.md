# Tasks: Invoice Discrepancy Management (ACHAT-2145)

**Input**: Design documents from `/specs/004-invoice-discrepancy/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/

**Organization**: Tasks are grouped by user story (phases reorganized from plan.md to group by story). Backend accounting engine tasks (US4-US9) are grouped in one phase because they modify the same method (`processInvoiceAccounting()`).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to
- **[API]** / **[APP]**: Target sub-project
- Include exact file paths in descriptions

---

## Phase 1: Foundational — Backend Data Model [API]

**Purpose**: Entity changes, enum, migration. MUST complete before any user story.

- [x] T001 [P] [API] Create `DiscrepancyDirectionEnum` backed enum in `procurement-api/src/Entity/Enumeration/DiscrepancyDirectionEnum.php` with cases: None, ClinicFavorable, VendorFavorable
- [x] T002 [P] [API] Add 3 discrepancy fields to `procurement-api/src/Entity/Clinic.php`: `discrepancyThreshold` (decimal 18,2, nullable), `clinicFavorableAccount` (ManyToOne Account, nullable), `vendorFavorableAccount` (ManyToOne Account, nullable). Add serialization groups: ClinicListing (threshold only), ClinicDetail, ClinicCreate, ClinicUpdate. Do NOT add Assert constraints here — validation is handled entirely in T005.
- [x] T003 [P] [API] Add 4 audit fields to `procurement-api/src/Entity/Invoice.php`: `discrepancyMotif` (string 500, nullable), `discrepancyScenario` (string 50, nullable), `discrepancyDecisionAt` (datetime_immutable, nullable), `discrepancyDecisionBy` (ManyToOne User, nullable). Add read-only serialization group: InvoiceDetail.
- [x] T004 [API] Generate and review Doctrine migration: `docker compose exec php php bin/console doctrine:migrations:diff`. Verify SQL matches data-model.md schema. Run: `docker compose exec php php bin/console doctrine:migrations:migrate`

**Checkpoint**: Entities ready. Clinic API exposes 3 new fields. Invoice API exposes 4 audit fields.

---

## Phase 2: US1 — Clinic Discrepancy Configuration `ACHAT-2172` (Priority: P1) — MVP

**Goal**: Clinic administrators can configure discrepancy parameters (threshold + 2 account references) on the clinic form.

**Independent Test**: Create/edit a clinic, fill the 3 discrepancy fields, save — verify fields persist and validation enforces completeness.

### Backend [API]

- [x] T005 [US1] [API] Add ALL clinic validation constraints in `procurement-api/src/Entity/Clinic.php`: `#[Assert\NotNull]` on all 3 fields, `#[Assert\GreaterThanOrEqual(0)]` on threshold, plus `#[Assert\Callback]` or custom validator to produce field-specific French messages per US1 acceptance scenarios (e.g., "Le paramétrage d'écart de la clinique [Nom] est incomplet : seuil d'écart TTC autorisé manquant."). When multiple fields missing, all messages displayed simultaneously.

### Frontend [APP]

- [x] T006 [P] [US1] [APP] Add 3 fields to Clinic model in `procurement-app/src/modules/_sharedMapping/Clinic/Model.ts`: `discrepancyThreshold` (number), `clinicFavorableAccount` (string IRI | null), `vendorFavorableAccount` (string IRI | null)
- [x] T007 [US1] [APP] Add 3 form fields to Clinic mapping in `procurement-app/src/modules/_sharedMapping/Clinic/Mapping.tsx`: threshold as number input (decimal, min 0), clinicFavorableAccount as resource field (Account dropdown), vendorFavorableAccount as resource field (Account dropdown). Add to both create and update views.
- [x] T008 [P] [US1] [APP] Add i18n keys in `procurement-app/src/i18n/locales/fr/clinic.json` (or appropriate namespace): labels for the 3 discrepancy fields + validation error messages

**Checkpoint**: Clinic form shows 3 discrepancy fields. Save blocked when any is missing. Threshold = 0 accepted.

---

## Phase 3: Accounting Engine — US7, US4, US5, US6, US8, US9 [API] (Priority: P1)

**Goal**: Refactor `AccountingService.processInvoiceAccounting()` to implement the 9-step processing order with all discrepancy scenarios.

**Independent Test**: Comptabilise invoices across all scenarios (zero, clinic-favorable, vendor-within, vendor-above, missing config) — verify correct entries or blocking.

- [x] T009 [US9] [API] Refactor `procurement-api/src/Service/AccountingService.php` — Step 1-2: Extract amount calculation into a dedicated method. If discrepancy = 0, call existing standard flow and return (US9 regression safety). Remove old hard-coded 0.01 tolerance.
- [x] T010 [US8] [API] Refactor `procurement-api/src/Service/AccountingService.php` — Step 3-4 (config checks): When discrepancy != 0, verify clinic has `discrepancyThreshold` + both account relations set. Return 422 with field-specific message if missing/incomplete (US8 messages). Check account validity — return 422 with "compte d'écart invalide" if account is unusable (FR-016).
- [x] T011 [US6] [API] Refactor `procurement-api/src/Service/AccountingService.php` — Step 4 (threshold check): If vendor-favorable AND discrepancy > threshold, return 422 with threshold-exceeded message including amounts in (Devise). No popup, no entries.
- [x] T012 [US4] [API] Refactor `procurement-api/src/Service/AccountingService.php` — Step 7 (clinic-favorable entries): When FRS_TTC < SYS_TTC, generate entries: debit charge accounts for SYS_TTC, credit vendor account for FRS_TTC, credit clinic-favorable discrepancy account for the difference.
- [x] T013 [US5] [API] Refactor `procurement-api/src/Service/AccountingService.php` — Step 7 (vendor-favorable entries): When FRS_TTC > SYS_TTC AND discrepancy <= threshold, generate entries: debit charge accounts for SYS_TTC, debit vendor-favorable discrepancy account for difference, credit vendor account for FRS_TTC.
- [x] T014 [US7] [API] Verify in `procurement-api/src/Service/AccountingService.php` — Step 8 (balance check): Ensure total debits = total credits for both discrepancy scenarios. Verify vendor account always uses `totalAmount` (= FRS_TTC). Verify `buildPaymentTerms()` already uses `totalAmount` (= FRS_TTC) for payment schedule (FR-013). If any check fails, fix the logic. Add a runtime assertion as safety net (FR-009).
- [x] T015 [US3] [API] Update `procurement-api/src/Controller/AccountingController.php`: accept optional `discrepancyMotif` string in request body for `POST /custom/accounting/{idInvoice}`. Pass motif to `AccountingService`. Validate motif required when vendor-favorable (FR-006) — return 422 with "Le motif de comptabilisation de l'écart est obligatoire lorsque l'écart est favorable au fournisseur."
- [x] T016 [API] Refactor `procurement-api/src/Service/AccountingService.php` — Step 9 (audit + status): On successful comptabilisation with non-zero discrepancy, persist audit fields on Invoice: `discrepancyMotif`, `discrepancyScenario` (e.g., "clinic_favorable" or "vendor_within_threshold"), `discrepancyDecisionAt` (now), `discrepancyDecisionBy` (current user). Set `accounted = true` as before.

**Checkpoint**: Backend handles all 9 processing steps. CT01-CT08 scenarios work via API.

---

## Phase 4: US2 — Discrepancy Display on Invoice Screen `ACHAT-2150` (Priority: P1) [APP]

**Goal**: Invoice screen shows discrepancy summary (system amount, vendor amount, discrepancy, direction, motif).

**Independent Test**: Open an invoice with FRS_TTC != SYS_TTC — verify all fields displayed with correct values.

- [x] T017 [P] [US2] [APP] Add audit fields to Invoice model in `procurement-app/src/modules/_sharedMapping/Invoice/Model.ts`: `discrepancyMotif` (string | null), `discrepancyScenario` (string | null), `discrepancyDecisionAt` (string | null), `discrepancyDecisionBy` (string IRI | null)
- [x] T018 [US2] [APP] Add discrepancy summary section to Invoice detail view in `procurement-app/src/modules/_sharedMapping/Invoice/Mapping.tsx`: display system amount (`calculatedTotalInclTax`), vendor amount (`totalAmount`), discrepancy (`amountDifference`, signed), direction label (derived: Nul / Favorable clinique / Favorable fournisseur). Add visual indicator (badge/color) for discrepancy status. Show motif (read-only) when present, hide when no discrepancy. All amounts with currency format using PO's `currencyCode`.
- [x] T019 [P] [US2] [APP] Add i18n keys in `procurement-app/src/i18n/locales/fr/invoice.json`: labels for discrepancy display section (montant_systeme, montant_fournisseur, ecart, sens_ecart, motif), direction values (nul, favorable_clinique, favorable_fournisseur)

**Checkpoint**: Invoice screen shows discrepancy summary. Motif hidden before comptabilisation. Currency labels correct.

---

## Phase 5: US3 — Confirmation Popup + Frontend Accounting Flow `ACHAT-2148` (Priority: P1) [APP]

**Goal**: Confirmation popup appears for non-zero discrepancies. Motif mandatory for vendor-favorable. Blocking messages for threshold/config/account issues.

**Independent Test**: Click Comptabiliser on a discrepancy invoice — verify popup, motif validation, cancel behavior, blocking messages.

- [x] T020 [US3] [APP] Create `procurement-app/src/modules/_sharedMapping/Invoice/components/DiscrepancyConfirmDialog.tsx`: Dialog component (not ConfirmationModal) displaying: discrepancy amount (absolute value), system amount, vendor amount, direction label, Textarea for motif, Confirmer/Annuler buttons. Motif validation: required if vendor-favorable, optional if clinic-favorable. Error message on empty mandatory motif. All amounts with CurrencyFormat using PO currency. Use exact popup template wording from US3.
- [x] T021 [US3] [APP] Refactor `procurement-app/src/modules/_sharedMapping/Invoice/components/AccountingButton.tsx`: Before calling API, detect non-zero discrepancy from invoice data (`amountDifference != 0`). Compute direction from sign. If discrepancy exists: show `DiscrepancyConfirmDialog` instead of simple `ConfirmationModal`. On confirm: send POST with `{ discrepancyMotif }` in body. On cancel: close dialog, no API call. Handle 422 responses: display blocking messages as toast (threshold exceeded, missing config, invalid account, missing motif). Zero discrepancy: use existing simple confirmation flow unchanged (US9).
- [x] T022 [P] [US3] [APP] Add i18n keys in `procurement-app/src/i18n/locales/fr/invoice.json`: popup title, popup description template, motif label, motif required error, blocking error messages (threshold exceeded, missing config, incomplete config, invalid account)

**Checkpoint**: Full accounting flow works end-to-end. Popup shows for discrepancies. Blocking messages for above-threshold/missing config. Zero discrepancy unchanged.

---

## Phase 6: FR-015 — Bulk Exclusion [API + APP]

**Goal**: Bulk comptabilisation excludes invoices with non-zero discrepancies, reporting excluded count.

**Independent Test**: Run bulk on mixed invoices — verify discrepancy invoices excluded and reported.

- [x] T023 [P] [API] Update `procurement-api/src/Controller/AccountingController.php` bulk endpoint: detect non-zero discrepancy per invoice before calling service. Add `excluded` status to response with reason `discrepancy_requires_confirmation`. Return enhanced summary: `{totalProcessed, success, failed, excluded}`. Note: frontend also filters client-side (T024) for UX speed; this backend check is a safety net.
- [x] T024 [P] [APP] Update `procurement-app/src/modules/_sharedMapping/Invoice/components/AccountingMass/useBulkAccounting.ts`: filter out invoices with `amountDifference != 0` before sending to bulk endpoint for UX speed (backend also validates as safety net — T023). Show excluded count in result summary. Update `BulkAccountingModal.tsx` to display excluded invoices message: "X comptabilisée(s), Y ignorée(s) — écart détecté, confirmation individuelle requise."
- [x] T025 [P] [APP] Add i18n keys in `procurement-app/src/i18n/locales/fr/invoice.json` for bulk exclusion message

**Checkpoint**: Bulk skips discrepancy invoices. Result shows success/excluded counts.

---

## Phase 7: Polish & Cross-Cutting

**Purpose**: Verify all scenarios, clean up.

- [ ] T026 Verify CT01-CT09 test cases pass end-to-end (manual or automated)
- [ ] T027 Verify CA01-CA10 acceptance criteria are met
- [x] T028 Verify accounting entries tab displays generated lines correctly (FR-014) in `procurement-app/src/modules/_sharedMapping/Invoice/Mapping.tsx` — the existing `UnifiedAccountingTable` component should already show the new discrepancy entries. Verify and adjust if needed.
- [x] T029 Verify concurrent comptabilisation is serialized (NFR-001) in `procurement-api/src/Service/AccountingService.php` — check for optimistic lock or DB-level serialization. Add safeguard if missing.
- [ ] T030 Run quickstart.md verification checklist

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Foundational)**: No dependencies — start immediately
- **Phase 2 (US1)**: Depends on Phase 1 (entity fields must exist)
- **Phase 3 (Accounting Engine)**: Depends on Phase 1 (entity fields) + Phase 2 (clinic validation)
- **Phase 4 (US2 Display)**: Depends on Phase 1 (audit fields on Invoice)
- **Phase 5 (US3 Popup)**: Depends on Phase 3 (backend engine) + Phase 4 (display section)
- **Phase 6 (Bulk)**: Depends on Phase 3 (engine must handle discrepancies)
- **Phase 7 (Polish)**: Depends on all phases

### Parallel Opportunities

```
Phase 1 ──┬──▶ Phase 2 (US1) ──▶ Phase 3 (Engine) ──▶ Phase 5 (Popup) ──▶ Phase 7
           │                                      ╲
           ├──▶ Phase 4 (US2 Display) ─────────────╲──▶ Phase 5
           │                                        ╲
           └──────────────────────────────────────────▶ Phase 6 (Bulk)
```

- **T001, T002, T003** can run in parallel (different files)
- **T006, T008** can run in parallel with T005 (different projects)
- **T017, T019** can run in parallel (different files)
- **T023, T024, T025** can all run in parallel (different files/projects)
- Phase 4 (Display) can start as soon as Phase 1 completes, in parallel with Phase 2/3

---

## Implementation Strategy

### MVP (Phase 1 + 2 + 3)

1. Complete Phase 1: Entity fields + migration
2. Complete Phase 2: US1 — clinic config form works
3. Complete Phase 3: Accounting engine handles all scenarios
4. **STOP and VALIDATE**: Clinic config + accounting logic works via API
5. Deploy backend to staging

### Incremental Frontend (Phase 4 + 5 + 6)

6. Phase 4: Invoice display shows discrepancy info
7. Phase 5: Popup + full frontend flow
8. Phase 6: Bulk exclusion
9. Deploy frontend to staging
10. QA validation against CT01-CT09

---

## Notes

- [P] tasks = different files, no dependencies
- [API] / [APP] = target sub-project (branches in procurement-api / procurement-app)
- Each phase = one or more PRs
- All blocking messages are French (per spec wording)
- Currency = invoice's currency (PO.currencyCode), not clinic's
