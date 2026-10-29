# C3-M2 CSP Connection: open points

**Version:** 0.1, as of 2026-10-29

The module is new with OSP 2.0. The README fixes every point so that the module can be built; the following decisions are provisional.

## Provisional decisions

| No. | Topic | Decision in the README | To be agreed with |
|---|---|---|---|
| 1 | Core services API | own proposal: register, search and transfer companies; DID, membership and connectors per BPN (README 11.1) | association, CSP |
| 2 | Authentication at the CSP | client credentials with `private_key_jwt`, `client_id` = `ospId`, key in the trust list (README 4) | CSP |
| 3 | Callbacks from the CSP | service account of the CSP in the Portal IAM of the OSP with the permission `report_csp_notification`; address of the installation in the trust list (README 4, 11.1) | CSP, C2-M5 |
| 4 | Long-running operations at the CSP | 202 with `operationId`, end by the callback `OperationCompleted`; no polling (README 8.1) | CSP |
| 5 | Assignment of the BPN | by the CSP (today from the master data service); for a known EBWOID, the CSP confirms the existing BPN | CSP |
| 6 | Duplicate check | compared at the CSP by EBWOID, EUID, another identifier (type and value), or name and country; assessed in C1-M3 (README 6.2) | CSP, C1-M3 |
| 7 | Transfer | `transfer` at the CSP; the CSP notifies the old home OSP; data does not move along (README 8.2, FRAMEWORK 5.6) | project owner, CSP |
| 8 | Stand-in | BPN from the hash of the EBWOID, otherwise of the EUID, otherwise of the first identifier; companies of other OSPs from `CSP_STUB_FILE` (README 11.3) | – |
| 9 | Identifiers at registration | `ebwoid` optional; required is `ebwoid`, `euid` or an identifier in `identifiers`, because without a Business Wallet (OP-35) there is no EBWOID (README 6.5, 8.1); the same for the transfer (8.2). Pointed out by the other part of the spec on 29 Oct 2026 | CSP, C1-M2 |
| 10 | Connectors in the discovery | steps `register-connector` and `deregister-connector`; the CSP names only connectors of companies with an active membership, so there is no own step on deactivation (README 6.6, 8.6). Requested by the other part of the spec on 29 Oct 2026 | CSP, Connectors (other part) |

## Assumptions about other modules and systems

| Who | Assumption |
|---|---|
| Process Orchestration | passes the data of the company as input to the steps (from the application or from C3-M1) |
| C1-M3 | calls `find-companies` in the automatic pre-check and assesses the matches |
| C3-M1 | takes over the BPN from `CompanyRegisteredAtCsp` through the step `assign-identifier` |
| C4-M2 | reports the DID with `DataspaceWalletCreated`; Process Orchestration then calls `register-did` |
| CSP | keeps the company identifier, home OSP, directory BPN → DID and membership; checks the home OSP on every changing call |
| Connectors (other part) | reports creating, changing the address of and deleting a connector as an event; Process Orchestration then calls `register-connector` or `deregister-connector` with `connectorId` and `endpointUrl` |
| Test tools | provide a shared CSP stand-in with the same core services API for the hackathon (still to be built) |

## Open questions

1. **Who defines the core services API?** The association, the CSP or the community; until then, the proposal applies.
2. **Companies with an existing BPN** from today's dataspace: taking over the BPN at the first registration through an OSP.
3. ~~**Discovery:**~~ Agreed with the other part of the spec on 29 Oct 2026: C3-M2 reports connectors to the CSP (README 8.6). Still open is the alternative of finding connectors through an entry in the DID document of the dataspace wallet instead of a list at the CSP (OP-24). Services other than connectors are not considered.
4. **Several CSPs:** federation of the company identifier and of the trust lists (later work).
5. **Transfer without a Business Wallet:** in a transfer, C1-M3 recognises the company only by EBWOID or EUID. Without a Business Wallet and without an EUID no identifier matches; should another identifier (such as the VAT ID) be enough?

## Out of scope

- Transfer of data, technical users and dataspace wallet.
- Billing between OSP and CSP.
