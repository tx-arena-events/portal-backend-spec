# C4-M2 Dataspace Wallet Interface

**Version:** 0.2, as of 2026-10-28
**Capability:** C4 Dataspace
**Kind:** Hackathon
**Files:** this README (behaviour), [openapi.yaml](openapi.yaml) (interface), [open-points.md](open-points.md)

This README and the OpenAPI description are sufficient to build the module. The binding framework is [../FRAMEWORK.md](../FRAMEWORK.md). Modules from other parts of the spec are referred to by name. Paths, field names and descriptions are in English.

## 1. Purpose

The module connects the installation of the onboarding service provider (OSP) with the dataspace wallets of the companies. It

- creates a dataspace wallet per company and checks the DID and the DID document,
- creates and deletes wallet accounts for technical users,
- keeps the credentials of a wallet account encrypted until a person of the company retrieves them once,
- gives C4-M1 Credentials the holder access for issuance,
- receives the callbacks of the providers.

C3-M2 CSP Connection makes the entry BPN → DID in the directory of the CSP with the step `register-did`, after this module has reported `DataspaceWalletCreated` (FRAMEWORK 5.2).

The module is provider-neutral. Behind a common interface there is one adapter per provider. The configuration of an installation decides which providers it offers (`ENABLED_PROVIDERS`). Several adapters can be active at the same time (parallel operation: company A at SAP DIV, company B in the dataspace wallet of the CSP).

| Adapter | Provider | Path | Status |
|---|---|---|---|
| `SAP_DIV` | SAP DIV (formerly DIM), today's path in Portal 2.6.0 | order to the SAP middleware, result by callback | first version |
| `IDENTITY_HUB` | Tractus-X IdentityHub (from the EDC IdentityHub), operated by the OSP; the template is the existing connection of the IdentityHub to the portal | participant context through the Identity API, immediate response | first version |
| `CSP` | dataspace wallet of the CSP, an optional service of the CSP | wallet through the wallet API of the CSP (proposal), sign-in with the key of the OSP, immediate response | first version |
| `BRING_YOUR_OWN` | own wallet of a company | The company brings its DID and credential service and proves that it controls the DID | later (OP-29) |

Callers:

| Who | What for |
|---|---|
| Process Orchestration | steps `create-wallet`, `create-wallet-account`, `delete-wallet-account` (flows FRAMEWORK 5.2 and 5.7) |
| People of a company (through the UI, scope `company`) | read the dataspace wallet and the wallet accounts of their own company, retrieve the credentials of a wallet account once |
| Applicant (through the UI, scope `own`) | later: prove control over an own wallet |
| SAP DIV (middleware, service account in the Portal IAM of the OSP) | callbacks: wallet created, wallet account created, wallet account deleted |
| C4-M1 Credentials | queries: wallet of the company, holder access for issuing the credentials |
| C2-M3 Technical Users, connectors (other part) | queries: wallet of the company, wallet account of a technical user |

## 2. Terms

| Term | Meaning |
|---|---|
| Dataspace wallet | Wallet of the company for the data exchange of the connectors (DCP). It holds the BPN credential and the membership credential. Not to be confused with the EUDI Wallet of the person and the Business Wallet of the company, with which people and companies identify themselves; C5-M1 checks those. |
| Provider, adapter | The provider operates the dataspace wallet (SAP, the OSP with its IdentityHub, the CSP, the company itself). The adapter is the part of this module that talks to exactly one provider. |
| DID | Identifier of the dataspace wallet, always `did:web`, e.g. `did:web:ih.example.org:BPNL00000000IH01` |
| DID document, DID document location | public JSON document for the DID with keys and services; location per the rules of `did:web`, e.g. `https://ih.example.org/BPNL00000000IH01/did.json` |
| Credential service | DCP endpoint of the wallet through which connectors query credentials. Listed in the DID document as a service of type `CredentialService`. |
| Directory BPN → DID | core service of the CSP (today BDRS, BPN-DID Resolution Service). Connectors use it to find the DID of a partner. C3-M2 makes the entry. |
| Wallet account | Access of a technical user (C2-M3) to the dataspace wallet of its company: token address, client ID, secret. With it, a connector fetches tokens from the Secure Token Service (STS) of the wallet. At SAP DIV this is called "technical user in DIM" today. |
| `walletAccountRef` | Identifier of a wallet account in this module (UUID). It is part of the events; C2-M3 stores it. |
| Holder access | Access to the wallet of the company through which credentials get into it. At SAP DIV the issuer uses it to write into the wallet (today `company_wallet_data`); with the IdentityHub and the dataspace wallet of the CSP, C4-M1 uses it to trigger the request to the issuer. |
| Participant context | Unit in the IdentityHub that bundles the DID, keys and credentials of a participant; the dataspace wallet of the CSP offers the same unit |
| Wallet API of the CSP | Interface of the dataspace wallet of the CSP for the OSPs (proposal, 11.3) |
| Control proof | JWT that a company signs with the key from the DID document of its own wallet. It proves that the company controls the DID (later). |
| Issuance mode | how credentials get into the wallet: `ISSUER_WRITES` (SAP DIV), `HOLDER_REQUESTS` (IdentityHub, dataspace wallet of the CSP), `CREDENTIAL_OFFER` (own wallet, later); section 9.3 |
| `ospId` | Identifier of the OSP at the CSP, from the installation settings (FRAMEWORK 4.8) |

## 3. Not part of this module

| What | Where |
|---|---|
| EUDI Wallet and Business Wallet, checking PID, LPID and mandate | C5-M1 EBW-Connector |
| Issue and track the BPN credential and the membership credential; make the holder known to the issuer; trigger the request in the wallet; status report "credential issued" | C4-M1 Credentials |
| Entry BPN → DID in the directory of the CSP | C3-M2 CSP Connection (`register-did`) |
| Assign the BPN | CSP through C3-M2; in the installation C3-M1 Company Profile |
| Technical user in the Portal IAM, wallet role | C2-M3 Technical Users |
| Order of the steps, re-triggering, runtime monitoring of the flows | Process Orchestration |
| Register a connector and link it to a technical user | Connectors (other part) |
| Operation of the wallets | SAP, OSP (IdentityHub), CSP or the company (own wallet) |
| Take the dataspace wallet along on a transfer to another OSP | later work (FRAMEWORK 5.6) |

## 4. Technical frame

- Python with FastAPI, part of the modular monolith. Process Orchestration and other modules call the module in the same process through its public interface. [openapi.yaml](openapi.yaml) describes the same contract as HTTP paths.
- Own database `c4_m2` in PostgreSQL with its own database user. The module reads and writes only there.
- HTTP calls carry a token of the Portal IAM of the OSP; the shared library validates it and provides the claims (`sub`, `company_id`, permissions, `azp`). Scope `company`: `company_id` in the token equals the company of the object, otherwise 404 (FRAMEWORK 4.3).
- **Sign-in at the wallet API of the CSP:** OAuth 2.0 client credentials with `private_key_jwt` (RFC 7523) at the token endpoint of the CSP, as in C3-M2. `client_id` is the `ospId`; the OSP signs with its key, whose public part is in the trust list. No secret. The access token of the CSP is reused until shortly before it expires. Timeout per call 10 seconds.
- **Callbacks:** Only SAP DIV calls this module back, with a service account in the Portal IAM of the OSP that has exactly the permission `submit_wallet_callback` (FRAMEWORK 4.2). The IdentityHub and the dataspace wallet of the CSP respond immediately and do not call back. The dataspace wallet of the CSP reports arrived credentials to C4-M1, not to this module.
- Events through the outbox of the module, in the same transaction as the change.
- Secrets (credentials, API keys) are encrypted with AES-256-GCM, key `SECRETS_KEY`, key version per row. Never in logs, events, step results or audit log entries. The HTTP clients of the adapters do not write bodies to the log. The key of the OSP is only in the configuration.
- Scheduled tasks run as a background task of the module (6.10). They only talk to the DID resolution and check deadlines. Process Orchestration does not poll the module.
- Several instances: steps and callbacks lock per company or per technical user with an advisory lock in PostgreSQL; scheduled tasks fetch their rows with `SELECT … FOR UPDATE SKIP LOCKED`.

### Configuration

| Variable | Meaning | Example |
|---|---|---|
| `DATABASE_URL` | own PostgreSQL database `c4_m2` | |
| `SETTINGS_FILE` | installation settings (FRAMEWORK 4.8): `ospId` for the sign-in at the CSP | `config/installation.json` |
| `ENABLED_PROVIDERS` | providers the installation offers, comma-separated; `BRING_YOUR_OWN` only later | `SAP_DIV,IDENTITY_HUB,CSP` |
| `DEFAULT_PROVIDER` | provider when the step names none and there is no control proof (`SAP_DIV`, `IDENTITY_HUB` or `CSP`) | `CSP` |
| `SECRETS_KEY`, `SECRETS_KEY_VERSION` | key (AES-256, hex) and its version for all secrets of the module; old versions stay readable through `SECRETS_KEY_<version>` | |
| `SAP_DIV_URL` | base address of the SAP middleware; empty = built-in stand-in (11.1) | `https://div.example.org/api/dim` |
| `SAP_DIV_TOKEN_URL`, `SAP_DIV_CLIENT_ID`, `SAP_DIV_CLIENT_SECRET` | access of the module to the middleware (client credentials) | |
| `SAP_DIV_DID_DOCUMENT_BASE_URL` | base of the DID document locations at SAP DIV | `https://div.example.org/did` |
| `SAP_DIV_CALLBACK_CLIENT_ID` | service account in the Portal IAM of the OSP with which the middleware calls back | `sa-div-callback` |
| `IDENTITYHUB_URL` | base address of the Identity API of the OSP's own IdentityHub; empty = built-in stand-in (11.2) | `https://ih-admin.example.org/api/identity` |
| `IDENTITYHUB_API_KEY` | management key of the Identity API | |
| `IDENTITYHUB_DID_BASE` | host (and path, if any) of the DIDs in the IdentityHub | `ih.example.org` |
| `IDENTITYHUB_CREDENTIAL_SERVICE_URL` | base of the credential service | `https://ih.example.org/api/credentials` |
| `IDENTITYHUB_STS_TOKEN_URL` | token address of the STS | `https://ih.example.org/api/sts/token` |
| `IDENTITYHUB_CONTEXT_ID_ENCODING` | identifier of the participant context in paths: `plain` or `base64` | `plain` |
| `CSP_WALLET_URL` | base address of the wallet API of the CSP; empty = built-in stand-in (11.3) | `https://wallet.csp.example.org/api/v1` |
| `CSP_TOKEN_URL` | token endpoint of the CSP | `https://core.example.org/oauth/token` |
| `OSP_SIGNING_KEY_FILE` | private key of the OSP (JWK, ES256) for `private_key_jwt`; the same as in C3-M2 | `osp-key.json` |
| `CONTROL_PROOF_AUDIENCE` | later: expected audience (`aud`) in the control proof | `https://portal.osp-a.example.org/api/c4-m2` |
| `CONTROL_CHALLENGE_TTL_MINUTES` | later: validity of a control proof challenge | `15` |
| `CONTROL_PROOF_MAX_AGE_DAYS` | later: how long a verified control proof is valid for `create-wallet` | `30` |
| `DID_RESOLVER_URL` | Universal Resolver; empty = built-in `did:web` resolution (11.4) | `https://resolver.example.org` |
| `TEST_DID_DOCUMENTS_DIR` | test only: directory with DID documents that are used before any resolution | `testdata/dids` |
| `DID_VALIDATION_INTERVAL_MINUTES`, `DID_VALIDATION_MAX_HOURS` | interval and deadline of the DID check | `10`, `168` |
| `BDRS_PUBLIC_URL` | address of the directory BPN → DID for connectors (display only, 7.1) | `https://bdrs.example.org/api/directory` |
| `PROVIDER_CALLBACK_TIMEOUT_MINUTES` | deadline for callbacks from SAP DIV | `1440` |
| `ACCOUNT_CREDENTIALS_RETENTION_DAYS` | how long credentials that were not retrieved are kept | `30` |
| `TIMER_INTERVAL_SECONDS` | interval of the scheduled tasks | `60` |
| `COMPANY_URL` | C3-M1: empty = stand-in from `stub-companies.json`, `in-process` = call in the same process, otherwise HTTP address (11.5) | `in-process` |
| `APPLICATION_URL` | C1-M2, only for the own wallet (later): as above, stand-in from `stub-applications.json` (11.6) | `in-process` |
| `STUB_CALLBACK_DELAY_SECONDS` | delay of the callbacks in the SAP DIV stand-in | `1` |
| `STUB_FAILURES` | failures that the stand-in providers simulate: `<BPN or technicalUserId>:<failure>`, comma-separated (14.3) | `BPNL000000FAIL01:DID_MISMATCH` |

## 5. Data

### Table `dataspace_wallet`

One row per company. A new attempt after `FAILED` uses the same row.

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key |
| `company_id` | uuid | unique; required |
| `bpn` | text | required; unique; `^BPNL[0-9A-Z]{12}$`; copy from C3-M1, needed for callbacks and orders |
| `provider` | text | `SAP_DIV`, `IDENTITY_HUB`, `CSP`, `BRING_YOUR_OWN` |
| `status` | text | `REQUESTED`, `VALIDATING`, `ACTIVE`, `FAILED` (6.2) |
| `failure_reason` | text | only with `FAILED` (section 10) |
| `did` | text | unique when set |
| `did_document_location` | text | derived from the DID |
| `credential_service_url` | text | from the DID document (service `CredentialService`), otherwise empty |
| `provider_ref` | text | identifier at the provider: for SAP DIV the BPN, for IdentityHub and CSP the identifier of the participant context |
| `control_challenge_id` | uuid | with `BRING_YOUR_OWN`: the verified control proof |
| `attempt` | int | counter of the attempts, starting at 1 |
| `provider_ordered_at` | timestamptz | order accepted by the provider; empty = still to be sent |
| `validating_since`, `did_validated_at`, `next_check_at` | timestamptz | progress of the DID check (6.4) |
| `activated_at`, `created_at`, `changed_at` | timestamptz | |

### Table `wallet_secret`

Accesses of the wallet itself, encrypted.

| Field | Type | Rule |
|---|---|---|
| `wallet_id` | uuid | part of the primary key |
| `kind` | text | part of the primary key: `HOLDER_ACCESS` (holder access) or `STS_ACCESS` (STS access of the participant context, IdentityHub only) |
| `url` | text | SAP DIV: `authenticationServiceUrl`; IdentityHub and CSP: Identity API or STS token address |
| `client_id` | text | SAP DIV: `clientID`; IdentityHub STS: `clientId`; IdentityHub and CSP holder access: identifier of the participant context |
| `secret_encrypted`, `nonce` | bytea | secret or API key, AES-256-GCM |
| `key_version` | int | version of `SECRETS_KEY` |
| `created_at` | timestamptz | |

### Table `wallet_account`

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key; is the `walletAccountRef` |
| `technical_user_id` | uuid | required; at most one row per technical user with a status other than `DELETED` (partial unique index) |
| `company_id` | uuid | required |
| `wallet_id` | uuid | required; wallet of the company |
| `provider` | text | copy from the wallet |
| `status` | text | `REQUESTED`, `ACTIVE`, `FAILED`, `DELETING`, `DELETED` (6.3) |
| `failure_reason` | text | only with `FAILED` |
| `provider_name` | text | name at the provider (SAP DIV and CSP: `tu-` + `technical_user_id`) |
| `provider_ordered_at`, `deletion_ordered_at` | timestamptz | order accepted |
| `credentials_state` | text | `NONE`, `AVAILABLE`, `RETRIEVED`, `EXPIRED` |
| `created_at`, `changed_at` | timestamptz | |

### Table `wallet_account_credentials`

Credentials of a wallet account until they are retrieved once. The row is deleted on retrieval, when the account is deleted or after `ACCOUNT_CREDENTIALS_RETENTION_DAYS`.

| Field | Type |
|---|---|
| `wallet_account_id` | uuid, primary key |
| `token_url` | text |
| `client_id` | text |
| `client_secret_encrypted`, `nonce` | bytea |
| `key_version` | int |
| `created_at`, `expires_at` | timestamptz |

### Table `wallet_control_challenge` (later)

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key |
| `created_by_sub` | text | `sub` of the applicant who created the challenge |
| `application_id` | uuid | application at C1-M2 |
| `company_id` | uuid | future `companyId` from the application |
| `did` | text | `did:web:…` |
| `credential_service_url` | text | `https://…` |
| `nonce` | text | 128 bit random, base64url |
| `status` | text | `OPEN`, `VERIFIED`, `EXPIRED`, `USED` |
| `expires_at`, `verified_at`, `created_at` | timestamptz | |

### Tables `audit_log`, `step_log`, `outbox`

- `audit_log`: audit log per FRAMEWORK 4.3. For each changing operation over HTTP (retrieving credentials, callback, control proof) and each retrieval of the holder access: time, `sub` of the person or the service account, `azp`, `company_id`, operation, object ID. In the same process without a token, the entry holds the name of the calling module. No secrets, no names.
- `step_log`: `step_run_id`, `process_id`, step, object ID, state answered, time. For traceability only (FRAMEWORK 4.4).
- `outbox`: events until delivery.

**Not stored:** DID documents (public, resolvable at any time), control proofs as JWT, company name (only in the order to SAP DIV), roles, credentials in the wallet, secrets in plain text, access tokens of the CSP beyond memory.

## 6. Rules

### 6.1 Choosing the provider

The step `create-wallet` chooses the provider in this order:

1. If there is a verified control proof (`VERIFIED`) for the company: `BRING_YOUR_OWN` (later).
2. If the failed last attempt was `BRING_YOUR_OWN` and there is no new control proof: 409 `control-proof-required`. A used proof does not count again; the applicant provides the proof again (7.4, 7.5). This way an own wallet never switches to another provider unnoticed.
3. If the input names `provider`: this provider.
4. Otherwise `DEFAULT_PROVIDER`.

A provider that is not in `ENABLED_PROVIDERS` gives 400 `provider-not-enabled`. The OSP decides between SAP DIV, IdentityHub and the dataspace wallet of the CSP through the configuration, not the company. As long as `BRING_YOUR_OWN` is not active, items 1 and 2 do not apply.

### 6.2 Status of the dataspace wallet

| From | Event | To | Event of the module |
|---|---|---|---|
| – | `create-wallet`, SAP DIV | `REQUESTED` | – |
| – | `create-wallet`, IdentityHub, CSP or own wallet | `VALIDATING` | – |
| `REQUESTED` | callback with valid DID and valid DID document | `VALIDATING` | – |
| `REQUESTED` | callback with wrong DID or invalid DID document | `FAILED` | `DataspaceWalletFailed` |
| `REQUESTED` | no callback within `PROVIDER_CALLBACK_TIMEOUT_MINUTES` | `FAILED` | `DataspaceWalletFailed` |
| `VALIDATING` | DID document checked (6.4) | `ACTIVE` | `DataspaceWalletCreated` |
| `VALIDATING` | content of the DID document invalid or deadline of the DID check passed | `FAILED` | `DataspaceWalletFailed` |
| `FAILED` | `create-wallet` again | `REQUESTED` or `VALIDATING`, `attempt` + 1 | – |

`ACTIVE` is final in version 0.2, even when the company is deactivated (FRAMEWORK 5.4). A new attempt deletes the rows in `wallet_secret`, clears `did` and the progress fields and can change the provider (6.1).

### 6.3 Status of a wallet account

| From | Event | To | Event of the module |
|---|---|---|---|
| – | `create-wallet-account`, SAP DIV | `REQUESTED` | – |
| – | `create-wallet-account`, IdentityHub or CSP | `ACTIVE` | `WalletAccountCreated` |
| – | `create-wallet-account`, own wallet | `FAILED` (`NOT_SUPPORTED_BY_PROVIDER`) | `WalletAccountFailed` |
| – | `create-wallet-account`, company not `ACTIVE` | no account; 409 `company-not-active` | – |
| `REQUESTED` | callback "created" | `ACTIVE` | `WalletAccountCreated` |
| `REQUESTED` | no callback within the deadline | `FAILED` (`PROVIDER_TIMEOUT`) | `WalletAccountFailed` |
| `ACTIVE` | `delete-wallet-account`, SAP DIV | `DELETING` | – |
| `ACTIVE` | `delete-wallet-account`, IdentityHub or CSP | `DELETED` | `WalletAccountDeleted` |
| `FAILED` | `delete-wallet-account` | `DELETED` | `WalletAccountDeleted` |
| `DELETING` | callback "deleted" | `DELETED` | `WalletAccountDeleted` |
| `DELETING` | no callback within the deadline | `DELETING` | `WalletAccountDeletionFailed` |

On the transition to `DELETED`, the row in `wallet_account_credentials` is deleted.

**IdentityHub:** The IdentityHub has one STS access per participant context. Every wallet account of a company gets a copy of this access (client ID and secret from `wallet_secret`, kind `STS_ACCESS`). Deleting a wallet account changes nothing at the IdentityHub; the access stays valid as long as a connector knows it.

**Dataspace wallet of the CSP:** It creates a separate access per wallet account. Deleting the wallet account revokes it at the CSP.

**Deactivated company:** No new wallet account is created for a company that is not `ACTIVE` at C3-M1 (FRAMEWORK 5.4, OP-07). Existing accounts remain; deletion is still possible.

### 6.4 Checking the DID and the DID document

The check starts immediately on entering `VALIDATING` and repeats every `DID_VALIDATION_INTERVAL_MINUTES` until the deadline `DID_VALIDATION_MAX_HOURS`, counted from `validating_since`.

1. **Form of the DID** (before resolution):
   - SAP DIV: exactly `did:web:` + host and path from `SAP_DIV_DID_DOCUMENT_BASE_URL` (without `https://`, `/` replaced by `:`) + `:` + BPN; case does not matter. This is how Portal 2.6.0 checks today. Deviation: `FAILED`, `DID_MISMATCH`, without further attempts.
   - IdentityHub: exactly `did:web:{IDENTITYHUB_DID_BASE}:{BPN}`.
   - Dataspace wallet of the CSP: the DID from the response of the wallet API; only the form `did:web:` is checked, the CSP decides host and path.
   - Own wallet: the DID from the control proof.
2. **Resolve** through `DID_RESOLVER_URL` or the built-in resolution (11.4). No result: next attempt at the next interval; after the deadline `FAILED`, `DID_NOT_RESOLVABLE`.
3. **Content**, otherwise `FAILED`, `INVALID_DID_DOCUMENT`, without further attempts:
   - `id` equals the DID,
   - at least one entry in `verificationMethod` with `publicKeyJwk` (`kty` `EC` with `crv`, `x`, `y` or `kty` `OKP` with `crv`, `x`),
   - IdentityHub, CSP and own wallet: a service of type `CredentialService`; for an own wallet with exactly the address from the control proof.
4. In one transaction: set `did_validated_at`, take `credential_service_url` from the service `CredentialService` (if present), status `ACTIVE`, `activated_at`, event `DataspaceWalletCreated`.

### 6.5 Directory BPN → DID

No longer in this module since version 0.2. C3-M2 makes the entry with the step `register-did`; Process Orchestration calls it after `DataspaceWalletCreated` with the DID from the event (FRAMEWORK 5.2). The wallet is `ACTIVE` as soon as the DID document is checked (6.4).

### 6.6 Control proof of an own wallet (later)

Applies only with the own wallet (OP-29); not to be implemented in the first version. A company with an own wallet proves before creation that it controls the DID. The applicant does this during onboarding, after C1-M2 has assigned the future `companyId` (FRAMEWORK 5.2, step 3).

1. The person creates a challenge (7.4): `applicationId`, `did`, `credentialServiceUrl`. The module reads the application at C1-M2 (11.6); if it does not belong to the person (`sub`), 404. The `companyId` comes from the application. The module resolves the DID document and checks it as in 6.4 step 3. It responds with `nonce`, `audience`, `companyId` and expiry time.
2. The company signs a JWT with a key from its DID document:
   - Header: `alg` `ES256`, `ES384` or `EdDSA`; `kid` = full identifier of the key in the DID document (`<DID>#<fragment>`).
   - Claims: `iss` and `sub` = DID, `aud` = `CONTROL_PROOF_AUDIENCE`, `nonce` = nonce of the challenge, `company_id` = `companyId`, `iat`, `exp` (at most 10 minutes after `iat`).
3. The person submits the JWT (7.5). The module checks:
   - challenge `OPEN`, not expired, created by the same person (`sub`),
   - resolve the DID document again; `kid` is in `verificationMethod` and in `authentication` or `assertionMethod`,
   - signature with the `publicKeyJwk` of this key,
   - `iss`, `sub`, `aud`, `nonce`, `company_id` as expected; `exp` not passed; `iat` not more than 5 minutes in the future,
   - the service `CredentialService` has exactly the given address,
   - no other wallet (status other than `FAILED`) uses this DID.
4. Success: status `VERIFIED`. Earlier `VERIFIED` challenges of the same `companyId` become `EXPIRED`. Only the latest proof counts.
5. `create-wallet` uses up the proof (status `USED`) if it is at most `CONTROL_PROOF_MAX_AGE_DAYS` old.

Only the applicant can provide a proof for the `companyId` of their application. This way nobody can slip another company a wallet that they control themselves. The procedure follows the pattern of a self-issued ID token in DCP. It proves control over the key of the DID, not that the credential service is reachable.

### 6.7 Credentials and other secrets

- Credentials of a wallet account are only in `wallet_account_credentials`, encrypted. They are handed out exactly once to a person of the company with the permission `retrieve_wallet_credentials` (scope `company`, 7.3). After that, the row is deleted and `credentials_state` is `RETRIEVED`.
- The module deletes credentials that were not retrieved after `ACCOUNT_CREDENTIALS_RETENTION_DAYS` (`credentials_state` `EXPIRED`). Whoever lost them deletes the technical user with wallet access and creates it again (C2-M3).
- The holder access (`wallet_secret`, kind `HOLDER_ACCESS`) is handed out only to C4-M1 (9.3), as often as C4-M1 needs it, never to people.
- Events, step results, `step_log`, `audit_log` and logs never contain secrets.

### 6.8 Permissions and service accounts

| Operation | Permission | Scope |
|---|---|---|
| `GET /companies/{companyId}/wallet` | `view_dataspace_wallet` | `company` |
| `GET /technical-users/{technicalUserId}/wallet-account` | `view_dataspace_wallet` | `company` |
| `POST /technical-users/{technicalUserId}/wallet-account/credentials` | `retrieve_wallet_credentials` | `company` |
| `POST /wallet-control-challenges`, `POST /wallet-control-challenges/{challengeId}/proof` (later) | `prove_wallet_control` | `own`; bound to the application of the person (6.6) |
| `POST /callbacks/sap-div/...` | `submit_wallet_callback`, plus client `SAP_DIV_CALLBACK_CLIENT_ID` | `osp` |
| all paths under `/steps` | `execute_steps_c4_m2` (HTTP only) | `osp` |
| `GET /internal/wallets/{companyId}`, `GET /internal/technical-users/{technicalUserId}/wallet-account` | `query_c4_m2` (HTTP only) | `osp` |
| `GET /internal/wallets/{companyId}/holder-access` | `read_wallet_holder_access` (HTTP only); in the same process only for C4-M1 | `osp` |

For service accounts, steps and queries, scope `osp` means: applies to all companies of the installation; only the permission is checked.

Proposal to C2-M5: `view_dataspace_wallet` and `retrieve_wallet_credentials` belong to the portal roles `IT Admin` and `Company Admin`; `prove_wallet_control` later to `Applicant`.

Service accounts for callbacks (one per system, exactly one permission; FRAMEWORK 4.2):

| System | Permission | Check | Sign-in at the Portal IAM |
|---|---|---|---|
| SAP middleware (SAP DIV) | `submit_wallet_callback` | additionally `azp` equals `SAP_DIV_CALLBACK_CLIENT_ID` | `private_key_jwt` if the middleware supports it, otherwise a client secret as a transition (OP-22) |

The IdentityHub and the dataspace wallet of the CSP do not call this module back. C4-M1 describes the service account of the dataspace wallet of the CSP for status reports about credentials (C4-M1 6.5).

### 6.9 Idempotency and concurrency

- Business key: `companyId` for the wallet, `technicalUserId` for a wallet account. `processId` and `stepRunId` are only used for the log.
- A long-running step responds `RUNNING` if its object was not yet complete at the time of the call, even if it completes in the same call. The end is then reported by the event that was written to the outbox in the same transaction.
- If the object was already complete at the time of the call (repetition), the step responds `DONE` with the result and writes no event.
- An order goes out to a provider at most once per attempt: the module sets `provider_ordered_at` only after an accepted response. If the order fails technically, the step responds 502 `upstream-error`; a repetition sends it again.
- Callbacks may arrive several times. Same content: 204 without change.
- Steps and callbacks for the same company or the same technical user never run at the same time (advisory lock on `company_id` or `technical_user_id`).

### 6.10 Scheduled tasks

Every `TIMER_INTERVAL_SECONDS`:

| Task | Selection | Effect |
|---|---|---|
| Callback deadline wallet | `REQUESTED`, `provider_ordered_at` older than `PROVIDER_CALLBACK_TIMEOUT_MINUTES` | `FAILED`, `PROVIDER_TIMEOUT`, `DataspaceWalletFailed` |
| DID check | `VALIDATING`, without `did_validated_at`, `next_check_at` reached | 6.4 |
| Callback deadline wallet account | `REQUESTED` or `DELETING`, order older than the deadline | 6.3 |
| Clean up credentials | `expires_at` reached | delete the row, `EXPIRED` |
| Clean up challenges (later) | `OPEN` and `expires_at` reached | `EXPIRED` |

This is not polling by Process Orchestration: the tasks check deadlines and the DID document, which can only be checked by resolving it. No provider is asked about the state of an order.

### 6.11 Adapter interface

Every adapter fulfils the same Python interface. New providers are added as a new adapter, without changes to steps, queries and events.

```python
class WalletAdapter(Protocol):
    provider: Literal["SAP_DIV", "IDENTITY_HUB", "CSP", "BRING_YOUR_OWN"]
    supports_wallet_accounts: bool
    issuance_mode: Literal["ISSUER_WRITES", "HOLDER_REQUESTS", "CREDENTIAL_OFFER"]

    async def order_wallet(self, order: WalletOrder) -> WalletOrderResult: ...
        # ORDERED (callback follows) or CREATED with did, secrets
    async def order_account(self, order: AccountOrder) -> AccountOrderResult: ...
        # ORDERED, CREATED with credentials or NOT_SUPPORTED
    async def order_account_deletion(self, account: WalletAccount) -> DeletionResult: ...
        # ORDERED or DELETED
    def did_for(self, bpn: str) -> str | None: ...
        # expected DID (6.4 step 1), None for CSP and own wallet
    def callback_router(self) -> APIRouter | None: ...
        # paths under /callbacks/<provider>/..., None without callbacks
```

| Adapter | `supports_wallet_accounts` | `issuance_mode` | Callbacks |
|---|---|---|---|
| `SAP_DIV` | yes | `ISSUER_WRITES` | yes (7.6) |
| `IDENTITY_HUB` | yes (shared STS access, 6.3) | `HOLDER_REQUESTS` | none; creation responds immediately |
| `CSP` | yes (separate access per account, 6.3) | `HOLDER_REQUESTS` | none; creation responds immediately |
| `BRING_YOUR_OWN` (later) | no | `CREDENTIAL_OFFER` | none |

## 7. Operations

### 7.1 `GET /companies/{companyId}/wallet`

Dataspace wallet of the caller's own company.

1. Token and permission `view_dataspace_wallet` (401, 403). `company_id` in the token not equal to `companyId` or missing: 404 `not-found`.
2. No row: 404 `not-found`.
3. Response 200 `Wallet`: provider, status, DID, DID document location, credential service, `bdrsUrl` (`BDRS_PUBLIC_URL`), `stsTokenUrl` (IdentityHub only; for SAP DIV and CSP the token address is per wallet account), `supportsWalletAccounts`, `failureReason`, timestamps. No secrets.

### 7.2 `GET /technical-users/{technicalUserId}/wallet-account`

1. Token and permission `view_dataspace_wallet` (401, 403).
2. Current wallet account of the technical user: the row with a status other than `DELETED`, otherwise the latest one. No row, or `company_id` of the account not equal to `company_id` in the token: 404.
3. Response 200 `WalletAccount` with `credentialsState` and `credentialsExpireAt`. No secrets.

### 7.3 `POST /technical-users/{technicalUserId}/wallet-account/credentials`

Hands out the credentials exactly once. Typically the IT Admin who created the technical user retrieves them (FRAMEWORK 5.7).

1. Token and permission `retrieve_wallet_credentials` (401, 403). Find the account as in 7.2, otherwise 404.
2. Status not `ACTIVE`: 409 `invalid-state`.
3. `credentials_state`: `RETRIEVED` → 410 `already-retrieved`; `EXPIRED` → 410 `credentials-expired`; `NONE` → 409 `no-credentials`.
4. In one transaction: read and decrypt the row, delete the row, `credentials_state` `RETRIEVED`, audit log entry with the `sub` of the person.
5. Response 200 `WalletAccountCredentials`: `tokenUrl`, `clientId`, `clientSecret`, plus `did` and `credentialServiceUrl`. Header `Cache-Control: no-store`.

Concurrent calls: only one gets the data (row lock with `FOR UPDATE`), the other gets 410.

### 7.4 `POST /wallet-control-challenges` (later)

Input: `applicationId`, `did` (`did:web:`), `credentialServiceUrl` (`https://`).

1. Token and permission `prove_wallet_control`. `BRING_YOUR_OWN` not in `ENABLED_PROVIDERS`: 400 `provider-not-enabled`.
2. Read the application at C1-M2 (11.6). Unknown or belonging to another person (`sub`): 404 `not-found`. `companyId` from the application.
3. There is a wallet with a status other than `FAILED` for `companyId`: 409 `wallet-exists`.
4. A wallet of another company (status other than `FAILED`) uses the DID: 409 `did-in-use`.
5. Resolve the DID document: not resolvable 422 `did-not-resolvable`; content violates 6.4 step 3: 422 `invalid-did-document`.
6. Create the challenge (`OPEN`, `created_by_sub`, nonce, `expires_at` = now + `CONTROL_CHALLENGE_TTL_MINUTES`), audit log entry.
7. Response 201 `ControlChallenge`: `challengeId`, `companyId`, `nonce`, `audience`, `expiresAt`, the expected claims.

### 7.5 `POST /wallet-control-challenges/{challengeId}/proof` (later)

Input: `proof` (compact JWT).

1. Token and permission. Challenge missing or created by another person: 404. Status `EXPIRED` or expired: 410 `challenge-expired`. Status `VERIFIED`: 200 with the stored result, without a new check (repetition). Status `USED`: 409 `invalid-state`.
2. Check per 6.6 step 3. Errors: 400 `invalid-proof` with the reason in `detail` (e.g. `nonce`, `signature`, `kid`, `credential-service`); the challenge stays `OPEN`.
3. Success: `VERIFIED`, `verified_at`, audit log entry. Response 200 `ControlProof`.

### 7.6 Callbacks from SAP DIV

The middleware calls back with a token of its service account in the Portal IAM of the OSP. The module checks the permission `submit_wallet_callback` and the client (`azp` equals `SAP_DIV_CALLBACK_CLIENT_ID`), otherwise 403. The contents match today's callbacks to Portal 2.6.0 (`POST /api/administration/registration/dim/{bpn}` and `POST /api/administration/serviceaccount/callback/{externalId}`). Every callback writes an audit log entry with `azp`.

**`POST /callbacks/sap-div/wallets/{bpn}`**: body `{ "did", "didDocument", "authenticationDetails": { "authenticationServiceUrl", "clientID", "clientSecret" } }`

1. Find the wallet with this BPN and provider `SAP_DIV`, otherwise 404.
2. Status `VALIDATING` or `ACTIVE` with the same DID: 204 without change. With another DID: 409 `conflict`. Status `FAILED`: 409 `invalid-state` (late callback after the deadline; a new attempt is needed).
3. DID form per 6.4 step 1. Wrong: `FAILED`, `DID_MISMATCH`, event; response 204.
4. Check `didDocument` against the schema of Portal 2.6.0 (`@context` list, `id`, `service` list with `id` and `type`, `verificationMethod` list with `id`, `type`, `publicKeyJwk` with `kty`, `crv`, `x`, `y`). Wrong: `FAILED`, `INVALID_DID_DOCUMENT`, event; response 204.
5. Store `authenticationDetails` encrypted as `HOLDER_ACCESS`; set `did`, `did_document_location`, `credential_service_url`; status `VALIDATING`, `validating_since`, `next_check_at` = now. Response 204.
6. The DID check (6.4) starts in the background immediately after the commit.

**`POST /callbacks/sap-div/technical-users/{externalId}`**: body `{ "authenticationServiceUrl", "clientID", "clientSecret" }`

1. `externalId` is the `walletAccountRef`. Account missing: 404.
2. Status `ACTIVE`: 204 without change. Status `FAILED`, `DELETING`, `DELETED`: 409 `invalid-state`.
3. Status `REQUESTED`: encrypt the credentials and store them in `wallet_account_credentials` (`token_url` = `authenticationServiceUrl` + `/oauth/token`, like today's display of the DIM addresses), `credentials_state` `AVAILABLE`, `expires_at`; status `ACTIVE`; event `WalletAccountCreated`. Response 204.

**`POST /callbacks/sap-div/technical-users/{externalId}/delete`**: no body

1. Account missing: 404. Status `DELETED`: 204. Status other than `DELETING`: 409 `invalid-state`.
2. Status `DELETED`, delete the credentials row, event `WalletAccountDeleted`. Response 204.

## 8. Steps for Process Orchestration

Common rules:

- Input always with `processId` and `stepRunId` (log only, `step_log`).
- Result `{ "stepRunId", "state": "DONE" | "RUNNING", "result": {…} }`, never with secrets.
- `RUNNING` and `DONE` per 6.9. Errors as problem details with `retryable` (section 12); whatever is already created stays, and a repetition continues.

### 8.1 `create-wallet` (long-running)

**Input:** `companyId`, optional `provider` (`SAP_DIV`, `IDENTITY_HUB` or `CSP`). The step reads the BPN and the name (only for the order to SAP DIV, not stored) at C3-M1 (11.5).

**Preconditions:** The company has a BPN (`CompanyIdentifierAssigned`); this means it is registered at the CSP with this OSP as home OSP (C3-M2, FRAMEWORK 5.2). For an own wallet (later): a verified control proof for `companyId`, at most `CONTROL_PROOF_MAX_AGE_DAYS` old.

**Behaviour:**

1. Check the input, otherwise 400 `validation`. Read the company at C3-M1; unknown: 404 `not-found`, without BPN: 409 `company-not-ready`. Wallet for `companyId` with another BPN, or wallet of another company with this BPN: 409 `conflict`.
2. Wallet exists:
   - `ACTIVE`: `DONE` with result, no event.
   - `REQUESTED` with `provider_ordered_at`, or `VALIDATING`: `RUNNING`, nothing else.
   - `REQUESTED` without `provider_ordered_at`: continue with step 4 (send the order again).
   - `FAILED`: new attempt (6.2), continue with step 3.
3. Choose the provider (6.1). For an own wallet: control proof too old 409 `control-proof-expired`; new attempt without a new proof 409 `control-proof-required`. Create or reset the row, status `REQUESTED`. Commit.
4. Order to the adapter:
   - **SAP DIV:** `POST {SAP_DIV_URL}/setup-dim?companyName=…&bpn=…&didDocumentLocation={SAP_DIV_DID_DOCUMENT_BASE_URL}/{bpn}/did.json` (as today). Accepted: `provider_ordered_at`, `provider_ref` = BPN. The callback (7.6) continues.
   - **IdentityHub:** create the participant context (11.2). Response with API key, STS client ID and STS secret: store them encrypted as `HOLDER_ACCESS` and `STS_ACCESS`; `did`, `did_document_location`, `provider_ref`, `provider_ordered_at`; status `VALIDATING`.
   - **CSP:** create the wallet through the wallet API of the CSP (11.3). Response with DID, credential service and holder access (Identity API, participant context, API key): store the holder access encrypted as `HOLDER_ACCESS`; `did`, `did_document_location`, `credential_service_url`, `provider_ref` = participant context, `provider_ordered_at`; status `VALIDATING`. If the CSP rejects on business grounds (other home OSP, BPN unknown): 409 `provider-rejected`, the row stays `REQUESTED` without `provider_ordered_at`.
   - **Own wallet (later):** control proof `USED`; take over or set `did`, `credential_service_url`, `did_document_location`, `control_challenge_id`, `provider_ordered_at`; status `VALIDATING`.
   - Technical error of the provider or no response: 502 `upstream-error`, the row stays `REQUESTED` without `provider_ordered_at`.
5. With `VALIDATING`: try 6.4 immediately. If the check succeeds, the wallet becomes `ACTIVE` within this call and `DataspaceWalletCreated` is in the outbox. Otherwise the scheduled tasks take over.
6. Response `RUNNING`.

**Result:** `walletId`, `companyId`, `provider`, `status`, `did` (if known), `didDocumentLocation` (if known), `failureReason` (with `FAILED`).

**Event:** `DataspaceWalletCreated` or `DataspaceWalletFailed`. After `DataspaceWalletCreated`, Process Orchestration calls C3-M2 `register-did` with the DID from the event (FRAMEWORK 5.2).

### 8.2 `create-wallet-account` (long-running)

**Input:** `technicalUserId`, `companyId` (from `WalletAccessRequested`).

**Preconditions:** The wallet of the company is `ACTIVE`, otherwise 409 `wallet-not-active`. The company is `ACTIVE` at C3-M1, otherwise 409 `company-not-active` (no new wallet account for a deactivated company, FRAMEWORK 5.4).

**Behaviour:**

1. An account for `technicalUserId` with a status other than `DELETED` exists:
   - different `companyId`: 409 `conflict`;
   - `ACTIVE` or `FAILED`: `DONE` with result, no event;
   - `REQUESTED` with `provider_ordered_at`: `RUNNING`;
   - `REQUESTED` without `provider_ordered_at`: continue with step 2, then step 4;
   - `DELETING`: 409 `invalid-state`.
2. Check the preconditions: status of the company's wallet in the module's own table, status of the company at C3-M1 (11.5).
3. Create the account: status `REQUESTED`, `provider_name` = `tu-` + `technicalUserId`. Commit.
4. Order to the adapter:
   - **SAP DIV:** `POST {SAP_DIV_URL}/technical-user/{bpn}` with `{ "externalId": "<walletAccountRef>", "name": "<provider_name>" }`. Accepted: `provider_ordered_at`. The callback continues.
   - **IdentityHub:** copy of the STS access into `wallet_account_credentials` (`token_url` = `IDENTITYHUB_STS_TOKEN_URL`), `credentials_state` `AVAILABLE`, status `ACTIVE`, event `WalletAccountCreated`.
   - **CSP:** `POST {CSP_WALLET_URL}/wallets/{bpn}/accounts` with `{ "externalId": "<walletAccountRef>", "name": "<provider_name>" }` (11.3). Response with `tokenUrl`, `clientId`, `clientSecret` into `wallet_account_credentials`, `credentials_state` `AVAILABLE`, `provider_ordered_at`, status `ACTIVE`, event `WalletAccountCreated`. Response 409 (account from an aborted attempt, secret missing): delete the account at the CSP and create it again.
   - **Own wallet (later):** status `FAILED`, `NOT_SUPPORTED_BY_PROVIDER`, event `WalletAccountFailed`. The company manages the accesses to its wallet itself.
   - Technical error of the provider: 502 `upstream-error`, the account stays `REQUESTED` without `provider_ordered_at`.
5. Response `RUNNING`.

**Result:** `walletAccountRef`, `technicalUserId`, `companyId`, `provider`, `status`, `failureReason`.

**Event:** `WalletAccountCreated` or `WalletAccountFailed`.

### 8.3 `delete-wallet-account` (long-running)

**Input:** `technicalUserId`, `companyId`, optional `walletAccountRef` (all from `WalletAccessRemovalRequested`). If `walletAccountRef` is given and does not belong to the current account of the technical user: 409 `conflict`.

**Preconditions:** No wallet account of the technical user in status `REQUESTED`, otherwise 409 `invalid-state`. The status of the company does not matter; an account can also be deleted for a deactivated company.

**Behaviour:**

1. No account, or only accounts with `DELETED`: `DONE` with status `DELETED` (or without `walletAccountRef` if there never was one), no event.
2. Different `companyId`: 409 `conflict`.
3. `FAILED`: status `DELETED`, event `WalletAccountDeleted`.
4. `ACTIVE`:
   - **SAP DIV:** status `DELETING`, commit; `POST {SAP_DIV_URL}/technical-user/{bpn}/delete` with `{ "externalId", "name" }`; accepted: `deletion_ordered_at`. The callback continues. Error: 502, the account stays `DELETING` without `deletion_ordered_at`.
   - **IdentityHub:** status `DELETED`, event `WalletAccountDeleted` (6.3).
   - **CSP:** `DELETE {CSP_WALLET_URL}/wallets/{bpn}/accounts/{walletAccountRef}`; 204 or 404: status `DELETED`, event `WalletAccountDeleted`. Error: 502, the account stays `ACTIVE`.
5. `DELETING` with `deletion_ordered_at`: `RUNNING`. Without: send the order again.
6. Delete the credentials row as soon as the account is `DELETED`. Response `RUNNING`.

**Result:** `walletAccountRef`, `technicalUserId`, `companyId`, `status`.

**Event:** `WalletAccountDeleted`; when the deadline passes, `WalletAccountDeletionFailed`.

## 9. Queries for other modules

In the same process without a token; over HTTP with the respective permission (6.8).

### 9.1 `GET /internal/wallets/{companyId}`

For C4-M1, C2-M3, connectors. Response `WalletInfo`: `walletId`, `companyId`, `bpn`, `provider`, `providerRef` (identifier at the provider, for IdentityHub and CSP the participant context), `status`, `did`, `didDocumentLocation`, `credentialServiceUrl`, `issuanceMode`, `supportsWalletAccounts`, `failureReason`, `activatedAt`. No wallet: 404. No secrets.

With `supportsWalletAccounts`, C2-M3 can tell in advance that a wallet role has no effect for an own wallet.

### 9.2 `GET /internal/technical-users/{technicalUserId}/wallet-account`

Current wallet account as in 7.2, without checking the relation to the company. Response `WalletAccount`, otherwise 404.

### 9.3 `GET /internal/wallets/{companyId}/holder-access`

Only for C4-M1. Returns how credentials get into the wallet, with the access needed for that. Header `Cache-Control: no-store`. Wallet not `ACTIVE`: 409 `wallet-not-active`.

| `issuanceMode` | Provider | Content | Use by C4-M1 |
|---|---|---|---|
| `ISSUER_WRITES` | SAP DIV | `holder` (= DID document location), `bpn`, `walletUrl`, `clientId`, `clientSecret` | as today, to the OSP's own credential issuer (`CreateBpnCredentialRequest`, `CreateMembershipCredentialRequest`) |
| `HOLDER_REQUESTS` | IdentityHub, dataspace wallet of the CSP | `did`, `bpn`, `identityApiUrl`, `participantContextId`, `apiKey`, `credentialServiceUrl` | request for the credentials through the Identity API of the participant context; the OSP's own IssuerService or the issuer of the CSP issues them (C4-M1) |
| `CREDENTIAL_OFFER` | own wallet (later) | `did`, `bpn`, `credentialServiceUrl`, no secrets | The issuer sends a DCP offer (credential offer) to the credential service; the wallet requests the credentials itself |

Every call writes an audit log entry (without a secret). The response is not logged.

## 10. Events

Written to the outbox in the same transaction as the change. Every event additionally carries `eventId`, `eventType`, `occurredAt`. None contains a secret.

| Event | When | Content |
|---|---|---|
| `DataspaceWalletCreated` | wallet `ACTIVE` (6.4) | `companyId`, `did`, `walletId`, `provider`, `didDocumentLocation`, `bpn` |
| `DataspaceWalletFailed` | wallet `FAILED` | `companyId`, `walletId`, `provider`, `reason` |
| `WalletAccountCreated` | account `ACTIVE` | `technicalUserId`, `companyId`, `walletAccountRef`, `provider` |
| `WalletAccountFailed` | account `FAILED` | `technicalUserId`, `companyId`, `walletAccountRef`, `reason` |
| `WalletAccountDeleted` | account `DELETED` | `technicalUserId`, `companyId`, `walletAccountRef` |
| `WalletAccountDeletionFailed` | deadline for the deletion callback passed | `technicalUserId`, `companyId`, `walletAccountRef`, `reason` |

Reasons (`reason`) for the wallet: `PROVIDER_TIMEOUT`, `DID_MISMATCH`, `INVALID_DID_DOCUMENT`, `DID_NOT_RESOLVABLE`. For wallet accounts: `PROVIDER_TIMEOUT`, `NOT_SUPPORTED_BY_PROVIDER`.

Use in the flows (Process Orchestration): `DataspaceWalletCreated` → C3-M2 `register-did` → `CompanyDidRegistered` → C4-M1 `issue-credentials` (FRAMEWORK 5.2). `WalletAccountCreated` or `WalletAccountFailed` → C2-M3 `apply-wallet-access` with `walletAccountRef` and result (FRAMEWORK 5.7). `WalletAccountDeleted` → C2-M3 `delete`.

## 11. Calls to other modules and external systems

If the address is empty, the module uses a built-in stand-in (FRAMEWORK 4.7). This way it can be run and tested on its own. The stand-in providers run through the same code as the real adapters, from the point at which the response of the provider arrives.

### 11.1 SAP DIV (`SAP_DIV_URL`)

Token by client credentials (`SAP_DIV_TOKEN_URL`, `SAP_DIV_CLIENT_ID`, `SAP_DIV_CLIENT_SECRET`). Paths as in Portal 2.6.0 (`Dim.Library/DimService.cs`):

| Purpose | Call | Response |
|---|---|---|
| Order a wallet | `POST /setup-dim?companyName=&bpn=&didDocumentLocation=` | 2xx; result by callback 7.6 |
| Order a wallet account | `POST /technical-user/{bpn}` with `{ "externalId", "name" }` | 2xx; result by callback |
| Delete a wallet account | `POST /technical-user/{bpn}/delete` with `{ "externalId", "name" }` | 2xx; result by callback |

The middleware knows the callback addresses of the installation from its own configuration; the order does not name them.

**Built-in stand-in** (`SAP_DIV_URL` empty): accepts orders and calls the callback code in the same process after `STUB_CALLBACK_DELAY_SECONDS`. Wallet: DID per 6.4 step 1, DID document with a new P-256 key and service `CredentialService` (`https://div-stub.invalid/api/credentials`), made-up `authenticationDetails`. Wallet account: made-up credentials. Deletion: immediate callback. The stand-in supplies the DID documents to the built-in resolution. `STUB_FAILURES` simulates failures (14.3).

For tests over HTTP, the Tractus-X wallet stub (`ssi-dim-wallet-stub`) serves this purpose; it mimics the DIM interface for the portal. It replaces the built-in stand-in when `SAP_DIV_URL` points to it.

### 11.2 IdentityHub (`IDENTITYHUB_URL`)

Identity API of the OSP's own IdentityHub, header `x-api-key: {IDENTITYHUB_API_KEY}`. Version v1alpha, as in the existing connection of the IdentityHub to the portal.

| Purpose | Call |
|---|---|
| Create participant context | `POST /v1alpha/participants` with `participantContextId` = BPN, `did`, `active: true`, `key` (`keyId` = `<did>#key-1`, `privateKeyAlias` = `<bpn>-key-1`, `keyGeneratorParams` `{ "algorithm": "EC", "curve": "secp256r1" }`), `serviceEndpoints` with one service `{ "id": "<did>#credential-service", "type": "CredentialService", "serviceEndpoint": "{IDENTITYHUB_CREDENTIAL_SERVICE_URL}/v1/participants/<context>" }`, `roles: []`. Response: `apiKey`, `clientId`, `clientSecret`. |
| Delete participant context | `DELETE /v1alpha/participants/<context>` |

`<context>` is the BPN, unchanged or base64url depending on `IDENTITYHUB_CONTEXT_ID_ENCODING`. If creation responds with 409 (the context exists from an aborted attempt, the secrets are missing), the adapter deletes the context and creates it again. This is safe because the wallet was not yet `ACTIVE`.

**Built-in stand-in** (`IDENTITYHUB_URL` empty): keeps participant contexts in memory, generates keys, API keys and the STS secret, supplies the DID document to the built-in resolution.

### 11.3 Dataspace wallet of the CSP (`CSP_WALLET_URL`)

The wallet API of the CSP is a proposal, like the core services API in C3-M2. If the CSP defines it differently, only this adapter changes. All calls with the access token of the CSP (sign-in per section 4); responses in JSON, errors as problem details. On every call, the CSP checks that the calling OSP is the home OSP of the BPN.

| Call | Purpose | Responses |
|---|---|---|
| `POST /wallets` with `bpn` | create the dataspace wallet | 201 `{ participantContextId, did, credentialServiceUrl, identityApiUrl, apiKey }`; 403 other home OSP or BPN unknown at the CSP; 409 wallet exists |
| `DELETE /wallets/{bpn}` | delete the wallet; only for the retry after 409 | 204 |
| `POST /wallets/{bpn}/accounts` with `externalId`, `name` | create a wallet account with its own STS access | 201 `{ tokenUrl, clientId, clientSecret }`; 409 account exists |
| `DELETE /wallets/{bpn}/accounts/{externalId}` | delete the wallet account, revoke the access | 204; 404 counts as deleted |

- `apiKey` is valid only for the participant context of the company. With it, C4-M1 addresses the Identity API of the dataspace wallet of the CSP (`identityApiUrl`, the same interface as with the IdentityHub) to request credentials (9.3).
- If `POST /wallets` responds with 409 (wallet from an aborted attempt, the secrets are missing), the adapter deletes the wallet at the CSP and creates it again. This is safe because the wallet was not yet `ACTIVE` in this installation.
- Response 403: 409 `provider-rejected` (`retryable: false`). Response 401 at the token endpoint or at the API, responses 5xx and timeout: 502 `upstream-error` (`retryable: true`).
- The dataspace wallet of the CSP does not call this module back. It reports arrived credentials to C4-M1 at `{address of the installation}/api/c4-m1/issuer-callbacks/{credentialKind}`, with its own service account (C4-M1 6.5). The address of the installation is in the trust list entry of the OSP.

**Built-in stand-in** (`CSP_WALLET_URL` empty): keeps wallets and wallet accounts in memory. DID `did:web:csp-wallet.example.org:{BPN}`, DID document with a new P-256 key and service `CredentialService` (`https://csp-wallet.example.org/api/credentials/v1/participants/{BPN}`), a made-up API key, made-up separate credentials per wallet account with `tokenUrl` `https://csp-wallet.example.org/api/sts/token`. The stand-in supplies the DID documents to the built-in resolution. `STUB_FAILURES` with `CSP_REJECTED` makes it respond with 403.

For the hackathon, the test tools provide a shared CSP stand-in for several installations (FRAMEWORK 4.7). If it offers the wallet API, `CSP_WALLET_URL` points to it.

### 11.4 DID resolution (`DID_RESOLVER_URL`)

- Set: `GET {DID_RESOLVER_URL}/1.0/identifiers/{did}` as today (`DimService.ValidateDid`). Success if HTTP 2xx and `didResolutionMetadata.error` is empty; the DID document is in `didDocument`.
- Empty: built-in resolution for `did:web` in this order: files from `TEST_DID_DOCUMENTS_DIR` (file name = host, e.g. `wallet.beispiel.example.json`), documents of the stand-in providers, otherwise `GET https://<host>/<path>/did.json` or `https://<host>/.well-known/did.json` per the `did:web` method, timeout 10 seconds, HTTPS only.

### 11.5 C3-M1 Company Profile (`COMPANY_URL`)

`GET /api/c3-m1/internal/companies/{companyId}` (or in the same process) returns `name`, `bpn`, `status`. Needed in `create-wallet` (BPN, name for SAP DIV) and `create-wallet-account` (status `ACTIVE`). **Stand-in:** reads `stub-companies.json` (`[{ "id", "name", "bpn", "status" }]`).

### 11.6 C1-M2 Registration (`APPLICATION_URL`, own wallet only, later)

`GET /api/c1-m2/internal/applications/{applicationId}` (or in the same process) returns `companyId` and `applicantSub` (`sub` of the applicant). **Stand-in:** reads `stub-applications.json` (`[{ "applicationId", "companyId", "applicantSub" }]`).

### 11.7 Portal IAM

Only through the shared token validation library. No writing to Keycloak.

## 12. Errors

Problem details (RFC 9457), `type` = `urn:c4-m2:problem:<name>`.

| `type` | HTTP | When | `retryable` for steps |
|---|---|---|---|
| `urn:c4-m2:problem:validation` | 400 | input invalid | `false` |
| `urn:c4-m2:problem:provider-not-enabled` | 400 | provider not in `ENABLED_PROVIDERS` | `false` |
| `urn:c4-m2:problem:invalid-proof` | 400 | control proof invalid; reason in `detail` | – |
| `urn:c4-m2:problem:unauthorized` | 401 | token missing or invalid | – |
| `urn:c4-m2:problem:forbidden` | 403 | permission missing, or wrong client on a callback | – |
| `urn:c4-m2:problem:not-found` | 404 | wallet, wallet account, application or challenge does not exist or is not visible (other company, other person) | `false` |
| `urn:c4-m2:problem:invalid-state` | 409 | the status does not allow the operation | `false` |
| `urn:c4-m2:problem:conflict` | 409 | contradictory data (other BPN, other DID, other company) | `false` |
| `urn:c4-m2:problem:company-not-ready` | 409 | company at C3-M1 without BPN | `false` |
| `urn:c4-m2:problem:company-not-active` | 409 | company at C3-M1 not `ACTIVE`; no new wallet account | `false` |
| `urn:c4-m2:problem:wallet-not-active` | 409 | wallet of the company not `ACTIVE` | `false` |
| `urn:c4-m2:problem:wallet-exists` | 409 | company already has a wallet | – |
| `urn:c4-m2:problem:did-in-use` | 409 | DID already belongs to another wallet | – |
| `urn:c4-m2:problem:control-proof-expired` | 409 | control proof older than allowed | `false` |
| `urn:c4-m2:problem:control-proof-required` | 409 | new attempt for an own wallet without a new control proof | `false` |
| `urn:c4-m2:problem:provider-rejected` | 409 | the CSP rejects on business grounds (other home OSP, BPN unknown at the CSP) | `false` |
| `urn:c4-m2:problem:no-credentials` | 409 | wallet account has no retrievable credentials | – |
| `urn:c4-m2:problem:already-retrieved` | 410 | credentials already retrieved | – |
| `urn:c4-m2:problem:credentials-expired` | 410 | credentials deleted after the deadline | – |
| `urn:c4-m2:problem:challenge-expired` | 410 | control proof challenge expired | – |
| `urn:c4-m2:problem:did-not-resolvable` | 422 | DID document not resolvable | – |
| `urn:c4-m2:problem:invalid-did-document` | 422 | DID document does not meet 6.4 step 3 | – |
| `urn:c4-m2:problem:upstream-error` | 502 | provider did not accept an order or is not reachable | `true` |

## 13. Acceptance

The module is done when these cases work with the built-in stand-ins for the providers, C3-M1 and C1-M2. Case 1 additionally against the Tractus-X wallet stub, case 9 against an IdentityHub of the test environment and case 8 against the shared CSP stand-in, if the hackathon environment provides them. Cases 11 to 13 and 18 apply only with the own wallet (OP-29) and are not part of the acceptance in version 0.2.

1. `create-wallet` for `BPNL0000000DIV01` with `SAP_DIV`: response `RUNNING`. After the callback of the stand-in, the wallet is `ACTIVE`, the DID is `did:web:div.example.org:did:BPNL0000000DIV01`, and exactly one `DataspaceWalletCreated` with this DID is in the outbox. The module does not call any directory BPN → DID.
2. Repetition of case 1 with a new `stepRunId`: `DONE`, same DID, no second order to the stand-in, no further event.
3. Repetition while the wallet is `REQUESTED`: `RUNNING`, no second order.
4. `BPNL000000FAIL01` (the stand-in returns a wrong DID): `FAILED` with `DID_MISMATCH`, event `DataspaceWalletFailed`. A new `create-wallet` (failure in the stand-in reset for the test) leads to `ACTIVE` with `attempt` = 2.
5. `BPNL000000FAIL02` (no callback), deadline in the test 1 minute: after expiry `FAILED` with `PROVIDER_TIMEOUT`. A late callback gives 409.
6. Callback without the permission `submit_wallet_callback` or from another service account: 403. Callback for an unknown BPN: 404. Duplicate callback with the same DID: 204, no change. Every accepted callback is in the `audit_log` with `azp`.
7. DID not resolvable (stand-in `DID_NOT_RESOLVABLE`), deadline in the test 2 minutes, interval 30 seconds: the wallet stays `VALIDATING`, then `FAILED` with `DID_NOT_RESOLVABLE`.
8. `create-wallet` for `BPNL00000000CSP1` with `CSP`: the module fetches a token from the CSP; the client assertion is signed with the key of the OSP, `iss` and `sub` are the `ospId` from `SETTINGS_FILE`. Wallet `ACTIVE` within the call, DID `did:web:csp-wallet.example.org:BPNL00000000CSP1`, service `CredentialService` in the DID document, response `RUNNING`, `DataspaceWalletCreated` in the outbox. `holder-access` returns `HOLDER_REQUESTS` with `identityApiUrl`, `participantContextId` and `apiKey`.
9. `create-wallet` for `BPNL00000000IH01` with `IDENTITY_HUB`: participant context with identifier = BPN, DID `did:web:ih.example.org:BPNL00000000IH01`, service `CredentialService` in the DID document; wallet `ACTIVE` within the call, response `RUNNING`, event in the outbox. `holder-access` returns `HOLDER_REQUESTS` with `apiKey`.
10. Parallel operation: cases 1, 8 and 9 in the same installation; `/internal/wallets/{companyId}` shows the right provider and issuance mode for each company.
11. (later) Own wallet: the applicant P3 creates a challenge for their application and `did:web:wallet.beispiel.example` (DID document from 14.2 in `TEST_DID_DOCUMENTS_DIR`); the response has the `companyId` of the application. Control proof signed with the test key: `VERIFIED`. `create-wallet`: provider `BRING_YOUR_OWN`, wallet `ACTIVE`; `wallet_secret` has no row; `holder-access` returns `CREDENTIAL_OFFER` without a secret.
12. (later) Control proof negative, each 400 `invalid-proof` and the challenge stays `OPEN`: wrong nonce, wrong `aud`, `kid` not in the DID document, signature with another key, `exp` passed, other credential service. Challenge after expiry: 410. DID of another active wallet: 409 `did-in-use`. Challenge for an application of another person: 404. Proof for a challenge of another person: 404.
13. (later) `create-wallet` with a control proof older than `CONTROL_PROOF_MAX_AGE_DAYS`: 409 `control-proof-expired`.
14. `create-wallet-account` at SAP DIV: `RUNNING`, after the callback `ACTIVE`, event `WalletAccountCreated` with `walletAccountRef`. Neither the event nor the step result nor `step_log` contains the client secret or the client ID of the credentials.
15. Person P1 (`company_id` of the company, `retrieve_wallet_credentials`) retrieves the credentials of a wallet account from case 20: 200 with `clientSecret`, audit log entry with the `sub` of P1 and without names; second call 410 `already-retrieved`. Person P2 of another company: 404. Person P4 without the permission: 403. Person P3 without a company and without the permission: 403 (the permission is checked before the company). Two concurrent calls: exactly one gets 200.
16. `create-wallet-account` before the wallet is `ACTIVE`: 409 `wallet-not-active`.
17. `create-wallet-account` for the deactivated company (`BPNL0000000INA01`, C3-M1 status `INACTIVE`, wallet `ACTIVE`): 409 `company-not-active`, `retryable` `false`, no account. `delete-wallet-account` for an existing account of this company: `DELETED` as usual.
18. (later) `create-wallet-account` with an own wallet: `RUNNING`, event `WalletAccountFailed` with `NOT_SUPPORTED_BY_PROVIDER`; repetition `DONE` with `FAILED`, no further event.
19. `create-wallet-account` at the IdentityHub for two technical users: both `ACTIVE`, both receive the same STS client ID.
20. `create-wallet-account` at the dataspace wallet of the CSP for two technical users: both `ACTIVE` within the call, two different client IDs. `delete-wallet-account` for one of them: `DELETED`, `WalletAccountDeleted`, the CSP stand-in no longer knows the account.
21. `delete-wallet-account` at SAP DIV: `RUNNING`, after the callback `DELETED`, event `WalletAccountDeleted`, credentials not retrieved are deleted. Repetition: `DONE`. Without a deletion callback (stand-in `NO_DELETE_CALLBACK`): after the deadline `WalletAccountDeletionFailed`, the status stays `DELETING`.
22. Credentials not retrieved after `ACCOUNT_CREDENTIALS_RETENTION_DAYS` (test: advance the clock): row deleted, `credentialsState` `EXPIRED`, retrieval 410 `credentials-expired`.
23. `holder-access` at SAP DIV returns `ISSUER_WRITES` with `walletUrl`, `clientId`, `clientSecret` from the wallet callback. Over HTTP without `read_wallet_holder_access`: 403.
24. The CSP rejects (`STUB_FAILURES` `BPNL00000000CSP2:CSP_REJECTED`): `create-wallet` 409 `provider-rejected`, `retryable` `false`; the wallet stays `REQUESTED` without `provider_ordered_at`, no event. CSP not reachable (`ORDER_ERROR`): 502 `upstream-error`, `retryable` `true`.
25. Secrets: a full-text search for the secrets of the stand-in providers (including API keys and account accesses of the dataspace wallet of the CSP) finds them neither in the log nor in `outbox`, `step_log`, `audit_log`, nor in plain text in any table.

## 14. Test data

Test IDs per FRAMEWORK 7a; further ones are defined here.

### 14.1 Companies, people, service accounts

| Purpose | `companyId` | BPN | Provider |
|---|---|---|---|
| SAP DIV | `1a2b3c4d-0001-4000-8000-000000000001` | `BPNL0000000DIV01` | `SAP_DIV` |
| IdentityHub | `1a2b3c4d-0002-4000-8000-000000000002` | `BPNL00000000IH01` | `IDENTITY_HUB` |
| Dataspace wallet of the CSP (Beispiel Maschinenbau GmbH) | `11111111-1111-4111-8111-111111111111` | `BPNL00000000CSP1` | `CSP` |
| CSP rejects | `1a2b3c4d-0007-4000-8000-000000000007` | `BPNL00000000CSP2` | `CSP` |
| own wallet (later) | `5b0c9d1e-2f3a-4b5c-8d6e-7f8091a2b3c4` | `BPNL0000000BYO01` | `BRING_YOUR_OWN` |
| wrong DID | `1a2b3c4d-0004-4000-8000-000000000004` | `BPNL000000FAIL01` | `SAP_DIV` |
| no callback | `1a2b3c4d-0005-4000-8000-000000000005` | `BPNL000000FAIL02` | `SAP_DIV` |
| deactivated | `1a2b3c4d-0006-4000-8000-000000000006` | `BPNL0000000INA01` | `SAP_DIV`, wallet `ACTIVE`, C3-M1 status `INACTIVE` |

`stub-companies.json` contains all companies of the table with name, BPN and status (`ACTIVE`, except "deactivated" with `INACTIVE`). `installation.json` (`SETTINGS_FILE`): `ospId` of OSP A `22222222-2222-4222-8222-222222222222`. Key of the OSP (`OSP_SIGNING_KEY_FILE`): a key generated in the test (ES256); the CSP stand-in or test server knows the public part for the `ospId`.

People and service accounts in the test Portal IAM:

| Who | Token | Purpose |
|---|---|---|
| P1 "Erika Mustermann" (test person per FRAMEWORK 7a) | `company_id` `11111111-1111-4111-8111-111111111111`, portal role `IT Admin` (`view_dataspace_wallet`, `retrieve_wallet_credentials`) | cases 15, 20 |
| P2 | `company_id` `1a2b3c4d-0002-4000-8000-000000000002`, the same permissions | other company (case 15) |
| P3 | without `company_id`, portal role `Applicant` (`prove_wallet_control`, later); application `a0000001-0000-4000-8000-000000000001` for `5b0c9d1e-2f3a-4b5c-8d6e-7f8091a2b3c4` in `stub-applications.json` | cases 11, 12, 15 |
| P4 | `company_id` `11111111-1111-4111-8111-111111111111`, portal role `Member` without permissions of this module | case 15 |
| `sa-div-callback` | service account with `submit_wallet_callback` (= `SAP_DIV_CALLBACK_CLIENT_ID`) | SAP DIV callbacks |
| `sa-other-callback` | service account with `submit_wallet_callback`, other client | case 6 |
| `sa-modules` | client with `query_c4_m2`, `execute_steps_c4_m2`, `read_wallet_holder_access` | calls over HTTP |

Configuration in the test: `ENABLED_PROVIDERS` = `SAP_DIV,IDENTITY_HUB,CSP` (for the later cases also `BRING_YOUR_OWN`), `SAP_DIV_DID_DOCUMENT_BASE_URL` = `https://div.example.org/did`, `IDENTITYHUB_DID_BASE` = `ih.example.org`, `CONTROL_PROOF_AUDIENCE` = `https://portal.example.org/api/c4-m2`, `SAP_DIV_URL`, `IDENTITYHUB_URL`, `CSP_WALLET_URL`, `DID_RESOLVER_URL` empty.

### 14.2 Own wallet (later)

DID document (file `wallet.beispiel.example.json` in `TEST_DID_DOCUMENTS_DIR`):

```json
{
  "@context": ["https://www.w3.org/ns/did/v1", "https://w3id.org/security/suites/jws-2020/v1"],
  "id": "did:web:wallet.beispiel.example",
  "verificationMethod": [{
    "id": "did:web:wallet.beispiel.example#key-1",
    "type": "JsonWebKey2020",
    "controller": "did:web:wallet.beispiel.example",
    "publicKeyJwk": {
      "kty": "EC", "crv": "P-256",
      "x": "zGFDpzmtLVxoJo5wJNLhZTpHq0GphHMxsBATEe-XLD8",
      "y": "WYc-FOevkwEhIjj8RmQ7CBgnwS4TpOhEi0RKu5-mXUg"
    }
  }],
  "authentication": ["did:web:wallet.beispiel.example#key-1"],
  "assertionMethod": ["did:web:wallet.beispiel.example#key-1"],
  "service": [{
    "id": "did:web:wallet.beispiel.example#credential-service",
    "type": "CredentialService",
    "serviceEndpoint": "https://wallet.beispiel.example/api/credentials/v1/participants/beispiel"
  }]
}
```

Private test key (for tests only, never in production):

```json
{ "kty": "EC", "crv": "P-256",
  "x": "zGFDpzmtLVxoJo5wJNLhZTpHq0GphHMxsBATEe-XLD8",
  "y": "WYc-FOevkwEhIjj8RmQ7CBgnwS4TpOhEi0RKu5-mXUg",
  "d": "8fcQyhGu3HkvPAbw-_gsxarCsX3LRw31A6SDYr7Agh4",
  "kid": "did:web:wallet.beispiel.example#key-1" }
```

Example of a control proof (nonce `q8Vb2xLr0YkE5mTt3WcZ1g`, `iat` 26 Oct 2026 10:05 UTC, `exp` 10:10 UTC; validly signed with the test key). Tests generate the proof again with the nonce of their challenge; the example shows the form.

```text
eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImRpZDp3ZWI6d2FsbGV0LmJlaXNwaWVsLmV4YW1wbGUja2V5LTEifQ.eyJpc3MiOiJkaWQ6d2ViOndhbGxldC5iZWlzcGllbC5leGFtcGxlIiwic3ViIjoiZGlkOndlYjp3YWxsZXQuYmVpc3BpZWwuZXhhbXBsZSIsImF1ZCI6Imh0dHBzOi8vcG9ydGFsLmV4YW1wbGUub3JnL2FwaS9jNC1tMiIsIm5vbmNlIjoicThWYjJ4THIwWWtFNW1UdDNXY1oxZyIsImNvbXBhbnlfaWQiOiI1YjBjOWQxZS0yZjNhLTRiNWMtOGQ2ZS03ZjgwOTFhMmIzYzQiLCJpYXQiOjE3OTMwMDkxMDAsImV4cCI6MTc5MzAwOTQwMH0.I-AorSbS_6ead58MrpniDIgYHW2wS-8L8E5QqvsrgHaq0kfomve3-bLIj6q9bWLkEUNxKTnk5QQzBY7dVwLR1w
```

Content: `{ "iss": "did:web:wallet.beispiel.example", "sub": "did:web:wallet.beispiel.example", "aud": "https://portal.example.org/api/c4-m2", "nonce": "q8Vb2xLr0YkE5mTt3WcZ1g", "company_id": "5b0c9d1e-2f3a-4b5c-8d6e-7f8091a2b3c4", "iat": 1793009100, "exp": 1793009400 }`

### 14.3 Failures of the stand-in providers (`STUB_FAILURES`)

| Value | Effect |
|---|---|
| `<BPN>:DID_MISMATCH` | the SAP DIV stand-in returns the DID `did:web:other.example:<BPN>` |
| `<BPN>:INVALID_DID_DOCUMENT` | DID document without `verificationMethod` |
| `<BPN>:NO_CALLBACK` | the SAP DIV stand-in does not call back |
| `<BPN>:ORDER_ERROR` | an order to the stand-in (SAP DIV, IdentityHub or CSP) ends with 503 |
| `<BPN>:CSP_REJECTED` | the stand-in of the wallet API of the CSP responds with 403 (other home OSP) |
| `<BPN>:DID_NOT_RESOLVABLE` | the built-in resolution does not find the DID document |
| `<technicalUserId>:NO_CALLBACK` | no callback for the wallet account |
| `<technicalUserId>:NO_DELETE_CALLBACK` | no deletion callback |

## 15. Changes from version 0.1

- **OSP 2.0:** The module runs in the installation of every OSP. Which providers an installation offers is configuration (`ENABLED_PROVIDERS`).
- **New adapter `CSP`** for the dataspace wallet of the CSP (11.3): wallet API as a proposal, sign-in by `private_key_jwt` with the key of the OSP (4), immediate response, separate access per wallet account, holder access in the same form as with the IdentityHub. New error `provider-rejected`.
- **Directory BPN → DID no longer here:** C3-M2 `register-did` makes the entry after `DataspaceWalletCreated` (6.5). The wallet is `ACTIVE` as soon as the DID document is checked. Dropped: `BDRS_URL`, `BDRS_API_KEY`, `BDRS_RETRY_INTERVAL_MINUTES`, `BDRS_MAX_ATTEMPTS`, the fields `bdrs_registered_at` and `bdrs_attempts`, the reason `BDRS_ERROR`, the scheduled task and the BDRS stand-in. `BDRS_PUBLIC_URL` stays for display.
- **Callers:** People of the company (portal roles `IT Admin`, `Company Admin`) read the wallet and wallet accounts and retrieve credentials with their own token (scope `company`). Path `GET /companies/{companyId}/wallet` instead of `GET /wallet` with `X-On-Behalf-Of`. Tags `Company`, `Provider`, `Queries`, `Steps`; `x-permission-scope` per operation.
- **Deactivated companies:** no new wallet accounts (FRAMEWORK 5.4, OP-07); `create-wallet-account` checks the status at C3-M1, new error `company-not-active`.
- **Own wallet** marked as "later" (OP-29). The control proof is bound to the applicant and their application (scope `own`, query at C1-M2) instead of the PSP; the error `control-proof-mismatch` is dropped.
- **Holder access:** C4-M1 uses it for all issuance modes, including the request at the IdentityHub and at the dataspace wallet of the CSP.
- **Dropped:** delegation (scope `technical-users`, `X-On-Behalf-Of`, `X-Acting-Person`, `check_delegation`, `DELEGATION_URL`, error `not-delegated`), `registeringPspId` and `pspId`, status report to PSPs (`notify-psp`), all references to PSPs and operators.
- **Audit log** per FRAMEWORK 4.3 with `sub` or `azp` instead of `psp_id` and the acting person.
- **Errors** with `retryable` (OP-18).
- **Terms** per FRAMEWORK 8: EUDI Wallet, Business Wallet and dataspace wallet are separate; "directory BPN → DID" instead of only BDRS.
- **Test data** with the shared test IDs (FRAMEWORK 7a); new cases 8, 17, 20 and 24, the BDRS case is dropped.
