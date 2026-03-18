# Data Model: Invoice Discrepancy Management

Parent: [plan.md](plan.md)

## Entity Changes

### Clinic (existing — `src/Entity/Clinic.php`)

New fields:

| Field | Type | Nullable | Default | Description |
|-------|------|----------|---------|-------------|
| `discrepancyThreshold` | `decimal(18,2)` | NO | — | Max vendor-favorable discrepancy allowed |
| `clinicFavorableAccount` | `ManyToOne → Account` | NO | — | Account credited when FRS_TTC < SYS_TTC |
| `vendorFavorableAccount` | `ManyToOne → Account` | NO | — | Account debited when FRS_TTC > SYS_TTC |

**Validation** (FR-002):
- All 3 fields required on save (Assert\NotNull + Assert\NotBlank for threshold)
- Threshold >= 0 (Assert\GreaterThanOrEqual(0))
- Accounts must reference valid, active Account entities
- Validation messages are field-specific (see US1 acceptance scenarios)

**Note**: Fields are NOT nullable in DB. Migration must handle existing clinics — set default threshold = 0 and require manual account configuration. Alternatively, make nullable in DB but enforce NOT NULL in API Platform validation (allows legacy clinics to exist without config, per US8 safety net).

**Recommended approach**: Make fields **nullable in DB** with API Platform `#[Assert\NotNull]` validation on write. This allows legacy clinics to exist without breaking, while US8 catches them at accounting time.

### Invoice (existing — `src/Entity/Invoice.php`, abstract)

New fields for audit (FR-011):

| Field | Type | Nullable | Default | Description |
|-------|------|----------|---------|-------------|
| `discrepancyMotif` | `string(500)` | YES | null | User-entered reason for discrepancy |
| `discrepancyScenario` | `string(50)` | YES | null | Applied scenario label (e.g., "clinic_favorable", "vendor_within_threshold") |
| `discrepancyDecisionAt` | `datetime_immutable` | YES | null | When the user confirmed |
| `discrepancyDecisionBy` | `ManyToOne → User` | YES | null | Who confirmed |

**Note**: These fields are set ONLY on successful comptabilisation with a non-zero discrepancy. Zero-discrepancy invoices leave them null. Blocked/cancelled attempts don't modify Invoice state. DH Auditor tracks all changes for historical audit.

### Account (existing — `src/Entity/Account.php`)

No changes. Referenced by new Clinic relations.

Existing fields used: `id`, `code` (string 8 chars), `name`.

## New Enum

### DiscrepancyDirectionEnum (`src/Entity/Enumeration/DiscrepancyDirectionEnum.php`)

```php
enum DiscrepancyDirectionEnum: string
{
    case Null = 'null';
    case ClinicFavorable = 'clinic_favorable';
    case VendorFavorable = 'vendor_favorable';
}
```

Used for: display label in invoice screen (US2), popup direction label (US3), logic branching in AccountingService.

**Not persisted** — derived at runtime from `amountDifference` sign. Per YAGNI, no need to store what can be calculated.

## Migration

### Doctrine Migration

```sql
-- Clinic: add discrepancy config fields (nullable for legacy data)
ALTER TABLE clinic ADD discrepancy_threshold NUMERIC(18,2) DEFAULT NULL;
ALTER TABLE clinic ADD clinic_favorable_account_id INT DEFAULT NULL;
ALTER TABLE clinic ADD vendor_favorable_account_id INT DEFAULT NULL;
ALTER TABLE clinic ADD CONSTRAINT FK_clinic_favorable_account FOREIGN KEY (clinic_favorable_account_id) REFERENCES account(id);
ALTER TABLE clinic ADD CONSTRAINT FK_vendor_favorable_account FOREIGN KEY (vendor_favorable_account_id) REFERENCES account(id);

-- Invoice: add audit fields
ALTER TABLE invoice ADD discrepancy_motif VARCHAR(500) DEFAULT NULL;
ALTER TABLE invoice ADD discrepancy_scenario VARCHAR(50) DEFAULT NULL;
ALTER TABLE invoice ADD discrepancy_decision_at TIMESTAMP DEFAULT NULL;
ALTER TABLE invoice ADD discrepancy_decision_by_id INT DEFAULT NULL;
ALTER TABLE invoice ADD CONSTRAINT FK_invoice_discrepancy_decision_by FOREIGN KEY (discrepancy_decision_by_id) REFERENCES "user"(id);
```

## Relationships Diagram

```
Clinic ──ManyToOne──▶ Account (clinicFavorableAccount)
Clinic ──ManyToOne──▶ Account (vendorFavorableAccount)
Invoice ──ManyToOne──▶ User (discrepancyDecisionBy)
```
