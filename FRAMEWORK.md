# Framework for the module specs (OSP 2.0)

**As of:** 2026-11-24 (switch to OSP 2.0 on 2026-10-28; feature-first architecture with readable base paths and one database per feature on 2026-11-17, 4.9, OP-55; joining a company with a mandate on 2026-11-24, 5.3, OP-56). Binding for all module specs of this part. Modules take over what is written here and do not reinvent it. Deviations go into the module's `open-points.md`. Open questions that concern several modules are in the cross-module [open-points.md](open-points.md). The version for the PSP architecture is in the git history of this repository.

Overview graphic: [architecture-overview.png](architecture-overview.png).

## 1. Architecture in brief

- **Every OSP runs its own portal** (OSP 2.0, decided on 28 Oct 2026). An onboarding service provider (OSP) runs its own **installation** with all modules of this part, its own Process Orchestration, its own databases, its own Portal IAM, its own EBW-Connector and its own user interface (UI). It configures its portal itself. The configuration of its installation (Helm chart) decides which services it runs itself and which it uses from the CSP.
- **The CSP provides trust.** The core service provider (CSP) runs the **core services** centrally: trust list (admitted OSPs and issuers), company identifier (BPN) with home OSP, directory BPN → DID (BDRS), discovery and common rules. The OSPs feed the core services through the OSP–CSP interface. Optionally, the CSP offers further services, such as an issuer or a dataspace wallet; an OSP uses them or runs its own.
- **CSP Connection (C3-M2).** All calls of an installation to the core services go through this one module. C3-M2 describes the interface to the CSP as a proposal; if the association later defines it differently, only C3-M2 changes.
- **No central core, no delegation.** An OSP and its portal are one. PSP admission (formerly C1-M1), delegation (formerly C2-M4) and the identity providers per PSP (formerly C2-M1) are dropped.
- **Two wallets.** The EUDI Wallet of the person (phone) holds the PID and the mandate. The Business Wallet of the company (browser) holds the LPID and issues mandates to employees. The EBW-Connector (C5-M1) checks both.
- **Sign-in without a password.** People sign in at the OSP's portal with the EUDI Wallet: the OSP's Portal IAM forwards to the EBW-Connector. There is no Company IAM and there are no passwords.
- **Hackathon scenario:** one CSP and two OSPs. In the target picture, OSP A issues credentials itself and OSP B uses the CSP's issuer. At the hackathon on 6 Nov 2026 no real credentials are issued; both run on the stand-in of C4-M1 (`STUB`, OP-53). The aim for the demo at the Community Days is a real issuer. Several CSPs with federation and the connection between dataspaces are later work.
- **Modular monolith of features** in Python with FastAPI. Every module of this part is implemented as one **feature**, named after the readable part of its folder (`C3-M1-CompanyProfile` → `company_profile`) and served under `/api/<feature>` (4.1, 4.9). Process Orchestration and other modules call a feature in the same process through its public interface; the UI calls over HTTP. `openapi.yaml` describes both as the same contract. Which features run together in one pod is a choice of the installation (runtimes, 4.9), not of the spec.
- **Process Orchestration** (other part of the spec): flows are data. Every step of a flow runs through Process Orchestration; modules call each other directly only for queries. Event-driven, no polling. Long-running steps answer "running" and end with an event of their own. Modules do not know Process Orchestration; they describe their steps with preconditions, without a fixed order.
- **Own database per feature** in one PostgreSQL instance of the installation, with its own database role (4.9). No module reads or writes the database of another module. References to data of other modules only by ID.
- **Events** are CloudEvents (section 6). They go out through an outbox per module and never contain secrets.
- **Roles only in the Portal IAM** of the OSP (Keycloak, configured only, no extension). Descriptions per language as role attributes `description.de`, `description.en`. Only modules of C2 write to Keycloak.

## 2. Modules

| No. | Module | Folder | Feature (base path) | Kind | Purpose |
|---|---|---|---|---|---|
| C1-M1 | PSP Admission | `C1-M1-PSPAdmission` | – | dropped | admission of an OSP through the trust list of the CSP (governance, not a module of the installation) |
| C1-M2 | Registration | `C1-M2-Registration` | `registration` (`/api/registration`) | Hackathon | application of a company at the OSP: data, verification attestation, consents, submission; reactivation and transfer |
| C1-M3 | Application Review | `C1-M3-ApplicationReview` | `application_review` (`/api/application-review`) | Hackathon | review case per application, automatic pre-check (with duplicate check at the CSP), decision by OSP staff |
| C2-M1 | Sign-in Services | `C2-M1-SignInService` | – | dropped | the only identity provider (EBW-Connector) is configuration in C2-M5 |
| C2-M2 | Users and App Access | `C2-M2-AppAccess` | `app_access` (`/api/app-access`) | Hackathon | accounts of people without passwords, binding to a company, invitations, portal roles and app roles of people; client template and instance clients per app |
| C2-M3 | Technical Users | `C2-M3-TechnicalUser` | `technical_user` (`/api/technical-user`) | Hackathon | technical users of companies (client with service account), wallet access as a property |
| C2-M4 | Delegation | `C2-M4-Delegation` | – | dropped | an OSP and its portal are one |
| C2-M5 | Portal IAM | `C2-M5-PortalIAM` | – (Keycloak paths) | Configuration | realm of the OSP, identity provider EBW-Connector, client `portal-core`, permissions and portal roles, token contents |
| C3-M1 | Company Profile | `C3-M1-CompanyProfile` | `company_profile` (`/api/company-profile`) | Hackathon | record of the company in the installation, company identifier (BPN) from the CSP, company roles, status |
| C3-M2 | CSP Connection | `C3-M2-CSPConnection` | `csp_connection` (`/api/csp-connection`) | Hackathon (new) | register companies with the CSP (BPN, home OSP), duplicate check, membership, directory BPN → DID, connectors in the discovery, notifications from the CSP |
| C4-M1 | Credentials | `C4-M1-Credentials` | `credentials` (`/api/credentials`) | Hackathon | trigger, track and revoke BPN and membership credentials at the issuer; own issuer or the CSP's |
| C4-M2 | Dataspace Wallet Interface | `C4-M2-DataspaceWalletInterface` | `dataspace_wallet_interface` (`/api/dataspace-wallet-interface`) | Hackathon | vendor-neutral: dataspace wallet per company, wallet accounts for technical users; adapter per provider, also for the CSP's dataspace wallet |
| C5-M1 | EBW-Connector | `C5-M1-EBWConnector` | – (paths of the standards) | Provided | checks EUDI Wallet and Business Wallet via OpenID4VP; OpenID Connect for Portal IAM, UI and Business Wallet; verification attestation for the registration |

Modules of the other part, referred to by name only: **Process Orchestration**, **Offers** (apps and services), **Subscriptions**, **Connectors**, **Notifications**, **MCP server**. Their features: Connectors `connector` (`/api/connector`, C6-M1), Offers `offers` (`/api/offers`, C8-M1), Subscriptions `subscriptions` (`/api/subscriptions`, C10-M1), Notifications `notifications` (`/api/notifications`, C11-M1), Process Orchestration `process_orchestration` (`/api/process-orchestration`, C12-M1). The **UI** of the OSP is built at the hackathon; it is not a module of this part.

External systems: EUDI Wallet and Business Wallet (through C5-M1), core services of the CSP (through C3-M2), issuers (own IssuerService or the CSP's, C4-M1), dataspace wallets (SAP DIV, IdentityHub, the CSP's dataspace wallet; C4-M2).

## 3. Structure of a spec

One folder per module in this repository, with:

- `README.md` – behaviour, in English. Together with `openapi.yaml`, it is enough to build the module. **No open points in the README**; where something is not agreed yet, the README sets a provisional decision and `open-points.md` names it.
- `openapi.yaml` – interface, OpenAPI 3.1; descriptions and identifiers (paths, fields, operationId) in English.
- `open-points.md` – provisional decisions (table: No., Topic, Decision in the README, To be agreed with), open questions, out of scope.

A dropped module keeps its folder with a short README: dropped since when, why, and where its tasks went.

### Outline of the README

```
# <No.> <Name>

**Version:** 0.2, as of 2026-10-28
**Capability:** <C… name>
**Kind:** <Hackathon | Provided | Configuration>
**Files:** this README (behaviour), [openapi.yaml](openapi.yaml) (interface), [open-points.md](open-points.md)

## 1. Purpose            (with a table of callers: Who | What for)
## 2. Terms
## 3. Not part of this module   (What | Where)
## 4. Technical frame  (first bullet: feature name and base path, 4.9; incl. ### Configuration as a table Variable | Meaning | Example)
## 5. Data              (tables of the feature's own database: Field | Type | Rule; what is deliberately not stored)
## 6. Rules             (status with transitions, permissions, repeatability)
## 7. Operations        (per path: step by step, checks, responses, errors)
## 8. Steps for Process Orchestration   (if any: input, preconditions, behaviour, result, event)
## 9. Queries for other modules          (if any)
## 10. Events         (Event | When | Content)
## 11. Calls to other modules and external systems   (with built-in stand-in, so that the module can be tested alone)
## 12. Errors             (type | HTTP | When)
## 13. Acceptance            (numbered, testable cases)
## 14. Test data
```

Sections that do not fit a module are dropped with a one-sentence reason; the numbering stays continuous. At the end there is a section **"Changes from version 0.1"**.

## 4. Conventions

### 4.1 Paths, formats

- Base per module: `/api/<feature>`, the feature name of the module (section 2, 4.9) with `-` instead of `_`, e.g. `/api/registration` for C1-M2 and `/api/company-profile` for C3-M1 (decided on 17 Nov 2026, OP-55; replaces the provisional `/api/<module>`). The `servers` entry of every `openapi.yaml` carries this base. The former base `/api/<module>` is not served (404). Exceptions: C5-M1 (standards) and C2-M5 (Keycloak).
- **Identifiers keep the module identifier.** Only the address uses the feature name. Problem types (`urn:<module>:problem:<name>`, 4.5), the event `source` (`urn:<module>`, 6), permissions (`execute_steps_<module>`, `query_<module>`, 4.2), service accounts (`svc-<module>`), the keys of `MODULE_URLS` and the references in flows (`<module>.<step>`, C12-M1) stay as they are.
- Resources in English, lower case with hyphens: `/applications/{id}/submit`.
- JSON fields in camelCase, times in ISO 8601 UTC, IDs as UUID.
- Lists with `page` (from 0) and `size` (1 to 100, default 20), response `{ "items", "page", "size", "total" }`.
- Paths under `/steps/...` are steps for Process Orchestration, paths under `/internal/...` are queries for other modules. Both are called in the same process; the HTTP form applies when a module is split out. Callbacks from external systems have their own paths, which the module names (such as `/csp-notifications`, `/issuer-callbacks/...`).
- Paths ending in `/events` are Server-Sent Events streams for the UI (`text/event-stream`, only from the server to the browser). The server part of the UI opens them with the person's token in the header `Authorization: Bearer`, never in the query string (C11-M1 7.6, C12-M1 7.6).
- OpenAPI extensions: `x-permission` (required permission) and `x-permission-scope` (`company`, `osp` or `own`, see 4.3).
- Tags: `Company` (calls by people of a company and by applicants in scope `own`), `OSP` (tasks of OSP staff), `Queries`, `Steps`; callbacks from external systems under one tag per system (such as `CSP`, `Issuer`, `Provider`).

### 4.2 Callers, tokens, permissions

- **People** call through the UI of the OSP, with their token from the OSP's Portal IAM. The shared token validation library (provided) checks the signature (JWKS), expiry and audience (`aud` contains `portal-core`) and returns the claims.
- **Token of a person** (defined in C2-M5): `sub` (identifier of the person in the Portal IAM), `company_id` and `bpn` (only if the account is bound to a company), `given_name`, `family_name`, the permissions under `resource_access.portal-core.roles`.
- **Permissions** are client roles on the client `portal-core` in the Portal IAM, named in English in snake_case, in the form `<verb>_<object>`, e.g. `register_company`, `review_application`. Each module lists its permissions in section 6, with their scope (4.3).
- **Portal roles** are composite roles on the client `portal-core` (defined in C2-M5) and bundle permissions. For people of a company: `Company Admin`, `IT Admin`, `Member`. For OSP staff: `OSP Reviewer`, `OSP Admin`. Every new account has the portal role `Applicant` (submit applications only, 5.2).
- **Technical users** of companies (connectors, programs) and **app users** (people at apps) do not call the modules; they get tokens only for apps and central services.
- **Process Orchestration and other modules** call in the same process without a token. For calls over HTTP, the permissions `execute_steps_<module>` and `query_<module>` apply, with the full module identifier in lower case (e.g. `execute_steps_c1_m2`, `query_c6_m1`).
- **Callbacks from external systems** (issuer, dataspace wallet, CSP): one service account per system in the Portal IAM with exactly one permission; authentication with `private_key_jwt` where the system supports it, otherwise a client secret as a transition (decided on 26 Oct 2026, OP-22).

### 4.3 Company reference instead of delegation

Every permission has a scope, in the README (section 6) and in `openapi.yaml` (`x-permission-scope`):

| Scope | Meaning | Check in the module |
|---|---|---|
| `company` | applies only to the person's company | `company_id` in the token equals the company of the object; otherwise 404 (the object stays invisible) |
| `osp` | applies to all companies of the installation: tasks of OSP staff, steps, queries and callbacks from external systems | the permission only |
| `own` | applies to the person's own objects, such as their own application or their own notifications, with or without a company | `sub` in the token equals the person who owns the object. Whether a binding to a company is allowed is up to the module (C1-M2: applications only without a company) |

The permissions `execute_steps_<module>` and `query_<module>` (4.2) and the permissions of the service accounts for callbacks also have the scope `osp`; every operation in `openapi.yaml` carries `x-permission-scope`.

**Audit log:** Every changing operation writes an audit log entry to the module's own database: time, `sub` of the acting person, `company_id` (if present), operation, object ID. No secrets, no names. The former header `X-Acting-Person` is dropped; the person is in the token.

### 4.4 Steps

- Input always with `processId` and `stepRunId` (only for the audit log and traceability).
- Every step is idempotent: it finds its object by a business key from the input, creates only what is missing and returns the same result. Events only if something changed.
- Result `{ "stepRunId", "state": "DONE" | "RUNNING", "result": {…} }`. `RUNNING` only for long-running steps; they end with the event named in the step. Errors as Problem Details with the field `retryable` (`true`: temporary; `false`: final). Retrying and notifying OSP staff are up to Process Orchestration (proposal to the other part, OP-18).
- Name the **preconditions** of each step explicitly (the state that must hold); no fixed order in the module.
- Results never contain secrets.

### 4.5 Errors

Problem Details (RFC 9457), `type` = `urn:<module>:problem:<name>`. Common names: `validation` (400), `unauthorized` (401), `forbidden` (403, permission missing), `not-found` (404, also for another company), `invalid-state` (409), `conflict` (409), `upstream-error` (502). Further names per module. Exception: C5-M1 answers according to its standards.

### 4.6 Secrets and data protection

- Secrets never in logs, events, step results or unencrypted in the database. Access credentials can be fetched only once.
- Personal data only where needed; from the wallets only the required extract.
- Apps receive from a person only the pseudonymous identifier, company, BPN and app roles, no name and no email address (decided on 26 Oct 2026).

### 4.7 Stand-ins for neighbours

Every module runs and can be tested on its own. The configuration variable for the address of a neighbour has three values: empty = built-in stand-in (described in section 11), `in-process` = call of the real module in the same process, otherwise the HTTP address of the split-out neighbour. For the core services of the CSP, C3-M2 has a built-in stand-in; for the hackathon, the test tools provide a shared CSP stand-in for several installations.

### 4.8 Installation settings

Every installation has a file with common settings that every module reads at start-up (proposal, OP-36):

| Setting | Meaning | Example |
|---|---|---|
| `ospId` | identifier of the OSP at the CSP (from the trust list) | `22222222-2222-4222-8222-222222222222` |
| `ospName` | display name of the OSP | `OSP A` |
| `ospCompanyId` | `companyId` of the company that owns the OSP (in its own installation) | `11111111-1111-4111-8111-111111111111` |
| `dataspace.memberOf`, `dataspace.name` | dataspace of the installation (OP-14) | `Catena-X` |
| `masterDataFile` | common reference data: countries, identifier types (OP-13) | `config/master-data.json` |
| `ospCompanyFile` | seed data of the OSP company (C3-M1, C3-M2): JSON object with `companyId` (equal to `ospCompanyId`), `name`, `bpn`, `address` (`street`, `houseNumber`, `postalCode`, `city`, `countryCode`), `identifiers` (each `type`, `value`) and `companyRoles` | `config/osp-company.json` |

Relative paths in the settings count from the settings file.

### 4.9 Features, runtimes and databases

Decided on 17 Nov 2026 (OP-55), with all modules of the backend migrated. The reference implementation is `dataspace-mgm-services` ([ADR-0003](https://github.com/eclipse-tractusx/dataspace-mgm-services/blob/main/docs/architecture/decisions/ADR-0003-feature-first-layout-and-runtimes.md)).

**Feature.** Every module with an interface is implemented as one feature: one package that one team owns, reviews, tests and can deploy alone. The spec stays the contract per module; the feature is how it is built and addressed.

| Module | Feature | Base path |
|---|---|---|
| C1-M2 Registration | `registration` | `/api/registration` |
| C1-M3 Application Review | `application_review` | `/api/application-review` |
| C2-M2 Users and App Access | `app_access` | `/api/app-access` |
| C2-M3 Technical Users | `technical_user` | `/api/technical-user` |
| C3-M1 Company Profile | `company_profile` | `/api/company-profile` |
| C3-M2 CSP Connection | `csp_connection` | `/api/csp-connection` |
| C4-M1 Credentials | `credentials` | `/api/credentials` |
| C4-M2 Dataspace Wallet Interface | `dataspace_wallet_interface` | `/api/dataspace-wallet-interface` |
| C6-M1 Connectors | `connector` | `/api/connector` |
| C8-M1 Offers | `offers` | `/api/offers` |
| C10-M1 Subscriptions | `subscriptions` | `/api/subscriptions` |
| C11-M1 Notifications | `notifications` | `/api/notifications` |
| C12-M1 Process Orchestration | `process_orchestration` | `/api/process-orchestration` |

- **Name:** the readable part of the module folder in snake case (`C2-M3-TechnicalUser` → `technical_user`), never a plural that the folder does not have and never a team name. The base path uses the same name with `-`. A new module gets its feature name from its folder in the same way.
- **Shape of a feature** (binding for the implementation, so code is found in the same place in every feature):

  | Part | Holds |
  |---|---|
  | index | name, module identifier, lifecycle (start, reset, receiving events) |
  | router | the controllers mounted under the base path; every operation is checked against the module's `openapi.yaml` and gets its `x-permission` from there |
  | `interface` | the in-process contract for Process Orchestration and other modules: the steps (section 8 of the README) and queries (section 9) |
  | `controllers/` | HTTP only, one folder per path segment of the API; checks the scope (4.3), calls one service, no business logic |
  | `services/` | business logic, without HTTP, so Process Orchestration and other modules can call it in-process |
  | `managers/` | state and adapters: repositories, storage, clients of neighbours and of external systems, configuration, stand-ins (4.7) |
  | `models/` | request bodies, step inputs, responses and value sets with the field names of `openapi.yaml`; the tables of the database |
  | runtime | the feature alone as its own service |

  Dependencies only go downwards: controllers → services → managers; models are used by all. A feature calls another module only through that module's interface (in-process) or its HTTP paths (split out, 4.7), never through its code, its models or its database. Code that several features need is shared code of the installation, not part of one feature.
- **Runtimes.** A runtime is a set of features that run together in one process: the whole installation (the modular monolith, default), a group (for example onboarding: `registration` and `company_profile`), or one feature alone. One image serves all of them; the installation chooses the runtime per pod. In a runtime, calls to its own features are `in-process`; every other module is called over HTTP with the token of the calling service account (4.2, 4.7), and events go through the event inbox (6). Moving a feature to another runtime changes configuration, not the spec.
- **Databases.** One PostgreSQL instance per installation. Inside it, one database per feature, named after the feature (`company_profile`), owned by a database role of the same name that can connect only to its own database. Databases, not schemas: references to other modules stay IDs, enforced by the server.
  - The feature reads its address from `<FEATURE>_DATABASE_URL` (`COMPANY_PROFILE_DATABASE_URL`). `DATABASE_URL` alone is accepted only by a runtime that serves exactly this one feature; a runtime with several features and only `DATABASE_URL` does not start, because its features would share one database.
  - Without an address, a feature runs on a private in-memory database, so it can be tested alone (as the stand-ins in 4.7).
  - Each feature owns its tables, its migrations and the version table in its own database and applies them at start with its own role. The credentials are one secret per feature; a pod receives only the secrets of the features it runs.
  - The table names in section 5 of a README are the tables of this database. The audit log (4.3) and the outbox (6) are tables of the same database.

**Configuration variables** (section 4 of each module README) come from one shared configuration of the installation. A name therefore means the same in every module that uses it. Where two modules need different values, their variables have different names (for example `INSTANCE_CLIENT_ID_PREFIX` in C2-M2 and `TECHNICAL_USER_CLIENT_ID_PREFIX` in C2-M3).

**Values that are not set:** Where the module README says what a missing or empty value means, that applies. Otherwise the value in the column "Example" is the default, so a module starts without any setting and the installation sets only what differs. Exceptions are addresses of other systems, secrets and keys: the installation must set them, and without them the module answers with an error that names the missing variable.

## 5. Flows across modules

Process Orchestration defines these flows; the modules provide the named steps and report the named events. All flows run within one installation; only C3-M2 talks to the CSP.

### 5.1 Admission of an OSP

No flow in the installation. The governance of the dataspace admits an OSP, and the CSP enters it in the trust list: `ospId`, name, key of the CSP Connection (JWKS), issuer key (if the OSP issues credentials itself), issuer of the Portal IAM (for central services, OP-37). The OSP enters `ospId` in the settings of its installation (4.8) and its private key in the configuration of C3-M2 (`OSP_SIGNING_KEY_FILE`).

### 5.2 Onboarding of a company at the OSP

**Two ways** (OP-35, still valid): **with Business Wallet** as below; **without Business Wallet** with supporting documents and a manual review by OSP staff (C1-M2, C1-M3). The OSP can switch off the way without Business Wallet by configuration. From step 4, both ways are the same.

1. **Sign-in.** The person signs in at the OSP's portal: the Portal IAM forwards to the EBW-Connector (scopes `openid profile`; PID from the EUDI Wallet). At the first login, an account without a company and with the portal role `Applicant` is created (C2-M5).
2. **Verification attestation.** "Register company": the server part of the UI (backend for frontend) fetches the verification attestation from the EBW-Connector, as an OIDC client of its own (client ID as in 7a; authorization code with PKCE and `private_key_jwt` with a key of the UI, C5-M1 6.8; a browser application alone cannot hold this key). Scopes `openid profile representation company attestation`: PID and mandate from the EUDI Wallet, LPID from the Business Wallet. The mandate is required. The attestation comes in the token response as `ebw_attestation`. It has the purpose `onboarding`; C1-M2 refuses an attestation with the purpose `representation` (5.3).
3. **Application.** The person creates the application (C1-M2, permission `register_company`, scope `own`): the UI sends company data, verification attestation and requested company roles with the person's token from the Portal IAM. C1-M2 assigns the future `companyId` and checks the verification attestation against the EBW-Connector of the installation. → `ApplicationCreated`
4. **Submission** with the consents (terms of the OSP and of the dataspace). → `ApplicationSubmitted`
   - Flow: C3-M1 `create-company` (status `REQUESTED`) → `CompanyCreated`, only without an existing company (`existingCompany` `false`); C1-M3 `open-review`. The automatic pre-check asks C3-M2 `find-companies`: is the company (EBWOID, EUID, name and country) already registered with an OSP, here or with another one?
5. **Decision** by OSP staff (C1-M3, portal role `OSP Reviewer`). → `ApplicationApproved` or `ApplicationDeclined` (then C1-M2 `close-application`, C3-M1 `reject-company`). Until the decision, the person can withdraw the application (C1-M2) → `ApplicationWithdrawn` → C1-M3 `close-review`, C3-M1 `reject-company`. `reject-company` only without an existing company. If C1-M3 has already approved, `close-review` answers with 409; Process Orchestration stops and notifies OSP staff.
   - Four-eyes principle (OP-16): depending on the setting, the approval needs a second person (C1-M3, status `AWAITING_SECOND_APPROVAL`). Nobody decides on an application they submitted themselves.
6. **Flow after `ApplicationApproved`:**
   - C3-M2 `register-company` (long-running): the CSP assigns or confirms the BPN and enters the OSP as home OSP. → `CompanyRegisteredAtCsp` (`companyId`, `bpn`) or `CompanyRegistrationAtCspFailed`
   - C3-M1 `assign-identifier` (with the BPN) → `CompanyIdentifierAssigned`
   - C4-M2 `create-wallet` (long-running; own provider or the CSP's dataspace wallet) → `DataspaceWalletCreated` (`companyId`, `did`)
   - C3-M2 `register-did` (directory BPN → DID) → `CompanyDidRegistered`
   - C4-M1 `issue-credentials` (long-running; own issuer or the CSP's) → `CredentialsIssued`
   - C3-M2 `register-membership` → `MembershipRegistered`
   - C2-M2 `assign-company-admin`: the applicant's account is bound to the company and gets the portal role `Company Admin`. → `UserAssignedToCompany`
   - C3-M1 `activate-company` → `CompanyActivated`
   - C1-M2 `complete-application` → `ApplicationCompleted`

### 5.3 People of a company

- **Invite:** The Company Admin invites a person (C2-M2, scope `company`): given name, family name, portal roles. C2-M2 creates an invitation code (valid once, time-limited), which the UI passes on as a link.
- **Accept:** The invited person signs in (the account is created as in 5.2 step 1) and redeems the code (C2-M2, scope `own`). C2-M2 binds the account to the company and assigns the portal roles. → `UserAssignedToCompany`
- **Accept with a mandate:** In place of the code the person shows a mandate of the company from their EUDI Wallet. The company's Business Wallet issued it and put its LPID into it (C5-M1 6.5 rule 4). The server part of the UI fetches a verification attestation with the purpose `representation` from the EBW-Connector (scopes `openid profile representation attestation`; the Business Wallet takes no part) and passes it to C2-M2 in place of the code (C2-M2 6.8a). The invitation with its portal roles is still required. → `UserAssignedToCompany`
- **Remove, change roles:** Company Admin (C2-M2). → `UserRemovedFromCompany`, `UserRolesChanged`
- An account is bound to at most one company.

### 5.4 Deactivation of a company

OSP staff deactivate a company (C3-M1, `OSP Admin`), or the company moves to another OSP (5.6). → `CompanyDeactivated` →
- C2-M2 `disable-company-users`: the accounts stay active, but for the duration of the deactivation they lose the company (`company_id`, `bpn`), the portal roles in the company and the app roles; C2-M2 remembers them. Apps see the people without a company and without app roles. The people can still sign in and, with a mandate, apply for reactivation (5.5; proposal, decided provisionally on 28 Oct 2026);
- C2-M3 `disable-company-technical-users`: technical users are deactivated, not deleted;
- C4-M1 `revoke-credentials`: only the membership credential; the BPN credential stays;
- C3-M2 `deregister-membership`: deregister the membership at the CSP.

The dataspace wallet remains; no new wallet accounts are created for a deactivated company (OP-07, still valid).

### 5.5 Reactivation of a company

1. A person with a mandate submits an application of kind `REACTIVATION` in C1-M2 for the existing `companyId`, with a verification attestation or supporting documents. → `ApplicationSubmitted`
2. C1-M3 `open-review`; the duplicate check recognises the installation's own deactivated company. OSP staff approve. → `ApplicationApproved`
3. C3-M1 `reactivate-company` → `CompanyReactivated` → C2-M2 `enable-company-users` (company and remembered roles restored), then C2-M2 `assign-company-admin` with the applicant (`applicantSub`); C2-M3 `enable-company-technical-users`, C4-M1 `issue-credentials` (membership credential), C3-M2 `register-membership`.

### 5.6 Transfer to another OSP (proposal, OP-38)

1. At the new OSP, a person submits an application of kind `TRANSFER` (C1-M2), with a verification attestation. The duplicate check (C3-M2 `find-companies`) finds the company with another home OSP; for `TRANSFER` this is expected.
2. After the approval: C3-M2 `transfer-company` (long-running): the CSP changes the home OSP; the BPN stays. → `CompanyRegisteredAtCsp`. Then as in 5.2 step 6, from `assign-identifier`.
3. At the old OSP, the CSP reports the transfer to C3-M2 (callback). → `CompanyTransferredAway` → C3-M1 `deactivate-company` (reason `TRANSFERRED`) → flow 5.4.

Data, technical users and the dataspace wallet do not move along; the company sets them up again at the new OSP (later work).

**Return** to an OSP the company moved away from earlier: application of kind `TRANSFER` referring to the existing `companyId` (`existingCompany` `true`). After the approval C3-M2 `transfer-company`, then C3-M1 `reactivate-company` instead of `create-company` and `activate-company`; then as in 5.5 step 3.

### 5.6a Core data changed

C3-M1 changes core data (with a new verification attestation or after the approval of a change request) → `CompanyDataChanged` with `coreDataChanged` → C3-M2 `update-company` → `CompanyUpdatedAtCsp`.

### 5.7 Technical user with wallet access

An IT Admin creates a technical user (C2-M3, scope `company`). With a wallet role → `WalletAccessRequested` → flow: C4-M2 `create-wallet-account` (long-running, ends with `WalletAccountCreated` or `WalletAccountFailed`) → C2-M3 `apply-wallet-access`. The person fetches the access credentials of the wallet account once from C4-M2 (scope `company`). Deletion works the same way with `WalletAccessRemovalRequested` → C4-M2 `delete-wallet-account` → `WalletAccountDeleted` → C2-M3 `delete`.

### 5.8 App approval, app subscription, app users

- App approved (Offers) → C2-M2 `create-client-template`; for a single-instance app also `create-instance-client`, `enable-instance-client`.
- Subscription to a multi-instance app activated (Subscriptions) → C2-M2 `create-instance-client`, C2-M3 `create-for-subscription`, wallet access if needed, C2-M2 `enable-instance-client`.
- The Company Admin assigns app roles to people of their company (C2-M2, scope `company`). App users are the same accounts as for the portal.
- Sign-in at an app: the app redirects to the OSP's Portal IAM, which forwards to the EBW-Connector. The token contains `company_id`, `bpn` and the app roles.
- Apps across the boundaries of an OSP (marketplace) are not in focus (OP-39).

In detail, as in the flow definitions of C12-M1 ([flows.yaml](C12-M1-ProcessOrchestration/flows.yaml)):

- Roles of an active app added or their descriptions changed (C8-M1 `OfferRolesAdded`, `OfferRoleDescriptionsChanged`) → C2-M2 `add-app-roles` or `update-role-descriptions`.
- A company subscribes to an offer (C10-M1 `SubscriptionRequested`) → C10-M1 `notify-provider`: the system of the provider is informed, if the provider has configured one.
- Setup of a multi-instance app or a service: the provider starts it (C10-M1 `SubscriptionSetupStarted`) → C2-M2 `create-instance-client` (apps only), C2-M3 `create-for-subscription` (only if the offer has a technical user profile). After the provider has activated the subscription (`SubscriptionActivated`): C2-M2 `enable-instance-client` (apps only), C10-M1 `send-activation-callback`. The flow ends early on `SubscriptionDeclined` or `SubscriptionDeactivated`.
- Single-instance app activated (C10-M1 `SubscriptionActivated` with `instanceKind` `SINGLE`) → C2-M3 `create-for-subscription` (only if the offer has a technical user profile), C10-M1 `send-activation-callback`.
- Subscription declined by the provider or ended by the customer (C10-M1 `SubscriptionDeclined`, `SubscriptionDeactivated`) → C2-M2 `disable-instance-client` (only if an instance client exists), C2-M3 `disable-subscription-technical-users`, C6-M1 `deactivate-subscription-connectors` → `ConnectorDeactivated` (then 5.9).
- Instance URL of a multi-instance app changed (C10-M1 `SubscriptionInstanceUrlChanged`) → C2-M2 `update-instance-url`.
- Technical user deleted (C2-M3 `TechnicalUserDeleted`) → C6-M1 `unlink-technical-user`: the connector that used the account stays `ACTIVE` without it.
- Notifications (C11-M1) has no flows and no steps. It reacts to the events of section 6 with fixed rules (C11-M1 6.2). Process Orchestration (C12-M1) reports its own events about processes (section 6).

### 5.9 Connectors in the discovery

A company creates a connector, changes its address or deletes it (Connectors, other part) → C3-M2 `register-connector` or `deregister-connector` → `ConnectorRegisteredAtCsp` or `ConnectorDeregisteredAtCsp`. In its discovery, the CSP names only connectors of companies with an active membership; deactivation and reactivation (5.4, 5.5) therefore need no own steps.

## 6. Event catalogue

Names in English in PascalCase. Content only IDs and business values, never secrets.

Every event is a CloudEvent (CloudEvents 1.0, JSON event format, structured; proposal, OP-54):

| Attribute | Value |
|---|---|
| `specversion` | `1.0` |
| `id` | UUID, new for every event (replaces `eventId`). Consumers detect a repeated delivery by `id`. |
| `source` | the publishing module: `urn:<module>`, module identifier in lower case, e.g. `urn:c1-m2` (same identifier as the Problem Details `type`, 4.5) |
| `type` | the event name from the catalogue, PascalCase, e.g. `ApplicationSubmitted` (replaces `eventType`; no reverse-DNS prefix: events stay in the installation) |
| `time` | time of the change, ISO 8601 UTC (replaces `occurredAt`) |
| `datacontenttype` | `application/json` |
| `data` | the content of the event (column "Content (at least)" of the catalogue below, column "Content" of the table in section 10 of the module README): JSON object, only IDs and business values, never secrets |

- No other attributes, no extensions; `subject` and `dataschema` are not used in this version.
- The attribute names are those of CloudEvents (lower case). The fields in `data` follow 4.1 (camelCase).
- In the installation the event bus delivers this JSON object; the outbox of a module stores it unchanged.
- When an event leaves the process (split-out module, broker), a CloudEvents protocol binding in structured content mode applies; over HTTP with `Content-Type: application/cloudevents+json`. Which broker is used stays open (C12-M1 open question 4).
- **Event inbox of the installation** (base skeleton, not a module; proposal, OP-54): an installation receives the events of a split-out module over HTTP at `POST /api/v1/events/inbox`, in structured content mode or in binary content mode (attributes in `ce-` headers, `data` as the body with `Content-Type: application/json`). The token is that of the service account of the publishing module (`svc-<module>`, 4.2); `source` must be `urn:<module>` of that account, and `type` must be an event of that module (catalogue below, section 10 of its README). The answer is 202: the event goes to the event bus unchanged (same `id` and `time`) and is delivered like an event published in the process. The same `source` and `id` a second time: 202, delivered once. Errors as Problem Details with `type` `urn:events:problem:<name>`: `validation` (400, an attribute is missing or wrong), `unauthorized` (401), `forbidden` (403, another service account, another `source` or an event of another module), `payload-too-large` (413, larger than the limit of the installation, default 64 KiB), `unsupported-media-type` (415, any other content type, also batched content mode).
- Field names in flows (C12-M1 6.1: `trigger.when`, `subject.key`, `correlateBy`, process variables, the step result from `completesOn`) and in the rules of Notifications (C11-M1 6.2) refer to fields of `data`.
- The Server-Sent Events stream messages for the UI (`process-changed`, `notification-created`, ...; 4.1) are not events of the bus and not CloudEvents.

Example:

```json
{ "specversion": "1.0", "id": "b8e1f7a0-1c2d-4e3f-9a8b-7c6d5e4f3a2b", "source": "urn:c1-m2",
  "type": "ApplicationSubmitted", "time": "2026-11-06T09:30:00Z", "datacontenttype": "application/json",
  "data": { "applicationId": "0f8e2d4c-6b1a-4c3e-8f7d-2a9b1c0d3e4f", "companyId": "c0000001-0000-4000-8000-000000000001",
            "kind": "ONBOARDING", "existingCompany": false, "applicantSub": "5a1c0001-0000-4000-8000-000000000001" } }
```

The column "Content (at least)" names the fields of `data`.

| Module | Event | Content (at least) |
|---|---|---|
| C1-M2 | `ApplicationCreated`, `ApplicationWithdrawn` | `applicationId`, `companyId`, `kind` (`ONBOARDING`, `REACTIVATION`, `TRANSFER`), `existingCompany` |
| C1-M2 | `ApplicationSubmitted` | as `ApplicationCreated`, plus `applicantSub` (for C2-M2 `assign-company-admin`) |
| C1-M2 | `ApplicationCompleted` | `applicationId`, `companyId`, `kind` |
| C1-M2 | `AgreementVersionActivated` | `agreementKey`, `version`, `party`, `reconsentRequired`, `reconsentDueAt` |
| C1-M3 | `ApplicationApproved` | `applicationId`, `companyId`, `kind`, `existingCompany`; `bpn` for `TRANSFER` and `REACTIVATION` |
| C1-M3 | `ApplicationDeclined` | `applicationId`, `companyId`, `kind`, `existingCompany`, `reason` |
| C2-M2 | `AppClientTemplateCreated`, `AppInstanceClientCreated`, `AppInstanceClientEnabled`, `AppInstanceClientDisabled`, `AppInstanceUrlUpdated`, `AppRolesAdded`, `AppRoleDescriptionsUpdated` | as before in C2-M2 |
| C2-M2 | `UserAssignedToCompany` | `userId`, `companyId`, `via` (`ONBOARDING`, `INVITATION`), `portalRoles` |
| C2-M2 | `UserRemovedFromCompany` | `userId`, `companyId`, `reason` |
| C2-M2 | `UserRolesChanged` | `userId`, `companyId`, `appId` (only for app roles), `addedRoles`, `removedRoles` |
| C2-M2 | `CompanyUsersDisabled`, `CompanyUsersEnabled` | `companyId`, `userIds` |
| C2-M2 | `OspStaffRolesChanged` | `userId`, `addedRoles`, `removedRoles` |
| C2-M3 | `TechnicalUserCreated`, `TechnicalUserDeleted` | `technicalUserId`, `companyId` |
| C2-M3 | `WalletAccessRequested`, `WalletAccessRemovalRequested` | `technicalUserId`, `companyId` |
| C3-M1 | `CompanyCreated` | `companyId`, `applicationId`, `coreDataSource` |
| C3-M1 | `CompanyActivated`, `CompanyReactivated` | `companyId`, `bpn`, `companyRoles` |
| C3-M1 | `CompanyRejected` | `companyId` |
| C3-M1 | `CompanyIdentifierAssigned` | `companyId`, `bpn` |
| C3-M1 | `CompanyDataChanged` | `companyId`, `changedFields` (field names without values), `coreDataChanged`, `coreDataSource` |
| C3-M1 | `CompanyChangeRequestSubmitted` | `companyId`, `changeRequestId` |
| C3-M1 | `CompanyChangeRequestClosed` | `companyId`, `changeRequestId`, `outcome` |
| C3-M1 | `CompanyDeactivated` | `companyId`, `reason` (`MANUAL`, `TRANSFERRED`) |
| C3-M1 | `CompanyRolesChanged` | `companyId`, `companyRoles` |
| C3-M2 | `CompanyRegisteredAtCsp` | `companyId`, `bpn` |
| C3-M2 | `CompanyRegistrationAtCspFailed` | `companyId`, `reason` |
| C3-M2 | `CompanyDidRegistered` | `companyId`, `bpn`, `did` |
| C3-M2 | `CompanyUpdatedAtCsp` | `companyId`, `bpn` |
| C3-M2 | `MembershipRegistered`, `MembershipDeregistered` | `companyId`, `bpn` |
| C3-M2 | `CompanyTransferredAway` | `companyId`, `bpn` |
| C3-M2 | `ConnectorRegisteredAtCsp`, `ConnectorDeregisteredAtCsp` | `companyId`, `bpn`, `connectorId` |
| C4-M1 | `CredentialsIssued` | `companyId`, `credentialTypes` |
| C4-M1 | `CredentialIssuanceFailed`, `CredentialRevoked` | `companyId`, `credentialType` |
| C4-M2 | `DataspaceWalletCreated` | `companyId`, `did` |
| C4-M2 | `DataspaceWalletFailed` | `companyId` |
| C4-M2 | `WalletAccountCreated`, `WalletAccountFailed`, `WalletAccountDeleted`, `WalletAccountDeletionFailed` | `technicalUserId`, `companyId`, `walletAccountRef` |
| C6-M1 | `ConnectorRegistered` | `connectorId`, `companyId` (host company), `providerCompanyId`, `type`, `endpoint`, `location` |
| C6-M1 | `ConnectorEndpointChanged` | `connectorId`, `companyId`, `endpoint` |
| C6-M1 | `ConnectorDeactivated`, `ConnectorDeleted` | `connectorId`, `companyId` |
| C8-M1 | `OfferSubmitted`, `OfferApproved`, `OfferDeactivated` | `offerId`, `offerType`, `providerCompanyId` |
| C8-M1 | `OfferDeclined` | `offerId`, `offerType`, `providerCompanyId`, `reason` |
| C8-M1 | `OfferRolesAdded`, `OfferRoleDescriptionsChanged` | `offerId`, `roles` |
| C10-M1 | `SubscriptionRequested` | `offerSubscriptionId`, `offerId`, `offerType`, `customerCompanyId`, `providerCompanyId` |
| C10-M1 | `SubscriptionSetupStarted` | as `SubscriptionRequested`, plus `instanceUrl` (apps) |
| C10-M1 | `SubscriptionActivated` | `offerSubscriptionId`, `offerId`, `offerType`, `instanceKind`, `customerCompanyId`, `providerCompanyId` |
| C10-M1 | `SubscriptionDeclined`, `SubscriptionDeactivated` | `offerSubscriptionId`, `offerId`, `customerCompanyId`, `providerCompanyId` |
| C10-M1 | `SubscriptionInstanceUrlChanged` | `offerSubscriptionId`, `offerId`, `instanceUrl` |
| C12-M1 | `ProcessStarted`, `ProcessCompleted`, `ProcessEnded` | `processId`, `flowKey`, `subjectType`, `subjectId` |
| C12-M1 | `ProcessStepFailed` | `processId`, `flowKey`, `subjectType`, `subjectId`, `stepKey`, `errorType`, `retryable` |
| C12-M1 | `ProcessStepRetriggered` | `processId`, `stepKey`, `retriggeredBy` |

## 7. Step catalogue

| Module | Step | Flow |
|---|---|---|
| C1-M2 | `complete-application`, `close-application` | Onboarding, reactivation, transfer |
| C1-M3 | `open-review`, `close-review` | Onboarding, reactivation, transfer |
| C2-M2 | `assign-company-admin` | Onboarding, transfer, reactivation |
| C2-M2 | `disable-company-users`, `enable-company-users` | Deactivation, reactivation |
| C2-M2 | `create-client-template`, `create-instance-client`, `enable-instance-client`, `disable-instance-client`, `update-instance-url`, `add-app-roles`, `update-role-descriptions` | App approval, app subscription |
| C2-M3 | `create-for-subscription`, `disable-subscription-technical-users`, `apply-wallet-access`, `delete` | Subscription, wallet access |
| C2-M3 | `disable-company-technical-users`, `enable-company-technical-users` | Deactivation, reactivation |
| C3-M1 | `create-company`, `assign-identifier`, `activate-company`, `reject-company` | Onboarding |
| C3-M1 | `deactivate-company`, `reactivate-company` | Deactivation, transfer, reactivation |
| C3-M2 | `register-company`, `transfer-company` | Onboarding, transfer |
| C3-M2 | `register-did` | Onboarding |
| C3-M2 | `update-company` | Core data changed |
| C3-M2 | `register-membership`, `deregister-membership` | Onboarding, reactivation, deactivation |
| C3-M2 | `register-connector`, `deregister-connector` | Connectors in the discovery |
| C4-M1 | `issue-credentials`, `revoke-credentials` | Onboarding, reactivation, deactivation |
| C4-M2 | `create-wallet`, `create-wallet-account`, `delete-wallet-account` | Onboarding, wallet access |
| C6-M1 | `deactivate-subscription-connectors` | Subscription declined, subscription ended |
| C6-M1 | `unlink-technical-user` | Technical user deleted |
| C10-M1 | `notify-provider` | Subscription requested |
| C10-M1 | `send-activation-callback` | Subscription setup, single-instance activation |

Queries between modules (direct, without Process Orchestration), at least:

- C1-M3 → C3-M2 `find-companies` (duplicates at the CSP) and → C3-M1 local duplicate search and master data;
- C1-M2 → C3-M1 master data of the company (reactivation, return); C1-M2 → C1-M3 state of the review case (withdrawal only until the decision);
- C3-M1 → C1-M2 application (`create-company`, `activate-company`) and terms of the company roles (C1-M2 keeps the terms, identifier `agreementKey` with `version`);
- C4-M1 → C3-M1 company, → C4-M2 wallet and holder access;
- C2-M2 → C3-M1 company (invitation to a company without binding); Notifications → C2-M2 people of a company per portal role.

Queries of C6-M1 to C12-M1, at least:

- C6-M1 → C3-M1 status of the host company, → C2-M3 technical user (`usableForConnector`; with `offerSubscriptionId` for a connector as a service), → C10-M1 subscription; C2-M3 → C6-M1 connectors that use a technical user;
- C8-M1 → C3-M1 company roles, → C2-M2 app roles of an active app, → C2-M3 technical user roles, → C10-M1 own subscription to an offer;
- C10-M1 → C8-M1 offer, → C3-M1 company, → C12-M1 processes of a subscription (setup status); C2-M2 and C2-M3 → C10-M1 subscriptions;
- C11-M1 → C10-M1 subscriptions (customers of an app, customer of a subscription);
- C12-M1 lookups → C3-M1 core data of a company, → C8-M1 offer, → C2-M2 instance client of a subscription.

Further queries per module in section 9.

## 7a. Common test IDs

Binding for all modules. An ID stands for exactly one object; a module defines further test IDs only for objects that are not listed here.

| Object | ID |
|---|---|
| Company of OSP A (in the installation of OSP A, `ospCompanyId`): "OSP A Beispiel GmbH", BPN `BPNL00000000OSPA`, company roles `ONBOARDING_SERVICE_PROVIDER`, `ACTIVE_PARTICIPANT`, `APP_PROVIDER` and `SERVICE_PROVIDER` (seed data, active without consents, C3-M1 4) | `11111111-1111-4111-8111-111111111111` |
| OSP A (`ospId`) | `22222222-2222-4222-8222-222222222222` |
| OSP B (`ospId`) | `99a5685c-7397-402c-88de-b1b3a939ecc9` |
| CSP | `92cb10a9-0a9e-4c96-aa09-16d3990b7a51` |
| Beispiel Maschinenbau GmbH, with Business Wallet (as in C5-M1): `EBWOID-DE-0000000001`, EUID `DEXX001T.HRB100001`, BPN from the stand-in of the CSP `BPNLIUK2REMLK3F4` (C3-M2 11.3) | `c0000001-0000-4000-8000-000000000001` |
| Beispiel Logistik GmbH, without Business Wallet: `COMMERCIAL_REG_NUMBER` `HRB 200002` | `c0000002-0000-4000-8000-000000000002` |

Test persons. `sub` in the token is the ID of the account in the Portal IAM (a UUID), not the user name `ebw.<sub at the EBW-Connector>` (C2-M5 5.5):

| Person | `sub` | In the scenario |
|---|---|---|
| Erika Mustermann (as in C5-M1) | `5a1c0001-0000-4000-8000-000000000001` | applicant, then `Company Admin` of Beispiel Maschinenbau GmbH |
| Max Mustermann | `5a1c0002-0000-4000-8000-000000000002` | invited to Beispiel Maschinenbau GmbH by Erika |
| Hans Beispiel | `5a1c0003-0000-4000-8000-000000000003` | applicant, then `Company Admin` of Beispiel Logistik GmbH |
| Lieschen Müller | `5a1c0004-0000-4000-8000-000000000004` | without a company, only `Applicant` |
| Otto Normalverbraucher | `5a1c0012-0000-4000-8000-000000000012` | `OSP Admin`; invites himself as `Company Admin` of the company of OSP A if it offers apps (C2-M2 7.20) |
| Rolf Prüfer | `bbede0f3-fbc4-433b-b7ca-1982e8724912` | `OSP Reviewer` |
| Renate Prüfer | `ed9d901d-cdc3-4a86-8ab4-0b477318506c` | second `OSP Reviewer`, for the four-eyes principle (C1-M3 6.4) |

**Start state of a hackathon installation:** C3-M1 holds only the company of the OSP (seed data). Erika, Max, Hans and Lieschen have accounts without a company, Otto, Rolf and Renate are OSP staff without a company. The flows bind them (5.2, 5.3). A module test may start from another state (for example C2-M5 14.2, where Erika is already bound).

**Client of the UI at the EBW-Connector** (5.2 step 2): `osp-a-ui` and `osp-b-ui`, each registered in `CLIENTS_FILE` (C5-M1 6.8) and allowed in `ATTESTATION_CLIENT_IDS` (C1-M2). Test data of single modules may use `demo-osp` (C5-M1 14.2).

## 8. Terms (binding)

| Term | Meaning |
|---|---|
| OSP | onboarding service provider; in OSP 2.0 it runs its own installation of the portal. Spell it out at its first use in a text. |
| OSP 2.0 | target picture of 28 Oct 2026: every OSP runs its portal, the CSP provides trust |
| CSP | core service provider; runs the core services centrally, optionally offers further services |
| Installation | all modules of an OSP with Process Orchestration, databases, Portal IAM, EBW-Connector and UI |
| Core services | trust list, company identifier with home OSP, directory BPN → DID, discovery, common rules |
| Home OSP | the OSP with which a company is registered; exactly one per company |
| OSP staff | people of the OSP with the portal roles `OSP Reviewer` or `OSP Admin` |
| Portal IAM | Keycloak of the OSP (one realm per installation) |
| Identity provider (sign-in service) | identity provider in the Portal IAM; in OSP 2.0 exactly one: the EBW-Connector of the OSP |
| EUDI Wallet | the person's wallet for identification (PID, mandate) |
| Business Wallet | the company's wallet in the browser (LPID; issues mandates) |
| Dataspace wallet | wallet for the data exchange of the connectors (SAP DIV, IdentityHub, the CSP's dataspace wallet; DCP). Never mix it up with the EUDI Wallet or the Business Wallet. |
| Verification attestation | signed result of the EBW-Connector about an onboarding (C5-M1 section 9) |
| Company identifier | identifier of the company in the dataspace, today the BPN; assigned by the CSP |
| Account | account of a person in the Portal IAM, without a password, linked to the EBW-Connector; used for the portal and for apps |
| Technical user | account of a program (client with service account) |
| Portal roles, app roles | as before: the configuration defines the portal roles (C2-M5), the provider of an app defines its app roles (C2-M2) |

## 9. Writing rules

- English, short sentences, active voice. No filler words, no marketing language.
- No internal names of people or companies of the project team, no product names of wallets of individual vendors. Sign-in only through the EBW-Connector; other kinds of sign-in are not described.
- Today's behaviour only where it helps; the source is the source code of the portal backend 2.6.0 (read only) or the test installation. Do not present assumptions as facts.
- `openapi.yaml` must be valid (checked with `openapi-spec-validator`).
