# C3-M2 CSP Connection

**Version:** 0.1, as of 2026-10-29 (new with OSP 2.0)
**Capability:** C3 Company
**Kind:** Hackathon
**Files:** this README (behaviour), [openapi.yaml](openapi.yaml) (interface), [open-points.md](open-points.md)

This README and the OpenAPI description are sufficient to build the module. The binding framework is [FRAMEWORK.md](../FRAMEWORK.md).

## 1. Purpose

The CSP Connection is the only place through which an installation uses the core services of the CSP (FRAMEWORK 1). It

- registers a company with the CSP: the CSP assigns or confirms the company identifier (BPN) and enters the OSP as the home OSP;
- checks before the registration whether a company is already registered with an OSP (duplicate check);
- registers and deregisters the membership in the dataspace;
- enters the DID of the company in the directory BPN → DID;
- passes changed core data of the company to the CSP;
- transfers a company to this OSP and accepts the notification that a company has moved away;
- enters the connectors of the companies in the discovery of the CSP and removes them again.

The interface to the CSP (section 11.1) is a proposal. If the association defines it differently, only this module changes.

Callers:

| Who | What for |
|---|---|
| Process Orchestration | steps: register, transfer, change core data, enter DID, register and deregister membership, register and deregister a connector (section 8) |
| C1-M3 Application Review | query `find-companies` for the duplicate check (section 9) |
| OSP staff (through the UI) | view the status of a company at the CSP |
| CSP | notifications to the installation (callback, 7.2) |

## 2. Terms

| Term | Meaning |
|---|---|
| CSP | core service provider; runs the core services centrally |
| Core services API | interface of the CSP for the OSPs (proposal, 11.1) |
| Home OSP | the OSP with which a company is registered; exactly one per company |
| Registration with the CSP | entry of the company in the company identifier service of the CSP, with BPN and home OSP |
| BPN | Business Partner Number; company identifier in the dataspace, assigned by the CSP (today from the master data service) |
| Membership | entry of the company as a member of the dataspace at the CSP |
| Directory BPN → DID | core service that returns the DID of the dataspace wallet for a BPN (today BDRS) |
| `ospId` | identifier of the OSP at the CSP, from the trust list (FRAMEWORK 4.8) |
| Operation | process at the CSP that does not finish at once; the CSP reports its end by callback |
| Identifier | identifier of the company with type and value, such as `VAT_ID` or `COMMERCIAL_REG_NUMBER` (identifier types from the reference data, FRAMEWORK 4.8) |
| Discovery | core service of the CSP that names the connectors of a company for a BPN |

## 3. Not part of this module

| What | Where |
|---|---|
| Record of the company in the installation, keeping the BPN in the installation | C3-M1 Company Profile |
| Trigger credentials at the issuer of the CSP | C4-M1 Credentials (its own adapter) |
| Create the dataspace wallet of the CSP | C4-M2 Dataspace Wallet Interface (its own adapter) |
| Read the trust list and check issuers | C5-M1 EBW-Connector |
| Admission of the OSP to the trust list | governance and CSP (FRAMEWORK 5.1) |
| Keeping the connectors of the companies (address, status) | Connectors (other part of the spec); this module only enters them at the CSP (8.6) |
| Take data, technical users and wallet along on a transfer | later work (FRAMEWORK 5.6) |

## 4. Technical frame

- Module of the modular monolith (Python, FastAPI), own database `c3_m2` in PostgreSQL.
- Authentication at the CSP: OAuth 2.0 client credentials with `private_key_jwt` (RFC 7523). `client_id` is the `ospId`; the OSP signs with its key, whose public part is in the trust list. No secret. The access token of the CSP is reused until shortly before it expires.
- Callbacks from the CSP: the CSP has a service account in the Portal IAM of the OSP with exactly the permission `report_csp_notification` (FRAMEWORK 4.2).
- Timeout per call to the CSP 10 seconds; for steps, errors of the CSP give `retryable: true`, except for a business rejection (section 12).
- No personal data: the module passes only data of the company to the CSP.

### Configuration

| Variable | Meaning | Example |
|---|---|---|
| `DATABASE_URL` | own database `c3_m2` | |
| `CSP_URL` | address of the core services API; empty = built-in stand-in (11.3) | `https://core.example.org/api/v1` |
| `CSP_TOKEN_URL` | token endpoint of the CSP | `https://core.example.org/oauth/token` |
| `OSP_SIGNING_KEY_FILE` | private key of the OSP (JWK, ES256) for `private_key_jwt` | `osp-key.json` |
| `SETTINGS_FILE` | installation settings (FRAMEWORK 4.8): `ospId`, `ospName` | `config/installation.json` |
| `CSP_STUB_FILE` | stand-in only: pre-registered companies of other OSPs (test data) | `csp-stub.json` |

## 5. Data

### Table `csp_registration`

One row per company of the installation that is being or was registered with the CSP.

| Field | Type | Rule |
|---|---|---|
| `company_id` | uuid | primary key; `companyId` from C3-M1 |
| `bpn` | text | from the CSP; empty until the registration |
| `status` | text | 6.1 |
| `failure_reason` | text | with `FAILED`: reason given by the CSP |
| `pending_operation_id` | text | operation at the CSP while it runs |
| `did` | text | registered DID, empty until `register-did` |
| `membership` | text | `NONE`, `REGISTERED`, `DEREGISTERED` |
| `registered_at`, `updated_at` | timestamptz | |

### Table `csp_connector`

One row per connector that the module has entered at the CSP.

| Field | Type | Rule |
|---|---|---|
| `connector_id` | uuid | primary key; identifier from the Connectors module |
| `company_id` | uuid | company of the connector |
| `bpn` | text | BPN when entered |
| `endpoint_url` | text | address of the connector (DSP endpoint) |
| `status` | text | `REGISTERED`, `DEREGISTERED` |
| `registered_at`, `updated_at` | timestamptz | |

### Table `csp_notification`

One row per accepted notification from the CSP: `notification_id` (unique, from the CSP), `type`, `bpn`, `received_at`. Duplicate notifications are detected and processed only once.

### Table `audit_entry`

One entry per call to the CSP and per notification: time, operation, `company_id`, `bpn`, result (HTTP status, error `type` of the CSP). No content of the responses, no tokens.

**Not stored:** data of the company beyond the BPN (C3-M1 keeps it); access tokens of the CSP beyond memory.

## 6. Rules

### 6.1 Status of the registration

| From | Event | To |
|---|---|---|
| – | `register-company` or `transfer-company` started | `PENDING` |
| `PENDING` | CSP confirms (at once or by callback) | `REGISTERED` |
| `PENDING` | CSP rejects | `FAILED` |
| `FAILED` | `register-company` again | `PENDING` |
| `REGISTERED` | notification `CompanyTransferredAway` | `TRANSFERRED_AWAY` |
| `TRANSFERRED_AWAY` | `transfer-company` (return) | `PENDING` |

`TRANSFERRED_AWAY` is final for this installation until the company moves back.

### 6.2 Duplicate check

`find-companies` asks the CSP for companies with the same EBWOID, the same EUID, the same other identifier (type and value) or the same name in the same country (compared at the CSP). For each match, the result gives the BPN, the home OSP and whether the home OSP is this OSP. C1-M3 assesses the match, not this module.

### 6.3 Permissions

| Permission | Scope | What for |
|---|---|---|
| `view_csp_registration` | `osp` | view the status of a company at the CSP (portal role `OSP Admin`) |
| `report_csp_notification` | `osp` | service account of the CSP only: send notifications |
| `execute_steps_c3_m2`, `query_c3_m2` | `osp` | steps and queries when called over HTTP (FRAMEWORK 4.2) |

### 6.4 Repeatability

- `register-company` with an existing row in `REGISTERED` returns the stored BPN without calling the CSP again. With `PENDING` and a running operation, the step answers `RUNNING` without starting a second operation.
- `register-did`, `register-membership` and `deregister-membership` call the CSP only if the value changes; the CSP also treats equal calls as idempotent (`PUT`).
- `register-connector` calls the CSP only if the connector is new or its address has changed; `deregister-connector` for an unknown or already removed connector returns `DONE` without a call.
- A notification with a known `notificationId` is acknowledged with 200 and not processed again.

### 6.5 Identifiers

Without a Business Wallet (OP-35) a company has no EBWOID. Therefore `register-company` needs at least one of `ebwoid`, `euid` or an identifier in `identifiers`. The CSP uses them for the duplicate check and to assign the BPN.

### 6.6 Connectors in the discovery

- A connector can be entered only for a company with status `REGISTERED`.
- In its discovery, the CSP names only connectors of companies with an active membership. On deactivation (membership deregistered, FRAMEWORK 5.4) no own step is therefore needed; after reactivation the connectors appear again.
- A connector is removed only when the Connectors module deletes it (`deregister-connector`).

## 7. Operations

### 7.1 `GET /api/c3-m2/companies/{companyId}`

Permission `view_csp_registration`. Response 200 `CspRegistration`: `companyId`, `bpn`, `status`, `failureReason`, `did`, `membership`, `registeredAt`, `updatedAt`. Unknown: 404.

### 7.2 `POST /api/c3-m2/csp-notifications`

Callback from the CSP, permission `report_csp_notification`. Input `CspNotification`: `notificationId`, `type`, `bpn`, with `OperationCompleted` also `operationId`, `outcome` (`SUCCEEDED`, `FAILED`), `reason`.

1. `notificationId` known: 200, no processing.
2. Find the row for the `bpn` or the `operationId`; no matching row: 404 `unknown-company`.
3. By `type`:
   - `OperationCompleted` with `SUCCEEDED`: status `REGISTERED`, take over the BPN → `CompanyRegisteredAtCsp`. With `FAILED`: status `FAILED` with the reason → `CompanyRegistrationAtCspFailed`.
   - `CompanyTransferredAway`: status `TRANSFERRED_AWAY` → `CompanyTransferredAway`.
4. Store the notification, response 200.

## 8. Steps for Process Orchestration

Input always with `processId`, `stepRunId` and `companyId` (FRAMEWORK 4.4).

### 8.1 `POST /api/c3-m2/steps/register-company`

- **Input:** in addition `legalName`, `country`, `ebwoid` (optional), `euid` (optional), `identifiers` (optional; list of `type` and `value`), `address` (optional) from the application or from C3-M1. At least `ebwoid`, `euid` or an identifier in `identifiers` (6.5); otherwise 400 `validation`.
- **Precondition:** application approved (`ApplicationApproved`); no row in `REGISTERED` for another company with the same EBWOID or EUID.
- **Behaviour:** `POST {CSP_URL}/companies`. 201: status `REGISTERED`, store the BPN. 202: status `PENDING`, store the `operationId`. 409 (already registered with another OSP): status `FAILED`, reason `REGISTERED_AT_OTHER_OSP`.
- **Result:** `DONE` with `bpn`, or `RUNNING`.
- **Event:** `CompanyRegisteredAtCsp` or `CompanyRegistrationAtCspFailed`; with `RUNNING`, later through the callback (7.2).

### 8.2 `POST /api/c3-m2/steps/transfer-company`

- **Input:** in addition `bpn` (from `find-companies`); `ebwoid`, `euid` and `identifiers` optional, for the check at the CSP.
- **Precondition:** application of kind `TRANSFER` approved.
- **Behaviour:** `POST {CSP_URL}/companies/{bpn}/transfer`; responses as in 8.1. The CSP notifies the previous home OSP itself.
- **Result, event:** as in 8.1.

### 8.3 `POST /api/c3-m2/steps/register-did`

- **Input:** in addition `did`.
- **Precondition:** status `REGISTERED`; dataspace wallet created (`DataspaceWalletCreated`).
- **Behaviour:** `PUT {CSP_URL}/companies/{bpn}/did` with `did`; store the DID.
- **Result:** `DONE` with `bpn`, `did`. **Event:** `CompanyDidRegistered`.

### 8.4 `POST /api/c3-m2/steps/update-company`

- **Input:** in addition `legalName`, `euid` (optional), `address` (optional) from C3-M1 (`CompanyDataChanged` with `coreDataChanged`).
- **Precondition:** status `REGISTERED`.
- **Behaviour:** `PATCH {CSP_URL}/companies/{bpn}` with the changed core data. Country and EBWOID do not change (C3-M1).
- **Result:** `DONE`. **Event:** `CompanyUpdatedAtCsp`.

### 8.5 `POST /api/c3-m2/steps/register-membership` and `deregister-membership`

- **Precondition:** status `REGISTERED`.
- **Behaviour:** `PUT {CSP_URL}/companies/{bpn}/membership` with `{ "member": true }` or `false`; store the value.
- **Result:** `DONE`. **Event:** `MembershipRegistered` or `MembershipDeregistered`.

### 8.6 `POST /api/c3-m2/steps/register-connector` and `deregister-connector`

- **Input:** in addition `connectorId`; for `register-connector` also `endpointUrl` (address of the connector, HTTPS).
- **Precondition:** status `REGISTERED` (otherwise 409 `invalid-state`); connector created or deleted in the Connectors module.
- **Behaviour:** `PUT {CSP_URL}/companies/{bpn}/connectors/{connectorId}` with `endpointUrl`, or `DELETE {CSP_URL}/companies/{bpn}/connectors/{connectorId}`; store the row in `csp_connector`.
- **Result:** `DONE` with `bpn` and `connectorId`. **Event:** `ConnectorRegisteredAtCsp` or `ConnectorDeregisteredAtCsp`, only if something changed.

## 9. Queries for other modules

### `GET /api/c3-m2/internal/find-companies`

Parameters: `ebwoid`, `euid`, `identifier` (form `<type>:<value>`, repeatable), `legalName`, `country` (at least `ebwoid`, `euid`, `identifier`, or `legalName` with `country`). Calls `GET {CSP_URL}/companies` with the same parameters.

Response 200: list of `CspCompanyMatch`: `bpn`, `legalName`, `country`, `matchedBy` (`EBWOID`, `EUID`, `IDENTIFIER`, `NAME`), `homeOspId`, `homeOspName`, `homeIsThisOsp`, `memberActive`. Empty list: no match. CSP not reachable: 502 `upstream-error`.

## 10. Events

| Event | When | Content |
|---|---|---|
| `CompanyRegisteredAtCsp` | registration or transfer confirmed | `companyId`, `bpn` |
| `CompanyRegistrationAtCspFailed` | CSP rejects | `companyId`, `reason` |
| `CompanyDidRegistered` | DID entered in the directory | `companyId`, `bpn`, `did` |
| `CompanyUpdatedAtCsp` | changed core data passed to the CSP | `companyId`, `bpn` |
| `MembershipRegistered`, `MembershipDeregistered` | membership registered or deregistered | `companyId`, `bpn` |
| `CompanyTransferredAway` | the company has moved to another OSP | `companyId`, `bpn` |
| `ConnectorRegisteredAtCsp`, `ConnectorDeregisteredAtCsp` | connector entered in or removed from the discovery | `companyId`, `bpn`, `connectorId` |

## 11. Calls to other modules and external systems

### 11.1 Core services API of the CSP (proposal)

All calls with the access token of the CSP (4). Responses in JSON; errors as Problem Details.

| Call | Purpose | Responses |
|---|---|---|
| `GET /companies?ebwoid=&euid=&identifier=&legalName=&country=` | duplicate check | 200 list with `bpn`, `legalName`, `country`, `matchedBy`, `homeOspId`, `homeOspName`, `memberActive` |
| `POST /companies` with `legalName`, `country`, `ebwoid`, `euid`, `identifiers`, `address` (at least one identifier) | register; the home OSP is the calling OSP | 201 `{ bpn }`; 202 `{ operationId }`; 409 `{ bpn, homeOspId }` if another OSP is the home OSP |
| `POST /companies/{bpn}/transfer` with `ebwoid`, `euid`, `identifiers` (optional) | transfer to the calling OSP | as `POST /companies` |
| `PATCH /companies/{bpn}` with `legalName`, `euid`, `address` | change core data | 204 |
| `PUT /companies/{bpn}/did` with `did` | directory BPN → DID | 204 |
| `PUT /companies/{bpn}/membership` with `member` | membership | 204 |
| `PUT /companies/{bpn}/connectors/{connectorId}` with `endpointUrl` | enter a connector in the discovery | 204 |
| `DELETE /companies/{bpn}/connectors/{connectorId}` | remove a connector | 204, also if it is unknown |

On every changing call, the CSP checks that the calling OSP is the home OSP (except for `transfer`), and calls back: `POST {address of the installation}/api/c3-m2/csp-notifications` (7.2). The address of the installation is in the trust list, in the entry of the OSP.

### 11.2 C3-M1 Company Profile

No calls. Process Orchestration passes the data of the company as input to the steps.

### 11.3 Built-in stand-in

With an empty `CSP_URL`, the module answers the calls from 11.1 itself, in memory:

- `POST /companies` assigns a BPN derived from the EBWOID, without an EBWOID from the EUID, otherwise from the first identifier in the form `<type>:<value>`: `BPNL` + the first 12 characters of the Base32 hash (SHA-256) of this value, in upper case; always 201, unless the company is in `CSP_STUB_FILE` with another home OSP (then 409).
- `GET /companies` searches the module's own registrations and `CSP_STUB_FILE`.
- `transfer`, `did`, `membership`, `connectors` answer at once; there are no callbacks.

For the hackathon, the test tools provide a shared CSP stand-in that several installations use (FRAMEWORK 4.7); it speaks the same core services API.

## 12. Errors

Problem Details, `type` = `urn:c3-m2:problem:<name>`.

| `type` | HTTP | When | `retryable` for steps |
|---|---|---|---|
| `validation` | 400 | invalid input | `false` |
| `unauthorized` | 401 | token missing or invalid | – |
| `forbidden` | 403 | permission missing | – |
| `not-found` | 404 | company without registration | – |
| `unknown-company` | 404 | notification for an unknown BPN or operation | – |
| `invalid-state` | 409 | a precondition of a step is missing | `false` |
| `registered-at-other-osp` | 409 | the CSP reports another home OSP | `false` |
| `upstream-error` | 502 | CSP not reachable or error 5xx | `true` |
| `csp-rejected` | 502 | the CSP rejects for business reasons (4xx except 409) | `false` |

## 13. Acceptance

1. `register-company` with the stand-in: `DONE` with BPN `BPNL…` (16 characters); a second call returns the same BPN without a new registration; `CompanyRegisteredAtCsp` exactly once.
2. `register-company` for an EBWOID from `CSP_STUB_FILE` with another home OSP: error `registered-at-other-osp`, status `FAILED`, `CompanyRegistrationAtCspFailed`.
3. `find-companies` with this EBWOID: match with `homeIsThisOsp` `false`; after case 1, for the own EBWOID: match with `homeIsThisOsp` `true`.
4. A test CSP at `CSP_URL` answers with 202 (the built-in stand-in always answers at once): step `RUNNING`; the callback `OperationCompleted` with `SUCCEEDED` leads to `REGISTERED` and `CompanyRegisteredAtCsp`; the same notification a second time: 200 without a second event.
5. `register-did` before `REGISTERED`: `invalid-state`. After it: DID stored, `CompanyDidRegistered`.
6. `update-company` after `REGISTERED`: call to the CSP, `CompanyUpdatedAtCsp`; before `REGISTERED`: `invalid-state`.
7. `register-membership`, then `deregister-membership`: `membership` `REGISTERED` or `DEREGISTERED`, one event each; a repetition does not call the CSP.
8. Callback `CompanyTransferredAway`: status `TRANSFERRED_AWAY`, event `CompanyTransferredAway`.
9. Callback without the permission `report_csp_notification`: 403. CSP not reachable: step with `upstream-error`, `retryable: true`.
10. Authentication at the CSP: the client assertion is signed with the key of the OSP; `iss` and `sub` are the `ospId`.
11. `register-company` without EBWOID and EUID, with `identifiers` `[{ "type": "COMMERCIAL_REG_NUMBER", "value": "HRB 200002" }]` (Beispiel Logistik GmbH): `DONE` with a BPN derived from this identifier. Without any identifier: 400 `validation`. `find-companies?identifier=COMMERCIAL_REG_NUMBER:HRB 200002` afterwards: a match with `matchedBy` `IDENTIFIER`.
12. `register-connector` before `REGISTERED`: `invalid-state`. Afterwards with `endpointUrl`: call at the CSP, row `REGISTERED`, `ConnectorRegisteredAtCsp`; the same call again: `DONE` without a call and without an event; a new `endpointUrl`: a new call and an event. `deregister-connector`: `DEREGISTERED`, `ConnectorDeregisteredAtCsp`; again: `DONE` without an event.

## 14. Test data

| Object | Value |
|---|---|
| OSP A (`ospId`) | `22222222-2222-4222-8222-222222222222` |
| OSP B (`ospId`) | `99a5685c-7397-402c-88de-b1b3a939ecc9` |
| Test company | Beispiel Maschinenbau GmbH, `EBWOID-DE-0000000001`, EUID `DEXX001T.HRB100001`, DE |
| Company without a Business Wallet | Beispiel Logistik GmbH, `COMMERCIAL_REG_NUMBER` `HRB 200002`, DE |
| Connector of Beispiel Maschinenbau | `connectorId` `7c1e5a2b-4d3f-4e6a-9b8c-0d1e2f3a4b5c`, `endpointUrl` `https://edc.beispiel-maschinenbau.example/api/v1/dsp` |
| Company at OSP B (`CSP_STUB_FILE`) | Andere Beispiel AG, `EBWOID-DE-0000000002`, BPN `BPNL00000000002B`, home OSP OSP B |

`CSP_STUB_FILE`:

```json
[
  { "bpn": "BPNL00000000002B", "legalName": "Andere Beispiel AG", "ebwoid": "EBWOID-DE-0000000002",
    "country": "DE", "homeOspId": "99a5685c-7397-402c-88de-b1b3a939ecc9", "homeOspName": "OSP B",
    "memberActive": true }
]
```
