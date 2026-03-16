# Success Criteria — Invoice Discrepancy Management

Parent: [spec.md](spec.md)

## Functional Test Cases (CT01-CT09)

| ID | Test Case | Expected Result |
|----|-----------|----------------|
| CT01 | Create a clinic with all 3 discrepancy fields filled. | Clinic saves successfully. |
| CT02 | Create/edit a clinic without threshold. | Blocked with field-specific validation message. |
| CT03 | Invoice with no discrepancy: FRS_TTC = SYS_TTC. | Standard accounting applies; no discrepancy logic triggered. |
| CT04 | Clinic-favorable discrepancy with complete config. | Confirmation popup, entry generated, status = Comptabilisé. |
| CT05 | Vendor-favorable discrepancy within threshold, no motif. | Confirmation blocked with mandatory-motif message. |
| CT06 | Vendor-favorable discrepancy within threshold, with motif. | Entry generated, status = Comptabilisé. |
| CT07 | Vendor-favorable discrepancy above threshold. | No entry; threshold-exceeded message displayed. |
| CT08 | Invoice with discrepancy on a clinic with no config. | Blocked with missing-config message. |
| CT09 | Verify invoice screen and popup display. | All amounts displayed with (Devise) label. |

## Acceptance Criteria (CA01-CA10)

| ID | Criterion |
|----|-----------|
| CA01 | Discrepancy config is present in the clinic form, accessible on creation and modification. |
| CA02 | System prevents saving a clinic with incomplete mandatory discrepancy config. |
| CA03 | Currency is derived automatically from the clinic's reference country — never manually entered. |
| CA04 | All amounts (displayed, calculated, checked, mentioned in messages) include the (Devise) label. |
| CA05 | Vendor account 4411/4481 always uses FRS_TTC (Devise). |
| CA06 | Clinic-favorable discrepancies are accounted after user confirmation, without threshold check. |
| CA07 | Vendor-favorable discrepancies are accounted after user confirmation only if within the authorized threshold. |
| CA08 | Any threshold breach blocks comptabilisation with the expected business message. |
| CA09 | Missing or incomplete config blocks comptabilisation with an explicit message. |
| CA10 | User confirmation and motif are traced per the defined audit rules. |

## Measurable Outcomes

- **SC-001**: All 9 functional test cases (CT01-CT09) pass
- **SC-002**: All 10 acceptance criteria (CA01-CA10) are met
- **SC-003**: Zero-discrepancy invoices process identically to the current system (no regression)
- **SC-004**: Accounting entries for all 3 discrepancy scenarios balance correctly
- **SC-005**: Audit trail for discrepancy decisions is queryable per invoice
