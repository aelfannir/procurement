# Implementation Plan: Multi-Currency MVP

**Branch**: `005-multi-currency` | **Date**: 2026-03-25 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/005-multi-currency/spec.md`

## Summary

Add multi-currency support to PurchaseOrder and Invoice. User selects a currency on the PO, backend fetches exchange rate from a free global API (fawazahmed0), frontend shows dual-currency columns. Invoice fetches its own rate at creation time. AccountingService converts to clinic currency before sending to Sage. 4 new database fields on 2 existing entities. No new entities.

## Technical Context

**Language/Version**: PHP 8.2 / Symfony 7.x / API Platform 4.x (backend), React 19 / TypeScript / Vite (frontend)
**Primary Dependencies**: API Platform, Doctrine ORM, Symfony HttpClient (backend); React Query v3, shadcn/ui, Formik (frontend)
**Storage**: PostgreSQL (existing — 4 new columns on 2 tables)
**Testing**: PHPUnit via `docker compose exec php php bin/phpunit` (backend), `pnpm test` (frontend)
**Target Platform**: Web application (Docker/FrankenPHP backend, Vite frontend)
**Project Type**: Web service (API + SPA)
**Performance Goals**: Currency API call < 2s per PO creation. Same-currency POs: zero performance impact.
**Constraints**: Currency API must be available for foreign-currency PO creation. Retry once on failure, then block.
**Scale/Scope**: ~50 concurrent users, ~500 POs/month. Minimal DB impact.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. API Platform First | ✅ Pass | New fields exposed via API Platform attributes/Groups. No custom controllers needed. |
| II. Domain-Grouped | ✅ Pass | Changes in PurchaseOrder and Invoice domains. New CurrencyApiService is a shared service, not a domain. |
| III. Enum as Source of Truth | ✅ Pass | No new enums needed. Currency codes come from Country entity (ISO 4217). |
| IV. YAGNI | ✅ Pass | MVP: 4 fields, 0 new entities, 0 settings screens. Companion fields deferred. |
| V. Lean MCP Layer | ✅ Pass | No MCP changes needed. |
| VI. Multi-Workspace Awareness | ✅ Pass | Clinic currency derived from `clinic.city.country.currencyCode`. Works across workspaces. |

## Project Structure

### Documentation (this feature)

```text
specs/005-multi-currency/
├── spec.md              # Feature specification (approved MVP)
├── plan.md              # This file
├── research.md          # Phase 0 output — 7 research decisions
├── data-model.md        # Phase 1 output — entity changes + migration SQL
├── quickstart.md        # Phase 1 output — getting started guide
├── contracts/
│   └── api-changes.md   # Phase 1 output — API contract changes
├── mvp-summary.md       # MVP summary for PO review
└── checklists/
    └── requirements.md  # Spec quality checklist
```

### Source Code (repository root)

```text
procurement-api/
├── src/
│   ├── Entity/
│   │   ├── PurchaseOrder.php          # Add currencyCode + exchangeRate
│   │   └── Invoice.php                # Add currencyCode + exchangeRate
│   ├── Service/
│   │   ├── CurrencyApiService.php     # NEW — fetch rate from fawazahmed0
│   │   └── AccountingService.php      # MODIFY — convert to clinic currency
│   └── Doctrine/
│       └── CurrencyExtension.php      # MODIFY — allow explicit currency filter
├── migrations/
│   └── VersionXXX.php                 # Add columns + backfill + NOT NULL
└── tests/

procurement-app/
├── src/
│   └── modules/_sharedMapping/PurchaseOrder/
│       ├── Mapping.tsx                # MODIFY — add currency dropdown + dual columns
│       ├── fields/
│       │   ├── CurrencyField.tsx      # NEW — currency selector with rate fetch
│       │   └── ProductField.tsx       # MODIFY — pass currencyCode filter
│       └── Model.ts                   # Already has currencyCode (read-only)
```

**Structure Decision**: Existing web application structure. Backend and frontend are separate projects (procurement-api, procurement-app). Changes are minimal — modifying existing files, adding 1 new service and 1 new frontend component.

## Complexity Tracking

No constitution violations. No complexity justification needed.
