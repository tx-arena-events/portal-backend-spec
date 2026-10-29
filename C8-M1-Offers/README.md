# C8-M1 Offers

**Version:** 0.2, as of 2026-10-29 (OSP 2.0)
**Capability:** C8 Offers (apps and services)
**Kind:** Hackathon
**Files:** this README (behaviour), [openapi.yaml](openapi.yaml) (interface), [open-points.md](open-points.md)

This README and the OpenAPI description are sufficient to build the module. The binding framework is [FRAMEWORK.md](../FRAMEWORK.md).

## 1. Purpose

Companies of the installation offer apps and services in a catalogue (the marketplace). OSP staff review an offer before it is released. The module keeps the offers with their descriptions, documents, planned app roles, technical user profile and agreements, and runs the release: draft, submit, approve or decline, deactivate.

One model for apps and services: `offerType` `APP` or `SERVICE` with shared fields and a few type-specific ones.

When an app is approved, the planned app roles go to the Portal IAM (C2-M2, flow `app-approval`). From then on, the Portal IAM holds the app roles; this module does not.

Callers:

| Who | What for |
|---|---|
| People of a company (UI) | browse the catalogue, mark favourites |
| People of a provider company (UI) | create, edit, submit, deactivate own offers; documents, roles, technical user profile, agreements |
| OSP staff (UI) | review offers, approve, decline |
| Process Orchestration (flows) | lookup `get-offer` for the app approval and the subscription setup |
| C10-M1 Subscriptions | query: status, type, instance, provider, agreements of an offer |

## 2. Terms

| Term | Meaning |
|---|---|
| Offer | app or service in the catalogue of the installation; ID `offerId` (for apps also the `appId` in C2-M2) |
| Provider company | company that offers; needs the company role `APP_PROVIDER` or `SERVICE_PROVIDER` (C3-M1) |
| Single-instance app | one instance for all customers, URL in the offer |
| Multi-instance app | one instance per subscription, URL from the provider at the setup (C10) |
| Planned app roles | app roles in the draft; after the approval they are in the Portal IAM |
| Technical user profile | name and roles of the technical user that a subscription creates (C2-M3) |
| Release agreements | agreements the provider accepts before submitting |
| Offer agreements | agreements of the offer that a customer accepts when subscribing (C10) |

## 3. Not part of this module

| What | Where |
|---|---|
| App roles of an active app, clients of the app | C2-M2 (steps `create-client-template`, `add-app-roles`, `update-role-descriptions`) |
| Subscriptions | C10-M1 Subscriptions |
| Technical users | C2-M3 |
| Order of the steps after the approval | C12-M1 Process Orchestration |
| Messages to provider and OSP staff | C11-M1 Notifications |
| Catalogue across installations | not in the first version (OP-39) |
| Texts of the release agreements | installation settings (OP-36) |

## 4. Technical frame

- Module of the modular monolith (Python, FastAPI), own database `c8_m1` in PostgreSQL. Documents are stored in the own database (up to `MAX_DOCUMENT_MB`).
- People call with their token from the Portal IAM (FRAMEWORK 4.2). The provider side is checked with the scope `company` against `providerCompanyId`.
- Events through the outbox of the module.

### Configuration

| Variable | Meaning | Example |
|---|---|---|
| `DATABASE_URL` | own database `c8_m1` | |
| `COMPANY_URL` | C3-M1: empty = built-in stand-in, `in-process`, or HTTP address (FRAMEWORK 4.7) | `in-process` |
| `APP_ACCESS_URL` | C2-M2 | `in-process` |
| `SUBSCRIPTION_URL` | C10-M1 | `in-process` |
| `SETTINGS_FILE` | installation settings with the release agreements and languages (OP-36) | `config/installation.json` |
| `MAX_DOCUMENT_MB` | largest document | `8` |
| `LANGUAGES` | languages of descriptions | `de,en` |

## 5. Data

### Table `offer`

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key; `offerId` |
| `offer_type` | text | `APP`, `SERVICE` |
| `status` | text | 6.1 |
| `name` | text | 3 to 100 characters |
| `provider_company_id` | uuid | company of the person who created the offer |
| `descriptions` | jsonb | per language: `lang`, `short` (≤ 255), `long` (≤ 4096) |
| `price_text`, `license_type` | text | `license_type` `COTS` or `FOSS` |
| `provider_url`, `contact_email` | text | contact of the provider company, not of a person |
| `instance_kind`, `instance_url` | text | apps: `SINGLE` or `MULTI`; `instance_url` for `SINGLE` |
| `service_types` | text[] | services: `DATASPACE_SERVICE`, `CONSULTANCY_SERVICE` |
| `use_case_ids`, `privacy_policies`, `tags` | text[] | apps |
| `decline_reason` | text | last reason of OSP staff |
| `created_at`, `submitted_at`, `released_at`, `changed_at` | timestamptz | |

### Table `planned_role`

Apps only: `offer_id`, `name` (unique per offer), `descriptions` (jsonb, `lang`, `text`). Can be changed only in `CREATED`. Kept after the approval for traceability, but not read any more.

### Table `technical_user_profile`

At most one per offer: `offer_id`, `name`, `roles` (names of technical roles, C2-M3).

### Table `offer_document`

`id`, `offer_id`, `document_type`, `name`, `media_type`, `content`, `status` (`PENDING`, `LOCKED`, `INACTIVE`), `created_at`. Document types: `CONTRACT`, `LEAD_IMAGE`, `IMAGE`, `TECHNICAL_INFORMATION`, `CONFORMITY_APPROVAL`, `ADDITIONAL_DETAILS`.

### Table `offer_agreement`

Agreements of the offer for customers: `id`, `offer_id`, `name`, `document_id` or `link`, `mandatory`.

### Table `release_consent`

`offer_id`, `agreement_key`, `consent` (`ACTIVE`, `INACTIVE`), `given_by` (`sub`), `given_at`.

### Table `favourite`

`sub`, `offer_id`.

### Table `audit_entry`

As FRAMEWORK 4.3.

**Not stored:** app roles of active apps (Portal IAM), names of people.

## 6. Rules

### 6.1 Status of an offer

| From | Operation | To | Event |
|---|---|---|---|
| – | create (7.3) | `CREATED` | – |
| `CREATED` | submit (7.6) | `IN_REVIEW` | `OfferSubmitted` |
| `IN_REVIEW` | approve (7.7) | `ACTIVE` | `OfferApproved` |
| `IN_REVIEW` | decline (7.8) | `CREATED` | `OfferDeclined` |
| `ACTIVE` | deactivate (7.9) | `INACTIVE` | `OfferDeactivated` |
| `CREATED` | delete (7.5) | – | – |

What can be changed: `CREATED` everything · `IN_REVIEW` nothing · `ACTIVE` descriptions, contact, provider URL, documents of type `IMAGE` and `ADDITIONAL_DETAILS`, add app roles, change role descriptions · `INACTIVE` nothing.

### 6.2 Checks

- **Create:** the company has the company role `APP_PROVIDER` (apps) or `SERVICE_PROVIDER` (services) in C3-M1; services need at least one service type.
- **Submit:** name and descriptions in at least one language; consent to every mandatory release agreement; apps: instance kind set (`SINGLE` with `instanceUrl`), at least one planned role, at least one privacy policy, a document of type `CONTRACT`. Pending documents become `LOCKED`.
- **Approve and decline:** by OSP staff; nobody approves an offer of the own company (`company_id` in the token equals `providerCompanyId`: 403 `own-offer`). Decline needs a reason.
- **Add roles to an active app:** names must not exist yet (query at C2-M2 `/internal/app-roles`); then event `OfferRolesAdded`. Roles of an active app cannot be removed or renamed (C2-M2 open points 25).

### 6.3 Permissions

| Permission | Scope | What for |
|---|---|---|
| `view_marketplace` | `company` | browse active offers, favourites |
| `view_offers` | `company` | read own offers as provider, including drafts and documents |
| `manage_offers` | `company` | create, edit, submit, deactivate, delete own offers |
| `approve_offers` | `osp` | read offers in review, approve, decline (portal role `OSP Admin`) |
| `query_c8` | `osp` | queries over HTTP (FRAMEWORK 4.2) |

The permissions are listed in C2-M5 6.2. `view_marketplace` is in the portal role `Member` and therefore in every portal role of a company; `view_offers` and `manage_offers` are in `Company Admin`, `approve_offers` in `OSP Admin` (C2-M5 6.3).

An offer of another company in `CREATED`, `IN_REVIEW` or `INACTIVE` is answered with 404 (FRAMEWORK 4.3), except for `approve_offers`.

### 6.4 Repeatability

Operations of people are not idempotent; approving an offer that is already `ACTIVE` answers 409. Favourites: `PUT` and `DELETE` are idempotent.

## 7. Operations

Lists as in FRAMEWORK 4.1. Three groups of paths, each with one permission, as in C3-M1:

| Paths | Who | Permission |
|---|---|---|
| `/marketplace/offers…`, `/reference-data` | people of all companies (tag `Company`) | `view_marketplace` |
| `/offers…`, `/release-agreements` | people of the provider company (tag `Provider`) | `view_offers` to read, `manage_offers` to change |
| `/osp/offers…` | OSP staff (tag `OSP`) | `approve_offers` |

### 7.1 Lists

- `GET /api/c8-m1/marketplace/offers`: offers in `ACTIVE`; filters `offerType`, `serviceType`, `useCaseId`, `search`, `favourite`.
- `GET /api/c8-m1/offers`: all offers of the own company; filters `offerType`, `status`.
- `GET /api/c8-m1/osp/offers`: offers for the review; filter `status`, default `IN_REVIEW`.

Response 200 `OfferPage` with `OfferSummary`.

### 7.2 One offer

`GET /marketplace/offers/{offerId}` (only `ACTIVE`, with `ownSubscriptionStatus` from C10), `GET /offers/{offerId}` (own offer, any status), `GET /osp/offers/{offerId}` (any status). Response 200 `Offer`. For an active app, `roles` comes from C2-M2 `/internal/app-roles`; for a draft, from `planned_role`.

### 7.3 `POST /api/c8-m1/offers`

Permission `manage_offers`. Input `OfferInput` with `offerType`. Checks 6.2. Response 201 `Offer` with `CREATED`.

### 7.4 `PUT /api/c8-m1/offers/{offerId}`

Permission `manage_offers`, own offer. Changes allowed by status (6.1); a field that may not change in this status: 409 `invalid-state`. Response 200.

### 7.5 `DELETE /api/c8-m1/offers/{offerId}`

Permission `manage_offers`, own offer in `CREATED`. Response 204.

### 7.6 `POST /api/c8-m1/offers/{offerId}/submit`

Permission `manage_offers`. Checks 6.2; missing items: 409 `submit-incomplete` with the list in `missing`. Event `OfferSubmitted`. Response 200.

### 7.7 `POST /api/c8-m1/osp/offers/{offerId}/approve`

Permission `approve_offers`. Status `IN_REVIEW`, otherwise 409. Status `ACTIVE`, `released_at`; event `OfferApproved`. Response 200. For an app, the flow `app-approval` then creates the template client with the planned roles and, for a single-instance app, the client of the instance.

### 7.8 `POST /api/c8-m1/osp/offers/{offerId}/decline`

Permission `approve_offers`. Input `reason` (1 to 1000 characters). Status `CREATED`, documents back to `PENDING`; event `OfferDeclined`. Response 200.

### 7.9 `POST /api/c8-m1/offers/{offerId}/deactivate`

Permission `manage_offers`, own offer in `ACTIVE`. Status `INACTIVE`; event `OfferDeactivated`. Response 200. Existing subscriptions stay (C10 open question 3).

### 7.10 Planned roles and roles of an active app

- `GET /marketplace/offers/{offerId}/roles`, `GET /offers/{offerId}/roles`: planned roles (draft) or roles from C2-M2 (active app).
- `PUT /offers/{offerId}/roles` (`manage_offers`, `CREATED`): replace the planned roles. Input list of `{ name, descriptions: [{ lang, text }] }`.
- `POST /offers/{offerId}/roles` (`manage_offers`, `ACTIVE` app): add roles; checks 6.2; event `OfferRolesAdded` with the roles. Response 202; the flow `app-roles-added` writes them to the Portal IAM.
- `PATCH /offers/{offerId}/roles` (`manage_offers`, `ACTIVE` app): change descriptions of existing roles; event `OfferRoleDescriptionsChanged`. Response 202.

### 7.11 Technical user profile

`GET`, `PUT`, `DELETE /offers/{offerId}/technical-user-profile` (`view_offers` / `manage_offers`, `CREATED`). Input `name`, `roles` (names of technical roles; checked against the assignable roles of C2-M3).

### 7.12 Documents

- `GET …/offers/{offerId}/documents` in all three groups: metadata; the marketplace shows only documents of active offers.
- `POST /offers/{offerId}/documents` (`manage_offers`): multipart `documentType`, `file`; allowed status as 6.1. A new `LEAD_IMAGE` or `CONTRACT` sets the previous one to `INACTIVE`.
- `GET …/offers/{offerId}/documents/{documentId}/content` in all three groups: download.
- `DELETE /offers/{offerId}/documents/{documentId}` (`manage_offers`): not `LOCKED`.

### 7.13 Agreements

- `GET /release-agreements?offerType=` (`manage_offers`): release agreements from the installation settings.
- `PUT /offers/{offerId}/release-consents` (`manage_offers`, `CREATED`): consents of the provider.
- `GET /marketplace/offers/{offerId}/agreements`, `GET` and `PUT /offers/{offerId}/agreements` (`PUT` with `manage_offers` in `CREATED`): offer agreements for customers.

### 7.14 Favourites

`PUT` and `DELETE /marketplace/offers/{offerId}/favourite` (`view_marketplace`, scope `company`). Favourites are stored per person (`sub`); a person sees and changes only their own.

### 7.15 `GET /api/c8-m1/reference-data`

Permission `view_marketplace`. Service types, license types, privacy policies, document types with the statuses in which they may be changed, languages.

## 8. Steps for Process Orchestration

None. The steps after the approval belong to C2-M2; this module reports events and answers the lookup (section 9).

## 9. Queries for other modules

### 9.1 `GET /api/c8-m1/internal/offers/{offerId}`

Response 200 `OfferInternal`:

| Field | Meaning |
|---|---|
| `offerId`, `status`, `offerType`, `name`, `providerCompanyId` | |
| `instanceKind`, `instanceUrl` | apps |
| `roles` | planned roles `{ name, descriptions: [{ lang, text }] }`, as C2-M2 `create-client-template` expects |
| `hasTechnicalUserProfile` | `true` if a profile exists |
| `technicalUserProfile` | `{ name, roles: [{ name }] }`, as C2-M3 `create-for-subscription` expects |
| `agreements` | `{ agreementId, name, mandatory }` |

Used by the flows `app-approval` and `subscription-setup` (lookup `offer`) and by C10-M1.

## 10. Events

| Event | When | Content |
|---|---|---|
| `OfferSubmitted` | submitted (7.6) | `offerId`, `offerType`, `providerCompanyId` |
| `OfferApproved` | approved (7.7) | `offerId`, `offerType`, `providerCompanyId` |
| `OfferDeclined` | declined (7.8) | `offerId`, `offerType`, `providerCompanyId`, `reason` |
| `OfferDeactivated` | deactivated (7.9) | `offerId`, `offerType`, `providerCompanyId` |
| `OfferRolesAdded` | roles added to an active app (7.10) | `offerId`, `roles` |
| `OfferRoleDescriptionsChanged` | descriptions changed (7.10) | `offerId`, `roles` |

Every event also carries `eventId`, `eventType`, `occurredAt`.

## 11. Calls to other modules and external systems

| Neighbour (variable) | Call | Built-in stand-in |
|---|---|---|
| C3-M1 (`COMPANY_URL`) | `GET /api/c3-m1/internal/companies/{companyId}`: `status`, `companyRoles` | every company `ACTIVE` with both provider roles |
| C2-M2 (`APP_ACCESS_URL`) | `GET /api/c2-m2/internal/app-roles?appId=`: roles of active apps | empty list |
| C2-M3 | assignable technical roles; C2-M3 offers them only to people, a query for modules is missing (open-points) | fixed list from `stub-technical-roles.json` |
| C10-M1 (`SUBSCRIPTION_URL`) | `GET /api/c10-m1/internal/subscriptions?customerCompanyId=&offerId=`: own subscription status in the offer view | none |

No external systems.

## 12. Errors

Problem Details, `type` = `urn:c8-m1:problem:<name>`.

| `type` | HTTP | When |
|---|---|---|
| `validation` | 400 | invalid input |
| `unauthorized` | 401 | token missing or invalid |
| `forbidden` | 403 | permission missing; company role `APP_PROVIDER` or `SERVICE_PROVIDER` missing |
| `own-offer` | 403 | OSP staff decide on an offer of their own company |
| `not-found` | 404 | offer unknown or not visible |
| `invalid-state` | 409 | status does not allow the operation or the change |
| `submit-incomplete` | 409 | checks of 6.2 not met; field `missing` |
| `role-exists` | 409 | role name already exists |
| `document-too-large` | 413 | document larger than `MAX_DOCUMENT_MB` |
| `upstream-error` | 502 | neighbour not reachable |

## 13. Acceptance

1. A company with `APP_PROVIDER` creates an app: 201, `CREATED`. Without the company role: 403.
2. Submit without contract or without planned role: 409 `submit-incomplete` with both items in `missing`. Complete: `IN_REVIEW`, `OfferSubmitted`.
3. OSP staff with `approve_offers` approve: `ACTIVE`, `OfferApproved` with `offerId`, `offerType`, `providerCompanyId`. Approve an offer of the own company: 403 `own-offer`.
4. `GET /internal/offers/{offerId}` after the approval returns the planned roles in the format of C2-M2 and `hasTechnicalUserProfile`.
5. Decline with reason: `CREATED`, documents `PENDING`, `OfferDeclined`. Decline without reason: 400.
6. Add a role to an active app: 202, `OfferRolesAdded`; the same name again: 409 `role-exists`. Change the name of a planned role after the approval: 409.
7. Another company reads a draft: 404. `GET /marketplace/offers` shows only `ACTIVE` offers.
8. Deactivate: `INACTIVE`, `OfferDeactivated`; the offer disappears from the marketplace.
9. Upload a second `LEAD_IMAGE`: the first becomes `INACTIVE`.

## 14. Test data

| Object | Value |
|---|---|
| Provider company (OSP A, roles `APP_PROVIDER`, `SERVICE_PROVIDER`) | `11111111-1111-4111-8111-111111111111` |
| Multi-instance app | `7c1d9e3a-2b4f-4a6c-8d0e-5f1a2b3c4d5e`, roles `Viewer`, `Editor`, profile `connector-sync` with role `Dataspace Admin` |
| Single-instance app | `8d2e0f4b-3c5a-4b7d-9e1f-6a2b3c4d5e6f`, URL `https://app.example.org` |
| Service | `9e3f1a5c-4d6b-4c8e-8f2a-7b3c4d5e6f70`, `DATASPACE_SERVICE` |

## Changes from version 0.1

- App roles of an active app are in the Portal IAM (C2-M2); this module keeps only the planned roles of the draft. `OfferRolesChanged` is replaced by `OfferRolesAdded` and `OfferRoleDescriptionsChanged`, which start flows of Process Orchestration.
- At most one technical user profile per offer, with name and roles, as C2-M3 `create-for-subscription` expects.
- Query `/internal/offers/{offerId}` for Process Orchestration and Subscriptions.
- Paths split into `/marketplace`, `/offers` and `/osp`, each with one permission (as C3-M1), instead of one list with a `view` parameter.
- Paths under `/api/c8-m1`; permissions `view_marketplace`, `view_offers`, `manage_offers`, `approve_offers` (C2-M5 6.2) instead of the legacy roles (`add_apps`, `approve_app_release`, …).
- OSP staff review offers; the operator role is dropped with OSP 2.0.
- Sales manager (a person) dropped; contact is the provider company.
- Events `OfferCreated`, `OfferUpdated`, `OfferDeleted` dropped (no consumer).
- View `assigned` dropped; the apps of a person come from C2-M2.
- Mapping of the legacy endpoints moved to [open-points.md](open-points.md).
