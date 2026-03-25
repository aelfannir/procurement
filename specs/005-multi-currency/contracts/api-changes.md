# API Contract Changes: Multi-Currency MVP

## PurchaseOrder — New fields in API responses

### GET /purchase_orders (listing)

```json
{
  "currencyCode": "SAR",
  "exchangeRate": 2.50,
  "totalExclTax": 1000,
  "totalInclTax": 1200
}
```

- `currencyCode`: ISO 4217 code (vendor currency)
- `exchangeRate`: rate to convert vendor → clinic currency
- `totalExclTax`, `totalInclTax`: in vendor currency (unchanged)
- Clinic amounts computed client-side: `total × exchangeRate`

### POST /purchase_orders (create)

New writable fields:

```json
{
  "currencyCode": "SAR"
}
```

- `currencyCode`: required. User-selected currency for this PO.
- `exchangeRate`: NOT sent by client. Backend fetches from API and stores.

### PUT /purchase_orders/{id} (update)

```json
{
  "currencyCode": "EUR"
}
```

- If `currencyCode` changes → backend re-fetches exchange rate from API.

---

## Invoice — New fields in API responses

### GET /invoices (listing + detail)

```json
{
  "currencyCode": "SAR",
  "exchangeRate": 2.55
}
```

- `currencyCode`: inherited from PO
- `exchangeRate`: fetched from API at invoice creation date (may differ from PO rate)

### POST /invoices (create)

- No new client input. Both fields auto-populated:
  - `currencyCode` from PO
  - `exchangeRate` from currency API

---

## ProductPricing — Filter change

### GET /product_pricings (collection)

New query parameter:

```
/product_pricings?vendor=/vendors/{id}&currencyCode=SAR&status=Enabled
```

- `currencyCode`: explicit filter parameter. When provided, overrides the automatic locale-based currency filter (CurrencyExtension).

---

## Sage Accounting Payload — Change

### Before

```json
{
  "currencyCode": "SAR",
  "entries": [{ "amount": 100 }]
}
```

### After

```json
{
  "currencyCode": "MAD",
  "entries": [{ "amount": 250 }]
}
```

- `currencyCode`: always clinic currency (e.g., MAD)
- `amount`: converted from vendor currency (`vendorAmount × invoice.exchangeRate`)
- Sage format unchanged — just different values for foreign-currency invoices

---

## Currency API — External call

### Backend service calls

```
GET https://cdn.jsdelivr.net/npm/@fawazahmed0/currency-api@{date}/v1/currencies/{code}.json
```

Example: PO in SAR, clinic in MAD, date 2026-03-24:
```
GET https://cdn.jsdelivr.net/npm/@fawazahmed0/currency-api@2026-03-24/v1/currencies/sar.json
Response: {"date":"2026-03-24","sar":{"mad":2.50,...}}
Extract: response.sar.mad → 2.50
Store: PO.exchangeRate = 2.50
```

Retry once on failure. Block document creation if still unavailable.
