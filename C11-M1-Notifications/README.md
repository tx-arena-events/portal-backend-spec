# C11-M1 Notifications

**Version:** 0.2, as of 2026-10-30 (OSP 2.0)
**Capability:** C11 Notifications
**Kind:** Hackathon
**Files:** this README (behaviour), [openapi.yaml](openapi.yaml) (interface), [open-points.md](open-points.md)

This README and the OpenAPI description are sufficient to build the module. The binding framework is [FRAMEWORK.md](../FRAMEWORK.md).

## 1. Purpose

The module informs people in the portal of the OSP. It listens to the events of the other modules, decides with a fixed set of rules (section 6.2) who is informed, and keeps an inbox per person: information, and tasks that are closed automatically when the matching event arrives.

Modules do not know this module and do not send it anything; they only publish their events (FRAMEWORK 1).

There are no e-mails in this version, only the inbox in the portal (decided on 29 Oct 2026). Accounts have no e-mail address, because the PID does not provide one (C2-M5 5.4, OP-47). E-mail may follow later as an add-on (open-points No. 1).

Callers:

| Who | What for |
|---|---|
| People of a company and applicants (UI) | read their inbox, mark as read, hide |
| OSP staff (UI) | the same, including tasks for OSP staff |
| Event bus of the installation | delivers the events of section 11.1 |

## 2. Terms

| Term | Meaning |
|---|---|
| Notification | entry in the inbox; `type` and `content` (IDs and business values), the UI renders the text |
| Topic | `INFO` (information) or `ACTION` (task) |
| Task | notification with topic `ACTION`; open until the rule's closing event arrives (`done`) |
| Rule | event → type, recipients, closing events (6.2) |
| Personal notification | addressed to one person (`sub`) |
| Staff notification | addressed to all OSP staff with one permission, e.g. `review_application`; every person with the permission sees it |
| Applicant | person without a company who submitted an application (`applicantSub` of `ApplicationSubmitted`) |

## 3. Not part of this module

| What | Where |
|---|---|
| Deciding when something happens | the module that publishes the event |
| Texts in the languages of the UI | UI; this module stores only `type` and `content` |
| Names and contact data of people | nowhere in this module (FRAMEWORK 4.6); the UI resolves names through its own queries |
| Invitation links | C2-M2 creates them, the UI passes them on (C2-M2 3) |
| E-mail | not in this version; later as an add-on (OP-47) |
| Failed steps and retriggers | C12-M1; this module only informs OSP staff |

## 4. Technical frame

- Module of the modular monolith (Python, FastAPI), own database `c11_m1` in PostgreSQL.
- People call with their token from the Portal IAM. The inbox of a person: personal notifications with their `sub`, plus staff notifications whose permission is in the token.
- Events are received at least once; an `eventId` is processed once.
- A recipient query that fails (C2-M2 not reachable) is repeated with growing wait times (5 s, doubling, at most 10 minutes, at most 10 attempts); after that the event is logged as not delivered.

### Configuration

| Variable | Meaning | Example |
|---|---|---|
| `DATABASE_URL` | own database `c11_m1` | |
| `APP_ACCESS_URL` | C2-M2: empty = built-in stand-in, `in-process`, or HTTP address (FRAMEWORK 4.7) | `in-process` |
| `SUBSCRIPTION_URL` | C10-M1 | `in-process` |
| `RETENTION_DAYS` | notifications that are read or done are deleted after this many days | `180` |
| `DISABLED_RULES` | rules the OSP switches off, e.g. `R14,R16` | |

## 5. Data

### Table `notification`

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key |
| `rule` | text | `R1` … (6.2) |
| `type`, `topic` | text | from the rule |
| `receiver_sub` | text | personal notification; empty for staff notifications |
| `receiver_permission` | text | staff notification; empty for personal ones |
| `company_id` | uuid | company the notification is about, if any |
| `subject_type`, `subject_id` | text, uuid | object of the notification, e.g. `OFFER` and the `offerId` |
| `content` | jsonb | IDs and business values from the event; never names, never secrets |
| `done` | boolean | tasks only |
| `dedup_key` | text | unique: rule, `eventId`, receiver |
| `created_at`, `done_at` | timestamptz | |

### Table `notification_state`

Per person and notification: `notification_id`, `sub`, `read_at`, `hidden_at`. A staff notification has one row per person who has read or hidden it.

### Table `applicant`

`application_id` (primary key), `applicant_sub`, `created_at`. Stored on `ApplicationSubmitted`, deleted after `ApplicationCompleted`, `ApplicationDeclined` or `ApplicationWithdrawn` has been processed. Needed because the later events of the application do not carry the applicant.

### Table `processed_event`

`event_id` (primary key), `event_type`, `received_at`, `status` (`DONE`, `FAILED`).

**Not stored:** names, e-mail addresses, texts.

## 6. Rules

### 6.1 Recipients

| Recipient in 6.2 | Resolved how |
|---|---|
| applicant | `applicant_sub` of the application (table `applicant`) |
| `<portal role>` of `<company field>` | C2-M2 `GET /internal/company-users?companyId=…&portalRole=…`: one personal notification per person, addressed to `userId` (C2-M2 `CompanyUserRef`; `userId` is the `sub` of the person, C2-M2 2) |
| staff with `<permission>` | one staff notification; every person with the permission in the token sees it |
| customers of the app | C10-M1 `GET /internal/subscriptions?offerId=…&status=ACTIVE` with `offerId` = `appId` of the event (an app is an offer; C12-M1 flow `app-approval` passes `offerId` as `appId`), then `Company Admin` of each `customerCompanyId` |

### 6.2 Rules

| No. | Event | Type | Topic | Recipients | Closed by (same object) |
|---|---|---|---|---|---|
| R1 | `ApplicationSubmitted` | `APPLICATION_SUBMITTED` | `INFO` | applicant | – |
| R2 | `ApplicationSubmitted` | `REVIEW_CASE_OPENED` | `ACTION` | staff with `review_application` | `ApplicationApproved`, `ApplicationDeclined`, `ApplicationWithdrawn` (`applicationId`) |
| R3 | `ApplicationDeclined` | `APPLICATION_DECLINED` | `INFO` | applicant | – |
| R4 | `ApplicationCompleted` | `APPLICATION_COMPLETED` | `INFO` | applicant | – |
| R5 | `CompanyChangeRequestSubmitted` | `CHANGE_REQUEST_OPENED` | `ACTION` | staff with `review_company_changes` | `CompanyChangeRequestClosed` (`changeRequestId`) |
| R6 | `OfferSubmitted` | `OFFER_REVIEW_REQUESTED` | `ACTION` | staff with `approve_offers` | `OfferApproved`, `OfferDeclined` (`offerId`) |
| R7 | `OfferApproved` | `OFFER_APPROVED` | `INFO` | `Company Admin` of `providerCompanyId` | – |
| R8 | `OfferDeclined` | `OFFER_DECLINED` | `INFO` | `Company Admin` of `providerCompanyId` | – |
| R9 | `SubscriptionRequested` | `SUBSCRIPTION_REQUESTED` | `ACTION` | `Company Admin` of `providerCompanyId` | `SubscriptionActivated`, `SubscriptionDeclined`, `SubscriptionDeactivated` (`offerSubscriptionId`) |
| R10 | `SubscriptionActivated` | `SUBSCRIPTION_ACTIVATED` | `INFO` | `Company Admin` of `customerCompanyId` | – |
| R11 | `SubscriptionDeclined` | `SUBSCRIPTION_DECLINED` | `INFO` | `Company Admin` of `customerCompanyId` | – |
| R12 | `SubscriptionInstanceUrlChanged` | `SUBSCRIPTION_URL_CHANGED` | `INFO` | `Company Admin` of the customer (C10-M1 `/internal/subscriptions/{id}`) | – |
| R13 | `AppRolesAdded` | `APP_ROLES_ADDED` | `INFO` | customers of the app (`appId`) | – |
| R14 | `WalletAccountCreated`, `WalletAccountFailed` | `TECHNICAL_USER_WALLET_READY`, `TECHNICAL_USER_WALLET_FAILED` | `INFO` | `IT Admin` of `companyId`; if the company has none, `Company Admin` | – |
| R15 | `CredentialRevoked` | `CREDENTIAL_REVOKED` | `INFO` | `Company Admin` of `companyId` | – |
| R16 | `ConnectorRegistered` | `CONNECTOR_REGISTERED` | `INFO` | `IT Admin` of `companyId`; if the company has none, `Company Admin` | – |
| R17 | `ProcessStepFailed` | `PROCESS_STEP_FAILED` | `ACTION` | staff with `retrigger_process_steps` | `ProcessStepRetriggered` (same `processId` and `stepKey`), `ProcessCompleted`, `ProcessEnded` (`processId`) |

`content` of a notification: the fields of the event without `eventId`, `eventType` and `occurredAt`. R17 additionally `flowKey`, `subjectType`, `subjectId`, `errorType`.

A closing event sets `done` on all open tasks of the rule for the same object. A closing event that arrives before its task (e.g. a quick approval) is remembered for 24 hours. A new task is created as `done` only if the event that creates it did not occur later than the remembered closing event (compare `occurredAt`). Example: after `ProcessStepRetriggered` the same step fails again; the new `ProcessStepFailed` is later than the retrigger, so the new task stays open (acceptance case 6).

### 6.3 Permissions

| Permission | Scope | What for |
|---|---|---|
| `view_notifications` | `own` | read, mark and hide the own notifications |

`view_notifications` is in the portal role `Applicant`, which every account keeps; so every person has it, with or without a company, including OSP staff (C2-M5 6.2, 6.3). Scope `own` as in FRAMEWORK 4.3: the own inbox is the personal notifications with the person's `sub` and the staff notifications whose permission is in the token (section 4).

There are no other operations. Nobody can create notifications through the API.

### 6.4 Repeatability

- An `eventId` is processed once (`processed_event`); `dedup_key` prevents double notifications if processing is repeated after a crash.
- Mark as read, mark as unread, hide: idempotent.

## 7. Operations

All operations on the own inbox (tag `Company`, FRAMEWORK 4.1), permission `view_notifications`, scope `own`. A notification that is not in the own inbox: 404.

### 7.1 `GET /api/c11-m1/notifications`

Filters `topic`, `type`, `read` (`true`, `false`), `done` (`true`, `false`, tasks only), `createdSince`; sorting `createdAt` descending (default) or ascending. Hidden notifications are left out. Paging as in FRAMEWORK 4.1. Response 200 `NotificationPage`.

### 7.2 `GET /api/c11-m1/notifications/count`

Response 200 `{ total, unread, openTasks, unreadInfo }` for the own inbox.

### 7.3 `GET /api/c11-m1/notifications/{notificationId}`

Response 200 `Notification` with `read` for the calling person.

### 7.4 `POST /api/c11-m1/notifications/{notificationId}/read` and `/unread`

Sets or removes `read_at` for the calling person. Response 204.

### 7.5 `POST /api/c11-m1/notifications/{notificationId}/hide`

Hides the notification for the calling person. An open task: 409 `task-open`. Response 204. Hidden notifications are deleted with the retention period (`RETENTION_DAYS`).

## 8. Steps for Process Orchestration

None. The module reacts to events only.

## 9. Queries for other modules

None.

## 10. Events

None. The module only consumes events (11.1).

## 11. Calls to other modules and external systems

### 11.1 Consumed events

All events named in 6.2: from C1-M2 (`ApplicationSubmitted`, `ApplicationCompleted`, `ApplicationWithdrawn`), C1-M3 (`ApplicationApproved`, `ApplicationDeclined`), C3-M1 (`CompanyChangeRequestSubmitted`, `CompanyChangeRequestClosed`), C2-M2 (`AppRolesAdded`), C4-M1 (`CredentialRevoked`), C4-M2 (`WalletAccountCreated`, `WalletAccountFailed`), C6-M1 (`ConnectorRegistered`), C8-M1 (`OfferSubmitted`, `OfferApproved`, `OfferDeclined`), C10-M1 (`SubscriptionRequested`, `SubscriptionActivated`, `SubscriptionDeclined`, `SubscriptionDeactivated`, `SubscriptionInstanceUrlChanged`), C12-M1 (`ProcessStepFailed`, `ProcessStepRetriggered`, `ProcessCompleted`, `ProcessEnded`).

### 11.2 Queries

| Neighbour (variable) | Call | Built-in stand-in |
|---|---|---|
| C2-M2 (`APP_ACCESS_URL`) | `GET /api/c2-m2/internal/company-users?companyId=&portalRole=` → `userId` (= `sub`) of the people | reads `stub-company-users.json` |
| C10-M1 (`SUBSCRIPTION_URL`) | `GET /api/c10-m1/internal/subscriptions?offerId=&status=ACTIVE`, `GET /api/c10-m1/internal/subscriptions/{id}` | empty list |

## 12. Errors

Problem Details, `type` = `urn:c11-m1:problem:<name>`.

| `type` | HTTP | When |
|---|---|---|
| `validation` | 400 | invalid filter |
| `unauthorized` | 401 | token missing or invalid |
| `forbidden` | 403 | permission missing |
| `not-found` | 404 | notification not in the own inbox |
| `task-open` | 409 | hide an open task |

## 13. Acceptance

1. `ApplicationSubmitted`: the applicant has `APPLICATION_SUBMITTED`; a person with `review_application` sees `REVIEW_CASE_OPENED` as open task; a person without it does not.
2. `ApplicationApproved` for the same `applicationId`: the task is `done` for everyone.
3. The same event a second time (same `eventId`): no second notification.
4. `ApplicationDeclined`: the applicant has `APPLICATION_DECLINED` with `reason`; afterwards the row in `applicant` is deleted.
5. `SubscriptionRequested`: every `Company Admin` of the provider company (stand-in: Otto) has an open task; `SubscriptionActivated` closes it and creates `SUBSCRIPTION_ACTIVATED` for the `Company Admin` of the customer.
6. `ProcessStepFailed`: a person with `retrigger_process_steps` sees `PROCESS_STEP_FAILED`; `ProcessStepRetriggered` for the same step closes it; a new `ProcessStepFailed` creates a new task.
7. A closing event arrives before the event of the task, and the task event has an earlier or equal `occurredAt`: the task is created as `done`. A task event with a later `occurredAt` (case 6 after a retrigger): the task is created open.
8. Hide an open task: 409; after `done`: 204, it no longer appears in the list.
9. One person marks a staff notification as read: for another staff member it stays unread (Rolf and Otto, both with `review_company_changes`, rule R5).
10. Another person's notification: 404. Notification contents contain no names (check against the stand-in data).
11. `DISABLED_RULES=R14`: `WalletAccountCreated` creates no notification.
12. `ConnectorRegistered` for a company without an `IT Admin` (stand-in): every `Company Admin` of the company has `CONNECTOR_REGISTERED`.

## 14. Test data

| Object | Value |
|---|---|
| Applicant | Erika Mustermann, `sub` `5a1c0001-0000-4000-8000-000000000001`, application `0f8e2d4c-6b1a-4c3e-8f7d-2a9b1c0d3e4f` |
| Provider company (company of OSP A) | `11111111-1111-4111-8111-111111111111`, `Company Admin` Otto Normalverbraucher `5a1c0012-0000-4000-8000-000000000012` (he invites himself, C2-M2 7.20) |
| Customer company | Beispiel Maschinenbau GmbH `c0000001-0000-4000-8000-000000000001`, `Company Admin` Erika `5a1c0001-0000-4000-8000-000000000001`, no `IT Admin` (case 12) |
| OSP staff | Rolf Prüfer `bbede0f3-fbc4-433b-b7ca-1982e8724912` (`OSP Reviewer`: `review_application`, `review_company_changes`); Otto (`OSP Admin`: `review_company_changes`, `approve_offers`, `retrigger_process_steps`) |
| Person without a company | Lieschen Müller `5a1c0004-0000-4000-8000-000000000004`, only `Applicant` |

IDs, persons and portal roles as in FRAMEWORK 7a and C2-M5 6.3.

`stub-company-users.json` maps `companyId` and portal role to these `sub` values.

## Changes from version 0.1

- No e-mail: accounts have no e-mail address (C2-M5, OP-47). Mail delivery, the process `MAILING`, the template catalogue and the events `MailRequested`, `MailSent`, `MailDeliveryFailed` are dropped (open-points No. 1).
- Modules no longer publish `NotificationRequested` or `NotificationsResolved`. The module derives notifications from the domain events with fixed rules (6.2) and closes tasks by event.
- Type catalogue replaced by the rules R1 to R17 for OSP 2.0 (OP-19 and failed steps of C12).
- Staff notifications addressed to a permission, because there is no query for OSP staff (open-points No. 2).
- Receiver is the `sub` of the person, no portal user ID.
- Paths under `/api/c11-m1`; permission `view_notifications` with scope `own`. `retrigger_mail` dropped.
- Deleting becomes hiding per person.
- A remembered closing event closes a new task only if the task event is not later (`occurredAt`, 6.2).
- R14 and R16 go to `Company Admin` if the company has no `IT Admin` (e.g. right after onboarding).
- Recipients from C2-M2 are read from `userId` (= `sub`); `appId` of `AppRolesAdded` is the `offerId` for the query at C10-M1.
- Tag `Company` instead of `Inbox` (FRAMEWORK 4.1) (30 Oct 2026).
- Test data with the common test IDs and test persons of FRAMEWORK 7a; `sub` values are account IDs, not user names (renamed from `ebw.test-erika`, `ebw.test-otto`, `ebw.test-lieschen`, `ebw.test-staff`) (30 Oct 2026).
