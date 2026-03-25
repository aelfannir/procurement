# Data Model: Multi-Currency MVP

## Modified Entities

### PurchaseOrder

**New fields:**

| Field | Type | Nullable | Default | Trait |
|-------|------|----------|---------|-------|
| `currencyCode` | string(3) | No (after migration) | clinic currency | `HasCurrencyCode` |
| `exchangeRate` | float | No (after migration) | 1 | Direct field |

**Method changes:**
- `getCurrencyCode()` — currently derives from `clinic.city.country.currencyCode` (line 623). Change to read from stored `currencyCode` field.
- Add `getExchangeRate(): float` / `setExchangeRate(float): self`

**Serialization groups** (add to existing groups):
- `currencyCode`: already serialized via trait — add to `PurchaseOrderCreate`, `PurchaseOrderUpdate`
- `exchangeRate`: add to `PurchaseOrderDetail`, `PurchaseOrderListing`, `PurchaseOrderCreate`, `PurchaseOrderUpdate`, `PURCHASE_ORDER_PRINT`, `PURCHASE_ORDER_EXPORT`

---

### Invoice (abstract)

**New fields:**

| Field | Type | Nullable | Default | Trait |
|-------|------|----------|---------|-------|
| `currencyCode` | string(3) | No (after migration) | clinic currency | `HasCurrencyCode` |
| `exchangeRate` | float | No (after migration) | 1 | Direct field |

**Method changes:**
- Add `getExchangeRate(): float` / `setExchangeRate(float): self`
- `getCurrencyCode()` provided by trait

**Serialization groups**:
- `currencyCode`: `InvoiceListing`, `InvoiceDetail`, `INVOICE_PRINT`, `CreditNoteListing`, `CreditNoteDetail`
- `exchangeRate`: same groups

---

## Migration Plan

### Step 1 — Add nullable columns

```sql
ALTER TABLE purchase_order ADD currency_code VARCHAR(3) DEFAULT NULL;
ALTER TABLE purchase_order ADD exchange_rate DOUBLE PRECISION DEFAULT NULL;
ALTER TABLE invoice ADD currency_code VARCHAR(3) DEFAULT NULL;
ALTER TABLE invoice ADD exchange_rate DOUBLE PRECISION DEFAULT NULL;
```

### Step 2 — Backfill

```sql
UPDATE purchase_order po
SET currency_code = (
    SELECT c.currency_code FROM clinic cl
    JOIN city ci ON cl.city_id = ci.id
    JOIN country c ON ci.country_id = c.id
    WHERE cl.id = po.clinic_id
),
exchange_rate = 1;

UPDATE invoice i
SET currency_code = (
    SELECT c.currency_code FROM clinic cl
    JOIN city ci ON cl.city_id = ci.id
    JOIN country c ON ci.country_id = c.id
    WHERE cl.id = i.clinic_id
),
exchange_rate = 1;
```

### Step 3 — Set NOT NULL

```sql
ALTER TABLE purchase_order ALTER COLUMN currency_code SET NOT NULL;
ALTER TABLE purchase_order ALTER COLUMN exchange_rate SET DEFAULT 1;
ALTER TABLE purchase_order ALTER COLUMN exchange_rate SET NOT NULL;
ALTER TABLE invoice ALTER COLUMN currency_code SET NOT NULL;
ALTER TABLE invoice ALTER COLUMN exchange_rate SET DEFAULT 1;
ALTER TABLE invoice ALTER COLUMN exchange_rate SET NOT NULL;
```

## Entity Relationships (unchanged)

```
Vendor ─┐
        ├─→ PurchaseOrder (currencyCode, exchangeRate) ─→ PurchaseOrderProduct
        │                                                       │
        │                                                       ↓
        │                                                  ReceiptProduct
        │                                                       │
        ├─→ Invoice (currencyCode, exchangeRate) ←──────────────┘
        │
        └─→ ProductPricing (currencyCode) — filtered by PO currency when adding products
```

## No New Entities

All changes are additive fields on existing entities.
