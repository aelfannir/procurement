# Quickstart: Invoice Discrepancy Management

Parent: [plan.md](plan.md)

## Prerequisites

- Docker running (backend)
- pnpm installed (frontend)
- Access to both `procurement-api/` and `procurement-app/` repos

## Backend Setup

```bash
cd procurement-api
git checkout dev && git pull
git checkout -b ACHAT-2145-invoice-discrepancy

# Generate migration after entity changes
docker compose exec php php bin/console doctrine:migrations:diff
docker compose exec php php bin/console doctrine:migrations:migrate

# Run tests
docker compose exec php php bin/phpunit
```

## Frontend Setup

```bash
cd procurement-app
git checkout dev && git pull
git checkout -b ACHAT-2145-invoice-discrepancy

pnpm install
pnpm dev
```

## Key Files to Modify

### Backend
| File | Change |
|------|--------|
| `src/Entity/Clinic.php` | Add 3 discrepancy fields + validation |
| `src/Entity/Invoice.php` | Add 4 audit fields |
| `src/Entity/Enumeration/DiscrepancyDirectionEnum.php` | New enum |
| `src/Service/AccountingService.php` | Refactor: 9-step processing order, 3 scenarios |
| `src/Controller/AccountingController.php` | Accept motif param, pass to service |

### Frontend
| File | Change |
|------|--------|
| `src/modules/_sharedMapping/Clinic/Model.ts` | Add 3 fields |
| `src/modules/_sharedMapping/Clinic/Mapping.tsx` | Add form fields |
| `src/modules/_sharedMapping/Invoice/Model.ts` | Add audit fields |
| `src/modules/_sharedMapping/Invoice/Mapping.tsx` | Add discrepancy display section |
| `src/modules/_sharedMapping/Invoice/components/AccountingButton.tsx` | Discrepancy detection + routing |
| NEW: `DiscrepancyConfirmDialog.tsx` | Popup with motif input |
| `src/modules/_sharedMapping/Invoice/components/AccountingMass/useBulkAccounting.ts` | Filter discrepancy invoices |
| `src/i18n/locales/fr/invoice.json` | Add French translation keys |

## Verification Checklist

- [ ] Clinic form shows 3 new fields (threshold + 2 account dropdowns)
- [ ] Clinic save blocked when any discrepancy field is missing
- [ ] Invoice screen shows discrepancy summary (amounts + direction)
- [ ] Comptabiliser on discrepancy invoice shows confirmation popup
- [ ] Vendor-favorable popup requires motif
- [ ] Above-threshold shows blocking message (no popup)
- [ ] Missing config shows blocking message
- [ ] Zero-discrepancy uses standard flow (no popup)
- [ ] Bulk excludes discrepancy invoices
- [ ] Accounting entries tab shows generated lines
