# C2-M3 Technical Users: open points

**Version:** 0.2, as of 2026-10-30

For version 0.2, the README fixes each of these points so that the module can be built. The following decisions are provisional and still to be agreed. Decided on 28 Oct 2026 is OSP 2.0: every OSP runs its own installation with its own Portal IAM; people call the modules with their own token; no delegation, no PSP and therefore no API access for PSPs. The binding framework is `../FRAMEWORK.md`; version 0.1, the version for the PSP architecture, is in the git history of this repository.

## Provisional decisions

| No. | Topic | Decision in the README | To be agreed with |
|---|---|---|---|
| 1 | Credentials of the wallet access | **Decided 21 Oct 2026:** stay at C4-M2 and are retrieved there once, now by a person of the company (scope `company`) (README 7.7) | C4-M2 |
| 2 | Credentials for the provider of a subscription | **Decided 21 Oct 2026:** not in the step result; when passing them on, Subscriptions has a new secret created (README 9.2) | Subscriptions (follow-up question R5) |
| 3 | Permissions and portal roles | `view_technical_users`, `manage_technical_users`, `delete_technical_users` in scope `company`; `view_all_technical_users` in scope `osp`; for modules `query_c2_m3`, `execute_steps_c2_m3`, `rotate_subscription_credentials`. Proposal for bundling: `IT Admin` and `Company Admin` get the three permissions in scope `company`; `OSP Reviewer` and `OSP Admin` get `view_all_technical_users` (README 6.4) | C2-M5 |
| 4 | OSP staff | see all technical users, change and delete none (README 6.3, 7.9, 7.10) | project owner (OP-41) |
| 5 | Paths with company | `/companies/{companyId}/technical-users…`; the check compares `{companyId}` with `company_id` in the token (README 6.4) | cross-cutting (path convention, OP-34 R4), C2-M2 |
| 6 | Deactivation | flag `disabled` next to the status instead of a separate status; client disabled, account, secret, roles and wallet access remain; change and rotate secret blocked, delete allowed (README 6.2, 8.4, 8.5) | C3-M1, Process Orchestration |
| 7 | Co-viewed accounts on deactivation | only accounts owned by the deactivated company; accounts on which it is only co-viewing remain (README 6.2) | Subscriptions |
| 8 | No events for deactivation and reactivation | steps 8.4 and 8.5 report nothing; the result goes to Process Orchestration (FRAMEWORK 6 does not provide any) | Process Orchestration, Notifications |
| 9 | Company ready | new technical users only for companies that are `ACTIVE` in C3-M1 and have a BPN; so none for deactivated ones (`INACTIVE`) (README 7.2, 8.1) | C3-M1 |
| 10 | Token content | attributes `company_id` and `bpn` on the service account; claims through the client scopes `catena` and `technical-roles`, `fullScopeAllowed` `false`; no separate BPN mapper per client (README 6.6) | C2-M5 |
| 11 | Assignable roles | configuration file in C2-M3: company role → roles (README 6.5); initial values without the roles that only contain rights on today's portal; `ONBOARDING_SERVICE_PROVIDER` gives no further roles; roles for managing central services (formerly `OPERATOR`) belong to the CSP | C2-M5, C3-M1, CSP (OP-24) |
| 12 | Changing the wallet role afterwards | not allowed; create the account again (README 7.4) | – |
| 13 | Wallet access failed | the account becomes `ACTIVE` with wallet `FAILED` and remains usable in the Portal IAM (README 6.1) | Process Orchestration (number of retries before `FAILED`) |
| 14 | Sign-in method of technical users | only `SECRET`; further methods later | – |
| 15 | No event of its own for the result of the wallet access | people learn it from `WalletAccountCreated` or `WalletAccountFailed` of C4-M2 (README 8.2, 10) | C4-M2, Notifications |
| 16 | Owner and co-viewing company | only the owner changes and rotates the secret (403 `not-owner`); the co-viewing company may also see and delete (README 6.3) | Subscriptions |
| 17 | Customer account without other roles | The customer account of a subscription is always created, even if no other roles remain after the split | Offers, Subscriptions |
| 18 | Connection to the neighbours | variable empty = stand-in, `in-process` = same process, otherwise HTTP address (FRAMEWORK 4.7) | base skeleton |
| 19 | End of a subscription | step `disable-subscription-technical-users`: the accounts become `INACTIVE` (final), client disabled, can only be deleted; today's portal also deactivates them (README 8.6). Alternative: delete them directly | C10-M1, C12-M1 |
| 20 | Connector as a service | `usableForConnector` with `offerSubscriptionId` also for the customer account of this subscription (README 9.1) | C6-M1 |
| 21 | Roles for the profiles of offers | new query `GET /internal/technical-roles` (README 9.3) | C8-M1 |

## Additions to the framework

No new events or steps. The queries `/internal/technical-users/{id}` (for Connectors) and `/internal/technical-users/{id}/credentials/rotate` (for Subscriptions, see A1) continue to apply as in version 0.1 (FRAMEWORK 7: further queries per module).

## Deviations from the framework

| No. | Deviation | Reason |
|---|---|---|
| A1 | `POST /internal/technical-users/{id}/credentials/rotate` is a call by another module that changes something (new secret), but is not a step; own permission `rotate_subscription_credentials` instead of `query_c2_m3` (FRAMEWORK 1, 4.2: modules call each other directly only for queries) | Step results must not contain secrets (FRAMEWORK 4.4), but Subscriptions needs the secret to pass it on to the provider (decision of 21 Oct 2026). |

## Assumptions about other modules

What C2-M3 expects from other modules. The form is provisional until their specs for OSP 2.0 are available.

| Module | Assumption |
|---|---|
| C2-M5 Portal IAM | client `technical_roles_management` with the roles from README 6.5 and descriptions as `description.{lang}`; clients of the central services and their roles with the same names in every installation; client scopes `catena` (mapper for `company_id`, `bpn`) and `technical-roles` (scope mapping of all roles of `technical_roles_management`); the user profile allows `company_id` and `bpn` on service account users; service account `svc-c2-m3`; short lifetime of the access tokens; portal roles with the permissions from README 6.4. |
| C3-M1 Company Profile | query 9.1 `GET /api/c3-m1/internal/companies/{companyId}` → `{ id, status, bpn, companyRoles }` with active roles only; status `ACTIVE` only for active companies, `INACTIVE` after deactivation. C2-M3 does not need the role catalogue (C3-M1 9.4); the mapping company role → technical user roles stays in `ROLE_ASSIGNMENT_FILE`. |
| C4-M2 Dataspace Wallet Interface | steps `create-wallet-account` (input from `WalletAccessRequested`: `technicalUserId`, `companyId`) and `delete-wallet-account` (from `WalletAccessRemovalRequested`: additionally `walletAccountRef`); events `WalletAccountCreated`, `WalletAccountFailed`, `WalletAccountDeleted` with `walletAccountRef`; one-time retrieval of the credentials by a person of the company in scope `company`; no new wallet accounts for deactivated companies. |
| Process Orchestration | flows from FRAMEWORK 5.4, 5.5, 5.7 and 5.8: `create-for-subscription` in the subscription flow, then, where needed, waiting for the wallet access; `disable-company-technical-users` on `CompanyDeactivated`, `enable-company-technical-users` on `CompanyReactivated`. |
| Subscriptions | query `GET /internal/subscriptions/{offerSubscriptionId}` → `{ status }`; calls 9.2 to pass the secret of a subscription account on to the provider. |
| Connectors | query `GET /internal/connectors?technicalUserId=` → list with `status`; use `usableForConnector` from 9.1, for a connector as a service with `offerSubscriptionId`. |
| Notifications | show the events of this module to the people of the company and to the co-viewing provider. |
| Token validation library | provides `sub`, `company_id` and the permissions; accepts a JWKS with a test key in the test (README 14.2). |

## Open questions

Questions that affect several modules are listed with their number (OP-xx) in `../open-points.md`.

1. **Tokens of technical users at central services of the CSP:** the tokens are issued by the OSP's Portal IAM. Central services at the CSP must trust this issuer. Proposal: the issuer of the Portal IAM of every admitted OSP is in the trust list; central services accept their tokens (README 6.7). Cross-cutting OP-37.
2. **Functions that technical users call on the portal today** (member list, search for connectors): where they will live in future and which roles remain on the client `technical_roles_management`. Cross-cutting OP-24.
3. **Transfer:** technical users do not move along; the company creates them again at the new OSP, with new secrets (FRAMEWORK 5.6). Cross-cutting OP-38.
4. **Subscription accounts across OSP boundaries:** if customer and provider are registered at different OSPs, their accounts are in different Portal IAMs; there is then no co-viewing company. Cross-cutting OP-39.
5. **Intervention by the staff:** should OSP staff be able to block or delete the technical users of a company, for example in case of misuse or a leaked secret?
6. **Accounts of a subscription when the subscription ends** (today `INACTIVE`); the step `delete` requires a deleted wallet access.

## Out of scope

- Accounts of apps with only one instance (today table `app_instance_assigned_technical_users`).
- Migration of existing technical users: cross-cutting OP-32.
- Further sign-in methods for technical users (`private_key_jwt`).
- Retention of `role_assignment_log` and `audit_log`: cross-cutting OP-20.
- Rights of the service account (`manage-clients` applies to all clients of the realm): cross-cutting OP-26.
- Bound tokens (DPoP, mTLS) for technical users: cross-cutting OP-27.
