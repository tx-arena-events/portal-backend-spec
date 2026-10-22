# C2-M6 App Clients and Roles

**Version:** 0.1, as of 2026-10-22
**Capability:** C2 Identity and Access
**Kind:** Hackathon (to be built at the hackathon)
**Files:** this README (behaviour), [openapi.yaml](openapi.yaml) (interface)

This README and the OpenAPI description are sufficient to build the module. Modules from other parts of the spec are referred to by name.

## 1. Purpose

The module manages the clients of apps in the Portal IAM and the app roles on them. For every app it keeps a **client template** that holds the app roles with their descriptions. For every instance of the app it creates an **instance client** from the template. It is the only module that writes app clients and app roles to the Portal IAM.

The module has no operations for the UI. Its callers:

| Who | What for |
|---|---|
| Process Orchestration | steps: create the template when an app is approved, create and enable instance clients, add app roles, change descriptions, change the URL of an instance, disable an instance client |
| Offers | read the app roles of one or more apps, e.g. for the marketplace |
| Users (C2-M2) | read the app roles and the instance clients on which a company's users receive them |
| Subscriptions | read the instance client of a subscription, e.g. to pass its client ID to the provider |

## 2. Terms

| Term | Meaning |
|---|---|
| App | an offer of kind app in the marketplace (Offers). Services have no clients and no app roles. |
| Instance | a running installation of an app under a URL |
| Single-instance app | one instance for all customers; the provider sets its URL during the release |
| Multi-instance app | one instance per subscription; the provider gives its URL when setting up the subscription |
| App role | a role defined by the provider of the app, e.g. `Viewer`. Users receive it on the instance client of their company's subscription. |
| Portal role | a role defined by the portal, e.g. `Company Admin` or `Connector Management`. Not managed here (C2-M5 Portal IAM). |
| Client template | one client per app in the Portal IAM. It holds the app roles with their descriptions and is disabled; nobody signs in with it. |
| Instance client | the client of exactly one instance. Users sign in to the app with it. It carries a copy of every app role of the template. |

## 3. Not part of this module

| What | Where |
|---|---|
| The app and its draft, including the planned app roles during the release | Offers |
| Subscriptions and their status | Subscriptions |
| Assigning app roles to users | C2-M2 Users |
| Technical users and their profiles (profiles refer to portal roles only) | C2-M3 Technical Users |
| Portal roles and the clients of the portal itself | C2-M5 Portal IAM (configuration) |
| Notifications, e.g. to the provider when app roles were added | Notifications, started by the flow |
| Order of the steps, retries | Process Orchestration |

## 4. Technical frame

- Python with FastAPI.
- Own database schema `c2_m6` in PostgreSQL. The module reads and writes only there. Other modules read through the interface.
- Process Orchestration and other modules call the module through its public interface **in the same process**. [openapi.yaml](openapi.yaml) describes the same contract as HTTP paths; they apply when the module runs as a service of its own.
- When called via HTTP, every call carries a token of the Portal IAM, checked by the shared token validation library (permissions in 6.6).
- Events go out through the outbox of the module skeleton: written in the same transaction as the change, delivered afterwards (section 9).
- The module never handles secrets. Instance clients are public clients.

### Configuration

| Variable | Meaning | Example |
|---|---|---|
| `DATABASE_URL` | PostgreSQL, schema `c2_m6` | |
| `IAM_URL` | base URL of the Portal IAM | `https://iam.example.org/auth` |
| `IAM_REALM` | realm | `CX-Central` |
| `IAM_ADMIN_CLIENT_ID`, `IAM_ADMIN_CLIENT_SECRET` | the module's own service account for the admin API; needs the realm permissions `manage-clients`, `view-clients`, `query-clients` | |
| `CLIENT_ID_PREFIX` | prefix of instance client IDs | `app` |
| `TEMPLATE_CLIENT_ID_PREFIX` | prefix of template client IDs | `app-template-` |
| `DEFAULT_CLIENT_SCOPES` | default client scopes of instance clients, comma-separated | `catena,web-origins,roles,profile,email` |
| `OPTIONAL_CLIENT_SCOPES` | optional client scopes of instance clients | `address,phone,offline_access,microprofile-jwt` |
| `LANGUAGES` | languages allowed for role descriptions | `de,en` |
| `QUERY_MAX_APPS` | maximum number of apps in one roles query | `50` |

## 5. Data

### Table `app_client_template`

| Field | Type | Rule |
|---|---|---|
| `app_id` | uuid | primary key |
| `provider_company_id` | uuid | required |
| `app_name` | text | 1 to 255 characters; used as client name |
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

At most one row per app with `instance_kind` `SINGLE` and a status other than `DISABLED` (partial unique index).

**Not stored:** app roles and their descriptions (only in the Portal IAM), the status of subscriptions (Subscriptions), app data apart from the name (Offers), who has which app role (Portal IAM, managed by C2-M2).

## 6. Rules

### 6.1 Client IDs

- Template: `TEMPLATE_CLIENT_ID_PREFIX` + app ID. It follows from the app ID, so a repeated step finds it again.
- Instance client: `CLIENT_ID_PREFIX` + next value of the sequence `client_number`. The number is taken and stored in `app_instance_client` **before** the client is created in the Portal IAM, so a repeated step uses the same ID.

### 6.2 App roles

- Name: 1 to 255 characters, unique within an app (case-sensitive, as in Keycloak).
- Descriptions: at least one; language from `LANGUAGES`, at most one per language; text 1 to 255 characters.
- In the Portal IAM, a role carries its descriptions as attributes `description.{lang}`, e.g. `description.de`, `description.en`. The field `description` of the role holds the English text, or the first one given if there is none.
- The template is the leading source. Every instance client of the app carries the same roles with the same name, description and attributes.
- Roles are only added, never removed or renamed.

### 6.3 Client settings

| Setting | Template | Instance client |
|---|---|---|
| `clientId` | see 6.1 | see 6.1 |
| `name` | app name | app name |
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
| `baseUrl` | – | instance URL |
| `redirectUris` | – | `{instance URL}/*`, without a double slash |
| `webOrigins` | – | `+` (origins of the redirect URIs) |
| attributes | – | `use.refresh.tokens: true`, `backchannel.logout.session.required: true` |
| protocol mapper | – | audience mapper (6.3.1) |
| client roles | all app roles | copy of all app roles of the template |

#### 6.3.1 Audience mapper

Name `{clientId}-mapper`, `protocol` `openid-connect`, `protocolMapper` `oidc-audience-mapper`, config `included.client.audience`: client ID, `access.token.claim`: `true`, `id.token.claim`: `false`. Access tokens for the instance client then contain its client ID in `aud`.

### 6.4 Status of an instance client

| From | Step | To |
|---|---|---|
| – | create instance client | `PENDING` (client disabled) |
| `PENDING` | enable | `ACTIVE` (client enabled) |
| `PENDING`, `ACTIVE` | disable | `DISABLED` (client disabled) |

`DISABLED` is final. A new subscription gets a new instance client.

### 6.5 Repetition and concurrency

Every step can be repeated with the same input and then changes nothing further:

- It looks up its object by a key that follows from the input: the app ID for the template, the offer subscription ID or (for `SINGLE`) the app ID for an instance client, the instance client ID for the other steps.
- In the Portal IAM it creates only what is missing and sets only what differs.
- It returns the same result as the first time.

So Process Orchestration can simply run a failed step again. `processId` and `stepRunId` serve only for logging. A step publishes an event only when it changed something.

Every step holds a PostgreSQL advisory lock on the app ID while it runs, so steps for the same app never run at the same time. This keeps the roles of the template and the instance clients in line, e.g. when a subscription is set up while roles are added.

### 6.6 Permissions (only when called via HTTP)

| Operation | Permission |
|---|---|
| all paths under `/internal` | `view_app_clients_internal` |
| all paths under `/steps` | `execute_app_client_steps` |

## 7. Steps

Common rules:
- Every step takes `processId` and `stepRunId`.
- A step is finished when it returns; the result has `state` `DONE`. No step runs longer than its call.
- On an error the step raises a problem (section 12) and leaves what it already created in place; running it again completes it (6.5).

### 7.1 `create-client-template`

When an app is approved. Input: `appId`, `providerCompanyId`, `appName`, `roles` (at least one; name and descriptions).

1. Validate as in 6.2, otherwise 400.
2. Row in `app_client_template` exists: continue with it. Otherwise insert it.
3. In the Portal IAM:
   1. Find the client by client ID. Missing: create it with the template settings (6.3).
   2. For every role of the input: missing on the client, create it; present, set description and attributes if they differ.
4. Store `iam_client_uuid`. If it was empty before: event `AppClientTemplateCreated`, in the same transaction. Otherwise, if roles were created: event `AppRolesAdded` with their names. Aligning existing roles publishes no event.
5. Result: `appId`, `iamClientId`, names of all app roles.

A role on the template that is not in the input stays. To add roles later, use 7.6.

### 7.2 `create-instance-client`

Input: `appId`, `instanceKind`, `instanceUrl`; for `SUBSCRIPTION` also `offerSubscriptionId` and `customerCompanyId`.

1. Validate, otherwise 400: URL as in section 5; for `SUBSCRIPTION` both IDs given, for `SINGLE` none of them.
2. A template for `appId` with `iam_client_uuid` must exist, otherwise 409 `template-missing`.
3. Find the row: for `SUBSCRIPTION` by `offerSubscriptionId`, for `SINGLE` by `appId` with status not `DISABLED`.
   - Found with status `DISABLED` (only possible for `SUBSCRIPTION`): 409 `invalid-state`.
   - Found with a different `appId`, `customerCompanyId` or `instanceUrl`: 409 `instance-exists`.
   - Not found: take the next number, insert the row with status `PENDING`.
4. In the Portal IAM:
   1. Find the client by client ID. Missing: create it, disabled, with the instance settings (6.3).
   2. Audience mapper missing: create it.
   3. Read the roles of the template (`briefRepresentation=false`). Every role missing on the instance client: create it with name, description and attributes; present: align description and attributes.
5. Store `iam_client_uuid`. If it was empty before: event `AppInstanceClientCreated`, in the same transaction.
6. Result: the instance client (`AppInstanceClient`).

### 7.3 `enable-instance-client`

Input: `appInstanceClientId`. Status `PENDING` without `iam_client_uuid` (creation not finished): 409 `invalid-state`. Status `PENDING`: enable the client in the Portal IAM, status `ACTIVE`, event `AppInstanceClientEnabled`. Status `ACTIVE`: nothing to do. Status `DISABLED`: 409 `invalid-state`. Result: the instance client.

### 7.4 `disable-instance-client`

Input: `appInstanceClientId`. Status `PENDING` or `ACTIVE`: disable the client in the Portal IAM, status `DISABLED`, event `AppInstanceClientDisabled`. Status `DISABLED`: nothing to do. Without `iam_client_uuid` only the status changes; a client that may exist from the interrupted creation is disabled anyway. The client and the role assignments of users stay in the Portal IAM. Result: the instance client.

### 7.5 `update-instance-url`

Input: `appInstanceClientId`, `instanceUrl`. Status `DISABLED` or no `iam_client_uuid`: 409 `invalid-state`. Otherwise set `baseUrl` and `redirectUris` (`{URL}/*`) of the client and `instance_url` of the row, event `AppInstanceUrlUpdated`. Same URL: nothing to do. Result: the instance client.

### 7.6 `add-app-roles`

Input: `appId`, `roles` (at least one; name and descriptions).

1. Validate as in 6.2, otherwise 400. A template with stored `iam_client_uuid` must exist, otherwise 409 `template-missing`.
2. Roles that already exist on the template with at least one attribute `description.{lang}` are skipped; their descriptions are not changed (use 7.7). A role on the template without such an attribute is left over from an interrupted run; it is completed with the given descriptions and counts as added.
3. Create the remaining roles on the template (create, then set description and attributes, section 10).
4. Every instance client of the app with a stored `iam_client_uuid`, in every status: create every role of the template that is missing there; give the roles added in this step the description and attributes of the template. Instance clients without `iam_client_uuid` are skipped; their step 7.2, when run again, copies all roles.
5. Event `AppRolesAdded` with the added role names, if there are any.
6. Result: `appId`, `addedRoles`, `skippedRoles`.

### 7.7 `update-role-descriptions`

Input: `appId`, `roles` (name and the complete new list of descriptions).

1. Validate as in 6.2, otherwise 400. A template with stored `iam_client_uuid` must exist, otherwise 409 `template-missing`. Every role must exist on the template, otherwise 404 `role-not-found`; nothing is changed then.
2. For every role: set the attributes `description.{lang}` to exactly the given descriptions (remove those of other languages) and the field `description` (6.2); on the template, then on every instance client of the app with a stored `iam_client_uuid`. If the role is missing on an instance client, create it there.
3. Event `AppRoleDescriptionsUpdated` with the changed role names, if there are any.
4. Result: `appId`, `updatedRoles`.

## 8. Queries

### 8.1 `GET /internal/app-roles?appId=…&appId=…&lang=`

App roles of 1 to `QUERY_MAX_APPS` apps, read from the templates (one call per app, `briefRepresentation=false`). Per role, sorted by name: name, `description` in `lang` (default `en`; otherwise English, otherwise the first one), and all descriptions. Apps without template come back with `templateExists: false` and no roles. The order of the apps follows the request. Callers may keep the result for a short time; the Portal IAM stays the leading source.

### 8.2 `GET /internal/app-instance-clients`

Filters: `appId`, `offerSubscriptionId`, `customerCompanyId`, `instanceKind`, `status` (may be given several times; default `PENDING`, `ACTIVE`). At least one of `appId` and `offerSubscriptionId` is required, otherwise 400. Sorted by `created_at`. Reads only the module's own table.

Typical use:
- Subscriptions: `offerSubscriptionId` → the client ID for the provider.
- C2-M2 for a multi-instance app: `appId` + `customerCompanyId` + `status=ACTIVE` → the clients on which the company's users receive app roles.
- C2-M2 for a single-instance app: `appId` + `instanceKind=SINGLE` + `status=ACTIVE`. Whether the company has an active subscription, C2-M2 asks Subscriptions.

### 8.3 `GET /internal/app-instance-clients/{id}`

One instance client, 404 if unknown.

## 9. Events

Written to the outbox in the same transaction as the change in the module's table. None contains a secret.

| Event | When | Content |
|---|---|---|
| `AppClientTemplateCreated` | 7.1 created the template | `appId`, `iamClientId`, `roleNames` |
| `AppInstanceClientCreated` | 7.2 created an instance client | `appInstanceClientId`, `appId`, `instanceKind`, `offerSubscriptionId`, `customerCompanyId`, `iamClientId`, `instanceUrl` |
| `AppInstanceClientEnabled` | 7.3 | `appInstanceClientId`, `appId`, `offerSubscriptionId`, `iamClientId` |
| `AppInstanceClientDisabled` | 7.4 | `appInstanceClientId`, `appId`, `offerSubscriptionId`, `iamClientId` |
| `AppInstanceUrlUpdated` | 7.5 | `appInstanceClientId`, `appId`, `offerSubscriptionId`, `instanceUrl` |
| `AppRolesAdded` | 7.1 (added roles to an existing template) or 7.6 | `appId`, `roleNames` |
| `AppRoleDescriptionsUpdated` | 7.7 | `appId`, `roleNames` |

Every event also carries `eventId` (uuid), `eventType` (the name from the table) and `occurredAt`. The schemas are in [openapi.yaml](openapi.yaml) under `components/schemas`.

## 10. Calls to the Portal IAM (Keycloak Admin REST API)

All calls with the token of the module's own service account (`IAM_ADMIN_CLIENT_ID`), realm `IAM_REALM`.

| Purpose | Call |
|---|---|
| Find client | `GET /admin/realms/{realm}/clients?clientId={clientId}` |
| Create client | `POST …/clients` with the settings from 6.3; the internal ID is in the `Location` header |
| Read / change client (enable, disable, URL) | `GET` / `PUT …/clients/{uuid}`; for `PUT` send the read representation with the changed fields |
| Protocol mappers | `GET` / `POST …/clients/{uuid}/protocol-mappers/models` |
| Roles of a client | `GET …/clients/{uuid}/roles?briefRepresentation=false` |
| Create role | `POST …/clients/{uuid}/roles` with `name`, `description` |
| Set description and attributes | `PUT …/clients/{uuid}/roles/{roleName}` (name URL-encoded) with `name`, `description`, `attributes`; always after creating a role |

Attributes are sent as lists, e.g. `"attributes": {"description.de": ["Lesen"], "description.en": ["Read"]}`.

## 11. Flows that use the module

For orientation. The flows themselves belong to Process Orchestration.

| Flow | Started by | Steps of this module |
|---|---|---|
| App approved | Offers | `create-client-template`; for a single-instance app also `create-instance-client` (`SINGLE`, URL from the release) and `enable-instance-client` |
| Subscription to a multi-instance app set up by the provider | Subscriptions | `create-instance-client` (`SUBSCRIPTION`); at activation `enable-instance-client` |
| Subscription to a single-instance app | Subscriptions | none |
| Provider adds app roles to an active app | Offers | `add-app-roles` |
| Provider changes role descriptions | Offers | `update-role-descriptions` |
| Provider changes the URL of an instance | Offers or Subscriptions | `update-instance-url` |
| Subscription ended | Subscriptions | `disable-instance-client` |

## 12. Errors

Errors come as Problem Details (RFC 9457). `type` is one of these identifiers:

| `type` | HTTP | When |
|---|---|---|
| `urn:c2-m6:problem:validation` | 400 | invalid input |
| `urn:c2-m6:problem:forbidden` | 403 | permission missing (HTTP only) |
| `urn:c2-m6:problem:not-found` | 404 | instance client does not exist |
| `urn:c2-m6:problem:role-not-found` | 404 | role does not exist on the template |
| `urn:c2-m6:problem:template-missing` | 409 | no template for the app |
| `urn:c2-m6:problem:instance-exists` | 409 | an instance client for this subscription or single-instance app exists with different data |
| `urn:c2-m6:problem:invalid-state` | 409 | status does not allow the step |
| `urn:c2-m6:problem:iam-error` | 502 | the Portal IAM did not answer as expected |

## 13. Acceptance

The module is done when these cases work against the test Portal IAM:

1. `create-client-template` with two roles in `de` and `en`: the client `app-template-{appId}` exists, is disabled and has both roles with the attributes `description.de`, `description.en` and the English text in `description`.
2. `create-instance-client` (`SUBSCRIPTION`): a disabled public client `app{n}` with `baseUrl`, redirect `{URL}/*`, the audience mapper and both roles including attributes. Status `PENDING`.
3. `enable-instance-client`: the client is enabled, status `ACTIVE`. A test user who has one of the roles on this client (assigned in the test via the admin API) gets an example access token (`GET …/clients/{uuid}/evaluate-scopes/generate-example-access-token?userId=`) that contains the role under `resource_access.{clientId}.roles` and the client ID in `aud`.
4. `add-app-roles` with one new and one existing role: the new role appears on the template and on every instance client, including disabled ones; the existing role is reported as skipped and keeps its descriptions.
5. `update-role-descriptions`: changed attributes on the template and on every instance client; an unknown role gives 404 and changes nothing.
6. Every step run twice with the same input: no second client, no second role, no second mapper, the same result, only one event.
7. Interrupted runs (simulated): a client created in the Portal IAM before `iam_client_uuid` was stored is completed by a repetition with the same client ID; a role created without attributes is completed by a repetition of `add-app-roles`.
8. `create-instance-client` before the template: 409 `template-missing`. A second `SINGLE` with a different URL: 409 `instance-exists`.
9. `disable-instance-client`: the client is disabled, status `DISABLED`; `enable-instance-client` afterwards gives 409.
10. `/internal/app-roles` for two apps, one without template: roles with description in the requested language for the first, `templateExists: false` for the second.
11. `/internal/app-instance-clients` without `appId` and `offerSubscriptionId`: 400.
12. `directAccessGrantsEnabled` is `false` on every client the module created.

## 14. Test data

The test Portal IAM needs:

- the realm `IAM_REALM` with the client scopes from `DEFAULT_CLIENT_SCOPES` and `OPTIONAL_CLIENT_SCOPES`
- a service account for the module with the permissions listed under `IAM_ADMIN_CLIENT_ID`
- one test user for case 3
- for calls via HTTP: the permissions from 6.6 for a test service account

Example app for the tests: app `Demo App`, roles `Viewer` (`de`: „Darf Daten lesen“, `en`: “May read data”) and `Editor` (`de`: „Darf Daten ändern“, `en`: “May change data”), instance URL `https://demo-app.example.org`.
