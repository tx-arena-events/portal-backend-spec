# C8-M1 Offers: open points

**Version:** 0.2, as of 2026-10-29

## Provisional decisions

| No. | Topic | Decision in the README | To be agreed with |
|---|---|---|---|
| 1 | Catalogue per installation | only offers of companies of the own installation (README 3) | project owner (OP-39) |
| 2 | App roles after the approval | only in the Portal IAM; the planned roles are kept but not read (README 5) | C2-M2 |
| 3 | One technical user profile per offer | name and roles, as C2-M3 `create-for-subscription` takes them; today several profiles are possible | C2-M3, providers |
| 4 | Review by OSP staff | `approve_offers` (portal role `OSP Admin`, as in C2-M5 6.3); nobody decides on an offer of the own company | project owner |
| 5 | Release agreements | from the installation settings (OP-36), not from a catalogue of this module | project owner |
| 6 | Documents in the own database | up to 8 MB per document | operators |
| 7 | Permission for the catalogue | `view_marketplace` (scope `company`) in the portal role `Member` and therefore in every portal role of a company (C2-M5 6.3); `view_offers` means the provider view. Favourites also use `view_marketplace` with scope `company`, stored per person | – |

## Assumptions about other modules

| Who | Assumption |
|---|---|
| C2-M5 | `view_marketplace` in `Member`; `view_offers`, `manage_offers` in `Company Admin`; `approve_offers` in `OSP Admin`; `query_c8` for the service account of Process Orchestration (C2-M5 6.2, 6.3) |
| C2-M2 | `create-client-template` takes the planned roles as `{ name, descriptions: [{ lang, text }] }`; `/internal/app-roles?appId=` returns the roles of an active app; `appId` equals `offerId` |
| C2-M3 | a query for the assignable technical roles for modules (today only for people) |
| C3-M1 | `GET /internal/companies/{companyId}` returns `companyRoles` with `APP_PROVIDER`, `SERVICE_PROVIDER` |
| C12-M1 | flows `app-approval`, `app-roles-added`, `app-role-descriptions-changed`; lookup `offer` uses section 9 |
| C10-M1 | reads status, instance and agreements through section 9 |
| C11-M1 | informs OSP staff on `OfferSubmitted` and the provider on `OfferApproved`, `OfferDeclined` |

## Open questions

1. **Marketplace across installations** (OP-39): should a company of OSP B see and subscribe to apps of OSP A? Then the catalogue would need a central part at the CSP.
2. **Offer deactivated:** what happens to the subscriptions (C10 open question 3) and to the clients in the Portal IAM?
3. **Change the URL of a single-instance app** after the approval: C2-M2 has the step `update-instance-url`, this module has no event for it yet.
4. **Use cases and language tags:** reference data (OP-13) or own tables?

## Legacy endpoints (portal backend 2.6.0)

The apps service (40 endpoints) and the services service (23 endpoints) become one model. Paths below are shortened: `…/AppReleaseProcess` = `/api/apps/AppReleaseProcess`, `…/ServiceRelease` = `/api/services/ServiceRelease`.

| Legacy | New |
|---|---|
| `POST …/AppReleaseProcess/createapp`, `POST …/ServiceRelease/addservice` | `POST /offers` |
| `PUT …/AppReleaseProcess/{appId}`, `PUT …/ServiceRelease/{serviceId}`, `PUT /api/apps/AppChange/{appId}/appupdate/description`, `POST …/AppReleaseProcess/instance-type/{appId}` | `PUT /offers/{offerId}` |
| `DELETE …/AppReleaseProcess/{appId}` | `DELETE /offers/{offerId}` |
| `GET /api/apps/{appId}`, `GET /api/services/{serviceId}`, `…/appStatus`, `…/serviceStatus`, `…/inReview/{id}`, `GET …/AppChange/{appId}/appupdate/description` | `GET /offers/{offerId}` |
| `GET /api/apps/active`, `GET /api/services/active`, `GET /api/apps/favourites` | `GET /marketplace/offers` |
| `GET /api/apps/provided`, `GET /api/services/provided` | `GET /offers` |
| `GET …/AppReleaseProcess/inReview`, `GET …/ServiceRelease/inReview` | `GET /osp/offers` |
| `PUT …/{id}/submit` (apps, services) | `POST /offers/{offerId}/submit` |
| `PUT …/AppReleaseProcess/{appId}/approveApp`, `PUT …/ServiceRelease/{serviceId}/approveService` | `POST /osp/offers/{offerId}/approve` |
| `PUT …/AppReleaseProcess/{appId}/declineApp`, `PUT …/ServiceRelease/{serviceId}/declineService` | `POST /osp/offers/{offerId}/decline` |
| `PUT /api/apps/AppChange/{appId}/deactivateApp`, `PUT /api/services/ServiceChange/{serviceId}/deactivateService` | `POST /offers/{offerId}/deactivate` |
| document endpoints of `AppReleaseProcess`, `AppChange`, `ServiceRelease` (upload, list, download, delete, lead image) | `/offers/{offerId}/documents…` |
| `GET`, `POST …/AppReleaseProcess/{appId}/role(s)`, `GET …/AppChange/{appId}/roles`, `POST …/AppChange/{appId}/role/activeapp`, `DELETE …/role/{roleId}` | `GET`, `PUT`, `POST`, `PATCH /offers/{offerId}/roles` |
| `GET`, `PUT …/technical-user-profiles` (apps, services) | `/offers/{offerId}/technical-user-profile` |
| `GET …/agreementData`, `GET …/consent/{id}`, `POST …/consent/{id}/agreementConsents` | `GET /release-agreements`, `PUT /offers/{offerId}/release-consents` |
| `GET /api/apps/appAgreementData/{appId}`, `GET /api/services/serviceAgreementData/{serviceId}` | `GET /offers/{offerId}/agreements` |
| `GET …/AppReleaseProcess/privacyPolicies`, `GET …/ServiceRelease/serviceTypes`, `GET /api/administration/staticdata/licenseType` | `GET /reference-data` |
| `POST`, `DELETE /api/apps/{appId}/favourite` | `PUT`, `DELETE /marketplace/offers/{offerId}/favourite` |
| `GET /api/apps/business` (apps with a role of the person) | dropped; C2-M2 knows the app roles |
| `GET …/AppReleaseProcess/ownCompany/salesManager` | dropped (no sales manager) |
| `GET /api/services/serviceConsent/{serviceConsentId}` | C10-M1 `GET /subscriptions/{id}` |

## Out of scope

- Self-descriptions of offers (clearing house).
- Prices and billing beyond a price text.
