# C2-M3 Technical Users

**Version:** 0.1, as of 2026-10-19
**Capability:** C2 Identity and Access
**Kind:** Hackathon (to be built at the hackathon)
**Files:** this README (behaviour), [openapi.yaml](openapi.yaml) (interface)

This README and the OpenAPI description are sufficient to build the module. Modules from other parts of the spec are referred to by name.

## 1. Purpose

The module manages technical users: accounts that programs use to sign in to the Portal IAM. It creates, shows, changes, rotates the secret of and deletes them. If an account needs access to the company's dataspace wallet, the module triggers this via Process Orchestration and records the result.

Callers:

| Who | What for |
|---|---|
| Company admin (via the UI) | manage the company's own accounts |
| Process Orchestration | create accounts when an app or service subscription is activated; apply wallet access; delete accounts |
| Connector | check whether an account fits when a connector is registered |

## 2. Terms

| Term | Meaning |
|---|---|
| Technical user, account | an account for a program; belongs to exactly one company |
| Account in the Portal IAM | a client with service account in the realm of the Portal IAM (Keycloak); sign-in via OAuth2 client credentials with client ID and secret |
| Role | role on the client `technical_roles_management` in the Portal IAM, assigned to the service account |
| Wallet role | role that triggers wallet access (configuration `WALLET_ROLES`). It is **not** assigned in the Portal IAM. |
| Provider-only role | role that only the provider receives in a subscription (configuration `PROVIDER_ONLY_ROLES`) |
| Wallet access | an account in the company's dataspace wallet; created by the Dataspace Wallet Interface (C4-M2) |
| Origin | `OWN`: created by the admin. `SUBSCRIPTION`: created when a subscription is activated |
| Co-viewing company | company that may see and delete another company's account: for customer accounts from a subscription, the provider |

## 3. Not part of this module

| What | Where |
|---|---|
| Accounts for humans | C2-M2 Users |
| Creating the realm, the client `technical_roles_management` and its roles | C2-M5 Portal IAM (configuration) |
| Creating and deleting the account in the dataspace wallet | C4-M2 Dataspace Wallet Interface |
| Company data, BPN, company roles | C3-M1 Company Profile |
| Storing the link between a connector and an account | Connector |
| Subscription flow, sending credentials to the provider | Subscriptions and Process Orchestration |
| Sending mails and notifications | Notifications |

## 4. Technical frame

- Python with FastAPI.
- Own database schema `c2_m3` in PostgreSQL. The module reads and writes only there. Other modules read through the interface.
- Every call carries a token of the Portal IAM. The shared token validation library checks signature, expiry and audience and provides user ID, company ID and BPN of the caller as well as their permissions. In the test Portal IAM the company ID is in the claim `company_id`.
- Secrets never appear in logs.

### Configuration

| Variable | Meaning | Example |
|---|---|---|
| `DATABASE_URL` | PostgreSQL, schema `c2_m3` | |
| `IAM_URL` | base URL of the Portal IAM | `https://iam.example.org/auth` |
| `IAM_REALM` | realm | `CX-Central` |
| `IAM_ADMIN_CLIENT_ID`, `IAM_ADMIN_CLIENT_SECRET` | the module's own service account for the admin API; needs the realm permissions `manage-clients`, `view-clients`, `query-clients`, `manage-users`, `view-users` | |
| `TECHNICAL_ROLES_CLIENT` | client holding the roles for technical users | `technical_roles_management` |
| `CLIENT_ID_PREFIX` | prefix of the client ID | `sa` |
| `BPN_ATTRIBUTE` | name of the attribute and claim for the BPN | `bpn` |
| `WALLET_ROLES` | wallet roles, comma-separated | `Identity Wallet Management` |
| `PROVIDER_ONLY_ROLES` | provider-only roles | `BPDM Pool Consumer,BPDM Sharing Output Consumer,BPDM Sharing Input Manager` |
| `ROLE_ASSIGNMENT_FILE` | JSON file: company role → assignable roles (section 6.3) | `role-assignment.json` |
| `WALLET_CREDENTIALS_KEY` | key (AES-256) for encrypting wallet credentials until they are retrieved | |
| `COMPANY_URL`, `PROCESS_URL`, `CONNECTOR_URL`, `SUBSCRIPTION_URL`, `NOTIFICATION_URL` | addresses of the neighbouring modules; empty = built-in stand-in (section 9) | |

## 5. Data

### Table `technical_user`

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key |
| `company_id` | uuid | owner; required |
| `name` | text | 1 to 80 characters; required; need not be unique |
| `description` | text | up to 255 characters |
| `origin` | text | `OWN` or `SUBSCRIPTION` |
| `offer_subscription_id` | uuid | required for `SUBSCRIPTION`, otherwise empty |
| `co_viewer_company_id` | uuid | co-viewing company or empty |
| `iam_client_id` | text | unique; `CLIENT_ID_PREFIX` + value of the sequence `client_number`, e.g. `sa123` |
| `iam_client_uuid` | text | internal ID of the client in Keycloak |
| `iam_service_account_user_id` | text | ID of the service account user in Keycloak |
| `status` | text | `PENDING`, `ACTIVE`, `PENDING_DELETION`, `DELETED` |
| `wallet_access` | text | `NONE`, `REQUESTED`, `ACTIVE`, `FAILED`, `REMOVING`; default `NONE` |
| `wallet_access_ref` | text | identifier of the wallet account at C4-M2 |
| `created_at`, `changed_at` | timestamptz | |
| `last_editor_id` | uuid | user or service account that changed it last |

Deleted accounts are kept with status `DELETED`.

### Table `wallet_credentials`

Credentials of a wallet access, encrypted with `WALLET_CREDENTIALS_KEY`, until the admin retrieves them once. Then the row is deleted.

| Field | Type |
|---|---|
| `technical_user_id` | uuid, primary key |
| `auth_service_url` | text |
| `client_id` | text |
| `client_secret_encrypted`, `iv` | bytea |
| `created_at` | timestamptz |

### Table `step_run`

Remembers steps of Process Orchestration so that a repetition does not create anything twice.

| Field | Type |
|---|---|
| `step_run_id` | uuid, primary key |
| `process_id` | uuid |
| `step` | text: `create-for-subscription`, `apply-wallet-access`, `delete` |
| `state` | text: `RUNNING`, `DONE`, `FAILED` |
| `result` | jsonb, **without** secrets |
| `created_at`, `changed_at` | timestamptz |

**Not stored:** roles (only in the Portal IAM), BPN (at C3-M1), secret of the account in the Portal IAM (only in Keycloak), link to a connector (at the Connector).

## 6. Rules

### 6.1 Status

| From | Event | To |
|---|---|---|
| – | create without wallet role | `ACTIVE`, wallet `NONE` |
| – | create with wallet role | `PENDING`, wallet `REQUESTED` |
| `PENDING` / `REQUESTED` | wallet access applied, outcome `ACTIVE` | `ACTIVE`, wallet `ACTIVE` |
| `PENDING` / `REQUESTED` | wallet access applied, outcome `FAILED` | `ACTIVE`, wallet `FAILED` |
| `ACTIVE`, wallet `NONE` or `FAILED` | delete | `DELETED` |
| `ACTIVE`, wallet `ACTIVE` | delete | `PENDING_DELETION`, wallet `REMOVING` |
| `PENDING_DELETION` | step "delete account" | `DELETED`, wallet `NONE` |

### 6.2 Visibility and permissions

A company sees the accounts whose `company_id` or `co_viewer_company_id` is its company ID. Otherwise the module answers 404.

| Operation | Permission |
|---|---|
| list, details | `view_tech_user_management` |
| create, change, rotate secret, retrieve wallet credentials | `add_tech_user_management` |
| delete | `delete_tech_user_management` |
| assignable roles | `technical_roles_management` |
| `GET /internal/technical-users/{id}` | `view_tech_user_internal` |
| all paths under `/steps` | `execute_tech_user_steps` |

### 6.3 Assignable roles

Which roles a company may assign follows from its company roles (from C3-M1) and the file `ROLE_ASSIGNMENT_FILE`. The assignable roles are the union over all company roles. Initial values, as in today's portal:

| Company role | Assignable roles |
|---|---|
| `ACTIVE_PARTICIPANT` | Connector Management, CX Membership Info, Dataspace Discovery, Identity Wallet Management, Semantic Model Management |
| `APP_PROVIDER` | BPDM Sharing Input Consumer, BPDM Sharing Output Consumer, CX Membership Info, Dataspace Discovery, Identity Wallet Management, Offer Management, Semantic Model Management |
| `SERVICE_PROVIDER` | same as `APP_PROVIDER` |
| `ONBOARDING_SERVICE_PROVIDER` | Registration External |
| `OPERATOR` | all BPDM roles, CX Membership Info, Dataspace Discovery, Identity Wallet Management, Registration External, Registration Internal, Semantic Model Management |

This check does not apply to subscription accounts. There, the roles from the offer's profile apply.

## 7. Behaviour of the operations

### 7.1 `GET /technical-users`

- Returns visible accounts, sorted by `created_at` descending, paged (`page` from 0, `size` 1 to 100, default 20).
- Without filter `status`: `PENDING`, `ACTIVE`, `PENDING_DELETION`.
- Entries without roles (`TechnicalUserSummary`), so the list needs no calls to the Portal IAM.
- `viewerRole` is `OWNER` if `company_id` is the caller's company, otherwise `CO_VIEWER`.

### 7.2 `POST /technical-users`

1. Validate, otherwise 400: `name` 1 to 80 characters; `description` up to 255; `roles` not empty, no duplicate names; `authMethod` missing or `SECRET`.
2. Read the company from C3-M1. Not active or without BPN: 409 `company-not-ready`.
3. Every role must be assignable according to 6.3 and exist on the client `TECHNICAL_ROLES_CLIENT`, otherwise 400 `role-not-assignable`.
4. Split the roles: wallet roles (`WALLET_ROLES`) and the remaining roles. Provider-only roles are not treated separately for `OWN`.
5. Get the next number from `client_number`, build the client ID.
6. In the Portal IAM, in this order (section 8):
   1. Create the client.
   2. Read the service account user.
   3. Set the BPN as attribute and create the BPN mapper on the client.
   4. Assign the remaining roles.
   5. Read the secret.

   If a step fails: delete the client again, 502 `iam-error`.
7. Write the row in `technical_user`: `origin` `OWN`; without wallet role `ACTIVE`, with wallet role `PENDING` and wallet `REQUESTED`. If this fails: delete the client, 500.
8. With wallet role: start a process at Process Orchestration (section 9), action `CREATE`.
9. Response 201 with the account and `credentials` (client ID, secret, token URL). The secret appears only in this response.

### 7.3 `GET /technical-users/{id}`

Not visible: 404. Read the roles from the Portal IAM (roles of the service account user on the client `TECHNICAL_ROLES_CLIENT`). If wallet access is not `NONE`, the wallet role is additionally listed in `roles`. No secret.

### 7.4 `PATCH /technical-users/{id}`

- Only with status `ACTIVE`, otherwise 409 `invalid-state`.
- `name`, `description`: change in the account and on the client in the Portal IAM.
- `roles`: new complete list. Check as in 7.2 step 3; remove roles no longer listed, assign new ones. Adding or removing a wallet role is not allowed: 409 `wallet-role-change`.
- Response 200 with the account.

### 7.5 `DELETE /technical-users/{id}`

1. Not visible: 404. Status not `ACTIVE`: 409 `invalid-state`. Wallet `REQUESTED`: 409 `wallet-in-progress`.
2. Ask the Connector whether a connector uses the account. Yes: 409 `in-use`.
3. For `SUBSCRIPTION`: ask Subscriptions whether the subscription is active. Yes: 409 `in-use`.
4. Wallet `NONE` or `FAILED`: delete the client in the Portal IAM, status `DELETED`, response 204.
5. Wallet `ACTIVE`: status `PENDING_DELETION`, wallet `REMOVING`, start a process at Process Orchestration (action `DELETE`), response 202. The step "delete account" deletes the client later.

### 7.6 `POST /technical-users/{id}/credentials/rotate`

Only with status `ACTIVE` or `PENDING`, otherwise 409. Generate a new secret in the Portal IAM, response 200 with `credentials`. Wallet access is not affected.

### 7.7 `POST /technical-users/{id}/wallet-access/credentials`

Returns the credentials of the wallet access exactly once and then deletes the row in `wallet_credentials`. Wallet not `ACTIVE`: 409 `invalid-state`. Already retrieved: 410 `already-retrieved`. Whoever lost them deletes the account and creates it again.

### 7.8 `GET /technical-users/assignable-roles`

Roles according to 6.3 for the caller's company. Name and description from the Portal IAM: role attribute `description.{lang}`, otherwise `description`. Flags `grantsWalletAccess` (in `WALLET_ROLES`) and `providerOnly` (in `PROVIDER_ONLY_ROLES`).

### 7.9 `GET /internal/technical-users/{id}?companyId=`

For the Connector. Always answers 200 (unless the permission is missing). `usableForConnector` is `true` if the account exists, `company_id` equals `companyId`, the status is `ACTIVE` or `PENDING` and the origin is `OWN`. Whether another connector already uses it is checked by the Connector itself.

### 7.10 Steps for Process Orchestration

Common rules:
- Every call carries `processId` and `stepRunId`.
- If the `stepRunId` already exists, the module returns the stored result and creates nothing new.
- The response is a `StepResult` with `state` `DONE`, `RUNNING` or `FAILED`.
- With `RUNNING`, Process Orchestration polls `GET /steps/{stepRunId}` later.

**`POST /steps/create-for-subscription`**
1. Split the roles into wallet roles, provider-only roles and remaining roles.
2. Create the **customer account** with the remaining roles as in 7.2 steps 5 to 7:
   - `company_id` is the customer, `co_viewer_company_id` the provider, `origin` `SUBSCRIPTION`
   - name `{client ID}-{name}`
   - the customer's BPN from C3-M1
3. If there are provider-only roles: create a **provider account** with these roles, `company_id` is the provider.
4. If there is a wallet role: wallet `REQUESTED` and status `PENDING` on the customer account. Process Orchestration triggers the wallet access itself as the next step of the subscription flow.
5. Result `DONE` with all created accounts and their `credentials`. `step_run.result` holds only the account IDs.
6. Repetition with the same `stepRunId`: no new accounts. A new secret is generated and returned for each account, so the credentials cannot get lost.

**`POST /steps/apply-wallet-access`**
- `outcome` `ACTIVE`: store `wallet_credentials` encrypted, set `wallet_access_ref`, wallet `ACTIVE`, status `ACTIVE`. Notification `C2-M3/wallet-access-ready` to the owner's admins.
- `outcome` `FAILED`: wallet `FAILED`, status `ACTIVE`. Notification `C2-M3/wallet-access-failed`.
- Result `DONE`.

**`POST /steps/delete`**
- Delete the client in the Portal IAM; if it is already gone, that counts as success.
- Status `DELETED`, wallet `NONE`.
- Result `DONE`.

**`GET /steps/{stepRunId}`**
Stored result, without secrets.

## 8. Calls to the Portal IAM (Keycloak Admin REST API)

All calls with the token of the module's own service account (`IAM_ADMIN_CLIENT_ID`), realm `IAM_REALM`.

| Purpose | Call |
|---|---|
| Create client | `POST /admin/realms/{realm}/clients` with `clientId`, `name`, `description`, `enabled: true`, `protocol: openid-connect`, `publicClient: false`, `serviceAccountsEnabled: true`, `standardFlowEnabled: false`, `directAccessGrantsEnabled: false`, `clientAuthenticatorType: client-secret`; the internal ID is in the `Location` header |
| Service account user | `GET …/clients/{uuid}/service-account-user` |
| Set BPN | `PUT …/users/{userId}` with attribute `{BPN_ATTRIBUTE}: [BPN]` |
| BPN mapper | `POST …/clients/{uuid}/protocol-mappers/models`, type `oidc-usermodel-attribute-mapper`, attribute and claim `{BPN_ATTRIBUTE}`, in access, ID and userinfo token |
| Find the roles client | `GET …/clients?clientId={TECHNICAL_ROLES_CLIENT}` |
| Read roles | `GET …/clients/{rolesClientUuid}/roles?briefRepresentation=false` |
| Assign / remove roles | `POST` / `DELETE …/users/{userId}/role-mappings/clients/{rolesClientUuid}` |
| Roles of an account | `GET …/users/{userId}/role-mappings/clients/{rolesClientUuid}` |
| Read / regenerate secret | `GET` / `POST …/clients/{uuid}/client-secret` |
| Change name, description | `PUT …/clients/{uuid}` |
| Delete client | `DELETE …/clients/{uuid}` |

Token URL for the response: `{IAM_URL}/realms/{IAM_REALM}/protocol/openid-connect/token`.

## 9. Calls to neighbouring modules

The module expects the following calls. If the address of a neighbouring module is empty, it uses the built-in stand-in. This way it can be run and tested on its own.

| Neighbour | Call | Response | Built-in stand-in |
|---|---|---|---|
| C3-M1 Company Profile (`COMPANY_URL`) | `GET /companies/{companyId}` | `{ "id", "status", "bpn", "companyRoles": [] }` | reads the file `stub-companies.json` |
| Process Orchestration (`PROCESS_URL`) | `POST /processes` with `{ "type": "technical-user-wallet-access", "action": "CREATE" \| "DELETE", "technicalUserId", "companyId" }` | 202 `{ "processId" }` | `CREATE`: immediately calls "apply wallet access" internally with `ACTIVE` and made-up credentials. `DELETE`: immediately calls "delete account". |
| Connector (`CONNECTOR_URL`) | `GET /connectors?technicalUserId={id}` | list of connectors | empty list |
| Subscriptions (`SUBSCRIPTION_URL`) | `GET /subscriptions/{offerSubscriptionId}` | `{ "status" }` | status `INACTIVE` |
| Notifications (`NOTIFICATION_URL`) | `POST /notifications` with `{ "template", "recipients": [user IDs], "values", "channel" }` | 202 | writes the notification to the log, without secrets |

Recipients of notifications are the owner's users with the permission `add_tech_user_management`. C2-M2 provides this list. In stand-in mode, the log is enough.

## 10. Errors

Errors come as Problem Details (RFC 9457). `type` is one of these identifiers:

| `type` | HTTP | When |
|---|---|---|
| `urn:c2-m3:problem:validation` | 400 | invalid input |
| `urn:c2-m3:problem:role-not-assignable` | 400 | role not assignable or unknown |
| `urn:c2-m3:problem:forbidden` | 403 | permission missing |
| `urn:c2-m3:problem:not-found` | 404 | account does not exist or is not visible |
| `urn:c2-m3:problem:company-not-ready` | 409 | company not active or without BPN |
| `urn:c2-m3:problem:invalid-state` | 409 | status does not allow the operation |
| `urn:c2-m3:problem:wallet-in-progress` | 409 | wallet access is still being created |
| `urn:c2-m3:problem:wallet-role-change` | 409 | adding or removing a wallet role via change |
| `urn:c2-m3:problem:in-use` | 409 | a connector uses the account, or the subscription is active |
| `urn:c2-m3:problem:already-retrieved` | 410 | wallet credentials already retrieved |
| `urn:c2-m3:problem:iam-error` | 502 | the Portal IAM did not answer as expected |

## 11. Acceptance

The module is done when these cases work against the test Portal IAM, with built-in stand-ins for the neighbouring modules:

1. Admin creates an account with `Connector Management`: 201 with secret. Using client ID and secret, a token can be obtained from the Portal IAM that contains the role and the claim `bpn`.
2. The details show roles but no secret. The list shows the account without roles.
3. A role the company may not assign: 400 `role-not-assignable`.
4. Company without BPN: 409 `company-not-ready`, and no client is left behind in the Portal IAM.
5. Create with `Identity Wallet Management`: first `PENDING`, after the Process Orchestration stand-in `ACTIVE` with wallet `ACTIVE`. The wallet credentials can be retrieved exactly once; the second time gives 410.
6. Rotate secret: the old secret no longer works, the new one does.
7. Change roles: the next token contains the new roles. Wallet role via change: 409.
8. Delete without wallet access: 204, the client is gone from the Portal IAM, the account has status `DELETED`. With wallet access: 202, then `DELETED`.
9. Another company does not see the account (404).
10. `create-for-subscription` with a provider-only role creates two accounts. A repetition with the same `stepRunId` creates no more accounts and returns new secrets.
11. `/internal/technical-users/{id}` reports `usableForConnector` only for own, active accounts with origin `OWN`.
12. No secret appears in the log, in `step_run` or unencrypted in the database.

## 12. Test data

The test Portal IAM needs:

- the client `technical_roles_management` with the roles from section 6.3, descriptions as attributes `description.de` and `description.en`
- the permissions from section 6.2 for test users and service accounts
- the claim `company_id` in the tokens of the test users
- a service account for the module with the permissions listed under `IAM_ADMIN_CLIENT_ID`

`stub-companies.json` contains at least one active company with BPN and role `ACTIVE_PARTICIPANT`, one with role `APP_PROVIDER` and one without BPN.
