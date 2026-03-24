# Feature Specification: Multi-Currency Procurement Support

**Feature Branch**: `005-multi-currency`
**Target**: both
**Created**: 2026-03-19
**Updated**: 2026-03-24
**Status**: Approved (MVP)
**Jira (active)**: ACHAT-2103, ACHAT-2104, ACHAT-2106
**Jira (deferred)**: ACHAT-2100, ACHAT-2107, ACHAT-2110
**Jira (closed)**: ACHAT-2099, ACHAT-2101, ACHAT-2102, ACHAT-2108, ACHAT-2109

## Overview

Add multi-currency support to purchase orders and invoices. The user selects a currency on the PO. The exchange rate is fetched automatically from a global currency API. The invoice gets its own rate at creation time. Accounting always receives clinic currency amounts — Sage sees no difference from today.

**4 new database fields. 0 new entities. 0 new settings screens.**

**Why MVP**: The team is blocked — can't invoice a Saudi vendor (SAR) because the system only supports clinic currency. The original PO spec had 11 stories and 3 new entities. After analysis, 8 stories were eliminated or deferred. This MVP unblocks the immediate need with minimal scope, and scales later using the same pattern.

### Design Principles

1. **Vendor-facing = vendor currency** — PO amounts, invoice amounts, PDFs use vendor currency.
2. **Sage = clinic currency** — AccountingService converts before sending. Sage always receives clinic currency.
3. **No companion fields** — clinic currency amounts are computed at read time (`amount × exchangeRate`).
4. **Clinic currency = `clinic.city.country.currencyCode`** — not hardcoded MAD.
5. **Same-currency POs = zero change** — `exchangeRate = 1`, no extra UI, identical to today.

### Design Decisions

1. **No Currency entity** — currency codes come from `Country.currencyCode` (ISO 4217).
2. **No ExchangeRate entity** — rates fetched on demand from a global currency API.
3. **No AnnualAverageRate entity** — daily rate at document creation is more accurate.
4. **No vendor country needed for MVP** — user selects currency directly on the PO.
5. **Invoice gets its own rate** — fetched from API at invoice creation time, not inherited from PO. More accurate for accounting.
6. **Sage always receives clinic currency** — AccountingService converts `amount × exchangeRate` before sending. Sage format unchanged.
7. **API down** → retry once, then block. No null rates.

### Currency API

Global API (supports any currency pair, historical rates, MAD included):

```
Latest:     https://cdn.jsdelivr.net/npm/@fawazahmed0/currency-api@latest/v1/currencies/{code}.json
Historical: https://cdn.jsdelivr.net/npm/@fawazahmed0/currency-api@{YYYY-MM-DD}/v1/currencies/{code}.json
```

- No API key required
- No rate limits (CDN-served)
- Free, open source
- Response: `{"date":"2026-03-23","eur":{"mad":10.80953137}}`

## New Database Fields

| Entity | Field | Type | Source | Migration |
|--------|-------|------|--------|-----------|
| PurchaseOrder | `currencyCode` | string(3) | User selects, default = clinic currency | clinic currency |
| PurchaseOrder | `exchangeRate` | float | Currency API, or 1 if same currency | 1 |
| Invoice | `currencyCode` | string(3) | Same as PO currency | clinic currency |
| Invoice | `exchangeRate` | float | Currency API at invoice creation date | 1 |

Migration: all existing records get clinic currency + rate 1. `amount × 1 = amount`. Zero data change.

## User Scenarios & Testing

### Story 1 — PO Multi-Currency (P1) [ACHAT-2103]

User selects a currency on the PO. System fetches the exchange rate. Amounts display in both vendor and clinic currencies.

**Target**: both (core story)

**PO Currency Selection**:

1. User creates PO, selects vendor
2. User selects currency from dropdown (source: distinct `currencyCode` from Country table, default = clinic currency)
3. If same as clinic → `exchangeRate` = 1, no API call, standard mode
4. If different → call currency API, store rate
5. Currency field stays **editable** — if changed after products are added, alert dialog: "Changer la devise va mettre à jour les prix de tous les produits. Continuer?"
6. On confirm → re-fetch ProductPricing for each product in the new currency + re-fetch exchange rate
7. If a product has no ProductPricing in the new currency → flag or remove

**Product Selection**:
- When adding products, filter ProductPricing by `vendor = PO vendor` AND `currencyCode = PO currency`
- Only prices matching the PO currency are shown

**Backend**:
- Store `currencyCode` + `exchangeRate` on PO
- `getCurrencyCode()` reads from stored field (replaces current clinic derivation)
- If API fails: retry once, then block with message "Taux de change indisponible. Veuillez réessayer."

**Frontend — standard mode** (same currency): zero change from today.

**Frontend — multi-currency mode** (different currencies):
- PO header: currency dropdown + rate displayed read-only + badge "SAR → MAD (2.50)"
- Product grid: 2 extra columns per line:
  - Montant HT (clinic currency) — computed: `priceExclTax × exchangeRate`
  - Montant TTC (clinic currency) — computed: `priceInclTax × exchangeRate`
- Totals at bottom in both currencies

**Acceptance Scenarios**:

1. **Given** PO with currency SAR and MAD clinic, **When** PO created, **Then** `currencyCode = SAR`, `exchangeRate` = API rate, dual columns shown.
2. **Given** PO with currency MAD and MAD clinic, **When** PO created, **Then** `exchangeRate = 1`, standard mode, no extra columns.
3. **Given** PO in SAR with rate 2.50, **When** product added with P.U = 100 SAR, **Then** vendor column = 100 SAR, clinic column = 250 MAD.
4. **Given** PO in SAR with products, **When** user changes currency to EUR, **Then** alert dialog shown. On confirm: ProductPricing refreshed to EUR, rate re-fetched.
5. **Given** PO in EUR, **When** product added but no EUR ProductPricing exists for that product, **Then** product flagged or not selectable.
6. **Given** API unavailable (after retry), **When** non-clinic currency selected, **Then** blocked with message.

---

### Story 2 — Invoice Currency & Rate (P2) [ACHAT-2104]

Invoice stores the PO's currency but fetches its own exchange rate at creation time.

**Target**: api (backend) + minimal frontend

**Logic**:
1. Invoice created from PO receipts
2. `currencyCode` = PO's `currencyCode` (same vendor, same currency)
3. `exchangeRate` = fresh API call for invoice creation date
4. If `currencyCode` = clinic currency → `exchangeRate` = 1, no API call
5. If API fails → retry once, then block invoice creation

**Frontend**: display currency + rate as read-only on invoice if different from clinic currency. No edit.

**Acceptance Scenarios**:

1. **Given** PO in SAR, **When** invoice created, **Then** invoice has `currencyCode = SAR`, `exchangeRate` = API rate for today (may differ from PO rate).
2. **Given** PO in MAD, **When** invoice created, **Then** `currencyCode = MAD`, `exchangeRate = 1`.
3. **Given** invoice in SAR, **When** user views invoice, **Then** currency and rate displayed as read-only.

---

### Story 3 — Accounting in Clinic Currency (P3) [ACHAT-2106]

AccountingService converts invoice amounts to clinic currency before sending to Sage.

**Target**: api (code change in AccountingService)

**Current code** (AccountingService.php line 222):
```php
"currencyCode" => $invoice->getPurchaseOrder()?->getCurrencyCode() ?? 'MAD',
```

**Change needed**:
```
"currencyCode" => clinic.city.country.currencyCode
amounts → amount × invoice.exchangeRate
```

Sage always receives clinic currency + clinic currencyCode. Format identical to today. Multi-currency is invisible to Sage.

**Acceptance Scenarios**:

1. **Given** invoice in SAR with rate 2.50 and total 100 SAR, **When** accounting triggered, **Then** Sage receives `currencyCode = MAD` and `amount = 250 MAD`.
2. **Given** invoice in MAD with rate 1, **When** accounting triggered, **Then** identical to today.
3. **Given** PO rate = 2.50 and invoice rate = 2.60 for same currency, **When** accounting triggered, **Then** Sage uses invoice rate (2.60), not PO rate. No exchange difference entry.

**Also verify**: PdfController uses `getCurrencyCode()` — PDFs should display vendor currency (correct behavior for vendor-facing documents).

---

### Edge Cases

- **Currency change on PO with products**: alert dialog → re-fetch pricing + rate. Products without pricing in new currency are flagged.
- **No ProductPricing in selected currency**: product not available for this PO.
- **PO rate ≠ invoice rate**: expected and correct. PO rate is for PO-time display. Invoice rate is for accounting. No exchange difference entry — Sage just gets the invoice-rate conversion.
- **API unavailable**: retry once, then block. No null rates.
- **Same currency**: `exchangeRate = 1`, standard mode, UX identical to today.
- **Rounding**: computed clinic amounts round to 2 decimal places for display.
- **Credit notes**: not in MVP scope. Deferred.
- **Rate direction**: the currency API returns a direct rate (e.g., `sar.mad = 2.50`). Clinic amount = `vendorAmount × rate`. Verify direction per currency pair.

## Functional Requirements

- **FR-001**: PO MUST have a currency selector (dropdown from Country.currencyCode values, default = clinic currency).
- **FR-002**: PO `currencyCode` stays editable. If changed after products exist, alert dialog + re-fetch pricing and rate.
- **FR-003**: On PO, if `currencyCode ≠ clinic currency`, fetch rate from currency API and store on PO.
- **FR-004**: If `currencyCode = clinic currency`, `exchangeRate = 1`, no API call.
- **FR-005**: Product selection on PO filters ProductPricing by `vendor + currencyCode`.
- **FR-006**: If API unavailable after retry, block currency selection with explicit message.
- **FR-007**: For multi-currency POs, frontend shows dual columns: vendor amounts (stored) + clinic amounts (computed: `amount × exchangeRate`).
- **FR-008**: For same-currency POs, UX is identical to today.
- **FR-009**: Invoice `currencyCode` = PO's currency. Invoice `exchangeRate` = fresh API call at invoice creation date.
- **FR-010**: AccountingService MUST convert amounts to clinic currency (`amount × invoice.exchangeRate`) and send clinic `currencyCode` to Sage.
- **FR-011**: Sage MUST NOT receive vendor currency amounts or vendor currencyCode.
- **FR-012**: No exchange difference entries in Sage.
- **FR-013**: Dashboard queries use `SUM(amount * exchangeRate)` for clinic currency aggregation.

## Assumptions

1. Clinic currency = `clinic.city.country.currencyCode`, not hardcoded.
2. PO currency is user-selected, not auto-derived from vendor.
3. Invoice currency = PO currency. Invoice rate = API rate at invoice creation date (not PO rate).
4. Exchange rates stored with up to 6 decimal places.
5. Displayed amounts rounded to 2 decimal places.
6. `getCurrencyCode()` on PO reads from stored field. PdfController displays vendor currency (correct for vendor-facing docs).
7. AccountingService converts to clinic currency before sending to Sage.
8. Currency API (fawazahmed0) is reliable (CDN-served). Fallback: retry + block.

## Success Criteria

- **SC-001**: Foreign-currency PO creation requires only one extra user action: selecting the currency.
- **SC-002**: Same-currency POs are identical to today — zero UX change.
- **SC-003**: 100% existing data accessible and correct after migration.
- **SC-004**: Saudi vendor PO can be created and invoiced — **unblocks current issue**.
- **SC-005**: Sage always receives clinic currency — format identical to today.
- **SC-006**: No new settings screens or admin configuration required.

## Deferred (post-MVP)

- ACHAT-2100 — Vendor country (auto-derive currency instead of manual selection)
- ACHAT-2107 — Currency statement report
- ACHAT-2110 — Product price currency inheritance
- PurchaseRequest currency awareness
- CreditNote / ReturnOrder currency
