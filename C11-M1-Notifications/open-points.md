# C11-M1 Notifications: open points

**Version:** 0.2, as of 2026-10-29

## Provisional decisions

| No. | Topic | Decision in the README | To be agreed with |
|---|---|---|---|
| 1 | E-mail | none for the hackathon, only the inbox in the portal (decided on 29 Oct 2026); e-mail later as an add-on. Accounts have no e-mail address (C2-M5: the PID provides none) | project owner (OP-47) |
| 2 | Recipients among OSP staff | staff notifications are addressed to a permission (e.g. `review_application`); C2-M2 has no query for staff | C2-M2 |
| 3 | Rules fixed in the module | R1 to R17 (README 6.2); an OSP can switch rules off (`DISABLED_RULES`), not add new ones | project owner |
| 4 | Customers of an app (R13) | all `Company Admin` of companies with an active subscription; not the people who already have app roles | Offers, Subscriptions |
| 5 | Applicant | `applicantSub` from `ApplicationSubmitted` is kept until the application is finished | data protection |
| 6 | Retention | read or done notifications are deleted after 180 days (OP-20) | OSP, data protection |
| 7 | Recipients without `IT Admin` | R14 and R16 go to `Company Admin` if the company has no `IT Admin`; after onboarding a company has only its `Company Admin` (C2-M2 8.8). Alternative: onboarding also assigns `IT Admin` | C2-M2 |

## Assumptions about other modules

| Who | Assumption |
|---|---|
| C2-M5 | `view_notifications` in the portal role `Applicant`, which every account keeps; `retrigger_process_steps` in `OSP Admin` (C2-M5 6.3) |
| C2-M2 | `GET /internal/company-users?companyId=&portalRole=` returns `userId` of the people, which is their `sub` (C2-M2 2, 9.4) |
| C1-M2 | `ApplicationSubmitted` carries `applicantSub` (FRAMEWORK 6) |
| C3-M1 | `CompanyChangeRequestSubmitted`, `CompanyChangeRequestClosed` with `changeRequestId` (FRAMEWORK 6) |
| C6-M1 | `ConnectorRegistered` with `connectorId`, `companyId` |
| C8-M1, C10-M1, C12-M1 | events as in their section 10 |
| Token | contains the permissions of the person under `resource_access.portal-core.roles` (FRAMEWORK 4.2), used for staff notifications |

## Open questions

1. **E-mail as an add-on:** where the address comes from is OP-47 (proposal there: a contact address that the person gives voluntarily and confirms, stored in this module). The rules of README 6.2 would also decide who gets an e-mail. Today's portal sends about 25 kinds of mail.
2. **Query for OSP staff:** a query `GET /internal/osp-staff?permission=` in C2-M2 would allow personal notifications for staff too. Needed only if staff should hide or read tasks independently of each other with different contents.
3. **Four-eyes principle:** C1-M3 has no event for "waiting for the second approval" (status `AWAITING_SECOND_APPROVAL`). OP-19 asks for a notice to staff.
4. **Deactivation of a company:** after `disable-company-users` the people no longer belong to the company; a notice "company deactivated" could not find them. Notify before, or not at all?
5. **New versions of the terms** (`AgreementVersionActivated` with `reconsentRequired`): notify every `Company Admin` of every company? To be agreed with C1-M2.
6. **Live updates in the UI:** polling `GET /notifications/count` or Server-Sent Events.

## Legacy endpoints (portal backend 2.6.0)

| Legacy | New |
|---|---|
| `GET /api/notification` | `GET /notifications` |
| `GET /api/notification/count`, `GET /api/notification/count-details` | `GET /notifications/count` |
| `GET /api/notification/{notificationId}` | `GET /notifications/{notificationId}` |
| `PUT /api/notification/{notificationId}/read?isRead=` | `POST /notifications/{notificationId}/read`, `/unread` |
| `DELETE /api/notification/{notificationId}` | `POST /notifications/{notificationId}/hide` |
| `POST /api/notification/ssi-credentials`, `POST /api/administration/Mail/ssi-credentials` | dropped; rules react to events |
| `POST /api/administration/Mail/retrigger-send-mail` | dropped (no mail) |

## Out of scope

- Push notifications to phones.
- Marketing or bulk messages.
