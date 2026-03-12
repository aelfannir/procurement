# US9 - Zero Discrepancy Pass-Through (Priority: P2)

Parent: [spec.md](../spec.md) | Requirements: FR-012

As a user comptabilising an invoice with no discrepancy (FRS_TTC = SYS_TTC),
the existing standard accounting flow applies unchanged.

**Why this priority**: Regression safety — the new feature MUST NOT alter
the existing zero-discrepancy flow.

**Independent Test**: Comptabilise an invoice where amounts match exactly —
verify standard flow with no popup, no discrepancy logic.

## Acceptance Scenarios

1. **Given** an invoice where FRS_TTC = SYS_TTC, **When** I click
   Comptabiliser, **Then** the standard accounting flow executes without
   any discrepancy popup or discrepancy account entries.
