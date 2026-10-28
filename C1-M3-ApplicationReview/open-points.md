# C1-M3 Application Review: open points

**Version:** 0.2 (OSP 2.0), as of 2026-10-28

For version 0.2 the README fixes each of these points, so that the module can be built. The decisions are provisional and still to be agreed. Questions that affect several modules are listed with their number (OP-xx) in `../open-points.md`.

## Provisional decisions

| No. | Topic | Decision in the README | To be agreed with |
|---|---|---|---|
| 1 | Six checks and their rules; in the verification path `DOCUMENTS` the three checks on the verification attestation are `NOT_APPLICABLE` | 6.2 | OSPs |
| 2 | `ATTESTATION_VALID` relates the validity to the time of submission; C1-M2 checks the signature on every query with the currently accepted issuer | 6.2 | C1-M2, C5-M1 |
| 3 | Duplicate check in the installation through C3-M1 `identifier-matches` (identifiers `EUID`, `EBWOID`, `VAT_ID`, `LEI_CODE`); `REQUESTED`, `ACTIVE`, `INACTIVE` count, `REJECTED` does not; the application's own company is excluded | 6.2, 11.2 | C3-M1, OSPs |
| 4 | Duplicate check at the CSP (`CSP_MATCH_EXPECTED`) with evaluation per application kind and `cspOutcome`; matches merged per BPN | 6.3 | C3-M2, CSP, OP-38 |
| 5 | Transfer only with a match by EBWOID or EUID, never by name only | 6.3 | OSPs, CSP |
| 6 | `TRANSFER` and `REACTIVATION` can only be approved with an expected CSP match (`csp-match-required`); for `ONBOARDING` a match is a finding that staff can override with a comment | 6.5 | OSPs, Process Orchestration |
| 7 | `CSP_MATCH_EXPECTED` always runs for `TRANSFER` and `REACTIVATION`, even if it is missing in `ENABLED_CHECKS` | 6.2 | OSPs |
| 8 | `FAILED` does not decline by itself; approval despite `FAILED` only with a comment | 6.5 | OSPs |
| 9 | Four-eyes principle (OP-16): cases `CHECK_FAILED`, `DOCUMENTS`, `ALWAYS`, default `CHECK_FAILED,DOCUMENTS`; status `AWAITING_SECOND_APPROVAL`; comparison by `sub` | 6.4 | OSPs |
| 10 | No own application: whoever made the application never decides on it (`own-application`), fixed and not configurable. The case `OWN_APPLICATION` (own application of the operator PSP) is dropped, because the OSP's company does not come from an application | 6.4 | OSPs, OP-41 |
| 11 | Automatic approval per application kind (`AUTO_APPROVE_KINDS`, default empty); never `REACTIVATION`, never verification path `DOCUMENTS`, never with a four-eyes case | 6.6 | OSPs, demonstrator with Business Wallet |
| 12 | `open-review` stores partial results and answers 502 with `retryable` `true` if a check has `ERROR`; Process Orchestration repeats | 8.1 | Process Orchestration (OP-18) |
| 13 | Without a review case, `close-review` creates a closed one, so that an `open-review` that runs later checks nothing; for an approved review case it answers 409 with `retryable` `false` | 8.2 | Process Orchestration, C1-M2 |
| 14 | Before every decision the module reads the status of the application at C1-M2; there is no decision on a withdrawn application | 6.5 | C1-M2 |
| 15 | Permissions `view_reviews` (read) and `review_application` (decide), both scope `osp`; `OSP Reviewer` has both, `OSP Admin` only `view_reviews` | 6.7 | C2-M5, OP-41 |
| 16 | No copy of the application data; the list fetches name, country and roles per page from C1-M2 | 4, 7.1 | C1-M2 |
| 17 | Reason for the decline in the event `ApplicationDeclined`, for `close-application` and notifications | 6.5, 10 | Process Orchestration, C1-M2 |
| 18 | BPN of the expected match in the event `ApplicationApproved`, for C3-M2 `transfer-company` | 6.3, 10 | Process Orchestration, C3-M2 |
| 19 | Checks per OSP in version 0.2 only through `ENABLED_CHECKS` | 6.10 | OSPs |

## Deviations from the framework and additions to the framework

Adopted into FRAMEWORK 5 to 7 on 28 Oct 2026 (events with their fields, queries, flows); the table remains as a record.

| No. | Kind | Framework | Deviation or addition | Reason |
|---|---|---|---|---|
| 1 | Addition to the framework | 6: `ApplicationApproved` with `applicationId`, `companyId`, `kind` | in addition `existingCompany` and, for `TRANSFER` and `REACTIVATION`, `bpn` | C3-M2 `transfer-company` needs `bpn` (C3-M2 README 8.2: "`bpn` from `find-companies`"); `existingCompany` see No. 2 |
| 2 | Addition to the framework | 6: `ApplicationDeclined` with `applicationId`, `companyId`, `kind` | in addition `existingCompany` and `reason` | The flow calls `reject-company` only for newly created companies; the reason goes to `close-application` without a further query |
| 3 | Addition to the framework | 7: queries C1-M3 → C3-M2 `find-companies` | new query C1-M2 → C1-M3 `GET /internal/review-cases?applicationId=` (9.1) | C1-M2 allows withdrawal only until the decision and shows the person the current state |
| 4 | Addition to the framework | 7: queries C1-M3 → C3-M2 | in addition C1-M3 → C3-M1 `identifier-matches` and `GET /internal/companies/{companyId}` | Duplicate check in the installation (simultaneous applications, deactivated companies); BPN of the own company for `REACTIVATION` and return |
| 5 | Addition to the framework | 5.2: status of the review case | new statuses `AWAITING_SECOND_APPROVAL` and `CLOSED` | four-eyes principle (OP-16), withdrawal (OP-15) |
| 6 | Deviation | 5.2 step 5: decision by OSP staff | With `AUTO_APPROVE_KINDS` the module approves by itself when all checks are `PASSED`. Default empty; without enabling it, there is no deviation | fully automatic onboarding with Business Wallet in the demonstrator |

## Assumptions about other modules

| Module | Assumption |
|---|---|
| C1-M2 Registration | `ApplicationInternal` (README 9.1) contains `kind`, `existingCompany`, `verification`, `applicant.sub`, `signatureCheck`, `attestation.representation.reasons` and the metadata of the supporting documents; `ApplicationSummary` (9.2) contains the status including `WITHDRAWN`. |
| C3-M1 Company Profile | `identifier-matches` (C3-M1 README 9.3) stays with `identifiers`, `excludeCompanyId`, `statuses` and the response `matches` (`companyId`, `status`, `bpn`, `matchedOn`); `GET /internal/companies/{companyId}` returns `status` and `bpn`, also for a deactivated company. |
| C3-M2 CSP Connection | `find-companies` (C3-M2 README 9) answers with one `CspCompanyMatch` per match; the same BPN may come several times (per `matchedBy`), C1-M3 merges them; CSP not reachable: 502. |
| C2-M5 Portal IAM | `view_reviews` and `review_application` are permissions on the client `portal-core`; `OSP Reviewer` bundles both, `OSP Admin` contains `view_reviews`. The token of staff contains `sub`. |
| Process Orchestration | calls `open-review` after `ApplicationSubmitted` and `close-review` after `ApplicationWithdrawn`; for `TRANSFER` passes the `bpn` from `ApplicationApproved` to C3-M2 `transfer-company`; calls `reject-company` after `ApplicationDeclined` only with `existingCompany` `false`; stops on 409 from `close-review` and alerts staff. |
| Notifications | alert staff to new review cases and pending first approvals (OP-19). |

## Open questions

1. **Check results for the applicant.** Should the applicant see findings, for example to correct data before a decline?
2. **Return instead of decline.** A status "returned for correction" would make the application editable again in C1-M2. In version 0.2 only decline and a new application.
3. **Repeat the check at the CSP at approval.** Between `open-review` and approval, the home OSP can change or another company can come into existence. For `TRANSFER`, C3-M2 `transfer-company` catches this; for `ONBOARDING` nothing does.
4. **Retract the first approval.** May the first reviewer retract their pending approval, instead of someone declining?
5. **Further criteria for duplicates** in the installation: similar name, same address.
6. **External checks.** General step `record-check-result` (later addition to the step catalogue), configuration per OSP: which checks are mandatory, which prevent automatic approval (6.10).
7. **Deadlines.** How long may a review case stay open?
8. **Return to a previous OSP.** The evaluation at the CSP is defined (6.3), the flow is not yet. Cross-module, OP-38.
9. **OSP staff:** own accounts, binding to the OSP's company, portal roles. Cross-module, OP-41.
10. **Four-eyes principle.** Cross-module, OP-16 (mechanism defined here).
11. **Repetition on `ERROR`.** Cross-module, OP-18.
12. **Notice about new review cases.** Cross-module, OP-19.
13. **Installation settings** (for example `FOUR_EYES_CASES` as a shared setting). Cross-module, OP-36.

## Out of scope

- Clearing house and self-description (deferred)
- Entering the BPN by hand by staff (belongs to C3-M1 or C3-M2)
- Checking the supporting documents by software (text recognition, comparison with registers)
- Taking over ongoing reviews from the current portal
- Analytics, for example time until the decision
