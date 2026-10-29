# C12-M1 Process Orchestration: open points

**Version:** 0.2, as of 2026-10-29

## Provisional decisions

| No. | Topic | Decision in the README | To be agreed with |
|---|---|---|---|
| 1 | Retry (OP-18) | 5 attempts, wait 5 s, doubling, at most 10 min; then `FAILED` and `ProcessStepFailed`; notice to OSP staff through Notifications (README 6.3, 10) | project owner, other part |
| 2 | Visibility | only OSP staff see processes (`view_processes`, scope `osp`); companies see the status at the owning module (README 6.6) | project owner |
| 3 | Permissions | `view_processes` and `retrigger_process_steps` for `OSP Admin`, as in C2-M5 6.2 and 6.3. `OSP Reviewer` does not see processes | C2-M5 |
| 4 | Order of the onboarding steps | strictly in the order of FRAMEWORK 5.2 step 6, one after the other. Steps without a dependency on each other (e.g. `register-membership` and `assign-company-admin`) could run in parallel later | other modules (preconditions) |
| 5 | One open process per flow and subject | a second trigger for the same correlation value is only recorded (README 6.5) | – |
| 6 | Wait for an event | at most 24 hours (`AWAIT_TIMEOUT`), then `FAILED`; retrigger calls the step again | operators of the external systems |
| 7 | Flows per OSP | every OSP may replace `flows.yaml` in its installation; the check at start-up (README 6.2) protects against broken flows | project owner |
| 8 | Built-in stand-ins | every step `DONE`, optional completing events (README 11.3) | test tools |

## Assumptions about other modules

| Who | Assumption |
|---|---|
| C1-M2 | `ApplicationSubmitted` carries `applicationId`, `companyId`, `kind`, `existingCompany`, `applicantSub` (FRAMEWORK 6) |
| C1-M3 | `ApplicationApproved` carries `bpn` for `TRANSFER` and `REACTIVATION` (used for `transfer-company` and `assign-company-admin`) |
| C3-M1 | `GET /internal/companies/{companyId}` provides `name`, `identifiers` (type and value) and `address.countryCode`; used as input of the C3-M2 steps. The list `identifiers` also contains `EBWOID` and `EUID` |
| C3-M2 | `register-company` needs `ebwoid`, `euid` or an entry in `identifiers` (C3-M2 6.5); the flows pass all three, so a company without a Business Wallet (OP-35) can be registered. Steps `register-connector`, `deregister-connector` (C3-M2 8.6) |
| C4-M2 | the result of `create-wallet` or the event `DataspaceWalletCreated` contains `did`; the result of `create-wallet-account` or `WalletAccountCreated` contains `walletAccountRef` |
| Modules with long-running steps | publish the completing event also if they answer `DONE` at once; Process Orchestration takes the step result in that case |
| Offers (version 0.2) | events `OfferApproved` (`offerId`, `offerType`, `providerCompanyId`), `OfferRolesAdded`, `OfferRoleDescriptionsChanged` (`offerId`, `roles`); query `GET /internal/offers/{offerId}` with the fields of README 11.2 |
| Subscriptions (version 0.2) | steps `notify-provider`, `send-activation-callback`; events `SubscriptionRequested`, `SubscriptionSetupStarted` (multi-instance apps and services: `offerSubscriptionId`, `offerId`, `customerCompanyId`, `providerCompanyId`, `instanceUrl`), `SubscriptionActivated`, `SubscriptionDeclined`, `SubscriptionDeactivated`, `SubscriptionInstanceUrlChanged` |
| Notifications | reacts to `ProcessStepFailed` and to the end events of README 10 |
| Connectors (version 0.2) | steps `deactivate-subscription-connectors`, `unlink-technical-user`; events `ConnectorRegistered`, `ConnectorEndpointChanged` (with `endpoint`), `ConnectorDeactivated`, `ConnectorDeleted`, all with `connectorId` and `companyId` (host company) |
| C2-M5 | service account of Process Orchestration with `execute_steps_<module>` and `query_<module>` for split-out modules |

## Open questions

1. **Technical users at the end of a subscription:** FRAMEWORK 5.8 names only `disable-instance-client`. Today the technical users of the subscription are deactivated as well. C2-M3 has no step for one subscription. To be agreed with C2-M3.
2. **URL of a single-instance app:** C2-M2 `update-instance-url` exists, but Offers has no event for it yet (C2-M2 open points).
3. **Order of `register-did` and `issue-credentials`:** the flow registers the DID first; whether C4-M1 needs the DID in the directory before issuing is not stated in C4-M1.
4. **Several workers and the event bus:** with an in-process bus, every instance of the monolith receives every event. The unique index on open processes (README 5) prevents double processes; whether one instance should be the only consumer is open (decision on the event transport of 22 Oct 2026).
5. **Retention:** how long finished processes and attempts are kept (OP-20).
6. **Live status for the UI:** Server-Sent Events for process changes were proposed on 20 Oct 2026. With one UI per OSP built at the hackathon, polling `GET /processes/{id}` is enough for now.
7. **BPN as company identifier** (raised at the workshop of 27 Oct 2026: a dataspace profile without BPDM and BPN should be possible). The flows pass `bpn` as a value only and do not depend on its format; the steps that require it belong to C3-M1, C3-M2 and C2-M2.

## Out of scope

- A workflow engine (Temporal, Camunda); the REST contract would stay the same.
- Flows across installations or with several CSPs (FRAMEWORK 1, OP-44).
- Manual steps with a person in the loop; decisions stay in the owning modules, which publish an event.
