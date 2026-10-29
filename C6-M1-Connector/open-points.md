# C6-M1 Connectors: open points

**Version:** 0.2, as of 2026-10-29

## Provisional decisions

| No. | Topic | Decision in the README | To be agreed with |
|---|---|---|---|
| 1 | Connectors in the discovery of the CSP | the module publishes events (README 10) and calls nothing at the CSP. Flows of C12-M1 call C3-M2 `register-connector` on `ConnectorRegistered` and `ConnectorEndpointChanged`, `deregister-connector` on `ConnectorDeactivated` and `ConnectorDeleted` (FRAMEWORK 5.9). An `INACTIVE` connector is no longer used, so it is also removed from the discovery | C3-M2, CSP (OP-42) |
| 2 | Company identifier | not checked and not stored; the host company must be `ACTIVE` (workshop of 27 Oct 2026: a dataspace profile without BPDM and BPN should be possible) | – |
| 3 | Reactions to other modules | steps for Process Orchestration instead of own event handlers (FRAMEWORK 1) | C12-M1 |
| 4 | Technical user on delete | stays; the company deletes it in C2-M3 | C2-M3 |
| 5 | Status `PENDING` | dropped; without a clearing house a connector is `ACTIVE` at once | – |

## Assumptions about other modules

| Who | Assumption |
|---|---|
| C3-M2 | steps `register-connector` (`companyId`, `connectorId`, `endpointUrl`) and `deregister-connector` (`companyId`, `connectorId`), C3-M2 8.6; `companyId` is the host company; the company must have the status `REGISTERED` at the CSP |
| C2-M3 | `/internal/technical-users/{id}?companyId=` with `usableForConnector` (C2-M3 9.1); event `TechnicalUserDeleted` with `technicalUserId`, `companyId` |
| C10-M1 | `/internal/subscriptions/{id}` with `status`, `providerCompanyId`, `customerCompanyId` |
| C12-M1 | flows `subscription-ended` (step 8.1), `technical-user-deleted` (step 8.2) and `connector-registered`, `connector-endpoint-changed`, `connector-deactivated`, `connector-deleted` (No. 1) |
| C2-M5 | `view_connectors`, `manage_connectors` in the portal roles `IT Admin` and `Company Admin`; `execute_steps_c6` for the service account of Process Orchestration (C2-M5 6.2, 6.3) |
| C11-M1 | rule R16 on `ConnectorRegistered` |

## Open questions

1. **Connector and dataspace wallet:** which data of the dataspace wallet (DID, credential service) does a connector need at registration? Today the portal does not check this; C4-M2 holds the wallet.
2. **Several subscriptions per connector:** today at most one subscription per connector as a service. Keep?
3. **Format of the discovery entry at the CSP:** C3-M2 proposes the address of the connector per BPN (C3-M2 11.1); the CSP decides (OP-42). Alternative: an entry in the DID document of the dataspace wallet (OP-24).
4. **Company configuration** (listed for this capability in the architecture proposal): part of C3-M1 or here?

## Legacy endpoints (portal backend 2.6.0, `/api/administration/Connectors/…`)

| Legacy | New |
|---|---|
| `GET` (root), `GET provided` | `GET /connectors?role=provider` |
| `GET managed` | `GET /connectors?role=host` |
| `GET {connectorId}` | `GET /connectors/{connectorId}` |
| `POST` (root, multipart) | `POST /connectors` (JSON) |
| `POST managed` (multipart) | `POST /connectors/as-a-service` (JSON) |
| `PUT {connectorId}/connectorUrl` | `PATCH /connectors/{connectorId}` |
| `DELETE {connectorId}?deleteServiceAccount=` | `DELETE /connectors/{connectorId}` (technical user stays) |
| `POST discovery` | dropped; discovery of the CSP (OP-24) |
| `GET offerSubscriptions?connectorIdSet=` | dropped; C10-M1 `GET /subscriptions?view=provider` |
| `POST clearinghouse/selfDescription`, `GET missing-sd-document`, `POST trigger-self-description`, `POST retrigger-self-description`, `POST retrigger-self-description-response` | dropped (clearing house) |

## Out of scope

- Deployment and configuration of connector software.
- Self-descriptions.
