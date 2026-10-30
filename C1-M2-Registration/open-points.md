# C1-M2 Registration: open points

**As of:** 2026-10-30, version 0.2 (OSP 2.0)

For version 0.2 the README fixes each of these points so that the module can be built. The decisions are provisional and still to be agreed. Questions that concern several modules are listed with their number (OP-xx) in `../open-points.md`.

## Provisional decisions

| No. | Topic | Decision in the README | To be agreed with |
|---|---|---|---|
| 1 | Verification attestation version 0.2: claims, `typ` `ebw-attestation+jwt`, `aud`, `ES256`; check of `attestation_version`, `purpose` and `sub`; `representation` and `evidence` stored but not evaluated | 6.4, example 14.4 | C5-M1 |
| 2 | The only recognised issuer is the EBW-Connector of its own installation (`ATTESTATION_ISSUER`, JWKS address); the list of several issuers from 0.1 is dropped. Optional check of `client_id` | 4, 6.4 | OSPs, C5-M1 |
| 3 | **Binding to the person:** first and last name in `person` of the verification attestation must be equal to `given_name` and `family_name` in the token, otherwise a failure (`reason` `person`), not just a hint. Reason: the applicant becomes Company Admin; with automatic approval nobody would read a hint. Replaces the binding to a PSP (`psp_id`, OP-17) | 6.4 | C2-M5 (names in the token from the PID, updated at every sign-in), C5-M1, OSPs |
| 4 | Maximum age of the verification attestation at submission `ATTESTATION_MAX_AGE`, default 30 days from `iat` (equal to the validity in C5-M1) | 6.4 | OSPs, C5-M1 |
| 5 | Every verification attestation (`jti`) is valid for exactly one application, also after replacement or withdrawal | 6.4 | C5-M1 |
| 6 | A verification attestation with `representation.confirmed: false` is accepted; it is assessed in C1-M3 | 6.4 | C1-M3, OSPs |
| 7 | Status model only with own states, new `WITHDRAWN`; "under review" and "approved" are kept by the review case in C1-M3 and appear as `reviewStatus` | 6.1, 7.4 | C1-M3, Process Orchestration |
| 8 | Fixed fields (EBWOID, EUID, country) must be equal; name and address are compared; comparison with simple normalisation | 6.5 | C1-M3, C5-M1 |
| 9 | Identifier types: `EUID` and `EBWOID` new, `VIES` dropped; formats of EUID and EBWOID loose (OP-12); no `EBWOID` in the verification path `DOCUMENTS` | 6.6 | C3-M1 |
| 10 | Countries and identifier types from the shared file of the installation (`masterDataFile`, FRAMEWORK 4.8); expected structure as in 14.2 | 4, 5, 14.2 | FRAMEWORK (OP-13, OP-36), C3-M1 |
| 11 | C1-M2 keeps the selectable company roles and the terms: initial values from `CATALOG_FILE`, new versions by the OSP Admin through the interface; versions immutable; one document per language; party `OSP` or `DATASPACE` | 5, 6.7, 6.8, 7.15, 7.16 | C3-M1 (own role catalogue), OSPs, OP-11 |
| 12 | Required terms: `osp-terms` for all applications instead of `psp-delegation`; terms of the dataspace per role as in portal 2.6.0 | 6.8, 14.3 | OSPs, governance of the dataspace |
| 13 | The person consents personally at submission, with the language of the document read; no additional proof of the consent in the EBW-Connector (the part of OP-17 on this is dropped) | 6.8 | C5-M1, OSPs |
| 14 | Verification path `DOCUMENTS`: kinds of supporting documents, PDF/PNG/JPEG with check of the first bytes, limits on size and count, register extract required, declaration of representation by the person; can be switched off (`DOCUMENTS_ENABLED`, default on); supporting documents in its own database | 6.9 | OSPs, data protection (OP-20) |
| 15 | Application only with an account without a company (403 `account-bound`); at most one open application per person and per existing company | 6.2, 6.3 | C2-M2, C2-M5 (OP-40) |
| 16 | Application kinds: reference through `companyId` or `bpn`; `REACTIVATION` only for `INACTIVE` without reason `TRANSFERRED`; return as `TRANSFER` with reference; roles of an existing company from C3-M1 if none are given | 6.2, 6.7 | C3-M1, C3-M2, Process Orchestration (OP-38) |
| 17 | Withdrawal until the decision, checked through a query at C1-M3; event `ApplicationWithdrawn` only after submission; a remaining race with a concurrent approval stays, C1-M3 and Process Orchestration catch it | 6.10 | C1-M3, Process Orchestration (OP-15, OP-18) |
| 18 | Repetition at creation via the `jti` of the verification attestation instead of `externalId` | 6.11 | UI |
| 19 | `close-application` receives the reason from `ApplicationDeclined`; on `WITHDRAWN` the step answers `DONE` without change | 8.2 | C1-M3, Process Orchestration |
| 20 | `complete-application` checks only its own status; the flow ensures approval and activation | 8.1 | Process Orchestration |
| 21 | Query `/internal/applications/{id}` checks the signature again on every call (`signatureCheck`) | 9.1 | C1-M3 |
| 22 | Permissions `register_company` (scope `own`), `view_applications` and `manage_agreements` (scope `osp`); assignment to the portal roles `Applicant`, `OSP Reviewer`, `OSP Admin` | 6.3 | C2-M5, OP-41 |
| 23 | `reviewStatus` when reading an application; `UNKNOWN` if C1-M3 is not reachable, instead of an error | 7.4 | C1-M3, UI |
| 24 | Company roles: `ONBOARDING_SERVICE_PROVIDER` not selectable; `PORTAL_SERVICE_PROVIDER` and `OPERATOR` dropped | 6.7 | C3-M1 |

## Deviations from the framework and additions to the framework

Taken over into FRAMEWORK 5 to 7 on 28 Oct 2026 (events with their fields, queries, flows); the table stays as a record.

| No. | Kind | Framework | Deviation or addition | Reason |
|---|---|---|---|---|
| 1 | addition to the framework | 6: `ApplicationSubmitted` with `applicationId`, `companyId`, `kind` | additionally `applicantSub` and `existingCompany` | C2-M2 `assign-company-admin` needs `applicantSub` after the approval; `existingCompany` see No. 2 |
| 2 | addition to the framework | 6: `ApplicationCreated` with `kind`; `ApplicationWithdrawn`, `ApplicationCompleted` with `applicationId`, `companyId` | `ApplicationCreated` and `ApplicationWithdrawn` additionally with `existingCompany`; `ApplicationWithdrawn` and `ApplicationCompleted` additionally with `kind` | The flow must know whether a company must be created or declined (`create-company`, `reject-company`): not for `REACTIVATION` and return |
| 3 | addition to the framework | 6: no events about terms | new event `AgreementVersionActivated` (`agreementKey`, `version`, `party`, `reconsentRequired`, `reconsentDueAt`) | trigger the re-consent of existing companies (OP-11) by event, without polling |
| 4 | addition to the framework | 7: queries C1-M3 → C3-M2, C1-M2 → C3-M1 | new query C1-M2 → C1-M3 `GET /internal/review-cases?applicationId=` (C1-M3 section 9) | withdrawal only until the decision; `reviewStatus` for the person |
| 5 | addition to the framework | 7: queries | new query `GET /internal/agreements` (9.3) for C3-M1 | terms of the company roles and versions for the re-consent |
| 6 | addition to the framework | 5.6: transfer without return | `TRANSFER` with reference to a company of this installation that has transferred away (return) | C3-M2 provides for the return (`TRANSFERRED_AWAY` → `transfer-company`); without a reference there would be two companies with the same BPN |
| 7 | deviation | 4.3: `own` applies to own objects | reference data and terms (7.1) with `register_company` in scope `own`, without object check | reference data belongs to nobody; the applicant needs it |

## Assumptions about other modules

| Module | Assumption |
|---|---|
| C2-M5 Portal IAM | The token of a person contains `sub`, `given_name` and `family_name` from the PID (sign-in through the EBW-Connector), updated at every sign-in, and `company_id` only for a bound account. Every new account has the portal role `Applicant` with `register_company`. `view_applications` belongs to `OSP Reviewer` and `OSP Admin`, `manage_agreements` to `OSP Admin`. |
| C2-M2 Users and App Access | `assign-company-admin` binds the account with the `applicantSub` from `ApplicationSubmitted` to the company; it fails cleanly if the account has meanwhile been bound to another company. |
| C3-M1 Company Profile | The queries `GET /internal/companies/{companyId}` and `?bpn=` additionally return `deactivationReason` (at least the value `TRANSFERRED`, FRAMEWORK 5.6), `identifiers` (type and value) and the company roles, also of a deactivated company. `create-company` reads the application through 9.1 and takes over `verification` as the origin of the core data (`EBW` or manual, OP-09). C3-M1 handles the re-consent of existing companies to new versions and reacts to `AgreementVersionActivated` for this (OP-11). |
| C1-M3 Application Review | reads 9.1 and 9.2; offers the query of the review case for an application; does not decide on a withdrawn application; `close-review` reports an already approved review case with 409. |
| C5-M1 EBW-Connector | issues the verification attestation in version 0.2 according to C5-M1 section 9; `client_id` is the client of the UI; `person` comes from the same PID as the names at sign-in. |
| Process Orchestration | calls neither `create-company` nor `reject-company` when `existingCompany` is `true`; after `ApplicationWithdrawn` it calls C1-M3 `close-review` and, only without an existing company, C3-M1 `reject-company`; on contradictory events (withdrawn and approved) it stops and alerts OSP staff. |
| UI of the OSP | fetches the verification attestation from the EBW-Connector (scope `attestation`) and passes it on unchanged; shows the terms in the language of the person and reports the language; on 409 `open-application-exists` continues with the existing application. |
| Notifications | inform the person on `ApplicationSubmitted`, `ApplicationDeclined` and `ApplicationCompleted` (OP-19). |

## Open questions

1. **Return of a transferred company.** FRAMEWORK 5.6 describes only the path to the new OSP. The flow for a return (reactivate the company instead of creating it, C3-M2 `transfer-company`) is still missing. Cross-module, OP-38.
2. ~~**Who files the application for reactivation?**~~ Proposal fixed on 28 Oct 2026 (provisional): the accounts of a deactivated company stay active, but without a company (C2-M2 8.9). A person with a mandate files the application with their account; after the approval, `assign-company-admin` makes them Company Admin (FRAMEWORK 5.4, 5.5).
3. **Applications by OSP staff.** May a member of OSP staff file an application personally, and is their account bound to the company of the OSP? Cross-module, OP-41.
4. **Name change between sign-in and onboarding.** If the Portal IAM does not update the names at every sign-in, the binding to the person fails (6.4 step 10). To be agreed with C2-M5.
5. **Return for correction** instead of decline: would need a transition `SUBMITTED` → `CREATED` (see open questions in C1-M3).
6. **Role catalogue in one place.** C1-M2 keeps the selectable company roles and the terms; C3-M1 reads the terms from here since 30 Oct 2026 and keeps only its list of roles. Cross-module, OP-46.
7. **Re-consent of existing companies:** place (C3-M1) and flow. Cross-module, OP-11.
8. **Supporting documents:** malware scan, storage outside the database for large volumes, retention. Cross-module for the retention periods, OP-20.
9. **Formats of EUID and EBWOID.** Cross-module, OP-12.
10. **Installation settings** and structure of the shared reference data. Cross-module, OP-36, OP-13.
11. **Notifications** to the person. Cross-module, OP-19.

## Out of scope

- further people of the company in the application; they come after the approval through invitations (C2-M2)
- prefilling the data through a search at the CSP
- migration of existing applications from the current portal
- later change of the company roles after onboarding (C3-M1)
- taking data along on transfer (FRAMEWORK 5.6)
- text recognition or automatic check of supporting documents
