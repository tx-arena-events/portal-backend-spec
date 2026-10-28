# C4-M2 Dataspace Wallet Interface: open points

**Version:** 0.2, as of 2026-10-28

The [README](README.md) fixes each of these points for version 0.2, so that the module can be built. The following decisions are provisional and still to be agreed. Decided on 28 Oct 2026: OSP 2.0 (every OSP runs its own installation, the CSP provides the core services and, optionally, a dataspace wallet), and C3-M2 makes the entry BPN → DID.

## Provisional decisions

| No. | Topic | Decision in the README | To be agreed with |
|---|---|---|---|
| 1 | Choice of the provider | Verified control proof (later) before `provider` from the input before `DEFAULT_PROVIDER` (README 6.1). The OSP decides through `ENABLED_PROVIDERS` which providers exist, not the company. | Project owner, OSPs, C1-M2 |
| 2 | Dataspace wallet of the CSP | own adapter `CSP`; wallet API of the CSP as a proposal of this spec: create a wallet, wallet accounts with their own access, Identity API per participant context in the same form as with the IdentityHub; immediate response, no callbacks to C4-M2 (README 11.3) | CSP, association |
| 3 | Sign-in at the wallet API of the CSP | client credentials with `private_key_jwt`, `client_id` = `ospId`, the same key and token endpoint as in C3-M2 (README 4) | CSP, C3-M2 |
| 4 | Rejection by the CSP on business grounds | 409 `provider-rejected`, `retryable: false`; the row stays `REQUESTED` without an order; technical errors 502 `upstream-error`, `retryable: true` (README 8.1, 11.3) | CSP, Process Orchestration (OP-18) |
| 5 | Retry at the CSP | If the CSP responds with 409 when creating a wallet or wallet account (aborted attempt, secrets missing), the adapter deletes and creates again, as with the IdentityHub (README 8.2, 11.3) | CSP |
| 6 | Control proof of an own wallet (later) | Challenge with nonce, JWT signed with a key from the DID document, `did:web` only, valid for 30 days; bound to the applicant and their application (scope `own`, query at C1-M2) instead of the PSP (README 6.6). Pattern: self-issued ID token in DCP. Proves only control over the key. | Project owner, Tractus-X community, C1-M2 |
| 7 | Own wallet and wallet accounts (later) | not supported; `create-wallet-account` ends with `WalletAccountFailed` (`NOT_SUPPORTED_BY_PROVIDER`). The company manages the accesses to its wallet itself. | C2-M3 |
| 8 | Wallet accounts at the IdentityHub | Every wallet account gets a copy of the STS access of the participant context. Deleting an account revokes nothing at the IdentityHub (README 6.3). The dataspace wallet of the CSP, in contrast, has a separate access per account. | OSPs, IdentityHub community, CSP |
| 9 | Creation in the IdentityHub | identifier of the participant context = BPN, DID `did:web:{IDENTITYHUB_DID_BASE}:{BPN}`, key EC P-256, Identity API v1alpha; on 409 delete and create again (README 11.2). The template is the existing connection of the IdentityHub to the portal. | OSPs, IdentityHub version |
| 10 | DID with the dataspace wallet of the CSP | the CSP decides host and path; only the form `did:web:`, resolvability, the content of the DID document and uniqueness in the installation are checked (README 6.4) | CSP |
| 11 | Access for issuance | query `holder-access` only for C4-M1, with issuance mode `ISSUER_WRITES`, `HOLDER_REQUESTS` or `CREDENTIAL_OFFER` (README 9.3). C4-M1 itself triggers the request in the IdentityHub and the dataspace wallet of the CSP with the API key of the participant context. Alternative: C4-M2 triggers it, then as a separate step `request-credentials`. | C4-M1 |
| 12 | Callbacks from SAP DIV | paths under `/callbacks/sap-div/...`, contents as today. If the middleware only allows setting the base address, the ingress of the installation rewrites the old paths (`/api/administration/registration/dim/{bpn}`, `/api/administration/serviceaccount/callback/{externalId}[/delete]`) to the new ones. Check through the permission `submit_wallet_callback` and a fixed service account. | OSPs, SAP, C2-M5 |
| 13 | Name of the wallet account | `tu-{technicalUserId}` at SAP DIV and CSP instead of the name of the technical user (today: name without spaces). Saves a query to C2-M3 and is unique. | C2-M3, OSPs |
| 14 | Token address at SAP DIV | `authenticationServiceUrl` + `/oauth/token`, like today's display of the DIM addresses (`CompanyDataBusinessLogic.GetDimServiceUrls`) | OSPs, SAP |
| 15 | Deadlines | callback from SAP DIV 24 hours; DID check every 10 minutes for up to 7 days (today `MaxValidationTimeInDays` = 7); credentials can be retrieved for 30 days | OSPs |
| 16 | Directory BPN → DID | no longer in C4-M2; C3-M2 `register-did` after `DataspaceWalletCreated`. The wallet is `ACTIVE` before the entry is in the directory (README 6.5). | C3-M2, Process Orchestration |
| 17 | Wallet account before the wallet is active and for companies that are not active | 409 `wallet-not-active` or `company-not-active`; for this, `create-wallet-account` reads the status at C3-M1 (README 8.2) | Process Orchestration, C3-M1 |
| 18 | Permissions and scopes | `view_dataspace_wallet`, `retrieve_wallet_credentials` (`company`; portal roles `IT Admin`, `Company Admin`), `prove_wallet_control` (`own`; later `Applicant`), `submit_wallet_callback`, `read_wallet_holder_access`, `execute_steps_c4_m2`, `query_c4_m2` (`osp`) (README 6.8) | C2-M5 Portal IAM |
| 19 | Scheduled tasks | deadlines and DID check run in the module (README 6.10); no provider is asked about the state | Process Orchestration, architecture |
| 20 | DID resolution | without `DID_RESOLVER_URL`, built-in `did:web` resolution, also in production | OSPs |
| 21 | Encryption | AES-256-GCM with key version per row (today AES with mode and IV from `EncryptionConfigs`) | Base skeleton |
| 22 | Wallet `ACTIVE` | final in 0.2; no deletion, no change of provider, not even when the company is deactivated | Project owner |

## Deviations from the framework

| No. | Deviation | Reason |
|---|---|---|
| A1 | Holder access with secret in a query to C4-M1, retrievable several times (FRAMEWORK 4.6: "credentials can be retrieved only once") | The issuer needs the access for every credential (BPN, membership, later more). Only C4-M1, own permission, response not in the log, audit log entry per retrieval. |

Clarification, not a deviation, but to be agreed with Process Orchestration: a long-running step responds `RUNNING` even if it completes in the same call; the end event is then already in the outbox and can arrive before the response. `DONE` only if the object was already complete at the time of the call (README 6.9).

## Additions to the framework

Taken over into FRAMEWORK 5 to 7 on 28 Oct 2026 (events with their fields, queries, flows); the table stays as a record.

| No. | Addition | Reason |
|---|---|---|
| E1 | Queries `GET /internal/wallets/{companyId}`, `GET /internal/technical-users/{technicalUserId}/wallet-account` and `GET /internal/wallets/{companyId}/holder-access` (C4-M1 only) (README 9) | FRAMEWORK 7 names only two queries and refers to section 9 of the modules. |
| E2 | Queries to other modules: C3-M1 `GET /internal/companies/{companyId}` (status, BPN, name); later C1-M2 `GET /internal/applications/{applicationId}` with `applicantSub` (README 11.5, 11.6) | Preconditions of `create-wallet` and `create-wallet-account`; binding of the control proof to the application. |
| E3 | Additional fields in events: `walletId`, `provider`, `didDocumentLocation`, `bpn` in `DataspaceWalletCreated`; `walletId`, `provider`, `reason` in `DataspaceWalletFailed`; `provider` or `reason` in the events of the wallet accounts | FRAMEWORK 6 names the minimum content. |

## Assumptions about other modules and systems

| Who | Assumption |
|---|---|
| Process Orchestration | calls `create-wallet` after `CompanyIdentifierAssigned` with `companyId` (FRAMEWORK 5.2); after `DataspaceWalletCreated` calls C3-M2 `register-did` with the DID from the event; handles an end event that arrives before the response `RUNNING`; monitors the runtime of the flows and triggers again after `DataspaceWalletFailed`; starts `create-wallet-account` on `WalletAccessRequested` (FRAMEWORK 5.7) |
| C3-M1 Company Profile | provides `name`, `bpn` and `status` through `GET /internal/companies/{companyId}`; a deactivated company has a status other than `ACTIVE` |
| C3-M2 CSP Connection | has registered the company at the CSP with this OSP as home OSP before `create-wallet` runs (BPN present); enters BPN → DID with `register-did` |
| C2-M3 Technical Users | `apply-wallet-access` takes over `walletAccountRef` and the result from `WalletAccountCreated` or `WalletAccountFailed`; C2-M3 stores no wallet credentials (decision of 21 Oct 2026); can use `supportsWalletAccounts` from 9.1; creates no technical users for deactivated companies |
| C4-M1 Credentials | starts after `CompanyDidRegistered`; reads the wallet through 9.1 and the holder access through 9.3 for all issuance modes; makes the holder known to the issuer and itself triggers the request in the IdentityHub and the dataspace wallet of the CSP; itself receives the status report "credential issued", including the one from the dataspace wallet of the CSP. Aligned with the draft of C4-M1 (version 0.2, 28 Oct 2026): `provider` with `CSP`, `issuanceMode`, holder access `HOLDER_REQUESTS` in the same form for IdentityHub and CSP. |
| C1-M2 Registration | later, for the own wallet: assigns the `companyId` when the application is created and provides `companyId` and `sub` of the applicant for an application |
| C2-M5 Portal IAM | creates the permissions; the portal roles `IT Admin` and `Company Admin` contain `view_dataspace_wallet` and `retrieve_wallet_credentials`; service account of the SAP middleware with `submit_wallet_callback` |
| CSP | offers the wallet API (proposal, README 11.3); checks the home OSP on every call; accepts the same key of the OSP as the core services; issues an API key for the Identity API per participant context; the dataspace wallet of the CSP reports arrived credentials to C4-M1 with its own service account |
| Test tools | The shared CSP stand-in for the hackathon also offers the wallet API (still to be built); Tractus-X wallet stub for SAP DIV |

## Open questions

Questions that concern several modules are listed with their number (OP-xx) in `../open-points.md`.

1. **IdentityHub, Identity API:** Are the paths and fields (v1alpha) right for the version in use? Does creating a participant context return the STS client ID and secret? Does the IdentityHub allow several STS accesses per participant context (then a separate account per technical user instead of a copy)?
2. **SAP DIV:** Can the callback paths be set in the middleware? Is `setup-dim` harmless when repeated for the same BPN? Can DIV report errors (today only success callbacks)? Does the Tractus-X wallet stub cover `setup-dim`, `technical-user` and the callbacks in the version in use, and can the callback addresses be set there?
3. **Wallet API of the CSP:** Who defines it (association, CSP)? Does it need long-running operations with a callback? Then the dataspace wallet of the CSP would need a second permission in the Portal IAM of the OSP (`submit_wallet_callback` next to `report_credential_status` for C4-M1); FRAMEWORK 4.2 provides for one permission per service account.
4. **Transfer to another OSP:** The dataspace wallet does not move along (FRAMEWORK 5.6). What happens to a wallet at the CSP when the company moves to an OSP that also uses the dataspace wallet of the CSP? Cross-module, OP-38.
5. **Operation of the dataspace wallets and interplay of the providers** (SAP DIV, IdentityHub, dataspace wallet of the CSP): cross-module, OP-30.
6. **Own wallet, credentials and checks:** cross-module, OP-29.
7. **Installation settings** (`ospId`): cross-module, OP-36.
8. **Sign-in at central services:** The module signs in at the CSP with the key of the OSP; whether central services instead accept tokens from the Portal IAMs of several OSPs: cross-module, OP-37.

## Out of scope

- Deleting a wallet (company declined or left) and removing the entry in the directory BPN → DID.
- Changing the provider (e.g. SAP DIV → dataspace wallet of the CSP) and moving the credentials.
- Transfer of the dataspace wallet to another OSP (FRAMEWORK 5.6).
- Key rotation and revocation in the DID document.
- Taking over existing wallets from Portal 2.6.0 (`company_wallet_data`, `companies.did_document_location`, technical users in DIM).
- Managed Identity Wallet and Custodian (superseded).
- Several wallets per company.
- DID methods other than `did:web`.
