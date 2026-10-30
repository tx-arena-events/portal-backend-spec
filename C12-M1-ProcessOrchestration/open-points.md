# C12-M1 Process Orchestration: open points

**Version:** 0.2, as of 2026-10-30

## Provisional decisions

| No. | Topic | Decision in the README | To be agreed with |
|---|---|---|---|
| 1 | Retry (OP-18) | 5 attempts, wait 5 s, doubling, at most 10 min; then `FAILED` and `ProcessStepFailed`; notice to OSP staff through Notifications (README 6.3, 10) | project owner, other part |
| 2 | Visibility | only OSP staff see processes (`view_processes`, scope `osp`); companies see the status at the owning module (README 6.6). Exception: an applicant follows the live state of the processes of their own application (README 7.8). A live stream for a company (scope `company`, e.g. the setup of a subscription) does not exist yet | project owner |
| 3 | Permissions | `view_processes` and `retrigger_process_steps` for `OSP Admin`, as in C2-M5 6.2 and 6.3. `OSP Reviewer` does not see processes | C2-M5 |
| 4 | Order of the onboarding steps | strictly in the order of FRAMEWORK 5.2 step 6, one after the other. Steps without a dependency on each other (e.g. `register-membership` and `assign-company-admin`) could run in parallel later | other modules (preconditions) |
| 5 | One open process per flow and subject | a second trigger for the same correlation value is only recorded (README 6.5) | – |
| 6 | Wait for an event | at most 24 hours (`AWAIT_TIMEOUT`) for a completing event, then `FAILED`; retrigger calls the step again. A `waitFor` event (a decision) has no time limit | operators of the external systems |
| 7 | Flows per OSP | every OSP may replace `flows.yaml` in its installation; the check at start-up (README 6.2) protects against broken flows | project owner |
| 8 | Built-in stand-ins | every step `DONE` with a result per step (`bpn`, `did`, `id`, `accounts`) and lookup answers with data; no events; `STANDIN_FILE` replaces the defaults (README 11.3) | test tools |
| 9 | Failed lookup | only 200 and 404 (or an empty list) give a result; any other answer makes the step fail like a step error (README 6.8) | – |
| 10 | Mapping in lookups | nested objects and `join` in `fields` build the input of the target step, e.g. the C3-M2 `Address` from the C3-M1 address; C3-M1 and C3-M2 keep their formats (README 6.8) | C3-M1, C3-M2 |
| 11 | Lists of internal queries | `GET /internal/processes` answers with a list without paging, as C2-M2 `GET /internal/app-instance-clients`; lookups address such answers with `[0]` | FRAMEWORK 4.1 |
| 12 | `endOn` after a decision | an `endOn` event after all `waitFor` events of the flow is only recorded; the process continues (README 6.5). Compensation (e.g. undo a registration at the CSP) is not part of the first version | C1-M2, project owner |
| 13 | Single-instance apps and declined subscriptions | flows `single-instance-activation` (technical user and callback after `SubscriptionActivated`) and `subscription-declined` (disable the instance client, deactivate the technical users of the subscription, deactivate connectors) | C10-M1, C2-M2, C2-M3, C6-M1 |
| 14 | Live updates for the UI | Server-Sent Events, no WebSockets (decided on 30 Oct 2026): data flows only to the browser, commands use the REST operations. One event name `process-changed` with the full state of the process (IDs, keys, states), so that a missed event leaves no gap; replay after `Last-Event-ID` from a buffer in memory (`SSE_REPLAY_SIZE`), otherwise the current state; heartbeat every 15 s (README 7.6) | user interface |
| 15 | Token of an event stream | only in the header `Authorization`, as for every other operation. The server part of the UI holds the session cookie, opens the stream with the token and passes the events on; no `access_token` in the URL, no cookie in the module. The stream ends when the token expires; the UI reconnects with `Last-Event-ID` (README 7.6) | user interface, C2-M5 |
| 16 | Permission of applicants for 7.8 | `register_company` (scope `own`), the permission of C1-M2 that every applicant has; the check is `applicantSub` of the process. C2-M5 6.2 names C1-M2 as the only module of this permission | C2-M5, C1-M2 |

## Assumptions about other modules

| Who | Assumption |
|---|---|
| C1-M2 | `ApplicationSubmitted` carries `applicationId`, `companyId`, `kind`, `existingCompany`, `applicantSub` (FRAMEWORK 6) |
| C1-M3 | `ApplicationApproved` carries `bpn` for `TRANSFER` and `REACTIVATION` (used for `transfer-company` and `assign-company-admin`) |
| C3-M1 | `GET /internal/companies/{companyId}` (`queryCompany`) provides `name`, `identifiers` (type and value) and `address` (`street`, `houseNumber`, `postalCode`, `city`, `countryCode`); used as input of the C3-M2 steps. The list `identifiers` also contains `EBWOID` and `EUID` |
| C3-M2 | `register-company` needs `ebwoid`, `euid` or an entry in `identifiers` (C3-M2 6.5); the flows pass all three, so a company without a Business Wallet (OP-35) can be registered. Steps `register-connector`, `deregister-connector` (C3-M2 8.6) |
| C2-M3 | step `disable-subscription-technical-users` with `offerSubscriptionId` (C2-M3 8.6) in the flows `subscription-ended` and `subscription-declined` |
| C4-M2 | the result of `create-wallet` or the event `DataspaceWalletCreated` contains `did`; the result of `create-wallet-account` or `WalletAccountCreated` contains `walletAccountRef` |
| Modules with long-running steps | publish the completing event also if they answer `DONE` at once; Process Orchestration takes the step result in that case |
| Offers (version 0.2) | events `OfferApproved` (`offerId`, `offerType`, `providerCompanyId`), `OfferRolesAdded`, `OfferRoleDescriptionsChanged` (`offerId`, `roles`); query `GET /internal/offers/{offerId}` (`getOfferInternal`) with the fields of README 11.2 |
| Subscriptions (version 0.2) | steps `notify-provider`, `send-activation-callback`; events `SubscriptionRequested`, `SubscriptionSetupStarted` (multi-instance apps and services: `offerSubscriptionId`, `offerId`, `offerType`, `customerCompanyId`, `providerCompanyId`, `instanceUrl`), `SubscriptionActivated` (with `offerType`, `instanceKind`), `SubscriptionDeclined`, `SubscriptionDeactivated`, `SubscriptionInstanceUrlChanged` |
| Notifications | reacts to `ProcessStepFailed` and to the end events of README 10 |
| Connectors (version 0.2) | steps `deactivate-subscription-connectors`, `unlink-technical-user`; events `ConnectorRegistered`, `ConnectorEndpointChanged` (with `endpoint`), `ConnectorDeactivated`, `ConnectorDeleted`, all with `connectorId` and `companyId` (host company) |
| C2-M5 | service account of Process Orchestration with `execute_steps_<module>` and `query_<module>` for split-out modules |

## Open questions

1. **Technical users at the end of a subscription:** closed. The flows `subscription-ended` and `subscription-declined` call C2-M3 `disable-subscription-technical-users` (C2-M3 8.6, FRAMEWORK 7).
2. **URL of a single-instance app:** C2-M2 `update-instance-url` exists, but Offers has no event for it yet (C2-M2 open points).
3. **Order of `register-did` and `issue-credentials`:** closed. C4-M1 8.1 issues in onboarding after `CompanyDidRegistered`; the flow keeps `register-did` before `issue-credentials`.
4. **Several workers and the event bus:** with an in-process bus, every instance of the monolith receives every event. The unique index on open processes (README 5) prevents double processes; whether one instance should be the only consumer is open (decision on the event transport of 22 Oct 2026).
5. **Retention:** how long finished processes and attempts are kept (OP-20).
6. **Live status for the UI:** closed. Server-Sent Events (README 7.6 to 7.8, decided on 30 Oct 2026); polling `GET /processes/{id}` still works.
7. **BPN as company identifier** (raised at the workshop of 27 Oct 2026: a dataspace profile without BPDM and BPN should be possible). The flows pass `bpn` as a value only and do not depend on its format; the steps that require it belong to C3-M1, C3-M2 and C2-M2.
8. **Several instances and event streams:** the buffer for `Last-Event-ID` is per instance in memory, and the event `id` names the instance. After a reconnect to another instance, the client gets the current state (no gap, because every event carries the full state). A shared buffer (e.g. a table, or the outbox) is needed only if that is not enough.
9. **Limits for event streams:** number of open streams per person and per instance, and the longest duration of a stream, are not defined yet (operations of the installation).
10. **Event streams in FRAMEWORK 4.1:** FRAMEWORK does not yet say that paths ending in `/events` are Server-Sent Events streams with the token in the header (7.6, also C11-M1 7.6). Proposal: one sentence in FRAMEWORK 4.1.
11. **Paths in the UI specification:** the UI specification assumes `GET /api/processes/{id}`, `/events` and fields in `snake_case`; this spec has `/api/c12-m1/processes/{processId}/events` (staff) and `/api/c12-m1/applications/{applicationId}/events` (applicant), fields in `camelCase` (FRAMEWORK 4.1). The process starts only after submission; the steps before (wallet connection, company data) belong to C1-M2 and C5-M1, not to a process.

## Out of scope

- A workflow engine (Temporal, Camunda); the REST contract would stay the same.
- Flows across installations or with several CSPs (FRAMEWORK 1, OP-44).
- Manual steps with a person in the loop; decisions stay in the owning modules, which publish an event.
