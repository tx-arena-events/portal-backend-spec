# C3-M1 Company Profile

**Version:** 0.2, as of 2026-10-28
**Capability:** C3 Company
**Kind:** Hackathon
**Files:** this README (behaviour), [openapi.yaml](openapi.yaml) (interface), [open-points.md](open-points.md)

This README and the OpenAPI description are sufficient to build the module. The binding framework is [../FRAMEWORK.md](../FRAMEWORK.md) (OSP 2.0). Other modules are referred to by number and name. Paths, field names and descriptions are in English.

## 1. Purpose

In the installation of an onboarding service provider (OSP), the module keeps the record of every company, from the application to the deactivation: name, address, identifiers, source of the core data, company identifier (BPN), status and company roles with the consents for them. It is the only place in the installation that stores and changes this data. Other modules store only the `companyId` and read the rest here.

The core service provider (CSP) assigns the BPN. C3-M2 registers the company at the CSP; C3-M1 then takes over the BPN with the step `assign-identifier`. C3-M1 does not talk to the CSP itself.

Callers:

| Who | What for |
|---|---|
| Process Orchestration | steps: create company, take over BPN, activate, reject, deactivate, reactivate (section 8) |
| People of a company (via the UI) | read their own company and its company roles; as Company Admin: change the short name, change core data with a verification attestation, submit a change request with supporting documents, change company roles with consents |
| OSP staff (via the UI) | search and read all companies; correct data and deactivate companies (`OSP Admin`); review and decide change requests (`OSP Reviewer`, `OSP Admin`) |
| other modules (queries) | read one or many companies, find companies with the same identifiers, read the role catalogue (section 9) |

## 2. Terms

| Term | Meaning |
|---|---|
| Company | organisation that is or wants to become a member of the dataspace through this OSP. C1-M2 assigns the ID (`companyId`) when it creates the application. |
| Company identifier | identifier of the company in the dataspace, today the BPN (BPNL, 16 characters). Assigned by the CSP. Field `bpn`. |
| Core data | name, address with country and the identifiers `EUID` and `EBWOID`. This is the data that the LPID from the Business Wallet also contains (6.3). |
| Source of the core data | `EBW`: from the LPID of a verification attestation of the EBW-Connector (C5-M1), so verified. `MANUAL`: declared and checked by OSP staff against supporting documents (path without Business Wallet, FRAMEWORK 5.2). Field `coreDataSource`. |
| Identifier | register-related identifier of the company: EUID, EBWOID, VAT ID, commercial register number, LEI, EORI (6.4) |
| Source of an identifier | `BUSINESS_WALLET`: from the verification attestation. `DECLARED`: declared; on the path without Business Wallet checked by OSP staff with the supporting documents. |
| Verification attestation | signed result of the EBW-Connector about an onboarding (C5-M1 section 9) |
| Change request | request of the Company Admin to change core data or further identifiers with supporting documents; OSP staff decide (6.6) |
| Supporting document | document for a change request, for example an extract from the commercial register |
| Company role | role of the company in the dataspace, for example `ACTIVE_PARTICIPANT` (6.5). Not to be confused with portal roles and app roles in the Portal IAM. |
| Terms | document that a company must consent to for a company role, for example terms of use. Identifier `agreementId`. |
| Consent | declaration of the company on terms: given (`ACTIVE`) or withdrawn (`INACTIVE`) |
| Role catalogue | configuration of the installation: company roles with description, terms and who assigns them |
| OSP company | the company that owns the OSP (`ospCompanyId`, FRAMEWORK 4.8); created from the seed data at first start |
| Installation settings | shared file that every module reads at start (FRAMEWORK 4.8) |

## 3. Not part of this module

| What | Where |
|---|---|
| Application, verification attestation of the application, supporting documents of the application, consents at submission | C1-M2 Registration |
| Review case, decision on duplicates and approval | C1-M3 Application Review |
| Request a BPN at the CSP, duplicate check at the CSP, register and deregister membership in the dataspace, directory BPN → DID, transfer at the CSP | C3-M2 CSP Connection |
| Accounts of people, binding to a company, portal roles | C2-M2 Users and App Access |
| Technical users | C2-M3 Technical Users |
| BPN credential, membership credential | C4-M1 Credentials |
| Dataspace wallet, DID | C4-M2 Dataspace Wallet Interface |
| Issue the verification attestation | C5-M1 EBW-Connector |
| Order of the steps, retry after errors | Process Orchestration |
| Notices to people, for example about a new change request | Notifications (other part) |
| Take data, technical users and dataspace wallet along in a transfer | later work (FRAMEWORK 5.6) |
| Self-description (Gaia-X, today field `self_description_document_id`), Clearing House | deferred |
| Use case participation, certificates | later |
| Sites, address BPN | not in scope |

## 4. Technical frame

- Python with FastAPI, part of the modular monolith. Process Orchestration and other modules call the module in the same process through its public interface; the UI calls over HTTP. [openapi.yaml](openapi.yaml) describes both.
- Own database `c3_m1` in PostgreSQL with its own database user. The module reads and writes only there.
- People call with their token from the Portal IAM of the OSP (FRAMEWORK 4.2). The shared token validation library checks the token and returns `sub`, `company_id` and the permissions. The module checks the scope as in 6.8.
- Events go out through the outbox of the base skeleton: stored in the same transaction as the change, delivered afterwards.
- No call to the CSP and no call to C3-M2. The BPN comes as input of the step `assign-identifier`.
- The module checks verification attestations itself, with the JWKS of the EBW-Connector of its own installation (6.7, 11.2).
- Supporting documents are stored in the module's own database. PDF, PNG and JPEG are allowed; the module checks the type from the first bytes of the file, not only from what the caller states.
- The module holds no secrets. Personal data: only the pseudonymous identifier `sub` of the acting person in the audit log, in consents and in change requests. Supporting documents may contain names of people; only OSP staff read them, and they never appear in logs or events.
- **Installation settings:** At start the module reads `SETTINGS_FILE` (FRAMEWORK 4.8): `ospCompanyId` for the seed data, `dataspace.name` and `dataspace.memberOf` for the responses to people (OP-14), `masterDataFile` for countries and identifier types (OP-13). The module does not keep the dataspace as company data; there is exactly one per installation.
- **Seed data:** If `OSP_COMPANY_FILE` is set and the company named in it does not exist, the module creates it at start: status `ACTIVE`, BPN from the file, core data source `MANUAL`, roles from the file (source `SEED`), without consents and without events. The `companyId` in the file must equal `ospCompanyId`; otherwise the start aborts with an error message.

### Configuration

| Variable | Meaning | Example |
|---|---|---|
| `DATABASE_URL` | own database `c3_m1` | |
| `SETTINGS_FILE` | installation settings (FRAMEWORK 4.8); empty = no seed data check, no dataspace name, reference data from 6.4 | `config/installation.json` |
| `COMPANY_ROLES_FILE` | role catalogue (6.5) | `config/company-roles.json` |
| `LANGUAGES` | languages for descriptions and names of terms | `de,en` |
| `OSP_COMPANY_FILE` | seed data of the OSP company; empty = no seed data | `config/osp-company.json` |
| `QUERY_MAX_COMPANIES` | maximum number of companies in a bulk query (9.2) | `100` |
| `REGISTRATION_URL` | C1-M2: empty = built-in stand-in, `in-process` = in the same process, otherwise HTTP address | `in-process` |
| `STUB_APPLICATIONS_FILE` | stand-in for C1-M2 only: applications (11.1) | `stub-applications.json` |
| `EBW_CONNECTOR_URL` | EBW-Connector of the installation: expected `iss` of the verification attestations; JWKS at `{EBW_CONNECTOR_URL}/jwks`; empty = built-in stand-in (11.2) | `https://ebw.osp-a.example.org` |
| `STUB_EBW_JWKS_FILE` | stand-in only: JWKS with the test keys | `stub-ebw-jwks.json` |
| `EBW_CLIENT_ID` | expected `client_id` in the verification attestation (client of the portal at the EBW-Connector); empty = no check | `demo-osp` |
| `ATTESTATION_AUDIENCE` | expected `aud` of the verification attestation | `portal-registration` |
| `ATTESTATION_MAX_AGE_HOURS` | maximum age of a verification attestation for a change, measured from `verified_at` | `24` |
| `CLOCK_SKEW_SECONDS` | tolerance in time checks | `60` |
| `JWKS_CACHE_SECONDS` | cache for the JWKS of the EBW-Connector | `300` |
| `CHANGE_REQUESTS_ENABLED` | change requests with supporting documents allowed (path without Business Wallet; the OSP can switch it off, FRAMEWORK 5.2) | `true` |
| `CHANGE_REQUEST_MAX_DOCUMENTS` | maximum number of supporting documents per change request | `3` |
| `CHANGE_REQUEST_MAX_DOCUMENT_BYTES` | maximum size per supporting document | `10485760` |

## 5. Data

### Table `company`

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key; assigned by C1-M2 |
| `name` | text | 1 to 160 characters, no spaces at the start or end; required |
| `short_name` | text | up to 100 characters |
| `street` | text | 1 to 255 characters; required |
| `house_number` | text | up to 50 characters |
| `address_addition` | text | up to 255 characters |
| `postal_code` | text | up to 12 characters |
| `city` | text | 1 to 255 characters; required |
| `region` | text | part after the hyphen of an ISO 3166-2 code, `^[A-Z0-9]{1,3}$`, for example `BW` |
| `country_code` | char(2) | ISO 3166-1 alpha-2, upper case, from the reference data; required |
| `status` | text | `REQUESTED`, `ACTIVE`, `REJECTED`, `INACTIVE` (6.1) |
| `bpn` | text | `^BPNL[A-Z0-9]{12}$`, in upper case; empty until `assign-identifier`; unique among all companies with a status other than `REJECTED` (partial unique index) |
| `bpn_assigned_at` | timestamptz | takeover of the BPN |
| `core_data_source` | text | `EBW` or `MANUAL` (6.3) |
| `core_data_verified_at` | timestamptz | last verification of the core data: `verified_at` of the verification attestation or approval by OSP staff; with `MANUAL` empty until activation |
| `verification_ref` | text | `jti` of the verification attestation the core data is based on, up to 200 characters; empty with `MANUAL` |
| `application_id` | uuid | application with which the company was created; empty for seed data |
| `deactivation_reason` | text | `MANUAL` or `TRANSFERRED`; filled only with status `INACTIVE` |
| `created_at`, `changed_at` | timestamptz | |
| `activated_at`, `rejected_at`, `deactivated_at`, `reactivated_at` | timestamptz | |

### Table `company_identifier`

| Field | Type | Rule |
|---|---|---|
| `company_id` | uuid | primary key together with `type` |
| `type` | text | `EUID`, `EBWOID`, `VAT_ID`, `COMMERCIAL_REG_NUMBER`, `LEI_CODE`, `EORI` (6.4); at most one identifier per type |
| `value` | text | as declared, format as in 6.4 |
| `value_normalized` | text | upper case, without spaces, `.`, `-`, `/`; index on (`type`, `value_normalized`) |
| `source` | text | `BUSINESS_WALLET` or `DECLARED` |
| `created_at`, `changed_at` | timestamptz | |

### Table `company_role_assignment`

| Field | Type | Rule |
|---|---|---|
| `company_id` | uuid | primary key together with `company_role` |
| `company_role` | text | role from the role catalogue |
| `status` | text | `REQUESTED` (requested with the application), `ACTIVE`, `INACTIVE` (removed) |
| `source_kind` | text | `APPLICATION`, `COMPANY` (Company Admin), `SEED` |
| `source_ref` | text | reference, for example `applicationId` or `sub` of the person, up to 100 characters |
| `activated_at`, `deactivated_at`, `changed_at` | timestamptz | |

### Table `consent`

One row per company and terms, with the latest state.

| Field | Type | Rule |
|---|---|---|
| `company_id` | uuid | primary key together with `agreement_id` |
| `agreement_id` | uuid | terms from the role catalogue |
| `status` | text | `ACTIVE` (given) or `INACTIVE` (withdrawn) |
| `source_kind` | text | `APPLICATION` or `COMPANY` |
| `source_ref` | text | for example `applicationId`, up to 100 characters |
| `consented_by` | text | `sub` of the person who consented, up to 200 characters; empty if unknown |
| `consented_at` | timestamptz | time of the declaration |
| `changed_at` | timestamptz | |

### Table `change_request`

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key; assigned by the module |
| `company_id` | uuid | required; at most one change request with status `OPEN` per company (partial unique index) |
| `status` | text | `OPEN`, `APPROVED`, `DECLINED`, `WITHDRAWN` |
| `changes` | jsonb | requested values: `name`, `address` (without country), `identifiers` (complete new list of the identifiers with source `DECLARED`); only the parts given |
| `comment` | text | justification of the Company Admin, 1 to 1000 characters |
| `requested_by` | text | `sub` of the Company Admin |
| `decided_by` | text | `sub` of the deciding person of the OSP |
| `decision_reason` | text | 1 to 1000 characters; required with `DECLINED` |
| `created_at`, `closed_at` | timestamptz | |

### Table `change_request_document`

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key |
| `change_request_id` | uuid | required |
| `file_name` | text | file name from the caller, up to 200 characters, without path parts |
| `media_type` | text | `application/pdf`, `image/png` or `image/jpeg`, determined from the first bytes |
| `size_bytes` | integer | |
| `sha256` | text | checksum of the content |
| `content` | bytea | content |
| `created_at` | timestamptz | |

### Table `used_attestation`

One row per used verification attestation (7.3): `jti` (primary key), `company_id`, `used_at`.

### Table `audit_log`

Audit log of every changing operation, every step and every read of a supporting document (FRAMEWORK 4.3). No secrets, no names of people.

| Field | Type |
|---|---|
| `id` | uuid, primary key |
| `occurred_at` | timestamptz |
| `sub` | text, acting person; empty for steps |
| `company_id` | uuid |
| `operation` | text, the `operationId` from [openapi.yaml](openapi.yaml) |
| `object_id` | text, for example the ID of the change request or the company role |
| `details` | jsonb: changed fields with old and new value; justification; for steps `processId` and `stepRunId` |

In addition the table `outbox` of the base skeleton.

**Not stored:** names and contact data of people (only `sub`); the verification attestation itself (only `jti` and time of verification); supporting documents of the application (at C1-M2); state at the CSP and membership (C3-M2); DID (C4-M2); credentials (C4-M1); self-description.

## 6. Rules

### 6.1 Status of the company

| From | Trigger | To | Event |
|---|---|---|---|
| – | step `create-company` | `REQUESTED` | `CompanyCreated` |
| `REQUESTED` | step `activate-company`, BPN present | `ACTIVE` | `CompanyActivated` |
| `REQUESTED` | step `reject-company` | `REJECTED` | `CompanyRejected` |
| `ACTIVE` | OSP staff deactivate (7.13, reason `MANUAL`) or step `deactivate-company` (8.5) | `INACTIVE` | `CompanyDeactivated` |
| `INACTIVE` | step `reactivate-company` (8.6) | `ACTIVE` | `CompanyReactivated` |

- `REJECTED` is final. A rejected company comes back with a new application and a new `companyId`.
- Reasons for deactivation: `MANUAL` (OSP staff) and `TRANSFERRED` (transfer to another OSP, FRAMEWORK 5.6). With `MANUAL` a reactivation (application of kind `REACTIVATION`, FRAMEWORK 5.5) leads back, with `TRANSFERRED` only a return (application of kind `TRANSFER`, OP-38). With `TRANSFERRED` another OSP is the home OSP; a reactivation without a transfer at the CSP would go nowhere.
- While `INACTIVE`, neither people of the company nor OSP staff change the data (409 `invalid-state`). Company roles remain unchanged.

### 6.2 Company identifier

- Only the CSP assigns the BPN. C3-M2 registers the company and reports `CompanyRegisteredAtCsp` with the BPN; Process Orchestration then calls `assign-identifier` (8.2).
- Once set, the BPN does not change. A transfer keeps the BPN (FRAMEWORK 5.6).
- A BPN belongs to at most one company with a status other than `REJECTED`.
- The module takes no BPN from the application and no BPN set by hand. If a company already has a BPN from today's dataspace, the CSP confirms it at registration (C3-M2).
- If the registration at the CSP fails, C3-M2 reports `CompanyRegistrationAtCspFailed` and shows the reason to OSP staff. C3-M1 changes nothing: the company stays `REQUESTED` without a BPN until the registration succeeds or the flow calls `reject-company`. C3-M1 has no error event of its own.

### 6.3 Core data and its source

Core data is `name`, the address (`street`, `houseNumber`, `addressAddition`, `postalCode`, `city`, `region`, `countryCode`) and the identifiers `EUID` and `EBWOID`.

- **At creation** the application determines the source: verification attestation → `EBW`; supporting documents → `MANUAL` (8.1).
- **Leading** is the source through which the company was verified (OP-09): with `EBW` the LPID, with `MANUAL` the supporting documents checked by OSP staff.
- **Passing to the CSP:** At registration, the core data goes to the CSP through C3-M2 `register-company`; Process Orchestration reads it through 9.1. `CompanyDataChanged` marks changed core data with `coreDataChanged: true`; Process Orchestration then passes it to the CSP through C3-M2, so that the duplicate check at the CSP works with the current name. The module no longer calls a master data service.

**Mapping of a verification attestation** (C5-M1 9.1) to the fields, in 7.3 and 8.6:

| Claim | Field | Rule |
|---|---|---|
| `company.legal_name` | `name` | taken over |
| `company.address.street_address` | `street` | taken over; `houseNumber` empty |
| `company.address.postal_code` | `postalCode` | taken over |
| `company.address.locality` | `city` | taken over |
| `company.country` | `countryCode` | must stay the same (6.6) |
| `company.ebwoid` | identifier `EBWOID` | must equal the stored EBWOID; source `BUSINESS_WALLET` |
| `company.euid` | identifier `EUID` | taken over with source `BUSINESS_WALLET`; if the claim is missing, the stored EUID stays |

- **Comparison** as in C1-M2: both values in Unicode normalisation form NFKC, leading and trailing whitespace removed, inner whitespace collapsed to one space, case ignored. Equal values count as unchanged; then the stored value stays, including its spelling.
- **Address:** It counts as unchanged if `street` and `houseNumber`, joined with a space, equal `street_address`, and `postalCode` and `city` equal the claims. Otherwise the module replaces the address: values from the claims, `houseNumber`, `addressAddition` and `region` empty. If `company.address` is missing, the address stays.
- A value that breaks the rules of section 5 (for example a name over 160 characters): 400 `validation`; nothing is changed.

### 6.4 Identifiers

Identifier types and validation patterns are in the shared file of the reference data (`masterDataFile`, OP-13). The table lists the values the module expects; if the file or a type in it is missing, these apply.

| Type | Meaning | Format | Today (Portal 2.6.0) |
|---|---|---|---|
| `EUID` | European Unique Identifier from the interconnection of business registers | 4 to 100 characters of letters, digits, `.` and `-`; no fixed pattern (OP-12) | no own type; the slot `VIES` is passed on as EUID |
| `EBWOID` | identifier of the owner of a Business Wallet (from the LPID) | 1 to 255 characters without whitespace; no fixed pattern (OP-12) | new |
| `VAT_ID` | VAT identification number | country `DE`: `^DE\d{9}$`; otherwise `^(?!.*\s$)([A-Za-z0-9](\.\|\s\|-\|\/)?){5,18}$` | `VAT_ID` |
| `COMMERCIAL_REG_NUMBER` | commercial register number | country `DE`: `^(?!.*\s$)([A-Za-zÀ-ÿ])([A-Za-zÀ-ÿ0-9.()](\s\|-\|_)?){4,50}$`; `FR`: `^\d{9}$`; otherwise `^(?!.*\s$)([A-Za-zÀ-ÿ0-9.()](\.\|\s\|-\|_)?){4,50}$` | `COMMERCIAL_REG_NUMBER` |
| `LEI_CODE` | Legal Entity Identifier | `^[A-Za-z0-9]{20}$` | `LEI_CODE` |
| `EORI` | EORI customs number | `^[A-Z]{2}[A-Za-z0-9]{1,15}$` | `EORI` |

- A company has at least one identifier and at most one per type. The type `VIES` is dropped.
- In the patterns, `\|` stands for the vertical bar (escaped in the table).
- The country that counts is the `country_code` of the company.
- Comparison for the duplicate search uses `value_normalized`.

### 6.5 Company roles and role catalogue

The role catalogue is configuration of the installation and is in `COMPANY_ROLES_FILE`:

```json
{
  "roles": [
    {
      "companyRole": "ACTIVE_PARTICIPANT",
      "assignableBy": "COMPANY",
      "descriptions": { "de": "Teilnehmer am Datenaustausch …", "en": "Participant in data exchange …" },
      "agreements": [
        {
          "agreementId": "aa0a0000-7fbc-1f2f-817f-bce0502c1010",
          "names": { "de": "Nutzungsbedingungen Teilnehmer", "en": "Terms & Conditions – Active Participant" },
          "documentUrl": "https://osp-a.example.org/terms/active-participant-v1.pdf"
        }
      ]
    }
  ]
}
```

`assignableBy` defines who assigns the role:

| Value | Meaning |
|---|---|
| `COMPANY` | selectable in the application (C1-M2) and added or removed by the Company Admin (7.8, 7.9) |
| `SEED` | only through the seed data |

Initial values, following today's portal (terms with status "active"):

| Company role | `assignableBy` | Terms | Remark |
|---|---|---|---|
| `ACTIVE_PARTICIPANT` | `COMPANY` | terms of use for participants (`aa0a0000-7fbc-1f2f-817f-bce0502c1010`), security rules for participants (`aa0a0000-7fbc-1f2f-817f-bce0502c1013`) | |
| `APP_PROVIDER` | `COMPANY` | terms of use for app providers (`aa0a0000-7fbc-1f2f-817f-bce0502c1011`) | |
| `SERVICE_PROVIDER` | `COMPANY` | terms of use for service providers (`aa0a0000-7fbc-1f2f-817f-bce0502c1017`) | |
| `ONBOARDING_SERVICE_PROVIDER` | `SEED` | none | only the OSP company; also exists today |

Rules:

- A role is active only if the company has consented to all terms of the role in the catalogue (consent `ACTIVE`).
- In 7.8 the module rejects consents to terms that do not belong to the role. At activation (8.3) it takes over only consents to terms of the requested roles. The consents to the terms of the OSP and the dataspace given at submission stay with C1-M2.
- A new version of terms gets a new `agreementId`. Existing roles stay active; re-consent is not part of version 0.2 (OP-11).
- An active company keeps at least one active role with `assignableBy` `COMPANY`.
- If the Company Admin removes a role, the module sets the consents to its terms to `INACTIVE`, unless the terms also belong to another active role.

### 6.6 What may be changed

| Field | Company Admin directly (7.2) | With verification attestation (7.3) | Change request (7.4) | OSP staff, correction (7.12) |
|---|---|---|---|---|
| `shortName` | yes | – | – | yes |
| `name` | no | yes | only with source `MANUAL` | yes |
| Address without country | no | yes | only with source `MANUAL` | yes |
| `countryCode` | no | no, must be equal | no | only at `REQUESTED` without BPN |
| Identifier `EBWOID` | no | no, must be equal | yes, if source `DECLARED` | yes, if source `DECLARED` |
| Identifier `EUID` | no | yes | yes, if source `DECLARED` | yes, if source `DECLARED` |
| Further identifiers (`VAT_ID`, `COMMERCIAL_REG_NUMBER`, `LEI_CODE`, `EORI`) | no | – | yes | yes |
| `bpn` | no | no | no | no (only `assign-identifier`) |
| `status` | no | no | no | deactivate only (7.13) |
| Company roles | yes, with consents (7.8, 7.9) | – | – | no |

- People of the company make changes only with status `ACTIVE`. OSP staff correct at `REQUESTED` and `ACTIVE`.
- Identifiers with source `BUSINESS_WALLET` are changed only by a verification attestation.
- **Source `EBW`:** Name and address change only with a new verification attestation (OP-10). A change request for these fields gives 409 `attestation-required`.
- **Correction by OSP staff** of the name or address of a company with source `EBW` sets the source to `MANUAL`: the data is then no longer backed by the LPID. A later verification attestation sets it back to `EBW`.
- **Change from `MANUAL` to `EBW`:** with a verification attestation whose EBWOID equals the stored one. The EBWOID must have been entered before as an identifier with source `DECLARED`, through a change request or a correction. Without an entered EBWOID the module rejects every verification attestation (409 `company-mismatch`); otherwise any Business Wallet could take over a company.
- **Country:** no longer changes after the BPN has been assigned. The CSP knows the company with this country.

### 6.7 Checking a verification attestation

In 7.3, in this order. The first error determines the reason `reason` in the response 400 `attestation-invalid`:

1. Compact JWS form, header readable; otherwise `format`.
2. `typ` is `ebw-attestation+jwt`, `alg` is `ES256`; `none` is never allowed. Otherwise `algorithm`.
3. `iss` equals `EBW_CONNECTOR_URL`, so the EBW-Connector of the own installation; otherwise `issuer`.
4. Key for `kid` from the JWKS of the EBW-Connector (11.2), signature valid. Otherwise `signature`. The module ignores header parameters such as `jku`, `x5u` or `jwk`. If the JWKS cannot be fetched: 502 `upstream-error`.
5. `aud` contains `ATTESTATION_AUDIENCE`; otherwise `audience`.
6. `exp` is after now minus `CLOCK_SKEW_SECONDS`, `iat` not after now plus `CLOCK_SKEW_SECONDS`; otherwise `expired` or `not-yet-valid`.
7. Required claims present and in the right format: `jti`, `verified_at`, `purpose` equal to `onboarding`, `company.legal_name`, `company.ebwoid`, `company.country`, `representation.confirmed`; `client_id` equal to `EBW_CLIENT_ID`, if set. Otherwise `claims`.
8. `verified_at` not older than `ATTESTATION_MAX_AGE_HOURS`; otherwise `too-old`.

Then, each with 409:

- `jti` already used (`used_attestation`): `attestation-used`.
- `representation.confirmed` not `true`: `representation-not-confirmed`. Without confirmed representation only the change request remains.
- `company.ebwoid` not equal to the stored EBWOID, or no EBWOID stored; `company.country` not equal to `countryCode`: `company-mismatch` with the fields in `fields`.
- `verified_at` not after `core_data_verified_at`: `attestation-outdated`. This way an older verification attestation, such as the one of the application, does not reset newer data.

The person in the verification attestation need not be the calling person. The Company Admin may have the verification confirmed by a representative of the company, for example through the QR code on the admin's screen. What counts is the confirmed representation.

### 6.8 Permissions and scope

| Permission | Scope | What for | Portal roles (proposal to C2-M5) |
|---|---|---|---|
| `view_company` | `company` | read own company, its company roles and change requests (7.1, 7.5, 7.7) | `Company Admin`, `IT Admin`, `Member` |
| `edit_company` | `company` | change the short name, change core data with a verification attestation, submit and withdraw a change request (7.2, 7.3, 7.4, 7.6) | `Company Admin` |
| `edit_company_roles` | `company` | add and remove company roles with consents (7.8, 7.9) | `Company Admin` |
| `view_companies` | `osp` | search and read all companies of the installation (7.10, 7.11) | `OSP Reviewer`, `OSP Admin` |
| `manage_companies` | `osp` | correct data, deactivate companies (7.12, 7.13) | `OSP Admin` |
| `review_company_changes` | `osp` | read and decide change requests and supporting documents (7.14 to 7.18) | `OSP Reviewer`, `OSP Admin` |
| `query_c3_m1` | `osp` | queries over HTTP (section 9) | service accounts only |
| `execute_steps_c3_m1` | `osp` | steps over HTTP (section 8) | service accounts only |

The portal role `Applicant` has no permission of this module. For queries and steps over HTTP the same check applies as in the scope `osp`: only the permission.

**Order of checks:**

1. Token missing or invalid: 401 `unauthorized`.
2. Permission missing: 403 `forbidden`.
3. Scope `company`: If the token has no claim `company_id` or it differs from the `companyId` in the path: 404 `not-found`. The same for an unknown company. A change request of another company also gives 404.
4. Then the checks of the operation.

**Audit log:** Every changing operation and every step writes an entry in `audit_log` with `sub` from the token (empty for steps, with `processId` and `stepRunId` instead). Reading a supporting document (7.16) is logged as well.

### 6.9 Idempotency and concurrency

- Every step finds its company through the `companyId` from the input, creates only what is missing and returns the same result. Events are emitted only if something has changed.
- Every changing operation and every step locks the row in `company` (`SELECT … FOR UPDATE`) before it checks and changes. Decisions on change requests also lock the row in `change_request`.
- Events go into the outbox in the same transaction as the change.
- A verification attestation is valid exactly once (7.3). A second call with the same one gives 409 `attestation-used`, even if it changed nothing.

## 7. Operations

All paths start with `/api/c3-m1`. Checks as in 6.8. Errors as in section 12.

### 7.1 `GET /api/c3-m1/companies/{companyId}`

Permission `view_company`, scope `company`. Returns the company (`Company`): master data, identifiers with source, source of the core data with time of the last verification, BPN, status, company roles with status and `dataspaceName` from the installation settings.

### 7.2 `PATCH /api/c3-m1/companies/{companyId}`

Permission `edit_company`, scope `company`. Changes the short name.

1. Company not `ACTIVE`: 409 `invalid-state`.
2. Only `shortName` is allowed. Other fields: 409 `field-locked` with the fields in `fields`. `null` deletes the short name.
3. Check as in section 5, otherwise 400 `validation`.
4. No change: 200 without event and without audit log entry.
5. Store, audit log with old and new value, event `CompanyDataChanged` with `changedFields` `[shortName]` and `coreDataChanged` `false`. Response 200 with the company.

### 7.3 `POST /api/c3-m1/companies/{companyId}/attestation-updates`

Permission `edit_company`, scope `company`. Changes the core data with a new verification attestation of the EBW-Connector of the installation, immediately and without OSP staff (OP-10). The UI obtains the verification attestation as in onboarding (scopes `openid profile representation company attestation`) and passes it on here. Input: `attestation` (compact JWS).

1. Company not `ACTIVE`: 409 `invalid-state`.
2. Check the verification attestation as in 6.7.
3. Build the new values as in 6.3 and check them as in section 5, otherwise 400 `validation`.
4. In one transaction: store changed core data; identifier `EBWOID` with source `BUSINESS_WALLET` (a previous source `DECLARED` becomes `BUSINESS_WALLET`); `EUID` from the claim, if present; `core_data_source` `EBW`; `core_data_verified_at` = `verified_at`; `verification_ref` = `jti`; `jti` in `used_attestation`. Audit log with old and new values.
5. If a field or the source has changed: event `CompanyDataChanged` with `changedFields`, `coreDataChanged` `true` and `coreDataSource` `EBW`.
6. Response 200 `AttestationUpdateResult`: `changedFields` (empty if the data was already correct) and the company.

An open change request remains; on approval the rules for the source valid at that time apply (7.17).

### 7.4 `POST /api/c3-m1/companies/{companyId}/change-requests`

Permission `edit_company`, scope `company`. Change request with supporting documents, for the path without Business Wallet and for further identifiers. Input as `multipart/form-data`:

- Part `data` (JSON, `ChangeRequestData`): at least one of `name`, `address` (address without country), `identifiers` (complete new list of the identifiers with source `DECLARED`); plus the required field `comment` (1 to 1000 characters).
- Part `documents`: 1 to `CHANGE_REQUEST_MAX_DOCUMENTS` files.

Procedure:

1. `CHANGE_REQUESTS_ENABLED` is `false`: 409 `change-requests-disabled`.
2. Company not `ACTIVE`: 409 `invalid-state`.
3. Already an open change request: 409 `change-request-open`.
4. Check the data as in section 5 and 6.4. `identifiers` must not contain a type with source `BUSINESS_WALLET`; afterwards at least one identifier must remain. Otherwise 400 `validation`.
5. Source `EBW` and `name` or `address` differ from the current state: 409 `attestation-required`.
6. No part differs from the current state: 400 `validation`.
7. Check the supporting documents: number, size per file up to `CHANGE_REQUEST_MAX_DOCUMENT_BYTES`, type PDF, PNG or JPEG from the first bytes. Otherwise 400 `document-invalid`.
8. Store with status `OPEN`, `requested_by` = `sub`; store the supporting documents; audit log; event `CompanyChangeRequestSubmitted`. Response 201 `ChangeRequest`.

### 7.5 `GET /api/c3-m1/companies/{companyId}/change-requests`

Permission `view_company`, scope `company`. Change requests of the company, newest first, paged (`page`, `size`). Per supporting document only `id`, `fileName`, `mediaType`, `sizeBytes`; no content, no `decidedBy`.

### 7.6 `POST /api/c3-m1/companies/{companyId}/change-requests/{changeRequestId}/withdraw`

Permission `edit_company`, scope `company`. Unknown or of another company: 404. `WITHDRAWN`: 200 without change. `APPROVED` or `DECLINED`: 409 `invalid-state`. Otherwise status `WITHDRAWN`, `closed_at`, audit log, event `CompanyChangeRequestClosed` with `outcome` `WITHDRAWN`. Response 200 `ChangeRequest`.

### 7.7 `GET /api/c3-m1/companies/{companyId}/company-roles?lang=`

Permission `view_company`, scope `company`. All roles of the catalogue with `assignableBy` `COMPANY` and all roles that the company has or had, each with status (`ACTIVE`, `REQUESTED`, `INACTIVE` or `NONE`), description in `lang` (default `en`) and, for each of the terms: name, document URL, consent status and time. This way the UI also shows the Company Admin the roles that can be added.

### 7.8 `POST /api/c3-m1/companies/{companyId}/company-roles`

Permission `edit_company_roles`, scope `company`. Add a role. Input: `companyRole`, `agreementIds` (terms that the Company Admin consents to for the company).

1. Company not `ACTIVE`: 409 `invalid-state`.
2. Role not in the catalogue: 400 `validation`. `assignableBy` not `COMPANY`: 409 `role-not-assignable`.
3. Role already `ACTIVE`: 200 with the roles, no change.
4. All terms of the role must be in `agreementIds`, otherwise 400 `consent-missing` with the missing IDs in `missing`. An ID that does not belong to the role: 400 `validation`.
5. Store the consents (status `ACTIVE`, source `COMPANY`, `consented_by` = `sub`, `consented_at` = now), role `ACTIVE` with source `COMPANY`.
6. Audit log, event `CompanyRolesChanged` with all active roles. Response 200 with the roles as in 7.7.

### 7.9 `DELETE /api/c3-m1/companies/{companyId}/company-roles/{companyRole}`

Permission `edit_company_roles`, scope `company`.

1. Company not `ACTIVE`: 409 `invalid-state`.
2. Role not in the catalogue: 404 `not-found`. `assignableBy` not `COMPANY`: 409 `role-not-assignable`.
3. Role not `ACTIVE`: 200, no change.
4. No active role with `assignableBy` `COMPANY` would remain: 409 `last-role`.
5. Role `INACTIVE`, consents set to `INACTIVE` as in 6.5. Audit log, event `CompanyRolesChanged`. Response 200 with the roles as in 7.7.

The module does not check consequences for offers or subscriptions of the company.

### 7.10 `GET /api/c3-m1/osp/companies`

Permission `view_companies`, scope `osp`. Filters: `status` (repeatable), `name` (contains, case-insensitive), `bpn`, `hasBpn` (`true` or `false`), `companyRole` (active only), `coreDataSource`. Sorted by `createdAt` descending, paged. Response `{ items, page, size, total }` with summaries (`CompanySummary`).

### 7.11 `GET /api/c3-m1/osp/companies/{companyId}`

Permission `view_companies`, scope `osp`. The company with all fields (`CompanyDetail`), including `applicationId`, `verificationRef`, `bpnAssignedAt`, `deactivationReason`, all timestamps and `openChangeRequestId`. Unknown: 404.

### 7.12 `PATCH /api/c3-m1/osp/companies/{companyId}`

Permission `manage_companies`, scope `osp`. Correction by OSP staff, for example after a hint in the Application Review or if the Business Wallet is no longer available. Input: fields as in 6.6 (`name`, `shortName`, `address` with country, `identifiers`) and the required field `reason` (1 to 500 characters).

1. Unknown: 404. Status `REJECTED` or `INACTIVE`: 409 `invalid-state`.
2. Check every field as in 6.6; not allowed: 409 `field-locked` with the fields in `fields`.
3. `identifiers`: complete new list of the identifiers with source `DECLARED`. Identifiers with source `BUSINESS_WALLET` stay and must not be in the list (400 `validation`). Afterwards at least one identifier must be present.
4. Check as in section 5 and 6.4, otherwise 400 `validation`.
5. No change: 200 without event and without audit log entry.
6. Store. If the name or address of a company with source `EBW` changes: source `MANUAL`, `verification_ref` empty. If core data changes: `core_data_verified_at` = now. Audit log with `reason`, event `CompanyDataChanged` (`coreDataChanged` `true` if core data is affected). Response 200 `CompanyDetail`.

### 7.13 `POST /api/c3-m1/osp/companies/{companyId}/deactivate`

Permission `manage_companies`, scope `osp`. Manual deactivation (FRAMEWORK 5.4). Input: `reason` (justification, 1 to 500 characters).

1. Unknown: 404. Status `INACTIVE`: 200 without change. Status not `ACTIVE`: 409 `invalid-state`.
2. Status `INACTIVE`, `deactivation_reason` `MANUAL`, `deactivated_at`; audit log with the justification; event `CompanyDeactivated` with `reason` `MANUAL`, without the justification text.
3. Response 200 `CompanyDetail`.

The flow in FRAMEWORK 5.4 controls the consequences (lock accounts and technical users, revoke the membership credential, deregister the membership at the CSP). An open change request stays open; it can be approved only after a reactivation, and declined at any time.

### 7.14 `GET /api/c3-m1/osp/change-requests`

Permission `review_company_changes`, scope `osp`. Filters: `status` (repeatable), `companyId`. Sorted by `createdAt` ascending (oldest first), paged. Response `{ items, page, size, total }` with `ChangeRequest` and the name of the company.

### 7.15 `GET /api/c3-m1/osp/change-requests/{changeRequestId}`

Permission `review_company_changes`, scope `osp`. The change request (`ChangeRequestDetail`) with the current state of the company for comparison (name, address, identifiers, source of the core data, status) and the information on the supporting documents. Unknown: 404.

### 7.16 `GET /api/c3-m1/osp/change-requests/{changeRequestId}/documents/{documentId}`

Permission `review_company_changes`, scope `osp`. Returns the content of a supporting document with the stored `Content-Type` and `Content-Disposition: attachment` with the file name. Unknown or belonging to another change request: 404. Audit log entry with `sub` and `documentId`.

### 7.17 `POST /api/c3-m1/osp/change-requests/{changeRequestId}/approve`

Permission `review_company_changes`, scope `osp`. Input: optional `comment` (up to 1000 characters, audit log only).

1. Unknown: 404. `APPROVED`: 200 without change. `DECLINED` or `WITHDRAWN`: 409 `invalid-state`.
2. `sub` equals `requested_by`: 409 `same-person`. Nobody approves their own request.
3. Company not `ACTIVE`: 409 `invalid-state`.
4. Check the rules again against the current state (6.6, section 5, 6.4): for example, source now `EBW` and name changed: 409 `attestation-required`; invalid values: 400 `validation`.
5. In one transaction: apply the changes; if core data changes: the source stays `MANUAL` or becomes `MANUAL` with name or address, `core_data_verified_at` = now. Status `APPROVED`, `decided_by`, `closed_at`. Audit log.
6. Events: `CompanyDataChanged`, if anything has changed compared with the current state; `CompanyChangeRequestClosed` with `outcome` `APPROVED`.
7. Response 200 `ChangeRequest`.

### 7.18 `POST /api/c3-m1/osp/change-requests/{changeRequestId}/decline`

Permission `review_company_changes`, scope `osp`. Input: `reason` (1 to 1000 characters, required; the Company Admin sees it in 7.5).

Unknown: 404. `DECLINED`: 200 without change, the stored justification stays. `APPROVED` or `WITHDRAWN`: 409 `invalid-state`. Otherwise status `DECLINED`, `decision_reason`, `decided_by`, `closed_at`, audit log, event `CompanyChangeRequestClosed` with `outcome` `DECLINED`. Response 200 `ChangeRequest`.

## 8. Steps for Process Orchestration

Common rules as in FRAMEWORK 4.4:

- Input always with `processId` and `stepRunId` (for the audit log only).
- Result `{ "stepRunId", "state": "DONE", "result" }`. All steps of this module are short; there is no `RUNNING`.
- Errors as Problem Details with `retryable` (section 12). What the step has already changed stays; a repetition completes it.

### 8.1 `POST /api/c3-m1/steps/create-company`

In onboarding and in a transfer to this OSP, after `ApplicationSubmitted` (kind `ONBOARDING` or `TRANSFER`). From submission on, the application data is fixed.

- **Input:** `companyId`, `applicationId`.
- **Preconditions:** C1-M2 holds the submitted application `applicationId` of kind `ONBOARDING` or `TRANSFER` with this `companyId`. The company does not yet exist in this installation.
- **Behaviour:**
  1. Company exists: the same `application_id` gives `DONE` with the stored state, without event; a different one gives 409 `conflict`.
  2. Read the application at C1-M2 (11.1). Unknown: 404 `not-found`. C1-M2 not reachable: 502 `upstream-error`. Different `companyId` in the application: 409 `conflict`. Kind `REACTIVATION`: 409 `conflict`; `reactivate-company` exists for that. Application not submitted: 409 `invalid-state`.
  3. Check the data as in section 5 and 6.4, the requested roles against the catalogue (`assignableBy` `COMPANY`). Verification method with verification attestation but without EBWOID: 400 `validation`.
  4. Create in one transaction: status `REQUESTED`, without BPN; core data source `EBW` (verification attestation) with `core_data_verified_at` = `verifiedAt` and `verification_ref` = `jti`, or `MANUAL` (supporting documents) without both; identifiers with the source from the application; requested roles with status `REQUESTED` and source `APPLICATION`. Audit log, event `CompanyCreated`.
- **Result:** `companyId`, `status`, `bpn` (empty), `coreDataSource`.
- **Event:** `CompanyCreated`.

On the return of a company that transferred away (application of kind `TRANSFER` for an existing `companyId`, OP-38), the flow does not call `create-company`; a call gives 409 `conflict` as in point 1.

### 8.2 `POST /api/c3-m1/steps/assign-identifier`

In onboarding and in a transfer, after `CompanyRegisteredAtCsp` from C3-M2. Replaces the former step `request-identifier`.

- **Input:** `companyId`, `bpn` (from `CompanyRegisteredAtCsp`).
- **Preconditions:** Company with status `REQUESTED`; registered at the CSP. The module does not check the second; the flow ensures it.
- **Behaviour:**
  1. Unknown: 404 `not-found`. `bpn` not in the format `BPNL` + 12 letters or digits (case does not matter): 400 `validation`.
  2. BPN already set: the same BPN gives `DONE` without event, in any status (also on the return of a company with status `INACTIVE`); a different BPN gives 409 `bpn-mismatch`.
  3. Status not `REQUESTED`: 409 `invalid-state`.
  4. BPN held by another company with a status other than `REJECTED`: 409 `bpn-taken`.
  5. Store the BPN in upper case, `bpn_assigned_at`; audit log; event `CompanyIdentifierAssigned`.
- **Result:** `companyId`, `status`, `bpn`.
- **Event:** `CompanyIdentifierAssigned`.

If the registration at the CSP fails, the flow does not call this step (6.2).

### 8.3 `POST /api/c3-m1/steps/activate-company`

In onboarding and in a transfer at the end, after `assign-company-admin` (C2-M2).

- **Input:** `companyId`, `applicationId`.
- **Preconditions:** Company `REQUESTED` with BPN. Consents of the application submitted at C1-M2.
- **Behaviour:**
  1. Status `ACTIVE`: `DONE` with the state, without event. `REJECTED` or `INACTIVE`: 409 `invalid-state`.
  2. Without BPN: 409 `identifier-missing`.
  3. `applicationId` not equal to the `application_id` of the company: 409 `conflict`.
  4. Read the consents of the application at C1-M2 (11.1). Every role with status `REQUESTED` whose terms all have consent becomes active; the consents are stored with source `APPLICATION`, `consented_by` and the time from the application. A role with missing consent stays `REQUESTED` and is listed in the result under `pendingCompanyRoles`. If no role would become active: 409 `roles-without-consent`, nothing is changed.
  5. In one transaction: roles and consents as in 4; with source `MANUAL` `core_data_verified_at` = now, if empty (OSP staff have approved the supporting documents in C1-M3); status `ACTIVE`, `activated_at`; audit log; event `CompanyActivated` with BPN and active roles. There is no separate `CompanyRolesChanged` in this case.
- **Result:** `companyId`, `status`, `bpn`, `companyRoles`, `pendingCompanyRoles`.
- **Event:** `CompanyActivated`.

C3-M2 registers the membership in the dataspace (`register-membership`), not this step.

### 8.4 `POST /api/c3-m1/steps/reject-company`

In onboarding and in a transfer, after `ApplicationDeclined` or `ApplicationWithdrawn`.

- **Input:** `companyId`.
- **Preconditions:** Company with status `REQUESTED`.
- **Behaviour:** Unknown: 404 `not-found`. `REJECTED`: `DONE` without event. `ACTIVE` or `INACTIVE`: 409 `invalid-state`. Otherwise status `REJECTED`, `rejected_at`, audit log, event `CompanyRejected`. A BPN, if already present, becomes free for other companies of the installation.
- **Result:** `companyId`, `status`.
- **Event:** `CompanyRejected`.

### 8.5 `POST /api/c3-m1/steps/deactivate-company`

In a transfer to another OSP, after `CompanyTransferredAway` from C3-M2, with reason `TRANSFERRED` (FRAMEWORK 5.6). Also usable for other flows with reason `MANUAL`. Manual deactivation runs through 7.13.

- **Input:** `companyId`, `reason` (`MANUAL` or `TRANSFERRED`).
- **Preconditions:** Company with status `ACTIVE`.
- **Behaviour:**
  1. Unknown: 404 `not-found`.
  2. Status `INACTIVE`: `DONE` without event. If `reason` is `TRANSFERRED` and the stored reason is `MANUAL`, the module sets the reason to `TRANSFERRED` (audit log), because a return then needs a transfer (6.1).
  3. Status `REQUESTED` or `REJECTED`: 409 `invalid-state`.
  4. Status `INACTIVE`, `deactivation_reason`, `deactivated_at`; audit log; event `CompanyDeactivated` with `reason`.
- **Result:** `companyId`, `status`, `deactivationReason`.
- **Event:** `CompanyDeactivated`, then the flow in FRAMEWORK 5.4.

### 8.6 `POST /api/c3-m1/steps/reactivate-company`

In a reactivation after `ApplicationApproved` with kind `REACTIVATION` (FRAMEWORK 5.5). On the return of a company that transferred away with kind `TRANSFER`, after `CompanyRegisteredAtCsp` and `assign-identifier`, instead of `activate-company` (OP-38).

- **Input:** `companyId`, `applicationId`.
- **Preconditions:** Company `INACTIVE` with BPN. Application approved, with this `companyId`, kind matching the reason of the deactivation: `REACTIVATION` with `MANUAL`, `TRANSFER` with `TRANSFERRED`.
- **Behaviour:**
  1. Unknown: 404 `not-found`. Status `ACTIVE`: `DONE` with the state, without event. `REQUESTED` or `REJECTED`: 409 `invalid-state`.
  2. Read the application at C1-M2 (11.1). Unknown: 404 `not-found`. C1-M2 not reachable: 502 `upstream-error`. Different `companyId`: 409 `conflict`. Kind does not match the reason: 409 `invalid-state`.
  3. Core data from the application:
     - Verification method with verification attestation: EBWOID equal to the stored one, country equal to `countryCode`, otherwise 409 `company-mismatch`. Name, address and EUID as in 6.3; source `EBW`, `core_data_verified_at` = `verifiedAt`, `verification_ref` = `jti`.
     - Verification method with supporting documents: country equal to `countryCode`, otherwise 409 `company-mismatch`. Name, address and the identifiers with source `DECLARED` from the application; identifiers with source `BUSINESS_WALLET` stay. Source `MANUAL`, `core_data_verified_at` = now (OSP staff have approved the supporting documents in C1-M3).
     - Check the values as in section 5 and 6.4, otherwise 400 `validation`.
  4. In one transaction: core data, status `ACTIVE`, `deactivation_reason` empty, `reactivated_at`; audit log; events `CompanyDataChanged` (if data has changed) and `CompanyReactivated` with BPN and active roles.
- **Result:** `companyId`, `status`, `bpn`, `companyRoles`, `changedFields`.
- **Event:** `CompanyReactivated`, then the flow in FRAMEWORK 5.5 no. 3.

Company roles and their consents stay as before the deactivation. C1-M2 obtains consent to the terms of the OSP and the dataspace when the application is submitted.

## 9. Queries for other modules

In the same process without a token; over HTTP with the permission `query_c3_m1`. The queries change nothing. Callers may cache the result briefly; this module remains authoritative.

### 9.1 `GET /api/c3-m1/internal/companies/{companyId}`

One company (`CompanyInternal`): `id`, `name`, `shortName`, `address`, `identifiers` (with source), `coreDataSource`, `status`, `bpn`, `companyRoles` (active only), `deactivationReason`, `createdAt`, `activatedAt`, `deactivatedAt`. Unknown: 404 `not-found`.

Used by: C2-M2, C2-M3, C4-M1, C4-M2 (status, BPN, name, roles); C1-M2 (master data in reactivation and return); Process Orchestration (data for C3-M2 `register-company`: `name` → `legalName`, identifiers `EBWOID` and `EUID`, `countryCode` → `country`, address with `street` and `houseNumber` joined into `streetAddress`).

### 9.2 `GET /api/c3-m1/internal/companies?id=…&id=…` or `?bpn=…&bpn=…`

Many companies at once, for example for lists of offers with the name of the provider. Exactly one of the two kinds must be given, with 1 to `QUERY_MAX_COMPANIES` values; otherwise 400 `validation`. BPN case-insensitive. Response `{ "items": [CompanyInternal], "notFound": [value] }`, `items` in the order of the request.

### 9.3 `POST /api/c3-m1/internal/companies/identifier-matches`

Duplicate search in the own installation. Complements the duplicate check at the CSP (C3-M2 `find-companies`): the CSP knows a company only after approval; only this query finds two open applications for the same company at this OSP. The query changes nothing; POST only because of the list in the body.

- Input: `identifiers` (1 to 10 pairs of `type` and `value`; `BPN` is also allowed as `type`), optional `name`, optional `excludeCompanyId` (the company of the application itself), optional `statuses` (default `REQUESTED`, `ACTIVE`, `INACTIVE`).
- Match: same type and same normalised value (6.4); with `BPN` the same BPN. With `name`: in addition the same name after trimming, collapsing spaces and ignoring case.
- Response: `matches`, per company `companyId`, `name`, `status`, `bpn` and `matchedOn` (matching identifier types, `BPN` or `NAME`). Sorted by number of matches, descending.

C1-M3 decides whether a match is a duplicate.

### 9.4 `GET /api/c3-m1/internal/company-roles?lang=`

Role catalogue: per role `companyRole`, `assignableBy`, description in `lang` and all descriptions, terms with names and document URL. For C1-M2 (selectable roles in the application and terms to be consented to at submission) and C2-M3 (assignable roles of technical users per company role). People do not read the catalogue here, but through C1-M2 (application) or 7.7 (own company).

## 10. Events

In the outbox in the same transaction as the change. Every event also carries `eventId`, `eventType`, `occurredAt`. None contains secrets or personal data.

| Event | When | Content |
|---|---|---|
| `CompanyCreated` | 8.1 | `companyId`, `applicationId`, `coreDataSource` |
| `CompanyIdentifierAssigned` | 8.2 | `companyId`, `bpn` |
| `CompanyActivated` | 8.3 | `companyId`, `bpn`, `companyRoles` |
| `CompanyRejected` | 8.4 | `companyId` |
| `CompanyDeactivated` | 7.13, 8.5 | `companyId`, `reason` (`MANUAL`, `TRANSFERRED`) |
| `CompanyReactivated` | 8.6 | `companyId`, `bpn`, `companyRoles` |
| `CompanyRolesChanged` | 7.8, 7.9 | `companyId`, `companyRoles` (all active after the change) |
| `CompanyDataChanged` | 7.2, 7.3, 7.12, 7.17, 8.6 | `companyId`, `changedFields` (field names, without values), `coreDataChanged`, `coreDataSource` |
| `CompanyChangeRequestSubmitted` | 7.4 | `companyId`, `changeRequestId` |
| `CompanyChangeRequestClosed` | 7.6, 7.17, 7.18 | `companyId`, `changeRequestId`, `outcome` (`APPROVED`, `DECLINED`, `WITHDRAWN`) |

The schemas are in [openapi.yaml](openapi.yaml) under `components/schemas` (names start with `Event`).

## 11. Calls to other modules and external systems

Every neighbour has a built-in stand-in (FRAMEWORK 4.7): empty address = stand-in. This way the module runs alone and can be tested without neighbours.

### 11.1 C1-M2 Registration (`REGISTRATION_URL`)

| Call | Response (fields used) | Built-in stand-in |
|---|---|---|
| `get_application(applicationId)`, over HTTP `GET /api/c1-m2/internal/applications/{applicationId}` | `applicationId`, `companyId`, `kind` (`ONBOARDING`, `REACTIVATION`, `TRANSFER`), `status`, `verificationMethod` (`ATTESTATION` or `DOCUMENTS`), with `ATTESTATION` the evaluated verification attestation with `jti` and `verifiedAt`; `company` (`legalName`, `shortName`, `address`, `identifiers` with `type`, `value`, `source`); `requestedCompanyRoles`; `consents` (each with `agreementId`, `status`, `consentedAt`, `consentedBy`; empty before submission) | reads `STUB_APPLICATIONS_FILE`; unknown ID: 404 |

Mapping: `legalName` → `name`; `address.streetName` → `street`, `address.streetNumber` → `houseNumber`, the other address fields with the same name; identifiers with source from the verification attestation → `BUSINESS_WALLET`, all others → `DECLARED`; `verificationMethod` `ATTESTATION` → `EBW`, `DOCUMENTS` → `MANUAL`.

### 11.2 C5-M1 EBW-Connector of the installation (`EBW_CONNECTOR_URL`)

| Call | Response | Built-in stand-in |
|---|---|---|
| `GET {EBW_CONNECTOR_URL}/jwks` | JWKS (RFC 7517) with the signing key of the verification attestations | keys from `STUB_EBW_JWKS_FILE`, no fetch; the expected `iss` is then `https://ebw-connector.example.org` |

Timeout 5 seconds, cache `JWKS_CACHE_SECONDS`. An unknown `kid` triggers a reload, at most once per minute. Not reachable without a valid cache: 502 `upstream-error`.

### 11.3 C3-M2 CSP Connection and CSP

No calls. The BPN comes as input of `assign-identifier`; Process Orchestration reads the core data for C3-M2 through 9.1.

### 11.4 Files of the installation

The module reads `SETTINGS_FILE`, `masterDataFile`, `COMPANY_ROLES_FILE` and `OSP_COMPANY_FILE` at start. Without `SETTINGS_FILE` it starts without the seed data check, without a dataspace name and with the reference data from 6.4.

## 12. Errors

Problem Details (RFC 9457), `type` = `urn:c3-m1:problem:<name>`. Additional fields: `retryable` for steps, `reason` with `attestation-invalid`, `fields` with `field-locked` and `company-mismatch`, `missing` with `consent-missing`, `errors` with `validation`.

| `type` | HTTP | When | `retryable` for steps |
|---|---|---|---|
| `validation` | 400 | input invalid | `false` |
| `attestation-invalid` | 400 | verification attestation not acceptable; reason in `reason` (6.7) | – |
| `document-invalid` | 400 | supporting document missing, too large, too many or wrong type | – |
| `consent-missing` | 400 | consent to terms of the role is missing (7.8) | – |
| `unauthorized` | 401 | token missing or invalid | – |
| `forbidden` | 403 | permission missing | – |
| `not-found` | 404 | company, application, change request, supporting document or role unknown; company outside the scope `company` | `false` |
| `invalid-state` | 409 | status does not allow the operation | `false` |
| `conflict` | 409 | company belongs to another application, application does not match the company | `false` |
| `field-locked` | 409 | field must not be changed (6.6) | – |
| `role-not-assignable` | 409 | role must not be assigned or removed this way | – |
| `last-role` | 409 | last active role with `assignableBy` `COMPANY` | – |
| `bpn-taken` | 409 | BPN already belongs to another company of the installation | `false` |
| `bpn-mismatch` | 409 | company already has a different BPN | `false` |
| `identifier-missing` | 409 | activation without BPN | `false` |
| `roles-without-consent` | 409 | `activate-company`: no requested role with complete consents | `false` |
| `company-mismatch` | 409 | EBWOID or country of the verification attestation or of the application does not match the company | `false` |
| `attestation-used` | 409 | verification attestation already used | – |
| `attestation-outdated` | 409 | verification attestation older than the state of the core data | – |
| `representation-not-confirmed` | 409 | representation not confirmed in the verification attestation | – |
| `attestation-required` | 409 | name or address with source `EBW` only with a verification attestation | – |
| `change-request-open` | 409 | already an open change request | – |
| `change-requests-disabled` | 409 | change requests with supporting documents switched off | – |
| `same-person` | 409 | approval of one's own change request | – |
| `upstream-error` | 502 | C1-M2 or JWKS of the EBW-Connector not reachable or unexpected response | `true` |

## 13. Acceptance

The module is done when these cases work with the built-in stand-ins for C1-M2 and the EBW-Connector (test data in section 14):

1. `create-company` for A1: U1 `REQUESTED` without BPN, source `EBW`, `verificationRef` equal to the `jti` from A1; `EBWOID` and `EUID` with source `BUSINESS_WALLET`, `VAT_ID` with `DECLARED`; role `ACTIVE_PARTICIPANT` `REQUESTED`; exactly one `CompanyCreated` with `coreDataSource` `EBW`.
2. `create-company` again with the same input: `DONE`, no second event, no second row. For U1 with A2: 409 `conflict`. For A5 (kind `REACTIVATION`): 409 `conflict`.
3. `activate-company` for U1 before `assign-identifier`: 409 `identifier-missing`.
4. `assign-identifier` for U1 with `BPNLIUK2REMLK3F4`: BPN set, exactly one `CompanyIdentifierAssigned`. Repetition: `DONE` without event. Different BPN for U1: 409 `bpn-mismatch`. The same BPN for U3: 409 `bpn-taken`.
5. `activate-company` for U1: `ACTIVE`, `ACTIVE_PARTICIPANT` active, consents with source `APPLICATION` and `consentedBy` of P1, one `CompanyActivated` with BPN and roles. Repetition: no second event. Neither the CSP nor C3-M2 is called.
6. U3 (A3 without consents) after `create-company` and `assign-identifier` with `BPNL00000000U3OZ`: `activate-company` 409 `roles-without-consent`, status stays `REQUESTED`.
7. `reject-company` for U3: `REJECTED`, `CompanyRejected`; repetition: `DONE` without event. `reject-company` for U1 (`ACTIVE`): 409 `invalid-state`.
8. Scope `company`: P1 reads U1 (200, `dataspaceName` `Catena-X`); P1 reads U2: 404; P3 reads U1: 404; P4 (without `view_company`): 403; unknown company: 404.
9. P2 (Member) reads U1 and its roles (200); `PATCH` by P2: 403.
10. `PATCH` by P1 with `shortName`: 200, `CompanyDataChanged` with `changedFields` `[shortName]` and `coreDataChanged` `false`, audit log entry with `sub` of P1. `name` in the body: 409 `field-locked`.
11. P1 presents T1: 200, name "Beispiel Maschinenbau AG", new address without house number and region, source `EBW`, `verificationRef` equal to the `jti` of T1, `CompanyDataChanged` with `coreDataChanged` `true`. T1 again: 409 `attestation-used`.
12. T2 (different EBWOID): 409 `company-mismatch` with `fields` `[EBWOID]`. T3 (representation not confirmed): 409 `representation-not-confirmed`. T4 (foreign key): 400 `attestation-invalid`, `reason` `signature`. T5 (`verified_at` three days ago): 400 `attestation-invalid`, `reason` `too-old`. T6 (`verified_at` before T1) after case 11: 409 `attestation-outdated`.
13. P1 submits a change request for U1 (source `EBW`) with a new name: 409 `attestation-required`. With a new `LEI_CODE` and a PDF: 201 `OPEN`, `CompanyChangeRequestSubmitted`. A second one: 409 `change-request-open`. Without a supporting document or with a text file named `auszug.pdf`: 400 `document-invalid`.
14. U2 (A2, supporting documents) goes through `create-company`, `assign-identifier` with `BPNL00000000U2LG` and `activate-company`: source `MANUAL`, `core_data_verified_at` set. P3 submits a change request with the name "Beispiel Logistik SE" and one supporting document. S1 sees it in `/osp/change-requests`, downloads the supporting document (200, `application/pdf`, audit log entry with `sub` of S1) and approves: name changed, source `MANUAL`, `CompanyDataChanged` with `coreDataChanged` `true`, `CompanyChangeRequestClosed` with `APPROVED`.
15. S1 declines the change request from case 13 without `reason`: 400 `validation`; with `reason`: `DECLINED`, `CompanyChangeRequestClosed` with `DECLINED`; P1 sees the justification in 7.5. P1 submits a new change request and withdraws it: `WITHDRAWN`, event; again: 200 without event.
16. S2 submits a change request for the OSP company (new `VAT_ID`) and approves it themselves: 409 `same-person`. S1 approves it: 200.
17. P1 calls `/osp/change-requests`: 403. S1 calls `PATCH /osp/companies/{U1}`: 403.
18. S2 corrects the address of U1 (source `EBW`) with `reason`: 200, source `MANUAL`, `verificationRef` empty, `CompanyDataChanged`. `countryCode` for U1 (`ACTIVE` with BPN): 409 `field-locked`.
19. S2 deactivates U1: `INACTIVE`, `deactivationReason` `MANUAL`, `CompanyDeactivated` with `reason` `MANUAL` without the justification text. Again: 200 without event. `PATCH` by P1 afterwards: 409 `invalid-state`.
20. `reactivate-company` for U1 with A5 (verification attestation, same EBWOID, address Musterstraße 7): `ACTIVE`, source `EBW`, `CompanyDataChanged` and `CompanyReactivated` with BPN and roles; roles unchanged. Repetition: `DONE` without events.
21. `deactivate-company` for U2 with `TRANSFERRED`: `INACTIVE`, `CompanyDeactivated` with `reason` `TRANSFERRED`; repetition without event. `reactivate-company` for U2 with A6 (kind `REACTIVATION`): 409 `invalid-state`. With A7 (kind `TRANSFER`, return): `assign-identifier` with `BPNL00000000U2LG` gives `DONE` without event, then `reactivate-company` `ACTIVE`, `CompanyReactivated`.
22. Transfer to this OSP: `create-company` for A8 (kind `TRANSFER`, Andere Beispiel AG), `assign-identifier` with `BPNL00000000002B`, `activate-company`: U4 `ACTIVE` with this BPN.
23. `GET /internal/companies?id=…` with U1, an unknown ID and U2: two `items` in the order of the request, one ID in `notFound`. With `id` and `bpn` together: 400.
24. `identifier-matches` with the VAT ID of U1 in a different spelling (`de 123.456.789`): match U1 with `matchedOn` `VAT_ID`; with `excludeCompanyId` = U1: no match.
25. Seed data: At first start the OSP company `11111111-1111-4111-8111-111111111111` is created, `ACTIVE` with `ONBOARDING_SERVICE_PROVIDER` and `BPNL00000000OSPA`, without events; a second start changes nothing. `OSP_COMPANY_FILE` with a `companyId` other than `ospCompanyId`: the start aborts with an error message.
26. Privacy: neither logs nor events nor `audit_log` contain a verification attestation, the content of a supporting document, a token or names of people.

## 14. Test data

Test IDs as in FRAMEWORK 7a; test person and test company as in C5-M1 and C3-M2. All companies and identifiers are invented.

**Installation settings** (`config/installation.json`):

```json
{
  "ospId": "22222222-2222-4222-8222-222222222222",
  "ospName": "OSP A",
  "ospCompanyId": "11111111-1111-4111-8111-111111111111",
  "dataspace": { "memberOf": "Catena-X", "name": "Catena-X" },
  "masterDataFile": "config/master-data.json"
}
```

**Tokens of the test Portal IAM** (`aud` contains `portal-core`):

| Label | `sub` | `company_id` | Permissions | Corresponds to |
|---|---|---|---|---|
| P1 | `5a1c0001-0000-4000-8000-000000000001` | U1 | `view_company`, `edit_company`, `edit_company_roles` | Erika Mustermann, Company Admin of U1 |
| P2 | `5a1c0002-0000-4000-8000-000000000002` | U1 | `view_company` | Member of U1 |
| P3 | `5a1c0003-0000-4000-8000-000000000003` | U2 | `view_company`, `edit_company`, `edit_company_roles` | Company Admin of U2 |
| P4 | `5a1c0004-0000-4000-8000-000000000004` | – | none of this module | `Applicant` |
| S1 | `5a1c0011-0000-4000-8000-000000000011` | OSP company | `view_companies`, `review_company_changes` | `OSP Reviewer` |
| S2 | `5a1c0012-0000-4000-8000-000000000012` | OSP company | `view_companies`, `manage_companies`, `review_company_changes`, `view_company`, `edit_company` | `OSP Admin`, also `Company Admin` of the OSP company |
| M1 | service account | – | `query_c3_m1`, `execute_steps_c3_m1` | other modules over HTTP |

**Companies:**

| Label | `companyId` | Name | Expected BPN |
|---|---|---|---|
| OSP company | `11111111-1111-4111-8111-111111111111` | OSP A Beispiel GmbH | `BPNL00000000OSPA` (seed data) |
| U1 | `c0000001-0000-4000-8000-000000000001` | Beispiel Maschinenbau GmbH | `BPNLIUK2REMLK3F4` (as the stand-in in C3-M2 derives it from the EBWOID) |
| U2 | `c0000002-0000-4000-8000-000000000002` | Beispiel Logistik GmbH | `BPNL00000000U2LG` |
| U3 | `c0000003-0000-4000-8000-000000000003` | Ohne Zustimmung GmbH | `BPNL00000000U3OZ` |
| U4 | `c0000004-0000-4000-8000-000000000004` | Andere Beispiel AG | `BPNL00000000002B` (as in C3-M2, previously at OSP B) |

**Applications** (`stub-applications.json`), all submitted or approved:

| Application | `applicationId` | Kind | Verification method | Company | Content |
|---|---|---|---|---|---|
| A1 | `a0000001-0000-4000-8000-000000000001` | `ONBOARDING` | `ATTESTATION`, `jti` `3f6c1e2a-7b4d-4e8f-9a51-2c0d6b8e4f17` | U1 | Musterstraße 1, 70569 Stuttgart, region `BW`, `DE`; `EBWOID` `EBWOID-DE-0000000001` and `EUID` `DEXX001T.HRB100001` from the verification attestation, `VAT_ID` `DE123456789` declared; role `ACTIVE_PARTICIPANT`; consents to both terms of the role, `consentedBy` P1 |
| A2 | `a0000002-0000-4000-8000-000000000002` | `ONBOARDING` | `DOCUMENTS` | U2 | Hafenstraße 5, 68159 Mannheim, `BW`, `DE`; `COMMERCIAL_REG_NUMBER` `HRB 200002`, `VAT_ID` `DE987654321`; role `ACTIVE_PARTICIPANT` with consents, `consentedBy` P3 |
| A3 | `a0000003-0000-4000-8000-000000000003` | `ONBOARDING` | `DOCUMENTS` | U3 | Ringstraße 3, 76131 Karlsruhe, `DE`; `VAT_ID` `DE111111111`; role `ACTIVE_PARTICIPANT`, no consents |
| A5 | `a0000005-0000-4000-8000-000000000005` | `REACTIVATION` | `ATTESTATION`, new `jti`, `verifiedAt` at test time | U1 | as A1, address Musterstraße 7 |
| A6 | `a0000006-0000-4000-8000-000000000006` | `REACTIVATION` | `DOCUMENTS` | U2 | as A2 |
| A7 | `a0000007-0000-4000-8000-000000000007` | `TRANSFER` | `DOCUMENTS` | U2 | as A2 (return) |
| A8 | `a0000008-0000-4000-8000-000000000008` | `TRANSFER` | `ATTESTATION` | U4 | Beispielweg 2, 80331 München, `BY`, `DE`; `EBWOID` `EBWOID-DE-0000000002`; role `ACTIVE_PARTICIPANT` with consents |

**Verification attestations for 7.3**, created at test time and signed with the test key from `stub-ebw-jwks.json` (private part only in the test data). Content as the example in C5-M1 14.5 with `iss` `https://ebw-connector.example.org`, `aud` `portal-registration`, `client_id` `demo-osp`, `purpose` `onboarding`, own `jti`. Differences:

| Label | Difference |
|---|---|
| T1 | `verified_at` one hour ago; `legal_name` "Beispiel Maschinenbau AG"; address Industriestraße 10, 70565 Stuttgart |
| T2 | as T1, `ebwoid` `EBWOID-DE-0000000009` |
| T3 | as T1, `representation.confirmed` `false`, `result` `NOT_CONFIRMED` |
| T4 | as T1, signed with a key that is not in `stub-ebw-jwks.json` |
| T5 | as T1, `verified_at` three days ago |
| T6 | as T1, `verified_at` two hours ago |

**Further files:**

- `company-roles.json`: initial values from 6.5, descriptions `de` and `en`.
- `osp-company.json`: `11111111-1111-4111-8111-111111111111`, "OSP A Beispiel GmbH", Am Prüfstand 1, 70569 Stuttgart, `DE`, BPN `BPNL00000000OSPA`, identifier `VAT_ID` `DE222222222`, roles `ONBOARDING_SERVICE_PROVIDER` and `ACTIVE_PARTICIPANT`.
- Supporting documents: a small valid PDF file `auszug.pdf`, a text file with the same name.

## 15. Changes from version 0.1

- **OSP 2.0:** The module runs in the installation of an OSP. All fields, headers and rules for Portal Service Providers (PSP) are dropped: `registering_psp_id`, `psp_id` in consents, delegation scope `company`, check at C2-M4, headers `X-On-Behalf-Of` and `X-Acting-Person`, error `not-delegated`, reading by the registering PSP. People call with their own token; the acting person is in the token (`sub`).
- **Permissions with scope:** `view_company`, `edit_company`, `edit_company_roles` in the scope `company`; new `view_companies` and `review_company_changes`, plus `manage_companies` in the scope `osp`. Other companies give 404. Paths for OSP staff under `/osp/...` instead of `/operator/...`. The role catalogue for people (`GET /company-roles`) is dropped; people see it through 7.7 or through C1-M2.
- **Company identifier:** The adapter to the master data service, the background polling and all settings for them are dropped. The new step `assign-identifier` takes over the BPN from `CompanyRegisteredAtCsp` (C3-M2) and replaces `request-identifier`. Dropped: the status of the company identifier (`identifierStatus`), the BPN from the application, the BPN set by hand (`PUT /operator/companies/{companyId}/bpn`) and the event `CompanyIdentifierFailed`; C3-M2 reports a failure with `CompanyRegistrationAtCspFailed`. New error `bpn-mismatch`.
- **Membership:** Reporting the membership in `activate-company` is dropped; registering and deregistering the membership is the job of C3-M2. The dataspace comes from the installation settings (`dataspaceName` in 7.1).
- **Core data (OP-09, OP-10):** Source of the core data `EBW` or `MANUAL` with time of the last verification. Change with a new verification attestation of the EBW-Connector of the installation (7.3, same EBWOID, immediate) or by change request with supporting documents and approval by OSP staff (7.4 to 7.6, 7.14 to 7.18). The address is part of the core data; the Company Admin directly changes only the short name. Passing data to the master data service is dropped; changed core data is marked in the event (`coreDataChanged`).
- **Status and flows:** new steps `deactivate-company` (reasons `MANUAL`, `TRANSFERRED`) and `reactivate-company` with event `CompanyReactivated`; `CompanyDeactivated` names the reason. `INACTIVE` is no longer final. The reason `NO_PSP` (OP-06) is dropped with OSP 2.0. `create-company` and `activate-company` also apply to a transfer to this OSP.
- **Company roles:** `PORTAL_SERVICE_PROVIDER` and `OPERATOR` are dropped, as are `assignableBy` `FLOW` and the step `assign-company-role` (and the planned `remove-company-role`). New: `assignableBy` `COMPANY` (instead of `PSP`) and the seed role `ONBOARDING_SERVICE_PROVIDER` for the OSP company. The Company Admin declares consent personally; `consentedAt` as input is dropped.
- **Seed data:** `OSP_COMPANY_FILE` with the OSP company instead of `OPERATOR_COMPANY_FILE`; the ID must be `ospCompanyId` from the settings.
- **Identifiers:** types and patterns from the shared file of the reference data (OP-13); EUID and EBWOID with loose validation (OP-12).
- **New events:** `CompanyChangeRequestSubmitted`, `CompanyChangeRequestClosed`. `CompanyDataChanged` with `coreDataChanged` and `coreDataSource`; `CompanyCreated` with `coreDataSource` instead of `registeringPspId`.
- **Queries:** `CompanyInternal` returns address, identifiers, source of the core data and reason of the deactivation; `registeringPspId` and `identifierStatus` are dropped. The duplicate search 9.3 complements the duplicate check at the CSP.
