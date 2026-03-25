# Tasks: Multi-Currency MVP

**Input**: Design documents from `/specs/005-multi-currency/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/

**Organization**: Tasks grouped by user story (3 stories from spec.md).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: US1 = PO Multi-Currency, US2 = Invoice Currency, US3 = Accounting Conversion
- Include exact file paths in descriptions

## Path Conventions

- **Backend**: `procurement-api/src/`
- **Frontend**: `procurement-app/src/`

---

## Phase 1: Setup

**Purpose**: New service + migration — shared infrastructure for all stories

- [ ] T001 [API] Create CurrencyApiService in `procurement-api/src/Service/CurrencyApiService.php` — inject HttpClientInterface, implement `getExchangeRate(string $fromCurrency, string $toCurrency, \DateTimeInterface $date): float` that calls `https://cdn.jsdelivr.net/npm/@fawazahmed0/currency-api@{date}/v1/currencies/{from}.json`, extracts rate as `response[fromCurrency][toCurrency]` (e.g., `response.sar.mad` = 2.50), retries once on failure, throws exception if unavailable
- [ ] T002 [API] Create Doctrine migration in `procurement-api/migrations/` — add `currency_code VARCHAR(3) DEFAULT NULL` and `exchange_rate DOUBLE PRECISION DEFAULT NULL` to `purchase_order` and `invoice` tables. Backfill: `currency_code` = clinic country currency, `exchange_rate` = 1. Then set both NOT NULL. Follow pattern from `Version20260305101500.php`.

**Checkpoint**: CurrencyApiService callable, migration runnable. No user-facing changes yet.

---

## Phase 2: Foundational

**Purpose**: Entity changes that all stories depend on

- [ ] T003 [P] [API] Add `currencyCode` field to PurchaseOrder entity in `procurement-api/src/Entity/PurchaseOrder.php` — use `HasCurrencyCode` trait. Add `exchangeRate` float field with ORM Column. Add both to serialization groups: `PurchaseOrderDetail`, `PurchaseOrderListing`, `PurchaseOrderCreate`, `PurchaseOrderUpdate`, `PURCHASE_ORDER_PRINT`, `PURCHASE_ORDER_EXPORT`. Override `getCurrencyCode()` method (line 623) to return `string` (non-nullable): `return $this->currencyCode` — trait returns `?string` but migration ensures NOT NULL.
- [ ] T004 [P] [API] Add `currencyCode` field to Invoice entity in `procurement-api/src/Entity/Invoice.php` — use `HasCurrencyCode` trait. Add `exchangeRate` float field. Add both to serialization groups: `InvoiceListing`, `InvoiceDetail`, `INVOICE_PRINT`, `CreditNoteListing`, `CreditNoteDetail`. Add `getExchangeRate()` / `setExchangeRate()` methods.
- [ ] T005 [API] Modify CurrencyExtension in `procurement-api/src/Doctrine/CurrencyExtension.php` — when an explicit `currencyCode` query parameter is present in the request, use that value instead of `LocaleService::getCurrencyCode()`. This allows PO product selection to filter ProductPricing by PO currency instead of user locale.

**Checkpoint**: Entities have new fields, migration done, CurrencyExtension supports explicit filter. API returns currencyCode + exchangeRate on PO and Invoice responses.

---

## Phase 3: User Story 1 — PO Multi-Currency (Priority: P1) [ACHAT-2103] 🎯 MVP

**Goal**: User selects a currency on PO. System fetches exchange rate. Frontend shows dual-currency columns.

**Independent Test**: Create PO with SAR currency for MAD clinic → rate fetched, dual columns displayed, amounts converted.

### Implementation

- [ ] T006 [US1] [API] Add PO currency resolution logic — in a state processor (API Platform First): on create/update, if `currencyCode` differs from clinic currency, call `CurrencyApiService::getExchangeRate()` with PO `createdAt` date and store result in `exchangeRate`. If same currency, set `exchangeRate = 1`. If API fails after retry, return 422 with message "Taux de change indisponible. Veuillez réessayer." Also handle currency change on existing POs (not just creation).
- [ ] T007 [US1] [API] Handle vendor change on PO — when vendor changes on an existing PO, if `currencyCode` changes as a result, re-fetch exchange rate. Ensure `PurchaseOrderProduct` prices are refreshed (re-read from ProductPricing filtered by new currencyCode).
- [ ] T008 [US1] [APP] Add currency dropdown to PO form in `procurement-app/src/modules/_sharedMapping/PurchaseOrder/Mapping.tsx` — add `currencyCode` field to FORM_VIEW with a select/combobox. Source: distinct `currencyCode` values from Country API (`/countries`). Default: clinic currency. Position: near vendor field.
- [ ] T009 [US1] [APP] Create CurrencyField component in `procurement-app/src/modules/_sharedMapping/PurchaseOrder/fields/CurrencyField.tsx` — handles currency selection, triggers rate fetch via API (handle 422 response: display "Taux de change indisponible. Veuillez réessayer." and revert currency), shows alert dialog (reuse ConfirmationModal pattern from VendorField.tsx) when currency changes with existing products. On confirm: re-fetch ProductPricing for all products in new currency.
- [ ] T010 [US1] [APP] Modify ProductField in `procurement-app/src/modules/_sharedMapping/PurchaseOrder/fields/ProductField.tsx` — add `currencyCode` to the ProductPricing filter (line 110-129). Pass `currencyCode` from form values alongside vendor and status filters. If no pricing found in selected currency, existing behavior applies (grossPrice = 0).
- [ ] T011 [US1] [APP] Add dual-currency columns to PO product grid in `procurement-app/src/modules/_sharedMapping/PurchaseOrder/Mapping.tsx` — when `exchangeRate ≠ 1`, show 2 extra computed columns: "Montant HT (clinic)" = `Math.round(priceExclTax × exchangeRate * 100) / 100`, "Montant TTC (clinic)" = same rounding pattern. Use `CurrencyFormat` component with clinic currency code. Show totals in both currencies at bottom. All computed clinic amounts rounded to 2 decimal places.
- [ ] T012 [US1] [APP] Add currency badge to PO header — when `exchangeRate ≠ 1`, display a read-only badge showing "SAR → MAD (2.5013)" with the rate (4 decimal places for FX precision). Hide when same currency.
- [ ] T013 [US1] [APP] Add i18n keys for multi-currency UI — add translation keys to `procurement-app/src/i18n/fr/` and `procurement-app/src/i18n/en/` for: currency change warning dialog, exchange rate unavailable message, clinic currency column headers.

**Checkpoint**: Foreign-currency PO can be created with dual-column display. Same-currency POs unchanged.

---

## Phase 4: User Story 2 — Invoice Currency & Rate (Priority: P2) [ACHAT-2104]

**Goal**: Invoice stores PO's currency and fetches its own rate at creation time.

**Independent Test**: Create invoice from SAR PO → invoice has currencyCode = SAR, exchangeRate = fresh API rate.

### Implementation

- [ ] T014 [US2] [API] Add invoice currency/rate auto-population — in a Doctrine listener or state processor for Invoice: on prePersist, read `currencyCode` from associated PO (`$invoice->getPurchaseOrder()->getCurrencyCode()`). If foreign currency, call `CurrencyApiService::getExchangeRate()` with invoice `createdAt` date. If same currency, set `exchangeRate = 1`. If API fails, return 422.
- [ ] T015 [US2] [APP] Display currency + rate on invoice view — in invoice detail/listing views, show `currencyCode` and `exchangeRate` as read-only when `exchangeRate ≠ 1`. No edit capability. Modify relevant Mapping files in `procurement-app/src/modules/_sharedMapping/` for RegularInvoice and CreditNote.

**Checkpoint**: Invoice inherits PO currency, fetches own rate. Displayed read-only in frontend.

---

## Phase 5: User Story 3 — Accounting in Clinic Currency (Priority: P3) [ACHAT-2106]

**Goal**: AccountingService converts amounts to clinic currency before sending to Sage.

**Independent Test**: Trigger accounting on SAR invoice → Sage receives MAD amounts + MAD currencyCode.

### Implementation

- [ ] T016 [US3] [API] Modify AccountingService in `procurement-api/src/Service/AccountingService.php` — change line 222: `"currencyCode"` must send `$invoice->getClinic()->getCity()->getCountry()->getCurrencyCode()` instead of PO's getCurrencyCode(). Multiply all amount values in the entries array by `$invoice->getExchangeRate()` before sending. Apply to: line 137 (invoiceAmount), lines 172-194 (all entry amounts), lines 452-531 (payment term amounts). Update line 100 error message to use clinic currency.
- [ ] T017 [US3] [API] Verify PdfController in `procurement-api/src/Controller/PdfController.php` — confirm that `getCurrencyCode()` calls (lines 96, 131, 176, 228, 274, 319) now correctly display vendor currency on PDFs. No code change expected — just verify the stored field returns vendor currency as intended.

**Checkpoint**: Sage receives clinic currency. PDFs display vendor currency. Accounting format identical to today for same-currency invoices.

---

## Phase 6: Polish & Cross-Cutting

**Purpose**: Final validation across all stories

- [ ] T018 [API] Update dashboard queries to use `SUM(amount * exchangeRate)` instead of `SUM(amount)` for all monetary aggregations on PurchaseOrder and Invoice — search for dashboard/reporting queries in `procurement-api/src/` that aggregate PO or Invoice totals and ensure they multiply by `exchangeRate` for clinic currency conversion.
- [ ] T019 Verify ReceiptProduct.getCurrencyCode() in `procurement-api/src/Entity/ReceiptProduct.php` (line 324) — ensure it correctly reads from PO's stored currencyCode field via the chain.
- [ ] T020 Run migration on staging and verify existing POs/invoices display correctly with backfilled currencyCode + exchangeRate = 1.
- [ ] T021 End-to-end test: create PO in SAR → add products → create receipt → create invoice → trigger accounting → verify Sage receives MAD amounts.
- [ ] T022 Verify dashboard totals show correct clinic currency amounts after multi-currency POs exist.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies — start immediately
- **Phase 2 (Foundational)**: Depends on T001 (CurrencyApiService) and T002 (migration)
- **Phase 3 (US1 - PO)**: Depends on Phase 2 completion. API tasks (T006-T007) before APP tasks (T008-T013).
- **Phase 4 (US2 - Invoice)**: Depends on Phase 2 (entity fields). Can run in parallel with Phase 3 APP tasks.
- **Phase 5 (US3 - Accounting)**: Depends on Phase 2 (entity fields) + T014 (invoice rate logic).
- **Phase 6 (Polish)**: Depends on all stories complete.

### User Story Dependencies

- **US1 (PO)**: Independent after Phase 2. Core story — delivers the most value.
- **US2 (Invoice)**: Needs PO entity changes from Phase 2. Can start API work in parallel with US1 frontend.
- **US3 (Accounting)**: Needs invoice exchangeRate (T014). Sequential after US2 API work.

### Parallel Opportunities

```
Phase 2: T003 ∥ T004 (PO entity + Invoice entity — different files)

Phase 3: T006-T007 (API) then T008-T013 (APP — all frontend, can partially overlap)
          T008 ∥ T009 ∥ T013 (different new files)

Phase 4: T014 (API) can start while Phase 3 APP tasks are in progress
          T015 (APP) after T014
```

---

## Implementation Strategy

### MVP First (US1 Only)

1. Complete Phase 1: Setup (T001-T002)
2. Complete Phase 2: Foundational (T003-T005)
3. Complete Phase 3: US1 PO Multi-Currency (T006-T013)
4. **STOP and VALIDATE**: Create SAR PO, verify dual columns, verify same-currency PO unchanged
5. Deploy to staging

### Full MVP (US1 + US2 + US3)

1. Setup + Foundational → Phase 1-2
2. PO Multi-Currency → Phase 3 → test independently
3. Invoice Currency → Phase 4 → test independently
4. Accounting Conversion → Phase 5 → test independently
5. Polish → Phase 6 → end-to-end validation
6. Deploy

---

## Notes

- [P] tasks = different files, no dependencies
- [API] = `procurement-api/`, [APP] = `procurement-app/`
- All existing amount fields remain in vendor currency
- Clinic amounts computed client-side: `amount × exchangeRate`
- Same-currency POs: `exchangeRate = 1`, zero visible change
- Commit after each task
