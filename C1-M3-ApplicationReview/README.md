# C1-M3 Application Review

**Version:** 0.2, as of 2026-10-30
**Capability:** C1 Onboarding
**Kind:** Hackathon
**Files:** this README (behaviour), [openapi.yaml](openapi.yaml) (interface), [open-points.md](open-points.md)

This README and the OpenAPI description are sufficient to build the module. The framework in [`../FRAMEWORK.md`](../FRAMEWORK.md) is binding as well. Paths and field names are in English.

## 1. Purpose

The module reviews applications that companies have submitted to the onboarding service provider (OSP). For each application it creates a review case, runs an automatic pre-check and records the result per check. The pre-check includes a duplicate check in the own installation (C3-M1) and at the CSP (C3-M2 `find-companies`). OSP staff decide: they approve the application or decline it with a reason; in defined cases two people approve (four-eyes principle). The module reports the decision as an event; Process Orchestration controls the rest of the flow (FRAMEWORK 5.2, 5.5, 5.6).

The module has no user interface. Callers:

| Who | What for |
|---|---|
| Process Orchestration | step `open-review` after `ApplicationSubmitted`: create the review case, run the pre-check; step `close-review` after `ApplicationWithdrawn` |
| OSP staff, portal role `OSP Reviewer` (permissions `view_reviews`, `review_application`) | list and read review cases, approve, decline |
| OSP staff, portal role `OSP Admin` (permission `view_reviews`) | list and read review cases |
| C1-M2 Registration | status of the review case for an application (query; withdrawal, display for the person) |

## 2. Terms

| Term | Meaning |
|---|---|
| Review case | review of a submitted application; exactly one per application |
| Check | one automatic check with a key and the result `PASSED`, `FAILED`, `NOT_APPLICABLE` or `ERROR` |
| Automatic pre-check | all enabled checks, run in the step `open-review` |
| Finding | details of a result, for example differing fields or companies found; without names of people and without values of the application's identifiers |
| Decision | approval or decline by OSP staff; final |
| Application kind | `ONBOARDING`, `REACTIVATION` or `TRANSFER` (C1-M2 README 6.2) |
| Verification path | `EBW` (with verification attestation) or `DOCUMENTS` (with supporting documents, checked by hand) |
| Duplicate | another company in this installation (C3-M1) with an identical identifier, status not `REJECTED` |
| CSP match | company that the CSP knows by EBWOID, EUID or name and country, with BPN and home OSP (C3-M2 README 9) |
| Expected match | CSP match that the application kind requires: for `TRANSFER` the company at the previous home OSP, for `REACTIVATION` the own company (6.3) |
| Reviewer | member of OSP staff who decides; identified by `sub` in the token |
| Applicant | person who made the application (`applicant.sub` of the application) |
| Four-eyes case | reason why a second person must approve (6.4) |

## 3. Not part of this module

| What | Where |
|---|---|
| Accept the application, verification attestation and supporting documents, check the signature, compare the company data, binding to the person | C1-M2 Registration |
| Check the credentials of the wallets | C5-M1 EBW-Connector |
| Close the application after a decline, complete it after onboarding | C1-M2, steps `close-application`, `complete-application` |
| Create, reject, activate, reactivate the company; company identifier in the installation | C3-M1 Company Profile |
| Search at the CSP, registration and transfer at the CSP | C3-M2 CSP Connection |
| Bind the applicant's account to the company | C2-M2, step `assign-company-admin` |
| Message to the person and notice to staff about new review cases | Notifications (another part), OP-19 |
| Dataspace wallet, credentials | C4-M2, C4-M1 |
| Order of the steps, retriggering | Process Orchestration |
| Clearing house, self-description | deferred (6.10) |
| User interface for staff | user interface of the OSP |

## 4. Technical frame

- Python with FastAPI, part of the modular monolith. Process Orchestration calls the steps in the same process; the HTTP paths in [openapi.yaml](openapi.yaml) describe the same contract.
- Own database `c1_m3` in PostgreSQL with its own database user.
- Every HTTP call of a person carries their token from the OSP's Portal IAM. The shared library validates it and provides the claims, including `sub` and the permissions (FRAMEWORK 4.2).
- Events through the outbox of the base skeleton, in the same transaction as the change.
- The module reads application data only through the queries of C1-M2, companies of the installation only through the queries of C3-M1, and the CSP only through C3-M2. It stores no copy of the application data; it keeps only what it needs for the decision and the events (application kind, verification path, `sub` of the applicant, BPN of the expected match).
- At start-up the module checks its configuration (6.6); invalid values prevent the start.

### Configuration

| Variable | Meaning | Example |
|---|---|---|
| `DATABASE_URL` | own database `c1_m3` | |
| `IAM_URL`, `IAM_REALM` | the OSP's Portal IAM for token validation | `https://iam.osp-a.example.org/auth`, `osp-a` |
| `ENABLED_CHECKS` | enabled checks, comma-separated (6.2) | `ATTESTATION_VALID,REPRESENTATION_CONFIRMED,COMPANY_DATA_MATCH,NO_DUPLICATE,CSP_MATCH_EXPECTED,CONSENTS_COMPLETE` |
| `DUPLICATE_CHECK_IDENTIFIER_TYPES` | identifier types for the duplicate check in the installation | `EUID,EBWOID,VAT_ID,LEI_CODE` |
| `FOUR_EYES_CASES` | four-eyes cases, comma-separated: `CHECK_FAILED`, `DOCUMENTS`, `ALWAYS`; empty = none (6.4) | `CHECK_FAILED,DOCUMENTS` |
| `AUTO_APPROVE_KINDS` | application kinds with automatic approval: `ONBOARDING`, `TRANSFER`; empty = none (6.6) | |
| `REGISTRATION_URL` | C1-M2: empty = built-in stand-in, `in-process` = call in the same process, otherwise HTTP address (FRAMEWORK 4.7) | `in-process` |
| `COMPANY_URL` | C3-M1: like `REGISTRATION_URL` | `in-process` |
| `CSP_CONNECTION_URL` | C3-M2: like `REGISTRATION_URL` | `in-process` |
| `STUB_APPLICATIONS_FILE` | file for the stand-in of C1-M2 (14.2) | `stub-applications.json` |
| `STUB_COMPANIES_FILE` | file for the stand-in of C3-M1 (14.3) | `stub-companies.json` |
| `STUB_CSP_FILE` | file for the stand-in of C3-M2 (14.4) | `stub-csp.json` |

## 5. Data

### Table `review_case`

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key; `reviewCaseId` |
| `application_id` | uuid | unique; application in C1-M2 |
| `company_id` | uuid | from the application |
| `kind` | text | application kind from the application |
| `existing_company` | boolean | from the application |
| `verification` | text | verification path from the application: `EBW` or `DOCUMENTS` |
| `applicant_sub` | text | `sub` of the applicant from the application; for the rule "no own application" (6.4) |
| `status` | text | `OPEN`, `AWAITING_SECOND_APPROVAL`, `APPROVED`, `DECLINED`, `CLOSED` (6.1) |
| `csp_bpn` | text | BPN of the expected CSP match (`TRANSFER`, `REACTIVATION`); otherwise empty |
| `four_eyes_reasons` | text[] | four-eyes cases at the first approval |
| `first_approved_by_sub`, `first_approved_at`, `first_approval_comment` | text, timestamptz, text | first approval in a four-eyes case |
| `decided_by_sub` | text | reviewer of the final decision; empty for automatic approval |
| `decided_at` | timestamptz | empty as long as not decided |
| `decision_comment` | text | reason for the decline or comment on the approval, 1 to 1000 characters |
| `auto_approved` | boolean | default `false` |
| `close_reason` | text | `WITHDRAWN` with status `CLOSED` |
| `opened_at`, `closed_at`, `changed_at` | timestamptz | |

### Table `review_check`

| Field | Type | Rule |
|---|---|---|
| `review_case_id` | uuid | part of the primary key |
| `check_key` | text | key of the check (6.2); part of the primary key |
| `result` | text | `PASSED`, `FAILED`, `NOT_APPLICABLE`, `ERROR` |
| `details` | jsonb | finding (6.2, 6.3): field names, identifier types, IDs, statuses, BPN and home OSP of CSP matches; no values of the application's identifiers, no names of people |
| `checked_at` | timestamptz | last run |
| `attempts` | integer | number of runs |

### Table `audit_log`

Time, `sub` of the acting person (empty for steps, with `processId` and `stepRunId` instead), `company_id`, operation, `review_case_id`, `application_id`.

### Table `outbox`

As specified by the base skeleton.

**Not stored:** application data, verification attestation, supporting documents, names of people, values of the application's identifiers (C1-M2); records of other companies (C3-M1); data of the CSP beyond the findings.

## 6. Rules

### 6.1 Status

| From | Trigger | To | Event |
|---|---|---|---|
| – | step `open-review` (8.1) | `OPEN` | – |
| – | step `open-review` or `close-review` for a withdrawn application without a review case (8.1, 8.2) | `CLOSED` | – |
| `OPEN` | `open-review` with automatic approval (6.6) | `APPROVED` | `ApplicationApproved` |
| `OPEN` | approve (7.3), no four-eyes case | `APPROVED` | `ApplicationApproved` |
| `OPEN` | approve (7.3), four-eyes case (6.4) | `AWAITING_SECOND_APPROVAL` | – |
| `AWAITING_SECOND_APPROVAL` | approval by another person (7.3) | `APPROVED` | `ApplicationApproved` |
| `OPEN`, `AWAITING_SECOND_APPROVAL` | decline (7.4) | `DECLINED` | `ApplicationDeclined` |
| `OPEN`, `AWAITING_SECOND_APPROVAL` | step `close-review` (8.2) | `CLOSED` | – |

`APPROVED`, `DECLINED` and `CLOSED` are final. `OPEN` and `AWAITING_SECOND_APPROVAL` correspond to "in review", `APPROVED` to "approved" in the overall view (C1-M2 README 6.1).

### 6.2 Checks

The pre-check runs every enabled check (`ENABLED_CHECKS`) independently of the others, in this order. The basis is the application from the query `GET /internal/applications/{id}` of C1-M2 (README 9.1 there). `CSP_MATCH_EXPECTED` always runs for `TRANSFER` and `REACTIVATION`, even if it is not enabled; the following steps need the BPN (6.3).

| Key | Checks | `PASSED` | `FAILED` | `ERROR` | Finding (`details`) |
|---|---|---|---|---|---|
| `ATTESTATION_VALID` | verification attestation valid and not expired at submission | `signatureCheck.result` `VALID` and `issuedAt` ≤ `submittedAt` < `expiresAt` | `INVALID`, or the time condition is violated | `UNAVAILABLE` | `signatureCheck`, `expiredAtSubmission` |
| `REPRESENTATION_CONFIRMED` | representation confirmed | `attestation.representation.confirmed` is `true` | `false` | – | `representationReasons` (reasons of the EBW-Connector, for example `COMPANY_MISMATCH`) |
| `COMPANY_DATA_MATCH` | comparison of the company data | no entry in `comparison` with `MISMATCH` | at least one `MISMATCH` | – | `mismatchedFields`, `unverifiedFields` (result `NOT_IN_ATTESTATION`) |
| `NO_DUPLICATE` | duplicate check in the installation (C3-M1) | no duplicate | at least one duplicate | C3-M1 not reachable | `duplicates`: per match `companyId`, `companyStatus`, `matchedOn` (identifier types) |
| `CSP_MATCH_EXPECTED` | CSP matches fit the application kind (6.3) | per 6.3 | per 6.3 | C3-M2, CSP or (where needed) C3-M1 not reachable | `cspOutcome`, `cspMatches` (6.3) |
| `CONSENTS_COMPLETE` | consents complete | every terms entry in `requiredAgreements` has a consent with the same key and the same version | otherwise | – | `missing`: key and version |

- **Verification path `DOCUMENTS`:** `ATTESTATION_VALID`, `REPRESENTATION_CONFIRMED` and `COMPANY_DATA_MATCH` give `NOT_APPLICABLE`; there is no verification attestation. Staff check the company and the representation by hand against the supporting documents (OP-35).
- **Duplicate check in the installation:** The module asks C3-M1 for companies with one of the application's identifiers whose type is in `DUPLICATE_CHECK_IDENTIFIER_TYPES` (11.2), excluding the application's own company (`excludeCompanyId`), with the statuses `REQUESTED`, `ACTIVE`, `INACTIVE`. Every match is a duplicate. Companies in status `REQUESTED` count; this way two simultaneous applications for the same company are also detected. If the application has no such identifier: `PASSED` without a request.
- With `ERROR`, the finding contains `error` with a short cause, for example "C3-M2 not reachable".

### 6.3 Duplicate check at the CSP (`CSP_MATCH_EXPECTED`)

**Request.** C3-M2 `find-companies` (11.3) with `ebwoid` (if the application has an EBWOID), `euid` (if it has an EUID), `legalName` and `country`. The module merges matches with the same BPN into one (`matchedBy` as a list).

**Evaluation by application kind.** The first matching row applies.

| Application kind | Condition | Result | `cspOutcome` |
|---|---|---|---|
| `ONBOARDING` | no match | `PASSED` | `NO_MATCH` |
| `ONBOARDING` | a match with home OSP equal to this OSP | `FAILED` | `REGISTERED_HERE` (hint: apply for `REACTIVATION`) |
| `ONBOARDING` | otherwise | `FAILED` | `REGISTERED_AT_OTHER_OSP` (hint: apply for `TRANSFER`) |
| `TRANSFER` | no match | `FAILED` | `NO_MATCH` |
| `TRANSFER` | a match with home OSP equal to this OSP | `FAILED` | `REGISTERED_HERE` |
| `TRANSFER` | more than one BPN | `FAILED` | `MULTIPLE_MATCHES` |
| `TRANSFER` | the only match is by name only (`matchedBy` without `EBWOID` and `EUID`) | `FAILED` | `NAME_MATCH_ONLY` |
| `TRANSFER` | return (`existingCompany`) and BPN not equal to the BPN of the company in C3-M1 | `FAILED` | `BPN_MISMATCH` |
| `TRANSFER` | otherwise: exactly one BPN at another OSP, found by EBWOID or EUID | `PASSED` | `EXPECTED_MATCH` |
| `REACTIVATION` | no match | `FAILED` | `NO_MATCH` |
| `REACTIVATION` | no match with the BPN of the company in C3-M1 | `FAILED` | `BPN_MISMATCH` |
| `REACTIVATION` | match with this BPN, the home OSP is another one | `FAILED` | `REGISTERED_AT_OTHER_OSP` (hint: apply for `TRANSFER`) |
| `REACTIVATION` | further matches with another BPN | `FAILED` | `MULTIPLE_MATCHES` |
| `REACTIVATION` | otherwise: exactly the company's BPN, the home OSP is this OSP | `PASSED` | `EXPECTED_MATCH` |

- For `REACTIVATION` and for a return, the module reads the BPN of the company in C3-M1 through C3-M1 (11.2).
- With `EXPECTED_MATCH`, the module stores the BPN in `csp_bpn`; it goes to the flow with `ApplicationApproved` (C3-M2 `transfer-company` needs it).
- A transfer based only on an identical name is excluded: otherwise someone could pull another company with the same name over to themselves.
- **Finding:** `cspOutcome` and `cspMatches`, per match `bpn`, `matchedBy`, `homeOspId`, `homeOspName`, `homeIsThisOsp`, `memberActive`, `expected` (is the expected match). The BPN is an identifier of the dataspace, not a value from the application.

### 6.4 Four-eyes principle and own application

**Four-eyes cases** (OP-16), set in `FOUR_EYES_CASES`:

| Case | Applies when |
|---|---|
| `CHECK_FAILED` | a check is `FAILED` (approval despite a finding) |
| `DOCUMENTS` | the verification path is `DOCUMENTS` |
| `ALWAYS` | always |

Default `CHECK_FAILED,DOCUMENTS`. Empty: approval by one person is enough.

- If at least one case applies at approval, the module records the first approval as pending: status `AWAITING_SECOND_APPROVAL`, with `sub`, time, comment and the cases. No event.
- The second approval must come from another person; the module compares `sub` in the token. It sets `APPROVED` and triggers `ApplicationApproved`. If the same person approves again: 200, nothing changed.
- In status `AWAITING_SECOND_APPROVAL`, any reviewer can decline, including the first one.
- The header `X-Acting-Person` from version 0.1 is dropped; the person is in the token.

**No own application.** A person never decides on an application they made themselves: if `sub` in the token equals `applicant_sub`, approve and decline answer 403 `own-application`. The rule always applies, independent of `FOUR_EYES_CASES`. It replaces the case `OWN_APPLICATION` from the PSP architecture (own application of the operator PSP): the OSP's company does not come from an application but from the installation settings (`ospCompanyId`, FRAMEWORK 4.8). What remains is the case where a member of staff makes an application themselves; a fixed rule is enough for this; no setting is needed.

### 6.5 Decision

- **Approve:** review case `OPEN` or `AWAITING_SECOND_APPROVAL`, application in C1-M2 not withdrawn. Every enabled check has a result `PASSED`, `FAILED` or `NOT_APPLICABLE`; if one is missing or one is `ERROR`: 409 `checks-incomplete`. For `TRANSFER` and `REACTIVATION`, `CSP_MATCH_EXPECTED` must be `PASSED`, otherwise 409 `csp-match-required`; without an expected match the flow cannot continue. If a check is `FAILED`, a comment is required at the first approval (1 to 1000 characters), otherwise 400 `comment-required`. At the second approval the comment is optional.
- **Decline:** review case `OPEN` or `AWAITING_SECOND_APPROVAL`, application not withdrawn. Reason required (1 to 1000 characters). Declining works regardless of the check results.
- **Withdrawn application:** Before every decision, the module reads the status of the application at C1-M2. If it is `WITHDRAWN`: 409 `application-withdrawn`; `close-review` closes the review case. C1-M2 not reachable: 502 `upstream-error`.
- The same decision again: 200, nothing changed, no event. A different decision after a final decision, or a decision on a `CLOSED` review case: 409 `invalid-state`.
- `FAILED` does not decline by itself; staff decide.
- The reason for a decline goes to the flow with `ApplicationDeclined` and reaches the applicant (C1-M2 `close-application`, notifications). It contains no names of people.

### 6.6 Automatic approval

`open-review` approves a review case without staff if all of these apply:

- `AUTO_APPROVE_KINDS` contains the kind of the application,
- verification path `EBW`,
- all enabled checks (and for `TRANSFER` also `CSP_MATCH_EXPECTED`) are `PASSED`,
- no four-eyes case applies; with `ALWAYS` there is therefore no automatic approval.

Then: status `APPROVED`, `auto_approved` `true`, without a reviewer, event `ApplicationApproved`. Otherwise the review case stays `OPEN` for staff.

- `REACTIVATION` is not allowed in `AUTO_APPROVE_KINDS`: staff decide on a reactivation (FRAMEWORK 5.5). The verification path `DOCUMENTS` is never approved automatically (OP-35). An invalid value prevents the start.
- The default is empty, so there is no automatic approval.

### 6.7 Permissions

| Permission | Scope | Operations | Portal role (proposal for C2-M5) |
|---|---|---|---|
| `view_reviews` | `osp` | list and read review cases (7.1, 7.2) | `OSP Reviewer`, `OSP Admin` |
| `review_application` | `osp` | approve, decline (7.3, 7.4) | `OSP Reviewer` |
| `execute_steps_c1_m3` | `osp` | steps (section 8), only for HTTP calls | service account |
| `query_c1_m3` | `osp` | queries (section 9), only for HTTP calls | service account |

Staff act for the OSP and see the review cases of all companies of the installation; there is no scope `company`. For steps and queries, `osp` means: applies to the whole installation, without reference to a person. In the same process they run without a token.

### 6.8 Idempotency and concurrency

- `open-review` finds the review case by `applicationId`. On repetition it runs only checks without a result or with `ERROR`; `PASSED`, `FAILED` and `NOT_APPLICABLE` stay. A check that is enabled later thus runs at the next repetition.
- A review case that is not `OPEN` is no longer checked.
- `close-review` on a closed or declined review case changes nothing.
- Approve, decline, `open-review` and `close-review` lock the row of the review case (`SELECT … FOR UPDATE`). Two reviewers who decide at the same time thus produce exactly one decision, or exactly one pending first approval and one second approval.
- Events only when the status changes.

### 6.9 Audit log

Every decision and every run of a step writes an entry to `audit_log` (FRAMEWORK 4.3): time, `sub` of the reviewer (for a step `processId` and `stepRunId`), `company_id`, operation, `review_case_id`, `application_id`.

### 6.10 Further checks (not in version 0.2)

The checks for clearing house and self-description that exist today are deferred. Further checks are added as follows, without rebuilding the module:

- **Built-in checks** are self-contained checks with a key and a function that derives the result and the finding from the application. A new built-in check is registered in the module; each OSP enables it with `ENABLED_CHECKS`.
- **External checks** take longer or belong to other services, for example a comparison with a register. An OSP adds them as additional steps to its onboarding flow, because every OSP can adapt its flows. A general step of C1-M3 brings their result into the review case, for example `record-check-result` with check, result and finding.
- Then to be defined per OSP: which checks run, which must have a result before a decision, and which prevent automatic approval when `FAILED`.

Of this, version 0.2 contains only `ENABLED_CHECKS`.

## 7. Operations

### 7.1 `GET /review-cases`

1. Filters: `status` (may be repeated; default `OPEN` and `AWAITING_SECOND_APPROVAL`), `kind`, `verification` (each may be repeated), `applicationId`. Sorted by `openedAt` ascending, oldest first. Paged (`page`, `size`).
2. Per page one call of the summary form at C1-M2 (`GET /internal/applications?applicationId=…`) for name, country and company roles. C1-M2 not reachable: 502 `upstream-error`.
3. Response `ReviewCasePage` with entries `ReviewCaseSummary`: review case, number of checks per result, summary of the application.

### 7.2 `GET /review-cases/{reviewCaseId}`

1. Unknown: 404.
2. Read the application at C1-M2 (`GET /internal/applications/{id}`); it contains the current signature check, the applicant and the metadata of the supporting documents. The user interface loads the supporting documents themselves from C1-M2 (`/osp/applications/{id}/documents/{documentId}`, permission `view_applications`). C1-M2 not reachable: 502 `upstream-error`.
3. Response `ReviewCase`: review case, all checks with findings, first approval, decision, application (`application`) unchanged from C1-M2.

### 7.3 `POST /review-cases/{reviewCaseId}/approve`

1. Unknown: 404. Comment too long or empty: 400 `validation`.
2. Read the status of the application at C1-M2 (summary form). `WITHDRAWN`: 409 `application-withdrawn`. C1-M2 not reachable: 502 `upstream-error`.
3. Lock the row. Status `APPROVED`: 200, nothing changed. `DECLINED`, `CLOSED`: 409 `invalid-state`.
4. `sub` equals `applicant_sub`: 403 `own-application`.
5. Status `AWAITING_SECOND_APPROVAL` and `sub` equals `first_approved_by_sub`: 200, nothing changed.
6. Checks without a result or with `ERROR`: 409 `checks-incomplete` with `pendingChecks`.
7. `TRANSFER` or `REACTIVATION` and `CSP_MATCH_EXPECTED` not `PASSED`: 409 `csp-match-required`.
8. Status `OPEN`, checks with `FAILED` and no comment: 400 `comment-required` with `failedChecks`.
9. In one transaction:
   - Status `OPEN` with a four-eyes case (6.4): `AWAITING_SECOND_APPROVAL`, `first_approved_by_sub`, `first_approved_at`, `first_approval_comment`, `four_eyes_reasons`; audit log entry; no event.
   - Status `OPEN` without a four-eyes case, or `AWAITING_SECOND_APPROVAL` with another person: `APPROVED`, `decided_at`, `decided_by_sub`, `decision_comment`; audit log entry; event `ApplicationApproved`.
10. Response 200 with the review case, without `application`.

### 7.4 `POST /review-cases/{reviewCaseId}/decline`

1. Unknown: 404. `reason` missing or invalid: 400 `validation`.
2. Read the status of the application at C1-M2. `WITHDRAWN`: 409 `application-withdrawn`. C1-M2 not reachable: 502 `upstream-error`.
3. Lock the row. Status `DECLINED`: 200, nothing changed; the stored reason stays. `APPROVED`, `CLOSED`: 409 `invalid-state`.
4. `sub` equals `applicant_sub`: 403 `own-application`.
5. In one transaction: status `DECLINED`, `decided_at`, `decided_by_sub`, `decision_comment` = `reason`; audit log entry; event `ApplicationDeclined` with `reason`.
6. Response 200 with the review case, without `application`.

## 8. Steps for Process Orchestration

Common (FRAMEWORK 4.4): input with `processId` and `stepRunId` (audit log only) and `applicationId`. Result `{ "stepRunId", "state": "DONE", "result": { … } }`. Both steps end with their call. Errors as Problem Details with `retryable` (section 12).

### 8.1 `open-review`

- **Flow:** onboarding, reactivation, transfer, after `ApplicationSubmitted` (FRAMEWORK 5.2 step 4, 5.5, 5.6).
- **Input:** `applicationId`.
- **Preconditions:** The application is submitted in C1-M2 (`SUBMITTED`) when the review case is newly created.
- **Behaviour:**
  1. Read the application through C1-M2. Unknown: 404 `not-found`. C1-M2 not reachable: 502 `upstream-error` (`retryable` `true`), nothing created.
  2. Look for the review case for `applicationId`. If there is none:
     - application `SUBMITTED`: create a review case `OPEN` with `company_id`, `kind`, `existing_company`, `verification` and `applicant_sub` from the application.
     - application `WITHDRAWN`: create a review case `CLOSED` with `close_reason` `WITHDRAWN`, no checks; continue at 6.
     - otherwise: 409 `invalid-state` (`retryable` `false`).
  3. Review case `OPEN`: run the enabled checks without a result or with `ERROR` (6.2, 6.3), store the results, increase `attempts`.
  4. Automatic approval per 6.6.
  5. Review case not `OPEN`: run nothing.
  6. Everything in one transaction, with an audit log entry and, where applicable, an event.
- **Result:** If no check has `ERROR`: `result` with `reviewCaseId`, `applicationId`, `status`, `checks` (`checkKey`, `result`). If a check has `ERROR`: what was stored stays, and the step answers 502 `upstream-error` with `retryable` `true`. Process Orchestration repeats it according to its rules (OP-18); the repetition runs only the missing checks.
- **Event:** none on creation; `ApplicationApproved` on automatic approval.

### 8.2 `close-review`

- **Flow:** onboarding, reactivation, transfer, after `ApplicationWithdrawn` (FRAMEWORK 5.2 step 5).
- **Input:** `applicationId`, `reason` (`WITHDRAWN`).
- **Preconditions:** The application is withdrawn in C1-M2. The flow ensures this; the module checks it only when it creates the review case.
- **Behaviour:**
  1. Look for the review case for `applicationId`. If there is none: read the application through C1-M2 (unknown: 404 `not-found`; not reachable: 502 `upstream-error`, `retryable` `true`; not `WITHDRAWN`: 409 `invalid-state`) and create a review case `CLOSED`. This way an `open-review` that runs later has nothing to do.
  2. `OPEN` or `AWAITING_SECOND_APPROVAL`: status `CLOSED`, `close_reason`, `closed_at`; a pending first approval stays in the review case.
  3. `CLOSED` or `DECLINED`: do nothing.
  4. `APPROVED`: 409 `invalid-state` with `retryable` `false`. The application was withdrawn after staff had approved it (C1-M2 README 6.10); Process Orchestration stops and alerts staff.
  5. Audit log entry in the same transaction.
- **Result:** `result` with `reviewCaseId`, `applicationId`, `status`.
- **Event:** none.

## 9. Queries for other modules

In the same process without a token; over HTTP with the permission `query_c1_m3`. The query changes nothing.

### 9.1 `GET /internal/review-cases?applicationId=…`

For C1-M2 (withdrawal only until the decision, `reviewStatus` for the person). Response `ReviewStatusInfo`: `reviewCaseId`, `applicationId`, `status`, `decidedAt`. No review case for this application: 404 `not-found`.

## 10. Events

Through the outbox, in the same transaction as the change. Every event also carries `eventId`, `eventType`, `occurredAt`. No secrets, no names of people. Schemas in [openapi.yaml](openapi.yaml) under `components/schemas` (names start with `Event`).

| Event | When | Content |
|---|---|---|
| `ApplicationApproved` | approval (7.3) or automatic approval (6.6) | `applicationId`, `companyId`, `kind`, `existingCompany`; `bpn` of the expected match for `TRANSFER` and `REACTIVATION` |
| `ApplicationDeclined` | decline (7.4) | `applicationId`, `companyId`, `kind`, `existingCompany`, `reason` |

- The flow needs `bpn` for `TRANSFER`, for C3-M2 `transfer-company` (C3-M2 README 8.2).
- `existingCompany` tells the flow whether C3-M1 `reject-company` runs after a decline: only with `false`.

## 11. Calls to other modules and external systems

Every neighbour has a built-in stand-in (FRAMEWORK 4.7). The stand-in files are read on every call, so that tests can change them. If a stand-in file contains the JSON string `"unavailable"` instead of a list, the stand-in reports "not reachable".

### 11.1 C1-M2 Registration (`REGISTRATION_URL`)

| Call | Response | Built-in stand-in |
|---|---|---|
| `GET /api/c1-m2/internal/applications/{applicationId}` | `ApplicationInternal` (C1-M2 README 9.1) | reads `STUB_APPLICATIONS_FILE`: list of `ApplicationInternal`; `signatureCheck` as in the file; application missing: 404 |
| `GET /api/c1-m2/internal/applications?applicationId=…` | list of `ApplicationSummary` (C1-M2 README 9.2), also for the status before a decision | built from the same file |

### 11.2 C3-M1 Company Profile (`COMPANY_URL`)

| Call | Response | Built-in stand-in |
|---|---|---|
| `POST /api/c3-m1/internal/companies/identifier-matches` with `identifiers`, `excludeCompanyId`, `statuses` (C3-M1 README 9.3) | `matches`: per company `companyId`, `status`, `bpn`, `matchedOn` | reads `STUB_COMPANIES_FILE`; comparison per type in upper case without spaces, `.`, `-`, `/` |
| `GET /api/c3-m1/internal/companies/{companyId}` (C3-M1 README 9.1), only for `REACTIVATION` and return | `companyId`, `status`, `bpn`; 404 if unknown | from the same file |

### 11.3 C3-M2 CSP Connection (`CSP_CONNECTION_URL`)

| Call | Response | Built-in stand-in |
|---|---|---|
| `GET /api/c3-m2/internal/find-companies?ebwoid=&euid=&legalName=&country=` (C3-M2 README 9) | list of `CspCompanyMatch`: `bpn`, `legalName`, `country`, `matchedBy`, `homeOspId`, `homeOspName`, `homeIsThisOsp`, `memberActive`; 502 if the CSP is not reachable | reads `STUB_CSP_FILE`: matches by the same EBWOID (`EBWOID`), the same EUID (`EUID`) or the same name and the same country, ignoring case (`NAME`); per entry one match with the strongest type |

## 12. Errors

Problem Details (RFC 9457). Additional fields are in [openapi.yaml](openapi.yaml) (`Problem`). `retryable` applies to steps only.

| `type` | HTTP | When | `retryable` |
|---|---|---|---|
| `urn:c1-m3:problem:validation` | 400 | input invalid; details in `errors` | `false` |
| `urn:c1-m3:problem:comment-required` | 400 | first approval despite `FAILED` without a comment; checks in `failedChecks` | – |
| `urn:c1-m3:problem:unauthorized` | 401 | token missing or invalid | – |
| `urn:c1-m3:problem:forbidden` | 403 | permission missing | – |
| `urn:c1-m3:problem:own-application` | 403 | decision on one's own application (6.4) | – |
| `urn:c1-m3:problem:not-found` | 404 | review case or application does not exist | `false` |
| `urn:c1-m3:problem:invalid-state` | 409 | application not submitted; review case decided differently or closed; `close-review` on an approved review case | `false` |
| `urn:c1-m3:problem:checks-incomplete` | 409 | approval although a check has no result or `ERROR`; checks in `pendingChecks` | – |
| `urn:c1-m3:problem:csp-match-required` | 409 | approval for `TRANSFER` or `REACTIVATION` without an expected CSP match; `cspOutcome` | – |
| `urn:c1-m3:problem:application-withdrawn` | 409 | decision on a withdrawn application | – |
| `urn:c1-m3:problem:upstream-error` | 502 | C1-M2, C3-M1 or C3-M2 not reachable; for `open-review` also when a check has `ERROR` | `true` |

## 13. Acceptance

The module is done when these cases work with built-in stand-ins for C1-M2, C3-M1 and C3-M2 and with test tokens of the Portal IAM. The applications are in 14.2, the reviewers R1 and R2 in 14.1. Unless stated otherwise, the default `FOUR_EYES_CASES=CHECK_FAILED,DOCUMENTS` and an empty `AUTO_APPROVE_KINDS` apply, and every case starts with an empty database.

1. `open-review` for A1: review case `OPEN`, six checks `PASSED`, result `DONE`.
2. `open-review` for A1 again: the same review case, `attempts` stays 1, same result.
3. A2 (name differs): `COMPANY_DATA_MATCH` `FAILED`, finding `mismatchedFields` = `LEGAL_NAME`.
4. A3 (representation not confirmed): `REPRESENTATION_CONFIRMED` `FAILED` with `representationReasons` `COMPANY_MISMATCH`.
5. A4 (EUID of an active company): `NO_DUPLICATE` `FAILED` with `companyId` `e0fbbb38-f0a7-4407-bc07-9b1991703a3a`, status `ACTIVE`, `matchedOn` `EUID`. In C3-M1, A1 finds only itself (excluded) and a rejected company: `PASSED`.
6. A5 (consent to `osp-terms` missing): `CONSENTS_COMPLETE` `FAILED`, `missing` contains `osp-terms`.
7. A6 (`signatureCheck` `INVALID`): `ATTESTATION_VALID` `FAILED`. A8 (`expiresAt` before `submittedAt`): `FAILED` with `expiredAtSubmission` `true`.
8. A7 (status `CREATED`): `open-review` answers 409 `invalid-state` with `retryable` `false`, no review case. Unknown `applicationId`: 404.
9. C3-M1 not reachable: `open-review` for A9 stores the review case with `NO_DUPLICATE` `ERROR` and answers 502 with `retryable` `true`. Approve: 409 `checks-incomplete`. After the stand-in is restored and `open-review` is repeated: `PASSED`, result `DONE`; `attempts` is 2 only for `NO_DUPLICATE`. The same with C3-M2 not reachable for `CSP_MATCH_EXPECTED`.
10. A10 (`ONBOARDING` for Andere Beispiel AG): `CSP_MATCH_EXPECTED` `FAILED`, `cspOutcome` `REGISTERED_AT_OTHER_OSP`, one match with `bpn` `BPNL00000000002B`, `homeOspId` of OSP B, `expected` `false`.
11. A11 (`TRANSFER` for Andere Beispiel AG): `CSP_MATCH_EXPECTED` `PASSED`, `cspOutcome` `EXPECTED_MATCH`. Approval by R1: `APPROVED`, `ApplicationApproved` with `kind` `TRANSFER`, `existingCompany` `false`, `bpn` `BPNL00000000002B`.
12. A12 (`REACTIVATION` of Beispiel Werkzeugbau GmbH): `NO_DUPLICATE` `PASSED` (own company excluded), `CSP_MATCH_EXPECTED` `PASSED` with `bpn` `BPNL00000000003C`. Approval by R1: `ApplicationApproved` with `kind` `REACTIVATION`, `existingCompany` `true`, `bpn` `BPNL00000000003C`.
13. A13 (`TRANSFER`, unknown at the CSP): `CSP_MATCH_EXPECTED` `FAILED`, `NO_MATCH`; approval with a comment: 409 `csp-match-required`. A17 (`TRANSFER`, only the same name at the CSP): `FAILED`, `NAME_MATCH_ONLY`.
14. R1 approves A1 without a comment: 200, `APPROVED`, `ApplicationApproved` with `applicationId`, `companyId`, `kind` `ONBOARDING`, `existingCompany` `false`, without `bpn`. Approve again: 200 without a second event. Then decline: 409 `invalid-state`.
15. Approve A2 without a comment: 400 `comment-required` with `COMPANY_DATA_MATCH`. With a comment by R1: `AWAITING_SECOND_APPROVAL` with `fourEyesReasons` `CHECK_FAILED`, no event. R1 again: 200, unchanged. R2: `APPROVED`, `ApplicationApproved`; the review case shows the first approval (R1) and the decision (R2).
16. A14 (verification path `DOCUMENTS`): `ATTESTATION_VALID`, `REPRESENTATION_CONFIRMED`, `COMPANY_DATA_MATCH` `NOT_APPLICABLE`, the others `PASSED`. With `AUTO_APPROVE_KINDS=ONBOARDING` it stays `OPEN`. Approval by R1: `AWAITING_SECOND_APPROVAL` with `DOCUMENTS`; by R2: `APPROVED`.
17. Decline A3 without `reason`: 400. With `reason`: `DECLINED`, `ApplicationDeclined` with `reason`, `kind`, `existingCompany`.
18. A15 (made by R1 themselves): approval and decline by R1: 403 `own-application`. Approval by R2: 200.
19. `close-review` for the open review case of A9: `CLOSED`, no event; repetition: `DONE`, unchanged; approving afterwards: 409 `invalid-state`. `close-review` for A16 (withdrawn, without a review case): a review case `CLOSED` is created; a following `open-review` for A16 answers `DONE` with `CLOSED` and no checks. `close-review` for A1 after its approval: 409 `invalid-state` with `retryable` `false`.
20. If A9 is `WITHDRAWN` in the stand-in of C1-M2 before `close-review` ran: approve and decline answer 409 `application-withdrawn`.
21. `GET /review-cases` without filters: only `OPEN` and `AWAITING_SECOND_APPROVAL`, oldest first, with name and country from C1-M2. With `kind=TRANSFER`: only transfers. With `status=APPROVED&status=DECLINED`: the decided ones.
22. `GET /review-cases/{id}`: checks with findings and the application from C1-M2 with `applicant.sub`.
23. Erika Mustermann (without permissions of this module): 403 for list, details, approval, decline. OSP Admin (only `view_reviews`): list 200, approval 403.
24. `AUTO_APPROVE_KINDS=ONBOARDING`: `open-review` for A1 gives `APPROVED`, `autoApproved` `true`, `ApplicationApproved`. A2 and A12 stay `OPEN`. In addition `FOUR_EYES_CASES=ALWAYS`: A1 stays `OPEN`. `AUTO_APPROVE_KINDS=REACTIVATION`: the module does not start.
25. `ENABLED_CHECKS` without `NO_DUPLICATE` and `CSP_MATCH_EXPECTED`: A1 has four checks, no call to C3-M1 and C3-M2. A11 (`TRANSFER`) still gets `CSP_MATCH_EXPECTED`.
26. `GET /internal/review-cases?applicationId=` for A1: `status`; for A7: 404.
27. Every decision and every step is in `audit_log`, with `sub` or `processId` respectively. Findings and events contain neither names of people nor values of the application's identifiers.

## 14. Test data

### 14.1 Test Portal IAM

Shared test IDs per FRAMEWORK 7a; the installation is the one of OSP A (`ospId` `22222222-2222-4222-8222-222222222222`).

| Account | `sub` | Permissions |
|---|---|---|
| R1, OSP Reviewer | `bbede0f3-fbc4-433b-b7ca-1982e8724912` | `view_reviews`, `review_application` |
| R2, OSP Reviewer | `ed9d901d-cdc3-4a86-8ab4-0b477318506c` | `view_reviews`, `review_application` |
| OSP Admin | `5a1c0012-0000-4000-8000-000000000012` | `view_reviews` |
| Erika Mustermann (applicant) | `5a1c0001-0000-4000-8000-000000000001` | `register_company` (C1-M2 only) |
| Service account | – | `execute_steps_c1_m3`, `query_c1_m3` |

### 14.2 Stand-in for C1-M2 (`stub-applications.json`)

Application A1 in full (verification attestation as in C5-M1 14.5; company and identifiers are invented):

```json
[
  {
    "id": "05b53523-7794-456e-92aa-2e36524e546e",
    "companyId": "c0000001-0000-4000-8000-000000000001",
    "kind": "ONBOARDING",
    "existingCompany": false,
    "verification": "EBW",
    "applicant": { "sub": "5a1c0001-0000-4000-8000-000000000001", "givenName": "Erika", "familyName": "Mustermann" },
    "status": "SUBMITTED",
    "companyData": {
      "legalName": "Beispiel Maschinenbau GmbH",
      "shortName": "Beispiel Maschinenbau",
      "address": { "streetName": "Musterstraße 1", "streetNumber": null, "postalCode": "70569",
                   "city": "Stuttgart", "region": null, "countryCode": "DE" },
      "identifiers": [
        { "type": "EBWOID", "value": "EBWOID-DE-0000000001", "source": "ATTESTATION" },
        { "type": "EUID", "value": "DEXX001T.HRB100001", "source": "ATTESTATION" },
        { "type": "VAT_ID", "value": "DE123456789", "source": "APPLICANT" }
      ]
    },
    "companyRoles": ["ACTIVE_PARTICIPANT"],
    "attestation": {
      "attestationId": "3f6c1e2a-7b4d-4e8f-9a51-2c0d6b8e4f17",
      "issuer": "https://ebw-connector.example.org",
      "keyId": "ebw-test-2026-1",
      "clientId": "demo-osp",
      "issuedAt": "2026-10-28T08:00:00Z",
      "expiresAt": "2026-11-27T08:00:00Z",
      "verifiedAt": "2026-10-28T08:00:00Z",
      "person": { "givenName": "Erika", "familyName": "Mustermann" },
      "representation": {
        "confirmed": true, "result": "CONFIRMED", "type": "legal_representative",
        "role": "Geschäftsführerin", "validUntil": "2027-12-31",
        "issuer": "https://register.test.example.org", "trust": "trust_list", "reasons": []
      },
      "company": {
        "legalName": "Beispiel Maschinenbau GmbH", "ebwoid": "EBWOID-DE-0000000001",
        "euid": "DEXX001T.HRB100001", "countryCode": "DE",
        "address": { "streetAddress": "Musterstraße 1", "postalCode": "70569",
                     "locality": "Stuttgart", "country": "DE" }
      },
      "evidence": [
        { "type": "PID", "vct": "urn:eudi:pid:1", "issuer": "https://pid.test.example.org",
          "wallet": "EUDI_WALLET", "statusChecked": false },
        { "type": "MANDATE", "vct": "urn:tx:test:mandate:1", "issuer": "https://register.test.example.org",
          "wallet": "EUDI_WALLET", "statusChecked": true },
        { "type": "LPID", "vct": "urn:tx:test:lpid:1", "issuer": "https://register.test.example.org",
          "wallet": "BUSINESS_WALLET", "statusChecked": false }
      ],
      "signatureCheck": { "result": "VALID", "checkedAt": "2026-10-28T08:30:00Z" }
    },
    "comparison": [
      { "field": "EBWOID", "result": "FROM_ATTESTATION" },
      { "field": "EUID", "result": "FROM_ATTESTATION" },
      { "field": "COUNTRY", "result": "FROM_ATTESTATION" },
      { "field": "LEGAL_NAME", "result": "FROM_ATTESTATION" },
      { "field": "STREET", "result": "FROM_ATTESTATION" },
      { "field": "POSTAL_CODE", "result": "FROM_ATTESTATION" },
      { "field": "CITY", "result": "FROM_ATTESTATION" }
    ],
    "documents": [],
    "representationDeclared": null,
    "consents": [
      { "agreementKey": "osp-terms", "version": "1.0", "language": "de", "givenAt": "2026-10-28T08:20:00Z" },
      { "agreementKey": "terms-active-participant", "version": "1.0", "language": "de", "givenAt": "2026-10-28T08:20:00Z" },
      { "agreementKey": "security-regulations", "version": "1.0", "language": "en", "givenAt": "2026-10-28T08:20:00Z" }
    ],
    "requiredAgreements": [
      { "agreementKey": "osp-terms", "version": "1.0" },
      { "agreementKey": "terms-active-participant", "version": "1.0" },
      { "agreementKey": "security-regulations", "version": "1.0" }
    ],
    "declineReason": null,
    "createdAt": "2026-10-28T08:10:00Z",
    "changedAt": "2026-10-28T08:20:00Z",
    "submittedAt": "2026-10-28T08:20:00Z",
    "withdrawnAt": null,
    "closedAt": null,
    "completedAt": null
  }
]
```

The other applications are copies of A1 with their own `id`, `companyId`, `attestationId`, EBWOID and EUID (no duplicates among each other), unless the table says otherwise:

| Application | Change compared with A1 |
|---|---|
| A2 | `legalName` "Beispiel Maschinenbau AG", `comparison` `LEGAL_NAME` `MISMATCH` |
| A3 | `representation.confirmed` `false`, `result` `NOT_CONFIRMED`, `reasons` `["COMPANY_MISMATCH"]` |
| A4 | EUID `DEXX001T.HRB654321` (in C3-M1 at an active company) |
| A5 | `consents` without `osp-terms` |
| A6 | `signatureCheck.result` `INVALID` |
| A7 | `status` `CREATED`, `submittedAt` `null`, `consents` and `requiredAgreements` empty |
| A8 | `expiresAt` `2026-10-28T08:15:00Z` (before `submittedAt`) |
| A9 | no change; for cases 9, 19 and 20 |
| A10 | Andere Beispiel AG, `EBWOID-DE-0000000002`, without EUID; `kind` `ONBOARDING` |
| A11 | like A10, `kind` `TRANSFER` |
| A12 | `kind` `REACTIVATION`, `existingCompany` `true`, `companyId` `cc080671-720e-42b6-95b2-0784c8314242`; Beispiel Werkzeugbau GmbH, `EBWOID-DE-0000000003`, EUID `DEXX001T.HRB100003` |
| A13 | `kind` `TRANSFER`, otherwise like A1 (unknown at the CSP) |
| A14 | `verification` `DOCUMENTS`: `attestation` `null`, `comparison` empty; Beispiel Feinmechanik GmbH with `COMMERCIAL_REG_NUMBER` `HRB 100005` (`source` `APPLICANT`); `documents` with one `COMMERCIAL_REGISTER_EXTRACT`; `representationDeclared` `true` |
| A15 | `applicant.sub` equal to R1 (`bbede0f3-fbc4-433b-b7ca-1982e8724912`) |
| A16 | `status` `WITHDRAWN`, `withdrawnAt` set; no review case |
| A17 | `kind` `TRANSFER`, `legalName` "Andere Beispiel AG", own EBWOID (only the same name at the CSP) |

### 14.3 Stand-in for C3-M1 (`stub-companies.json`)

```json
[
  { "companyId": "c0000001-0000-4000-8000-000000000001", "status": "REQUESTED", "bpn": null,
    "identifiers": [ { "type": "EUID", "value": "DEXX001T.HRB100001" }, { "type": "EBWOID", "value": "EBWOID-DE-0000000001" } ] },
  { "companyId": "e0fbbb38-f0a7-4407-bc07-9b1991703a3a", "status": "ACTIVE", "bpn": "BPNL00000000009Z",
    "identifiers": [ { "type": "EUID", "value": "DEXX001T.HRB654321" } ] },
  { "companyId": "91d4a2b8-9045-48a8-8982-88131f6c7fae", "status": "REJECTED", "bpn": null,
    "identifiers": [ { "type": "VAT_ID", "value": "DE123456789" } ] },
  { "companyId": "cc080671-720e-42b6-95b2-0784c8314242", "status": "INACTIVE", "bpn": "BPNL00000000003C",
    "identifiers": [ { "type": "EBWOID", "value": "EBWOID-DE-0000000003" }, { "type": "EUID", "value": "DEXX001T.HRB100003" } ] }
]
```

The first company is the one of A1 itself, the third one was rejected earlier and has the same VAT ID as A1, the fourth is the deactivated company of A12. None of them counts as a duplicate for A1 or A12.

### 14.4 Stand-in for C3-M2 (`stub-csp.json`)

As the test data of C3-M2 (README 14), extended by the deactivated company of this installation:

```json
[
  { "bpn": "BPNL00000000002B", "legalName": "Andere Beispiel AG", "country": "DE",
    "ebwoid": "EBWOID-DE-0000000002", "euid": null,
    "homeOspId": "99a5685c-7397-402c-88de-b1b3a939ecc9", "homeOspName": "OSP B",
    "homeIsThisOsp": false, "memberActive": true },
  { "bpn": "BPNL00000000003C", "legalName": "Beispiel Werkzeugbau GmbH", "country": "DE",
    "ebwoid": "EBWOID-DE-0000000003", "euid": "DEXX001T.HRB100003",
    "homeOspId": "22222222-2222-4222-8222-222222222222", "homeOspName": "OSP A",
    "homeIsThisOsp": true, "memberActive": false }
]
```

## 15. Changes from version 0.1

- **OSP 2.0 instead of the PSP architecture.** OSP staff review and decide with their own token (portal role `OSP Reviewer`, scope `osp`). Dropped: the operator PSP, the registering PSP, `psp_id`, `pspId` in events and review cases, the filter `pspId` and the header `X-Acting-Person`. New permission `view_reviews` for list and details; `review_application` now only for decisions.
- **Duplicate check at the CSP:** new check `CSP_MATCH_EXPECTED` through C3-M2 `find-companies`, evaluated per application kind (6.3); BPN of the expected match in `ApplicationApproved`. The local duplicate check now uses the query `identifier-matches` of C3-M1.
- **Application kinds** `ONBOARDING`, `REACTIVATION`, `TRANSFER` and **verification path** `DOCUMENTS`: new result `NOT_APPLICABLE`; approval of `TRANSFER` and `REACTIVATION` only with an expected CSP match.
- **Four-eyes principle** (OP-16): `FOUR_EYES_CASES` with `CHECK_FAILED`, `DOCUMENTS`, `ALWAYS`, new status `AWAITING_SECOND_APPROVAL`; comparison by `sub`. The case `OWN_APPLICATION` is dropped; instead, nobody ever decides on their own application (`own-application`).
- **Automatic approval** per application kind (`AUTO_APPROVE_KINDS` instead of `AUTO_APPROVE`); never for `REACTIVATION` and the verification path `DOCUMENTS`.
- **Withdrawal** (OP-15): new step `close-review`, status `CLOSED`; no decision on withdrawn applications (`application-withdrawn`); new query `GET /internal/review-cases?applicationId=` for C1-M2.
- The check `CONSENTS_COMPLETE` no longer requires a consent `psp-delegation`. `REPRESENTATION_CONFIRMED` shows the reasons of the EBW-Connector.
- Events with `kind` and `existingCompany` instead of `pspId`. Errors of the steps with `retryable`.
- Test IDs per FRAMEWORK 7a; test companies and verification attestation as in C5-M1.
