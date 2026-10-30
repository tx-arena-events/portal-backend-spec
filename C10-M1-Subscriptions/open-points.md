# C10-M1 Subscriptions: open points

**Version:** 0.2, as of 2026-10-30

## Provisional decisions

| No. | Topic | Decision in the README | To be agreed with |
|---|---|---|---|
| 1 | Catalogue per installation | a company subscribes only to offers of its own installation (README 3) | project owner (OP-39) |
| 2 | Provider operations by people | start the setup, activate, decline through the UI with `manage_offers`. Today providers call the setup by machine with a technical user; FRAMEWORK 4.2 does not let technical users call modules (OP-50) | C2-M3, C2-M5 |
| 3 | Secret in the activation callback | a new secret through C2-M3 `credentials/rotate` on every callback (decided 21 Oct 2026, C2-M3 R5) | provider systems |
| 4 | Declined subscriptions | kept with `DECLINED` (today deleted) | project owner, data protection |
| 5 | Several subscriptions | apps: one open subscription per company; services: several (as today) | project owner |
| 6 | Visibility | the provider sees `requestedBy` and the consents; the customer does not see the provider configuration | project owner |
| 7 | BPN | not checked and not stored; only the company status (README 6.2) | – |
| 8 | Activation and setup | activation only when the setup process at C12-M1 is not `RUNNING` or `FAILED`; customer and provider see `setupStatus` (README 6.3, 7.3). Alternative: an own subscription status for a failed setup | C12-M1, project owner |
| 9 | Single-instance app with a technical user profile | the flow `single-instance-activation` creates the technical user after `SubscriptionActivated` and sends the callback (README 8.2). Alternative: C8-M1 forbids a technical user profile for single-instance apps | C8-M1, C12-M1, provider systems |

## Assumptions about other modules

| Who | Assumption |
|---|---|
| C8-M1 Offers | `GET /internal/offers/{offerId}` with `status`, `offerType`, `instanceKind`, `instanceUrl`, `providerCompanyId`, `agreements` |
| C3-M1 | `GET /internal/companies/{companyId}` with `status`, `name`, `address.countryCode` |
| C2-M2 | creates, enables, disables the client of the instance in the flows; queries `/internal/subscriptions` |
| C2-M3 | step `create-for-subscription` creates the technical user (result `accounts`); step `disable-subscription-technical-users` deactivates the accounts of an ended or declined subscription (C2-M3 8.6); `POST /internal/technical-users/{id}/credentials/rotate` with `offerSubscriptionId`, permission `rotate_subscription_credentials` |
| C2-M5 | `view_subscriptions` in `IT Admin` and `Company Admin`, `manage_subscriptions` and `manage_offers` in `Company Admin`; `execute_steps_c10_m1`, `query_c10_m1` for the service account of Process Orchestration; `rotate_subscription_credentials`, `query_c8_m1`, `query_c3_m1`, `query_c12_m1` for the service account of this module (C2-M5 6.2, 6.3) |
| C12-M1 | flows `subscription-requested`, `subscription-setup`, `single-instance-activation`, `subscription-declined`, `subscription-ended`, `instance-url-changed` in `flows.yaml`; `GET /internal/processes` for the setup status |
| C6-M1, C11-M1 | react to `SubscriptionDeactivated` and the other events |

## Open questions

1. **Provider systems without a person:** today a provider calls `start-autoSetup` by machine; see OP-50 in the cross-module [open-points.md](../open-points.md).
2. **Technical users at the end of a subscription:** closed. C2-M3 `disable-subscription-technical-users` (C2-M3 8.6) runs in the flows `subscription-ended` and `subscription-declined` (C12-M1 flows.yaml).
3. **Offer deactivated:** what happens to `PENDING` and `ACTIVE` subscriptions? Today nothing changes.
4. **Compatibility for existing providers:** webhook fields and the old path `start-autoSetup`. Alias during the migration?
5. **Subscriptions across installations** (OP-39): a company of OSP B subscribes to an app of a provider at OSP A.

## Legacy endpoints (portal backend 2.6.0)

| Legacy | New |
|---|---|
| `POST /api/apps/{appId}/subscribe`, `POST /api/services/{serviceId}/subscribe` | `POST /subscriptions` |
| `GET /api/apps/subscribed/…`, `GET /api/services/subscribed/subscription-status` | `GET /subscriptions?view=customer` |
| `GET /api/apps/provided/subscription-status`, `GET /api/services/provided/subscription-status` | `GET /subscriptions?view=provider` |
| `GET …/subscription/{id}/provider`, `…/subscriber`, `GET /api/services/subscription/{id}` | `GET /subscriptions/{id}` |
| `PUT /api/apps/subscription/{id}/activate`, `…/activate-single-instance`, `PUT /subscription/{id}/activate` (services) | `POST /subscriptions/{id}/activate` |
| `PUT …/subscription/{id}/decline` | `POST /subscriptions/{id}/decline` |
| `PUT /api/apps/{id}/unsubscribe`, `PUT /api/services/{id}/unsubscribe` | `POST /subscriptions/{id}/end` |
| `POST /api/apps/start-autoSetup`, `POST /api/services/start-autoSetup` | `POST /subscriptions/{id}/start-setup` |
| `POST /api/apps/autoSetup`, `POST /api/services/autoSetup` | dropped |
| `PUT /api/apps/AppChange/{appId}/subscription/{id}/tenantUrl` | `PUT /subscriptions/{id}/instance-url` |
| `GET/PUT/DELETE /api/administration/SubscriptionConfiguration/owncompany` | `GET/PUT/DELETE /provider-configuration` |
| `…/SubscriptionConfiguration/process/offer-subscription/{id}` and five `retrigger-*` | C12-M1 `GET /processes?subjectType=SUBSCRIPTION&subjectId=…`, retrigger |

## Out of scope

- Billing and usage.
- Self-descriptions of subscriptions (clearing house).
