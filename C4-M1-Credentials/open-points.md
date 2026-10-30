# C4-M1 Credentials: open points

**Version:** 0.2, as of 2026-10-30

The [README](README.md) fixes each point for version 0.2, so that the module can be built. The following decisions are provisional and still to be agreed. Decided on 28 Oct 2026: OSP 2.0 (each OSP runs its own installation, the CSP provides the core services and, optionally, an issuer); in the hackathon, OSP A issues credentials itself and OSP B uses the CSP issuer.

## Provisional decisions

| No. | Topic | Decision in the README | To be agreed with |
|---|---|---|---|
| 1 | Order of the credentials | BPN credential and membership credential are triggered at the same time (6.2). Today the portal triggers the membership credential only after the BPN credential has succeeded. Whether the Credential Issuer copes with simultaneous requests is to be checked; otherwise a setting for "one after the other" is added. | OSPs with their own Credential Issuer |
| 2 | Issuer mode | one setting per installation: `ISSUER_MODE` `OWN` or `CSP`; with `OWN` the route per wallet provider through `ISSUER_ROUTES`, with `CSP` always `CSP_ISSUER` (6.3). No mixed operation of own issuer and CSP issuer in one installation. | project owner, OSPs, CSP |
| 3 | Issuer API of the CSP | own proposal: create the holder (`PUT /holders/{bpn}`), grant per credential type (`PUT`/`DELETE …/grants/{kind}`), issuance via DCP, callback with `{ bpn, status, message }` (11.5) | CSP, association |
| 4 | Sign-in at the CSP issuer | client credentials with `private_key_jwt`, `client_id` = `ospId`, the same key and token endpoint as in C3-M2 (4) | CSP, C3-M2 |
| 5 | CSP issuer via DCP only | The CSP issuer issues only into wallets with issuance mode `HOLDER_REQUESTS` (IdentityHub, dataspace wallet of the CSP). For SAP DIV, an OSP needs its own Credential Issuer; the holder access of a wallet never goes to the CSP (6.3, 4). | CSP, project owner |
| 6 | Checks at the CSP | The CSP issuer grants only if the calling OSP is the home OSP and the DID is in the directory BPN → DID; that is why `issue-credentials` runs after `CompanyDidRegistered` (6.4, 8.1) | CSP, Process Orchestration |
| 7 | Access to the holder's wallet | C4-M1 fetches the holder access from C4-M2 once per run (Credential Issuer: write access with secret; DCP: API key of the participant context), keeps it only in memory and passes it on only to its own Credential Issuer (4, 11.2) | C4-M2 |
| 8 | Request the credential in the wallet | C4-M1 calls the identity interface of the holder's wallet (IdentityHub or dataspace wallet of the CSP) directly, although the wallet otherwise goes through C4-M2. Reason: modules call each other directly only for queries, and the step catalogue has no step at C4-M2 for this (11.4). | C4-M2, architecture |
| 9 | Format and assignment of the status report | as today: `{ bpn, status, message }`, assignment by BPN and credential type, without an identifier of the order; the same for all senders, also for the CSP (7.4) | Credential Issuer, callback adapter, CSP |
| 10 | Sender of the status report | per route: own Credential Issuer; callback adapter of the wallet (own IdentityHub or dataspace wallet of the CSP); CSP issuer. One service account per system with `report_credential_status` (6.5). With the route `CSP_ISSUER`, both issuer and wallet report; the second report changes nothing. | C2-M5, CSP, OSPs |
| 11 | Path of the status report | `/api/c4-m1/issuer-callbacks/bpn` or `/membership`; the ingress of the installation redirects the old paths of the portal until the callback adapter for the IdentityHub is configurable (11.4) | OSPs (operations) |
| 12 | Hanging orders | no time limit in the module, no polling. A repetition of the step sends an open order again if it is older than `RESEND_AFTER_HOURS` (24 h) (8.1). | Process Orchestration |
| 13 | Late status reports | success after `FAILED` counts (`ISSUED`); success after `CANCELLED` leads to an immediate revocation (6.1, 7.4) | OSPs |
| 14 | Revocation | Without input, `revoke-credentials` revokes only the membership credential (OP-07). Route `CREDENTIAL_ISSUER`: in 0.2 not through the interface, but by hand at the Credential Issuer with confirmation by OSP staff (`REVOCATION_PENDING`, 7.3). Route `CSP_ISSUER`: remove the grant (8.2, 11.5). | OSPs, operation of the Credential Issuer, CSP |
| 15 | Reactivation | Process Orchestration passes `credentialTypes` = `["MEMBERSHIP"]`; issued types are never triggered again (6.2, 8.1). A membership credential in `REVOCATION_PENDING` blocks the reactivation until OSP staff confirm the revocation. | Process Orchestration, OSPs |
| 16 | Identifiers with DCP | holder ID = BPN; `holderPid` = `{companyId}:{credentialType}:{attempt}`, so that a new order has a new identifier (11.4, 11.5) | operation of IdentityHub and IssuerService, CSP |
| 17 | Register the holder with the issuer | done by C4-M1 once per run before the first request, not by C4-M2 (11.4, 11.5) | C4-M2, CSP |
| 18 | `memberOf` | from `dataspace.memberOf` in the installation settings (OP-14, OP-36). Credential Issuer: in the order; own IssuerService: property of the holder that the credential definition takes over; CSP issuer: in the grant, the CSP checks the dataspace (11.3 to 11.5) | project owner, operation of the IssuerService, CSP |
| 19 | Precondition for the company | status `REQUESTED` (onboarding) or `ACTIVE` (reactivation) with BPN; a deactivated company gets no credentials (8.1) | C3-M1, Process Orchestration |
| 20 | View of the people of the company | only status and times, no messages of the issuer and no routes (7.1) | project owner |
| 21 | Format with DCP | `CREDENTIAL_FORMAT` = `VC1_0_JWT` for the own IssuerService and the CSP issuer | operation of IdentityHub and IssuerService, CSP |
| 22 | Permissions and scopes | `view_credentials` (`company`; portal roles `Company Admin`, `IT Admin`), `manage_credentials` (`osp`; `OSP Admin`), `report_credential_status` (`osp`; service accounts only), `query_c4_m1`, `execute_steps_c4_m1` (`osp`) (6.5) | C2-M5 Portal IAM |
| 23 | Event on failure | one per failed credential type, with the truncated message of the issuer as `reason` (10) | Process Orchestration |
| 24 | Business refusal by the CSP | 409 `issuer-rejected`, `retryable: false`; technical errors 502 `upstream-error`, `retryable: true` (11.5, 12) | Process Orchestration (OP-18) |

## Deviations from the framework

None. Since 28 Oct 2026, FRAMEWORK 4.1 and 4.3 cover scope `osp` for steps, queries and callbacks, as well as separate tags for callbacks.

## Additions to the framework

Adopted into FRAMEWORK 5 to 7 on 28 Oct 2026 (events with their fields, queries, flows); the table remains as a record.

| No. | Addition | Reason |
|---|---|---|
| E1 | Query `GET /internal/companies/{companyId}/credentials` (README 9.1) | FRAMEWORK 7 names only two queries and refers to section 9 of the modules. |
| E2 | Queries to other modules: C3-M1 `get_company`, C4-M2 `get_wallet` and `get_holder_access` (README 11.1, 11.2) | Preconditions and route of the step `issue-credentials`; C3-M1 and C4-M2 describe the queries in their section 9. |
| E3 | Additional fields in events: `reason` with `CredentialIssuanceFailed`, `revocationReason` with `CredentialRevoked` | FRAMEWORK 6 names the minimum content. |
| E4 | Optional input `credentialTypes` for `issue-credentials` and `revoke-credentials` | Reactivation (membership credential only) and deactivation (default: membership credential only) with the same steps. |

## Assumptions about other modules and systems

| Who | Assumption |
|---|---|
| C4-M2 Dataspace Wallet Interface | offers `get_wallet(companyId)` with `status` (`ACTIVE` = ready), `provider` (`SAP_DIV`, `IDENTITY_HUB`, `CSP`, `BRING_YOUR_OWN`), `issuanceMode`, `did`, `didDocumentLocation`, `providerRef`; offers `get_holder_access(companyId)` only for C4-M1: with `ISSUER_WRITES` address, client ID and secret of the wallet, with `HOLDER_REQUESTS` identity interface, participant context and API key, in the same form for the IdentityHub and the dataspace wallet of the CSP. In the IdentityHub, the identifier of the participant context is the BPN (the callback adapter reports with this identifier). |
| C3-M1 Company Profile | returns status, BPN and name through `GET /internal/companies/{companyId}`; a deactivated company has a status other than `REQUESTED` or `ACTIVE`; reports `CompanyDeactivated` and `CompanyReactivated`. |
| C3-M2 CSP Connection | enters the DID in the directory BPN → DID with `register-did` before Process Orchestration starts `issue-credentials`; registers the membership after `CredentialsIssued`. Uses the same key of the OSP and the same token endpoint of the CSP. |
| C2-M5 Portal IAM | creates the permissions; portal roles `Company Admin` and `IT Admin` contain `view_credentials`, `OSP Admin` contains `manage_credentials`; one service account each with `report_credential_status` for the own Credential Issuer, the callback adapter in the own IdentityHub, the dataspace wallet of the CSP and the CSP issuer, the two of the CSP with `private_key_jwt` and the keys of the CSP. |
| Process Orchestration | calls `issue-credentials` in onboarding after `CompanyDidRegistered` and at reactivation after `CompanyReactivated` with `["MEMBERSHIP"]`; calls `revoke-credentials` after `CompanyDeactivated`; triggers the step again after `CredentialIssuanceFailed` (OP-18). |
| CSP | offers the issuer API (proposal, README 11.5); checks home OSP, directory BPN → DID and dataspace; issues via DCP; calls back with its own service account, only to addresses of the installation from the trust list. The dataspace wallet of the CSP reports arrived credentials like a callback adapter. The CSP issuer is on the trust list. |
| Governance | when admitting an OSP that issues credentials itself, adds the DID and the key of its issuer to the trust list (FRAMEWORK 5.1). Without this entry, connectors of other companies do not accept the credentials. |
| Connectors (another part) | accept credentials of the issuers on the trust list. |
| Test tools | test servers for the Credential Issuer, the IssuerService with IdentityHub and the issuer API of the CSP. The hackathon does not need them: it runs on the stand-in `STUB` (OP-53). |
| Base skeleton | outbox, token validation, Problem Details; ingress of the installation with forwarding of the old status report paths. |

## Open questions

Questions that affect several modules are listed with their number (OP-xx) in `../open-points.md`.

1. **Revocation at the Credential Issuer:** Which interface does it offer, and which identifier of the credential does that interface need? Today's status report provides none. Until this is clarified, OSP staff revoke by hand.
2. **Callback adapter for the IdentityHub:** So far it exists only in a branch of a fork of the IdentityHub, not in the main line. Who maintains it, and will it be contributed to Tractus-X? An alternative would be an event interface in the IdentityHub itself.
3. **Interfaces of IdentityHub and IssuerService:** check paths, fields and identifiers against the version in use (11.4 is based on documentation of the previous connection); also how the IssuerService takes over `memberOf` from a property of the holder.
4. **Issuer API of the CSP:** Who defines it (association, CSP)? Can the CSP issuer send an offer (credential offer) to the wallet via DCP, so that C4-M1 no longer has to trigger the request in the wallet?
5. **Trust in own issuers:** How do the issuers of the trust list get into the connectors (today configured per connector)? What happens to issued credentials when an issuer leaves the list?
6. **Change of the issuer mode:** Do credentials of the previous issuer stay valid, or does the new issuer issue them again? Until this is clarified, a revocation always uses the route of the order (6.3).
7. **Transfer to another OSP:** At the old OSP, the flow "deactivation" revokes the membership credential; whether the BPN credential stays in the old wallet and how the new OSP issues follows from the transfer (cross-module, OP-38).
8. **Own wallet (`BRING_YOUR_OWN`):** cross-module, OP-29.
9. **Operation of the dataspace wallets and interplay of the providers:** cross-module, OP-30.
10. **Trigger again after a failure:** cross-module, OP-18.
11. **Installation settings** (`ospId`, `dataspace.memberOf`): cross-module, OP-36.
12. **Sign-in at central services:** The module signs in at the CSP with the OSP's key; whether central services accept tokens from the Portal IAMs of several OSPs instead: cross-module, OP-37.

## Out of scope

- Expiry and renewal of credentials.
- Further credential types (use case participation, certificates) and credential-based issuance on the basis of the Business Wallet; see Outlook in the README.
- Display of the credentials themselves (today "wallet page"); the content is in the dataspace wallet.
- Taking over existing states from Portal 2.6.0 (checks `BPNL_CREDENTIAL` and `MEMBERSHIP_CREDENTIAL`).
- Several CSPs and credentials across the boundaries of a dataspace (later work, FRAMEWORK 1).
