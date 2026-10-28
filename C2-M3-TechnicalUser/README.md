# C2-M3 Technical Users

**Version:** 0.2, as of 2026-10-28
**Capability:** C2 Identity and Access
**Kind:** Hackathon
**Files:** this README (behaviour), [openapi.yaml](openapi.yaml) (interface), [open-points.md](open-points.md)

This README and the OpenAPI description are sufficient to build the module. The binding framework is [`../FRAMEWORK.md`](../FRAMEWORK.md) (OSP 2.0). Modules from other parts of the spec are referred to by name.

## 1. Purpose

The module manages, in the Portal IAM of the onboarding service provider (OSP), the accounts of the companies' programs:

- **Technical users:** one client with service account per account, sign-in with client ID and secret, roles on the client `technical_roles_management`, `company_id` and `bpn` in the token. Programs use these tokens to call apps and central services; they do not call the modules of the installation (FRAMEWORK 4.2). The module creates the accounts, shows them, changes them, rotates their secret and deletes them.
- **Wallet access** as a property of a technical user: if an account needs access to the company's dataspace wallet, the module reports this as an event. C4-M2 creates the wallet account; the module applies the result as a step (FRAMEWORK 5.7).
- **Deactivation together with the company:** if a company is deactivated, the module disables its technical users without deleting them, and enables them again on reactivation (FRAMEWORK 5.4, 5.5).

It is the only module that writes these clients into the Portal IAM. It has no user interface of its own; the UI of the OSP calls it.

| Who | What for |
|---|---|
| IT Admin and Company Admin of a company (scope `company`) | create, view, change and delete technical users; rotate the secret; read assignable roles |
| OSP staff (scope `osp`) | view the technical users of all companies |
| Process Orchestration | steps: create accounts for a subscription; apply wallet access; delete an account after the wallet account has been deleted; disable and re-enable the technical users of a company |
| Subscriptions | rotate the secret of a subscription account to pass it on to the provider |
| Connectors | check whether a technical user is suitable for a connector |

## 2. Terms

| Term | Meaning |
|---|---|
| Technical user, account | account of a program (client with service account in the Portal IAM); belongs to exactly one company |
| Technical user role | client role on the client `technical_roles_management` (C2-M5), assigned to the service account; composed of roles on the clients of the central services. Not a portal role and not an app role. |
| Wallet role | role that triggers wallet access (configuration `WALLET_ROLES`). It is **not** assigned in the Portal IAM. |
| Provider-only role | role that only the provider receives in a subscription (configuration `PROVIDER_ONLY_ROLES`) |
| Wallet access | account of the technical user in the company's dataspace wallet (wallet account); created by the Dataspace Wallet Interface (C4-M2). Its identifier at C4-M2 is called `walletAccountRef`. |
| Origin | `OWN`: created by a person of the company (7.2). `SUBSCRIPTION`: created when a subscription is activated (8.1). |
| Subscription party | for origin `SUBSCRIPTION`: `CUSTOMER` (account of the customer) or `PROVIDER` (account of the provider with the provider-only roles) |
| Co-viewing company | company that may see and delete an account of another company: for customer accounts from a subscription, the provider |
| Disabled | account of a deactivated company (`disabled`): client disabled in the Portal IAM, account and roles remain (6.2) |
| OSP staff | people with the portal roles `OSP Reviewer` or `OSP Admin` |

## 3. Not part of this module

| What | Where |
|---|---|
| Accounts of people, portal roles, app roles | C2-M2 Users and App Access |
| Realm, client `technical_roles_management` and its roles, client scopes, clients of the central services, token lifetime | C2-M5 Portal IAM (configuration) |
| Create and delete the wallet account; keep the credentials of the wallet account and hand them out once | C4-M2 Dataspace Wallet Interface |
| Company data, BPN, company roles, status of the company | C3-M1 Company Profile |
| Link between a connector and an account | Connectors |
| Profiles of the offers (which roles subscription accounts receive) | Offers |
| Subscription flow, passing the credentials on to the provider | Subscriptions and Process Orchestration |
| Trust of central services at the CSP in the Portal IAM of the OSP | CSP, trust list (OP-37) |
| Notifications to people | Notifications (other part), from the events of this module |
| Order of the steps, retries | Process Orchestration |

## 4. Technical frame

- Python with FastAPI.
- Own database `c2_m3` in PostgreSQL with its own database user. The module reads and writes only there. Other modules read through the interface.
- People call through the UI of the OSP over HTTP, with their token from the Portal IAM. The shared token validation library checks signature, expiry and audience (`aud` contains `portal-core`) and provides the claims: `sub`, `company_id` (only for a bound account) and the permissions (FRAMEWORK 4.2).
- Every operation requires exactly one permission with one scope (6.4, FRAMEWORK 4.3).
- Process Orchestration and other modules call the module **in the same process** through its public interface, without a token. [openapi.yaml](openapi.yaml) describes both as the same contract; the paths under `/steps` and `/internal` apply as HTTP paths when the module runs as a separate service.
- Events go out through the outbox of the base skeleton: written in the same transaction as the change, delivered afterwards (section 10). The module does not call Process Orchestration.
- Secrets never appear in logs, events, step results or in the module's database. The secret of a technical user is kept only in Keycloak and appears only in the response that creates it.

### Configuration

| Variable | Meaning | Example |
|---|---|---|
| `DATABASE_URL` | own PostgreSQL database `c2_m3` | |
| `IAM_URL` | base address of the OSP's Portal IAM | `https://iam.osp-a.example.org/auth` |
| `IAM_REALM` | realm of the installation | `CX-Central` |
| `IAM_ADMIN_CLIENT_ID`, `IAM_ADMIN_CLIENT_SECRET` | the module's own service account for the Admin API; needs the rights `manage-clients`, `view-clients`, `query-clients`, `manage-users`, `view-users`, `query-users` in the realm | |
| `TECHNICAL_ROLES_CLIENT` | client with the technical user roles | `technical_roles_management` |
| `CLIENT_ID_PREFIX` | prefix of the client ID of technical users | `sa` |
| `COMPANY_ID_ATTRIBUTE`, `BPN_ATTRIBUTE` | names of the attributes on the service account user | `company_id`, `bpn` |
| `DEFAULT_CLIENT_SCOPES` | default client scopes of the clients of technical users, comma-separated | `basic,roles,catena,technical-roles` |
| `WALLET_ROLES` | wallet roles, comma-separated | `Identity Wallet Management` |
| `PROVIDER_ONLY_ROLES` | provider-only roles | `BPDM Pool Consumer,BPDM Sharing Output Consumer,BPDM Sharing Input Manager` |
| `ROLE_ASSIGNMENT_FILE` | JSON file: company role → assignable roles (6.5) | `role-assignment.json` |
| `COMPANY_URL`, `CONNECTOR_URL`, `SUBSCRIPTION_URL` | connection to the neighbours C3-M1, Connectors and Subscriptions: empty = built-in stand-in (11.2), `in-process` = call in the same process, otherwise HTTP address of the neighbour running as a separate service | `in-process` |
| `PROCESS_STUB` | `true` = built-in stand-in for Process Orchestration, for testing the module alone (11.3); `false` in operation | `false` |

## 5. Data

### Table `technical_user`

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key |
| `company_id` | uuid | owner; required |
| `name` | text | 1 to 255 characters; required; need not be unique. For `OWN` from the input (1 to 80), for `SUBSCRIPTION` `{iam_client_id}-{Name}` |
| `description` | text | up to 255 characters |
| `origin` | text | `OWN` or `SUBSCRIPTION` |
| `offer_subscription_id` | uuid | required for `SUBSCRIPTION`, otherwise empty |
| `subscription_party` | text | `CUSTOMER` or `PROVIDER`; required for `SUBSCRIPTION`, otherwise empty |
| `co_viewer_company_id` | uuid | co-viewing company or empty |
| `iam_client_id` | text | unique; `CLIENT_ID_PREFIX` + value of the sequence `client_number`, for example `sa123` |
| `iam_client_uuid` | text | internal ID of the client in Keycloak; empty as long as a step has not completed the creation |
| `iam_service_account_user_id` | text | ID of the service account user in Keycloak |
| `status` | text | `PENDING`, `ACTIVE`, `PENDING_DELETION`, `DELETED` |
| `wallet_access` | text | `NONE`, `REQUESTED`, `ACTIVE`, `FAILED`, `REMOVING`; default `NONE` |
| `wallet_account_ref` | text | identifier of the wallet account at C4-M2 |
| `disabled` | boolean | `true` as long as the owner's company is deactivated (6.2); default `false` |
| `created_at`, `changed_at` | timestamptz | |

Unique: `offer_subscription_id` with `subscription_party` among the rows with a status other than `DELETED` (partial unique index). Deleted accounts are kept with status `DELETED`. The sequence `client_number` lives in the module's database.

### Table `role_assignment_log`

Audit log of every change of a role assignment (6.8). The admin events of Keycloak are not enough for this: they only name the calling client, which is the service account of this module.

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key |
| `occurred_at` | timestamptz | when |
| `technical_user_id` | uuid | to whom |
| `company_id` | uuid | owner of the account |
| `iam_client_id` | text | client ID of the account, for example `sa123` |
| `role_name` | text | which role on the client `TECHNICAL_ROLES_CLIENT` |
| `action` | text | `ADDED` or `REMOVED` |
| `actor_type` | text | `PERSON` (operation) or `PROCESS` (step) |
| `actor_sub` | uuid | for `PERSON`: `sub` of the acting person |
| `process_id`, `step_run_id` | uuid | only for `PROCESS` |
| `operation` | text | `operationId` or name of the step |

### Table `audit_log`

One entry per changing operation and per changing execution of a step (FRAMEWORK 4.3).

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key |
| `occurred_at` | timestamptz | |
| `actor_type` | text | `PERSON` or `PROCESS` |
| `actor_sub` | uuid | for `PERSON` from the token |
| `company_id` | uuid | company from the path or from the input of the step |
| `process_id`, `step_run_id` | uuid | for `PROCESS` |
| `operation` | text | `operationId` from [openapi.yaml](openapi.yaml) or name of the step, for example `createTechnicalUser` |
| `object_id` | uuid | ID of the technical user |

**Not stored:** roles (only in the Portal IAM); BPN (at C3-M1; in the Portal IAM only as an attribute on the service account); secrets of the technical users (only in Keycloak); credentials of the wallet accounts (at C4-M2); link with a connector (at Connectors); names of people; progress of individual steps (steps find their object through business keys, 6.9).

## 6. Rules

### 6.1 Status of a technical user

| From | Trigger | To | Event of the module |
|---|---|---|---|
| – | person creates, without wallet role (7.2) | `ACTIVE`, wallet `NONE` | `TechnicalUserCreated` |
| – | person creates, with wallet role (7.2) | `PENDING`, wallet `REQUESTED` | `TechnicalUserCreated`, `WalletAccessRequested` |
| – | step `create-for-subscription` creates the row (8.1) | `PENDING`, client not yet created | – |
| `PENDING`, client not created | step completes the creation, without wallet role | `ACTIVE`, wallet `NONE` | `TechnicalUserCreated` |
| `PENDING`, client not created | step completes the creation, with wallet role | `PENDING`, wallet `REQUESTED` | `TechnicalUserCreated`, `WalletAccessRequested` |
| `PENDING`, wallet `REQUESTED` | `apply-wallet-access` with `ACTIVE` | `ACTIVE`, wallet `ACTIVE` | – |
| `PENDING`, wallet `REQUESTED` | `apply-wallet-access` with `FAILED` | `ACTIVE`, wallet `FAILED` | – |
| `ACTIVE`, wallet `NONE` or `FAILED` | delete (7.5) | `DELETED`, wallet `NONE` | `TechnicalUserDeleted` |
| `ACTIVE`, wallet `ACTIVE` | delete (7.5) | `PENDING_DELETION`, wallet `REMOVING` | `WalletAccessRemovalRequested` |
| `PENDING_DELETION`, wallet `REMOVING` | step `delete` (8.3) | `DELETED`, wallet `NONE` | `TechnicalUserDeleted` |

Deactivation does not change this status; it is a separate flag (6.2).

### 6.2 Deactivation together with the company

- `disable-company-technical-users` (8.4) sets `disabled` to `true` on all accounts that belong to the company (`company_id`) and disables their clients in the Portal IAM. Programs then get no new tokens; tokens already issued stay valid until they expire (C2-M5).
- `enable-company-technical-users` (8.5) resets `disabled` and enables the clients again. Secret, roles, attributes and wallet access are unchanged.
- Accounts on which the company is only co-viewing are not touched; they belong to another company.
- As long as an account is disabled: change (7.4) and rotate secret (7.6, 9.2) return 409 `invalid-state`. Delete (7.5) and the steps `apply-wallet-access` and `delete` still work. `usableForConnector` is `false` (9.1).
- No new technical users are created for a company that is not `ACTIVE` in C3-M1 (7.2, 8.1: 409 `company-not-ready`). C4-M2 creates no new wallet accounts for a deactivated company (FRAMEWORK 5.4).
- The wallet account of a disabled account remains at C4-M2.

### 6.3 Visibility

A person acts for the company from `company_id` in their token; it must equal `{companyId}` in the path. The person sees the accounts whose `company_id` or `co_viewer_company_id` is this company. Other accounts do not exist for them: 404 `not-found`.

| Operation | Owner | Co-viewing company |
|---|---|---|
| list, details | yes | yes |
| create | yes (the company becomes the owner) | – |
| change, rotate secret | yes | no: 403 `not-owner` |
| delete | yes | yes |

OSP staff see the accounts of all companies (7.9, 7.10), but change nothing.

### 6.4 Permissions

| Permission | Scope | Operations |
|---|---|---|
| `view_technical_users` | `company` | list, details, assignable roles (7.1, 7.3, 7.8) |
| `manage_technical_users` | `company` | create, change, rotate secret (7.2, 7.4, 7.6) |
| `delete_technical_users` | `company` | delete (7.5) |
| `view_all_technical_users` | `osp` | view the technical users of all companies (7.9, 7.10) |
| `rotate_subscription_credentials` | `osp` | rotate the secret of a subscription account (9.2); only the service account of Subscriptions, only over HTTP |
| `query_c2_m3` | `osp` | query 9.1, only over HTTP |
| `execute_steps_c2_m3` | `osp` | steps under `/steps` (section 8), only over HTTP |

Checks on calls by people, in this order:

1. Token valid, otherwise 401 `unauthorized`.
2. Permission in the token, otherwise 403 `forbidden`.
3. Scope `company`: `company_id` in the token equals `{companyId}` in the path, otherwise 404 `not-found`. The account in the path is visible to this company (6.3), otherwise 404 `not-found`.
4. Scope `osp`: only the permission.

C2-M5 decides which portal role contains which permissions. Proposal of this module: `IT Admin` and `Company Admin`: `view_technical_users`, `manage_technical_users`, `delete_technical_users`; `OSP Reviewer` and `OSP Admin`: `view_all_technical_users`.

### 6.5 Assignable roles

Which roles a company may give its technical users follows from its active company roles (from C3-M1, 9.1) and the file `ROLE_ASSIGNMENT_FILE`. The assignable roles are the union over all company roles. Initial values, based on today's portal:

| Company role | Assignable roles |
|---|---|
| `ACTIVE_PARTICIPANT` | Dataspace Discovery, Identity Wallet Management, Semantic Model Management |
| `APP_PROVIDER` | BPDM Sharing Input Consumer, BPDM Sharing Output Consumer, Dataspace Discovery, Identity Wallet Management, Semantic Model Management |
| `SERVICE_PROVIDER` | same as `APP_PROVIDER` |
| `ONBOARDING_SERVICE_PROVIDER` | no further roles; the OSP's company receives the roles of its other company roles |

- Compared with today's portal, `Connector Management`, `CX Membership Info`, `Offer Management`, `Registration External` and `Registration Internal` are missing: they only contain rights on today's portal (C2-M5), and technical users do not call the modules (FRAMEWORK 4.2). Where these functions will live in future is open (OP-24).
- The row `OPERATOR` (all BPDM roles) is dropped together with the company role `OPERATOR` (C3-M1). Roles for managing central services belong to the CSP, not to an installation.

This check does not apply to subscription accounts. There, the roles from the profile of the offer apply; they must exist on the client `TECHNICAL_ROLES_CLIENT`.

### 6.6 Account of a technical user in the Portal IAM

| Setting | Value |
|---|---|
| `clientId` | `CLIENT_ID_PREFIX` + next value of the sequence `client_number`, for example `sa123` |
| `name`, `description` | name and description of the account |
| `enabled` | `true`; `false` as long as the account is disabled (6.2) |
| `protocol` | `openid-connect` |
| `publicClient` | `false` |
| `clientAuthenticatorType` | `client-secret` |
| `serviceAccountsEnabled` | `true` |
| `standardFlowEnabled`, `implicitFlowEnabled`, `directAccessGrantsEnabled` | `false` |
| `fullScopeAllowed` | `false` |
| `defaultClientScopes` | `DEFAULT_CLIENT_SCOPES`; no optional scopes |
| Service account user | attributes `{COMPANY_ID_ATTRIBUTE}: [companyId of the owner]`, `{BPN_ATTRIBUTE}: [BPN of the owner]` |
| Roles | roles on the client `TECHNICAL_ROLES_CLIENT`, assigned to the service account; never wallet roles |

The client scope `catena` writes the claims `company_id` and `bpn` into the token, the client scope `technical-roles` the roles (both from C2-M5). A separate mapper per client is not needed.

### 6.7 Tokens for apps and central services

- Programs get tokens from the OSP's Portal IAM: OAuth 2.0 client credentials with client ID and secret at `{IAM_URL}/realms/{IAM_REALM}/protocol/openid-connect/token`. The issuer (`iss`) is the Portal IAM of this OSP.
- Apps and services that trust the OSP's Portal IAM validate the tokens against it.
- **Central services at the CSP** do not know the Portal IAM of an OSP at first. Proposal (OP-37): the issuer of the Portal IAM of every admitted OSP is in the trust list (FRAMEWORK 5.1); central services accept tokens of these issuers and evaluate `bpn` and the roles of their own client. For this, the clients of the central services and the roles on the client `technical_roles_management` have the same names in every installation (C2-M5).
- The module changes nothing for this; it only hands out the token address of its own Portal IAM. Until OP-37 is decided, the tokens only work with apps and services that trust the Portal IAM of this OSP.

### 6.8 Audit log

- Every changing operation and every step that changes something writes an entry to `audit_log`, in the same transaction as the change.
- Every added or removed role writes an entry to `role_assignment_log`: on creation `ADDED` per role, on a change of the roles one entry per change, on deletion `REMOVED` per role the account had until then.
- Without secrets, without names.

### 6.9 Repetition and concurrency

- Every step finds its object through a business key from the input: subscription ID and subscription party for `create-for-subscription`, the ID of the technical user for `apply-wallet-access` and `delete`, the `companyId` for `disable-company-technical-users` and `enable-company-technical-users`. It creates only what is missing, sets only what differs, and returns the same result. `processId` and `stepRunId` only serve the audit log. An event is only created if something has changed.
- The client ID is formed and stored **before** a step creates the client in the Portal IAM. A repeated step finds the client again through it.
- Every step holds a PostgreSQL advisory lock on its key while it runs; operations on an account hold a lock on its ID. `disable-company-technical-users` and `enable-company-technical-users` additionally take the lock of each account.
- `POST …/technical-users` is not an idempotent operation: every call creates a new account. If the creation in the Portal IAM fails, the module removes the created client again (7.2).

## 7. Operations

All operations in this section are called by people over HTTP with their token. Permissions and checks as in 6.4. Lists as in FRAMEWORK 4.1: `page` from 0, `size` 1 to 100 (default 20), response `{ items, page, size, total }`.

**Technical users of the company** (tag `Company`)

### 7.1 `GET /companies/{companyId}/technical-users`

- Accounts visible to the company (6.3), sorted by `created_at` descending, page by page.
- Filters: `status` (can be given several times; without the filter `PENDING`, `ACTIVE`, `PENDING_DELETION`), `origin`, `walletAccess`.
- Entries without roles (`TechnicalUserSummary`), so that the list needs no calls to the Portal IAM.
- `viewerRole` is `OWNER` if `company_id` is the company from the path, otherwise `CO_VIEWER`.

### 7.2 `POST /companies/{companyId}/technical-users`

1. Checks as in 6.4. Validate the input, otherwise 400 `validation`: `name` 1 to 80 characters; `description` up to 255; `roles` not empty, no duplicate names; `authMethod` missing or `SECRET`.
2. Read the company at C3-M1. Not `ACTIVE` or without BPN: 409 `company-not-ready`. A deactivated company is not `ACTIVE`; so no new account is created for it.
3. Every role must be assignable as in 6.5 and exist on the client `TECHNICAL_ROLES_CLIENT`, otherwise 400 `role-not-assignable`.
4. Split the roles: wallet roles (`WALLET_ROLES`) and other roles. Provider-only roles are not treated separately for `OWN`.
5. Get the next number from `client_number`, form the client ID.
6. In the Portal IAM, in this order (11.1):
   1. Create the client (6.6).
   2. Read the service account user.
   3. Set the attributes `company_id` and `bpn` on the service account user.
   4. Assign the other roles.
   5. Read the secret.

   If one of these fails: delete the client again, 502 `iam-error`.
7. Write the row in `technical_user`: `origin` `OWN`; without wallet role status `ACTIVE`, with wallet role status `PENDING` and wallet `REQUESTED`. In the same transaction: entry in `audit_log` with `actor_sub`, one entry `ADDED` in `role_assignment_log` per assigned role, event `TechnicalUserCreated`, with wallet role also `WalletAccessRequested`. If that fails: delete the client, 500.
8. Response 201 with the account and `credentials` (client ID, secret, token address). The secret appears only in this response.

With a wallet role, Process Orchestration starts the flow "Wallet access" on `WalletAccessRequested` (FRAMEWORK 5.7).

### 7.3 `GET /companies/{companyId}/technical-users/{id}`

Read the roles from the Portal IAM (roles of the service account user on the client `TECHNICAL_ROLES_CLIENT`). If the wallet access is not `NONE`, the wallet role is also in `roles`. With wallet `ACTIVE`, the response contains `walletAccountRef`. No secret.

### 7.4 `PATCH /companies/{companyId}/technical-users/{id}`

- Only the owner (6.3), otherwise 403 `not-owner`. Only with status `ACTIVE` and not disabled, otherwise 409 `invalid-state`.
- `name`, `description`: change them in the account and on the client in the Portal IAM.
- `roles`: new complete list. Check as in 7.2 step 3; remove roles that are no longer listed, assign new ones. Adding or removing a wallet role is not allowed: 409 `wallet-role-change`.
- In the same transaction: entry in `audit_log`, one entry in `role_assignment_log` per changed assignment.
- Response 200 with the account.

### 7.5 `DELETE /companies/{companyId}/technical-users/{id}`

1. Checks as in 6.4; owner or co-viewing company. Wallet `REQUESTED`: 409 `wallet-in-progress`. Status not `ACTIVE`: 409 `invalid-state`.
2. Ask Connectors whether a connector uses the account. Yes: 409 `in-use`.
3. For `SUBSCRIPTION`: ask Subscriptions whether the subscription is active. Yes: 409 `in-use`.
4. Wallet `NONE` or `FAILED`: read the roles of the service account (for the audit log), delete the client in the Portal IAM, status `DELETED`, wallet `NONE`. In the same transaction: `audit_log`, one entry `REMOVED` per role, event `TechnicalUserDeleted`. Response 204.
5. Wallet `ACTIVE`: status `PENDING_DELETION`, wallet `REMOVING`. In the same transaction: `audit_log`, event `WalletAccessRemovalRequested`. Response 202 with the account. The step `delete` (8.3) deletes the client later, after C4-M2 has deleted the wallet account.

A disabled account can be deleted.

### 7.6 `POST /companies/{companyId}/technical-users/{id}/credentials/rotate`

Only the owner, otherwise 403 `not-owner`. Only with status `ACTIVE` or `PENDING` with a created client and not disabled, otherwise 409 `invalid-state`. Create a new secret in the Portal IAM; the old one is no longer valid afterwards. Entry in `audit_log`. Response 200 with `credentials`. The wallet access is not affected.

### 7.7 Credentials of the wallet access

This module does not hand them out. C4-M2 keeps the credentials of the wallet account encrypted; a person of the company retrieves them there once (scope `company`, FRAMEWORK 5.7), after C4-M2 has reported `WalletAccountCreated`. The link to the technical user is the ID of the technical user or `walletAccountRef`.

### 7.8 `GET /companies/{companyId}/technical-users/assignable-roles`

Roles as in 6.5 for the company from the path. Name and description from the Portal IAM: attribute `description.{lang}` of the role, otherwise the field `description`. Flags `grantsWalletAccess` (in `WALLET_ROLES`) and `providerOnly` (in `PROVIDER_ONLY_ROLES`).

**Technical users of all companies** (tag `OSP`)

### 7.9 `GET /osp/technical-users`

Like 7.1 across all companies, without the visibility check. Additional filters `companyId` and `disabled`. `viewerRole` is `OSP`.

### 7.10 `GET /osp/technical-users/{id}`

Like 7.3 for any account; `viewerRole` is `OSP`. Unknown: 404 `not-found`.

## 8. Steps for Process Orchestration

Common rules (FRAMEWORK 4.4):

- Every step receives `processId` and `stepRunId`, only for the audit log and traceability.
- Every step is idempotent (6.9).
- All steps of this module are short: they are done when they respond. The result has `state` `DONE`. Long-running operations (creating and deleting the wallet account) run at C4-M2 and end with its events.
- On an error, the step reports a problem with `retryable` (section 12) and leaves in place what it has already created; a retry completes it.
- Results contain no secrets.
- Every step names its **preconditions**. Process Orchestration decides the order, not the module.

For orientation, the flows that use the steps (FRAMEWORK 5):

| Flow | Triggered by | Steps of this module |
|---|---|---|
| Subscription activated (FRAMEWORK 5.8) | Subscriptions | `create-for-subscription` |
| Wallet access (FRAMEWORK 5.7) | `WalletAccessRequested` → C4-M2 `create-wallet-account` → `WalletAccountCreated` or `WalletAccountFailed` | `apply-wallet-access` |
| Remove wallet access (FRAMEWORK 5.7) | `WalletAccessRemovalRequested` → C4-M2 `delete-wallet-account` → `WalletAccountDeleted` | `delete` |
| Deactivation of a company (FRAMEWORK 5.4) | `CompanyDeactivated` | `disable-company-technical-users` |
| Reactivation of a company (FRAMEWORK 5.5) | `CompanyReactivated` | `enable-company-technical-users` |

### 8.1 `create-for-subscription`

Input: `customerCompanyId`, `providerCompanyId`, `offerSubscriptionId`, `name`, `roles` (from the profile of the offer).

Preconditions: customer with status `ACTIVE` and BPN (C3-M1); if the roles contain provider-only roles, the provider as well. Every role exists on the client `TECHNICAL_ROLES_CLIENT`.

1. Check, otherwise 400 `validation`: `name` 1 to 200 characters, `roles` not empty, no duplicate names. Unknown role: 400 `role-not-assignable`. Company not ready: 409 `company-not-ready`.
2. Split the roles into wallet roles, provider-only roles and other roles.
3. **Customer account**, key `offerSubscriptionId` + `CUSTOMER`:
   1. Row exists: continue with it. Otherwise draw the next number and create the row: `company_id` = customer, `co_viewer_company_id` = provider, `origin` `SUBSCRIPTION`, name `{client ID}-{name}`, status `PENDING`, wallet `NONE`.
   2. In the Portal IAM: search the client by its client ID, create it if it is missing (6.6); read the service account user; set the customer's attributes `company_id` and `bpn` if they differ; assign missing other roles.
   3. If `iam_client_uuid` was empty before: store it; without wallet role status `ACTIVE`, with wallet role status `PENDING` and wallet `REQUESTED`. In the same transaction: one entry `ADDED` per assigned role (`actor_type` `PROCESS`), event `TechnicalUserCreated`, with wallet role also `WalletAccessRequested`.
4. If there are provider-only roles: **provider account** with these roles as in step 3, key `offerSubscriptionId` + `PROVIDER`, `company_id` = provider, no co-viewing company, attributes of the provider, never with wallet access.
5. Result: `offerSubscriptionId` and per account `technicalUserId`, `subscriptionParty`, `iamClientId`, `status`, `walletAccess`, `roles`. **No credentials.** Subscriptions creates the secret when it passes it on to the provider, through 9.2.

Process Orchestration starts the wallet access of the customer account on `WalletAccessRequested` (FRAMEWORK 5.7). A retry finds both accounts through their keys, completes what is missing and returns the same result.

### 8.2 `apply-wallet-access`

Input: `technicalUserId`, `outcome` (`ACTIVE` or `FAILED`), for `ACTIVE` also `walletAccountRef`.

Preconditions: technical user with wallet `REQUESTED`; C4-M2 has reported `WalletAccountCreated` or `WalletAccountFailed`.

- Unknown: 404 `not-found`. `outcome` `ACTIVE` without `walletAccountRef`: 400 `validation`.
- Wallet `REQUESTED`, `outcome` `ACTIVE`: set `wallet_account_ref`, wallet `ACTIVE`, status `ACTIVE`.
- Wallet `REQUESTED`, `outcome` `FAILED`: wallet `FAILED`, status `ACTIVE`. The account remains usable in the Portal IAM.
- Wallet already `ACTIVE` or `FAILED` with the same `outcome` (and the same `walletAccountRef`): nothing to do.
- Otherwise: 409 `invalid-state`.

A disabled account stays disabled. No event of its own: people learn the result from `WalletAccountCreated` or `WalletAccountFailed` of C4-M2 and retrieve the credentials there (7.7). Result: `technicalUserId`, `status`, `walletAccess`, `walletAccountRef`.

### 8.3 `delete`

Input: `technicalUserId`.

Preconditions: technical user with status `PENDING_DELETION` and wallet `REMOVING`; C4-M2 has reported `WalletAccountDeleted`.

- Unknown: 404 `not-found`. Status `DELETED`: nothing to do. Other status: 409 `invalid-state`.
- Read the roles of the service account (for the audit log), delete the client in the Portal IAM; if it is already gone, this counts as success.
- Status `DELETED`, wallet `NONE`, clear `wallet_account_ref`. In the same transaction: one entry `REMOVED` per role (`actor_type` `PROCESS`), event `TechnicalUserDeleted`.

Result: `technicalUserId`, `status`.

### 8.4 `disable-company-technical-users`

Input: `companyId`.

Preconditions: company deactivated (`CompanyDeactivated`, FRAMEWORK 5.4).

1. Read all accounts with `company_id` = `companyId`, a status other than `DELETED` and `disabled` = `false`.
2. Per account, each in its own transaction: with `iam_client_uuid`, read the client in the Portal IAM and set `enabled` = `false` (`PUT`); if the client is missing, only set the flag. `disabled` = `true`, entry in `audit_log` (`actor_type` `PROCESS`).
3. Result: `companyId`, `technicalUserIds` (all accounts of the company with a status other than `DELETED` and `disabled` = `true`).

No event; FRAMEWORK 6 does not provide one. Accounts without `iam_client_uuid` (interrupted creation) only get the flag; `create-for-subscription` does not complete their creation as long as the company is not `ACTIVE`. A retry finds no open accounts any more and returns the same result.

### 8.5 `enable-company-technical-users`

Input: `companyId`.

Preconditions: company reactivated (`CompanyReactivated`, FRAMEWORK 5.5).

1. Read all accounts with `company_id` = `companyId`, a status other than `DELETED` and `disabled` = `true`.
2. Per account, each in its own transaction: with `iam_client_uuid`, enable the client (`enabled` = `true`); `disabled` = `false`, entry in `audit_log`.
3. Result: `companyId`, `technicalUserIds` (all accounts of the company with a status other than `DELETED` and `disabled` = `false`).

No event. The programs get tokens again with the same secret, with the same roles.

## 9. Queries for other modules

### 9.1 `GET /internal/technical-users/{id}?companyId=`

For Connectors. The response is always 200, also for an unknown ID (then `exists: false`). `usableForConnector` is `true` if the account exists, `company_id` equals `companyId`, the status is `ACTIVE` or `PENDING`, the origin is `OWN` and `disabled` is `false`. Connectors checks itself whether another connector already uses the account.

### 9.2 `POST /internal/technical-users/{id}/credentials/rotate`

For Subscriptions, when passing the credentials of a subscription account on to the provider. Input: `offerSubscriptionId`.

- The account must have the origin `SUBSCRIPTION` and this `offerSubscriptionId`, otherwise 404 `not-found`.
- Status `ACTIVE` or `PENDING` with a created client and not disabled, otherwise 409 `invalid-state`.
- Create a new secret in the Portal IAM; an earlier one is no longer valid afterwards. Response 200 with `credentials`.

The secret appears only in this response. The module writes an entry to `audit_log` (`actor_type` `PROCESS`), without the secret. If the hand-over is repeated, a new secret is created.

## 10. Events

Written to the outbox, in the same transaction as the change in the module's table. No event contains a secret.

| Event | When | Content |
|---|---|---|
| `TechnicalUserCreated` | an account is fully created in the Portal IAM (7.2, 8.1) | `technicalUserId`, `companyId`, `origin`, `offerSubscriptionId`, `subscriptionParty`, `coViewerCompanyId`, `iamClientId` |
| `TechnicalUserDeleted` | status `DELETED` (7.5, 8.3) | `technicalUserId`, `companyId`, `coViewerCompanyId`, `iamClientId` |
| `WalletAccessRequested` | account with wallet role created (7.2, 8.1) | `technicalUserId`, `companyId`, `offerSubscriptionId` |
| `WalletAccessRemovalRequested` | account with wallet `ACTIVE` is to be deleted (7.5) | `technicalUserId`, `companyId`, `walletAccountRef` |

Every event also carries `eventId` (uuid), `eventType` (name from the table) and `occurredAt`. The schemas are in [openapi.yaml](openapi.yaml) under `components/schemas` (names start with `Event`).

Notifications (other part) can show the events to the people of the affected company, for subscription accounts also to the co-viewing provider. The events `WalletAccountCreated` and `WalletAccountFailed` of C4-M2 report that a wallet access is ready or has failed.

## 11. Calls to other modules and external systems

### 11.1 Portal IAM (Keycloak Admin REST API)

All calls with the token of the module's own service account (`IAM_ADMIN_CLIENT_ID`), realm `IAM_REALM`. Paths start at `/admin/realms/{realm}`.

| Purpose | Call |
|---|---|
| Search client | `GET …/clients?clientId={clientId}` |
| Create client | `POST …/clients` with the settings from 6.6, including `defaultClientScopes`; the internal ID is in the `Location` header |
| Read / change client (name, description, enable, disable) | `GET` / `PUT …/clients/{uuid}`; for `PUT`, send the representation read before with the changed fields |
| Service account user | `GET …/clients/{uuid}/service-account-user` |
| Set attributes | `GET` / `PUT …/users/{userId}`; for `PUT`, the representation read before with `{COMPANY_ID_ATTRIBUTE}: [companyId]`, `{BPN_ATTRIBUTE}: [BPN]` |
| Find the roles client | `GET …/clients?clientId={TECHNICAL_ROLES_CLIENT}` |
| Read roles | `GET …/clients/{rolesClientUuid}/roles?briefRepresentation=false` |
| Assign / remove roles | `POST` / `DELETE …/users/{userId}/role-mappings/clients/{rolesClientUuid}` with a list of roles with `id` and `name` |
| Roles of an account | `GET …/users/{userId}/role-mappings/clients/{rolesClientUuid}` |
| Read / regenerate secret | `GET` / `POST …/clients/{uuid}/client-secret` |
| Delete client | `DELETE …/clients/{uuid}`; in 8.3, 404 counts as success |

Token address for responses and results: `{IAM_URL}/realms/{IAM_REALM}/protocol/openid-connect/token`.

### 11.2 Neighbouring modules

In the shared process, the module calls the public interface of the neighbour; the HTTP form applies when the neighbour runs as a separate service. If the connection is empty, the module uses the built-in stand-in. This way it can be run and tested alone. If a neighbour does not respond as expected: 502 `upstream-error`.

| Neighbour (variable) | Call | Response | Built-in stand-in |
|---|---|---|---|
| C3-M1 Company Profile (`COMPANY_URL`) | `GET /api/c3-m1/internal/companies/{companyId}` | `{ "id", "status", "bpn", "companyRoles": [] }` (active roles only), status `REQUESTED`, `ACTIVE`, `REJECTED` or `INACTIVE`; 404 if unknown | reads `stub-companies.json` on every call |
| Connectors (`CONNECTOR_URL`) | `GET /internal/connectors?technicalUserId={id}` (base as in the spec of Connectors) | list of `{ "connectorId", "status" }` | empty list |
| Subscriptions (`SUBSCRIPTION_URL`) | `GET /internal/subscriptions/{offerSubscriptionId}` (base as in the spec of Subscriptions) | `{ "offerSubscriptionId", "status" }` | status `INACTIVE` |

A connector "uses" the account if it is in the list with status `ACTIVE` or `PENDING`.

### 11.3 Stand-in for Process Orchestration

With `PROCESS_STUB=true`, a built-in stand-in reads the module's events from the outbox and replays the flows from FRAMEWORK 5.7 without C4-M2:

- on `WalletAccessRequested`: step `apply-wallet-access` with `outcome` `ACTIVE` and `walletAccountRef` `stub-{technicalUserId}`;
- on `WalletAccessRemovalRequested`: step `delete`.

It calls the steps through the same public interface as Process Orchestration. It can deliver every event twice to check idempotency. The test calls steps 8.4 and 8.5 directly.

## 12. Errors

Errors come as Problem Details (RFC 9457). `type` is `urn:c2-m3:problem:<name>`. For steps, the error carries `retryable` (FRAMEWORK 4.4).

| Name | HTTP | When | `retryable` for steps |
|---|---|---|---|
| `validation` | 400 | invalid input | `false` |
| `role-not-assignable` | 400 | role not assignable or unknown | `false` |
| `unauthorized` | 401 | token missing or invalid (HTTP only) | – |
| `forbidden` | 403 | permission missing (HTTP only) | – |
| `not-owner` | 403 | co-viewing company tries to change the account or rotate the secret | – |
| `not-found` | 404 | account does not exist or is not visible, or the path names another company | `false` |
| `company-not-ready` | 409 | company not `ACTIVE` or without BPN | `false` |
| `invalid-state` | 409 | status or deactivation does not allow the operation or the step | `false` |
| `wallet-in-progress` | 409 | wallet access is still being created | – |
| `wallet-role-change` | 409 | adding or removing a wallet role through a change | – |
| `in-use` | 409 | a connector uses the account, or the subscription is active | – |
| `iam-error` | 502 | Portal IAM did not respond as expected | `true` |
| `upstream-error` | 502 | a neighbouring module did not respond as expected | `true` |

## 13. Acceptance

The module is done when these cases work against the test Portal IAM and with built-in stand-ins for the neighbouring modules and Process Orchestration. People call with test tokens as in 14.2.

Technical users of the company:

1. Max Mustermann (IT Admin of Beispiel Maschinenbau) creates an account with `Dataspace Discovery`: 201 with secret. With client ID and secret, one gets a token from the Portal IAM; it contains the role, the contained roles of the central services, `company_id` and `bpn` of the company, but not `portal-core` in `aud`. Event `TechnicalUserCreated`; one entry each in `audit_log` and `role_assignment_log` with Max's `actor_sub`, the account and the role.
2. The details show roles but no secret. The list shows the account without roles, with `viewerRole` `OWNER`.
3. A role that the company may not assign: 400 `role-not-assignable`.
4. Company without BPN (Ohne Zustimmung GmbH, `REQUESTED`): 409 `company-not-ready`, and no client is left behind in the Portal IAM.
5. Max calls `/companies/{Beispiel Logistik}/technical-users`: 404. An account of Beispiel Logistik under his own path: 404. A token without `view_technical_users`: 403. A token with the permission but without `company_id`: 404.
6. Create with `Identity Wallet Management`: first `PENDING` with wallet `REQUESTED`, event `WalletAccessRequested` without secret. After the stand-in for Process Orchestration, `ACTIVE` with wallet `ACTIVE` and `walletAccountRef`. Event delivered twice: `apply-wallet-access` changes nothing the second time.
7. Rotate secret: the old secret no longer works, the new one does. The co-viewing company gets 403 `not-owner`.
8. Change roles: the next token contains the new roles; `role_assignment_log` records `ADDED` and `REMOVED`. Wallet role through a change: 409.
9. Delete without wallet access: 204, the client is gone from the Portal IAM, the account is `DELETED`, event `TechnicalUserDeleted`, one entry `REMOVED` per role. With wallet access: 202, event `WalletAccessRemovalRequested`, `DELETED` after the stand-in.
10. Delete while wallet `REQUESTED`: 409 `wallet-in-progress`.
11. `create-for-subscription` (customer Beispiel Maschinenbau, provider Beispiel Logistik) with a provider-only role creates two accounts; the result contains no credentials. Hans Beispiel (Company Admin of Beispiel Logistik) sees the customer account with `viewerRole` `CO_VIEWER`. A retry, also with a new `stepRunId`, creates no further accounts, returns the same result and no second event. An interrupted creation (client created in the Portal IAM, `iam_client_uuid` not stored, simulated) is completed on retry through the same client ID.
12. `POST /internal/technical-users/{id}/credentials/rotate` for the customer account from case 11 returns a secret with which one gets a token; with a wrong `offerSubscriptionId`: 404.
13. `/internal/technical-users/{id}` reports `usableForConnector` only for own, active, not disabled accounts with origin `OWN`.

OSP staff:

14. A token with `view_all_technical_users` (OSP staff) reads `/osp/technical-users` and sees accounts of all companies with `viewerRole` `OSP`; the filter `companyId` works; the details show roles without secret. The same token on `/companies/{Beispiel Maschinenbau}/technical-users`: 403.

Deactivation and reactivation:

15. `disable-company-technical-users` for Beispiel Maschinenbau: all own accounts have `disabled` `true`, their clients are disabled in the Portal IAM, client credentials with their secret no longer return a token. The provider account of Beispiel Logistik from case 11 stays active. Retry: the same result. `PATCH` and rotate secret: 409 `invalid-state`; `usableForConnector` is `false`.
16. If Beispiel Maschinenbau is `INACTIVE` in the stand-in for C3-M1, `POST …/technical-users` returns 409 `company-not-ready`, and so does `create-for-subscription` as customer.
17. `enable-company-technical-users`: clients enabled, `disabled` `false`; the same secret returns a token again, with the same roles.
18. Abort after the first account (simulated): the retry of 8.4 or 8.5 completes the rest.

General:

19. No secret appears in the log, in an event, in a step result or in the module's database. No audit log entry contains a name.

## 14. Test data

### 14.1 Test Portal IAM

The test Portal IAM needs:

- the client `technical_roles_management` with the roles from 6.5, descriptions as attributes `description.de` and `description.en`, composed of roles on the test clients of the central services (at least `Dataspace Discovery` with roles on `Cl21-CX-DF` and `Cl22-CX-BPND`);
- the client scopes `basic`, `roles`, `catena` (mapper of the attributes `company_id` and `bpn` to claims of the same name) and `technical-roles` (scope mapping of all roles of `technical_roles_management`);
- a user profile that allows the attributes `company_id` and `bpn` on service account users;
- a service account for the module with the rights listed for `IAM_ADMIN_CLIENT_ID`.

### 14.2 Tokens of the test persons

For calls over HTTP, the test signs tokens itself with a test key; in the test, the token validation library gets its JWKS. Claims as in FRAMEWORK 4.2. IDs as in the test data of C3-M1 and C2-M2.

| Person | `sub` | `company_id` | Permissions of this module |
|---|---|---|---|
| Max Mustermann, IT Admin of Beispiel Maschinenbau | `5a1c0002-0000-4000-8000-000000000002` | U1 | `view_technical_users`, `manage_technical_users`, `delete_technical_users` |
| Erika Mustermann, Company Admin of Beispiel Maschinenbau | `5a1c0001-0000-4000-8000-000000000001` | U1 | same as Max |
| Hans Beispiel, Company Admin of Beispiel Logistik | `5a1c0003-0000-4000-8000-000000000003` | U2 | same as Max |
| Lieschen Müller, without company | `5a1c0004-0000-4000-8000-000000000004` | – | none (case 5 with an additional `view_technical_users`) |
| Otto Normalverbraucher, OSP Admin | `5a1c0012-0000-4000-8000-000000000012` | – | `view_all_technical_users` |

### 14.3 Stand-ins for the neighbours

`stub-companies.json` (stand-in for C3-M1; the companies are fictional):

| Company | `companyId` | Status | BPN | Company roles |
|---|---|---|---|---|
| Company of OSP A | `11111111-1111-4111-8111-111111111111` | `ACTIVE` | `BPNL00000000OSPA` | `ONBOARDING_SERVICE_PROVIDER`, `ACTIVE_PARTICIPANT` |
| U1 Beispiel Maschinenbau GmbH | `c0000001-0000-4000-8000-000000000001` | `ACTIVE` (case 16: `INACTIVE`) | `BPNLIUK2REMLK3F4` | `ACTIVE_PARTICIPANT` |
| U2 Beispiel Logistik GmbH | `c0000002-0000-4000-8000-000000000002` | `ACTIVE` | `BPNL00000000U2LG` | `ACTIVE_PARTICIPANT`, `APP_PROVIDER` |
| U3 Ohne Zustimmung GmbH | `c0000003-0000-4000-8000-000000000003` | `REQUESTED` | – | – |

For the subscription accounts from case 11: subscription `0b5d7e2a-3c4f-4a1b-9e8d-7f6a5b4c3d21`, profile with `Dataspace Discovery`, `Identity Wallet Management` and the provider-only role `BPDM Sharing Output Consumer`.

## 15. Changes from version 0.1

- **Callers:** people of a company (IT Admin, Company Admin) with their own token instead of PSPs with delegation. `X-On-Behalf-Of`, `X-Acting-Person`, `x-delegation-scope`, the delegation scope `technical-users` and the error `not-delegated` are dropped. Paths carry the company (`/companies/{companyId}/technical-users…`); tags `Company`, `OSP`, `Queries`, `Steps`; `x-permission-scope` per operation.
- **New for OSP staff:** `GET /osp/technical-users`, `GET /osp/technical-users/{id}` with the permission `view_all_technical_users` (scope `osp`); `viewerRole` `OSP`.
- **API access of PSPs dropped:** table `psp_access`, steps `create-psp-access`, `disable-psp-access`, `enable-psp-access` (and the planned `delete-psp-access`), query `/internal/psp-accesses/{pspId}`, event `PspAccessCreated`, mappers for `psp_id` and audience, configuration `CORE_CLIENT_ID`, `PSP_*`, `OPERATOR_ROLE_BUNDLE`, neighbours C1-M1 and C2-M4, error `psp-not-ready`.
- **New steps** `disable-company-technical-users`, `enable-company-technical-users` with the flag `disabled` (6.2); no new technical users for companies that are not `ACTIVE`.
- **Token content:** attributes `company_id` and `bpn` on the service account, client scopes `catena` and `technical-roles` (as in C2-M5), `fullScopeAllowed` `false`; the separate BPN mapper per client is dropped.
- **Assignable roles:** without the roles that only contain rights on today's portal (OP-24); `PORTAL_SERVICE_PROVIDER` and `OPERATOR` are dropped, `ONBOARDING_SERVICE_PROVIDER` gives no further roles.
- **Tokens for central services at the CSP:** note on OP-37 (6.7).
- **Audit log:** `actor_sub` from the token instead of PSP and `X-Acting-Person`; steps also write to `audit_log`.
- Errors carry `retryable` for steps. Section 8 renumbered.
