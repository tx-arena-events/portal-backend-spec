# C6-M1 Connectors

**Version:** 0.2, as of 2026-10-30 (OSP 2.0)
**Capability:** C6 Technical integration
**Kind:** Hackathon
**Files:** this README (behaviour), [openapi.yaml](openapi.yaml) (interface), [open-points.md](open-points.md)

This README and the OpenAPI description are sufficient to build the module. The binding framework is [FRAMEWORK.md](../FRAMEWORK.md).

## 1. Purpose

A company takes part in the dataspace with one or more dataspace connectors. The module keeps the register of the connectors of the companies of the installation: name, endpoint, location, status, the technical user the connector uses, and for a connector run by a provider the subscription it belongs to.

Finding the connectors of a business partner (discovery) is a core service of the CSP (FRAMEWORK 1). This module does not answer discovery requests. It reports changes of its connectors as events; Process Orchestration then enters the connectors in the discovery of the CSP or removes them through C3-M2 (FRAMEWORK 5.9).

Two kinds of connectors:

- **Company connector** (`COMPANY_CONNECTOR`): run by the company itself; provider and host are the same company.
- **Connector as a service** (`CONNECTOR_AS_A_SERVICE`): a provider runs the connector for a customer of one of its offers; it is linked to that subscription (C10-M1).

Callers:

| Who | What for |
|---|---|
| People of a company (UI) | register, view, change, delete own connectors; view connectors that a provider runs for the company |
| People of a provider company (UI) | register a connector as a service for a customer |
| Process Orchestration | steps: deactivate the connectors of an ended or declined subscription, remove the link to a deleted technical user (section 8) |
| C2-M3 Technical Users | query: connectors that use a technical user, before the account is deleted (section 9) |

## 2. Terms

| Term | Meaning |
|---|---|
| Connector | dataspace connector of a company, e.g. an Eclipse Dataspace Connector; ID `connectorId` |
| Provider company | company that runs the connector |
| Host company | company whose identity the connector represents in the dataspace |
| Endpoint | URL at which other connectors reach this connector |
| Location | country of the connector, ISO 3166-1 alpha-2 |
| Discovery | core service of the CSP that finds the connectors of a business partner |

## 3. Not part of this module

| What | Where |
|---|---|
| Discovery of connectors, entering connectors there | core services of the CSP; C3-M2 `register-connector`, `deregister-connector` (FRAMEWORK 5.9) |
| Directory company identifier → DID | core services of the CSP, through C3-M2 |
| Running or deploying connector software | the company or its provider |
| Technical users, their secrets and wallet access | C2-M3, C4-M2 |
| Self-descriptions and clearing house | dropped |
| MCP server for AI agents | C6-M2 (separate spec) |

## 4. Technical frame

- Module of the modular monolith (Python, FastAPI), own database `c6_m1` in PostgreSQL.
- People call with their token from the Portal IAM; scope `company` against provider or host company. A token without `company_id` gets 403 `forbidden` for these operations.
- Events through the outbox of the module.

### Configuration

| Variable | Meaning | Example |
|---|---|---|
| `DATABASE_URL` | own database `c6_m1` | |
| `COMPANY_URL` | C3-M1: empty = built-in stand-in, `in-process`, or HTTP address (FRAMEWORK 4.7) | `in-process` |
| `TECHNICAL_USER_URL` | C2-M3 | `in-process` |
| `SUBSCRIPTION_URL` | C10-M1 | `in-process` |
| `SETTINGS_FILE` | installation settings with the country list (OP-13) | `config/installation.json` |

## 5. Data

### Table `connector`

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key; `connectorId` |
| `name` | text | 1 to 255 characters |
| `endpoint` | text | HTTPS URL |
| `location` | text | country code from the reference data |
| `type` | text | `COMPANY_CONNECTOR`, `CONNECTOR_AS_A_SERVICE` |
| `status` | text | 6.1 |
| `provider_company_id`, `host_company_id` | uuid | equal for `COMPANY_CONNECTOR` |
| `technical_user_id` | uuid | optional; account of the host company (C2-M3) |
| `offer_subscription_id` | uuid | only `CONNECTOR_AS_A_SERVICE` |
| `created_at`, `changed_at` | timestamptz | |

Unique: `name` with `endpoint`; `technical_user_id` (one connector per account) and `offer_subscription_id` (one connector per subscription), both only among connectors that are not `INACTIVE`.

### Table `audit_entry`

As FRAMEWORK 4.3.

**Not stored:** company identifier (BPN) and DID; both are at C3-M1 and the CSP.

## 6. Rules

### 6.1 Status of a connector

| From | Event | To | Event published |
|---|---|---|---|
| – | registered (7.2, 7.3) | `ACTIVE` | `ConnectorRegistered` |
| `ACTIVE` | endpoint changed (7.5) | `ACTIVE` | `ConnectorEndpointChanged` |
| `ACTIVE` | subscription ended or declined (8.1), or deleted while a subscription is linked (7.6) | `INACTIVE` | `ConnectorDeactivated` |
| `ACTIVE`, `INACTIVE` | deleted (7.6), no subscription linked | – | `ConnectorDeleted` |

`INACTIVE` connectors are kept for traceability and left out of lists.

### 6.2 Checks

- **Host company:** status `ACTIVE` in C3-M1. The company identifier is not checked here; its format is not fixed in this module.
- **Duplicate:** same `name` and `endpoint`: 409 `duplicate`.
- **Location:** known country code, otherwise 400.
- **Technical user (optional):** C2-M3 `/internal/technical-users/{id}?companyId=<host>` answers `usableForConnector: true`, and the account is not linked to another connector; otherwise 409 `technical-user-not-usable`. For a connector as a service (7.3) the call also passes `offerSubscriptionId`; then the customer account of this subscription (origin `SUBSCRIPTION`, C2-M3 9.1) is usable as well.
- **Connector as a service:** the subscription exists (C10-M1), the offer belongs to the calling company (otherwise 404), status `PENDING` or `ACTIVE` (otherwise 409 `invalid-state`), no other connector that is not `INACTIVE` linked (409 `subscription-linked`).
- **Change:** only the provider company changes the endpoint; not in `INACTIVE`.
- **Delete:** provider or host company. While the linked subscription is `ACTIVE`: 409 `subscription-active`.

### 6.3 Permissions

| Permission | Scope | What for |
|---|---|---|
| `view_connectors` | `company` | read connectors of which the own company is provider or host |
| `manage_connectors` | `company` | register, change, delete them |
| `execute_steps_c6_m1` | `osp` | steps over HTTP (FRAMEWORK 4.2) |
| `query_c6_m1` | `osp` | queries over HTTP (section 9, FRAMEWORK 4.2) |

The permissions are listed in C2-M5 6.2; `view_connectors` and `manage_connectors` are in the portal roles `IT Admin` and `Company Admin` (C2-M5 6.3).

### 6.4 Repeatability

Both steps find their objects by `offerSubscriptionId` or `technicalUserId` and change only what is still to be changed. Operations of people are not idempotent.

## 7. Operations

Lists as in FRAMEWORK 4.1. Tag `Company`.

### 7.1 `GET /api/c6-m1/connectors`

Permission `view_connectors`. Parameter `role` (`provider`: connectors the own company runs, including its company connectors; `host`: connectors as a service that a provider runs for the own company; default both). Filters `type`, `status` (default without `INACTIVE`). Response 200 `ConnectorPage`.

### 7.2 `POST /api/c6-m1/connectors`

Permission `manage_connectors`. Input `name`, `endpoint`, `location`, optional `technicalUserId`. Company connector of the own company; checks 6.2. Response 201 `Connector`, event `ConnectorRegistered`.

### 7.3 `POST /api/c6-m1/connectors/as-a-service`

Permission `manage_connectors`. Input as 7.2 plus `offerSubscriptionId`; the technical user must belong to the customer (host). C6-M1 passes `offerSubscriptionId` to C2-M3 (6.2), so the customer account that C2-M3 created for this subscription can be used. Provider = own company, host = customer of the subscription. Response 201, event `ConnectorRegistered`.

### 7.4 `GET /api/c6-m1/connectors/{connectorId}`

Permission `view_connectors`, provider or host company. Response 200 `Connector`.

### 7.5 `PATCH /api/c6-m1/connectors/{connectorId}`

Permission `manage_connectors`, provider company. Input `endpoint`. Same value: 200 without change. Event `ConnectorEndpointChanged`.

### 7.6 `DELETE /api/c6-m1/connectors/{connectorId}`

Permission `manage_connectors`, provider or host company. Checks 6.2. With a linked subscription that is not active: link stays, status `INACTIVE`, event `ConnectorDeactivated`. Otherwise the row is removed, event `ConnectorDeleted`. The technical user stays; the company deletes it in C2-M3 if needed. Response 204.

## 8. Steps for Process Orchestration

Input always with `processId` and `stepRunId` (FRAMEWORK 4.4).

### 8.1 `POST /api/c6-m1/steps/deactivate-subscription-connectors`

- **Flow:** `subscription-ended`, after `SubscriptionDeactivated`; `subscription-declined`, after `SubscriptionDeclined` (a connector as a service may exist for a `PENDING` subscription).
- **Input:** in addition `offerSubscriptionId`.
- **Precondition:** none; without a linked connector the step does nothing.
- **Behaviour:** the connector of the subscription becomes `INACTIVE`, the link to the technical user is removed.
- **Result:** `DONE` with `connectorIds`. **Event:** `ConnectorDeactivated` per changed connector.

### 8.2 `POST /api/c6-m1/steps/unlink-technical-user`

- **Flow:** `technical-user-deleted`, after `TechnicalUserDeleted` (C2-M3).
- **Input:** in addition `technicalUserId`.
- **Behaviour:** removes `technical_user_id` from the connector that uses it; the connector stays `ACTIVE`.
- **Result:** `DONE` with `connectorIds` (empty if none).

## 9. Queries for other modules

Connectors of a company are read by people through 7.1; discovery is at the CSP.

### 9.1 `GET /api/c6-m1/internal/connectors`

Permission `query_c6_m1`. Parameter `technicalUserId` (required). Response 200: list of `{ "connectorId", "status" }` of the connectors that use this technical user and are `ACTIVE`; empty list if none. Used by C2-M3 before it deletes a technical user (C2-M3 7.5: a connector uses the account → 409 `in-use`). `INACTIVE` connectors do not count; step 8.2 removes their link after the account is deleted.

## 10. Events

| Event | When | Content |
|---|---|---|
| `ConnectorRegistered` | registered (7.2, 7.3) | `connectorId`, `companyId` (host company), `providerCompanyId`, `type`, `endpoint`, `location` |
| `ConnectorEndpointChanged` | endpoint changed (7.5) | `connectorId`, `companyId`, `endpoint` |
| `ConnectorDeactivated` | status `INACTIVE` (7.6, 8.1) | `connectorId`, `companyId` |
| `ConnectorDeleted` | removed (7.6) | `connectorId`, `companyId` |

Every event also carries `eventId`, `eventType`, `occurredAt`. They start the flows of FRAMEWORK 5.9: `ConnectorRegistered` and `ConnectorEndpointChanged` lead to C3-M2 `register-connector` (`endpoint` is passed as `endpointUrl`), `ConnectorDeactivated` and `ConnectorDeleted` to `deregister-connector` (open-points No. 1).

## 11. Calls to other modules and external systems

| Neighbour (variable) | Call | Built-in stand-in |
|---|---|---|
| C3-M1 (`COMPANY_URL`) | `GET /api/c3-m1/internal/companies/{companyId}`: `status` | every company `ACTIVE`, unless `stub-companies.json` gives it another status (for acceptance case 3) |
| C2-M3 (`TECHNICAL_USER_URL`) | `GET /api/c2-m3/internal/technical-users/{id}?companyId=&offerSubscriptionId=` (`offerSubscriptionId` only for a connector as a service): `exists`, `usableForConnector` | every account usable |
| C10-M1 (`SUBSCRIPTION_URL`) | `GET /api/c10-m1/internal/subscriptions/{id}`: `status`, `providerCompanyId`, `customerCompanyId` | reads `stub-subscriptions.json` |

No external systems.

## 12. Errors

Problem Details, `type` = `urn:c6-m1:problem:<name>`.

| `type` | HTTP | When | `retryable` for steps |
|---|---|---|---|
| `validation` | 400 | invalid input, unknown country | `false` |
| `unauthorized` | 401 | token missing or invalid | – |
| `forbidden` | 403 | permission missing; change by the host company | – |
| `not-found` | 404 | connector or subscription unknown or not visible | `false` |
| `company-not-active` | 409 | host company not `ACTIVE` | – |
| `duplicate` | 409 | same name and endpoint | – |
| `technical-user-not-usable` | 409 | technical user unknown, of another company, disabled or already linked | – |
| `subscription-linked` | 409 | the subscription already has a connector | – |
| `subscription-active` | 409 | delete while the linked subscription is `ACTIVE` | – |
| `invalid-state` | 409 | change of an `INACTIVE` connector; connector as a service for a subscription that is not `PENDING` or `ACTIVE` | – |
| `upstream-error` | 502 | neighbour not reachable | `true` |

## 13. Acceptance

1. Register a company connector with a usable technical user: 201, `ACTIVE`, `ConnectorRegistered` with `companyId`.
2. The same name and endpoint again: 409 `duplicate`. The same technical user for a second connector: 409 `technical-user-not-usable`.
3. Host company not `ACTIVE`: 409 `company-not-active`. A company without BPN but `ACTIVE`: 201 (no BPN check).
4. Connector as a service for a subscription of an own offer: 201, provider = own company, host = customer. For a subscription of another provider: 404.
5. The customer sees the connector with `role=host`; another company: 404.
6. Change the endpoint as host company: 403; as provider: 200, `ConnectorEndpointChanged`.
7. Delete while the subscription is `ACTIVE`: 409. After the end of the subscription the step 8.1 sets it `INACTIVE`, `ConnectorDeactivated`; delete then removes nothing and answers 204.
8. Step 8.2 with the ID of a linked technical user: link removed, connector stays `ACTIVE`; the same step again: `DONE` with an empty list.
9. `GET /internal/connectors?technicalUserId=` with the technical user of an `ACTIVE` connector: one entry; after step 8.2 or for an unknown account: empty list; without `query_c6_m1`: 403.
11. Connector as a service with the customer account that C2-M3 created for the subscription (origin `SUBSCRIPTION`): C6-M1 calls C2-M3 9.1 with `offerSubscriptionId`, 201. The same account for a company connector of the customer (7.2, without `offerSubscriptionId`): 409 `technical-user-not-usable`.
10. A connector as a service of a `PENDING` subscription is deleted (`INACTIVE`, link stays): a new connector as a service for the same subscription is accepted (201).

## 14. Test data

| Object | Value |
|---|---|
| Company (host) | Beispiel Maschinenbau GmbH `c0000001-0000-4000-8000-000000000001` (FRAMEWORK 7a) |
| Provider company (company of OSP A, FRAMEWORK 7a) | `11111111-1111-4111-8111-111111111111` |
| Technical user of the host | `c3d4e5f6-a7b8-4c9d-8e0f-1a2b3c4d5e6f` |
| Subscription (service of the provider) | `d4e5f6a7-b8c9-4d0e-9f1a-2b3c4d5e6f70` |
| Connector | `connectorId` `7c1e5a2b-4d3f-4e6a-9b8c-0d1e2f3a4b5c` (as in C3-M2 14), name `edc-beispiel`, endpoint `https://edc.beispiel-maschinenbau.example/dsp`, location `DE` |

## Changes from version 0.1

- Discovery dropped: it is a core service of the CSP (OSP 2.0). `POST /connectors/discovery` is removed. C3-M2 enters the connectors at the CSP (FRAMEWORK 5.9).
- No BPN check; the host company must be `ACTIVE` (the format of the company identifier is not fixed, workshop of 27 Oct 2026).
- Reactions to `SubscriptionDeactivated` and `TechnicalUserDeleted` are steps of Process Orchestration (flows `subscription-ended`, `technical-user-deleted`), not own event handlers.
- `GET /connectors/subscription-links` dropped; providers find their subscriptions in C10-M1.
- Deleting a connector no longer deletes its technical user (`deleteTechnicalUser` dropped); the company does that in C2-M3.
- Paths under `/api/c6-m1`, `POST /connectors/managed` becomes `/connectors/as-a-service`, `PUT` of the endpoint becomes `PATCH`; permissions `view_connectors`, `manage_connectors` instead of `add_connectors`, `modify_connectors`, `delete_connectors`; `execute_steps_c6_m1` for steps over HTTP (C2-M5 6.2).
- Self-description fields and the clearing house dropped.
- `NotificationRequested` dropped; Notifications reacts to `ConnectorRegistered`.
- Query `GET /internal/connectors?technicalUserId=` for C2-M3 (9.1), permission `query_c6_m1`.
- Step 8.1 also runs after `SubscriptionDeclined` (flow `subscription-declined`); `INACTIVE` connectors no longer block a new connector for the same subscription or account.
- Permissions with the full module identifier as in FRAMEWORK 4.2: `execute_steps_c6_m1` (renamed from `execute_steps_c6`), `query_c6_m1` (renamed from `query_c6`); tag `Queries` instead of `Internal` (FRAMEWORK 4.1) (30 Oct 2026).
- Connector as a service: the check of the technical user passes `offerSubscriptionId` to C2-M3 9.1, so the customer account of the subscription is usable (6.2, 7.3, case 11) (30 Oct 2026).
- Test data with the common test IDs of FRAMEWORK 7a (host company Beispiel Maschinenbau GmbH) (30 Oct 2026).
