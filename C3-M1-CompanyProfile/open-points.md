# C3-M1 Company Profile: open points

**Version:** 0.2, as of 2026-10-28 (move to OSP 2.0)

The [README](README.md) fixes every point so that the module can be built. The following decisions are provisional and still to be agreed. Questions that affect several modules are listed with their number (OP-xx) in `../open-points.md`.

## Provisional decisions

| No. | Topic | Decision in the README | To be agreed with |
|---|---|---|---|
| 1 | Status of the company | `REQUESTED`, `ACTIVE`, `REJECTED`, `INACTIVE` (6.1). `REJECTED` is final; `INACTIVE` leads back through `reactivate-company`. Today there is also `DELETED`; dropped. | Architecture, project owner |
| 2 | Data for `create-company` | The step receives only `companyId` and `applicationId` and reads the application at C1-M2 (8.1, 11.1). Only for applications of kind `ONBOARDING` and `TRANSFER`; not for `REACTIVATION` and not on the return of a company that transferred away. | C1-M2, Process Orchestration |
| 3 | Company identifier only from the CSP | The BPN comes only through `assign-identifier` from `CompanyRegisteredAtCsp` (8.2). No BPN from the application, no BPN set by hand, no separate status of the company identifier (6.2). The CSP confirms an existing BPN from today's dataspace. | C3-M2, CSP, C1-M2 |
| 4 | Failure at the CSP | No event of its own in C3-M1 (previously `CompanyIdentifierFailed`). `CompanyRegistrationAtCspFailed` from C3-M2 is enough; the company stays `REQUESTED` without a BPN until the registration succeeds or `reject-company` follows (6.2). | C3-M2, Process Orchestration |
| 5 | Core data and source | Core data is name, address with country, EUID and EBWOID; source per company `EBW` or `MANUAL` with time of the last verification (6.3, OP-09). The source per identifier (`BUSINESS_WALLET`, `DECLARED`) remains alongside. | C1-M2, C3-M2, project owner |
| 6 | Mapping of the LPID | `street_address` → `street`, house number empty; with a changed address, region and addition are dropped; comparison after normalisation as in C1-M2 (6.3). | C1-M2, C5-M1 |
| 7 | Changed core data to the CSP | Yes. `CompanyDataChanged` carries `coreDataChanged: true`; Process Orchestration then passes the core data to the CSP through C3-M2 (6.3). Reason: the duplicate check at the CSP also compares name and country. C3-M2 passes it on with the step `update-company` (C3-M2 8.4, FRAMEWORK 5.6a). | C3-M2, CSP, Process Orchestration |
| 8 | Change with verification attestation (OP-10) | Verification attestation of the EBW-Connector of the own installation, purpose `onboarding` (C5-M1 0.2 knows no other), `aud` `portal-registration`, same EBWOID and same country, representation confirmed, `verified_at` at most 24 hours old and newer than the stored state, `jti` once. Takes effect immediately (6.7, 7.3). | C5-M1, project owner |
| 9 | Person in the verification attestation | Need not be the calling person; what counts is the confirmed representation. The Company Admin can have a representative confirm by QR code (6.7). | Project owner, data protection |
| 10 | Change request with supporting documents (OP-10) | The Company Admin submits it with 1 to 3 supporting documents (PDF, PNG, JPEG, each up to 10 MB) stored in the module's own database; at most one open per company; OSP staff decide with `review_company_changes`; nobody approves their own request; can be switched off with `CHANGE_REQUESTS_ENABLED` (7.4 to 7.6, 7.14 to 7.18). | Project owner, C2-M5, data protection |
| 11 | Source `EBW` | Name and address only through a new verification attestation; a change request for them gives `attestation-required`. If OSP staff correct these fields, the source becomes `MANUAL` (6.6). | Project owner |
| 12 | Change from `MANUAL` to `EBW` | Only with a verification attestation whose EBWOID was entered before as an identifier (source `DECLARED`), through a change request or a correction (6.6). Otherwise any Business Wallet could take over a company. | Project owner, C1-M2 |
| 13 | Country | Cannot be changed after the BPN has been assigned, not even with a verification attestation (6.6). | Project owner, CSP |
| 14 | Direct changes | Without approval the Company Admin changes only the short name; the address belongs to the core data (6.6). In version 0.1 the PSP was allowed to change the address directly. | Project owner |
| 15 | Company roles | `assignableBy` `COMPANY` or `SEED`; the Company Admin assigns and removes roles with consents, the time is the call. Seed role `ONBOARDING_SERVICE_PROVIDER` for the OSP company instead of `OPERATOR` (6.5). | C2-M3, C2-M5, project owner |
| 16 | Role catalogue | Configuration per installation with today's `agreementId`s; new version = new `agreementId`; no re-consent in version 0.2 (6.5, OP-11). | OSP, project owner |
| 17 | Role catalogue for people | no separate path any more. Applicants see it through C1-M2, people of a company through 7.7 (9.4). | C1-M2 |
| 18 | Deactivation | Operation of OSP staff (reason `MANUAL`, 7.13) and step `deactivate-company` (mainly reason `TRANSFERRED`, 8.5). The justification is only in the audit log. A later `TRANSFERRED` overwrites a `MANUAL`. | Process Orchestration, project owner |
| 19 | Reactivation | Step with `applicationId`; takes over the core data from the application; the kind must match the reason (`REACTIVATION` with `MANUAL`, `TRANSFER` with `TRANSFERRED`); roles and consents stay (8.6). | C1-M2, Process Orchestration, OP-38 |
| 20 | Permissions and portal roles | `view_company`, `edit_company`, `edit_company_roles` (scope `company`); `view_companies`, `manage_companies`, `review_company_changes` (scope `osp`); mapping to portal roles as in 6.8. `OSP Reviewer` decides change requests but does not deactivate. | C2-M5, OP-41 |
| 21 | Scope for steps and queries | `x-permission-scope: osp` for `query_c3_m1` and `execute_steps_c3_m1`; only the permission is checked (6.8). | Framework, C2-M5 |
| 22 | Paths for OSP staff | under `/osp/...` | Framework |
| 23 | Seed data | `OSP_COMPANY_FILE` with the OSP company; `companyId` equal to `ospCompanyId`; BPN from the file; source `MANUAL` (4). | OP-36, CSP (BPN of the OSP company) |
| 24 | Dataspace | Name from the installation settings in the response to people (`dataspaceName`); no membership in C3-M1 (OP-14). | C3-M2, C4-M1 |
| 25 | Identifier types | from the shared file of the reference data (OP-13); EUID and EBWOID with loose validation (OP-12); `VIES` dropped (6.4). | C1-M2, other part |
| 26 | Local duplicate search | stays alongside C3-M2 `find-companies`, for open applications that the CSP does not know yet (9.3). | C1-M3 |
| 27 | Bulk query | `GET /internal/companies` with repeated `id` or `bpn`, up to 100 values (9.2), as in version 0.1 | Offers, Subscriptions |
| 28 | Region | optional (required today), because the LPID does not provide it | C1-M2, C5-M1 |
| 29 | Last role | An active company keeps at least one role with `assignableBy` `COMPANY` (6.5). | Project owner |
| 30 | Field name of the company identifier | `bpn`, as long as the CSP assigns BPNs | Architecture |

## Additions to the framework

Taken over into FRAMEWORK 5 to 7 on 28 Oct 2026 (events with their fields, queries, flows); the table remains as a record.

| No. | Addition | Reason |
|---|---|---|
| E1 | Events `CompanyChangeRequestSubmitted` and `CompanyChangeRequestClosed` (not in the catalogue in FRAMEWORK 6) | Notices to OSP staff and to the Company Admin through Notifications (OP-19). |
| E2 | Additional fields: `CompanyCreated` with `applicationId`, `coreDataSource`; `CompanyActivated` and `CompanyReactivated` with `bpn`, `companyRoles`; `CompanyDataChanged` with `coreDataChanged`, `coreDataSource` | FRAMEWORK names the minimum content. Follow-up steps (C4-M1, C3-M2) need the BPN or the flag for changed core data. |
| E3 | Return of a company that transferred away: `reactivate-company` instead of `create-company` and `activate-company` (8.1, 8.6) | FRAMEWORK 5.6 describes only the transfer to a new OSP. C3-M2 provides for the return with the same `companyId` (C3-M2 6.1, `TRANSFERRED_AWAY` → `PENDING`). |
| E4 | Changed core data to the CSP: flow on `CompanyDataChanged` with `coreDataChanged: true` → C3-M2 `update-company` (decision 7) | now FRAMEWORK 5.6a and 7 |
| E5 | Query 9.3 (local duplicate search) for C1-M3 alongside C3-M2 `find-companies` | FRAMEWORK 7 names only `find-companies` for C1-M3. |

## Deviations from the framework

None. FRAMEWORK 4.1 and 4.3 have covered the scope `osp` for steps, queries and callbacks, and separate tags for callbacks, since 28 Oct 2026.

## Assumptions about other modules

| Module | Assumption |
|---|---|
| C1-M2 Registration | assigns the `companyId`; offers `get_application(applicationId)` with the fields from README 11.1, among them `kind`, `verificationMethod`, `jti` and `verifiedAt` of the verification attestation and, per consent, `consentedBy` (`sub`); checks company data by the same rules (README 5, 6.4) and consents against the role catalogue (9.4); shows the role catalogue in the application; reads the master data through 9.1 in reactivation and return and checks the EBWOID in doing so; represents the return of a company that transferred away as an application of kind `TRANSFER` with the existing `companyId`. |
| C1-M3 Application Review | uses C3-M2 `find-companies` and the local duplicate search 9.3; on the path without Business Wallet checks the supporting documents before `ApplicationApproved`, also in reactivation and return. |
| C3-M2 CSP Connection | reports `CompanyRegisteredAtCsp` with `companyId` and `bpn`, also in a transfer and on return (then with the same BPN); receives the data for `register-company` from 9.1 through Process Orchestration; reports `CompanyTransferredAway`, upon which the flow calls `deactivate-company` with `TRANSFERRED`; alone manages the membership (`register-membership`, `deregister-membership`); passes changed core data to the CSP with `update-company` (decision 7). |
| C5-M1 EBW-Connector | verification attestation as in C5-M1 9.1 (version 0.2), JWKS at `{iss}/jwks`, `aud` from `ATTESTATION_AUDIENCE`, purpose `onboarding` only. If C5-M1 later introduces a purpose for changes, C3-M1 accepts it in addition. |
| C2-M2 Users and App Access | binds the applicant to the company before `activate-company` (`assign-company-admin`); reads status and BPN through 9.1; locks and unlocks accounts after `CompanyDeactivated` or `CompanyReactivated`. |
| C2-M3 Technical Users | reads status, BPN and roles through 9.1; replaces `PORTAL_SERVICE_PROVIDER` and `OPERATOR` in its table of assignable roles with `ONBOARDING_SERVICE_PROVIDER`. |
| C2-M5 Portal IAM | creates the permissions from README 6.8 and bundles them into the portal roles as proposed there; the token of a person carries `company_id` if the account is bound. |
| C4-M1, C4-M2 | read name, BPN and status through 9.1; `registeringPspId` is dropped. |
| Process Orchestration | flows as in FRAMEWORK 5.2, 5.4 to 5.6 with the steps from README 8; return as in E3; passing on changed core data as in E4; inputs from the fields of the events. |
| Notifications | notice to OSP staff on `CompanyChangeRequestSubmitted`, to the Company Admin on `CompanyChangeRequestClosed` (OP-19). |
| Base skeleton | outbox, token validation, Problem Details. |

## Open questions

- **Changed core data at the CSP:** which fields the CSP needs for its duplicate check is fixed by the core services API (OP-42); C3-M2 passes on name, EUID and address (decision 7).
- **Change from `MANUAL` to `EBW`** without a previously entered EBWOID, for example if a company gets a Business Wallet later (decision 12).
- **Moving the registered office to another country:** excluded today (decision 13); new onboarding or a change at the CSP?
- **Consequences of removing a role:** May a company give up `APP_PROVIDER` while it has active offers?
- **New version of terms:** cross-module, OP-11.
- **Installation settings:** cross-module, OP-36.
- **Transfer and return:** cross-module, OP-38.
- **Accounts of people, OSP staff:** cross-module, OP-40 and OP-41 (for example whether the staff are bound to the OSP company).
- **Retention of supporting documents and audit logs:** cross-module, OP-20.
- **Notifications:** cross-module, OP-19.
- **Formats of EUID and EBWOID:** cross-module, OP-12.

## Out of scope

- Migration of the data from Portal 2.6.0 (tables `companies`, `addresses`, `company_identifiers`, `company_assigned_roles`, `consents`).
- Deletion of companies and supporting documents, retention and deletion periods (OP-20).
- Scanning supporting documents for malware.
- Assigning or removing company roles by OSP staff.
- Taking data, technical users and dataspace wallet along in a transfer (FRAMEWORK 5.6).
- Sites and address BPN.
- Use cases and company certificates.
- Self-description (Gaia-X) and Clearing House.
- Names in several languages.

## Dropped with OSP 2.0

Points of version 0.1 that no longer apply:

| Previously | Reason |
|---|---|
| Role `PORTAL_SERVICE_PROVIDER`, steps `assign-company-role` and `remove-company-role` (OP-04, OP-05) | no PSPs any more |
| Reading by the registering PSP, 403 `not-delegated` for unknown companies | no delegation; scope `company` with 404 as in FRAMEWORK 4.3 |
| Deactivation with reason `NO_PSP` (OP-06) | an OSP and its portal are one |
| Query at the master data service, identifier of the request, mapping to the format of the master data service, reporting the membership | BPN and membership run through C3-M2 and the CSP |
| BPN from the application, BPN set by hand | only the CSP assigns the BPN |
| Role `OPERATOR` | replaced by `ONBOARDING_SERVICE_PROVIDER` for the OSP company |
