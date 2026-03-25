# Quickstart: Multi-Currency MVP

## What this feature does

Adds multi-currency support to Purchase Orders and Invoices. Users select a currency on the PO, the system fetches the exchange rate automatically, and displays amounts in both vendor and clinic currencies.

## Architecture at a glance

```
User selects currency on PO
       │
       ▼
Backend fetches rate from currency API (fawazahmed0)
       │
       ▼
PO stores: currencyCode + exchangeRate
       │
       ├──→ Frontend shows dual columns (vendor × rate = clinic)
       │
       ▼
Invoice created → gets own rate from API
       │
       ▼
Accounting → converts amounts × invoice.exchangeRate → sends clinic currency to Sage
```

## Key files to modify

### Backend (procurement-api)

| File | Change |
|------|--------|
| `src/Entity/PurchaseOrder.php` | Add `currencyCode` (trait) + `exchangeRate` field. Change `getCurrencyCode()` to read stored field. |
| `src/Entity/Invoice.php` | Add `currencyCode` (trait) + `exchangeRate` field. |
| `src/Service/CurrencyApiService.php` | **New**. Fetch rate from fawazahmed0 API. Retry logic. |
| `src/Doctrine/CurrencyExtension.php` | Allow explicit `currencyCode` filter to override locale-based filtering. |
| `src/Service/AccountingService.php` | Line 222: send clinic currency. Convert all amounts × `invoice.exchangeRate`. |
| `migrations/VersionXXX.php` | Add columns, backfill, set NOT NULL. |

### Frontend (procurement-app)

| File | Change |
|------|--------|
| `src/modules/_sharedMapping/PurchaseOrder/Mapping.tsx` | Add currency dropdown to form. Add computed clinic amount columns. |
| `src/modules/_sharedMapping/PurchaseOrder/fields/CurrencyField.tsx` | **New**. Currency selector + rate fetch + alert on change. |
| `src/modules/_sharedMapping/PurchaseOrder/fields/ProductField.tsx` | Pass `currencyCode` filter when fetching ProductPricing. |
| `src/modules/_sharedMapping/PurchaseOrder/fields/AmountUnit.tsx` | Already uses `currencyCode` — no change needed. |

## How to test

1. **Same currency PO**: Create PO with MAD currency for MAD clinic → identical to today, no extra columns
2. **Foreign currency PO**: Create PO with SAR currency → rate fetched, dual columns shown, amounts converted
3. **Currency change on PO**: Change currency after adding products → alert dialog, pricing refreshed
4. **Invoice**: Create invoice from foreign PO → gets own rate, currency displayed read-only
5. **Accounting**: Trigger accounting on foreign invoice → Sage receives MAD amounts + MAD currencyCode
