# Feature Specification: Multi-Currency Procurement Support

**Feature Branch**: `005-multi-currency`
**Target**: both
**Created**: 2026-03-19
**Updated**: 2026-03-24
**Status**: Approved (MVP)
**Jira (active)**: ACHAT-2100, ACHAT-2103, ACHAT-2104, ACHAT-2106
**Jira (closed)**: ACHAT-2099, ACHAT-2101, ACHAT-2102, ACHAT-2108, ACHAT-2109
**Jira (parked)**: ACHAT-2107, ACHAT-2110

## Overview

Add multi-currency support to purchase orders and invoices. A vendor has a country; the PO stores the vendor's currency code and an exchange rate fetched from a global currency API. Invoices inherit both from their PO. All existing amount fields remain in vendor currency. Clinic currency amounts are computed at read time (`amount × exchangeRate`) — no companion fields stored.

**5 new database fields. 0 new entities. 0 new settings screens.**

### Design Principles

1. **Vendor-facing = vendor currency** — PO amounts, invoice amounts, PDFs, Sage accounting all use vendor currency.
2. **Internal = clinic currency** — dashboards, reports, budget tracking compute clinic currency at read time.
3. **No companion fields** — clinic currency is always `amount × exchangeRate`, computed not stored.
4. **Clinic currency = `clinic.city.country.currencyCode`** — not hardcoded MAD.
5. **Same-currency POs = zero change** — `exchangeRate = 1`, no extra UI, identical to today.

### Design Decisions

1. **No Currency entity** — currency codes come from `Country.currencyCode` (ISO 4217).
2. **No ExchangeRate entity** — rates fetched on demand from a global currency API.
3. **No AnnualAverageRate entity** — daily rate at PO creation is more accurate.
4. **No supplier type field** — foreign = `vendor.country.currencyCode ≠ clinic.country.currencyCode`, derived not stored.
5. **Invoice inherits from PO** — no user input for invoice currency/rate.
6. **Rate stored on PO only** — single source of truth. Everything downstream derives from it.
7. **API down** → retry once, then block PO creation. No null rates.

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
| Vendor | `country` | FK nullable → Country | User selects | null (fallback = clinic currency) |
| PurchaseOrder | `currencyCode` | string(3) | `vendor.country.currencyCode`, fallback clinic currency | clinic currency |
| PurchaseOrder | `exchangeRate` | float | Currency API, or 1 if same currency | 1 |
| Invoice | `currencyCode` | string(3) | Inherited from PO | clinic currency |
| Invoice | `exchangeRate` | float | Inherited from PO | 1 |

Migration: all existing records get clinic currency + rate 1. `amount × 1 = amount`. Zero data change.

## User Scenarios & Testing

### Story 1 — Vendor Country (P1) [ACHAT-2100]

Add a country field to the vendor form.

**Target**: both

**Acceptance Scenarios**:

1. **Given** a vendor form, **When** the user selects country "Saudi Arabia", **Then** the country is saved on the vendor.
2. **Given** a vendor without a country, **When** a PO is created, **Then** `currencyCode` = clinic currency, `exchangeRate` = 1.
3. **Given** a vendor with country Saudi Arabia (SAR), **When** a PO is created for a MAD clinic, **Then** `currencyCode` = SAR, rate fetched from API.
4. **Given** a vendor with existing POs in SAR, **When** the country changes, **Then** existing POs keep SAR. New POs use the new currency.

---

### Story 2 — PO Dual-Currency (P2) [ACHAT-2103]

When a PO's vendor currency differs from the clinic currency, show amounts in both currencies.

**Target**: both (core story)

**Currency Resolution**:

1. User selects vendor
2. `currencyCode` = `vendor.country.currencyCode` (fallback: `clinic.city.country.currencyCode`)
3. If same as clinic → `exchangeRate` = 1, no API call, standard mode
4. If different → call currency API with PO creation date, store rate
5. If vendor changes → recalculate both fields

**Backend**:
- Store `currencyCode` + `exchangeRate` on PO
- `getCurrencyCode()` reads from stored field (replaces current clinic derivation)
- API call: `https://cdn.jsdelivr.net/npm/@fawazahmed0/currency-api@{date}/v1/currencies/{vendorCurrency}.json` → extract rate to clinic currency
- If API fails: retry once, then block with message "Taux de change indisponible. Veuillez réessayer."

**Frontend — standard mode** (same currency): zero change from today.

**Frontend — multi-currency mode** (different currencies):
- PO header: badge "SAR → MAD (2.50)" with rate displayed read-only
- Product grid: 2 extra columns per line:
  - Montant HT (clinic currency) — computed: `priceExclTax × exchangeRate`
  - Montant TTC (clinic currency) — computed: `priceInclTax × exchangeRate`
- Totals at bottom in both currencies

**Acceptance Scenarios**:

1. **Given** a SAR vendor + MAD clinic, **When** PO created, **Then** `currencyCode = SAR`, `exchangeRate` = API rate, dual columns shown.
2. **Given** a MAD vendor + MAD clinic, **When** PO created, **Then** `exchangeRate = 1`, standard mode, no extra columns.
3. **Given** a vendor without country, **When** PO created, **Then** clinic currency, rate = 1, standard mode.
4. **Given** PO in SAR with rate 2.50, **When** product added with P.U = 100 SAR, **Then** vendor column = 100 SAR, clinic column = 250 MAD.
5. **Given** PO in SAR, **When** vendor changed to EUR vendor, **Then** `currencyCode` updates to EUR, rate re-fetched, amounts recalculated.
6. **Given** API unavailable (after retry), **When** PO created for SAR vendor, **Then** blocked with message.

---

### Story 3 — Invoice Inherits PO Currency (P3) [ACHAT-2104]

Invoice automatically inherits `currencyCode` and `exchangeRate` from its PO.

**Target**: api (backend only — no UI changes)

**Acceptance Scenarios**:

1. **Given** PO in SAR with rate 2.50, **When** invoice created from this PO, **Then** invoice has `currencyCode = SAR`, `exchangeRate = 2.50` automatically.
2. **Given** PO in MAD with rate 1, **When** invoice created, **Then** invoice has `currencyCode = MAD`, `exchangeRate = 1`.
3. **Given** invoice from SAR PO, **When** user views invoice, **Then** no currency/rate fields editable.

---

### Story 4 — Verify Accounting with Vendor Currency (P4) [ACHAT-2106]

Ensure Sage accounting receives correct vendor currency amounts and code.

**Target**: api (verification + potential fix)

**Current code** (AccountingService.php line 222):
```php
"currencyCode" => $invoice->getPurchaseOrder()?->getCurrencyCode() ?? 'MAD',
```

This already reads from PO. After Story 2, `getCurrencyCode()` returns the stored vendor currency. Amounts are in vendor currency. **Should work as-is** — verify and test.

**Acceptance Scenarios**:

1. **Given** invoice from SAR PO, **When** accounting triggered, **Then** Sage receives SAR amounts + `currencyCode = SAR`.
2. **Given** invoice from MAD PO, **When** accounting triggered, **Then** Sage receives MAD amounts + `currencyCode = MAD` (identical to today).

**Also verify**: PdfController uses `getCurrencyCode()` — PDFs should display vendor currency automatically.

---

### Edge Cases

- **Vendor country change**: existing POs/invoices keep their original currency. Only new POs use updated currency.
- **Vendor without country**: fallback to clinic currency, rate = 1, zero difference from today.
- **API unavailable**: retry once, then block PO creation. No null rates.
- **Same currency**: `exchangeRate = 1`, standard mode, UX identical to today.
- **Rounding**: computed clinic amounts round to 2 decimal places for display.
- **Credit notes**: not in MVP scope. Deferred.
- **`uniteDevise` / rate direction**: the currency API returns a direct rate (e.g., `eur.mad = 10.81`). If PO is in EUR for a MAD clinic: `exchangeRate = 10.81`, clinic amount = `eurAmount × 10.81`. Verify rate direction for each currency pair.

## Functional Requirements

- **FR-001**: Vendor MAY have a country (nullable FK to Country).
- **FR-002**: On PO creation, `currencyCode` = `vendor.country.currencyCode`, fallback clinic currency.
- **FR-003**: On PO creation, if `currencyCode ≠ clinic currency`, fetch rate from currency API and store on PO.
- **FR-004**: If `currencyCode = clinic currency`, `exchangeRate = 1`, no API call.
- **FR-005**: `currencyCode` and `exchangeRate` are read-only for the user.
- **FR-006**: If vendor changes on PO, recalculate `currencyCode` + `exchangeRate`.
- **FR-007**: If API unavailable after retry, block PO creation with explicit message.
- **FR-008**: For multi-currency POs, frontend shows dual columns: vendor amounts (stored) + clinic amounts (computed: `amount × exchangeRate`).
- **FR-009**: For same-currency POs, UX is identical to today.
- **FR-010**: Invoice inherits `currencyCode` + `exchangeRate` from PO automatically. No user input.
- **FR-011**: Sage accounting receives vendor currency amounts + vendor `currencyCode` (existing behavior via `getCurrencyCode()`).
- **FR-012**: Dashboard queries use `SUM(amount * exchangeRate)` for clinic currency aggregation.

## Assumptions

1. Clinic currency = `clinic.city.country.currencyCode`, not hardcoded.
2. Vendor has one currency (from country). One vendor = one currency at a time.
3. PO is the single source of truth for currency + rate. Everything downstream inherits.
4. Exchange rates stored with up to 6 decimal places.
5. Displayed amounts rounded to 2 decimal places.
6. `getCurrencyCode()` on PO changes from derived (clinic path) to stored field. Existing callers (AccountingService, PdfController) work without modification.
7. Currency API (fawazahmed0) is reliable (CDN-served). Fallback: retry + block.

## Success Criteria

- **SC-001**: Foreign-currency PO creation requires zero manual input for currency/rate.
- **SC-002**: Same-currency POs are identical to today — zero UX change.
- **SC-003**: 100% existing data accessible and correct after migration.
- **SC-004**: Saudi vendor PO can be created and invoiced — **unblocks current issue**.
- **SC-005**: No new settings screens or admin configuration required.

## Deferred (post-MVP)

- ACHAT-2107 — Currency statement report
- ACHAT-2110 — Product price currency inheritance
- PurchaseRequest currency awareness
- CreditNote / ReturnOrder currency
- ProductPricing clinic currency display
