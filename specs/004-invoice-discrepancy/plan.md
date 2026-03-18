# Implementation Plan: Invoice Discrepancy Management

**Branch**: `ACHAT-2145-invoice-discrepancy` | **Date**: 2026-03-16 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `specs/004-invoice-discrepancy/spec.md`

## Summary

Replace the hard-coded 0.01 tolerance in `AccountingService` with a clinic-level parameterized discrepancy management system. Three new fields on Clinic (threshold + 2 account references), a confirmation popup with motif for non-zero discrepancies, and a 9-step processing order that handles clinic-favorable, vendor-favorable-within-threshold, vendor-favorable-above-threshold, missing config, and zero-discrepancy scenarios.

## Technical Context

**Language/Version**: PHP 8.2 / Symfony 7.x / API Platform 4.x (backend), React 19 / TypeScript / Vite (frontend)
**Primary Dependencies**: API Platform, Doctrine ORM, DH Auditor (backend); React Query v3, shadcn/ui, Formik, i18next (frontend)
**Storage**: PostgreSQL (existing)
**Testing**: PHPUnit via `docker compose exec php php bin/phpunit` (backend); `pnpm test` (frontend)
**Target Platform**: Web (Docker/FrankenPHP backend, Vite dev server frontend)
**Project Type**: Full-stack web application (API + SPA)
**Constraints**: External Sage API integration for accounting entries, existing DH Auditor for entity change tracking

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. API Platform First | JUSTIFIED BYPASS | AccountingController is a custom controller (not API Platform CRUD) because it orchestrates Sage API calls. See [R1 in research.md](research.md#r1--custom-controller-justification-constitution-i-api-platform-first). Clinic entity changes use standard API Platform serialization groups. |
| II. Domain-Grouped | PASS | All changes are within the existing invoice/clinic domain. No new domain created. |
| III. Enum as Source of Truth | PASS | `DiscrepancyDirectionEnum` created as PHP backed enum. See [R2 in research.md](research.md#r2--enum-strategy-constitution-iii-enum-as-source-of-truth). |
| IV. YAGNI | PASS | No separate audit entity (DH Auditor handles history). No DiscrepancyScenario enum (string label sufficient). Discrepancy direction not persisted (derived from amountDifference). |
| V. Lean MCP Layer | N/A | No MCP changes in this feature. |
| VI. Multi-Workspace Awareness | PASS | Discrepancy config is on Clinic (workspace-independent). Accounting permissions are already workspace-scoped. See [R10 in research.md](research.md#r10--workspace-impact-constitution-vi-multi-workspace-awareness). |

**Post-Phase 1 Re-check**: All gates PASS. No new violations introduced by data model or contracts.

## Project Structure

### Documentation (this feature)

```text
specs/004-invoice-discrepancy/
├── spec.md
├── requirements.md
├── success-criteria.md
├── stories/US1-US9
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/
│   └── accounting-api.md
└── tasks.md             # Phase 2 output (via /speckit.tasks)
```

### Source Code

```text
procurement-api/
├── src/Entity/Clinic.php                          # +3 fields
├── src/Entity/Invoice.php                         # +4 audit fields
├── src/Entity/Enumeration/DiscrepancyDirectionEnum.php  # NEW
├── src/Service/AccountingService.php              # Core refactor
├── src/Controller/AccountingController.php        # Accept motif param
└── migrations/VersionXXX.php                      # Schema changes

procurement-app/
├── src/modules/_sharedMapping/Clinic/Model.ts     # +3 fields
├── src/modules/_sharedMapping/Clinic/Mapping.tsx   # +3 form fields
├── src/modules/_sharedMapping/Invoice/Model.ts    # +4 audit fields
├── src/modules/_sharedMapping/Invoice/Mapping.tsx  # Discrepancy display
├── src/modules/_sharedMapping/Invoice/components/
│   ├── AccountingButton.tsx                        # Discrepancy routing
│   ├── DiscrepancyConfirmDialog.tsx                # NEW: popup
│   └── AccountingMass/useBulkAccounting.ts         # Exclusion filter
└── src/i18n/locales/fr/invoice.json               # Translation keys
```

**Structure Decision**: Both sub-projects already exist. Changes are additions to existing files + 2 new files (enum + dialog component). No new modules or architectural changes.

## Implementation Phases

### Phase 1: Backend Data Foundation [API]

**Scope**: US1 (Clinic config) + audit fields + enum + migration

**Tasks**:
1. Create `DiscrepancyDirectionEnum` backed enum
2. Add 3 fields to `Clinic.php` (nullable in DB, validated via Assert)
3. Add 4 audit fields to `Invoice.php`
4. Generate Doctrine migration
5. Add serialization groups for new fields
6. Add clinic validation constraints (FR-002: all 3 required, field-specific messages)
7. Tests: clinic CRUD with discrepancy fields

**Depends on**: Nothing (foundation)

### Phase 2: Backend Accounting Engine [API]

**Scope**: US3-US9 + FR-001 through FR-016

**Tasks**:
1. Refactor `AccountingService.processInvoiceAccounting()`:
   - Remove hard-coded 0.01 tolerance
   - Implement 9-step processing order
   - Step 1: Calculate amounts (existing logic)
   - Step 2: Zero discrepancy → existing flow (US9)
   - Step 3: Verify clinic config exists (US8)
   - Step 4: Check config completeness + account validity + threshold (US6, FR-016)
   - Step 5-6: Return discrepancy details for popup (response contract)
   - Step 7: Generate entries per scenario (US4, US5)
   - Step 8: Balance check (FR-009)
   - Step 9: Update status + persist audit fields (FR-011)
2. Update `AccountingController.php`:
   - Accept optional `discrepancyMotif` in request body
   - Pass motif to service
   - Return discrepancy details on first attempt (for popup)
3. Handle vendor-favorable entry generation (debit discrepancy account)
4. Handle clinic-favorable entry generation (credit discrepancy account)
5. Validate motif required for vendor-favorable (FR-006)
6. Tests: all 9 CT test cases

**Depends on**: Phase 1

### Phase 3: Backend Bulk Exclusion [API]

**Scope**: FR-015

**Tasks**:
1. Modify bulk endpoint to detect and exclude discrepancy invoices
2. Add `excluded` status to bulk response
3. Tests: bulk with mixed zero/non-zero discrepancy invoices

**Depends on**: Phase 2

### Phase 4: Frontend Clinic Form [APP]

**Scope**: US1

**Tasks**:
1. Add 3 fields to Clinic `Model.ts`
2. Add form fields to Clinic `Mapping.tsx` (threshold input + 2 account dropdowns)
3. Add validation schema (Zod: all 3 required)
4. Add i18n keys for field labels + validation messages

**Depends on**: Phase 1 (API fields available)

### Phase 5: Frontend Invoice Display [APP]

**Scope**: US2

**Tasks**:
1. Add discrepancy summary section to Invoice `Mapping.tsx` detail view
2. Display: system amount, vendor amount, discrepancy (signed), direction label
3. Add visual indicator (badge/color) for discrepancy status
4. Show motif (read-only) when present
5. Add i18n keys

**Depends on**: Phase 1 (API fields available)

### Phase 6: Frontend Popup + Accounting Flow [APP]

**Scope**: US3 + US4 + US5 + US6 + US8

**Tasks**:
1. Create `DiscrepancyConfirmDialog.tsx`:
   - Display amounts (absolute value for discrepancy), direction label
   - Textarea for motif (mandatory/optional based on direction)
   - Confirm / Cancel buttons
   - Client-side motif validation
2. Refactor `AccountingButton.tsx`:
   - Detect non-zero discrepancy from invoice data
   - Compute direction from `amountDifference` sign
   - Check for blocking conditions (threshold, config) from API response
   - Route to popup or show blocking message
   - Send motif in POST body on confirmation
3. Handle 422 responses with new violation messages (blocking)
4. Add i18n keys for popup template + error messages
5. Update `useBulkAccounting.ts`: filter out discrepancy invoices pre-request, report excluded count
6. Tests: component tests for popup

**Depends on**: Phase 2 + Phase 5

## Complexity Tracking

No constitution violations requiring justification beyond R1 (custom controller, already justified).

## Artifacts

| Artifact | Path | Status |
|----------|------|--------|
| Research | [research.md](research.md) | Complete |
| Data Model | [data-model.md](data-model.md) | Complete |
| API Contracts | [contracts/accounting-api.md](contracts/accounting-api.md) | Complete |
| Quickstart | [quickstart.md](quickstart.md) | Complete |
| Tasks | tasks.md | Pending `/speckit.tasks` |
