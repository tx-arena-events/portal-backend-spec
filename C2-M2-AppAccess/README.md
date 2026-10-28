# C2-M2 Users and App Access

**Version:** 0.2, as of 2026-10-28
**Capability:** C2 Identity and Access
**Kind:** Hackathon
**Files:** this README (behaviour), [openapi.yaml](openapi.yaml) (interface), [open-points.md](open-points.md)

This README and the OpenAPI description are sufficient to build the module. The binding framework is [`../FRAMEWORK.md`](../FRAMEWORK.md) (OSP 2.0). The folder keeps the name `C2-M2-AppAccess`.

## 1. Purpose

The module manages the people in the Portal IAM of the onboarding service provider (OSP) and their access to the portal and the apps:

- **Accounts of people.** An account is created by itself in the Portal IAM at the first login with the EUDI Wallet (configuration in C2-M5). The module then binds it to exactly one company, assigns portal roles and app roles, unbinds it and, while a company is deactivated, removes the company and the roles from its accounts. It never creates an account itself.
- **Onboarding:** after an application is approved, it binds the account of the applicant to the new company and gives the applicant the portal role `Company Admin`.
- **Invitations:** the Company Admin invites people to the company. The invited person redeems the code with their own token.
- **OSP staff:** the OSP Admin assigns the portal roles of the staff (`OSP Reviewer`, `OSP Admin`).
- **App clients:** for every app it keeps a **client template** with the app roles and their descriptions. For every instance of the app it creates an **instance client** from it.
- **App roles of people:** it assigns and removes them on the instance clients of the active subscriptions of their company. App users are the same accounts as for the portal.

It is the only module that writes bindings, portal roles of people, app clients and app roles to the Portal IAM. It has no UI of its own; the UI of the OSP calls it.

| Who | What for |
|---|---|
| Company Admin of a company (scope `company`) | view, invite and remove people of the company; set their portal roles and app roles |
| other people of a company (scope `company`) | view people of the company, depending on the portal role (C2-M5) |
| person without a company (scope `own`) | accept an invitation with the code |
| OSP staff (scope `osp`) | view people of all companies; invite people to any company, set portal roles in the company and remove people, to resolve stuck cases; invite staff and set their roles |
| Process Orchestration | steps: bind the Company Admin during onboarding; deactivate and reactivate the accounts of a company; create the template when an app is approved; create, enable, disable instance clients; change the URL of an instance; add app roles; change descriptions |
| Offers | read the app roles of one or more apps, e.g. for the marketplace |
| Subscriptions | read the instance client of a subscription, e.g. to pass its client ID to the provider |
| Notifications | find people of a company with a portal role, e.g. all Company Admins |

## 2. Terms

| Term | Meaning |
|---|---|
| Account | account of a person in the Portal IAM, without a password, linked to the EBW-Connector. It serves for the portal and the apps. |
| `userId` | ID of the account in the Portal IAM; also `sub` in the person's token |
| Binding | assignment of an account to exactly one company; in the Portal IAM the attributes `company_id` and `bpn` on the account, in the module's database a row in `company_user` |
| Account without a company | account without a binding, e.g. after the first login or after removal from a company; has only the portal role `Applicant` (and staff roles, if any) |
| Portal role | composite role on the client `portal-core` that bundles permissions (C2-M5). Not to be confused with app roles and with the company roles from C3-M1. |
| Company portal roles | `Company Admin`, `IT Admin`, `Member`; only for bound accounts |
| Staff portal roles | `OSP Reviewer`, `OSP Admin`; independent of the binding |
| `Applicant` | portal role of every account of a person (assigned by the identity provider at the first login, C2-M5 5.5); this module never assigns or removes it |
| OSP staff | accounts with a staff portal role |
| OSP company | the company the OSP belongs to; `ospCompanyId` from the installation settings (FRAMEWORK 4.8) |
| Invitation | offer to a person, redeemed with a code; kind `COMPANY` (binds to a company) or `STAFF` (assigns staff roles) |
| Invitation code | random, single-use and time-limited code of an invitation; the module stores only its hash |
| App | offer of kind app in the marketplace (Offers). Services have no clients and no app roles. |
| Instance | running installation of an app under a URL |
| Single-instance app | one instance for all customers; the provider sets its URL during the approval |
| Multi-instance app | one instance per subscription; the provider gives its URL when setting up the subscription |
| App role | role defined by the provider of the app, e.g. `Viewer`. People receive it on the instance client of their company's subscription. |
| Client template | one client per app in the Portal IAM. It holds the app roles with their descriptions and is disabled; nobody signs in with it. |
| Instance client | client of exactly one instance. People sign in to the app with it. It carries a copy of every app role of the template. |
| Target client | instance client on which the app roles of a person for an app apply (6.10) |

## 3. Not part of this module

| What | Where |
|---|---|
| Create the account at the first login and link it to the EBW-Connector; take over the names from the PID; assign the portal role `Applicant` | C2-M5 Portal IAM (configuration) |
| Sign-in with EUDI Wallet and Business Wallet | C5-M1 EBW-Connector |
| Define portal roles and fill them with permissions; client scopes, mappers for `company_id` and `bpn`, user profile | C2-M5 Portal IAM |
| Application, review, decision | C1-M2 Registration, C1-M3 Application Review |
| Company data, BPN, status of the company | C3-M1 Company Profile |
| Technical users | C2-M3 Technical Users |
| The app and its draft, including the planned app roles during the approval | Offers |
| Subscriptions and their status | Subscriptions |
| Deliver the invitation link (display, copy, email) | UI; the module sends nothing |
| Notifications, e.g. to the provider when app roles were added | Notifications (other part of the spec) |
| Order of the steps, retries | Process Orchestration |

## 4. Technical frame

- Python with FastAPI.
- Own database `c2_m2` in PostgreSQL with its own database user. The module reads and writes only there. Other modules read through the interface.
- People call through the UI of the OSP via HTTP, with their token from the Portal IAM. The shared token validation library checks signature, expiry and audience (`aud` contains `portal-core`) and returns the claims: `sub`, `company_id` and `bpn` (only for a bound account), `given_name`, `family_name`, the permissions (FRAMEWORK 4.2).
- Every operation requires exactly one permission with a scope (6.11, FRAMEWORK 4.3).
- Process Orchestration and other modules call the module **in the same process** through its public interface, without a token. [openapi.yaml](openapi.yaml) describes both as the same contract; the paths under `/steps` and `/internal` apply as HTTP paths when the module runs as a separate service.
- Events go out through the outbox of the base skeleton: written in the same transaction as the change, delivered afterwards (section 10).
- Secrets: instance clients are public clients, accounts have no credentials in the Portal IAM. The only secret is the invitation code: it appears only in the response that creates it; the module stores only its hash (6.8).
- A background job in the module cleans up expired invitations once an hour (6.8).

### Configuration

| Variable | Meaning | Example |
|---|---|---|
| `DATABASE_URL` | own PostgreSQL database `c2_m2` | |
| `IAM_URL` | base URL of the Portal IAM of the OSP | `https://iam.osp-a.example.org/auth` |
| `IAM_REALM` | realm of the installation | `CX-Central` |
| `IAM_ADMIN_CLIENT_ID`, `IAM_ADMIN_CLIENT_SECRET` | the module's own service account for the admin API; needs the realm permissions `manage-clients`, `view-clients`, `query-clients`, `manage-users`, `view-users`, `query-users` | |
| `PORTAL_CORE_CLIENT_ID` | client with permissions and portal roles | `portal-core` |
| `COMPANY_PORTAL_ROLES` | company portal roles, comma-separated | `Company Admin,IT Admin,Member` |
| `COMPANY_ADMIN_ROLE` | portal role of the Company Admin | `Company Admin` |
| `STAFF_PORTAL_ROLES` | staff portal roles | `OSP Reviewer,OSP Admin` |
| `STAFF_ADMIN_ROLE` | portal role of the OSP Admin | `OSP Admin` |
| `COMPANY_ID_ATTRIBUTE`, `BPN_ATTRIBUTE` | names of the attributes on the account | `company_id`, `bpn` |
| `INVITATION_TTL_HOURS` | validity of an invitation | `168` |
| `INVITATION_MAX_OPEN` | maximum number of open invitations per company or for the staff | `50` |
| `INVITATION_MAX_FAILED_ATTEMPTS` | failed attempts with differing names after which an invitation lapses | `5` |
| `INVITATION_NAME_CHECK` | check the names on acceptance (6.8) | `true` |
| `BOOTSTRAP_ADMIN_FILE` | bootstrap of the first OSP Admin (6.9); empty = none | `config/bootstrap-admin.json` |
| `SETTINGS_FILE` | installation settings (FRAMEWORK 4.8); the module reads `ospCompanyId` | `config/installation.json` |
| `CLIENT_ID_PREFIX` | prefix of the client IDs of instance clients | `app` |
| `TEMPLATE_CLIENT_ID_PREFIX` | prefix of the client IDs of templates | `app-template-` |
| `DEFAULT_CLIENT_SCOPES` | default client scopes of instance clients, comma-separated | `basic,acr,roles,web-origins,catena` |
| `OPTIONAL_CLIENT_SCOPES` | optional client scopes of instance clients | empty |
| `LANGUAGES` | languages allowed for role descriptions | `de,en` |
| `QUERY_MAX_APPS` | maximum number of apps in one roles query | `50` |
| `SUBSCRIPTION_URL`, `COMPANY_URL` | connection to Subscriptions or C3-M1: empty = built-in stand-in (11.2), `in-process` = call in the same process, otherwise HTTP address | `in-process` |

## 5. Data

### Table `app_client_template`

| Field | Type | Rule |
|---|---|---|
| `app_id` | uuid | primary key |
| `provider_company_id` | uuid | required |
| `app_name` | text | 1 to 255 characters; used as the name of the client |
| `iam_client_id` | text | unique; `TEMPLATE_CLIENT_ID_PREFIX` + `app_id`, e.g. `app-template-6f1c…` |
| `iam_client_uuid` | text | internal ID of the client in Keycloak; empty until the step has finished |
| `created_at`, `changed_at` | timestamptz | |

### Table `app_instance_client`

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key |
| `app_id` | uuid | required; a template must exist |
| `instance_kind` | text | `SINGLE` or `SUBSCRIPTION` |
| `offer_subscription_id` | uuid | required and unique for `SUBSCRIPTION`, empty for `SINGLE` |
| `customer_company_id` | uuid | required for `SUBSCRIPTION`, empty for `SINGLE` |
| `iam_client_id` | text | unique; `CLIENT_ID_PREFIX` + value of the sequence `client_number`, e.g. `app123` |
| `iam_client_uuid` | text | internal ID of the client in Keycloak; empty until the step has finished |
| `instance_url` | text | absolute `http` or `https` URL without `#`, up to 255 characters |
| `status` | text | `PENDING`, `ACTIVE`, `DISABLED` |
| `created_at`, `changed_at` | timestamptz | |

At most one row per app with `instance_kind` `SINGLE` and a status other than `DISABLED` (partial unique index). The sequence `client_number` lives in the module's database.

### Table `company`

One row per company to which the module binds accounts. C3-M1 stays the leading source for the company's data; this table holds only what the module needs for bindings.

| Field | Type | Rule |
|---|---|---|
| `company_id` | uuid | primary key |
| `bpn` | text | BPN from the step `assign-company-admin` or from C3-M1 (6.6); `^BPNL[0-9A-Z]{12}$` |
| `status` | text | `ACTIVE`, `DISABLING`, `DISABLED`, `ENABLING` (6.6) |
| `created_at`, `changed_at` | timestamptz | |

### Table `company_user`

One row per binding of an account to a company.

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key |
| `user_id` | uuid | account in the Portal IAM (`sub`) |
| `company_id` | uuid | required; a row in `company` must exist |
| `given_name`, `family_name` | text | copy of the names from the PID at binding, for lists; empty for `REMOVED` |
| `status` | text | `ACTIVE`, `DISABLED`, `REMOVED` (6.6) |
| `joined_via` | text | `ONBOARDING` or `INVITATION` |
| `invitation_id` | uuid | for `INVITATION` |
| `removal_reason` | text | `REMOVED_BY_COMPANY`, `REMOVED_BY_OSP`, `ACCOUNT_DELETED`; empty as long as not removed |
| `suspended_roles` | jsonb | only for `DISABLED`: company portal roles and app roles (client ID, role) that the account had before the deactivation (8.9); otherwise empty |
| `created_at`, `changed_at`, `removed_at` | timestamptz | |

Unique: `user_id` among the rows with status `ACTIVE` or `DISABLED` (partial unique index). So an account is bound to at most one company. Rows with `REMOVED` are kept for the audit log.

### Table `invitation`

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key |
| `kind` | text | `COMPANY` or `STAFF` |
| `company_id` | uuid | required for `COMPANY`, empty for `STAFF` |
| `given_name`, `family_name` | text | 1 to 255 characters; only while `OPEN`, cleared afterwards |
| `roles` | text[] | portal roles to assign; for `COMPANY` from `COMPANY_PORTAL_ROLES`, for `STAFF` from `STAFF_PORTAL_ROLES`; not empty |
| `code_hash` | text | SHA-256 of the code, hexadecimal, unique |
| `status` | text | `OPEN`, `ACCEPTED`, `REVOKED`, `EXPIRED` (6.8) |
| `end_reason` | text | for `REVOKED`: `REVOKED`, `COMPANY_DISABLED`, `TOO_MANY_ATTEMPTS` |
| `failed_attempts` | integer | default 0 |
| `expires_at` | timestamptz | creation + `INVITATION_TTL_HOURS` |
| `created_by` | uuid | `sub` of the inviting person; empty for the bootstrap (6.9) |
| `accepted_by`, `accepted_at` | uuid, timestamptz | for `ACCEPTED` |
| `created_at`, `ended_at` | timestamptz | |

### Table `role_assignment_log`

Log of every change of a role assignment on an account (6.13). The admin events of Keycloak are not enough for this: they name only the calling client, that is, the service account of this module.

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key |
| `occurred_at` | timestamptz | when |
| `user_id` | uuid | to whom |
| `company_id` | uuid | company of the binding; empty for staff roles |
| `role_kind` | text | `COMPANY_PORTAL`, `STAFF_PORTAL` or `APP` |
| `iam_client_id` | text | `portal-core` or the client ID of the instance client, e.g. `app123` |
| `app_id`, `app_instance_client_id` | uuid | only for `APP` |
| `role_name` | text | which role |
| `action` | text | `ADDED` or `REMOVED` |
| `actor_type` | text | `PERSON` (operation) or `PROCESS` (step) |
| `actor_sub` | uuid | for `PERSON`: `sub` of the acting person |
| `process_id`, `step_run_id` | uuid | for `PROCESS` |
| `operation` | text | `operationId` from [openapi.yaml](openapi.yaml) or the name of the step |

### Table `audit_log`

One entry per changing operation and per changing run of a step (FRAMEWORK 4.3).

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key |
| `occurred_at` | timestamptz | |
| `actor_type` | text | `PERSON`, `PROCESS` or `SYSTEM` (bootstrap) |
| `actor_sub` | uuid | for `PERSON` from the token |
| `company_id` | uuid | company concerned (binding, invitation); empty for actions of the staff without a company |
| `process_id`, `step_run_id` | uuid | for `PROCESS` |
| `operation` | text | `operationId` or name of the step, e.g. `createInvitation` |
| `object_id` | uuid | account, invitation or instance client |

**Not stored:** role assignments (portal roles, staff roles, app roles; only in the Portal IAM, the audit log records only changes); app roles and their descriptions (only in the Portal IAM); invitation codes (only their hash); names after the end of a binding or invitation; the identifier of the person at the EBW-Connector (only in the link in the Portal IAM); date of birth and other data of the PID; status of subscriptions (Subscriptions); data of the app apart from the name (Offers); further data of the company (C3-M1).

## 6. Rules

### 6.1 Client IDs

- Template: `TEMPLATE_CLIENT_ID_PREFIX` + app ID. It follows from the app ID; a repeated step finds the template again by it.
- Instance client: `CLIENT_ID_PREFIX` + next value of the sequence `client_number`. The number is taken and stored in `app_instance_client` **before** the client is created in the Portal IAM. So a repeated step uses the same ID.

### 6.2 App roles

- Name: 1 to 255 characters, unique within an app (case-sensitive, as in Keycloak).
- Descriptions: at least one; language from `LANGUAGES`, at most one per language; text 1 to 255 characters.
- In the Portal IAM, a role carries its descriptions as attributes `description.{lang}`, e.g. `description.de`, `description.en`. The field `description` of the role holds the English text, or the first one given if there is none.
- The template is the leading source. Every instance client of the app carries the same roles with the same name, the same description and the same attributes.
- Roles are only added, never removed or renamed.

### 6.3 Client settings

| Setting | Template | Instance client |
|---|---|---|
| `clientId` | see 6.1 | see 6.1 |
| `name` | name of the app | name of the app |
| `description` | `Template of app {appId}. Not for sign-in.` | `App {appId}` or `App {appId}, subscription {offerSubscriptionId}` |
| `enabled` | `false` | `false` when created, `true` after the step "enable" |
| `protocol` | `openid-connect` | `openid-connect` |
| `publicClient` | `false` | `true` |
| `standardFlowEnabled` | `false` | `true` |
| `implicitFlowEnabled`, `directAccessGrantsEnabled`, `serviceAccountsEnabled` | `false` | `false` |
| `fullScopeAllowed` | `false` | `false` |
| `consentRequired`, `frontchannelLogout` | `false` | `false` |
| `defaultClientScopes`, `optionalClientScopes` | none | `DEFAULT_CLIENT_SCOPES`, `OPTIONAL_CLIENT_SCOPES` |
| `rootUrl` | – | not set |
| `baseUrl` | – | URL of the instance |
| `redirectUris` | – | `{URL of the instance}/*`, without a double slash |
| `webOrigins` | – | `+` (origins of the redirect URIs) |
| attributes | – | `use.refresh.tokens: true`, `backchannel.logout.session.required: true` |
| protocol mapper | – | audience mapper (6.3.1) |
| client roles | all app roles | copy of all app roles of the template |

The description texts of the clients stay in English because they appear in the Keycloak UI.

The client scopes `profile`, `email`, `address`, `phone` and `microprofile-jwt` are neither default nor optional scopes of an instance client. So apps get no name and no email address of a person, even though the account carries names from the PID (FRAMEWORK 4.6).

#### 6.3.1 Audience mapper

Name `{clientId}-mapper`, `protocol` `openid-connect`, `protocolMapper` `oidc-audience-mapper`, configuration `included.client.audience`: client ID, `access.token.claim`: `true`, `id.token.claim`: `false`. Access tokens for the instance client then contain its client ID in `aud`.

### 6.4 Status of an instance client

| From | Step | To |
|---|---|---|
| – | create instance client | `PENDING` (client disabled) |
| `PENDING` | enable | `ACTIVE` (client enabled) |
| `PENDING`, `ACTIVE` | disable | `DISABLED` (client disabled) |

`DISABLED` is final. A new subscription gets a new instance client.

### 6.5 Accounts of people in the Portal IAM

| What on the account | Who sets it | When |
|---|---|---|
| Account, link to the EBW-Connector (alias and its `sub`) | Portal IAM, first broker login (C2-M5) | first login of the person |
| `firstName`, `lastName` | Portal IAM from the ID token of the EBW-Connector (C2-M5) | every login; the source is the PID |
| Portal role `Applicant` | Portal IAM, mapper of the identity provider (C2-M5 5.5) | creation; this module never changes it |
| Attributes `{COMPANY_ID_ATTRIBUTE}`, `{BPN_ATTRIBUTE}` | this module | bind (6.12.1); removed on unbind (6.12.3) |
| Company portal roles | this module | bind, set roles (6.12.2), unbind |
| Staff portal roles | this module | 6.9 |
| App roles on instance clients | this module | 7.5; removed on unbind |
| `enabled` | this module | stays `true`; the module sets it back to `true` when binding and unbinding, in case an earlier state set it to `false` |
| Email address, password, other credentials | nobody | – |

`userId` is the ID of the account in the Portal IAM and also `sub` in the token. The module always finds an account by this ID (`GET …/users/{userId}`), never by names.

**Sign-in to apps** (FRAMEWORK 5.8): The app redirects to the Portal IAM of the OSP. The Portal IAM forwards to the EBW-Connector, its only identity provider; the app needs no hint `kc_idp_hint`. Afterwards the Portal IAM issues a token for the instance client. It contains:

- `sub` (the `userId`) as a pseudonymous identifier,
- `company_id` and `bpn` from the attributes (mappers in the client scope `catena`, defined in C2-M5),
- the app roles of this instance client under `resource_access.{clientId}.roles`,
- the client ID of the instance client in `aud` (6.3.1),
- no names and no email address (6.3).

An account without a company can sign in to an app but gets neither `company_id` nor app roles there. While the company is deactivated, the account has no company and no company roles (8.9).

### 6.6 Binding to a company

**Status of a binding** (`company_user.status`):

| From | Trigger | To |
|---|---|---|
| – | `assign-company-admin` (8.8) or invitation accepted (7.10) | `ACTIVE` |
| `ACTIVE` | `disable-company-users` (8.9) | `DISABLED`; the account stays enabled but loses company, company portal roles and app roles (saved in `suspended_roles`) |
| `DISABLED` | `enable-company-users` (8.10) | `ACTIVE`; company and saved roles are restored |
| `DISABLED` | `assign-company-admin` for this account and this company | `ACTIVE`; company and saved roles are restored, plus `Company Admin` |
| `ACTIVE` | Company Admin removes the person (7.4) | `REMOVED`, reason `REMOVED_BY_COMPANY` |
| `ACTIVE`, `DISABLED` | OSP staff removes the person (7.14) | `REMOVED`, reason `REMOVED_BY_OSP`; account without a company, saved roles are dropped |
| `ACTIVE`, `DISABLED` | account missing in the Portal IAM (6.12.3, 8.9, 8.10) | `REMOVED`, reason `ACCOUNT_DELETED` |

`REMOVED` is final for this row. The account remains, without a company, with `Applicant`. If it is bound again later, a new row is created.

**Status of a company** (`company.status`):

| From | Trigger | To |
|---|---|---|
| – | `assign-company-admin` creates the row | `ACTIVE` |
| – | invitation for a company without a row that is `ACTIVE` in C3-M1 (7.7, 7.20) | `ACTIVE` |
| `ACTIVE` | `disable-company-users` starts | `DISABLING` |
| `DISABLING` | all bindings of the company are `DISABLED` | `DISABLED` |
| `DISABLED` | `enable-company-users` starts | `ENABLING` |
| `ENABLING` | all `DISABLED` bindings are `ACTIVE` again | `ACTIVE` |
| `DISABLED` | `assign-company-admin` (return after a transfer, FRAMEWORK 5.6) | `ACTIVE` |

New invitations and new bindings exist only for `ACTIVE`. The intermediate states `DISABLING` and `ENABLING` make the two steps repeatable (8.9, 8.10).

**Company without a row.** If an invitation is to be created for a company without a row in `company` (7.7, 7.20), the module reads the company from C3-M1 (11.2). If it is `ACTIVE` there and has a BPN, the module creates the row with this BPN and status `ACTIVE`; otherwise 409 `company-not-active`. This mainly concerns the OSP company: C3-M1 creates it at the first start from seed data, without onboarding and therefore without `assign-company-admin`.

### 6.7 Portal roles of people

- The module assigns only whole portal roles, never single permissions. C2-M5 defines what a portal role contains.
- **Company portal roles** (`COMPANY_PORTAL_ROLES`) are held only by a bound account. Every binding has at least one of them. They are set by the Company Admin (scope `company`) and by the OSP staff (scope `osp`).
- **Staff portal roles** (`STAFF_PORTAL_ROLES`) are set only by the OSP Admin (6.9). Company Admins can neither assign nor remove them.
- The module never assigns or removes `Applicant`; the role stays on bound accounts too. Whether a permission in scope `own` also applies with a company is decided by the module that checks it (e.g. C1-M2: new applications only without a company; here: 7.10).
- Roles are always set as the complete list of one role set (6.12.2). Roles of other sets on the same account stay untouched.

**Limits:**

1. **Last Company Admin.** A company keeps at least one account with `Company Admin` and binding `ACTIVE`. Through scope `company`, the role cannot be removed from the last Company Admin, and the last Company Admin cannot be removed, not even by themselves: 409 `last-company-admin`. Counted are the accounts that have the role `COMPANY_ADMIN_ROLE` in the Portal IAM (`GET …/clients/{portalCoreUuid}/roles/{role}/users`) and a binding `ACTIVE` in this company. The OSP staff is not bound by this limit (7.13, 7.14), so it can resolve stuck cases, e.g. when the only Company Admin has left the company; it then invites a new person as Company Admin (7.20).
2. **Last OSP Admin.** The installation keeps at least one enabled account with `STAFF_ADMIN_ROLE`: 409 `last-osp-admin`.
3. **One binding per account.** An account with binding `ACTIVE` or `DISABLED` cannot be bound to another company: 409 `user-already-bound`. It must be removed first.
4. **Staff and customers kept apart.** An account with a staff portal role can only be bound to the OSP company (`ospCompanyId`), and only an account without a company or with a binding to the OSP company gets staff roles: otherwise 409 `staff-company-conflict`. If `ospCompanyId` is not set, only accounts without a company.
5. **Bindings of a deactivated company** (`DISABLED`) are changed only by the steps 8.8 and 8.10 and by the OSP staff through removal (7.14).

### 6.8 Invitations

- **Kinds:** `COMPANY` binds the account to the company of the invitation and gives the company portal roles from the invitation (Company Admin 7.7, staff 7.20). `STAFF` gives staff portal roles (OSP Admin, 7.17).
- **Code:** 32 random bytes, base64url without padding, so 43 characters (`^[A-Za-z0-9_-]{43}$`). The module returns it exactly once, in the response of the creation, and stores only `code_hash` = SHA-256 of the code (UTF-8), hexadecimal. The code never appears in logs, events, step results or the audit log. Given the length of the code, a plain hash is enough.
- **Link:** The UI builds the link. The code in the fragment is recommended, e.g. `https://portal.osp-a.example.org/invitation#code=…`, so that it does not end up in server logs. The module accepts the code only in the body (7.10), never in the path or query.
- **Validity:** once, until `expires_at` (`INVITATION_TTL_HOURS`).
- **Name check:** If `INVITATION_NAME_CHECK` is `true`, only the person whose names in the token (`given_name`, `family_name`, from the PID) match the names of the invitation can accept it. The comparison uses the normalisation of C5-M1 6.5: Unicode NFC, leading and trailing spaces removed, case-insensitive. So a forwarded link is of no use to another person.
- **Failed attempts:** Every mismatch of the names increments `failed_attempts`. At `INVITATION_MAX_FAILED_ATTEMPTS` the invitation lapses (`REVOKED`, reason `TOO_MANY_ATTEMPTS`).
- **Maximum:** per company or for the staff at most `INVITATION_MAX_OPEN` open, unexpired invitations: otherwise 409 `too-many-invitations`.
- **Names only while open:** When an invitation ends (`ACCEPTED`, `REVOKED`, `EXPIRED`), the module clears `given_name` and `family_name`. The row stays for the audit log.
- **Cleanup:** Once an hour the module sets open invitations with `expires_at` in the past to `EXPIRED` and clears their names. Queries treat such invitations as `EXPIRED` even before that.

| From | Trigger | To |
|---|---|---|
| – | Company Admin (7.7), staff (7.17, 7.20) or bootstrap (6.9) creates it | `OPEN` |
| `OPEN` | accepted (7.10) | `ACCEPTED` |
| `OPEN` | revoked (7.9, 7.19, 7.22) | `REVOKED`, reason `REVOKED` |
| `OPEN` | company deactivated (8.9) | `REVOKED`, reason `COMPANY_DISABLED` |
| `OPEN` | `INVITATION_MAX_FAILED_ATTEMPTS` failed attempts | `REVOKED`, reason `TOO_MANY_ATTEMPTS` |
| `OPEN` | `expires_at` passed | `EXPIRED` |

`ACCEPTED`, `REVOKED` and `EXPIRED` are final. After the reactivation of a company, the Company Admin invites again.

### 6.9 OSP staff and bootstrap

- The OSP Admin assigns staff portal roles through an invitation of kind `STAFF` (7.17) or directly to a known account (7.16). Directly works only for accounts that are already staff or are bound to the OSP company; the staff does not know other accounts and invites them.
- Staff roles are independent of a binding. The staff can work without a company. The OSP Admin invites the first Company Admin of the OSP company (7.20), possibly themselves.
- Limits 2 and 4 from 6.7 apply.

**Bootstrap.** Accounts are created only at the first login; so nobody can invite the first OSP Admin in the portal. For this there is `BOOTSTRAP_ADMIN_FILE`:

```json
{
  "invitationId": "b0000000-0000-4000-8000-000000000001",
  "givenName": "Otto",
  "familyName": "Normalverbraucher",
  "roles": ["OSP Admin"],
  "codeSha256": "c68d28b4c81ec4b186207d59bbed6e865c82d16020cf9ca6b75bdfba7ac89b88",
  "expiresAt": "2026-11-30T00:00:00Z"
}
```

- Whoever sets up the installation generates the code themselves (32 random bytes, base64url, 43 characters), gives it to the person and enters only the hash. `roles` missing: `["OSP Admin"]`. `expiresAt` missing: start + `INVITATION_TTL_HOURS`.
- At start the module checks: if no enabled account has `STAFF_ADMIN_ROLE` and there is no invitation with `invitationId` yet, it creates this invitation of kind `STAFF` with status `OPEN`, with an audit log entry (`actor_type` `SYSTEM`). Otherwise it does nothing.
- The person signs in (the account is created) and accepts the invitation like any other (7.10).
- For another bootstrap, e.g. after all OSP Admins are gone, a new `invitationId` with a new code is entered.

### 6.10 Target clients for app roles

The app roles of a person for an app apply on all **target clients**: the instance clients of this app with status `ACTIVE` and a stored `iam_client_uuid` that belong to an active subscription of the person's company.

- Multi-instance app: the instance client with the `offer_subscription_id` of an active subscription.
- Single-instance app: the one instance client with `instance_kind` `SINGLE`, if the company has an active subscription to the app.

Subscriptions provides the active subscriptions (section 11). The same roles apply on all target clients. Assignments on other instance clients of the app, e.g. on one disabled after the subscription ended, stay untouched; they have no effect because the client is disabled. When a new subscription is added, people get roles there only when the Company Admin assigns them.

### 6.11 Permissions

| Permission | Scope | Operations |
|---|---|---|
| `view_company_users` | `company` | list and view people of the company (7.1, 7.2) |
| `manage_company_users` | `company` | set portal roles, remove people, create, list and revoke invitations (7.3, 7.4, 7.7, 7.8, 7.9) |
| `view_app_roles` | `company` | read the app roles of an app for the company (7.6) |
| `assign_app_roles` | `company` | set the app roles of a person (7.5) |
| `accept_invitation` | `own` | accept an invitation (7.10) |
| `view_users` | `osp` | view people of all companies, their invitations and the staff (7.11, 7.12, 7.15, 7.21) |
| `manage_users` | `osp` | for every company: set company portal roles, remove people, invite, revoke invitations (7.13, 7.14, 7.20, 7.22) |
| `manage_osp_staff` | `osp` | set staff roles; create, list, revoke staff invitations (7.16 to 7.19) |
| `query_c2_m2` | `osp` | queries under `/internal` (section 9), only via HTTP |
| `execute_steps_c2_m2` | `osp` | steps under `/steps` (section 8), only via HTTP |

Checks on calls by people, in this order:

1. Token valid, otherwise 401 `unauthorized`.
2. Permission in the token, otherwise 403 `forbidden`.
3. Scope `company`: `company_id` in the token equals `{companyId}` in the path, otherwise 404 `not-found`. The account, invitation or binding in the path belongs to this company, otherwise 404 `not-found`.
4. Scope `osp`: only the permission.
5. Scope `own`: the module acts only for the account from `sub` in the token; an invitation of kind `COMPANY` binds only an account without a company (7.10).

C2-M5 defines which portal role contains which permissions. Proposal of this module: `Applicant`: `accept_invitation`; `Member`: no permissions of this module; `IT Admin`: `view_company_users`, `view_app_roles`; `Company Admin`: `view_company_users`, `manage_company_users`, `view_app_roles`, `assign_app_roles`; `OSP Reviewer`: `view_users`; `OSP Admin`: `view_users`, `manage_users`, `manage_osp_staff`.

### 6.12 Procedures in the Portal IAM

Several operations and steps use the same procedures. Calls to the admin API as in 11.1.

#### 6.12.1 Bind

Input: `userId`, company (`companyId`, BPN from the table `company`), company portal roles, route (`ONBOARDING` or `INVITATION`), invitation and names where applicable.

1. Read the account. Missing: 404 `user-not-found`.
2. Set the attributes `{COMPANY_ID_ATTRIBUTE}` = `[companyId]`, `{BPN_ATTRIBUTE}` = `[BPN]` and `enabled` = `true` if anything differs (`PUT` with the read representation).
3. Set the company portal roles to exactly the list (6.12.2).
4. In one transaction: row in `company_user` with status `ACTIVE`; one entry `ADDED` in `role_assignment_log` per added role; entry in `audit_log`; event `UserAssignedToCompany`.

The names of the row come from `firstName` and `lastName` of the account for onboarding, and from `given_name` and `family_name` of the token for an invitation. Both come from the PID.

If the procedure stops after step 2 or 3, the account has attributes without a row in `company_user` and counts as unbound for the module. The repetition sets the same values and writes the row. Because steps 2 and 3 set the values exactly, any other later binding also overwrites such leftovers.

#### 6.12.2 Set portal roles

Input: `userId`, role set (company portal roles or staff roles), new complete list.

1. Read the account's assignments on the client `portal-core`.
2. Assign the roles of the list that are missing.
3. Remove the roles of the role set that are not in the list.
4. `Applicant` and the roles of the other set stay.

Result: added and removed roles. The module reads the role IDs from the client `portal-core` at start and keeps them in memory; an unknown name triggers a new read.

#### 6.12.3 Unbind

Input: `userId`, binding, reason.

1. Read all role mappings of the account (`GET …/users/{userId}/role-mappings`). Account missing: continue with step 6, reason `ACCOUNT_DELETED`.
2. App roles: for every client under `clientMappings` whose client ID belongs to an instance client of this module, remove all assignments.
3. Remove the company portal roles. Staff roles and `Applicant` stay.
4. Read the account; remove the attributes `{COMPANY_ID_ATTRIBUTE}` and `{BPN_ATTRIBUTE}` (empty list) and set `enabled` = `true` (`PUT`).
5. End the account's sessions (`POST …/users/{userId}/logout`). Issued access tokens stay valid until they expire (C2-M5); a refresh fails.
6. In one transaction: binding `REMOVED` with reason and `removed_at`, clear the names; one entry `REMOVED` in `role_assignment_log` per removed role; entry in `audit_log`; event `UserRemovedFromCompany`.

The account remains, without a company and with `Applicant`. The person can submit an application or accept a new invitation.

### 6.13 Audit log

- Every changing operation and every step that changes something writes an entry in `audit_log`, in the same transaction as the change.
- Every added or removed role writes an entry in `role_assignment_log`: company portal roles, staff roles and app roles per instance client.
- No secrets, no invitation codes, no names (FRAMEWORK 4.3).

### 6.14 Repetition and concurrency

Every step can be repeated with the same input and then changes nothing further:

- It finds its object by a key from the input: the app ID for the template, the subscription ID or (for `SINGLE`) the app ID for an instance client, the ID of the instance client for the other client steps, `companyId` and `applicantSub` for `assign-company-admin`, `companyId` for `disable-company-users` and `enable-company-users`.
- In the Portal IAM it creates only what is missing and sets only what differs.
- It returns the same result as the first time.

So Process Orchestration can simply run a failed step again. `processId` and `stepRunId` serve only for the audit log. A step publishes an event only when it changed something.

Operations of people:

- `PUT …/portal-roles` and `PUT …/roles` set complete lists; a repetition changes nothing.
- `DELETE` on an already removed person or an already revoked invitation answers 204 and changes nothing.
- A repeated acceptance of the same invitation by the same account answers 200 with the same result (7.10).
- `POST …/invitations` is not repeatable: every call creates a new invitation with a new code.

Advisory locks in PostgreSQL:

- per app ID for all steps of an app, so that the roles of the template and of the instance clients stay the same;
- per `companyId` for everything that changes bindings or company portal roles (7.3, 7.4, 7.10, 7.13, 7.14, 8.8 to 8.10), so that the limit "last Company Admin" holds;
- per `userId` for everything that changes an account;
- one lock for all changes to staff roles (7.10 for `STAFF`, 7.16), so that the limit "last OSP Admin" holds;
- per invitation on acceptance.

The module always takes several locks in this order: app, company, staff, invitation, account.

## 7. Operations

People call all operations of this section via HTTP with their token. Permissions and checks as in 6.11. Lists as in FRAMEWORK 4.1: `page` from 0, `size` 1 to 100 (default 20), response `{ items, page, size, total }`.

**People of the company** (tag `Company`)

### 7.1 `GET /companies/{companyId}/users`

- Bindings of the company from `company_user`, sorted by `family_name`, `given_name`, then `created_at`.
- Filter: `status` (may be given several times; default `ACTIVE`, `DISABLED`).
- Per entry (`CompanyUser`): `userId`, `companyId`, `givenName`, `familyName`, `status`, `joinedVia`, `portalRoles`, `createdAt`, `changedAt`. The module reads `portalRoles` per entry in the Portal IAM (assignments on the client `portal-core`, only roles from `COMPANY_PORTAL_ROLES`); for `REMOVED` the list is empty.

### 7.2 `GET /companies/{companyId}/users/{userId}`

Binding `ACTIVE` or `DISABLED` to this company, otherwise 404 `not-found`. The module reads all role mappings of the account in one call (`GET …/users/{userId}/role-mappings`):

- `portalRoles`: assignments on the client `portal-core`, only roles from `COMPANY_PORTAL_ROLES`;
- `appRoles`: per instance client of this module `appId`, `appInstanceClientId`, `iamClientId`, status of the instance client, role names. Assignments on other clients are left out.

### 7.3 `PUT /companies/{companyId}/users/{userId}/portal-roles`

Input: `portalRoles`, the complete new list of company portal roles.

1. Checks as in 6.11. Binding to this company: none or `REMOVED`: 404 `not-found`; `DISABLED`: 409 `invalid-state`.
2. List not empty, without duplicates, only from `COMPANY_PORTAL_ROLES`; otherwise 400 `validation`.
3. If the account loses `COMPANY_ADMIN_ROLE` and it is the last Company Admin of the company: 409 `last-company-admin` (6.7).
4. Set portal roles (6.12.2).
5. In one transaction: one entry in `role_assignment_log` per change, one entry in `audit_log`, and, if something changed, event `UserRolesChanged` (without `appId`).
6. Response 200 with `CompanyUser`.

The person sees the new permissions only with the next token.

### 7.4 `DELETE /companies/{companyId}/users/{userId}`

1. Checks as in 6.11. Binding to this company: `ACTIVE` → continue; the last binding of the account to this company is `REMOVED` → 204 without change; none → 404 `not-found`; `DISABLED` → 409 `invalid-state`.
2. If the account is the last Company Admin of the company: 409 `last-company-admin`.
3. Unbind (6.12.3), reason `REMOVED_BY_COMPANY`.
4. Response 204.

### 7.5 `PUT /companies/{companyId}/users/{userId}/apps/{appId}/roles`

Input: `roles`, the complete new list of the person's app roles for this app. An empty list removes all app roles of the app.

1. Checks as in 6.11. Binding `ACTIVE` to this company, otherwise 404 `not-found` (none, `REMOVED`) or 409 `invalid-state` (`DISABLED`). Duplicate names: 400 `validation`.
2. A template of the app with a stored `iam_client_uuid` must exist, otherwise 409 `template-missing`. Read the roles of the template; every role of the input must exist there, otherwise 400 `unknown-app-role`.
3. Determine the target clients as in 6.10 (query to Subscriptions). None: 409 `no-active-subscription`.
4. Read the roles of every target client. If a role of the input is missing there (the step `add-app-roles` has not finished yet): 409 `invalid-state`; nothing is changed.
5. Per target client, read the current assignments of the account; assign missing roles, remove app roles no longer listed.
6. In one transaction: one entry in `role_assignment_log` per changed assignment, one entry in `audit_log`, and, if something changed, event `UserRolesChanged` with `appId`.
7. Response 200: `userId`, `appId`, `roles` (now valid), `addedRoles`, `removedRoles` (names added or removed on at least one target client), `instanceClients` (the target clients).

If the Portal IAM does not answer as expected in step 5: the changes already made are logged as in step 6, response 502 `iam-error`. The same call again completes the rest.

### 7.6 `GET /companies/{companyId}/apps/{appId}/roles`

The app roles of an app from its template, as in query 9.1 for one app, with `lang` for the language of `description`. In addition, for this company: `assignable` (`true` if there is at least one target client as in 6.10) and `instanceClients` (the target clients). Without a template: 404 `not-found`.

**Invitations** (tag `Company`)

### 7.7 `POST /companies/{companyId}/invitations`

Input: `givenName`, `familyName`, `portalRoles`.

1. Checks as in 6.11. Names 1 to 255 characters each, without control characters; `portalRoles` not empty, without duplicates, only from `COMPANY_PORTAL_ROLES`; otherwise 400 `validation`.
2. Company in the table `company` with status `ACTIVE`; if the row is missing, check with C3-M1 as in 6.6; otherwise 409 `company-not-active`.
3. Maximum of open invitations (6.8), otherwise 409 `too-many-invitations`.
4. Generate the code, compute the hash. Row in `invitation`: kind `COMPANY`, status `OPEN`, `expires_at`, `created_by` = `sub`. In the same transaction: entry in `audit_log`.
5. Response 201 with the invitation (`Invitation`) and `code`. The code appears only in this response.

### 7.8 `GET /companies/{companyId}/invitations`

Invitations of the company, newest first. Filter `status` (may be given several times; default `OPEN`). Expired open invitations appear as `EXPIRED`. Names only for `OPEN`. Never a code.

### 7.9 `DELETE /companies/{companyId}/invitations/{invitationId}`

Invitation of kind `COMPANY` of this company, otherwise 404 `not-found`. `OPEN`: status `REVOKED`, reason `REVOKED`, clear the names, entry in `audit_log`, response 204. `REVOKED`: 204 without change. `ACCEPTED` or `EXPIRED`: 409 `invitation-not-open`.

### 7.10 `POST /invitations/accept`

Scope `own`. Input: `code`.

1. Checks as in 6.11. Code not in the format from 6.8: 400 `validation`.
2. Find the invitation by `code_hash`. None: 404 `invitation-not-found`.
3. Status `ACCEPTED` and `accepted_by` = `sub`: response 200 with the same result as the first time. Otherwise status not `OPEN`: 409 `invitation-not-open`. `expires_at` passed: status `EXPIRED`, clear the names, 409 `invitation-not-open`.
4. Check the account, using the module's own database and the roles in the Portal IAM:
   - Kind `COMPANY`: binding `ACTIVE` or `DISABLED` exists: 409 `user-already-bound`. Company in `company` not `ACTIVE`: 409 `company-not-active`. The account has a staff role and the company is not `ospCompanyId`: 409 `staff-company-conflict`.
   - Kind `STAFF`: binding to a company other than `ospCompanyId`: 409 `staff-company-conflict`.
5. Name check (6.8). If a name differs: increment `failed_attempts` and let the invitation lapse if applicable, commit, response 403 `name-mismatch`. Names missing in the token count as a mismatch.
6. Kind `COMPANY`: bind (6.12.1) with the roles of the invitation, route `INVITATION`, names from the token. Kind `STAFF`: set staff roles (6.12.2), list = existing staff roles plus the roles of the invitation; event `OspStaffRolesChanged` if something changed.
7. In the same transaction as the binding or the event: invitation `ACCEPTED`, `accepted_by` = `sub`, `accepted_at`, clear the names; entry in `audit_log`.
8. Response 200: `invitationId`, `kind`, `userId`, `companyId` (for `COMPANY`), `roles`.

The UI then fetches a new token (refresh). Only this token contains `company_id`, `bpn` and the permissions of the new portal roles.

**People of all companies** (tag `OSP`)

### 7.11 `GET /osp/users`

Bindings from `company_user` as in 7.1, across all companies. Filters: `companyId`, `status` (default `ACTIVE`, `DISABLED`). Sorted by `companyId`, then as in 7.1.

### 7.12 `GET /osp/users/{userId}`

Any account of the installation, also without a company. Read the account in the Portal IAM; unknown: 404 `user-not-found`. Response (`UserAccount`): `userId`, `givenName`, `familyName` (from `firstName`, `lastName` of the account), `enabled`, `createdAt` (creation in the Portal IAM), `binding` (binding `ACTIVE` or `DISABLED`, otherwise `null`), `portalRoles`, `staffRoles`, `appRoles` (as in 7.2).

### 7.13 `PUT /osp/companies/{companyId}/users/{userId}/portal-roles`

As 7.3 for any company, without the limit "last Company Admin". Binding `ACTIVE` to this company, otherwise 404 `not-found` or 409 `invalid-state` (`DISABLED`).

### 7.14 `DELETE /osp/companies/{companyId}/users/{userId}`

As 7.4 for any company, without the limit "last Company Admin", reason `REMOVED_BY_OSP`. A binding `DISABLED` can be removed as well; the saved roles are dropped. A person of a deactivated company can submit an application of kind `REACTIVATION` without this too: their account is not bound during the deactivation (8.9).

**OSP staff** (tag `OSP`)

### 7.15 `GET /osp/staff`

Accounts with at least one role from `STAFF_PORTAL_ROLES`: per role `GET …/clients/{portalCoreUuid}/roles/{role}/users`, page by page to the end, merged. Per account (`StaffMember`): `userId`, `givenName`, `familyName`, `enabled`, `staffRoles`, `companyId` (binding `ACTIVE` or `DISABLED`, otherwise `null`). Sorted by `familyName`, pages as in FRAMEWORK 4.1.

### 7.16 `PUT /osp/staff/{userId}/roles`

Input: `staffRoles`, the complete new list of staff roles; empty = the account is no longer staff.

1. Checks as in 6.11. Only roles from `STAFF_PORTAL_ROLES`, without duplicates; otherwise 400 `validation`.
2. Read the account in the Portal IAM; unknown: 404 `user-not-found`.
3. If the account has no staff role yet and the list is not empty: it must be bound to the OSP company, otherwise 409 `staff-company-conflict`. The OSP Admin invites accounts without a company (7.17).
4. If the account loses `STAFF_ADMIN_ROLE` and it is the last enabled account with it: 409 `last-osp-admin`.
5. Set roles (6.12.2, staff role set).
6. In one transaction: one entry in `role_assignment_log` per change, one entry in `audit_log`, and, if something changed, event `OspStaffRolesChanged`.
7. Response 200 with `StaffMember`.

### 7.17 `POST /osp/staff-invitations`

As 7.7 with kind `STAFF`: input `givenName`, `familyName`, `staffRoles` (not empty, only from `STAFF_PORTAL_ROLES`). No check of a company; maximum of open invitations for the staff (6.8). Response 201 with the invitation and `code`.

### 7.18 `GET /osp/staff-invitations`

As 7.8 for the invitations of kind `STAFF`.

### 7.19 `DELETE /osp/staff-invitations/{invitationId}`

As 7.9 for an invitation of kind `STAFF`.

**Invitations to a company by the staff** (tag `OSP`)

### 7.20 `POST /osp/companies/{companyId}/invitations`

As 7.7 for any company; `created_by` is the staff member. With this, the OSP Admin invites the first Company Admin of the OSP company, or a new person when a company no longer has a Company Admin. If the row in `company` is missing, 6.6 applies (query to C3-M1).

### 7.21 `GET /osp/companies/{companyId}/invitations`

As 7.8 for any company.

### 7.22 `DELETE /osp/companies/{companyId}/invitations/{invitationId}`

As 7.9 for any company.

## 8. Steps for Process Orchestration

Common rules (FRAMEWORK 4.4):

- Every step receives `processId` and `stepRunId`, only for the audit log and traceability.
- Every step is repeatable (6.14).
- All steps of this module are short: they are finished when they answer. The result has `state` `DONE`.
- On an error the step reports a problem with `retryable` (section 12) and leaves in place what it has already created; a repetition completes it.
- Results contain no secrets and no names.
- Every step names its **preconditions**. Process Orchestration decides the order, not the module.

For orientation, the flows that use the steps (the flows themselves belong to Process Orchestration, FRAMEWORK 5):

| Flow | Triggered by | Steps of this module |
|---|---|---|
| Onboarding, after the approval (FRAMEWORK 5.2) | `ApplicationApproved`, after `register-membership` | `assign-company-admin` |
| Transfer to this OSP (FRAMEWORK 5.6) | as onboarding | `assign-company-admin` |
| Deactivation of a company (FRAMEWORK 5.4) | `CompanyDeactivated` | `disable-company-users` |
| Reactivation of a company (FRAMEWORK 5.5) | `CompanyReactivated` | `enable-company-users` |
| App approved (FRAMEWORK 5.8) | Offers | `create-client-template`; for a single-instance app also `create-instance-client` (`SINGLE`, URL from the approval) and `enable-instance-client` |
| Subscription to a multi-instance app set up | Subscriptions | `create-instance-client` (`SUBSCRIPTION`); at activation `enable-instance-client` |
| Subscription to a single-instance app | Subscriptions | none |
| Provider adds app roles to an active app | Offers | `add-app-roles` |
| Provider changes role descriptions | Offers | `update-role-descriptions` |
| Provider changes the URL of an instance | Offers or Subscriptions | `update-instance-url` |
| Subscription ended | Subscriptions | `disable-instance-client` |

### 8.1 `create-client-template`

Input: `appId`, `providerCompanyId`, `appName`, `roles` (at least one; name and descriptions).

Preconditions: none in the module. Process Orchestration calls the step when the app is approved.

1. Validate as in 6.2, otherwise 400 `validation`.
2. Row in `app_client_template` exists: continue with it. Otherwise create it.
3. In the Portal IAM:
   1. Find the client by client ID. Missing: create it with the template settings (6.3).
   2. For every role of the input: missing on the client, create it; present, set description and attributes if they differ.
4. Store `iam_client_uuid`. If it was empty before: event `AppClientTemplateCreated`, in the same transaction. Otherwise, if roles were created: event `AppRolesAdded` with their names. Aligning existing roles publishes no event.
5. Result: `appId`, `iamClientId`, names of all app roles.

A role on the template that is not in the input stays. To add roles later: 8.6.

### 8.2 `create-instance-client`

Input: `appId`, `instanceKind`, `instanceUrl`; for `SUBSCRIPTION` also `offerSubscriptionId` and `customerCompanyId`.

Preconditions: template of the app created (8.1 finished, `iam_client_uuid` stored).

1. Validate, otherwise 400 `validation`: URL as in section 5; for `SUBSCRIPTION` both IDs given, for `SINGLE` neither of them.
2. A template for `appId` with `iam_client_uuid` must exist, otherwise 409 `template-missing`.
3. Find the row: for `SUBSCRIPTION` by `offerSubscriptionId`, for `SINGLE` by `appId` with a status other than `DISABLED`.
   - Found with status `DISABLED` (only possible for `SUBSCRIPTION`): 409 `invalid-state`.
   - Found with a different `appId`, `customerCompanyId` or `instanceUrl`: 409 `instance-exists`.
   - Not found: take the next number, create the row with status `PENDING`.
4. In the Portal IAM:
   1. Find the client by client ID. Missing: create it, disabled, with the instance settings (6.3).
   2. Audience mapper missing: create it.
   3. Read the roles of the template (`briefRepresentation=false`). Create every role missing on the instance client with name, description and attributes; existing roles: align description and attributes.
5. Store `iam_client_uuid`. If it was empty before: event `AppInstanceClientCreated`, in the same transaction.
6. Result: the instance client (`AppInstanceClient`).

### 8.3 `enable-instance-client`

Input: `appInstanceClientId`.

Preconditions: instance client created (8.2 finished), status `PENDING` or `ACTIVE`.

- Unknown ID: 404 `not-found`.
- Status `PENDING` without `iam_client_uuid` (creation not finished): 409 `invalid-state`.
- Status `PENDING`: enable the client in the Portal IAM, status `ACTIVE`, event `AppInstanceClientEnabled`.
- Status `ACTIVE`: nothing to do.
- Status `DISABLED`: 409 `invalid-state`.

Result: the instance client.

### 8.4 `disable-instance-client`

Input: `appInstanceClientId`.

Preconditions: row of the instance client exists.

- Unknown ID: 404 `not-found`.
- Status `PENDING` or `ACTIVE`: disable the client in the Portal IAM, status `DISABLED`, event `AppInstanceClientDisabled`. Without `iam_client_uuid` only the status changes; a client that already exists from the interrupted creation is found by client ID and disabled anyway.
- Status `DISABLED`: nothing to do.

The client and the role assignments of people stay in the Portal IAM. Result: the instance client.

### 8.5 `update-instance-url`

Input: `appInstanceClientId`, `instanceUrl`.

Preconditions: instance client created (`iam_client_uuid` stored), status not `DISABLED`.

- Unknown ID: 404 `not-found`. Status `DISABLED` or no `iam_client_uuid`: 409 `invalid-state`.
- Otherwise set `baseUrl` and `redirectUris` (`{URL}/*`) of the client and `instance_url` of the row, event `AppInstanceUrlUpdated`. Same URL: nothing to do.

Result: the instance client.

### 8.6 `add-app-roles`

Input: `appId`, `roles` (at least one; name and descriptions).

Preconditions: template of the app created (`iam_client_uuid` stored).

1. Validate as in 6.2, otherwise 400 `validation`. A template with a stored `iam_client_uuid` must exist, otherwise 409 `template-missing`.
2. Roles that already exist on the template with at least one attribute `description.{lang}` are skipped; their descriptions stay unchanged (use 8.7 for that). A role on the template without such an attribute is left over from an interrupted run; it is completed with the given descriptions and counts as added.
3. Create the remaining roles on the template (create, then set description and attributes, 11.1).
4. Every instance client of the app with a stored `iam_client_uuid`, in every status: create every role of the template that is missing there; give the roles added in this step the description and attributes of the template. Instance clients without `iam_client_uuid` are skipped; their step 8.2 copies all roles when it is repeated.
5. Event `AppRolesAdded` with the added role names, if there are any.
6. Result: `appId`, `addedRoles`, `skippedRoles`.

### 8.7 `update-role-descriptions`

Input: `appId`, `roles` (name and the complete new list of descriptions).

Preconditions: template of the app created; every named role exists on the template.

1. Validate as in 6.2, otherwise 400 `validation`. A template with a stored `iam_client_uuid` must exist, otherwise 409 `template-missing`. Every role must exist on the template, otherwise 404 `role-not-found`; nothing is changed then.
2. For every role: set the attributes `description.{lang}` to exactly the given descriptions (remove those of other languages) and the field `description` (6.2); on the template, then on every instance client of the app with a stored `iam_client_uuid`. If the role is missing on an instance client, create it there.
3. Event `AppRoleDescriptionsUpdated` with the changed role names, if there are any.
4. Result: `appId`, `updatedRoles`.

### 8.8 `assign-company-admin`

Input: `companyId`, `bpn`, `applicantSub` (`sub` of the applicant in the Portal IAM, from the application in C1-M2).

Preconditions: application approved (`ApplicationApproved`); BPN assigned (`CompanyRegisteredAtCsp`, `CompanyIdentifierAssigned`). The account of the applicant exists because the applicant signed in before applying. The company does not need to be `ACTIVE` in C3-M1 yet; `activate-company` follows afterwards (FRAMEWORK 5.2).

1. Validate, otherwise 400 `validation` (`retryable: false`): `bpn` matches `^BPNL[0-9A-Z]{12}$`.
2. Check the row in `company` without changing anything: `bpn` differs: 409 `conflict` (`retryable: false`). Status `DISABLING` or `ENABLING`: 409 `invalid-state` (`retryable: true`).
3. Check the account and the binding without changing anything:
   - Read the account in the Portal IAM. Missing: 404 `user-not-found` (`retryable: false`).
   - Binding `ACTIVE` or `DISABLED` to another company: 409 `user-already-bound` (`retryable: false`).
   - No binding, the account has a staff role and the company is not `ospCompanyId`: 409 `staff-company-conflict` (`retryable: false`).
4. Row in `company`: if missing, create it with `bpn` and status `ACTIVE`. Status `DISABLED`: `ACTIVE` (return after a transfer); other `DISABLED` bindings stay and come back only with `enable-company-users` or are removed.
5. Binding of the account to this company:
   - `ACTIVE`: assign `COMPANY_ADMIN_ROLE` if missing (then event `UserRolesChanged`); align the attributes.
   - `DISABLED`: restore as in 8.10 step 3, plus assign `COMPANY_ADMIN_ROLE` if missing; event `UserAssignedToCompany`.
   - none: bind (6.12.1) with `[COMPANY_ADMIN_ROLE]`, route `ONBOARDING`.
6. Result: `companyId`, `userId`, `portalRoles`.

Event: `UserAssignedToCompany` with `via` `ONBOARDING` if the account was newly bound. If the step fails in 1 to 3, it changes nothing. A repetition finds the binding `ACTIVE` with `Company Admin` and returns the same result without an event.

### 8.9 `disable-company-users`

Input: `companyId`.

Preconditions: company deactivated (`CompanyDeactivated`, FRAMEWORK 5.4).

1. `companyId` is `ospCompanyId`: 409 `invalid-state` (`retryable: false`). This step does not deactivate the OSP company, otherwise the staff would lock itself out.
2. No row in `company`: result with an empty list, no event. Status `ENABLING`: 409 `invalid-state` (`retryable: true`). Status `DISABLED`: continue with 5.
3. Status `ACTIVE`: in one transaction status `DISABLING`; all open invitations of the company `REVOKED` with reason `COMPANY_DISABLED`, clear the names.
4. Per binding `ACTIVE` of the company:
   1. Read all role mappings of the account. Account missing: binding `REMOVED`, reason `ACCOUNT_DELETED`, event `UserRemovedFromCompany`; continue with the next binding.
   2. If `suspended_roles` is empty: store the company portal roles and the app roles on instance clients of this module there (own transaction). If it is already filled (repetition after an abort), it stays unchanged.
   3. Remove these roles (as in 6.12.3 steps 2 and 3), remove the attributes `{COMPANY_ID_ATTRIBUTE}` and `{BPN_ATTRIBUTE}`, end the sessions (`POST …/logout`). `Applicant` and staff roles stay; `enabled` stays `true`.
   4. In one transaction: binding `DISABLED`, entry in `audit_log`.
5. If the status was `DISABLING`: in one transaction status `DISABLED` and event `CompanyUsersDisabled`.
6. Result: `companyId`, `userIds` (all bindings of the company with status `DISABLED`).

The people can still sign in, but have only `Applicant` and no company in the token. Apps see them without a company and without app roles. With a mandate, one of them can apply for the reactivation (FRAMEWORK 5.5). Issued access tokens stay valid until they expire. The binding remains; so the account cannot be bound to another company unless the staff removes it (7.14). A repetition after an abort continues at status `DISABLING` and reports the event exactly once.

### 8.10 `enable-company-users`

Input: `companyId`.

Preconditions: company reactivated (`CompanyReactivated`, FRAMEWORK 5.5).

1. No row in `company`: result with an empty list, no event. Status `DISABLING`: 409 `invalid-state` (`retryable: true`). Status `ACTIVE` (e.g. because `assign-company-admin` ran first): restore `DISABLED` bindings as in step 3, then continue with 5, without an event.
2. Status `DISABLED`: status `ENABLING`.
3. Per binding `DISABLED` of the company: read the account (missing: as in 8.9 step 4.1); set the attributes `{COMPANY_ID_ATTRIBUTE}` and `{BPN_ATTRIBUTE}`; assign the roles from `suspended_roles`, app roles only for instance clients that are not yet `DISABLED`; in one transaction binding `ACTIVE`, clear `suspended_roles`, one entry `ADDED` in `role_assignment_log` per role, entry in `audit_log`.
4. If the status was `ENABLING`: in one transaction status `ACTIVE` and event `CompanyUsersEnabled`.
5. Result: `companyId`, `userIds` (all bindings of the company with status `ACTIVE`).

Afterwards the accounts have their portal roles and app roles again. Invitations revoked during the deactivation stay revoked.

## 9. Queries for other modules

### 9.1 `GET /internal/app-roles?appId=…&appId=…&lang=`

App roles of 1 to `QUERY_MAX_APPS` apps, read from the templates (one call per app, `briefRepresentation=false`). Per role, sorted by name: name, `description` in `lang` (default `en`; otherwise English, otherwise the first one) and all descriptions. Apps without a template come back with `templateExists: false` and without roles. The order of the apps follows the request. Callers may cache the result for a short time; the Portal IAM stays the leading source.

### 9.2 `GET /internal/app-instance-clients`

Filters: `appId`, `offerSubscriptionId`, `customerCompanyId`, `instanceKind`, `status` (may be given several times; default `PENDING`, `ACTIVE`). At least one of `appId` and `offerSubscriptionId` is required, otherwise 400 `validation`. Sorted by `created_at`. Reads only the module's own table.

Typical use by Subscriptions: `offerSubscriptionId` → the client ID for the provider.

### 9.3 `GET /internal/app-instance-clients/{id}`

One instance client, 404 `not-found` if unknown.

### 9.4 `GET /internal/company-users?companyId=&portalRole=&status=`

People of a company, e.g. for notifications. `companyId` is required, otherwise 400 `validation`. `status` may be given several times, default `ACTIVE`. With `portalRole` (from `COMPANY_PORTAL_ROLES`) only accounts with this role: the module reads the accounts with the role in the Portal IAM and intersects them with the bindings. Response: list of `{ userId, status }`, without names.

## 10. Events

Written to the outbox in the same transaction as the change in the module's table. No event contains a secret, an invitation code or a name.

| Event | When | Content |
|---|---|---|
| `AppClientTemplateCreated` | 8.1 created the template | `appId`, `iamClientId`, `roleNames` |
| `AppInstanceClientCreated` | 8.2 created an instance client | `appInstanceClientId`, `appId`, `instanceKind`, `offerSubscriptionId`, `customerCompanyId`, `iamClientId`, `instanceUrl` |
| `AppInstanceClientEnabled` | 8.3 | `appInstanceClientId`, `appId`, `offerSubscriptionId`, `iamClientId` |
| `AppInstanceClientDisabled` | 8.4 | `appInstanceClientId`, `appId`, `offerSubscriptionId`, `iamClientId` |
| `AppInstanceUrlUpdated` | 8.5 | `appInstanceClientId`, `appId`, `offerSubscriptionId`, `instanceUrl` |
| `AppRolesAdded` | 8.1 (roles added to an existing template) or 8.6 | `appId`, `roleNames` |
| `AppRoleDescriptionsUpdated` | 8.7 | `appId`, `roleNames` |
| `UserAssignedToCompany` | an account was bound to a company (8.8, 7.10) | `userId`, `companyId`, `via` (`ONBOARDING`, `INVITATION`), `portalRoles` |
| `UserRemovedFromCompany` | a binding is `REMOVED` (7.4, 7.14, 8.9, 8.10) | `userId`, `companyId`, `reason` (`REMOVED_BY_COMPANY`, `REMOVED_BY_OSP`, `ACCOUNT_DELETED`) |
| `UserRolesChanged` | company portal roles (7.3, 7.13, 8.8) or app roles (7.5) have changed | `userId`, `companyId`, `appId` (only for app roles), `addedRoles`, `removedRoles` |
| `CompanyUsersDisabled` | 8.9 finished | `companyId`, `userIds` |
| `CompanyUsersEnabled` | 8.10 finished | `companyId`, `userIds` |
| `OspStaffRolesChanged` | staff roles have changed (7.10, 7.16) | `userId`, `addedRoles`, `removedRoles` |

Every event also carries `eventId` (uuid), `eventType` (the name from the table) and `occurredAt`. The schemas are in [openapi.yaml](openapi.yaml) under `components/schemas` (names start with `Event`).

## 11. Calls to other modules and external systems

### 11.1 Portal IAM (Keycloak Admin REST API)

All calls with the token of the module's own service account (`IAM_ADMIN_CLIENT_ID`), realm `IAM_REALM`. Paths are relative to `/admin/realms/{realm}`. Role names in paths URL-encoded (`Company%20Admin`).

| Purpose | Call |
|---|---|
| Find a client (also `portal-core` at start) | `GET …/clients?clientId={clientId}` |
| Create a client | `POST …/clients` with the settings from 6.3; the internal ID is in the `Location` header |
| Read / change a client (enable, disable, URL) | `GET` / `PUT …/clients/{uuid}`; for `PUT` send the read representation with the changed fields |
| Protocol mappers | `GET` / `POST …/clients/{uuid}/protocol-mappers/models` |
| Roles of a client | `GET …/clients/{uuid}/roles?briefRepresentation=false` |
| Read one role (ID for assignments) | `GET …/clients/{uuid}/roles/{roleName}` |
| Create a role | `POST …/clients/{uuid}/roles` with `name`, `description` |
| Set description and attributes | `PUT …/clients/{uuid}/roles/{roleName}` with `name`, `description`, `attributes`; always after creating a role |
| Accounts with a role | `GET …/clients/{portalCoreUuid}/roles/{roleName}/users?first=&max=`, page by page until an empty page |
| Read / change an account (attributes, `enabled`) | `GET` / `PUT …/users/{userId}`; for `PUT` send the read representation with the changed fields; removing an attribute = empty list |
| End the sessions of an account | `POST …/users/{userId}/logout` |
| All role mappings of an account | `GET …/users/{userId}/role-mappings` (`clientMappings` per client ID) |
| Read / assign / remove assignments on a client | `GET` / `POST` / `DELETE …/users/{userId}/role-mappings/clients/{clientUuid}`; for `POST` and `DELETE` a list of roles with `id` and `name` |

Attributes are sent as lists, e.g. `"attributes": {"description.de": ["Lesen"], "description.en": ["Read"]}` or `{"company_id": ["…"], "bpn": ["BPNL…"]}`. The module creates no accounts and deletes none.

### 11.2 Neighbouring modules

In the shared process, the module calls the public interface of the neighbour; the HTTP form applies when the neighbour runs as a separate service. If the connection is empty, the module uses the built-in stand-in. So it can be run and tested on its own. If a neighbour does not answer as expected: 502 `upstream-error`.

| Neighbour (variable) | Call | Response | Built-in stand-in |
|---|---|---|---|
| Subscriptions (`SUBSCRIPTION_URL`) | active subscriptions of a company to an app: `GET /internal/subscriptions?customerCompanyId=&offerId=&status=ACTIVE` (base as in the spec of Subscriptions) | list of `{ "offerSubscriptionId", "offerId", "customerCompanyId", "status" }` | reads `stub-subscriptions.json` |
| C3-M1 Company Profile (`COMPANY_URL`) | only for a company without a row in `company` (6.6): `GET /api/c3-m1/internal/companies/{companyId}` | `{ "id", "status", "bpn", … }` with status `REQUESTED`, `ACTIVE`, `REJECTED` or `INACTIVE`; 404 if unknown | reads `stub-companies.json` |

Apart from these, the module calls no modules. The BPN of a company usually comes with the step `assign-company-admin`; the steps 8.9 and 8.10 report the status of the company.

### 11.3 Installation settings

At start the module reads `ospCompanyId` from `SETTINGS_FILE` (FRAMEWORK 4.8, proposal OP-36). If the value is missing: no OSP company; staff only without a company (6.7 limit 4), and 8.9 rejects no company.

## 12. Errors

Errors come as Problem Details (RFC 9457). `type` is `urn:c2-m2:problem:<name>`. For steps the error carries `retryable` (FRAMEWORK 4.4).

| Name | HTTP | When | `retryable` for steps |
|---|---|---|---|
| `validation` | 400 | invalid input | `false` |
| `unknown-app-role` | 400 | the app role to assign does not exist on the template | – |
| `unauthorized` | 401 | token missing or invalid (HTTP only) | – |
| `forbidden` | 403 | permission missing (HTTP only) | – |
| `name-mismatch` | 403 | names in the token do not match the invitation | – |
| `not-found` | 404 | instance client, template, binding or invitation unknown, or the path names another company | `false` |
| `role-not-found` | 404 | role does not exist on the template (8.7) | `false` |
| `invitation-not-found` | 404 | no invitation for this code | – |
| `user-not-found` | 404 | account unknown in the Portal IAM | `false` |
| `template-missing` | 409 | no template for the app | `false` |
| `instance-exists` | 409 | an instance client for this subscription or this single-instance app exists with different data | `false` |
| `invalid-state` | 409 | the status does not allow the step or the operation | `false`; `true` for `DISABLING` or `ENABLING` (8.8 to 8.10) |
| `conflict` | 409 | the BPN of the input differs from the stored one (8.8) | `false` |
| `invitation-not-open` | 409 | invitation accepted, revoked or expired | – |
| `user-already-bound` | 409 | the account is already bound to a company | `false` |
| `staff-company-conflict` | 409 | staff and company do not fit together (6.7 limit 4) | `false` |
| `last-company-admin` | 409 | last Company Admin of the company | – |
| `last-osp-admin` | 409 | last OSP Admin of the installation | – |
| `too-many-invitations` | 409 | too many open invitations | – |
| `company-not-active` | 409 | company without a row in `company` or not `ACTIVE` | – |
| `no-active-subscription` | 409 | the company has no active subscription to the app with an active instance client | – |
| `iam-error` | 502 | the Portal IAM did not answer as expected | `true` |
| `upstream-error` | 502 | a neighbouring module did not answer as expected | `true` |

## 13. Acceptance

The module is done when these cases work against the test Portal IAM (section 14) and with the built-in stand-ins for Subscriptions and C3-M1. People call with test tokens as in 14.3.

App clients and app roles:

1. `create-client-template` with two roles in `de` and `en`: the client `app-template-{appId}` exists, is disabled and has both roles with the attributes `description.de`, `description.en` and the English text in `description`.
2. `create-instance-client` (`SUBSCRIPTION`): a disabled public client `app{n}` with `baseUrl`, redirect `{URL}/*`, audience mapper, both roles including attributes and the client scopes from `DEFAULT_CLIENT_SCOPES`, without `profile` and `email`. Status `PENDING`.
3. `enable-instance-client`: the client is enabled, status `ACTIVE`.
4. `add-app-roles` with one new and one existing role: the new role appears on the template and on every instance client, including disabled ones; the existing role is listed under `skippedRoles` and keeps its descriptions.
5. `update-role-descriptions`: changed attributes on the template and on every instance client; an unknown role gives 404 and changes nothing.
6. Every step run twice with the same input: no second client, no second role, no second mapper, the same result, only one event.
7. Interrupted runs (simulated): a client created in the Portal IAM before `iam_client_uuid` was stored is completed by a repetition with the same client ID. A role without attributes is completed by a repetition of `add-app-roles`.
8. `create-instance-client` before the template: 409 `template-missing`. A second `SINGLE` with a different URL: 409 `instance-exists`.
9. `disable-instance-client`: the client is disabled, status `DISABLED`; `enable-instance-client` afterwards gives 409.
10. `/internal/app-roles` for two apps, one without a template: roles with descriptions in the requested language for the first, `templateExists: false` for the second.
11. `/internal/app-instance-clients` without `appId` and `offerSubscriptionId`: 400.
12. `directAccessGrantsEnabled` is `false` on every client the module created.

Onboarding and binding:

13. `assign-company-admin` for Erika Mustermann (without a company) and Beispiel Maschinenbau GmbH: in the Portal IAM, the account has `company_id` and `bpn` of the company, the role `Company Admin` and still `Applicant`. Binding `ACTIVE`, route `ONBOARDING`, event `UserAssignedToCompany`. A repetition returns the same result without an event.
14. `assign-company-admin` for Hans Beispiel, who is bound to Beispiel Logistik GmbH: 409 `user-already-bound` with `retryable: false`; nothing changes. With a different BPN for a known company: 409 `conflict`.
15. After case 13, a new token of Erika contains `company_id`, `bpn` and the permissions of `Company Admin`.

Invitations:

16. Erika invites Max Mustermann with `Member`: 201 with `code` (43 characters). The database holds only the hash. The list of invitations shows it as `OPEN` with names, without the code. No log, event or audit log entry contains the code.
17. Max (without a company) accepts: binding `ACTIVE`, route `INVITATION`, `company_id`, `bpn` and `Member` in the Portal IAM; invitation `ACCEPTED` without names; event `UserAssignedToCompany`. A token after a refresh contains `company_id`. Max accepts the same code again: 200 with the same result, no event.
18. Lieschen Müller (without a company) redeems an open code for Max: 403 `name-mismatch`; after the fifth attempt the invitation is `REVOKED` with reason `TOO_MANY_ATTEMPTS`. With `INVITATION_NAME_CHECK=false` the acceptance succeeds.
19. Expired code: 409 `invitation-not-open`; the invitation is `EXPIRED` without names. Unknown code: 404 `invitation-not-found`. Code in the wrong format: 400.
20. Hans Beispiel (bound to Beispiel Logistik GmbH) accepts an invitation from Erika: 409 `user-already-bound`.
21. Erika revokes an open invitation: 204, status `REVOKED`; acceptance afterwards: 409. The 51st open invitation: 409 `too-many-invitations`.

Portal roles, removal, scopes:

22. Erika sets Max to `IT Admin`, `Member`: assignments in the Portal IAM changed, one entry per change in `role_assignment_log` with Erika's `actor_sub`, event `UserRolesChanged` with `addedRoles: ["IT Admin"]`. The same call again: no change, no event.
23. Erika removes `Company Admin` from herself or removes herself: 409 `last-company-admin`. If Max also has `Company Admin`, it succeeds.
24. Erika removes Max: `company_id`, `bpn`, the company portal roles and the app roles are gone from the account, `Applicant` stays, the sessions are ended. Binding `REMOVED` without names, event `UserRemovedFromCompany`. A second `DELETE`: 204 without an event. Max then accepts a new invitation with `Member` and is bound again.
25. Erika calls `/companies/{Beispiel Logistik}/users`: 404. Max with `Member` calls `POST …/invitations`: 403. A token with `view_company_users` but without `company_id` calls `/companies/{Beispiel Maschinenbau}/users`: 404.

App roles:

26. `PUT …/users/{Max}/apps/{appId}/roles` with `["Viewer"]` with an active subscription (stand-in): the role is assigned on the instance client of the subscription; entry in `role_assignment_log`; event `UserRolesChanged` with `appId`. The same call again: no change, no event.
27. Then `["Editor"]`: `Viewer` removed, `Editor` assigned, both logged. An empty list removes all app roles of the app. Unknown role: 400 `unknown-app-role`. Company without an active subscription: 409 `no-active-subscription`, and 7.6 shows `assignable: false`.
28. Sign-in to the app: Max signs in through the instance client from case 3 and the stand-in for the EBW-Connector. No further account is created. The access token contains `sub` = `userId`, `company_id`, `bpn`, the client ID in `aud`, `Viewer` under `resource_access.{clientId}.roles` and none of `given_name`, `family_name`, `name` or `email`.

Deactivation and reactivation:

29. `disable-company-users` for Beispiel Maschinenbau: all bindings `DISABLED` with a filled `suspended_roles`; the accounts stay enabled but have neither `company_id` nor `bpn` nor company portal roles or app roles, only `Applicant`; sessions ended; open invitations `REVOKED` with reason `COMPANY_DISABLED`, company `DISABLED`, event `CompanyUsersDisabled` exactly once. A new token of Max contains no company and no app role. Repetition: the same result, no event, `suspended_roles` unchanged.
30. Abort after the first person (simulated): the repetition continues at status `DISABLING` and reports the event exactly once.
31. `enable-company-users`: accounts again with `company_id`, `bpn` and the saved portal roles and app roles, `suspended_roles` empty, bindings `ACTIVE`, event `CompanyUsersEnabled`. A new token contains the same permissions as before the deactivation. If `assign-company-admin` for Erika runs first: her binding is `ACTIVE` with `Company Admin`; `enable-company-users` then restores the other bindings, without the event `CompanyUsersEnabled`.
32. `disable-company-users` for the OSP company: 409 `invalid-state` with `retryable: false`.

OSP staff:

33. Bootstrap: start with `BOOTSTRAP_ADMIN_FILE` and without an OSP Admin: the invitation `STAFF` exists. Otto Normalverbraucher accepts it with the test code and has `OSP Admin`; event `OspStaffRolesChanged`. Restart: no second invitation.
34. Otto removes `OSP Admin` from himself: 409 `last-osp-admin`. Otto invites Lieschen as `OSP Reviewer`; she accepts. A staff invitation for Hans Beispiel (bound to Beispiel Logistik GmbH, not the OSP company) fails on acceptance with 409 `staff-company-conflict`. `PUT /osp/staff/{Hans}/roles` with `OSP Reviewer`: also 409.
35. Lieschen (`OSP Reviewer`) sees `GET /osp/users` and `GET /osp/staff`; `PUT /osp/companies/…/portal-roles` gives her 403. Otto removes the last Company Admin of a company through 7.14: allowed. Otto removes a person with binding `DISABLED`: afterwards the account is enabled, without a company.
36. Otto (OSP Admin, without a company) invites himself through 7.20 as `Company Admin` to the OSP company: the module creates the row in `company` with the BPN from the stand-in for C3-M1 (`BPNL00000000OSPA`). Otto accepts; afterwards he is Company Admin of the OSP company and still OSP Admin. The same invitation for Ohne Zustimmung GmbH (`REQUESTED`): 409 `company-not-active`.
37. `/internal/company-users?companyId=…&portalRole=Company%20Admin` returns exactly the Company Admins with binding `ACTIVE`, without names.

General:

38. No event, no step result and no audit log entry contains a name or an invitation code. Every changing operation has an entry in `audit_log` with `actor_sub`.

## 14. Test data

### 14.1 Test Portal IAM

The test Portal IAM needs:

- the realm `IAM_REALM` with the client scopes from `DEFAULT_CLIENT_SCOPES`; in the client scope `catena` one mapper each from the attribute `company_id` and `bpn` to the claim of the same name;
- a user profile that allows the attributes `company_id` and `bpn` and lets only administrators change them;
- the client `portal-core` with the portal roles `Applicant`, `Member`, `IT Admin`, `Company Admin`, `OSP Reviewer`, `OSP Admin` (they may be empty for the tests of this module); `Applicant` is assigned by the identity provider at the first login (C2-M5 5.5), not a default role of the realm;
- a service account for the module with the permissions listed under `IAM_ADMIN_CLIENT_ID`;
- a stand-in for the EBW-Connector: a second realm of the same Keycloak as identity provider (OIDC) with the test persons from 14.2; as a stand-in for an external system it may have test passwords; first broker login as in C2-M5 (create the account, link it);
- the test persons from 14.2 as accounts with a fixed ID, with `firstName`, `lastName`, `Applicant` and the link to the stand-in, without credentials.

### 14.2 Test values

| Object | Value |
|---|---|
| OSP company of OSP A (`ospCompanyId`, FRAMEWORK 7a) | `11111111-1111-4111-8111-111111111111`, BPN `BPNL00000000OSPA` (seed data in C3-M1) |
| Beispiel Maschinenbau GmbH (`companyId`) | `c0000001-0000-4000-8000-000000000001`, BPN `BPNLIUK2REMLK3F4` (as assigned by the stand-in in C3-M2 for `EBWOID-DE-0000000001`) |
| Beispiel Logistik GmbH | `c0000002-0000-4000-8000-000000000002`, BPN `BPNL00000000U2LG` |
| Ohne Zustimmung GmbH | `c0000003-0000-4000-8000-000000000003`; `REQUESTED` in the stand-in for C3-M1, without a BPN |
| Erika Mustermann (`userId`) | `5a1c0001-0000-4000-8000-000000000001`; applicant of Beispiel Maschinenbau, becomes Company Admin (case 13) |
| Max Mustermann | `5a1c0002-0000-4000-8000-000000000002`; without a company, invited as `Member` |
| Hans Beispiel | `5a1c0003-0000-4000-8000-000000000003`; Company Admin of Beispiel Logistik GmbH beforehand (in the test through `assign-company-admin`) |
| Lieschen Müller | `5a1c0004-0000-4000-8000-000000000004`; without a company |
| Otto Normalverbraucher | `5a1c0012-0000-4000-8000-000000000012`; first OSP Admin through the bootstrap, then Company Admin of the OSP company (case 36) |
| Bootstrap invitation | ID `b0000000-0000-4000-8000-000000000001`; test code `c39vGXcoIJyPomVOlPQyk4WrA0LojyC21FQuF6S9hOc` (public, for tests only); hash as in 6.9 |
| Demo App | `appId` `6f1c2a4e-8b0d-4c55-9a3e-2d7b1f0c9e11`, roles `Viewer` (`de`: „Darf Daten lesen“, `en`: “May read data”) and `Editor` (`de`: „Darf Daten ändern“, `en`: “May change data”), URL of the instance `https://demo-app.example.org` |
| Subscription of Beispiel Maschinenbau to the Demo App | `offerSubscriptionId` `0b5d7e2a-3c4f-4a1b-9e8d-7f6a5b4c3d21` |

People and companies are fictional. The IDs of the people and companies are the same as in the test data of C3-M1 (P1 to P4, S2, U1 to U3).

Files:

- `config/installation.json` with `ospCompanyId` `11111111-1111-4111-8111-111111111111`;
- `config/bootstrap-admin.json` as in 6.9;
- `stub-subscriptions.json`: one active subscription of Beispiel Maschinenbau to the Demo App with the ID above, which the instance client from case 2 carries; none for Beispiel Logistik GmbH.
- `stub-companies.json` (stand-in for C3-M1): the OSP company `ACTIVE` with BPN `BPNL00000000OSPA`, Beispiel Maschinenbau and Beispiel Logistik `ACTIVE` with the BPNs above, Ohne Zustimmung GmbH `REQUESTED` without a BPN.

### 14.3 Tokens of the test persons

- For calls via HTTP in the tests of this module, the test signs tokens itself with a test key; the token validation library gets its JWKS in the test. Claims as in FRAMEWORK 4.2: `sub` = `userId`, `company_id` and `bpn` when bound, `given_name`, `family_name`, permissions as in the proposal in 6.11.
- End-to-end (cases 15, 17, 28, 29, 31) with real tokens of the test Portal IAM after sign-in through the stand-in for the EBW-Connector.

## 15. Changes from version 0.1

- **New name and wider purpose:** "Users and App Access" instead of "App Access". The module now manages the accounts of all people: binding to a company, portal roles, app roles, deactivation. App users are the same accounts as for the portal.
- **Accounts are created at the first login** (C2-M5), no longer by the module. App users created in advance with an alias of the identity provider and an identifier at the PSP are dropped, as are the table `app_user`, `POST /app-users`, `pspSubject`, `signInAlias`.
- **New:** tables `company`, `company_user`, `invitation`; invitations with a code (only the hash stored), name check and limits, by the Company Admin or the OSP staff; OSP staff with bootstrap; limits "last Company Admin" and "last OSP Admin".
- **New steps:** `assign-company-admin`, `disable-company-users`, `enable-company-users`. Dropped: `remove-app-users` (end of a delegation).
- **New events:** `UserAssignedToCompany`, `UserRemovedFromCompany`, `UserRolesChanged`, `CompanyUsersDisabled`, `CompanyUsersEnabled`, `OspStaffRolesChanged`. Dropped: `AppUserCreated`, `AppUserRolesChanged`, `AppUserRemoved`; the app roles of a person are now reported by `UserRolesChanged` with `appId`.
- **New query:** `/internal/company-users`.
- **Callers and permissions:** people with their own token and a company context (scopes `company`, `osp`, `own`) instead of PSPs with delegation. `X-On-Behalf-Of`, `X-Acting-Person`, `x-delegation-scope`, `psp_id` and the error `not-delegated` are dropped. Paths carry the company (`/companies/{companyId}/…`); tags `Company`, `OSP`, `Queries`, `Steps`. New permissions `view_company_users`, `manage_company_users`, `accept_invitation`, `view_users`, `manage_users`, `manage_osp_staff`; `view_app_users` and `manage_app_users` are dropped.
- **Neighbours:** C2-M1 and C2-M4 are dropped. The module queries C3-M1 only for companies without a row of their own, e.g. the OSP company (6.6); plus Subscriptions as before. The errors `company-not-ready`, `sign-in-service-missing` and `app-user-exists` are dropped.
- **Instance clients without `profile` and `email`:** apps get no names (FRAMEWORK 4.6), even though accounts now carry names from the PID. Optional client scopes are empty. Sign-in to apps no longer needs `kc_idp_hint`.
- **Audit log:** `actor_sub` from the token instead of PSP and `X-Acting-Person`; steps also write to the `audit_log`.
- Errors carry `retryable` for steps.
