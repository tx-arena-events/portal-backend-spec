# C4-M1 Credentials

**Version:** 0.2, as of 2026-10-30
**Capability:** C4 Dataspace
**Kind:** Hackathon
**Files:** this README (behaviour), [openapi.yaml](openapi.yaml) (interface), [open-points.md](open-points.md)

This README and the OpenAPI description are sufficient to build the module. The binding framework is [../FRAMEWORK.md](../FRAMEWORK.md). Other modules are referred to by number and name. Paths and field names are in English.

## 1. Purpose

The module triggers the basic credentials of a company at the issuer: the BPN credential and the membership credential. It tracks their status and revokes the membership credential when the company is deactivated. The credentials themselves are in the company's dataspace wallet; the module only stores which credential was triggered, issued or revoked, and when.

The configuration of the installation of the onboarding service provider (OSP) defines which issuer signs (`ISSUER_MODE`, 6.3):

- **Own issuer of the OSP** (`OWN`). The OSP runs its own issuer, which is on the CSP's trust list (6.4). The module knows two kinds:
  - **Credential Issuer** (today's route, together with SAP DIV): The module instructs the Tractus-X Credential Issuer. It has the credential signed in the OSP's wallet and places it in the company's dataspace wallet.
  - **IssuerService** (IdentityHub family, DCP): The module registers the company as a holder with the IssuerService and triggers the request in the company's dataspace wallet. The wallet fetches the credential via DCP.
- **CSP issuer** (`CSP`). The CSP offers an issuer as an optional service. It works like an IssuerService: the module grants the credential types for the company at the CSP issuer and triggers the request in the wallet; the wallet fetches the credential via DCP. The module signs in at the CSP with the OSP's key.

In the hackathon, OSP A issues credentials itself (own IssuerService), and OSP B uses the CSP issuer (FRAMEWORK 1). For Process Orchestration, the user interface and other modules, the module is the same in both cases: the same steps, queries, events and status reports.

In all cases the issuer or a callback adapter tells the module as soon as a credential is issued or has failed. The module does not poll.

Callers:

| Who | What for |
|---|---|
| Process Orchestration | steps: have credentials issued (onboarding, reactivation), revoke the membership credential (deactivation) |
| People of a company (through the user interface, scope `company`) | read the status of the credentials of their own company |
| OSP staff (through the user interface, scope `osp`) | read the status of all companies, find failed and open credentials, confirm a revocation done by hand |
| Issuers and callback adapters (one service account each in the OSP's Portal IAM) | status report per credential: issued or failed |
| other modules (queries) | read the status of a company's credentials |

### Outlook

Later, further credential types will be added, above all credentials for participation in use cases (today "framework" credentials, at the Credential Issuer on request of the company and with confirmation by the operator of the issuer) and certificates (for example ISO certificates). It is also conceivable to issue the membership credential only when verified credentials from the Business Wallet are present (at the IssuerService through its "attestations"). For this, a permission to apply for credentials would be added for people of the company (scope `company`). Version 0.2 contains none of this; the data model allows further credential types.

## 2. Terms

| Term | Meaning |
|---|---|
| Credential | Verifiable Credential in the company's dataspace wallet; connectors present it during data exchange |
| Credential type | `BPN` (BpnCredential: company identifier of the holder) or `MEMBERSHIP` (MembershipCredential: membership in the dataspace of the installation) |
| Issuer | issues and signs credentials: own issuer of the OSP (Credential Issuer or IssuerService) or CSP issuer |
| Issuer mode | setting `ISSUER_MODE` of the installation: `OWN` (own issuer) or `CSP` (CSP issuer) |
| Route | adapter through which the module triggers a credential: `CREDENTIAL_ISSUER`, `ISSUER_SERVICE`, `CSP_ISSUER` or `STUB` (built-in stand-in) |
| Wallet provider | provider of the dataspace wallet according to C4-M2 (field `provider`): `SAP_DIV`, `IDENTITY_HUB`, `CSP` (dataspace wallet of the CSP), `BRING_YOUR_OWN` (own wallet, later); in the built-in stand-in of C4-M2 also `STUB` |
| Issuance mode of the wallet | according to C4-M2 (field `issuanceMode`): `ISSUER_WRITES` (the issuer writes into the wallet; SAP DIV), `HOLDER_REQUESTS` (the wallet requests via DCP; IdentityHub, dataspace wallet of the CSP), `CREDENTIAL_OFFER` (own wallet, later) |
| Holder | the company with its dataspace wallet and DID |
| Holder access | access to the holder's dataspace wallet that C4-M2 gives only to this module: for the Credential Issuer to write into the wallet, for DCP to trigger the request |
| Grant | at the CSP issuer: permission for a holder to request a credential type |
| Status report | message of the issuer or callback adapter: credential of a type for a BPN issued or failed |
| Callback adapter | part of the holder's dataspace wallet that reports when a requested credential has arrived; needed because the IssuerService itself does not call back. An add-on in the OSP's own IdentityHub, part of the service in the CSP's dataspace wallet |
| Issuance round | one run of `issue-credentials` for a company, until all credential types of the round are issued or one fails |
| Trust list | list of the CSP with the admitted OSPs and issuers (FRAMEWORK 1); connectors accept only credentials from issuers on this list |
| Service account | client in the OSP's Portal IAM with which an external system calls back; exactly one permission (FRAMEWORK 4.2) |

## 3. Not part of this module

| What | Where |
|---|---|
| Create the dataspace wallet, DID, location of the DID document, wallet accounts | C4-M2 Dataspace Wallet Interface |
| Entry BPN → DID in the CSP's directory | C3-M2 CSP Connection (`register-did`) |
| Register and deregister the membership at the CSP | C3-M2 CSP Connection |
| Store and present credentials | dataspace wallet of the company |
| Sign credentials | issuer |
| Add issuers to the trust list | governance and CSP (FRAMEWORK 5.1) |
| Company identifier (BPN), status of the company | C3-M1 Company Profile |
| Check credentials of the EUDI Wallet and the Business Wallet (PID, LPID, mandate) | C5-M1 EBW-Connector |
| Order of the steps, retriggering | Process Orchestration |
| Notify people | Notifications (another part) |
| Further credential types | later, see Outlook |

## 4. Technical frame

- Python with FastAPI, part of the modular monolith. Process Orchestration and other modules call the module in the same process; the user interface, issuers and callback adapters call it over HTTP. [openapi.yaml](openapi.yaml) describes both.
- Own database `c4_m1` in PostgreSQL with its own database user.
- Every HTTP call carries a token of the OSP's Portal IAM, validated with the shared library. Scope `company`: `company_id` in the token equals `companyId` in the path, otherwise 404 (FRAMEWORK 4.3).
- Events through the module's outbox, in the same transaction as the change.
- **No polling:** The module waits for status reports. It asks neither the issuer nor the wallet for the state. If a credential hangs, Process Orchestration triggers the step again (8.1).
- **Sign-in at the CSP** (route `CSP_ISSUER` only): OAuth 2.0 client credentials with `private_key_jwt` (RFC 7523) at the CSP's token endpoint, as in C3-M2. `client_id` is the `ospId` from the installation settings; the OSP signs with its key, whose public part is on the trust list. No secret. The CSP's access token is reused until shortly before it expires. Time limit per call to the CSP: 10 seconds.
- **Callbacks:** Every issuer and every callback adapter calls back with its own service account in the OSP's Portal IAM, which has exactly the permission `report_credential_status` (FRAMEWORK 4.2; here 6.5).
- **Secrets:** The module fetches the holder access (Credential Issuer: address, client ID, secret of the wallet; DCP: API key of the participant context) from C4-M2 once per run, immediately before the first order (8.1). It keeps it only in memory, passes it on only in the order to its own Credential Issuer, and never stores or logs it. The module passes no holder access to the CSP. Its own secrets (access to Credential Issuer and IssuerService, key of the OSP) are only in the configuration.
- **Installation settings** (FRAMEWORK 4.8): At start-up the module reads `ospId` (sign-in at the CSP) and `dataspace.memberOf` (membership credential) from `SETTINGS_FILE`.
- No personal data. The audit log (FRAMEWORK 4.3) contains only identifiers, no names.

### Configuration

| Variable | Meaning | Example |
|---|---|---|
| `DATABASE_URL` | own PostgreSQL database `c4_m1` | |
| `SETTINGS_FILE` | installation settings (FRAMEWORK 4.8): `ospId`, `dataspace.memberOf` | `config/installation.json` |
| `ISSUER_MODE` | `OWN` = own issuer of the OSP; `CSP` = CSP issuer (6.3) | `OWN` |
| `ONBOARDING_CREDENTIAL_TYPES` | credential types in onboarding, comma-separated | `BPN,MEMBERSHIP` |
| `ISSUER_ROUTES` | only with `OWN`: route per wallet provider; empty = all through the stand-in `STUB` | `SAP_DIV:CREDENTIAL_ISSUER,IDENTITY_HUB:ISSUER_SERVICE,CSP:ISSUER_SERVICE` |
| `CALLBACK_BASE_URL` | externally reachable address of the installation for status reports | `https://portal.osp-a.example.org` |
| `RESEND_AFTER_HOURS` | from this age on, a repetition of the step may send an open order again | `24` |
| `CREDENTIAL_ISSUER_URL` | `OWN` only: base address of the own Credential Issuer | `https://ssi-credential-issuer.osp-a.example.org` |
| `CREDENTIAL_ISSUER_TOKEN_URL`, `CREDENTIAL_ISSUER_CLIENT_ID`, `CREDENTIAL_ISSUER_CLIENT_SECRET` | access of the module to the Credential Issuer (OAuth 2.0 client credentials) | |
| `ISSUER_SERVICE_ADMIN_URL`, `ISSUER_SERVICE_API_KEY` | `OWN` only: administration interface of the own IssuerService and its key | |
| `ISSUER_SERVICE_PARTICIPANT_ID` | participant context of the issuer in the own IssuerService | `issuer` |
| `ISSUER_DID` | DID of the own IssuerService; is on the trust list (6.4) | `did:web:issuer.osp-a.example.org` |
| `ISSUER_SERVICE_CREDENTIAL_DEFINITIONS` | credential definition per credential type in the own IssuerService | `BPN:bpn-credential-def,MEMBERSHIP:membership-credential-def` |
| `CSP_ISSUER_URL` | `CSP` only: base address of the CSP's issuer API (11.5); empty = stand-in `STUB` | `https://issuer.csp.example.org/api/v1` |
| `CSP_ISSUER_DID` | DID of the CSP issuer; the wallet requests there | `did:web:issuer.csp.example.org` |
| `CSP_ISSUER_CREDENTIAL_DEFINITIONS` | identifier per credential type at the CSP issuer | `BPN:bpn-credential,MEMBERSHIP:membership-credential` |
| `CSP_TOKEN_URL` | token endpoint of the CSP | `https://core.example.org/oauth/token` |
| `OSP_SIGNING_KEY_FILE` | private key of the OSP (JWK, ES256) for `private_key_jwt`; the same as in C3-M2 | `osp-key.json` |
| `CREDENTIAL_FORMAT` | format of the requested credentials for DCP routes | `VC1_0_JWT` |
| `STUB_ISSUER_DELAY_SECONDS` | stand-in: waiting time until the status report | `5` |
| `STUB_ISSUER_FAIL_TYPES` | stand-in: credential types that fail | `MEMBERSHIP` |
| `COMPANY_URL` | C3-M1: empty = stand-in, `in-process` = in the same process, otherwise HTTP address | `in-process` |
| `WALLET_URL` | C4-M2: as above | `in-process` |

The variables of a route that the installation does not use stay empty. A revocation needs the variables of the route through which the credential was issued (6.3).

## 5. Data

### Table `credential`

One row per company and credential type with the current state.

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key |
| `company_id` | uuid | required; unique together with `credential_type` |
| `credential_type` | text | `BPN`, `MEMBERSHIP` |
| `status` | text | `REQUESTED`, `ISSUED`, `FAILED`, `CANCELLED`, `REVOCATION_PENDING`, `REVOKED` (6.1) |
| `route` | text | route of the last order: `CREDENTIAL_ISSUER`, `ISSUER_SERVICE`, `CSP_ISSUER`, `STUB` |
| `issuer_did` | text | DID of the issuer at the last order (`ISSUER_DID` or `CSP_ISSUER_DID`); empty for `CREDENTIAL_ISSUER` and `STUB` |
| `wallet_provider` | text | wallet provider at the last order |
| `bpn` | text | BPN of the holder at the order, in upper case; key for status reports |
| `holder_did` | text | DID of the holder at the order |
| `request_ref` | text | identifier of the order at the route, for example the request ID in the wallet; empty if the route has none |
| `attempt` | integer | counter of the orders, starting at 1 |
| `requested_at`, `issued_at`, `failed_at`, `revoked_at` | timestamptz | |
| `failure_reason` | text | message of the issuer with `FAILED`, up to 1000 characters, without secrets |
| `revocation_reason` | text | `COMPANY_DEACTIVATED` or `OTHER` |
| `created_at`, `changed_at` | timestamptz | |

Index on (`bpn`, `credential_type`).

### Table `credential_history`

Every status change of a row in `credential`, for traceability.

| Field | Type |
|---|---|
| `id` | uuid, primary key |
| `credential_id` | uuid |
| `occurred_at` | timestamptz |
| `status` | text, new status |
| `attempt` | integer |
| `note` | text, up to 1000 characters, for example the message of the issuer or the note of OSP staff |

### Table `issuance_run`

One row per company with the last issuance round.

| Field | Type | Rule |
|---|---|---|
| `company_id` | uuid | primary key |
| `credential_types` | text[] | credential types of the round |
| `status` | text | `RUNNING`, `DONE`, `FAILED`, `CANCELLED` |
| `process_id`, `step_run_id` | uuid | last run of the step, for the audit log only |
| `started_at`, `finished_at` | timestamptz | |

### Table `audit_entry`

Audit log per FRAMEWORK 4.3: one row per changing operation over HTTP (7.3, 7.4).

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key |
| `occurred_at` | timestamptz | |
| `sub` | text | `sub` from the token: person or service account |
| `client_id` | text | `azp` from the token; for status reports the service account |
| `company_id` | uuid | `company_id` from the token, if present |
| `operation` | text | `CONFIRM_REVOCATION`, `REPORT_CREDENTIAL_STATUS` |
| `object_ref` | text | `credential_id`, or BPN and credential type |
| `result` | text | HTTP status of the response |

In addition, the table `outbox` of the base skeleton.

**Not stored:** the credentials themselves (dataspace wallet), the holder access (only in memory during the order), access tokens of the CSP beyond memory, DID documents, names and other personal data.

## 6. Rules

### 6.1 Status per credential

| From | Trigger | To | Event |
|---|---|---|---|
| – , `FAILED`, `CANCELLED`, `REVOKED` | `issue-credentials` triggers | `REQUESTED` (`attempt` + 1) | – |
| `REQUESTED` | status report successful | `ISSUED` | `CredentialsIssued`, if this completes the round (6.2) |
| `REQUESTED` | status report failed | `FAILED` | `CredentialIssuanceFailed` |
| `FAILED` | status report successful (late report) | `ISSUED` | as above |
| `ISSUED` | `revoke-credentials`, route can revoke | `REVOKED` | `CredentialRevoked` |
| `ISSUED` | `revoke-credentials`, route cannot revoke | `REVOCATION_PENDING` | – |
| `REVOCATION_PENDING` | OSP staff confirm the revocation (7.3) | `REVOKED` | `CredentialRevoked` |
| `REQUESTED`, `FAILED` | `revoke-credentials` | `CANCELLED` | – |
| `CANCELLED` | status report successful (late report) | revoked immediately: `REVOKED` or `REVOCATION_PENDING` | `CredentialRevoked` with `REVOKED` |

Every change writes a row to `credential_history`.

### 6.2 Issuance round

- `issue-credentials` starts a round for the credential types from the input; without them, for those from `ONBOARDING_CREDENTIAL_TYPES`. All credential types are triggered at the same time.
- The round is `DONE` as soon as all its credential types are `ISSUED`. Then, and only at the change to `DONE`, there is exactly one `CredentialsIssued` with all credential types of the round.
- If a credential type fails, the round becomes `FAILED`, and there is a `CredentialIssuanceFailed` for this type. The module continues to process status reports for the other types. A later success of all types still sets the round to `DONE`, with `CredentialsIssued`.
- `revoke-credentials` sets a running round to `CANCELLED`, without an event.
- Reactivation (FRAMEWORK 5.5): the round contains only `MEMBERSHIP`. The BPN credential is still `ISSUED` and is never triggered again.

### 6.3 Choice of the route

The route follows from `ISSUER_MODE` and the company's dataspace wallet (wallet provider and issuance mode from C4-M2), in this order:

1. Wallet provider `STUB`: route `STUB`.
2. `ISSUER_MODE` = `OWN`: route from `ISSUER_ROUTES` for the wallet provider. If `ISSUER_ROUTES` is empty, `STUB` applies. If the wallet provider has no entry: 409 `no-issuer-route`.
3. `ISSUER_MODE` = `CSP`: issuance mode `HOLDER_REQUESTS` gives `CSP_ISSUER`, with an empty `CSP_ISSUER_URL` `STUB`. Other issuance modes: 409 `no-issuer-route`. The CSP issuer issues only via DCP; for SAP DIV an OSP needs its own Credential Issuer.
4. The route must fit the issuance mode of the wallet: `CREDENTIAL_ISSUER` only with `ISSUER_WRITES`, `ISSUER_SERVICE` and `CSP_ISSUER` only with `HOLDER_REQUESTS`. Otherwise 409 `no-issuer-route` (configuration error).

An own wallet (`BRING_YOUR_OWN`, issuance mode `CREDENTIAL_OFFER`) has no route in version 0.2; it comes later (OP-29).

The route is stored with the order. A revocation uses the route of the order, even if `ISSUER_MODE` has been set differently in the meantime.

| Route | Issuer | Fits | Procedure | Status report | Revocation in 0.2 |
|---|---|---|---|---|---|
| `CREDENTIAL_ISSUER` | own Credential Issuer (`OWN`) | `ISSUER_WRITES`: `SAP_DIV` | order with holder access (11.3) | Credential Issuer | not supported → `REVOCATION_PENDING`; OSP staff revoke at the Credential Issuer and confirm |
| `ISSUER_SERVICE` | own IssuerService (`OWN`) | `HOLDER_REQUESTS`: `IDENTITY_HUB`, `CSP` | create the holder, request in the wallet (11.4) | callback adapter of the holder's wallet | through the administration interface of the IssuerService |
| `CSP_ISSUER` | CSP issuer (`CSP`) | `HOLDER_REQUESTS`: `IDENTITY_HUB`, `CSP` | holder and grant at the CSP, request in the wallet (11.5) | CSP issuer | remove the grant at the CSP |
| `STUB` | built-in stand-in | `STUB`, test | 11.6 | stand-in in the same process | yes |

### 6.4 Trust in the issuer

- **Own issuer:** The DID of the own issuer (`ISSUER_DID`; for the Credential Issuer the DID of the OSP's wallet it signs with) is on the CSP's trust list. Governance adds it when the OSP is admitted (FRAMEWORK 5.1). Only then do connectors of other companies accept the credentials. The module does not check this; it is an assumption (open-points.md).
- **CSP issuer:** It is on the trust list as a service of the CSP. The CSP grants credentials only for companies whose home OSP is the calling OSP and whose DID is in the directory BPN → DID (11.5). That is why `issue-credentials` runs only after `register-did` of C3-M2 (FRAMEWORK 5.2).

### 6.5 Permissions and service accounts

| Permission | Scope | Operations | Who |
|---|---|---|---|
| `view_credentials` | `company` | `GET /companies/{companyId}/credentials` | people of a company; proposal to C2-M5: portal roles `Company Admin`, `IT Admin` |
| `manage_credentials` | `osp` | `GET /credentials`, `POST /credentials/{credentialId}/confirm-revocation` | OSP staff; proposal: portal role `OSP Admin` |
| `report_credential_status` | `osp` | `POST /issuer-callbacks/{credentialKind}` | only service accounts of the issuers and callback adapters (table below) |
| `query_c4_m1` | `osp` | all paths under `/internal` (over HTTP only) | other modules |
| `execute_steps_c4_m1` | `osp` | all paths under `/steps` (over HTTP only) | Process Orchestration |

For service accounts, steps and queries, scope `osp` means: applies to all companies of the installation; only the permission is checked.

Order of checks for scope `company`: token (401), permission (403 `forbidden`), `company_id` in the token equals `companyId` in the path (otherwise 404 `not-found`, also if the token has no `company_id`).

Service accounts for status reports (one per system, exactly one permission; created by C2-M5):

| System | Needed for | Permission | Sign-in at the Portal IAM |
|---|---|---|---|
| own Credential Issuer | route `CREDENTIAL_ISSUER` | `report_credential_status` | `private_key_jwt` if the Credential Issuer supports it, otherwise a client secret as a transition (OP-22) |
| callback adapter in the own IdentityHub | route `ISSUER_SERVICE` with wallet provider `IDENTITY_HUB` | `report_credential_status` | as above |
| dataspace wallet of the CSP (callback function) | wallet provider `CSP` | `report_credential_status` | `private_key_jwt` with the CSP's key |
| CSP issuer | route `CSP_ISSUER` | `report_credential_status` | `private_key_jwt` with the CSP's key |

With the route `CSP_ISSUER`, the callback adapter of the wallet reports as well. The second identical report changes nothing (7.4).

### 6.6 Idempotency and concurrency

- Steps find their object by the `companyId`; per credential type, only what is missing is triggered (8.1). Events only on a change.
- Steps and status reports for the same company run one after the other: PostgreSQL advisory lock on the `companyId` for the duration of the transaction.
- Duplicate status reports change nothing (7.4).
- Creating holders and grants at the issuer is repeatable: an existing holder or an existing grant counts as success (11.4, 11.5).

## 7. Operations

### 7.1 `GET /companies/{companyId}/credentials`

For people of the company. Checks per 6.5 (scope `company`). Returns per credential type of the company: `credentialType`, `status`, `requestedAt`, `issuedAt`, `revokedAt`. Credential types from `ONBOARDING_CREDENTIAL_TYPES` without a row appear with status `NONE`. No messages of the issuer, no routes.

### 7.2 `GET /credentials`

For OSP staff. Filters: `status` (repeatable), `credentialType`, `companyId`, `route`, `requestedBefore` (to find open orders). Sorted by `changedAt` descending, paged (`page`, `size`). Returns all fields except secrets, also `failureReason`, `route`, `issuerDid`, `attempt`, `requestRef`.

### 7.3 `POST /credentials/{credentialId}/confirm-revocation`

OSP staff have revoked a credential at the Credential Issuer by hand and confirm this. Input: `note` (1 to 500 characters).

1. Token and permission `manage_credentials` (401, 403).
2. Unknown: 404. Status `REVOKED`: 200 without a change. Status not `REVOCATION_PENDING`: 409 `invalid-state`.
3. Otherwise in one transaction: `REVOKED`, `revoked_at`, history with `note`, audit log entry with `sub` of the person (5, `audit_entry`), event `CredentialRevoked`.
4. Response 200 with the credential.

### 7.4 `POST /issuer-callbacks/{credentialKind}`

Status report of an issuer or callback adapter. `credentialKind` is `bpn` or `membership`. Body as in today's portal: `{ "bpn", "status": "SUCCESSFUL" | "UNSUCCESSFUL", "message" }`.

1. Token and permission `report_credential_status` (401, 403).
2. `credentialKind` unknown: 404 `not-found`. `bpn` not in the format `BPNL` + 12 letters or digits (case-insensitive), or `status` invalid: 400 `validation`.
3. Look for the row for (`bpn` in upper case, credential type). None: 404 `unknown-request`.
4. Per status of the row:

| Status | `SUCCESSFUL` | `UNSUCCESSFUL` |
|---|---|---|
| `REQUESTED` | `ISSUED`; check the round (6.2) | `FAILED` with `message`; `CredentialIssuanceFailed`; round `FAILED` |
| `ISSUED` | nothing (duplicate report) | 409 `invalid-state` |
| `FAILED` | `ISSUED` (late report); check the round | nothing |
| `CANCELLED` | revoke immediately (6.1); if the revocation fails technically, the credential stays `CANCELLED` and the response is 502, so that the sender reports again | nothing |
| `REVOCATION_PENDING`, `REVOKED` | nothing | nothing |

5. Audit log entry with `sub` and `azp` of the service account. Response 204.

`message` is stored truncated to 1000 characters. The status report carries no identifier of the order; as today, the module assigns it by BPN and credential type. A late report for an earlier order therefore cannot be distinguished from one for the current order.

## 8. Steps for Process Orchestration

Common rules per FRAMEWORK 4.4: input with `processId` and `stepRunId`; result `{ "stepRunId", "state", "result" }`; errors as Problem Details with `retryable` (section 12); results without secrets.

### 8.1 `issue-credentials` (long-running)

In onboarding after `CompanyDidRegistered` (FRAMEWORK 5.2: dataspace wallet created, DID entered in the directory). In reactivation after `CompanyReactivated`, only for the membership credential (FRAMEWORK 5.5).

- **Input:** `companyId`, optional `credentialTypes` (subset of `BPN`, `MEMBERSHIP`; default `ONBOARDING_CREDENTIAL_TYPES`). For reactivation, Process Orchestration passes `["MEMBERSHIP"]`. Without it, the result would be the same, because the issued BPN credential is not triggered again; but `CredentialsIssued` would then name both types.
- **Preconditions:**
  - C3-M1: company with status `REQUESTED` (onboarding) or `ACTIVE` (reactivation) and with a BPN. Otherwise 409 `company-not-ready`. A deactivated company gets no credentials.
  - C4-M2: dataspace wallet of the company ready (`ACTIVE`), with a DID; for the route `CREDENTIAL_ISSUER` also with the location of the DID document. Otherwise 409 `wallet-not-ready`.
  - There is a route (6.3). Otherwise 409 `no-issuer-route`.
  - Route `CSP_ISSUER`: the DID is in the directory BPN → DID (`CompanyDidRegistered`). The module does not check this itself; if the entry is missing, the CSP refuses (409 `issuer-rejected`).
- **Behaviour:**
  1. Advisory lock on the `companyId`. Read the company at C3-M1 and the wallet at C4-M2 (11.1, 11.2), check the preconditions, determine the route.
  2. Per credential type:
     - `ISSUED`: nothing.
     - `REQUESTED`, younger than `RESEND_AFTER_HOURS`: nothing (waits for the status report).
     - `REVOCATION_PENDING`: 409 `invalid-state`, before anything is sent. OSP staff first confirm the revocation (7.3).
     - `REQUESTED` and older, `FAILED`, `CANCELLED`, `REVOKED` or no row yet: send the order through the route (11.3 to 11.6), `attempt` + 1. For all routes except `STUB`, the module fetches the holder access from C4-M2 once per run, immediately before the first order. For the routes `ISSUER_SERVICE` and `CSP_ISSUER`, it also registers the holder with the issuer once per run, before the first order. Success: store status `REQUESTED`, `requested_at` now, `route`, `issuer_did`, `wallet_provider`, `bpn`, `holder_did`, `request_ref`, in a separate transaction per credential type.
  3. If an order fails technically (issuer, CSP or wallet not reachable, sign-in at the CSP failed, unexpected response): 502 `upstream-error`. If the CSP issuer refuses the holder or the grant on business grounds: 409 `issuer-rejected`. Orders already sent stay `REQUESTED`; a repetition sends only the rest.
  4. Create the round in `issuance_run` or set it to `RUNNING` (credential types, `process_id`, `step_run_id`) if a type is not yet `ISSUED`.
  5. All types `ISSUED`: `DONE`, without an event (there already was one, or the types were issued before). Otherwise `RUNNING`.
- **Result:** `companyId`, per credential type `credentialType` and `status`.
- **Event at the end:** `CredentialsIssued` (all issued) or `CredentialIssuanceFailed` (per failed type). After that, Process Orchestration triggers the step again; it sends only the failed types again.

### 8.2 `revoke-credentials`

In the flow "deactivation" after `CompanyDeactivated` (FRAMEWORK 5.4).

- **Input:** `companyId`, `reason` (`COMPANY_DEACTIVATED` or `OTHER`), optional `credentialTypes` (default `["MEMBERSHIP"]`: only the membership credential; the BPN credential stays, OP-07).
- **Preconditions:** none; without credentials of the given types, the step is `DONE` immediately.
- **Behaviour:** Advisory lock on the `companyId`. Per given credential type:
  - `ISSUED`: revoke through the route of the order. Route can revoke: `REVOKED`, `CredentialRevoked`. Route cannot: `REVOCATION_PENDING`, no event. Technical error: 502 `upstream-error`; credentials already revoked stay revoked, a repetition does the rest.
  - `REQUESTED`, `FAILED`: `CANCELLED`, no event. With the route `CSP_ISSUER`, the module first removes the grant at the CSP, so that a request still running is no longer issued; if this fails technically, 502 `upstream-error`, and the credential keeps its status.
  - `CANCELLED`, `REVOCATION_PENDING`, `REVOKED`: nothing.
  - Running round: `CANCELLED`.
- **Result:** `DONE` with `revoked`, `cancelled`, `manualRevocation` (types in `REVOCATION_PENDING`). OSP staff find them through 7.2 and confirm them after the revocation at the Credential Issuer (7.3).
- **Event:** `CredentialRevoked` per revoked type.

## 9. Queries for other modules

In the same process without a token; over HTTP with the permission `query_c4_m1`.

### 9.1 `GET /internal/companies/{companyId}/credentials`

Per credential type of the company `credentialType`, `status`, `requestedAt`, `issuedAt`, `revokedAt`, plus the status of the last round. Unknown company: 200 with an empty list and round `null`.

## 10. Events

In the outbox, in the same transaction as the change. Every event also carries `eventId`, `eventType`, `occurredAt`. None contains secrets.

| Event | When | Content |
|---|---|---|
| `CredentialsIssued` | round changes to `DONE` (6.2) | `companyId`, `credentialTypes` |
| `CredentialIssuanceFailed` | status report `UNSUCCESSFUL` for an open order | `companyId`, `credentialType`, `reason` (message of the issuer, truncated to 200 characters) |
| `CredentialRevoked` | credential changes to `REVOKED` | `companyId`, `credentialType`, `revocationReason` |

The schemas are in [openapi.yaml](openapi.yaml) under `components/schemas` (names start with `Event`).

Use in the flows (Process Orchestration): `CredentialsIssued` → C3-M2 `register-membership` (FRAMEWORK 5.2, 5.5). `CredentialIssuanceFailed` → repeat or notice to OSP staff (OP-18).

## 11. Calls to other modules and external systems

Every neighbour has a built-in stand-in (FRAMEWORK 4.7): address empty = stand-in.

### 11.1 C3-M1 Company Profile (`COMPANY_URL`)

| Call | Response | Built-in stand-in |
|---|---|---|
| `get_company(companyId)`, over HTTP `GET /api/c3-m1/internal/companies/{companyId}` | `id`, `status`, `bpn`, `name` (name only for the holder in the own IssuerService) | reads `stub-companies.json`; unknown ID: status `REQUESTED`, BPN `BPNL` + first 12 hex characters of the `companyId` in upper case |

### 11.2 C4-M2 Dataspace Wallet Interface (`WALLET_URL`)

| Call | Response | Built-in stand-in |
|---|---|---|
| `get_wallet(companyId)`, over HTTP `GET /api/c4-m2/internal/wallets/{companyId}` | `companyId`, `status` (ready: `ACTIVE`), `provider`, `issuanceMode`, `did`, `didDocumentLocation`, `providerRef` | reads `stub-wallets.json`; unknown ID: `ACTIVE`, provider `STUB`, DID `did:web:wallet.example.org:{bpn}` |
| `get_holder_access(companyId)`, over HTTP `GET /api/c4-m2/internal/wallets/{companyId}/holder-access`; not for `STUB` | with `ISSUER_WRITES`: `holder` (location of the DID document), `walletUrl`, `clientId`, `clientSecret`; with `HOLDER_REQUESTS`: `identityApiUrl`, `participantContextId`, `apiKey` | fixed test values from `stub-wallets.json` |

The holder access stays only in memory and never appears in the log.

### 11.3 Route `CREDENTIAL_ISSUER` (own Credential Issuer)

Current behaviour (source: Portal 2.6.0 `IssuerComponent.Library`). Token for the access `CREDENTIAL_ISSUER_CLIENT_ID` via client credentials at `CREDENTIAL_ISSUER_TOKEN_URL`.

| Purpose | Call |
|---|---|
| BPN credential | `POST {CREDENTIAL_ISSUER_URL}/api/issuer/bpn` with `holder` (location of the DID document), `businessPartnerNumber`, `technicalUserDetails` (`walletUrl`, `clientId`, `clientSecret` from `get_holder_access`), `callbackUrl` = `{CALLBACK_BASE_URL}/api/c4-m1/issuer-callbacks/bpn` |
| Membership credential | `POST {CREDENTIAL_ISSUER_URL}/api/issuer/membership` with `holder`, `businessPartnerNumber`, `memberOf` = `dataspace.memberOf` from the settings, `technicalUserDetails`, `callbackUrl` = `…/issuer-callbacks/membership` |
| Status report | The Credential Issuer calls `callbackUrl` with `{ bpn, status, message }` (7.4), with its service account (6.5). |
| Revocation | not supported in 0.2 (6.3) |

Every 2xx response counts as accepted. `request_ref` and `issuer_did` stay empty.

### 11.4 Route `ISSUER_SERVICE` (own IssuerService)

Based on the documentation of the previous connection of the IdentityHub to the portal; check paths and fields against the version in use when building.

| Purpose | Call |
|---|---|
| Register the holder | administration interface of the own IssuerService (`ISSUER_SERVICE_ADMIN_URL`, key `ISSUER_SERVICE_API_KEY`), participant context `ISSUER_SERVICE_PARTICIPANT_ID`: create the holder with DID, holder ID = BPN, name (from C3-M1) and property `memberOf` = `dataspace.memberOf`. If it already exists, that counts as success; a differing DID is updated. |
| Request the credential | identity interface of the holder's dataspace wallet with the holder access from C4-M2 (`identityApiUrl`, `participantContextId`, header `x-api-key` = `apiKey`); applies to the IdentityHub and to the dataspace wallet of the CSP: credential request to `ISSUER_DID` with `holderPid` = `{companyId}:{credentialType}:{attempt}` and the credential definition from `ISSUER_SERVICE_CREDENTIAL_DEFINITIONS` in the format `CREDENTIAL_FORMAT`. `request_ref` = `holderPid`, `issuer_did` = `ISSUER_DID`. |
| Issuance | The holder's wallet fetches the credential from the IssuerService via DCP and stores it. The IssuerService takes the value for `memberOf` in the membership credential from the property of the holder. |
| Status report | The callback adapter of the wallet reports when the request reaches the state "issued" or "error", with `{ bpn, status, message }` (7.4); holder identifier = BPN. In the own IdentityHub it is an add-on; the dataspace wallet of the CSP reports itself to `{address of the installation}/api/c4-m1/issuer-callbacks/{credentialKind}`. Both with their own service account (6.5). |
| Revocation | administration interface of the IssuerService: search the holder's credentials of this type, revoke each one. |

The callback adapter for the IdentityHub that exists today calls fixed paths of the old portal (`/api/administration/registration/issuer/bpncredential` and `…/membershipcredential`). Until it is configurable, the ingress of the installation redirects these paths to `/api/c4-m1/issuer-callbacks/bpn` or `/membership`.

### 11.5 Route `CSP_ISSUER` (CSP issuer)

The issuer API of the CSP is a proposal, like the core services API in C3-M2. If the CSP defines it differently, only this adapter changes. All calls with the CSP's access token (sign-in per section 4); responses in JSON, errors as Problem Details.

| Call | Purpose | Responses |
|---|---|---|
| `PUT {CSP_ISSUER_URL}/holders/{bpn}` with `did` | Create the holder or update its DID. The CSP checks: the calling OSP is the home OSP of the BPN, and the DID equals the entry in the directory BPN → DID. | 200 (exists), 201 (new); 403 other home OSP; 409 DID does not match the directory |
| `PUT {CSP_ISSUER_URL}/holders/{bpn}/grants/{credentialKind}` with `callbackUrl` = `{CALLBACK_BASE_URL}/api/c4-m1/issuer-callbacks/{credentialKind}`, for `membership` also `memberOf` = `dataspace.memberOf` | Grant the credential type to the holder; the wallet may then request it via DCP. `credentialKind` is `bpn` or `membership`. | 204; 403 other home OSP; 404 holder unknown; 422 `memberOf` is not the dataspace of the CSP |
| `DELETE {CSP_ISSUER_URL}/holders/{bpn}/grants/{credentialKind}` | Remove the grant and revoke issued credentials of this type | 204, also if there was nothing to revoke |

Procedure of an order:

1. Register the holder (once per run).
2. Set the grant for the credential type.
3. Request the credential in the wallet as in 11.4, but to `CSP_ISSUER_DID` and with the identifier from `CSP_ISSUER_CREDENTIAL_DEFINITIONS`. `request_ref` = `holderPid`, `issuer_did` = `CSP_ISSUER_DID`.
4. The wallet fetches the credential from the CSP issuer via DCP.
5. The CSP issuer calls `callbackUrl` with `{ bpn, status, message }` as soon as it has handed the credential to the wallet or the issuance has failed (7.4). It uses its service account in the OSP's Portal IAM (6.5). The CSP calls only addresses under the address of the installation that is on the trust list for the OSP.

Responses 403, 404, 409 and 422 of the CSP give 409 `issuer-rejected` (`retryable: false`). Response 401 at the token endpoint or at the API, 5xx responses and timeouts give 502 `upstream-error` (`retryable: true`). The revocation (`DELETE`) with 204 sets the credential to `REVOKED`.

### 11.6 Route `STUB` (built-in stand-in)

- Order: remembers (company, credential type, BPN) in memory, `request_ref` = `stub-{attempt}`.
- After `STUB_ISSUER_DELAY_SECONDS` it calls the processing of 7.4 in the same process: `UNSUCCESSFUL` with `message` "Issuance declined by the stand-in" for types in `STUB_ISSUER_FAIL_TYPES`, otherwise `SUCCESSFUL`.
- Revocation: always successful.

With `ISSUER_MODE` = `CSP` and an empty `CSP_ISSUER_URL`, this stand-in applies. For the hackathon, the test tools provide a shared stand-in of the CSP for several installations (FRAMEWORK 4.7); if it offers the issuer API from 11.5, `CSP_ISSUER_URL` points to it.

### 11.7 Portal IAM

Only through the shared token validation library. No writing to Keycloak.

## 12. Errors

Problem Details (RFC 9457), `type` = `urn:c4-m1:problem:<name>`.

| `type` | HTTP | When | `retryable` for steps |
|---|---|---|---|
| `urn:c4-m1:problem:validation` | 400 | input invalid | `false` |
| `urn:c4-m1:problem:unauthorized` | 401 | token missing or invalid | – |
| `urn:c4-m1:problem:forbidden` | 403 | permission missing | – |
| `urn:c4-m1:problem:not-found` | 404 | credential or kind of status report unknown; company is not the person's company | – |
| `urn:c4-m1:problem:unknown-request` | 404 | status report without a matching order (BPN and type) | – |
| `urn:c4-m1:problem:invalid-state` | 409 | the status does not allow the operation | `false` |
| `urn:c4-m1:problem:company-not-ready` | 409 | company without BPN or not `REQUESTED`/`ACTIVE` | `false` |
| `urn:c4-m1:problem:wallet-not-ready` | 409 | dataspace wallet not ready or without DID | `false` |
| `urn:c4-m1:problem:no-issuer-route` | 409 | no route for issuer mode and wallet (6.3) | `false` |
| `urn:c4-m1:problem:issuer-rejected` | 409 | the CSP issuer refuses the holder or the grant (other home OSP, DID not in the directory, wrong dataspace) | `false` |
| `urn:c4-m1:problem:upstream-error` | 502 | issuer, CSP, wallet, C3-M1 or C4-M2 did not respond as expected | `true` |

## 13. Acceptance

The module is done when these cases work with built-in stand-ins for C3-M1, C4-M2 and the issuer (test data in section 14). Cases 12 and 13 run against a test server that emulates the Credential Issuer, or the IssuerService and the IdentityHub; cases 14 to 16 run against a test server of the CSP's issuer API (or the shared CSP stand-in of the test tools).

1. `issue-credentials` for U1: `RUNNING`, both types `REQUESTED`, `attempt` 1. After `STUB_ISSUER_DELAY_SECONDS` both `ISSUED`, exactly one `CredentialsIssued` with `BPN` and `MEMBERSHIP`, round `DONE`.
2. `issue-credentials` again for U1: `DONE`, no order, no event.
3. Repetition while `REQUESTED` (high delay): `RUNNING`, no second order. With `RESEND_AFTER_HOURS` = 0: new order, `attempt` 2.
4. `STUB_ISSUER_FAIL_TYPES` = `MEMBERSHIP`: `BPN` `ISSUED`, `MEMBERSHIP` `FAILED`, one `CredentialIssuanceFailed`, no `CredentialsIssued`, round `FAILED`. Empty the fail list, run the step again: only `MEMBERSHIP` is triggered again; then `CredentialsIssued`.
5. Company U2 without BPN: 409 `company-not-ready`. Company U3 with a wallet that is not `ACTIVE`: 409 `wallet-not-ready`. Wallet provider `BRING_YOUR_OWN`: 409 `no-issuer-route`. In no case is a row created; `retryable` is `false`.
6. Status report over HTTP with a service account with `report_credential_status` for an open order: 204, `ISSUED`, audit log entry with `azp` of the service account. The same report again: 204, no change, no event. Without the permission: 403. Unknown BPN: 404 `unknown-request`. `credentialKind` `foo`: 404.
7. `UNSUCCESSFUL` after `ISSUED`: 409. `SUCCESSFUL` after `FAILED`: `ISSUED`, and if this means all are issued, `CredentialsIssued`.
8. Person P1 (`company_id` U1, `view_credentials`) reads `GET /companies/U1/credentials`: 200, without `failureReason` and route. The same person for U4: 404. Person P3 without `company_id`: 404. Person P4 without the permission: 403.
9. `revoke-credentials` for U1 (route `STUB`) without `credentialTypes`: `MEMBERSHIP` `REVOKED`, one `CredentialRevoked`; `BPN` stays `ISSUED`. Repetition: no change, no event.
10. `revoke-credentials` while `REQUESTED`: `CANCELLED`, no event. A later success report: revoked immediately, `REVOKED`, one `CredentialRevoked`.
11. Revoke the membership credential through `CREDENTIAL_ISSUER` (test server): `REVOCATION_PENDING`, the result names it under `manualRevocation`. Person P2 (OSP staff, `manage_credentials`) finds it through `GET /credentials?status=REVOCATION_PENDING` and confirms (7.3): `REVOKED`, `CredentialRevoked`, audit log entry with `sub` of P2 and without names.
12. Route `CREDENTIAL_ISSUER` (`ISSUER_MODE` `OWN`, wallet provider `SAP_DIV`, test server): orders to `/api/issuer/bpn` and `/api/issuer/membership` with `holder`, BPN, `memberOf` from `SETTINGS_FILE`, `technicalUserDetails` and `callbackUrl`; the test server calls back; both `ISSUED`. The secret from `get_holder_access` is neither in the log nor in the database nor in an event or step result.
13. Route `ISSUER_SERVICE` (`ISSUER_MODE` `OWN`, wallet provider `IDENTITY_HUB`, test server): the holder is created once (second run: "exists" counts as success), credential request through the identity interface with `participantContextId` and `apiKey` from `get_holder_access`, `holderPid` `{companyId}:BPN:1`, issuer `ISSUER_DID`; the status report through the redirected old path arrives in 7.4. The same with wallet provider `CSP` (U6): the same calls to the identity interface from the holder access.
14. Route `CSP_ISSUER` (`ISSUER_MODE` `CSP`, wallet provider `CSP`, test server): the module fetches a token from the CSP; the client assertion is signed with the OSP's key, `iss` and `sub` are the `ospId` from `SETTINGS_FILE`. It calls `PUT /holders/{bpn}` with the DID, then per type `PUT …/grants/bpn` or `…/grants/membership` (with `memberOf`), then the credential request to `CSP_ISSUER_DID`. The test server calls back with the service account "CSP issuer": both `ISSUED`, `CredentialsIssued`, `issuerDid` = `CSP_ISSUER_DID`. Then `revoke-credentials`: `DELETE …/grants/membership`, `REVOKED`, `CredentialRevoked`. No holder access goes to the CSP.
15. `ISSUER_MODE` `CSP` with wallet provider `SAP_DIV` (U4): 409 `no-issuer-route`, no call to the CSP.
16. The CSP test server answers `PUT /holders/{bpn}` with 403 (other home OSP): 409 `issuer-rejected`, `retryable` `false`, no row `REQUESTED`. If it answers with 503: 502 `upstream-error`, `retryable` `true`.
17. Reactivation: after case 9 (U1: `BPN` `ISSUED`, `MEMBERSHIP` `REVOKED`), `issue-credentials` with `credentialTypes` `["MEMBERSHIP"]` and U1 in status `ACTIVE`: only `MEMBERSHIP` is triggered, `attempt` 2; then `CredentialsIssued` with `["MEMBERSHIP"]`. `BPN` stays unchanged.
18. U5 in status `INACTIVE` (deactivated): `issue-credentials` 409 `company-not-ready`.
19. Issuer not reachable: 502 `upstream-error`, the credential does not become `REQUESTED`; after the fix, a repetition triggers it.
20. `GET /internal/companies/{companyId}/credentials` for an unknown company: 200 with an empty list.

## 14. Test data

Test IDs per FRAMEWORK 7a; further ones are defined here.

The test Portal IAM needs:

- P1 "Erika Mustermann" (test person per FRAMEWORK 7a), `company_id` U1, portal role `Company Admin` with `view_credentials`
- P2 OSP staff, portal role `OSP Admin` with `manage_credentials`
- P3 person without a company (portal role `Applicant`)
- P4 person of U1 with portal role `Member` without `view_credentials`
- service accounts `sa-credential-issuer` (own Credential Issuer), `sa-ih-callback` (callback adapter), `sa-csp-issuer` (CSP issuer), each with `report_credential_status`
- a client for HTTP calls of other modules with `query_c4_m1` and `execute_steps_c4_m1`

Files:

- `installation.json` (`SETTINGS_FILE`): `ospId` of OSP A `22222222-2222-4222-8222-222222222222` (or, for cases 14 to 16, OSP B `99a5685c-7397-402c-88de-b1b3a939ecc9`), `dataspace.memberOf` `catena-x`, `dataspace.name` `Catena-X` (example values, OP-14).
- `stub-companies.json`: U1 `c0000001-0000-4000-8000-000000000001` (Beispiel Maschinenbau GmbH, FRAMEWORK 7a, `REQUESTED`, BPN `BPNLIUK2REMLK3F4`; for case 17 `ACTIVE`), and without names only for the tests of this module: U2 `c4100002-0000-4000-8000-000000000002` (`REQUESTED`, without BPN), U3 `c4100003-0000-4000-8000-000000000003` (`REQUESTED`, BPN `BPNL0000000000U3`), U4 `c4100004-0000-4000-8000-000000000004` (`ACTIVE`, BPN `BPNL0000000000U4`), U5 `c4100005-0000-4000-8000-000000000005` (`ACTIVE`, BPN `BPNL0000000000U5`; for case 18 `INACTIVE`), U6 `c4100006-0000-4000-8000-000000000006` (`REQUESTED`, BPN `BPNL0000000000U6`).
- `stub-wallets.json`: U1 `ACTIVE`, provider `STUB`, DID `did:web:wallet.example.org:BPNLIUK2REMLK3F4`; U3 not ready (`REQUESTED`); U4 `ACTIVE`, provider `SAP_DIV`, issuance mode `ISSUER_WRITES`, location of the DID document `https://wallet.example.org/BPNL0000000000U4/did.json`, holder access with test values; U5 `ACTIVE`, provider `IDENTITY_HUB`, `HOLDER_REQUESTS`, `providerRef` `BPNL0000000000U5`, holder access with a test API key; U6 `ACTIVE`, provider `CSP`, `HOLDER_REQUESTS`, DID `did:web:csp-wallet.example.org:BPNL0000000000U6`, holder access with a test API key; one more company with provider `BRING_YOUR_OWN`, `CREDENTIAL_OFFER`.
- Key of the OSP (`OSP_SIGNING_KEY_FILE`): a key generated in the test (ES256); the CSP test server knows the public part for the `ospId`.
- For cases 12 and 13: `ISSUER_MODE` = `OWN`, `ISSUER_ROUTES` = `SAP_DIV:CREDENTIAL_ISSUER,IDENTITY_HUB:ISSUER_SERVICE,CSP:ISSUER_SERVICE` and the addresses of the test servers.
- For cases 14 to 16: `ISSUER_MODE` = `CSP`, `CSP_ISSUER_URL` and `CSP_TOKEN_URL` pointing to the test server, `CSP_ISSUER_DID` = `did:web:issuer.csp.example.org`.

Hackathon scenario: installation of OSP A with `ISSUER_MODE` = `OWN` (own IssuerService, `ISSUER_ROUTES` = `IDENTITY_HUB:ISSUER_SERVICE,CSP:ISSUER_SERVICE`); installation of OSP B with `ISSUER_MODE` = `CSP`.

## 15. Changes from version 0.1

- **OSP 2.0:** The module runs in the installation of each OSP. The issuer is configuration (`ISSUER_MODE`): own issuer of the OSP or CSP issuer. New route `CSP_ISSUER` with the CSP's issuer API as a proposal (11.5) and sign-in at the CSP via `private_key_jwt` with the OSP's key (4).
- **Trust:** An own issuer must be on the CSP's trust list (6.4); the CSP issuer checks the home OSP and the directory BPN → DID. That is why `issue-credentials` runs after `CompanyDidRegistered`.
- **Route `ISSUER_SERVICE`** also applies to the dataspace wallet of the CSP. The request in the wallet uses the holder access from C4-M2; `IDENTITYHUB_IDENTITY_URL` and `IDENTITYHUB_API_KEY` are dropped.
- **`memberOf`** comes from the installation settings (`dataspace.memberOf`, FRAMEWORK 4.8, OP-14) instead of `MEMBER_OF`; new also at the IssuerService (property of the holder) and at the CSP (grant).
- **Deactivation and reactivation:** Without input, `revoke-credentials` revokes only the membership credential (OP-07) and is now in the step catalogue. `issue-credentials` also runs at reactivation, only for the membership credential (OP-08).
- **Callers:** People of the company read the credentials of their company (scope `company`, `GET /companies/{companyId}/credentials`); OSP staff read all of them and confirm revocations (scope `osp`, `GET /credentials`, `POST /credentials/{credentialId}/confirm-revocation` instead of `/operator/...`). Tags `Company`, `OSP`, `Issuer`, `Queries`, `Steps`; `x-permission-scope` per operation.
- **Status reports:** one service account per issuer and callback adapter with exactly the permission `report_credential_status` (OP-22), also for the issuer and the dataspace wallet of the CSP (6.5).
- **Dropped:** delegation (scope `credentials`, header `X-On-Behalf-Of`, `check_delegation`, `DELEGATION_URL`, error `not-delegated`), header `X-Acting-Person`, `psp_id` in the history, status reports to PSPs, all references to PSPs and operators.
- **Audit log** per FRAMEWORK 4.3 in the new table `audit_entry` with `sub` instead of a name.
- **Errors** with `retryable` (OP-18); new error `issuer-rejected`.
- **Data:** new field `issuer_did`; wallet provider `CSP`; route `CSP_ISSUER`.
- **Test data** with the shared test IDs (FRAMEWORK 7a); new cases 14 to 18.
