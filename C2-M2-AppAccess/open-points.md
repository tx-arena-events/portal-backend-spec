# C2-M2 Users and App Access: open points

**Version:** 0.2, as of 2026-10-28

For version 0.2 the README settles each of these points so that the module can be built. The following decisions are provisional and still to be agreed. Decided on 28 Oct 2026 is OSP 2.0: every OSP runs its own installation with its own Portal IAM; people sign in with the EUDI Wallet through the EBW-Connector and call the modules with their own token; no delegation, no PSP, no identity providers per PSP. The binding framework is `../FRAMEWORK.md`; the version for the PSP architecture is in the git history of this repository.

## Provisional decisions

| No. | Topic | Decision in the README | To be agreed with |
|---|---|---|---|
| 1 | Accounts are created at the first login | The Portal IAM creates the account at the first login (C2-M5); the module never creates accounts and deletes none. `userId` = ID in the Portal IAM = `sub` in the token (README 1, 6.5) | C2-M5 (OP-40) |
| 2 | Binding | attributes `company_id`, `bpn` on the account, plus a row in `company_user`; at most one binding `ACTIVE` or `DISABLED` per account (README 5, 6.6) | C2-M5 (OP-40) |
| 3 | Removal from the company | unbinds (attributes, company portal roles, app roles gone; sessions ended); the account stays with `Applicant` (README 6.12.3) | project owner, data protection (OP-40) |
| 4 | Deactivation | the account stays enabled; company, company portal roles and app roles are removed and saved in `suspended_roles`; intermediate states `DISABLING`, `ENABLING` for repeatability (README 6.6, 8.9, 8.10). See open question 1 | project owner, Process Orchestration |
| 5 | Names in the module's own database | copy of given name and family name from the PID at binding, for lists; cleared on removal; in invitations only while they are open (README 5, 6.8) | data protection (OP-20) |
| 6 | Invitation code | 32 random bytes, base64url; only the SHA-256 stored; accepted only in the body; link with the code in the fragment recommended (README 6.8) | UI, data protection |
| 7 | Validity of an invitation | once, 7 days (`INVITATION_TTL_HOURS`); cleanup every hour (README 6.8) | project owner |
| 8 | Name check on acceptance | given name and family name in the token equal those of the invitation, normalised as in C5-M1 6.5; can be switched off with `INVITATION_NAME_CHECK`; after 5 failed attempts the invitation lapses (README 6.8, 7.10) | project owner, UI |
| 9 | Maximum of open invitations | 50 per company or for the staff (README 6.8) | project owner |
| 10 | Last Company Admin | through scope `company` the role cannot be removed and the person cannot be removed; the OSP staff may do it (README 6.7) | project owner |
| 11 | Last OSP Admin | the role cannot be removed (README 6.7, 7.16) | project owner (OP-41) |
| 12 | Staff and customers kept apart | staff roles only for accounts without a company or in the OSP company (`ospCompanyId`); accounts with staff roles can be bound only there (README 6.7 limit 4) | project owner (OP-41) |
| 13 | Setting up staff | through invitations of kind `STAFF`; directly only for accounts in the OSP company or accounts that are already staff (README 6.9, 7.16, 7.17) | project owner (OP-41) |
| 14 | Bootstrap | invitation of the first OSP Admin from `BOOTSTRAP_ADMIN_FILE` with the hash of a code generated during setup; the OSP Admin then invites the first Company Admin of the OSP company (README 6.9, 7.20) | operations (OP-36, OP-41) |
| 15 | `Applicant` | stays on every account; the module never assigns or removes it; permissions in scope `own` take effect only without a company (README 6.7) | C2-M5, C1-M2 |
| 16 | Permissions and portal roles | permissions with a scope (README 6.11); proposed bundling: `Applicant`: `accept_invitation`; `IT Admin`: `view_company_users`, `view_app_roles`; `Company Admin`: in addition `manage_company_users`, `assign_app_roles`; `Member`: none; `OSP Reviewer`: `view_users`; `OSP Admin`: `view_users`, `manage_users`, `manage_osp_staff` | C2-M5 |
| 17 | Paths with the company | `/companies/{companyId}/…` for scope `company`, `/osp/…` for scope `osp`; the check compares `{companyId}` with `company_id` in the token (README 6.11) | cross-module (path convention, OP-34 R4), C2-M3 |
| 18 | Roles in lists | no copy of the role assignments in the module's own database; lists read the portal roles per entry in the Portal IAM, the limit "last Company Admin" reads the accounts with the role (README 6.7, 7.1) | – |
| 19 | Deactivating the OSP company | `disable-company-users` rejects `ospCompanyId` (409, `retryable: false`), so that the staff does not lock itself out (README 8.9) | C3-M1, Process Orchestration |
| 20 | Return after a transfer | `assign-company-admin` sets a deactivated company back to `ACTIVE`; old bindings stay `DISABLED` (README 8.8) | Process Orchestration (OP-38) |
| 21 | BPN in the module | from the input of `assign-company-admin`; only for a company without a row of its own (mainly the OSP company from the seed data of C3-M1) from C3-M1 9.1 (README 6.6, 11.2) | C3-M1 |
| 22 | Client scopes of the instance clients | default `basic,acr,roles,web-origins,catena`, no optional ones; without `profile`, `email`, so that apps get no names (README 6.3, FRAMEWORK 4.6) | C2-M5, Offers |
| 23 | `directAccessGrantsEnabled` | `false` on all clients (today `true`). Apps that today sign in directly at the token endpoint with username and password will then no longer work. | Offers, project owner |
| 24 | End of a subscription | the instance client is disabled (today it stays enabled) | Subscriptions |
| 25 | Removing or renaming roles of an active app | not possible (as today) | Offers |
| 26 | Changing descriptions | step `update-role-descriptions` | Offers |
| 27 | Languages | `de`, `en` from the configuration; at least one description per role (as today) | Offers |
| 28 | Client ID of the template | `app-template-{appId}` | C2-M5 |
| 29 | Single-instance client | created only at the approval (today already when the instance kind is set, but disabled) | Offers |
| 30 | Name on the client | name of the app on the template and on every instance client; a later change of the app name is not carried over | Offers |
| 31 | Assigning app roles | `PUT` with a complete list per app, valid on all instance clients of the company's active subscriptions (as today). A new subscription does not carry roles over; assignments on disabled clients stay in place. | Subscriptions |
| 32 | Concurrency | advisory locks per app, per company, for the staff, per invitation, per account, in a fixed order (README 6.14) | base skeleton |
| 33 | Invitations by the staff | The OSP staff with `manage_users` invites people to any company (README 7.20 to 7.22), e.g. the first Company Admin of the OSP company or a new one when none is left | project owner (OP-41) |
| 34 | Kind of the module | Hackathon. Onboarding, Offers and Subscriptions need C2-M2; either it is built beforehand or the teams use a stand-in. | architecture, project owner |

## Additions to the framework

| Kind | Name | Reason |
|---|---|---|
| Event | `OspStaffRolesChanged` (`userId`, `addedRoles`, `removedRoles`) | Staff roles belong to no company; `UserRolesChanged` has `companyId` as a required field. |
| Query | `GET /internal/company-users` (README 9.4) | Notifications need recipients, e.g. all Company Admins of a company. |
| Content of events | beyond the minimum from FRAMEWORK 6: `via`, `portalRoles` (`UserAssignedToCompany`), `reason` (`UserRemovedFromCompany`), `appId`, `addedRoles`, `removedRoles` (`UserRolesChanged`), `userIds` (`CompanyUsersDisabled`, `CompanyUsersEnabled`) | Recipients should know what changed without asking back. `UserRolesChanged` reports app roles with `appId`; the events `AppUser…` from 0.1 are dropped. |

## Deviations from the framework

| No. | Deviation | Reason |
|---|---|---|
| A1 | Scope `own` for `POST /invitations/accept`: the check is not who created the object (FRAMEWORK 4.3), but that the operation changes only the caller's own account (`sub`) and that an invitation of kind `COMPANY` binds only an account without a company. | The Company Admin creates the invitation, not the invited person. Possession of the code and the name check replace the check of the creator. FRAMEWORK 5.3 explicitly names scope `own` for accepting. |
| A2 | Tag `Company` also for `POST /invitations/accept`, although the person does not belong to any company yet (FRAMEWORK 4.1: "calls by people of a company"). | There is no tag for people without a company; `OSP` does not fit. |

## Assumptions about other modules

What C2-M2 expects from other modules. The form is provisional until their specs for OSP 2.0 are available.

| Module | Assumption |
|---|---|
| C2-M5 Portal IAM | The EBW-Connector is the only identity provider; the browser flow redirects there without `kc_idp_hint`. The first broker login creates an account without a form and links it (alias and `sub` of the EBW-Connector); no required email address, no verification. Mappers of the identity provider: `given_name` → `firstName`, `family_name` → `lastName`; **no** mappers to `company_id` or `bpn`. Username without real names. Portal role `Applicant` for every new account through a mapper of the identity provider, not as a default role of the realm (C2-M5 5.5). |
| C2-M5 Portal IAM | Client `portal-core` with the portal roles `Applicant`, `Member`, `IT Admin`, `Company Admin`, `OSP Reviewer`, `OSP Admin` as composite roles and the permissions from README 6.11; user profile with `company_id` (UUID) and `bpn`, changeable only by administrators; client scope `catena` with mappers for `company_id`, `bpn` (and `preferred_username`); tokens of people as in FRAMEWORK 4.2 with `given_name`, `family_name`; a refresh returns changed attributes and roles. |
| C2-M5 Portal IAM | Service account `svc-c2-m2` with `manage-clients`, `view-clients`, `query-clients`, `manage-users`, `view-users`, `query-users`. Test installation with test persons with fixed IDs, a stand-in for the EBW-Connector and tokens for test persons (README 14). |
| C5-M1 EBW-Connector | returns `given_name` and `family_name` from the PID in the ID token; `sub` per client stays the same as long as names and date of birth stay the same (C5-M1 6.10). |
| C1-M2 Registration | keeps the `sub` of the applicant on the application and passes it to Process Orchestration as `applicantSub`; accepts applications of kind `ONBOARDING` only from people without `company_id` in the token, so that `assign-company-admin` does not fail with `user-already-bound`. |
| Process Orchestration | calls `assign-company-admin` with `companyId`, `bpn` (from `CompanyRegisteredAtCsp`) and `applicantSub` (FRAMEWORK 5.2, 5.6); `disable-company-users` on `CompanyDeactivated`, `enable-company-users` on `CompanyReactivated` (FRAMEWORK 5.4, 5.5); retries on `retryable: true`. |
| Subscriptions | query of the active subscriptions of a company to an app, filters `customerCompanyId`, `offerId`, `status`. |
| C3-M1 Company Profile | query 9.1 `GET /api/c3-m1/internal/companies/{companyId}` with `status` and `bpn`; creates the OSP company at the first start from seed data (`ACTIVE`, with BPN), without onboarding. |
| Offers | calls the steps for the template and the instance clients through Process Orchestration (FRAMEWORK 5.8); reads app roles through `/internal/app-roles`. |
| Notifications | use the events and `/internal/company-users` to notify people. |
| UI | shows the invitation code once and builds the link with the code in the fragment; fetches a new token after an invitation is accepted; shows people of a deactivated company that their company is deactivated and offers them the application for reactivation. |
| Token validation library | returns `sub`, `company_id`, `bpn`, `given_name`, `family_name` and the permissions; accepts a JWKS with a test key in the test (README 14.3). |

## Open questions

Questions that concern several modules are listed with their number (OP-xx) in `../open-points.md`.

1. **Reactivation (FRAMEWORK 5.4 and 5.5), proposal set on 28 Oct 2026, provisional:** (a) and (b) together. `disable-company-users` does not deactivate the accounts, but removes company, company portal roles and app roles and saves them (README 8.9); `enable-company-users` restores them (8.10). After `CompanyReactivated`, Process Orchestration first calls `enable-company-users`, then `assign-company-admin` with the applicant (FRAMEWORK 5.4, 5.5). To be agreed with the project owner and Process Orchestration.
2. **Cleaning up accounts without a company, deleting an account on request:** Every person with an EUDI Wallet can create an account by signing in. When are unused accounts deleted, and how does a person delete their account? Cross-module: OP-40, OP-20.
3. **Change of name:** If the name in the PID changes, the `sub` at the EBW-Connector changes, and a new account without a company is created. The Company Admin invites the person again and removes the old account. Is that enough, or is a way to merge accounts needed? (C5-M1 open question 2.)
4. **Company Admin through the mandate:** If a company no longer has a Company Admin, in the README only the OSP staff can help (7.13, 7.20), after a check outside the portal. Should a person with a mandate be able to make themselves Company Admin? For this, the result of the representation check of the EBW-Connector would have to reach the token or C2-M2.
5. **Staff roles:** bundling, four-eyes principle, bootstrap. Cross-module: OP-41.
6. **Is the name check sufficient?** Is checking given name and family name enough, or is more needed (e.g. the date of birth, which is not in the token today)? Several given names in the PID lead to mismatches if the Company Admin enters only the name the person goes by.
7. **Locking single accounts:** Should the OSP staff be able to lock an account, e.g. in case of misuse, without reference to a company?
8. **Tokens of several OSPs at apps and central services:** cross-module OP-37. **Apps across the boundaries of an OSP:** cross-module OP-39.
9. **Transfer:** people, bindings and app roles do not move along; the Company Admin invites again at the new OSP. Cross-module: OP-38.
10. **Notification about new app roles:** cross-module OP-19.
11. **Retention** of `role_assignment_log`, `audit_log` and ended invitations: cross-module OP-20.
12. **Permissions of the service account:** `manage-users` applies to all accounts and roles of the realm; check finer permissions. Cross-module: OP-26.
13. **Keycloak organizations** instead of attributes on the account: cross-module OP-28.
14. **Installation settings** (`ospCompanyId`): cross-module OP-36.

## Out of scope

- Migration of existing apps, accounts and roles from today's installation: cross-module OP-32.
- Deleting clients: after the end of a subscription they are only disabled; deactivating or deleting an app changes nothing on the clients (as today).
- Re-enabling a disabled instance client; a new subscription gets a new client.
- Changing the URL of a single-instance app after the approval: the step `update-instance-url` can do it; a flow at Offers for it is not described.
- Enforcing PKCE on the public clients (`pkce.code.challenge.method: S256`): more secure, but could break existing apps.
- Sign-in without a subscription: every person can sign in to every active instance client but gets no app roles there (as today). The app must check the roles.
- The BPN changes: the attribute `bpn` on bound accounts is not updated.
- Sending invitations by email; the module knows no email addresses.
- Search for accounts without a company by the staff.
- Performance: lists read the portal roles per entry (up to 100 calls to the Portal IAM); the limit "last Company Admin" reads all accounts with `Company Admin`; `add-app-roles` and `update-role-descriptions` call Keycloak per instance client. Measure with many companies.
