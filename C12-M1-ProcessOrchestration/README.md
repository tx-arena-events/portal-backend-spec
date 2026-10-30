# C12-M1 Process Orchestration

**Version:** 0.2, as of 2026-10-30 (OSP 2.0)
**Capability:** C12 Process Orchestration
**Kind:** Provided
**Files:** this README (behaviour), [openapi.yaml](openapi.yaml) (interface), [flows.yaml](flows.yaml) (flow definitions), [open-points.md](open-points.md)

This README, the OpenAPI description and the flow definitions are sufficient to build the module. The binding framework is [FRAMEWORK.md](../FRAMEWORK.md); this module implements the flows of FRAMEWORK section 5.

## 1. Purpose

Process Orchestration runs the flows of an installation. A flow is data: a trigger event, a list of steps of other modules, and the events that complete them. The module

- starts a process when a trigger event arrives (for example `ApplicationSubmitted`);
- calls the steps of the modules in the order of the flow, with the input the flow defines;
- waits for the event of a long-running step (`RUNNING`), or for an event that the flow waits for (for example `ApplicationApproved`);
- repeats steps after temporary errors and notifies OSP staff when a step fails for good (OP-18);
- lets OSP staff view processes and retrigger a failed step;
- sends changes of a process to the UI as Server-Sent Events, so that the UI does not poll (7.6).

Modules do not know Process Orchestration (FRAMEWORK 1). They offer steps under `/steps/...`, publish events, and name their preconditions.

Callers:

| Who | What for |
|---|---|
| Event bus of the installation | delivers every published event (in the same process) |
| OSP staff (through the UI) | view processes and steps, retrigger a failed step, follow a process live (7.7) |
| Applicants (through the UI) | follow the processes of their own application live (7.8) |
| Other modules | query the processes of a subject (section 9), e.g. Subscriptions for the setup status |
| Notifications | react to `ProcessStepFailed` and inform OSP staff |

## 2. Terms

| Term | Meaning |
|---|---|
| Flow | definition in [flows.yaml](flows.yaml): trigger, subject, steps; identified by `key` |
| Process | one run of a flow for one subject, e.g. the onboarding of one application |
| Subject | the object a process is about: `APPLICATION`, `COMPANY`, `TECHNICAL_USER`, `OFFER`, `SUBSCRIPTION`, `CONNECTOR`, with its ID |
| Step run | one step of one process; its ID is the `stepRunId` that the module receives |
| Correlation key | field of an event (e.g. `companyId`) that assigns the event to an open process |
| Process variables | fields of the trigger event and of correlated events, plus values set by the flow; never secrets |
| Lookup | query to another module that a flow declares, e.g. the core data of a company for the CSP |

## 3. Not part of this module

| What | Where |
|---|---|
| Business logic and business checks of a step | the module that owns the step |
| Decisions (approve, decline, activate) | the owning module, e.g. C1-M3, Subscriptions |
| Status shown to companies (e.g. "application under review") | the owning module (C1-M2, Subscriptions) |
| Messages to people | Notifications, triggered by events |
| Transport of events | event bus of the installation with an outbox per module (FRAMEWORK 1) |
| Flows across installations | none: all flows run within one installation; only C3-M2 talks to the CSP (FRAMEWORK 5) |

## 4. Technical frame

- Module of the modular monolith (Python, FastAPI), own database `c12_m1` in PostgreSQL.
- **Calling steps:** in the same process through the public interface of the module; if a module is split out, over HTTP at `{module address}/steps/{step}` with a token of the service account of Process Orchestration (permission `execute_steps_<module>`, FRAMEWORK 4.2). Lookups the same way under `/internal/...` (permission `query_<module>`).
- **Receiving events:** the module subscribes to all event types that its flows name. Delivery is at least once; the module ignores an `eventId` it has already processed.
- **Worker:** picks due step runs, locks them (`SELECT … FOR UPDATE SKIP LOCKED`), calls the step, stores the result. Several workers may run; a step run is never called twice at the same time.
- **Flow definitions** are read from `FLOWS_FILE` at start-up and checked (section 6.2). Each process keeps the version of the flow it started with.
- Timeout per step call 30 seconds.

### Configuration

| Variable | Meaning | Example |
|---|---|---|
| `DATABASE_URL` | own database `c12_m1` | |
| `FLOWS_FILE` | flow definitions | `config/flows.yaml` |
| `MODULE_URLS` | address per module: empty = built-in stand-in (section 11.3), `in-process`, or HTTP address (FRAMEWORK 4.7) | `{"c3-m2": "in-process", "c8-m1": ""}` |
| `RETRY_BASE_SECONDS`, `RETRY_FACTOR`, `RETRY_MAX_SECONDS` | wait time before a repetition | `5`, `2`, `600` |
| `MAX_ATTEMPTS` | attempts per step, unless the flow sets its own | `5` |
| `AWAIT_TIMEOUT` | longest wait for a completing event, unless the flow sets its own (ISO 8601) | `PT24H` |
| `WORKER_POLL_SECONDS` | interval of the worker for due repetitions | `2` |
| `STANDIN_FILE` | optional results of the built-in stand-ins that replace the defaults of 11.3 | `config/standins.yaml` |
| `SSE_HEARTBEAT_SECONDS` | interval of the heartbeat comment in an event stream (7.6) | `15` |
| `SSE_REPLAY_SIZE` | number of stream events the module keeps in memory for a reconnect (7.6) | `1000` |

## 5. Data

### Table `process`

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key; `processId` |
| `flow_key`, `flow_version` | text, int | from [flows.yaml](flows.yaml) |
| `subject_type`, `subject_id` | text, uuid | from the trigger event |
| `correlation_key`, `correlation_value` | text, text | e.g. `companyId` and its value |
| `status` | text | 6.4 |
| `variables` | jsonb | process variables; only IDs and business values, never secrets |
| `lookups` | jsonb | results of the lookups, kept for the whole process |
| `trigger_event_id` | uuid | `eventId` of the trigger event |
| `end_reason` | text | with `ENDED`: the event that ended the process |
| `created_at`, `updated_at`, `finished_at` | timestamptz | |

At most one process with status `RUNNING`, `WAITING` or `FAILED` per flow and correlation value (unique index).

### Table `step_run`

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key; `stepRunId`, stays the same for all attempts |
| `process_id` | uuid | |
| `step_key`, `call` | text | from the flow |
| `state` | text | 6.3 |
| `attempts`, `max_attempts` | int | |
| `next_attempt_at` | timestamptz | due time of the next attempt |
| `await_until` | timestamptz | end of the wait for a completing event |
| `result` | jsonb | `result` of the step, or the payload of the completing event |
| `last_error` | jsonb | `type`, `status`, `title`, `retryable` of the last Problem Details |
| `started_at`, `finished_at` | timestamptz | |

Unique: `process_id` with `step_key`.

### Table `step_attempt`

One row per call: `step_run_id`, `number`, `trigger` (`WORKER`, `RETRIGGER`), `triggered_by` (`sub` with `RETRIGGER`), `started_at`, `finished_at`, `outcome` (`DONE`, `RUNNING`, `ERROR`), `error_type`.

### Table `processed_event`

`event_id` (primary key), `event_type`, `received_at`. Detects repeated delivery.

### Table `audit_entry`

Retriggers by OSP staff: time, `sub`, `process_id`, `step_key`, comment (FRAMEWORK 4.3).

**Not stored:** tokens, secrets, personal data. Events with names do not exist (FRAMEWORK 4.6); if a module ever sent one, only the fields named in the flow would be kept.

## 6. Rules

### 6.1 Format of a flow

| Field | Meaning |
|---|---|
| `key` | unique name of the flow |
| `trigger.event`, `trigger.when` | event that starts a process; `when`: fields of the event with the required value (all must match) |
| `subject.type`, `subject.key` | subject of the process and the field of the trigger event that holds its ID |
| `correlateBy` | field that assigns later events to this process |
| `endOn` | events that end the process early (status `ENDED`); open steps become `SKIPPED`. Only as long as a `waitFor` event of the flow has not arrived (6.5) |
| `steps[]` | in order; a step starts when the previous step is `DONE` or `SKIPPED` |
| `steps[].call` | `<module>.<step>`, e.g. `c3-m2.register-company` |
| `steps[].waitFor` | event that must have arrived before the step starts (e.g. `ApplicationApproved`) |
| `steps[].when` | condition: source (process variable or `lookup.<name>.<field>`) → required value; all must match; not met: `SKIPPED` |
| `steps[].lookup` | lookup to run before the step and before its `when` (once per process; the result is kept; 6.8) |
| `steps[].input` | input of the step: field → source (see below); `processId` and `stepRunId` are always added |
| `steps[].completesOn` | events that complete the step after `RUNNING`; optional `set` adds process variables |
| `steps[].failsOn` | events that make the step `FAILED` (not retryable) |
| `maxAttempts`, `awaitTimeout` | per step, otherwise from `defaults` |

Sources of an input value: `name` (process variable), `steps.<step>.<field>` (result of an earlier step), `lookup.<name>.<field>` (lookup, 6.8), `=VALUE` (fixed value). A missing source value is left out of the input; if the step requires it, the step answers 400 and the step run becomes `FAILED`. A lookup always provides the field `found` (6.8).

`version` at the top of [flows.yaml](flows.yaml) is the version of every flow in the file (`flow_version` in table `process`, `version` in the answer of 7.5).

The key for a waiting step is `waitFor`, not `on`: YAML 1.1 parsers read `on` as a boolean.

### 6.2 Check at start-up

The module refuses to start if a flow names an unknown step, an unknown event, an unknown lookup, a step source that is not earlier in the flow, or a required input of a step without a source. The check script in the test tools does the same against the `openapi.yaml` of all modules.

### 6.3 Status of a step run

| From | Event | To |
|---|---|---|
| – | process started | `WAITING` |
| `WAITING` | previous step `DONE` or `SKIPPED` and `waitFor` event arrived | `READY`, or `SKIPPED` if `when` is not met |
| `READY` | worker calls the step (after the lookup of the step, 6.8) | `IN_PROGRESS` |
| `READY` | the lookup of the step fails (6.8) | as an error of the call: `READY` with `next_attempt_at`, or `FAILED` |
| `IN_PROGRESS` | answer `DONE` | `DONE` |
| `IN_PROGRESS` | answer `RUNNING` | `AWAITING_EVENT` |
| `AWAITING_EVENT` | event from `completesOn` | `DONE` |
| `AWAITING_EVENT` | event from `failsOn`, or `await_until` passed | `FAILED` |
| `IN_PROGRESS` | error with `retryable: true`, attempts left | `READY` with `next_attempt_at` |
| `IN_PROGRESS` | error with `retryable: false`, or no attempts left | `FAILED` |
| `FAILED` | retrigger by OSP staff | `READY`, attempts start again |
| `WAITING`, `READY`, `AWAITING_EVENT` | process ended (`endOn`) | `SKIPPED` |
| `IN_PROGRESS` | process ended (`endOn`) | the running call is completed and recorded, then `SKIPPED`; no further step starts |

Wait time before attempt n+1: `RETRY_BASE_SECONDS × RETRY_FACTOR^(n−1)`, at most `RETRY_MAX_SECONDS`. Errors without `retryable` count as `retryable: false`. Timeouts and connection errors count as `retryable: true`.

`await_until` (`AWAIT_TIMEOUT`) applies only to `AWAITING_EVENT`. A step that waits for its `waitFor` event (e.g. the decision on an application) has no time limit.

### 6.4 Status of a process

Derived from the step runs and stored for queries:

| Status | When |
|---|---|
| `RUNNING` | a step is `READY`, `IN_PROGRESS` or `AWAITING_EVENT` |
| `WAITING` | the next step waits for a `waitFor` event, e.g. the decision on an application |
| `FAILED` | a step is `FAILED` |
| `COMPLETED` | all steps `DONE` or `SKIPPED` |
| `ENDED` | an `endOn` event arrived (e.g. `ApplicationDeclined`) |

### 6.5 Events and processes

1. An event whose type is a `trigger.event` and whose fields meet `trigger.when` starts a process, unless a process of this flow with the same correlation value is open (`RUNNING`, `WAITING`, `FAILED`); then the event is only recorded.
2. An event that a step of an open process names in `waitFor`, `completesOn`, `failsOn` or the flow in `endOn` is assigned through `correlateBy`. Its fields are added to the process variables; with `completesOn` the payload becomes the result of the step.
3. An event that arrives while its step is still `WAITING` or `IN_PROGRESS` is kept and applied as soon as the step can use it (the module may publish the event before it answers `RUNNING`).
4. Events that no flow names are not stored.
5. An `endOn` event ends the process only as long as at least one `waitFor` event of the flow has not arrived. Afterwards the decision is taken; the event is only recorded and the process continues. Example: `ApplicationWithdrawn` after `ApplicationApproved` (C1-M2 6.10) does not end the onboarding; the flow `application-withdrawn` then fails at `close-review`, and OSP staff are informed through `ProcessStepFailed` (FRAMEWORK 5.2 step 5).

### 6.6 Permissions

| Permission | Scope | What for |
|---|---|---|
| `view_processes` | `osp` | view processes, step runs and flows (portal role `OSP Admin`) |
| `retrigger_process_steps` | `osp` | retrigger a failed step (portal role `OSP Admin`) |
| `query_c12_m1` | `osp` | query processes of a subject over HTTP (FRAMEWORK 4.2) |
| `register_company` | `own` | follow the processes of the own application live (7.8); permission of C1-M2 that every applicant has (portal role `Applicant`) |

The permissions and portal roles are defined in C2-M5 6.2 and 6.3.

Companies do not see processes; they see the status at the owning module (proposal, open-points No. 2). An applicant sees only the live state of the processes of their own application (7.8), without variables, results or errors.

### 6.7 Repeatability

- A step run keeps its `stepRunId` across all attempts and retriggers. Modules find their object by a business key (FRAMEWORK 4.4); a repeated call returns the same result.
- An `eventId` is processed once.
- Retrigger of a step that is not `FAILED`: 409, nothing changes.

### 6.8 Lookups

A lookup is declared once under `lookups` in [flows.yaml](flows.yaml). It runs when the first step that names it (`steps[].lookup`) becomes `READY`, before its `when` is checked. A successful result is kept for the whole process.

| Field | Meaning |
|---|---|
| `query` | `<module>.<operationId>` of a `GET` operation under `/internal` in the `openapi.yaml` of that module, e.g. `c3-m1.queryCompany` for `GET /api/c3-m1/internal/companies/{companyId}` |
| `params` | parameter of that operation → source (as for `input`, 6.1). The operation defines whether a parameter is a path or a query parameter |
| `fields` | field of the lookup → field path in the answer, or a mapping (below) |

Field paths:

| Form | Meaning | Example |
|---|---|---|
| `a.b` | field `b` of the object `a` | `address.countryCode` |
| `a[n]` | element `n` of the list `a`, counted from 0 | `items[0].id` |
| `[n]` at the start | element `n` of an answer that is a list | `[0].id` |
| `a[TYPE]` | first element of the list `a` whose field `type` has the value `TYPE` (capital letters, digits, `_`, `-`) | `identifiers[EBWOID].value` |

A path that does not exist gives no value; the field is then left out (6.1).

Mappings, when the value of a field is an object instead of a path:

- **Object:** every key is a field of the result, every value is a path or a mapping. Fields without a value are left out. Example: the C3-M2 `Address` (`streetAddress`, `postalCode`, `locality`, `country`) from the C3-M1 address in the lookup `company-core-data`.
- **`join`:** an object with the only key `join` and a list of paths. The values that are not empty, joined with one space; no value if all are empty. Example: `streetAddress: { join: [address.street, address.houseNumber] }`.

Result of a lookup:

| Answer of the query | Result |
|---|---|
| 200 with an object or a non-empty list | `found` `true` and the fields |
| 200 with an empty list, or 404 | `found` `false`, no other fields |
| any other answer (400, 401, 403, 409, 5xx), timeout, connection error | the lookup failed: the step is not called, and the step run is treated as after an error of the call (6.3). `retryable` from the Problem Details of the answer; 5xx, timeouts and connection errors `retryable: true`; otherwise `false`. `last_error.type` is the `type` of the answer, or `urn:c12-m1:problem:lookup-failed` without Problem Details |

A failed lookup is never `found: false`; otherwise steps with a `when` on the lookup would be `SKIPPED` without notice. The next attempt or a retrigger runs the lookup again.

## 7. Operations

All operations by OSP staff over HTTP with a token of the Portal IAM (tag `OSP`), except 7.8 for applicants (tag `Company`, scope `own`).

### 7.1 `GET /api/c12-m1/processes`

Permission `view_processes`. Filters: `flowKey`, `status`, `subjectType`, `subjectId`, `updatedSince`; paging as in FRAMEWORK 4.1, sorted by `updatedAt` descending. Response 200 `ProcessSummaryPage`.

### 7.2 `GET /api/c12-m1/processes/{processId}`

Permission `view_processes`. Response 200 `Process` with all step runs (without attempts). Unknown: 404.

### 7.3 `GET /api/c12-m1/processes/{processId}/steps/{stepKey}`

Permission `view_processes`. Response 200 `StepRunDetail` with the attempts. Unknown process or step: 404.

### 7.4 `POST /api/c12-m1/processes/{processId}/steps/{stepKey}/retrigger`

Permission `retrigger_process_steps`. Input optional `comment` (up to 500 characters).

1. Process or step unknown: 404.
2. Step not `FAILED`: 409 `invalid-state`.
3. Step becomes `READY`, attempts start at 0, `await_until` is reset; process status `RUNNING`.
4. Audit log entry, event `ProcessStepRetriggered`. Response 202 with the step run.

### 7.5 `GET /api/c12-m1/flows` and `GET /api/c12-m1/flows/{flowKey}`

Permission `view_processes`. The loaded flow definitions as JSON, so that the UI can show the steps of a process before they run.

### 7.6 Live updates (Server-Sent Events)

The UI learns about changes of a process without polling. Data flows only from the server to the browser; commands use the other operations. These rules apply to 7.7 and 7.8:

- **Format:** response 200 with `Content-Type: text/event-stream` (Server-Sent Events, HTML Living Standard). Errors before the stream starts (400, 401, 403, 404) as Problem Details.
- **Token:** in the header `Authorization: Bearer`, as for every other operation, and checked in the same way. The browser holds only a session cookie; the server part of the UI (backend for frontend, as in FRAMEWORK 5.2 step 2) opens the stream with the person's token and passes the events on to the browser. The module accepts no token in the query string, because URLs end up in access logs, and reads no cookie.
- **End of the token:** the module checks the token when the stream opens. When `exp` of the token has passed, the module closes the stream. The UI opens it again with a new token and `Last-Event-ID`.
- **Event:** name `process-changed`; `data` is one JSON object `ProcessChanged` ([openapi.yaml](openapi.yaml)): `processId`, `flowKey`, `subjectType`, `subjectId`, `status`, `currentStep`, `endReason`, `steps` (per step run `stepKey` and `state`), `updatedAt`. Only IDs, keys and states: no process variables, no step results, no error details, never secrets. Every event carries the full state of the process, so a missed event leaves no gap.
- **When:** after every stored change of the status of a process (6.4) or of the state of a step run (6.3). States that a step passes within one call (e.g. `IN_PROGRESS` of a step that answers `DONE` at once) may be combined into one event. A process that is `COMPLETED` or `ENDED` sends no further events; the stream stays open until the UI closes it.
- **`id`:** every event has an `id`. Clients treat it as an opaque value and only send it back. In this version it consists of an identifier of the module instance and a number that increases.
- **Start and reconnect:** without the header `Last-Event-ID`, the stream first sends one `process-changed` with the current state of each process of the stream. With `Last-Event-ID`, the module sends the kept events after this `id` that belong to the stream, then new ones. The module keeps the last `SSE_REPLAY_SIZE` events in memory. If it no longer has this `id` (too old, restart, another instance), it sends the current state as without the header.
- **Heartbeat:** the module sends a comment line (`: keep-alive`) every `SSE_HEARTBEAT_SECONDS`, so that proxies keep the connection open. The first message of a stream sets `retry: 3000` (the browser reconnects after 3 seconds).
- The event stream is a message to the UI, not an event of the event bus (section 10).

### 7.7 `GET /api/c12-m1/processes/{processId}/events`

Permission `view_processes` (scope `osp`), as 7.2. Events of this process (7.6). Unknown process: 404.

### 7.8 `GET /api/c12-m1/applications/{applicationId}/events`

Permission `register_company` (scope `own`, FRAMEWORK 4.3). The applicant follows the processes of their own application, e.g. the onboarding after submission (FRAMEWORK 5.2), without access to 7.1 to 7.5.

1. `applicationId` is not a UUID: 400.
2. The stream contains the processes with subject `APPLICATION` and this `applicationId` whose process variable `applicantSub` equals `sub` in the token (from `ApplicationSubmitted`). Other processes are never sent, e.g. `application-declined`, whose trigger does not carry `applicantSub`; the onboarding process shows the end with status `ENDED` and `endReason`.
3. The stream opens even if no such process exists yet: the process starts only when `ApplicationSubmitted` has been delivered, so the UI may open the stream right after submitting. For the application of another person, the stream stays empty; there is no 404, and nothing about the application is revealed.

## 8. Steps for Process Orchestration

None. This module calls the steps of the other modules; it offers none itself.

## 9. Queries for other modules

### `GET /api/c12-m1/internal/processes`

Parameters `subjectType`, `subjectId` (both required). Response 200: list of `ProcessSummary` of this subject, newest first. Use: Subscriptions shows the setup status of a subscription; C1-M2 may show why an approved application is not yet active.

## 10. Events

| Event | When | Content |
|---|---|---|
| `ProcessStarted` | a process was started | `processId`, `flowKey`, `subjectType`, `subjectId` |
| `ProcessStepFailed` | a step became `FAILED` | `processId`, `flowKey`, `subjectType`, `subjectId`, `stepKey`, `errorType`, `retryable` |
| `ProcessStepRetriggered` | OSP staff retriggered a step | `processId`, `stepKey`, `retriggeredBy` (`sub`) |
| `ProcessCompleted`, `ProcessEnded` | status `COMPLETED` or `ENDED` | `processId`, `flowKey`, `subjectType`, `subjectId` |

`errorType` in `ProcessStepFailed`: the `type` of the Problem Details of the last error (also of a failed lookup, 6.8); `event:<EventType>` for an event from `failsOn`; `await-timeout` when `await_until` has passed.

Notifications reacts to `ProcessStepFailed` (notice to OSP staff with `OSP Admin`) and closes the notice on `ProcessStepRetriggered`, `ProcessCompleted` or `ProcessEnded` (OP-18, OP-19).

The stream messages `process-changed` of 7.6 go only to the UI; they are not published on the event bus.

## 11. Calls to other modules and external systems

### 11.1 Steps

All steps of the step catalogue (FRAMEWORK 7), as named in [flows.yaml](flows.yaml). The flows and the events they use:

| Flow | Trigger | Waits for | Steps |
|---|---|---|---|
| `onboarding` | `ApplicationSubmitted` (`ONBOARDING`) | `ApplicationApproved`; ends on `ApplicationDeclined`, `ApplicationWithdrawn` | C3-M1 create-company (only new company), C1-M3 open-review, C3-M2 register-company, C3-M1 assign-identifier, C4-M2 create-wallet, C3-M2 register-did, C4-M1 issue-credentials, C3-M2 register-membership, C2-M2 assign-company-admin, C3-M1 activate-company, C1-M2 complete-application |
| `transfer` | `ApplicationSubmitted` (`TRANSFER`, new company) | as onboarding | as onboarding, with C3-M2 transfer-company instead of register-company |
| `return` | `ApplicationSubmitted` (`TRANSFER`, existing company) | as onboarding | C1-M3 open-review, C3-M2 transfer-company, C3-M1 reactivate-company, then as `reactivation` |
| `reactivation` | `ApplicationSubmitted` (`REACTIVATION`) | as onboarding | C1-M3 open-review, C3-M1 reactivate-company, C2-M2 enable-company-users, C2-M2 assign-company-admin, C2-M3 enable-company-technical-users, C4-M1 issue-credentials, C3-M2 register-membership, C1-M2 complete-application |
| `application-declined` | `ApplicationDeclined` | – | C1-M2 close-application, C3-M1 reject-company (only new company) |
| `application-withdrawn` | `ApplicationWithdrawn` | – | C1-M3 close-review, C3-M1 reject-company (only new company) |
| `deactivation` | `CompanyDeactivated` | – | C2-M2 disable-company-users, C2-M3 disable-company-technical-users, C4-M1 revoke-credentials, C3-M2 deregister-membership |
| `transferred-away` | `CompanyTransferredAway` | – | C3-M1 deactivate-company (`TRANSFERRED`), which leads to `deactivation` |
| `core-data-changed` | `CompanyDataChanged` (`coreDataChanged`) | – | C3-M2 update-company |
| `wallet-access` | `WalletAccessRequested` | – | C4-M2 create-wallet-account, C2-M3 apply-wallet-access |
| `wallet-access-removal` | `WalletAccessRemovalRequested` | – | C4-M2 delete-wallet-account, C2-M3 delete |
| `app-approval` | `OfferApproved` (`APP`) | – | C2-M2 create-client-template; single-instance app: create-instance-client, enable-instance-client |
| `app-roles-added`, `app-role-descriptions-changed` | `OfferRolesAdded`, `OfferRoleDescriptionsChanged` | – | C2-M2 add-app-roles, update-role-descriptions |
| `subscription-requested` | `SubscriptionRequested` | – | C10-M1 notify-provider |
| `subscription-setup` | `SubscriptionSetupStarted` | `SubscriptionActivated`; ends on `SubscriptionDeclined`, `SubscriptionDeactivated` | C2-M2 create-instance-client (apps, `offerType` of the event), C2-M3 create-for-subscription (if the offer has a technical user profile), C2-M2 enable-instance-client, C10-M1 send-activation-callback |
| `single-instance-activation` | `SubscriptionActivated` (`instanceKind` `SINGLE`) | – | C2-M3 create-for-subscription (if the offer has a technical user profile), C10-M1 send-activation-callback |
| `subscription-declined` | `SubscriptionDeclined` | – | C2-M2 disable-instance-client (if the setup created one), C2-M3 disable-subscription-technical-users, C6-M1 deactivate-subscription-connectors |
| `subscription-ended` | `SubscriptionDeactivated` | – | C2-M2 disable-instance-client, C2-M3 disable-subscription-technical-users, C6-M1 deactivate-subscription-connectors |
| `instance-url-changed` | `SubscriptionInstanceUrlChanged` | – | C2-M2 update-instance-url |
| `technical-user-deleted` | `TechnicalUserDeleted` | – | C6-M1 unlink-technical-user |
| `connector-registered`, `connector-endpoint-changed` | `ConnectorRegistered`, `ConnectorEndpointChanged` | – | C3-M2 register-connector (FRAMEWORK 5.9) |
| `connector-deactivated`, `connector-deleted` | `ConnectorDeactivated`, `ConnectorDeleted` | – | C3-M2 deregister-connector (FRAMEWORK 5.9) |

The events of Offers, Subscriptions and Connectors are defined in their specs (version 0.2).

### 11.2 Lookups

| Lookup | Query (`operationId`) | Used for |
|---|---|---|
| `company-core-data` | C3-M1 `queryCompany`, `GET /internal/companies/{companyId}`: `name`, identifiers (`EBWOID` and `EUID` separately, and the full list), `address` mapped to the C3-M2 `Address` (`street` and `houseNumber` → `streetAddress`, `postalCode`, `city` → `locality`, `countryCode` → `country`; C3-M1 9.1) | input of C3-M2 `register-company`, `transfer-company`, `update-company` |
| `offer` | Offers `getOfferInternal`, `GET /internal/offers/{offerId}`: `offerType`, `name`, `providerCompanyId`, `instanceKind`, `instanceUrl`, `roles`, `hasTechnicalUserProfile`, `technicalUserProfile` | app approval, subscription setup, single-instance activation |
| `subscription-instance-client` | C2-M2 `listAppInstanceClients`, `GET /internal/app-instance-clients?offerSubscriptionId=` (answer: list; `found` if not empty) | end or decline of a subscription, URL change |

### 11.3 Built-in stand-in

With an empty address in `MODULE_URLS`, the module answers the steps and lookups of that module itself. The stand-in answers every step with `DONE` at once and never publishes events. A step with `completesOn` gets the `set` values of its first entry, as if that event had arrived. This way a flow can be run end to end, also in mixed operation where only some neighbours are real modules.

Step results of the stand-in (all other steps: empty `result`):

| Step | `result` |
|---|---|
| `c3-m2.register-company` | `bpn` derived from the input as the stand-in of C3-M2 does it (C3-M2 11.3); from an EBWOID: `BPNL` + the first 12 characters of the Base32 hash (SHA-256) of the EBWOID, in upper case |
| `c3-m2.transfer-company` | `bpn` from the input |
| `c4-m2.create-wallet` | `did` = `did:web:wallet.example.org:` + `companyId` |
| `c4-m2.create-wallet-account` | `walletAccountRef` = the `stepRunId` |
| `c2-m2.create-instance-client` | `id` = the `stepRunId`, `iamClientId` = `standin-` + the `stepRunId` |
| `c2-m3.create-for-subscription` | `accounts` = `[]` |

Lookup answers of the stand-in:

| Query | Answer |
|---|---|
| `c3-m1.queryCompany` | the company of section 14 with the `companyId` of the parameter |
| `c8-m1.getOfferInternal` | the multi-instance app of section 14 with the `offerId` of the parameter: `offerType` `APP`, `instanceKind` `MULTI`, roles `Viewer` and `Editor`, `hasTechnicalUserProfile` `true`, profile `connector-sync` with role `Dataspace Discovery` (as C8-M1 14) |
| `c2-m2.listAppInstanceClients` | empty list (`found` `false`) |

`STANDIN_FILE` (YAML) replaces these defaults: `steps` maps `<module>.<step>` to a `result`, `lookups` maps `<module>.<operationId>` to an answer.

## 12. Errors

Problem Details, `type` = `urn:c12-m1:problem:<name>`.

| `type` | HTTP | When |
|---|---|---|
| `validation` | 400 | invalid parameter or input |
| `unauthorized` | 401 | token missing or invalid |
| `forbidden` | 403 | permission missing |
| `not-found` | 404 | process, step or flow unknown |
| `invalid-state` | 409 | retrigger of a step that is not `FAILED` |

Errors of the called steps are not passed on; they are stored in `last_error` of the step run (type, HTTP status, title, `retryable`).

## 13. Acceptance

1. Start-up with [flows.yaml](flows.yaml) (25 flows): the check (6.2) passes. A flow with an unknown step stops the start with a message naming flow and step.
2. `ApplicationSubmitted` (`ONBOARDING`, `existingCompany` `false`) with all stand-ins: `create-company` and `open-review` are `DONE`, process `WAITING` on `ApplicationApproved`.
3. `ApplicationApproved` for the same `companyId`: the steps from `register-company` to `complete-application` run in order; process `COMPLETED`; `ProcessCompleted` once.
4. The same `ApplicationSubmitted` (same `eventId`) a second time: no second process. A new event with another `eventId` for the same company while the process is open: no second process.
5. `ApplicationDeclined` while the onboarding process waits: process `ENDED`, open steps `SKIPPED`; flow `application-declined` runs `close-application` and `reject-company`.
6. `ApplicationSubmitted` with `existingCompany` `true` (`REACTIVATION`): flow `reactivation`, no `create-company`.
7. A step answers `RUNNING`: step `AWAITING_EVENT`; the event from `completesOn` completes it, its payload is the step result (e.g. `did` for `register-did`). An event from `failsOn`: step `FAILED`, `ProcessStepFailed`.
8. The completing event arrives before the step answer `RUNNING`: the step still becomes `DONE` (6.5 point 3).
9. A step fails with `retryable: true` three times, then succeeds: 4 attempts, wait times 5 s, 10 s, 20 s; process continues.
10. A step fails with `retryable: false`: `FAILED` after one attempt, `ProcessStepFailed` with `stepKey` and `errorType`.
11. Retrigger by a person with `retrigger_process_steps`: 202, step `READY`, attempts from 0, `ProcessStepRetriggered`; without the permission 403; for a `DONE` step 409.
12. `when` not met (e.g. `create-instance-client` for an offer of type `SERVICE`): step `SKIPPED`, next step starts.
13. `GET /internal/processes?subjectType=SUBSCRIPTION&subjectId=…` returns the subscription setup process.
14. Two workers on the same database: no step run is called twice at the same time (test with a step that takes 2 seconds).
15. The lookup `offer` answers 403: `create-instance-client` is not called and not `SKIPPED`; the step run is `FAILED` with `last_error.type` of the answer and `retryable` `false`; `ProcessStepFailed`. The lookup answers 503 three times, then 200: the step runs after 4 attempts.
16. Onboarding with the C3-M1 address Musterstraße 1, 70569 Stuttgart, `DE`: `register-company` receives `address` `{ "streetAddress": "Musterstraße 1", "postalCode": "70569", "locality": "Stuttgart", "country": "DE" }`.
17. Lookup `subscription-instance-client` with the answer `[{ "id": "…" }]`: `found` `true`, `appInstanceClientId` is the `id`; with `[]`: `found` `false`, `disable-instance-client` `SKIPPED`.
18. All neighbours as stand-ins: the onboarding runs to `COMPLETED`; `assign-identifier` receives the stand-in BPN and `register-did` the stand-in DID. Mixed: real C3-M1, all other stand-ins: the same.
19. `ApplicationWithdrawn` after `ApplicationApproved`: the onboarding process is not `ENDED` and continues; the flow `application-withdrawn` fails at `close-review` with `ProcessStepFailed`.
20. OSP staff (Otto) open 7.7 for the onboarding process of case 2: the first event `process-changed` shows `WAITING`, `create-company` and `open-review` `DONE`. After `ApplicationApproved`, further events follow; the last one has status `COMPLETED`. No event contains process variables, results or secrets.
21. The applicant (Erika, `applicantSub` of the trigger event) opens 7.8 for the application of case 2 before `ApplicationSubmitted` is delivered: the stream stays open without events, then sends the state of the onboarding process. Lieschen opens 7.8 for the same application: no event. 7.7 with Erika's token: 403.
22. Reconnect with `Last-Event-ID` of an earlier event: the stream sends the events after it that were missed, then new ones. With an unknown `Last-Event-ID`: the current state, as at the start.
23. 7.7 or 7.8 without a token or with an invalid token: 401 as Problem Details, no stream. A token in the query string (`access_token`) counts as no token.

## 14. Test data

| Object | Value |
|---|---|
| Company (new) | `companyId` `c0000001-0000-4000-8000-000000000001`, Beispiel Maschinenbau GmbH, `EBWOID-DE-0000000001` (FRAMEWORK 7a) |
| Application | `applicationId` `0f8e2d4c-6b1a-4c3e-8f7d-2a9b1c0d3e4f`, kind `ONBOARDING`, `existingCompany` `false`, `applicantSub` `5a1c0001-0000-4000-8000-000000000001` (Erika Mustermann, FRAMEWORK 7a) |
| Address (C3-M1) | Musterstraße 1, 70569 Stuttgart, region `BW`, `DE` (as C3-M1 14) |
| BPN from the stand-in | `BPNLIUK2REMLK3F4` (from `EBWOID-DE-0000000001`, 11.3) |
| App (multi-instance) | `offerId` `7c1d9e3a-2b4f-4a6c-8d0e-5f1a2b3c4d5e`, provider: company of OSP A `11111111-1111-4111-8111-111111111111` (FRAMEWORK 7a) |

Example event that starts the onboarding:

```json
{ "eventId": "b8e1f7a0-1c2d-4e3f-9a8b-7c6d5e4f3a2b", "eventType": "ApplicationSubmitted",
  "occurredAt": "2026-11-06T09:30:00Z", "applicationId": "0f8e2d4c-6b1a-4c3e-8f7d-2a9b1c0d3e4f",
  "companyId": "c0000001-0000-4000-8000-000000000001", "kind": "ONBOARDING",
  "existingCompany": false, "applicantSub": "5a1c0001-0000-4000-8000-000000000001" }
```

## Changes from version 0.1

- Flows are data in [flows.yaml](flows.yaml) and cover FRAMEWORK 5 (OSP 2.0). The legacy process types (`APPLICATION_CHECKLIST`, `INVITATION`, `PARTNER_REGISTRATION`, `IDENTITY_PROVIDER_PROVISIONING`, `MAILING`, …) and their mapping are dropped.
- Modules no longer register executors or complete steps themselves; the module calls `/steps/...` and completes long-running steps by event (decision of 20/22 Oct 2026, now the only model).
- Step result `DONE`/`RUNNING` and errors with `retryable` (FRAMEWORK 4.4) instead of `recoverable` and step kinds `AUTOMATIC`/`MANUAL`/`AWAIT_CALLBACK`.
- Paths under `/api/c12-m1`, permissions with scope `osp`; the legacy retrigger roles (`approve_new_partner`, `retrigger_subscription`, …) are dropped.
- The tenant field of a process is dropped: one installation per OSP.
- Companies no longer see processes (subject policies dropped).
- Event names and contents as in FRAMEWORK 6; `common.yaml` and its event envelope are dropped.
- Server-Sent Events for the UI moved to open points (No. 6).
- Lookups (6.8): `query` names the `operationId`, a path grammar for fields, mappings with nested objects and `join` (address C3-M1 → C3-M2), `found` only for 200 and 404; any other answer makes the step fail like a step error.
- `subscription-instance-client` reads `[0].id`, because C2-M2 answers with a list.
- Built-in stand-in (11.3) returns results per step (`bpn`, `did`, `id`, `accounts`) and lookup answers; `STANDIN_EVENTS` dropped, `STANDIN_FILE` added.
- `endOn` applies only before the `waitFor` events of a flow have arrived; `IN_PROGRESS` and `AWAITING_EVENT` on end (6.3, 6.5).
- Flows `single-instance-activation` and `subscription-declined`; `subscription-setup` also ends on `SubscriptionDeactivated`, decides on `offerType` of the event and passes `clientId` to the callback.
- `errorType` values (10); `version` of the file is the version of every flow (6.1); `waitFor` has no time limit (6.3).
- Flows `subscription-ended` and `subscription-declined` call C2-M3 `disable-subscription-technical-users` (C2-M3 8.6); open question 1 closed (30 Oct 2026).
- Permission `query_c12_m1` (renamed from `query_c12`) with the full module identifier as in FRAMEWORK 4.2 (30 Oct 2026).
- Test data with the common test IDs of FRAMEWORK 7a; `applicantSub` is the account ID (renamed from `ebw.test-erika`) (30 Oct 2026).
- Live updates for the UI as Server-Sent Events, no WebSockets (decided on 30 Oct 2026; open question 6 closed): 7.6 to 7.8, event `process-changed` with the full state of the process, `Last-Event-ID` with a buffer in memory, heartbeat, token in the header only. Applicants follow the processes of their own application with `register_company` (7.8, 6.6). Configuration `SSE_HEARTBEAT_SECONDS`, `SSE_REPLAY_SIZE` (30 Oct 2026).
