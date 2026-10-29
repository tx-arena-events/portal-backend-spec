# C10-M1 Subscriptions

**Version:** 0.2, as of 2026-10-29 (OSP 2.0)
**Capability:** C10 Subscriptions
**Kind:** Hackathon
**Files:** this README (behaviour), [openapi.yaml](openapi.yaml) (interface), [open-points.md](open-points.md)

This README and the OpenAPI description are sufficient to build the module. The binding framework is [FRAMEWORK.md](../FRAMEWORK.md).

## 1. Purpose

A company subscribes to an active offer of the installation: an app or a service (C8-M1 Offers). The module keeps the subscription and its status, informs the provider, and lets the provider set up, activate or decline the subscription. Everything the subscriber needs to use the offer (client of the app instance, technical user) is created by other modules in the flow `subscription-setup` of Process Orchestration (FRAMEWORK 5.8).

One model for apps and services.

Callers:

| Who | What for |
|---|---|
| People of the subscribing company (UI) | subscribe, view, end a subscription |
| People of the provider company (UI) | view subscriptions to own offers, start the setup, activate, decline, change the instance URL, manage the provider configuration |
| Process Orchestration | steps: inform the provider, send the activation callback (section 8) |
| C2-M2 Users and App Access | query: active subscriptions of a company to an app (section 9) |
| C12-M1 Process Orchestration | events of this module start and continue the flows `subscription-requested`, `subscription-setup`, `single-instance-activation`, `subscription-declined`, `subscription-ended`, `instance-url-changed` |

## 2. Terms

| Term | Meaning |
|---|---|
| Offer | app or service in the catalogue of the installation (C8-M1) |
| Subscription | agreement of a customer company to use one offer; ID `offerSubscriptionId` |
| Customer company | company that subscribes (`customerCompanyId`) |
| Provider company | company that offers (`providerCompanyId`) |
| Single-instance app | app with one instance for all customers; the provider activates without a setup |
| Multi-instance app | app with one instance per subscription; the provider sets up the instance and reports its URL |
| Setup | creation of the client of the instance (apps) and of the technical user (if the offer has a technical user profile) |
| Provider configuration | addresses and client credentials with which the module calls the systems of the provider |

## 3. Not part of this module

| What | Where |
|---|---|
| Offers, agreements, technical user profiles, app roles | C8-M1 Offers |
| Client of an app instance in the Portal IAM | C2-M2 (steps `create-instance-client`, `enable-instance-client`, `disable-instance-client`, `update-instance-url`) |
| Technical users of a subscription, their secrets | C2-M3 (step `create-for-subscription`, call `credentials/rotate` with permission `rotate_subscription_credentials`) |
| Order of the setup steps, retries | C12-M1 Process Orchestration |
| Messages to people | C11-M1 Notifications |
| Connectors linked to a subscription | C6-M1 Connectors |
| Subscriptions across the boundary of an installation | not in the first version (OP-39) |
| Billing, usage | not part of the portal |

## 4. Technical frame

- Module of the modular monolith (Python, FastAPI), own database `c10_m1` in PostgreSQL.
- People call with their token from the Portal IAM (FRAMEWORK 4.2). The provider side is checked with the scope `company` against the provider company of the offer.
- Calls to the systems of the provider: HTTPS, OAuth 2.0 client credentials with the data of the provider configuration; timeout 10 seconds.
- Secrets of the provider configuration are stored encrypted (`ENCRYPTION_KEY`), never returned and never written to logs or events.
- Events through the outbox of the module (FRAMEWORK 1).

### Configuration

| Variable | Meaning | Example |
|---|---|---|
| `DATABASE_URL` | own database `c10_m1` | |
| `ENCRYPTION_KEY` | key for the client secrets of the provider configurations | |
| `OFFER_URL` | C8-M1 Offers: empty = built-in stand-in, `in-process`, or HTTP address (FRAMEWORK 4.7) | `in-process` |
| `COMPANY_URL` | C3-M1 Company Profile | `in-process` |
| `TECHNICAL_USER_URL` | C2-M3 Technical Users | `in-process` |
| `PROCESS_URL` | C12-M1 Process Orchestration (status of the setup, 6.3) | `in-process` |
| `PROVIDER_CALL_TIMEOUT_SECONDS` | timeout for calls to the provider | `10` |

## 5. Data

### Table `subscription`

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key; `offerSubscriptionId` |
| `offer_id`, `offer_type` | uuid, text | `offer_type` `APP` or `SERVICE`, copied at subscription time |
| `instance_kind` | text | `SINGLE` or `MULTI` for apps, empty for services |
| `customer_company_id`, `provider_company_id` | uuid | |
| `requested_by` | text | `sub` of the person who subscribed |
| `status` | text | 6.1 |
| `setup_started` | boolean | `true` after `start-setup` |
| `instance_url` | text | multi-instance apps: URL of the instance; single-instance apps: URL of the app |
| `decline_reason` | text | with `DECLINED` |
| `requested_at`, `activated_at`, `ended_at` | timestamptz | |

### Table `subscription_consent`

`subscription_id`, `agreement_id`, `consent` (`ACTIVE`, `INACTIVE`), `given_by` (`sub`), `given_at`. One row per agreement of the offer.

### Table `provider_configuration`

One row per provider company: `company_id` (primary key), `setup_url`, `callback_url`, `token_url`, `client_id`, `client_secret` (encrypted), `updated_at`, `updated_by` (`sub`).

### Table `audit_entry`

As FRAMEWORK 4.3: time, `sub`, `company_id`, operation, object ID. No secrets, no names.

**Not stored:** credentials of technical users (C2-M3 hands them over once, 8.2); names and e-mail addresses of people.

## 6. Rules

### 6.1 Status of a subscription

| From | Event | To | Event published |
|---|---|---|---|
| – | subscribe (7.2) | `PENDING` | `SubscriptionRequested` |
| `PENDING` | provider starts the setup (7.5) | `PENDING`, `setup_started` | `SubscriptionSetupStarted` |
| `PENDING` | provider activates (7.6) | `ACTIVE` | `SubscriptionActivated` |
| `PENDING` | provider declines (7.7) | `DECLINED` | `SubscriptionDeclined` |
| `PENDING`, `ACTIVE` | customer ends the subscription (7.4) | `INACTIVE` | `SubscriptionDeactivated` |
| `ACTIVE` | provider changes the instance URL (7.8) | `ACTIVE` | `SubscriptionInstanceUrlChanged` |

`DECLINED` and `INACTIVE` are final. A declined subscription stays stored.

### 6.2 Checks when subscribing

- The offer is `ACTIVE` (query at C8-M1).
- The customer company is `ACTIVE` in the installation (query at C3-M1). The BPN is not checked; the module does not depend on its format.
- Consent to every mandatory agreement of the offer.
- Apps: at most one subscription per customer company and app in `PENDING` or `ACTIVE`. Services: several allowed.
- A company cannot subscribe to its own offer.

### 6.3 Checks for the provider

- Start the setup: only multi-instance apps and services; status `PENDING`; not started yet; for apps an `instanceUrl` (HTTPS, without `#`).
- Activate: status `PENDING`; multi-instance apps and services only after the setup was started and the setup has created everything; single-instance apps directly (the instance URL comes from the offer).
- Setup status: query at C12-M1 `GET /internal/processes?subjectType=SUBSCRIPTION&subjectId=<offerSubscriptionId>`, newest process of the flow `subscription-setup`. Status `RUNNING`: activate answers 409 `setup-not-finished`. Status `FAILED`: activate answers 409 `setup-failed`. Status `WAITING` (the flow waits for `SubscriptionActivated`) or no process found: activation allowed. The status is also shown to the customer and the provider as `setupStatus` (7.3); a setup that fails after the activation (e.g. the callback, 8.2) is visible there as `FAILED`.
- Change the instance URL: only multi-instance apps in `ACTIVE`.

### 6.4 Permissions

| Permission | Scope | What for |
|---|---|---|
| `view_subscriptions` | `company` | view subscriptions of the own company, as customer or as provider |
| `manage_subscriptions` | `company` | subscribe and end subscriptions of the own company (customer side) |
| `manage_offers` | `company` | provider side: start the setup, activate, decline, change the instance URL, manage the provider configuration (the company must be the provider of the offer) |
| `execute_steps_c10`, `query_c10` | `osp` | steps and queries over HTTP (FRAMEWORK 4.2) |

The permissions are listed in C2-M5 6.2. `view_subscriptions` is in the portal roles `IT Admin` and `Company Admin`, `manage_subscriptions` and `manage_offers` in `Company Admin` (C2-M5 6.3). A subscription of another company is answered with 404 (FRAMEWORK 4.3).

### 6.5 Repeatability

- Every step finds the subscription by `offerSubscriptionId`. `notify-provider` calls the provider again on a repetition; the provider must accept a repeated call (same `offerSubscriptionId`).
- `send-activation-callback` creates a new secret on every call (C2-M3 9.2); the provider uses the last one it received.
- Operations of people are not idempotent; a repeated activation of an `ACTIVE` subscription answers 409.

## 7. Operations

Lists as in FRAMEWORK 4.1. Tags: `Company` (customer side), `Provider` (provider side).

### 7.1 `GET /api/c10-m1/subscriptions`

Permission `view_subscriptions`. Parameter `view` (`customer` or `provider`, default `customer`), filters `status`, `offerType`, `offerId`. `customer`: subscriptions of the own company; `provider`: subscriptions to offers of the own company. Response 200 `SubscriptionPage`.

### 7.2 `POST /api/c10-m1/subscriptions`

Permission `manage_subscriptions`. Input: `offerId`, `agreementConsents` (`agreementId`, `consent`).

1. Checks as in 6.2; offer unknown or not active: 409 `offer-not-active`; missing consent: 400 `consent-missing`; open app subscription: 409 `already-subscribed`; own offer: 409 `own-offer`.
2. Store the subscription with `PENDING` and the consents.
3. Event `SubscriptionRequested`. Response 201 `Subscription`.

### 7.3 `GET /api/c10-m1/subscriptions/{offerSubscriptionId}`

Permission `view_subscriptions`. Visible to the customer company and to the provider company. Both see `setupStatus` (status of the newest process of the flows `subscription-setup` or `single-instance-activation` at C12-M1, `null` without a process). The provider additionally sees `requestedBy` and the consents. Response 200 `Subscription`.

### 7.4 `POST /api/c10-m1/subscriptions/{offerSubscriptionId}/end`

Permission `manage_subscriptions`, customer company only. Status `PENDING` or `ACTIVE`, otherwise 409 `invalid-state`. Status `INACTIVE`, event `SubscriptionDeactivated`. Response 200.

### 7.5 `POST /api/c10-m1/subscriptions/{offerSubscriptionId}/start-setup`

Permission `manage_offers`, provider company only. Input: `instanceUrl` (apps). Checks as in 6.3. Set `setup_started` and the URL; event `SubscriptionSetupStarted`. Response 202.

### 7.6 `POST /api/c10-m1/subscriptions/{offerSubscriptionId}/activate`

Permission `manage_offers`, provider company only. Checks as in 6.3, including the setup status. Status `ACTIVE`, `activated_at`; event `SubscriptionActivated`. Response 200. The client of the instance is enabled afterwards by the flow `subscription-setup`; for a single-instance app the flow `single-instance-activation` creates the technical user, if the offer has a profile, and sends the callback.

### 7.7 `POST /api/c10-m1/subscriptions/{offerSubscriptionId}/decline`

Permission `manage_offers`, provider company only. Input optional `reason` (up to 1000 characters). Status `PENDING`, otherwise 409. Status `DECLINED`; event `SubscriptionDeclined`. Response 200.

### 7.8 `PUT /api/c10-m1/subscriptions/{offerSubscriptionId}/instance-url`

Permission `manage_offers`, provider company only. Input `instanceUrl`. Checks as in 6.3. Event `SubscriptionInstanceUrlChanged`. Response 200.

### 7.9 `GET`, `PUT`, `DELETE /api/c10-m1/provider-configuration`

Permission `manage_offers`, for the own company. `PUT` input: `setupUrl`, `callbackUrl`, `tokenUrl`, `clientId`, `clientSecret` (write-only; left out = keep the stored one). `GET` returns everything except the secret, plus `clientSecretSet`. Without a configuration: `GET` 404.

## 8. Steps for Process Orchestration

Input always with `processId`, `stepRunId` and `offerSubscriptionId` (FRAMEWORK 4.4).

### 8.1 `POST /api/c10-m1/steps/notify-provider`

- **Flow:** `subscription-requested`, after `SubscriptionRequested`.
- **Precondition:** none. If the subscription is no longer `PENDING` (the provider activated or declined it, or the customer ended it before the step ran): `DONE` with `notified` `false`, no call.
- **Behaviour:** if the provider company has a configuration with `setupUrl`: `POST {setupUrl}` with `offerSubscriptionId`, `offerId`, `customerCompanyId`, customer name and country (query at C3-M1), and for multi-instance apps a hint that the setup is expected. Without a configuration: nothing to do.
- **Result:** `DONE` with `notified` (`true` or `false`). Provider not reachable or 5xx: 502 `upstream-error`, `retryable: true`; 4xx: `provider-rejected`, `retryable: false`.

### 8.2 `POST /api/c10-m1/steps/send-activation-callback`

- **Flow:** `subscription-setup`, after the client of the instance is enabled; `single-instance-activation`, after `SubscriptionActivated` of a single-instance app.
- **Input:** in addition optional `technicalUsers` (result `accounts` of C2-M3 `create-for-subscription`), `appInstanceClientId` and `clientId` (`iamClientId` of the instance client, result of C2-M2 `create-instance-client`).
- **Precondition:** subscription `ACTIVE`.
- **Behaviour:** without a `callbackUrl` in the provider configuration: nothing to do. Otherwise, for the account with `subscriptionParty` `CUSTOMER`: call C2-M3 `POST /internal/technical-users/{id}/credentials/rotate` with the `offerSubscriptionId` (permission `rotate_subscription_credentials`, C2-M3 6.4; a new secret, decided 21 Oct 2026). Then `POST {callbackUrl}` with `offerSubscriptionId`, `instanceUrl`, `clientId` (multi-instance apps) and the credentials of the technical user. The secret is neither stored nor in the step result.
- **Result:** `DONE` with `called` (`true` or `false`). Errors as in 8.1.

## 9. Queries for other modules

### 9.1 `GET /api/c10-m1/internal/subscriptions`

Parameters `customerCompanyId`, `offerId`, `status` (each optional, at least one). Response 200: list of `{ "offerSubscriptionId", "offerId", "customerCompanyId", "status" }`. Used by C2-M2 for app roles (C2-M2 11).

### 9.2 `GET /api/c10-m1/internal/subscriptions/{offerSubscriptionId}`

Response 200 `SubscriptionInternal`: as 9.1 plus `offerType`, `instanceKind`, `providerCompanyId`, `instanceUrl`. Used by C6-M1 (linked connectors) and C11-M1 (recipients).

## 10. Events

| Event | When | Content |
|---|---|---|
| `SubscriptionRequested` | subscribed (7.2) | `offerSubscriptionId`, `offerId`, `offerType`, `customerCompanyId`, `providerCompanyId` |
| `SubscriptionSetupStarted` | setup started (7.5) | as `SubscriptionRequested`, plus `instanceUrl` (apps) |
| `SubscriptionActivated` | activated (7.6) | `offerSubscriptionId`, `offerId`, `offerType`, `instanceKind`, `customerCompanyId`, `providerCompanyId` |
| `SubscriptionDeclined` | declined (7.7) | `offerSubscriptionId`, `offerId`, `customerCompanyId`, `providerCompanyId` |
| `SubscriptionDeactivated` | ended (7.4) | `offerSubscriptionId`, `offerId`, `customerCompanyId`, `providerCompanyId` |
| `SubscriptionInstanceUrlChanged` | instance URL changed (7.8) | `offerSubscriptionId`, `offerId`, `instanceUrl` |

Every event also carries `eventId`, `eventType`, `occurredAt`. No secrets, no names.

## 11. Calls to other modules and external systems

### 11.1 Modules

| Neighbour (variable) | Call | Permission of the service account | Built-in stand-in |
|---|---|---|---|
| C8-M1 Offers (`OFFER_URL`) | `GET /api/c8-m1/internal/offers/{offerId}`: `status`, `offerType`, `instanceKind`, `instanceUrl`, `providerCompanyId`, `agreements` (with `mandatory`) | `query_c8` | reads `stub-offers.json` |
| C3-M1 Company Profile (`COMPANY_URL`) | `GET /api/c3-m1/internal/companies/{companyId}`: `status`, `name`, `address.countryCode` | `query_c3_m1` | every company `ACTIVE` |
| C2-M3 Technical Users (`TECHNICAL_USER_URL`) | `POST /api/c2-m3/internal/technical-users/{id}/credentials/rotate` with `offerSubscriptionId` | `rotate_subscription_credentials` | fixed test credentials |
| C12-M1 Process Orchestration (`PROCESS_URL`) | `GET /api/c12-m1/internal/processes?subjectType=SUBSCRIPTION&subjectId=`: `flowKey`, `status` | `query_c12` | empty list (no setup status, activation allowed) |

### 11.2 Systems of the provider

| Call | When | Content |
|---|---|---|
| `POST {setupUrl}` | step 8.1 | `offerSubscriptionId`, `offerId`, `customerCompanyId`, `customerName`, `customerCountry`, `setupExpected` |
| `POST {callbackUrl}` | step 8.2 | `offerSubscriptionId`, `instanceUrl`, `clientId`, `technicalUser` (`clientId`, `clientSecret`) |

Authentication with a token from `tokenUrl` (client credentials). For the hackathon, the test tools provide a provider stand-in that answers both calls with 200.

## 12. Errors

Problem Details, `type` = `urn:c10-m1:problem:<name>`.

| `type` | HTTP | When | `retryable` for steps |
|---|---|---|---|
| `validation` | 400 | invalid input | `false` |
| `consent-missing` | 400 | a mandatory agreement without consent | – |
| `unauthorized` | 401 | token missing or invalid | – |
| `forbidden` | 403 | permission missing; provider operation by the customer company of the subscription (any other company gets 404) | – |
| `not-found` | 404 | subscription unknown or of another company | `false` |
| `invalid-state` | 409 | status does not allow the operation or the step | `false` |
| `offer-not-active` | 409 | offer unknown or not `ACTIVE` | – |
| `already-subscribed` | 409 | open subscription of the company to the app | – |
| `own-offer` | 409 | subscription to an offer of the own company | – |
| `setup-not-finished` | 409 | activate while the setup process is still `RUNNING` | – |
| `setup-failed` | 409 | activate while the setup process is `FAILED` | – |
| `upstream-error` | 502 | provider or neighbour not reachable, 5xx | `true` |
| `provider-rejected` | 502 | provider answers 4xx | `false` |

## 13. Acceptance

1. Subscribe to an active multi-instance app with all consents: 201, `PENDING`, `SubscriptionRequested` once.
2. Without consent to a mandatory agreement: 400 `consent-missing`. A second subscription to the same app: 409 `already-subscribed`. To an own offer: 409 `own-offer`.
3. A service can be subscribed to twice by the same company.
4. Provider starts the setup with `instanceUrl`: 202, `SubscriptionSetupStarted` with the URL. Again: 409. For a single-instance app: 409.
5. Provider activates a multi-instance app before the setup: 409; after it: `ACTIVE`, `SubscriptionActivated`.
6. Provider activates a single-instance app directly: `ACTIVE`.
7. Another company calls a provider operation: 404 (FRAMEWORK 4.3).
8. Customer ends an `ACTIVE` subscription: `INACTIVE`, `SubscriptionDeactivated`; ending again: 409.
9. Step `notify-provider` with a configuration: one call to `setupUrl`; without one: `DONE` with `notified` `false`; provider answers 503: 502 with `retryable` `true`.
10. Step `send-activation-callback`: calls C2-M3 `credentials/rotate` and then `callbackUrl`; the secret appears neither in the step result, nor in the audit log, nor in an event.
11. `GET /internal/subscriptions?customerCompanyId=…&offerId=…&status=ACTIVE` returns exactly the active subscriptions.
12. `GET /provider-configuration` never returns `clientSecret`; `PUT` without `clientSecret` keeps the stored one.
13. Setup started, C12-M1 (stand-in with data) reports the `subscription-setup` process `FAILED`: activate answers 409 `setup-failed`, `GET /subscriptions/{id}` shows `setupStatus` `FAILED` to customer and provider; with the process `WAITING`: 200 `ACTIVE`.
14. Step `notify-provider` for a subscription that the provider has already activated: `DONE` with `notified` `false`, no call to `setupUrl`.

## 14. Test data

| Object | Value |
|---|---|
| Provider company (OSP A) | `11111111-1111-4111-8111-111111111111` |
| Customer company | `5b0c7a52-3f7e-4b1e-9d0a-1c2d3e4f5a6b` (Beispiel Maschinenbau GmbH) |
| Multi-instance app | `7c1d9e3a-2b4f-4a6c-8d0e-5f1a2b3c4d5e` |
| Single-instance app | `8d2e0f4b-3c5a-4b7d-9e1f-6a2b3c4d5e6f`, URL `https://app.example.org` |
| Service | `9e3f1a5c-4d6b-4c8e-8f2a-7b3c4d5e6f70` |
| Mandatory agreement | `a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d` |

`stub-offers.json` contains the three offers with status `ACTIVE`, the provider company above and the agreement.

## Changes from version 0.1

- Setup through Process Orchestration (flows `subscription-requested`, `subscription-setup`, `subscription-ended`, `instance-url-changed`) instead of the own process `OFFER_SUBSCRIPTION` with ten steps. Client and technical user are created by C2-M2 and C2-M3.
- Provider calls are two steps (`notify-provider`, `send-activation-callback`); the secret for the callback comes from C2-M3 `credentials/rotate`.
- `start-auto-setup` becomes `start-setup`, `tenant-url` becomes `instance-url`, `unsubscribe` becomes `end`.
- New events `SubscriptionSetupStarted`, `SubscriptionInstanceUrlChanged`; `SubscriptionTenantUrlChanged` dropped.
- Query `/internal/subscriptions` for C2-M2.
- Paths under `/api/c10-m1`, permissions `view_subscriptions`, `manage_subscriptions`, `manage_offers` with scope `company` (C2-M5 6.2) instead of the legacy roles.
- `subscriberCompanyId` is now `customerCompanyId`, as in C2-M2 and C2-M3.
- Wallet context of the provider (vendor-specific) dropped; wallet access of the technical user runs through C2-M3 and C4-M2 (FRAMEWORK 5.7).
- Mapping of the legacy endpoints moved to [open-points.md](open-points.md).
- Activation depends on the setup status at C12-M1 (6.3, errors `setup-not-finished`, `setup-failed`); `setupStatus` in `Subscription`; new neighbour C12-M1 (`PROCESS_URL`).
- `notify-provider` answers `DONE` with `notified` `false` instead of 409 when the subscription is no longer `PENDING`.
- `send-activation-callback` takes `clientId`; it also runs in the flow `single-instance-activation`. `SubscriptionActivated` carries `offerType` and `instanceKind`.
- The call to C2-M3 `credentials/rotate` names its permission `rotate_subscription_credentials`; section 11.1 names the permission for every call.
