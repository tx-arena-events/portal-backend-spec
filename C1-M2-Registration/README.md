# C1-M2 Registration

**Version:** 0.2, as of 2026-10-30
**Capability:** C1 Onboarding
**Kind:** Hackathon
**Files:** this README (behaviour), [openapi.yaml](openapi.yaml) (interface), [open-points.md](open-points.md)

This README and the OpenAPI description are sufficient to build the module. The framework in [`../FRAMEWORK.md`](../FRAMEWORK.md) is binding as well: architecture, conventions, flows, catalogue of events and steps. Paths and field names are in English.

## 1. Purpose

The module takes the application of a company at the onboarding service provider (OSP). A person files the application personally: the person signs in to the portal of the OSP with their EUDI Wallet and calls the module with their own token (FRAMEWORK 5.2). With a Business Wallet, a verification attestation of the EBW-Connector proves the company and the representation; without a Business Wallet, the person uploads supporting documents. The module

- checks the verification attestation: signature, issuer, validity, single use, binding to the signed-in person,
- compares the company data of the application with the verification attestation,
- accepts supporting documents if the OSP allows the path without a Business Wallet,
- assigns the future company ID (`companyId`) or, for reactivation and return, takes over the existing one; C3-M1 keeps the record of the company,
- records the requested company roles and the consents of the person,
- reports "application created", "submitted", "withdrawn" and "completed", and closes the application after a decline,
- keeps the selectable company roles and the versions of the terms; it reads countries and identifier types from the shared reference data of the installation.

Application kinds: `ONBOARDING` (new company), `REACTIVATION` (deactivated company of this installation, FRAMEWORK 5.5), `TRANSFER` (transfer from another OSP, FRAMEWORK 5.6).

The module has no user interface. Callers:

| Who | What for |
|---|---|
| applicant (portal role `Applicant`) | read reference data and terms; create, change, read their own application, replace the verification attestation, upload supporting documents, submit, withdraw |
| OSP staff (`OSP Reviewer`, `OSP Admin`) | read all applications and supporting documents |
| OSP staff (`OSP Admin`) | read and create versions of the terms |
| Process Orchestration | steps `complete-application` and `close-application` |
| C1-M3 Application Review | read the application with verification attestation, comparison, supporting documents and consents (query) |
| C3-M1 Company Profile | read the company data of the application to create the company; read terms (queries) |

## 2. Terms

| Term | Meaning |
|---|---|
| Application | application of a company at the OSP. It belongs to the person who created it (`sub` in the token). |
| Applicant | person who creates and submits the application. After the approval, C2-M2 binds their account to the company (`assign-company-admin`). |
| Application kind | `ONBOARDING`, `REACTIVATION` or `TRANSFER` (6.2) |
| Verification path | `EBW`: with Business Wallet and verification attestation; `DOCUMENTS`: without Business Wallet, with supporting documents (6.9) |
| Verification attestation | JWT signed by the EBW-Connector (C5-M1) about an onboarding: extract of the person, data of the company from the LPID, result of the representation check (C5-M1 section 9; here 6.4) |
| Recognised issuer | the EBW-Connector of its own installation (`ATTESTATION_ISSUER`) |
| Representative | person who may act for the company according to the mandate; in the verification path `EBW` the same person as the applicant (6.4 step 10) |
| Supporting document | uploaded document in the verification path `DOCUMENTS`, for example a commercial register extract or a power of attorney |
| Identifier | pair of identifier type and value, e.g. `VAT_ID` and `DE123456789` |
| EBWOID | identifier of the owner of a Business Wallet, from the LPID; identifier type `EBWOID` |
| EUID | European identifier of a company in the commercial register (European Unique Identifier); identifier type `EUID` |
| Comparison | comparison of the company data of the application with the verification attestation, with one result per field (6.5) |
| Company role | role of the company in the dataspace, e.g. `ACTIVE_PARTICIPANT` |
| Terms | document the person must consent to: terms of the OSP or of the dataspace. Key and version, e.g. `terms-active-participant` in version `1.0`, with one document per language (6.8) |
| Existing company | company that C3-M1 already keeps; reference of an application of kind `REACTIVATION` and of a return (`TRANSFER` with reference, 6.2) |
| Open application | application in status `CREATED` or `SUBMITTED` |

## 3. Not part of this module

| What | Where |
|---|---|
| Check the credentials of the wallets (OpenID4VP), issue the verification attestation | C5-M1 EBW-Connector |
| Sign-in, account of the person, portal roles | C2-M5 Portal IAM, C2-M2 |
| Bind the account to the company after the approval | C2-M2, step `assign-company-admin` |
| Review case, automatic pre-check, duplicate check in the installation and at the CSP, decision | C1-M3 Application Review |
| Create, activate, reactivate, decline the record of the company; company identifier (BPN) | C3-M1 Company Profile |
| Registration and transfer at the CSP | C3-M2 CSP Connection |
| Consent of existing companies to new versions of the terms | C3-M1 (assumption, see open-points.md) |
| Message to the person (submitted, declined, active) | notifications (other part), triggered by the events |
| Dataspace wallet and credentials | C4-M2, C4-M1 |
| Order of the steps, restart | Process Orchestration |
| Maintain countries and identifier types | shared reference data of the installation (FRAMEWORK 4.8) |

## 4. Technical frame

- Python with FastAPI, part of the modular monolith. Process Orchestration and other modules call steps and queries in the same process through the public interface of the module. The HTTP paths in [openapi.yaml](openapi.yaml) describe the same contract.
- Own database `c1_m2` in PostgreSQL with its own database user. Other modules read only through the queries in section 9.
- Every HTTP call of a person carries their token from the Portal IAM of the OSP. The shared library validates it and returns the claims: `sub`, `company_id` (only for a bound account), `given_name`, `family_name` and the permissions (FRAMEWORK 4.2).
- Events go out through the outbox of the base skeleton, stored in the same transaction as the change.
- The module checks verification attestations itself with a JOSE library, e.g. `joserfc` or `PyJWT` with `cryptography`. It fetches the keys from the JWKS address of the recognised issuer and caches them for `ATTESTATION_JWKS_CACHE_SECONDS`. A `kid` that is missing in the cache triggers one reload.
- Supporting documents are stored in its own database (`bytea`), each at most `DOCUMENT_MAX_BYTES`.
- Personal data: `sub` and first and last name of the applicant (from the token), first and last name of the representative (from the verification attestation), supporting documents. No email addresses, no birth date.
- The verification attestation and the content of supporting documents never appear in the log, neither in whole nor in part.
- The module reads the shared installation settings (FRAMEWORK 4.8) at start from `SETTINGS_FILE`; of these it uses `masterDataFile`.

### Configuration

| Variable | Meaning | Example |
|---|---|---|
| `DATABASE_URL` | own database `c1_m2` | |
| `IAM_URL`, `IAM_REALM` | Portal IAM of the OSP for token validation | `https://iam.osp-a.example.org/auth`, `osp-a` |
| `SETTINGS_FILE` | installation settings (FRAMEWORK 4.8); from these `masterDataFile` with countries and identifier types (structure in 14.2) | `config/installation.json` |
| `CATALOG_FILE` | initial values for company roles and terms, taken over at start (5, structure in 14.3) | `config/registration-catalog.json` |
| `ATTESTATION_ISSUER` | `iss` of the EBW-Connector of the installation (`PUBLIC_URL` in C5-M1); the only recognised issuer | `https://ebw.osp-a.example.org` |
| `ATTESTATION_JWKS_URL` | JWKS of the issuer; empty = `{ATTESTATION_ISSUER}/jwks` | |
| `ATTESTATION_JWKS_FILE` | tests only: JWKS as a file; set = no fetch | `test-ebw-jwks.json` |
| `ATTESTATION_AUDIENCE` | value that `aud` of the verification attestation must contain (equal to `ATTESTATION_AUDIENCE` in C5-M1) | `portal-registration` |
| `ATTESTATION_ALGORITHMS` | allowed signature algorithms, comma-separated | `ES256` |
| `ATTESTATION_CLIENT_IDS` | allowed values of `client_id` in the verification attestation (client of the UI at the EBW-Connector), comma-separated; empty = no check | `osp-a-ui` |
| `ATTESTATION_MAX_AGE` | maximum age of the verification attestation at submission, counted from `iat` (ISO 8601) | `P30D` |
| `ATTESTATION_CLOCK_SKEW_SECONDS` | tolerance when comparing points in time | `60` |
| `ATTESTATION_JWKS_CACHE_SECONDS` | cache time of the keys | `300` |
| `DOCUMENTS_ENABLED` | allow the verification path without Business Wallet (6.9) | `true` |
| `DOCUMENT_MAX_BYTES` | maximum size of a supporting document | `10485760` |
| `DOCUMENT_MAX_COUNT` | maximum number of supporting documents per application | `10` |
| `COMPANY_URL` | C3-M1: empty = built-in stand-in, `in-process` = call in the same process, otherwise HTTP address (FRAMEWORK 4.7) | `in-process` |
| `REVIEW_URL` | C1-M3: like `COMPANY_URL` | `in-process` |
| `STUB_COMPANIES_FILE` | file for the stand-in of C3-M1 (14.5) | `stub-companies.json` |
| `STUB_REVIEWS_FILE` | file for the stand-in of C1-M3 (14.5) | `stub-reviews.json` |

## 5. Data

### Table `application`

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key; `applicationId` |
| `company_id` | uuid | newly generated or reference to an existing company (6.2); at most one open application per `company_id` (partial unique index) |
| `kind` | text | `ONBOARDING`, `REACTIVATION`, `TRANSFER` (6.2); immutable |
| `existing_company` | boolean | `true` if `company_id` is an existing company |
| `verification` | text | `EBW` or `DOCUMENTS` (6.9); immutable |
| `applicant_sub` | text | `sub` from the token at creation; owner; at most one open application per person (partial unique index) |
| `applicant_given_name`, `applicant_family_name` | text | from the token at creation; for the manual review in the verification path `DOCUMENTS` and for display |
| `status` | text | `CREATED`, `SUBMITTED`, `WITHDRAWN`, `DECLINED`, `COMPLETED` (6.1) |
| `legal_name` | text | 1 to 160 characters |
| `short_name` | text | up to 60 characters, empty allowed |
| `street_name` | text | 1 to 150 characters |
| `street_number` | text | up to 20 characters, empty allowed |
| `postal_code` | text | up to 20 characters, empty allowed |
| `city` | text | 1 to 150 characters |
| `region` | text | code of the region according to ISO 3166-2 without the country part, `^[A-Z0-9]{1,3}$`, empty allowed |
| `country_code` | text | ISO 3166-1 alpha-2, from the reference data |
| `company_roles` | text[] | at least one selectable company role, without duplicates |
| `comparison` | jsonb | result of the comparison per field (6.5); empty in the verification path `DOCUMENTS` |
| `required_agreements` | jsonb | required terms with version, recorded at submission; empty before |
| `representation_declared` | boolean | only `DOCUMENTS`: declaration of the person at submission that they may represent the company |
| `decline_reason` | text | reason for the decline, set by `close-application` |
| `created_at`, `changed_at`, `submitted_at`, `withdrawn_at`, `closed_at`, `completed_at` | timestamptz | |

### Table `application_identifier`

| Field | Type | Rule |
|---|---|---|
| `application_id` | uuid | part of the primary key |
| `type` | text | identifier type (6.6); part of the primary key, so at most one per type |
| `value` | text | normalised (6.6) |
| `source` | text | `ATTESTATION` (from the verification attestation) or `APPLICANT` (given by the person) |

### Table `attestation`

Every accepted verification attestation, also replaced ones. Per application at most one is current; in the verification path `DOCUMENTS` there is none.

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key |
| `application_id` | uuid | required |
| `jti` | text | identifier of the verification; unique across all applications |
| `issuer` | text | `iss` |
| `key_id` | text | `kid` from the header |
| `client_id` | text | client of the transaction at the EBW-Connector |
| `raw_jwt` | text | the verification attestation as received; evidence |
| `issued_at`, `expires_at`, `verified_at` | timestamptz | from `iat`, `exp`, `verified_at` |
| `person_given_name`, `person_family_name` | text | extract of the representative |
| `company_claims` | jsonb | data of the company from the verification attestation (claim `company`) |
| `representation` | jsonb | claim `representation`: `confirmed`, `result`, `type`, `role`, `valid_until`, `issuer`, `trust`, `reasons` |
| `evidence` | jsonb | claim `evidence`: per credential `type`, `vct`, `issuer`, `wallet`, `status_checked` |
| `is_current` | boolean | `true` for the current one; at most one per application (partial unique index) |
| `received_at` | timestamptz | |

### Table `document`

| Field | Type | Rule |
|---|---|---|
| `id` | uuid | primary key; `documentId` |
| `application_id` | uuid | required |
| `document_type` | text | `COMMERCIAL_REGISTER_EXTRACT`, `POWER_OF_ATTORNEY`, `OTHER` (6.9) |
| `file_name` | text | 1 to 200 characters, without path parts and control characters |
| `content_type` | text | `application/pdf`, `image/png` or `image/jpeg` |
| `size_bytes` | integer | 1 to `DOCUMENT_MAX_BYTES` |
| `sha256` | text | checksum of the content, hexadecimal |
| `content` | bytea | content; unchanged |
| `uploaded_by_sub`, `uploaded_at` | text, timestamptz | |

### Table `consent`

| Field | Type | Rule |
|---|---|---|
| `application_id` | uuid | part of the primary key |
| `agreement_key` | text | part of the primary key |
| `agreement_version` | text | version that was consented to |
| `language` | text | language of the document the person read: `de` or `en` |
| `given_at` | timestamptz | time of submission |
| `given_by_sub` | text | `sub` of the person at submission |
| `attestation_id` | uuid | current verification attestation at submission (only `EBW`); links the consent to the verified representative |

### Company roles and terms

Initial values from `CATALOG_FILE`, taken over at start: missing roles and missing versions are created, existing roles are aligned. The start never changes an existing version. If the file brings a new version of a key, it becomes active and the previous one `INACTIVE` (6.8).

| Table | Fields |
|---|---|
| `company_role` | `role` (primary key), `name_de`, `name_en`, `description_de`, `description_en`, `selectable` (selectable at registration) |
| `agreement` | `key`, `version` (together primary key), `party` (`OSP` or `DATASPACE`), `title_de`, `title_en`, `document_url_de`, `document_url_en` (at least one), `scope` (`ALL` or `COMPANY_ROLE`), `status` (`ACTIVE`, `INACTIVE`), `reconsent_required` (boolean), `reconsent_due_at` (timestamptz, empty allowed), `created_at`, `created_by_sub` (empty for initial values) |
| `company_role_agreement` | `role`, `agreement_key` |

Countries and identifier types are not in the database. The module reads them at start from `masterDataFile` into memory (OP-13).

### Table `audit_log`

Time, `sub` (empty for steps), `company_id`, operation, `application_id`, object ID (for example `documentId` or key and version of terms); for steps `processId` and `stepRunId`. Without verification attestation, without company data, without names.

### Table `outbox`

As defined by the base skeleton.

**Not stored:** email addresses, birth dates, further people of the company, accounts, roles and rights (Portal IAM, C2-M2), the record of the company and its BPN (C3-M1), results of the Application Review (C1-M3), countries and identifier types (shared reference data).

## 6. Rules

### 6.1 Status

| From | Trigger | To | Event |
|---|---|---|---|
| – | create (7.2) | `CREATED` | `ApplicationCreated` |
| `CREATED` | change (7.5), replace verification attestation (7.6), upload or delete supporting document (7.7, 7.9) | `CREATED` | – |
| `CREATED` | submit (7.10) | `SUBMITTED` | `ApplicationSubmitted` |
| `CREATED` | withdraw (7.11) | `WITHDRAWN` | – |
| `SUBMITTED` | withdraw (7.11), review case not yet decided | `WITHDRAWN` | `ApplicationWithdrawn` |
| `SUBMITTED` | step `close-application` (8.2) | `DECLINED` | – |
| `SUBMITTED` | step `complete-application` (8.1) | `COMPLETED` | `ApplicationCompleted` |

`WITHDRAWN`, `DECLINED` and `COMPLETED` are final. From `SUBMITTED` on, data, roles, verification attestation and supporting documents can no longer be changed. After a final status the person files a new application; for `ONBOARDING` the company gets a new `companyId`.

The module keeps only the states it sets with its own operations and steps. C1-M3 keeps review and approval in the review case. The overall view:

| Overall view | Status in C1-M2 | Review case in C1-M3 |
|---|---|---|
| created | `CREATED` | – |
| submitted, under review | `SUBMITTED` | `OPEN`, `AWAITING_SECOND_APPROVAL` |
| approved, setup running | `SUBMITTED` | `APPROVED` |
| declined | `DECLINED` | `DECLINED` |
| withdrawn | `WITHDRAWN` | – or `CLOSED` |
| completed (company active) | `COMPLETED` | `APPROVED` |

`GET /applications/{applicationId}` and `GET /osp/applications/{applicationId}` also return the status of the review case as `reviewStatus` (7.4).

### 6.2 Application kinds

| Kind | Reference (`companyId` or `bpn` at creation) | Company in C3-M1 | `companyId` of the application | Company roles |
|---|---|---|---|---|
| `ONBOARDING` | not allowed | – | new | required |
| `REACTIVATION` | required | status `INACTIVE`, reason of the deactivation not `TRANSFERRED` | existing | optional; if not given, the selectable roles of the company from C3-M1 |
| `TRANSFER` without reference | – | – (the installation does not know the company) | new | required |
| `TRANSFER` with reference (return) | `companyId` or `bpn` | status `INACTIVE`, reason `TRANSFERRED` | existing | optional as for `REACTIVATION` |

- `companyId` and `bpn` must not both be given. If a required reference is missing or a forbidden one is given: 400 `validation`.
- With a reference, the module reads the company at C3-M1 (11.2), with `bpn` through the search by BPN. Unknown, or status or reason does not fit: 409 `company-not-eligible` with `companyStatus` (missing for an unknown company). C3-M1 not reachable: 502 `upstream-error`.
- With a reference, the company of the application must be the existing one: if C3-M1 keeps an `EBWOID`, it equals `company.ebwoid` of the verification attestation; if C3-M1 keeps an `EUID` and the application has one, both are equal. Otherwise 400 `attestation-mismatch` with the fields. In the verification path `DOCUMENTS` this applies to the `EUID` that the person gives.
- For an existing company there is at most one open application. If there already is one: 409 `company-has-open-application`.
- Whether the company at the CSP fits the application kind (another home OSP for `TRANSFER`, this OSP for `REACTIVATION`) is checked by C1-M3 in the automatic pre-check, not by this module.
- `kind` and the reference are immutable after creation.

### 6.3 Ownership, visibility and permissions

- An application belongs to the person whose `sub` was in the token at creation (scope `own`, FRAMEWORK 4.3). Through the paths under `/applications` only this person reads, changes, submits and withdraws it. For other people the module answers with 404.
- **Create only without a company.** If `company_id` is in the token, the account is already bound to a company: 403 `account-bound`. After the approval, C2-M2 binds the account of the applicant to the new company; an account belongs to at most one company (FRAMEWORK 5.3). Reading own applications also works with a bound account.
- **One open application per person.** If the person already has an open application: 409 `open-application-exists` with `existingApplicationId` (6.11).
- OSP staff read all applications through the paths under `/osp` (scope `osp`).

| Permission | Scope | Operations | Portal role (proposal for C2-M5) |
|---|---|---|---|
| `register_company` | `own` | read reference data and terms (7.1); create, list, read, change own application, replace verification attestation, upload, read and delete supporting documents, submit, withdraw (7.2 to 7.11) | `Applicant` |
| `view_applications` | `osp` | read all applications and supporting documents (7.12 to 7.14) | `OSP Reviewer`, `OSP Admin` |
| `manage_agreements` | `osp` | read and create versions of the terms (7.15, 7.16) | `OSP Admin` |
| `execute_steps_c1_m2` | `osp` | steps (section 8), only for HTTP calls | service account |
| `query_c1_m2` | `osp` | queries (section 9), only for HTTP calls | service account |

Reference data and terms (7.1) belong to nobody; there the scope check does not apply. For steps and queries, `osp` means: applies to all applications of the installation, without reference to a person. In the same process they run without a token.

### 6.4 Verification attestation

**Structure.** The verification attestation is a JWS in compact serialisation (RFC 7515), at most 16 KB, according to C5-M1 section 9.1 in version `0.2`.

| Part | Claim | Required | Content |
|---|---|---|---|
| Header | `alg` | yes | one of `ATTESTATION_ALGORITHMS` |
| Header | `kid` | yes | key of the issuer |
| Header | `typ` | yes | `ebw-attestation+jwt` |
| Payload | `iss` | yes | equal to `ATTESTATION_ISSUER` |
| Payload | `sub` | yes | equal to `company.ebwoid` |
| Payload | `aud` | yes | contains `ATTESTATION_AUDIENCE` (string or list) |
| Payload | `iat`, `exp` | yes | issuance, end of validity |
| Payload | `jti` | yes | identifier of the verification, 1 to 200 characters |
| Payload | `attestation_version` | yes | `"0.2"` |
| Payload | `purpose` | yes | `"onboarding"` |
| Payload | `verified_at` | yes | time of the last step in the EBW-Connector |
| Payload | `client_id` | yes | client of the transaction at the EBW-Connector |
| Payload | `person.given_name`, `person.family_name` | yes | extract of the representative from the PID |
| Payload | `company.legal_name`, `company.ebwoid`, `company.country` | yes | data of the company from the LPID; country according to ISO 3166-1 alpha-2 |
| Payload | `company.euid` | no | EUID |
| Payload | `company.address.street_address`, `.postal_code`, `.locality`, `.country` | no | address |
| Payload | `representation.confirmed` | yes | `true` if and only if the EBW-Connector confirmed the representation |
| Payload | `representation.result`, `.type`, `.role`, `.valid_until`, `.issuer`, `.trust`, `.reasons` | no | details of the representation check (C5-M1 6.5) |
| Payload | `evidence` | yes | list of the checked credentials with `type`, `vct`, `issuer`, `wallet`, `status_checked` |

The module stores `representation` and `evidence` in full for display in the review case; it does not decide on them. Further claims are ignored. An example is in 14.4.

**Check at creation and replacement.** In this order; the first failure determines the `reason` in the 400 `attestation-invalid` response:

1. Compact JWS serialisation, header readable; otherwise `format`.
2. `alg` allowed; `none` is never allowed. `typ` is correct. Otherwise `algorithm`.
3. `iss` equal to `ATTESTATION_ISSUER`; otherwise `issuer`.
4. Key for `kid` from the JWKS of the issuer (`ATTESTATION_JWKS_URL` or `ATTESTATION_JWKS_FILE`); signature valid. Otherwise `signature`. Header parameters such as `jku`, `x5u` or `jwk` are ignored; keys come only from the configuration. If the JWKS cannot be fetched: 502 `upstream-error`.
5. `aud` contains `ATTESTATION_AUDIENCE`; otherwise `audience`.
6. `exp` is after now minus tolerance, `iat` is not after now plus tolerance; otherwise `expired` or `not-yet-valid`.
7. `attestation_version` is `"0.2"`; otherwise `version`. `purpose` is `"onboarding"`; otherwise `purpose`.
8. Required claims present and in the correct format; `sub` equal to `company.ebwoid`; `company.country` a country of the reference data. Otherwise `claims`.
9. `ATTESTATION_CLIENT_IDS` not empty: `client_id` is in it; otherwise `client`.
10. **Person:** `person.given_name` and `person.family_name` are equal to `given_name` and `family_name` in the token, after normalisation as in C5-M1 6.5 (Unicode NFC, leading and trailing spaces removed, case ignored). Otherwise `person`. If one of the claims is missing in the token: also `person`.

**Why step 10 is a failure and not a hint.** The verification attestation proves that the representative may act for the company. After the approval, the applicant becomes Company Admin. Step 10 ensures that both are the same person. It replaces the binding of the verification attestation to a PSP from version 0.1 (`psp_id`, OP-17). A hint to C1-M3 would not suffice: with automatic approval nobody reads it. Both names come from the same PID of the EUDI Wallet (sign-in and verification attestation through the same EBW-Connector); normally no difference arises.

**Single use.** A `jti` is valid for exactly one application. A replaced verification attestation and that of a withdrawn application also stay used. Second use: 409 `attestation-used`.

**At submission.** The current verification attestation must not be expired (`exp`) and not older than `ATTESTATION_MAX_AGE` from `iat`. Otherwise 409 `attestation-expired`; the person fetches a new one from the EBW-Connector and replaces it (7.6).

**Representation.** The module also accepts a verification attestation with `representation.confirmed: false`. C1-M3 does the assessment in the automatic pre-check.

### 6.5 Company data and comparison

Applies in the verification path `EBW`. The data from the verification attestation is the basis. The person may omit fields; the module then takes them from the verification attestation.

| Comparison field (`field`) | Field in the application | Claim | Kind |
|---|---|---|---|
| `EBWOID` | identifier `EBWOID` | `company.ebwoid` | fixed |
| `EUID` | identifier `EUID` | `company.euid` | fixed, if the claim is present |
| `COUNTRY` | `address.countryCode` | `company.country` | fixed |
| `LEGAL_NAME` | `legalName` | `company.legal_name` | compared |
| `STREET` | `address.streetName` and `address.streetNumber`, joined with a space | `company.address.street_address` | compared |
| `POSTAL_CODE` | `address.postalCode` | `company.address.postal_code` | compared |
| `CITY` | `address.city` | `company.address.locality` | compared |

- **Fixed fields** must be equal. If a value of the person differs: 400 `attestation-mismatch` with the fields in `fields`. Identifiers `EBWOID` and `EUID` from the verification attestation get `source` `ATTESTATION`. If `company.euid` is missing, the person may give an EUID (`source` `APPLICANT`).
- **Compared fields** may differ. The module records the result; C1-M3 assesses it.
- Results per field:

| Result | Meaning |
|---|---|
| `FROM_ATTESTATION` | value taken from the verification attestation |
| `MATCH` | value of the person matches the verification attestation |
| `MISMATCH` | value of the person differs (only for compared fields) |
| `NOT_IN_ATTESTATION` | claim missing; the value of the person is not proven |

- If a field is missing on both sides (for example no EUID), its entry in `comparison` is omitted.
- **Comparison:** both values in Unicode normalisation form NFKC, leading and trailing whitespace removed, inner whitespace collapsed to one space, case ignored (`casefold`). No further alignment; "Str." and "Straße" count as different.
- If the address is missing in the verification attestation, the person must give `streetName` and `city`, otherwise 400 `validation`.
- The module compares again when a field changes (7.5) or the verification attestation is replaced (7.6).
- The data at submission is decisive. From `SUBMITTED` on, it is fixed. That is why C3-M1 creates a new company only after `ApplicationSubmitted`.

In the verification path `DOCUMENTS` all data comes from the person; `comparison` stays empty, all identifiers have `source` `APPLICANT` (6.9).

### 6.6 Identifiers

| Identifier type | Name | Countries (initial values) |
|---|---|---|
| `EUID` | EUID (commercial register, EU) | EU and EEA |
| `EBWOID` | EBWOID (Business Wallet) | EU and EEA |
| `COMMERCIAL_REG_NUMBER` | commercial register number | as in portal 2.6.0 |
| `VAT_ID` | VAT identification number | as in portal 2.6.0 |
| `LEI_CODE` | Legal Entity Identifier | as in portal 2.6.0 |
| `EORI` | EORI number | all countries |

- Per application at most one identifier per type, at most 10 identifiers.
- Verification path `EBW`: `EBWOID` is required; it comes from the verification attestation. Verification path `DOCUMENTS`: `EBWOID` is not allowed, because without a Business Wallet nothing proves it; at least one other identifier is required. Otherwise 400 `validation`.
- Only identifier types that the reference data assigns to the country of the company are allowed. Otherwise 400 `validation`.
- Format: regular expression per country from `masterDataFile`, otherwise the expression with country `Worldwide`. Initial values from portal 2.6.0 (`ValidationExpressions.cs`), extended by `EUID` and `EBWOID`; extract in 14.2. EUID and EBWOID are checked only loosely (length, allowed characters, OP-12). For `EUID` in addition: the first two characters are the country of the company.
- Normalisation before check and storage: remove leading and trailing whitespace; `EUID`, `VAT_ID`, `LEI_CODE`, `EORI` in upper case.
- The current identifier type `VIES` is dropped; `EUID` takes its place.

### 6.7 Company roles

- Company roles with `selectable = true` are selectable. Initial values: `ACTIVE_PARTICIPANT`, `APP_PROVIDER`, `SERVICE_PROVIDER`. `ONBOARDING_SERVICE_PROVIDER` is not selectable (only the company of the OSP, C3-M1). The roles `PORTAL_SERVICE_PROVIDER` and `OPERATOR` are dropped with OSP 2.0.
- The role names are the same as in the role catalogue of C3-M1 (C3-M1 6.5). This module keeps the assignment of terms to roles; C3-M1 reads it through 9.3.
- If the module takes the roles of an existing company from C3-M1 (6.2), it takes only the selectable ones. If none remains: 400 `validation`; the person then gives the roles personally.

### 6.8 Terms and consents

- A **terms document** has a key, a party (`OSP`: terms of the OSP; `DATASPACE`: terms of the dataspace) and versions. A version is immutable and has one document per language (`de`, `en`, at least one). Per key exactly one version is active.
- **Required terms** of an application: all active terms with `scope` `ALL` and all active terms assigned to a selected company role, each in the active version. Initial values:

| Terms (`agreementKey`) | Party | applies to |
|---|---|---|
| `osp-terms` | `OSP` | all applications |
| `terms-active-participant` | `DATASPACE` | `ACTIVE_PARTICIPANT` |
| `security-regulations` | `DATASPACE` | `ACTIVE_PARTICIPANT` |
| `terms-app-provider` | `DATASPACE` | `APP_PROVIDER` |
| `terms-service-provider` | `DATASPACE` | `SERVICE_PROVIDER` |

- The consent `psp-delegation` is dropped; there is no PSP any more that acts for the company.
- The UI shows the person the required terms (7.1, `GET /agreements`) in their language. The person consents personally at submission: per required terms document the key, the version and the language of the document read. There is no rejection; if a consent is missing, the submission fails.
- Stored are key, version, language, time, `sub` of the person and, in the verification path `EBW`, the current verification attestation.
- At submission, the module records the required terms in `required_agreements`. Later versions do not change them. If a new version becomes active before submission, the submission requires the new one (400 `consents-incomplete` with the expected version).
- **New version** (7.16): OSP staff create it; the previous active version of the same key becomes `INACTIVE`. With the version they define whether existing companies must consent again (`reconsentRequired`) and by when (`reconsentDueAt`). The re-consent of existing companies is not handled by this module (OP-11); it reports the new version with `AgreementVersionActivated`.

### 6.9 Verification path without Business Wallet (supporting documents)

- The verification path `DOCUMENTS` is allowed as long as `DOCUMENTS_ENABLED` is `true` (default). Otherwise creation answers with 400 `documents-disabled`. The setting takes effect only at creation; existing applications stay editable.
- **Data:** all company data comes from the person: `legalName`, `address` and at least one identifier (6.6).
- **Supporting documents:**

| `documentType` | Meaning | Required at submission |
|---|---|---|
| `COMMERCIAL_REGISTER_EXTRACT` | commercial register extract or comparable register extract | yes, at least one |
| `POWER_OF_ATTORNEY` | power of attorney or proof of signing authority; needed if the person is not listed as a representative in the register extract (OSP staff check this) | no |
| `OTHER` | further supporting document | no |

- Formats: PDF, PNG, JPEG. The module checks the first bytes of the content (`%PDF-`, PNG signature, JPEG signature `FF D8 FF`), not only the declared type. At most `DOCUMENT_MAX_BYTES` per supporting document and `DOCUMENT_MAX_COUNT` per application. Otherwise 400 `validation`.
- **Declaration of representation:** at submission the person declares that they may represent the company (`representationDeclared: true`).
- If the register extract or the declaration is missing at submission: 400 `documents-incomplete` with `missingDocumentTypes` or `representationDeclarationMissing`.
- OSP staff review the supporting documents manually in C1-M3; there is no automatic approval (OP-35). First and last name of the person from the token are in the application so that OSP staff can compare them with the supporting documents.
- Supporting documents are never changed. Deletion is possible only in status `CREATED`.

### 6.10 Withdraw

- The person withdraws their application in `CREATED` or `SUBMITTED` as long as OSP staff have not decided (OP-15).
- In `SUBMITTED` the module asks C1-M3 for the review case of the application (11.3). If it is `APPROVED` or `DECLINED`: 409 `already-decided` with `reviewStatus`. If there is no review case or it is `OPEN` or `AWAITING_SECOND_APPROVAL`, the withdrawal goes ahead. C1-M3 not reachable: 502 `upstream-error`, nothing changed.
- Status `WITHDRAWN`, `withdrawn_at`. Event `ApplicationWithdrawn` only when withdrawing from `SUBMITTED`; before submission there is no review case and no new company, so nothing for other modules to do.
- **Concurrent approval.** Between the query at C1-M3 and saving, OSP staff may approve. Then there are both `ApplicationWithdrawn` and `ApplicationApproved` for the same application. C1-M3 rejects every decision about a withdrawn application (C1-M3 6.5), and `close-review` reports an already approved review case with 409 (C1-M3 8.2). Process Orchestration then stops and alerts OSP staff (OP-18).

### 6.11 Idempotency and concurrency

- **Create:** if the person has an open application whose current verification attestation has the same `jti`, this is a repetition: response 200 with this application, no change, no event. Every other call while an open application exists results in 409 `open-application-exists` with `existingApplicationId`; the UI then continues with this application. This also applies in the verification path `DOCUMENTS`.
- **Submit:** an application in `SUBMITTED` answers a repeated submission with 200 and changes nothing.
- **Withdraw:** an application in `WITHDRAWN` answers a repeated withdrawal with 200 and changes nothing.
- **Steps:** find the application by `applicationId` and return the same result on repetition, without a second event.
- Every changing operation and every step locks the row of the application (`SELECT … FOR UPDATE`). So, for example, change, submit and withdraw exclude each other. The partial unique indexes (one open application per person and per company) prevent duplicates on concurrent creation; a violation results in the same 409 response as the check before.

### 6.12 Audit log

Every changing operation and every step writes an entry in `audit_log` (FRAMEWORK 4.3): time, `sub` of the acting person (empty for steps, instead `processId` and `stepRunId`), `company_id`, operation (`operationId` from [openapi.yaml](openapi.yaml)), `application_id`, object ID.

## 7. Operations

### 7.1 Reference data and terms

All with `lang` (`de` or `en`, default `en`) for names, titles and descriptions.

- `GET /countries`: all countries, sorted by code.
- `GET /countries/{countryCode}/identifier-types`: identifier types of the country with name and regular expression. Unknown country: 404.
- `GET /company-roles`: selectable company roles with name, description and the keys of their terms.
- `GET /agreements?companyRole=…`: active required terms for the given roles (the parameter may be repeated), including the terms with `scope` `ALL`; per terms document the key, version, party, title and the documents per language (`documentUrls`). Without `companyRole`: only the terms with `scope` `ALL`. Unknown or not selectable role: 400 `validation`.

### 7.2 `POST /applications`

Input `ApplicationCreate`: `kind`, `verification`, `attestation` (only `EBW`), reference `companyId` or `bpn` (6.2), `companyData`, `companyRoles`.

1. `company_id` in the token: 403 `account-bound`.
2. Check the input, otherwise 400 `validation` with `errors`: `attestation` required for `EBW` (at most 16 KB), forbidden for `DOCUMENTS`; reference according to 6.2; `companyRoles` without duplicates, required according to 6.2; `companyData` according to [openapi.yaml](openapi.yaml), for `DOCUMENTS` with `legalName`, `address` and at least one identifier.
3. `DOCUMENTS` with `DOCUMENTS_ENABLED=false`: 400 `documents-disabled`.
4. For `EBW`: check the verification attestation (6.4). Failure: 400 `attestation-invalid` with `reason`; JWKS cannot be fetched: 502 `upstream-error`.
5. Open application of this person: repetition → 200 with this application; otherwise 409 `open-application-exists` (6.11).
6. `jti` already used: 409 `attestation-used`.
7. With a reference (6.2): read the company at C3-M1; check the status (409 `company-not-eligible`); compare identifiers (400 `attestation-mismatch`); open application for the company (409 `company-has-open-application`); take over the roles if none are given. C3-M1 not reachable: 502 `upstream-error`.
8. Build the company data: for `EBW` from verification attestation and input, with comparison (6.5); fixed fields differ: 400 `attestation-mismatch` with `fields`. For `DOCUMENTS` from the input.
9. Check against the reference data (6.6, 6.7): country, identifier types, formats, selectable roles. Otherwise 400 `validation` with `errors`.
10. In one transaction: generate `companyId` or take it from the reference; rows in `application` (status `CREATED`, `applicant_sub` and names from the token), `application_identifier`, for `EBW` `attestation` (`is_current`); audit log entry; event `ApplicationCreated`.
11. Response 201 with the application (`Application`) and header `Location`.

### 7.3 `GET /applications`

Own applications of the person (`applicant_sub` equal to `sub`), sorted by `createdAt` descending, paged. Filter `status` (may be repeated). Entries as `ApplicationSummary`.

### 7.4 `GET /applications/{applicationId}`

Own application, otherwise 404. Response `ApplicationWithReview`: company data, roles, extract of the current verification attestation (without the JWT itself), comparison, supporting documents (metadata only), consents, reason of a decline and `reviewStatus`:

| `reviewStatus` | Meaning |
|---|---|
| `NONE` | no review case; in status `CREATED` without a request to C1-M3 |
| `OPEN`, `AWAITING_SECOND_APPROVAL`, `APPROVED`, `DECLINED`, `CLOSED` | status of the review case in C1-M3 (11.3) |
| `UNKNOWN` | C1-M3 not reachable; the application is returned anyway |

### 7.5 `PATCH /applications/{applicationId}`

1. Not visible: 404. Status not `CREATED`: 409 `invalid-state`.
2. At least one field of `legalName`, `shortName`, `address`, `identifiers`, `companyRoles`. Otherwise 400 `validation`.
3. Given fields replace, missing ones stay. `address` is replaced as a whole. `identifiers` is the complete list of identifiers with `source` `APPLICANT`; identifiers with `source` `ATTESTATION` stay. In the verification path `EBW`: an `EBWOID` or `EUID` that differs from the verification attestation results in 400 `attestation-mismatch`; a value equal to the verification attestation is ignored. In the verification path `DOCUMENTS`: `EBWOID` results in 400 `validation`.
4. With a reference and changed `identifiers`: read the company again at C3-M1 and compare the identifiers (6.2); different: 400 `attestation-mismatch`. C3-M1 not reachable: 502 `upstream-error`.
5. Compare again (6.5) and check against the reference data as in 7.2 step 9.
6. Save, audit log entry. No event.
7. Response 200 with the application.

### 7.6 `PUT /applications/{applicationId}/attestation`

Replaces the verification attestation, for example if it expired before submission.

1. Not visible: 404. Status not `CREATED` or verification path `DOCUMENTS`: 409 `invalid-state`.
2. Check as in 7.2 step 4, so including the person (6.4 step 10).
3. Same `jti` as the current one: 200, nothing changed. `jti` already used: 409 `attestation-used`.
4. Same company: `company.ebwoid` and `company.country` equal to the current verification attestation; `company.euid` if both contain it. If the application has an EUID with `source` `APPLICANT` and the new verification attestation has an EUID, both must be equal; the identifier then gets `source` `ATTESTATION`. Otherwise 400 `attestation-mismatch`.
5. Take fields with result `FROM_ATTESTATION` from the new verification attestation, compare the others again (6.5).
6. In one transaction: previous verification attestation `is_current = false`, store the new one; audit log entry. No event.
7. Response 200 with the application.

### 7.7 `POST /applications/{applicationId}/documents`

Uploads a supporting document (`multipart/form-data` with `file` and `documentType`).

1. Not visible: 404. Status not `CREATED` or verification path `EBW`: 409 `invalid-state`.
2. `documentType` valid, file name, format by the first bytes, size, count (6.9). Otherwise 400 `validation` with `errors`.
3. In one transaction: row in `document` with checksum; audit log entry. No event.
4. Response 201 with the metadata (`DocumentInfo`) and header `Location`.

### 7.8 `GET /applications/{applicationId}/documents/{documentId}`

Own application and supporting document of this application, otherwise 404. Response 200 with the content, `Content-Type` as stored, `Content-Disposition: attachment` with the file name, `X-Content-Type-Options: nosniff`.

### 7.9 `DELETE /applications/{applicationId}/documents/{documentId}`

1. Not visible: 404. Status not `CREATED`: 409 `invalid-state`.
2. Delete the row; audit log entry. Response 204.

### 7.10 `POST /applications/{applicationId}/submit`

1. Not visible: 404. Status `SUBMITTED`: 200 with the application, nothing changed. `WITHDRAWN`, `DECLINED`, `COMPLETED`: 409 `invalid-state`.
2. Format of the consents (key, version, language `de` or `en`): otherwise 400 `validation`.
3. Verification path `EBW`: current verification attestation expired or too old (6.4): 409 `attestation-expired`.
4. Verification path `DOCUMENTS`: register extract and `representationDeclared: true` present (6.9); otherwise 400 `documents-incomplete`.
5. Determine the required terms (6.8). If a consent is missing or names a different version: 400 `consents-incomplete` with the list `missing` (key and expected version). Consent to unknown or not required terms: 400 `validation`.
6. In one transaction: store the consents (6.8), record `required_agreements`, for `DOCUMENTS` `representation_declared`, status `SUBMITTED`, `submitted_at`; audit log entry; event `ApplicationSubmitted` with `applicantSub`.
7. Response 200 with the application.

### 7.11 `POST /applications/{applicationId}/withdraw`

1. Not visible: 404. Status `WITHDRAWN`: 200 with the application, nothing changed. `DECLINED`, `COMPLETED`: 409 `invalid-state`.
2. Status `SUBMITTED`: query the review case at C1-M3 (6.10). `APPROVED` or `DECLINED`: 409 `already-decided` with `reviewStatus`. C1-M3 not reachable: 502 `upstream-error`.
3. In one transaction: status `WITHDRAWN`, `withdrawn_at`; audit log entry; from `SUBMITTED` event `ApplicationWithdrawn`.
4. Response 200 with the application.

### 7.12 `GET /osp/applications`

All applications of the installation, for OSP staff. Filters `status`, `kind`, `verification` (each may be repeated) and `companyId`. Sorted by `createdAt` descending, paged. Entries as `ApplicationSummary`.

### 7.13 `GET /osp/applications/{applicationId}`

One application in any status, otherwise 404. Response as in 7.4 (`ApplicationWithReview`).

### 7.14 `GET /osp/applications/{applicationId}/documents/{documentId}`

One supporting document, otherwise 404. Response as in 7.8. Every retrieval writes an audit log entry (reading personal supporting documents).

### 7.15 `GET /osp/agreements`

All versions of all terms, including inactive ones, sorted by key and `createdAt` descending. Filters `agreementKey` and `status`. Entries as `AgreementVersion`.

### 7.16 `POST /osp/agreements`

Creates a new version. Input `AgreementVersionCreate`: `agreementKey` (existing or new, `^[a-z0-9-]{1,64}$`), `version` (1 to 20 characters), `party`, `scope`, `companyRoles` (required and only for `COMPANY_ROLE`, only selectable roles), `title` (`de`, `en`), `documentUrls` (`de`, `en`, at least one, HTTPS), `reconsentRequired`, `reconsentDueAt` (required with `reconsentRequired`, in the future).

1. Check the input, otherwise 400 `validation`.
2. Version for this key already exists: 409 `conflict`.
3. In one transaction: new version `ACTIVE`, previous active version of the key `INACTIVE`; replace the assignment to company roles for the key; audit log entry; event `AgreementVersionActivated`.
4. Response 201 with the version.

## 8. Steps for Process Orchestration

Common (FRAMEWORK 4.4): input with `processId` and `stepRunId` (audit log only). Result `{ "stepRunId", "state": "DONE", "result": { "applicationId", "companyId", "kind", "status" } }`. Both steps finish within their call. Errors as Problem Details with `retryable` (section 12).

### 8.1 `complete-application`

- **Flow:** onboarding and transfer after `activate-company`, reactivation after `reactivate-company` (FRAMEWORK 5.2 step 6, 5.5, 5.6).
- **Input:** `applicationId`.
- **Preconditions:** application in `SUBMITTED`; review case approved and company active. The flow ensures the last two; the module checks only its own status.
- **Behaviour:** `SUBMITTED` → `COMPLETED`, `completed_at`; audit log entry; event `ApplicationCompleted`. Already `COMPLETED`: do nothing. `CREATED`, `WITHDRAWN` or `DECLINED`: 409 `invalid-state`. Unknown: 404 `not-found`.
- **Event:** `ApplicationCompleted`, only on the transition.

### 8.2 `close-application`

- **Flow:** onboarding, reactivation, transfer after `ApplicationDeclined` (FRAMEWORK 5.2 step 5).
- **Input:** `applicationId`, `reason` (1 to 1000 characters; reason from `ApplicationDeclined`).
- **Preconditions:** application in `SUBMITTED`.
- **Behaviour:** `SUBMITTED` → `DECLINED`, `decline_reason`, `closed_at`; audit log entry. Already `DECLINED`: do nothing, the stored reason stays. `WITHDRAWN`: do nothing, `DONE` with status `WITHDRAWN` (the application has already ended, 6.10). `CREATED` or `COMPLETED`: 409 `invalid-state`. Unknown: 404 `not-found`.
- **Event:** none. The person learns of the decline through the notifications triggered by `ApplicationDeclined` and sees the reason in the application (`declineReason`).

## 9. Queries for other modules

In the same process without a token; over HTTP with the permission `query_c1_m2`. The queries change nothing.

### 9.1 `GET /internal/applications/{applicationId}`

For C1-M3 (automatic pre-check, display in the review case) and C3-M1 (data to create the company; `verification` as the origin of the core data, OP-09). Response `ApplicationInternal`, in any status:

- all fields of `Application`, among them `kind`, `existingCompany`, `verification`, `applicant` (`sub`, first and last name), supporting documents (metadata only), `representationDeclared`, plus `requiredAgreements` (from `SUBMITTED` on),
- the current verification attestation with all evaluated claims (`company` with address, `issuer`, `keyId`, `clientId`, `representation`, `evidence`),
- `signatureCheck`: on every query the module checks the stored verification attestation again according to 6.4 steps 1 to 4, with the currently recognised issuer and its JWKS, without time check. `result` is `VALID`, `INVALID` (issuer no longer recognised or key no longer in the JWKS) or `UNAVAILABLE` (JWKS cannot be fetched), plus `checkedAt`.

The query does not return the JWT itself or the content of supporting documents. Unknown: 404.

### 9.2 `GET /internal/applications?applicationId=…`

Short form of several applications (`ApplicationSummary`), e.g. for the list of review cases in C1-M3. `applicationId` 1 to 100 times. Order as in the request; unknown IDs are missing from the response.

### 9.3 `GET /internal/agreements`

Terms with their versions (`AgreementVersion`), filter `status` (default `ACTIVE`). For C3-M1: the terms of a company role (C3-M1 6.5, 7.8, 8.3) and the re-consent of existing companies (6.8, OP-11).

## 10. Events

Through the outbox, in the same transaction as the change. Every event also carries `eventId`, `eventType`, `occurredAt`. No names, no secrets; as identifier of a person only the pseudonymous `applicantSub`. Schemas in [openapi.yaml](openapi.yaml) under `components/schemas` (names start with `Event`).

| Event | when | Content |
|---|---|---|
| `ApplicationCreated` | application created (7.2) | `applicationId`, `companyId`, `kind`, `existingCompany` |
| `ApplicationSubmitted` | application submitted (7.10) | `applicationId`, `companyId`, `kind`, `existingCompany`, `applicantSub` |
| `ApplicationWithdrawn` | submitted application withdrawn (7.11) | `applicationId`, `companyId`, `kind`, `existingCompany` |
| `ApplicationCompleted` | step `complete-application` (8.1) | `applicationId`, `companyId`, `kind` |
| `AgreementVersionActivated` | new version of terms active (7.16 or start with a new version in `CATALOG_FILE`) | `agreementKey`, `version`, `party`, `reconsentRequired`, `reconsentDueAt` |

- The flow needs `applicantSub` after the approval for C2-M2 `assign-company-admin`.
- `existingCompany` tells the flow whether a company must be created (`create-company`) or, after decline or withdrawal, declined (`reject-company`): only for `false`.

## 11. Calls to other modules and external systems

### 11.1 C5-M1 EBW-Connector of the installation

| Call | Response | Built-in stand-in |
|---|---|---|
| `GET {ATTESTATION_JWKS_URL}` (default `{ATTESTATION_ISSUER}/jwks`), timeout 5 seconds | JWKS (RFC 7517) | `ATTESTATION_JWKS_FILE`: JWKS from the file, no fetch |

### 11.2 C3-M1 Company Profile (`COMPANY_URL`)

Only for applications with a reference (6.2).

| Call | Response | Built-in stand-in |
|---|---|---|
| in the same process `get_company(company_id)`; over HTTP `GET /api/c3-m1/internal/companies/{companyId}` | `companyId`, `name`, `countryCode`, `status`, `deactivationReason`, `bpn`, `identifiers` (each `type`, `value`), `companyRoles`; 404 if unknown | reads `STUB_COMPANIES_FILE`: list of such entries |
| in the same process `find_companies_by_bpn([bpn])`; over HTTP `GET /api/c3-m1/internal/companies?bpn={bpn}` | `{ "items": [ … ], "notFound": [ … ] }` with the same fields | from the same file |

### 11.3 C1-M3 Application Review (`REVIEW_URL`)

When withdrawing a submitted application (7.11) and for `reviewStatus` (7.4, 7.13).

| Call | Response | Built-in stand-in |
|---|---|---|
| in the same process `get_review_status(application_id)`; over HTTP `GET /api/c1-m3/internal/review-cases?applicationId={id}` | `{ "reviewCaseId", "applicationId", "status", "decidedAt" }`; 404 if there is no review case | reads `STUB_REVIEWS_FILE`: list of `{ "applicationId", "status" }` |

The stand-in files are read on every call so that tests can change them. If a stand-in file contains the JSON string `"unavailable"` instead of a list, the stand-in reports "not reachable".

### 11.4 Process Orchestration

No calls. It reads the events from the outbox and calls the steps.

## 12. Errors

Problem Details (RFC 9457). Additional fields per error are in [openapi.yaml](openapi.yaml) (`Problem`). `retryable` applies only to steps.

| `type` | HTTP | when | `retryable` |
|---|---|---|---|
| `urn:c1-m2:problem:validation` | 400 | input invalid, identifier, role or supporting document not allowed; details in `errors` | `false` |
| `urn:c1-m2:problem:attestation-invalid` | 400 | verification attestation not acceptable; reason in `reason` (6.4) | – |
| `urn:c1-m2:problem:attestation-mismatch` | 400 | fixed field differs from the verification attestation, new verification attestation concerns another company, or identifier does not fit the existing company; fields in `fields` | – |
| `urn:c1-m2:problem:consents-incomplete` | 400 | consent to required terms missing or with a different version; list in `missing` | – |
| `urn:c1-m2:problem:documents-incomplete` | 400 | verification path `DOCUMENTS`: register extract or declaration of representation missing | – |
| `urn:c1-m2:problem:documents-disabled` | 400 | verification path `DOCUMENTS` is switched off | – |
| `urn:c1-m2:problem:unauthorized` | 401 | token missing or invalid | – |
| `urn:c1-m2:problem:forbidden` | 403 | permission missing | – |
| `urn:c1-m2:problem:account-bound` | 403 | creation with an account that is already bound to a company | – |
| `urn:c1-m2:problem:not-found` | 404 | application, supporting document or country does not exist or is not visible | `false` |
| `urn:c1-m2:problem:invalid-state` | 409 | status or verification path does not allow the operation or the step | `false` |
| `urn:c1-m2:problem:conflict` | 409 | version of the terms already exists | – |
| `urn:c1-m2:problem:open-application-exists` | 409 | the person already has an open application; ID in `existingApplicationId` | – |
| `urn:c1-m2:problem:company-has-open-application` | 409 | there already is an open application for the existing company | – |
| `urn:c1-m2:problem:company-not-eligible` | 409 | reference unknown, or status or reason of the deactivation does not fit the application kind; `companyStatus` | – |
| `urn:c1-m2:problem:attestation-used` | 409 | verification attestation (`jti`) already used | – |
| `urn:c1-m2:problem:attestation-expired` | 409 | at submission: verification attestation expired or older than `ATTESTATION_MAX_AGE` | – |
| `urn:c1-m2:problem:already-decided` | 409 | withdrawal after the decision; `reviewStatus` | – |
| `urn:c1-m2:problem:upstream-error` | 502 | JWKS of the issuer, C3-M1 or C1-M3 not reachable | `true` |

## 13. Acceptance

The module is done when these cases work with built-in stand-ins for the neighbours and with test tokens of the Portal IAM. The tests sign verification attestations themselves (14.4).

1. Creation by Erika Mustermann with `kind` `ONBOARDING`, `verification` `EBW`, verification attestation and `companyRoles`: 201, status `CREATED`, new `companyId`; name, address, `EBWOID` and `EUID` from the verification attestation, comparison `FROM_ATTESTATION` everywhere; `applicant` with her `sub` and names. The outbox holds `ApplicationCreated` with `applicationId`, `companyId`, `kind` `ONBOARDING`, `existingCompany` `false`.
2. Verification attestation with another issuer, wrong key, `alg` `none`, wrong `aud`, expired `exp`, missing `company.ebwoid`, `attestation_version` `"0.1"`, `purpose` `"delegation"`, and with `ATTESTATION_CLIENT_IDS=demo-osp` one with `client_id` `other-client`: each 400 `attestation-invalid` with `reason` `issuer`, `signature`, `algorithm`, `audience`, `expired`, `claims`, `version`, `purpose`, `client`. No application is created.
3. Verification attestation with `person` Max Mustermann, created by Erika Mustermann: 400 `attestation-invalid` with `reason` `person`. With `person` " ERIKA " and "mustermann": 201.
4. Token with `company_id`: 403 `account-bound`, no application. The same account reads `GET /applications`: 200.
5. The same verification attestation again from Erika Mustermann: 200 with the same application, no second event. Another verification attestation while the application is open: 409 `open-application-exists` with `existingApplicationId`. After withdrawal, the same verification attestation in a new application: 409 `attestation-used`.
6. Different EUID from the person: 400 `attestation-mismatch` with `EUID`. Different name: 201 with `LEGAL_NAME` `MISMATCH`.
7. `LEI_CODE` for a country without this identifier type, `VAT_ID` `DE12345` for Germany, role `ONBOARDING_SERVICE_PROVIDER`: each 400 `validation`.
8. Max Mustermann reads, changes, submits or withdraws the application of Erika Mustermann: each 404.
9. Changing the roles in `CREATED`: 200. After submission: 409 `invalid-state`.
10. Submission without consent to `osp-terms`: 400 `consents-incomplete`, `missing` contains `osp-terms`. With all required terms and language: 200, status `SUBMITTED`, `ApplicationSubmitted` with `applicantSub` equal to her `sub`; the consents carry language, `sub` and verification attestation. Repeated submission: 200 without a second event.
11. Submission with a verification attestation older than `ATTESTATION_MAX_AGE` (clock shifted in the test): 409 `attestation-expired`. After `PUT …/attestation` with a new verification attestation of the same company it succeeds. The old `jti` stays used.
12. `PUT …/attestation` with a verification attestation of another company (different `EBWOID`): 400 `attestation-mismatch`. `PUT …/attestation` on an application in the verification path `DOCUMENTS`: 409 `invalid-state`.
13. `REACTIVATION` with the `companyId` of Beispiel Werkzeugbau GmbH (14.5) and a matching verification attestation, without `companyRoles`: 201, `existingCompany` `true`, `companyId` equal to the existing one, roles from C3-M1. The same with `bpn` instead of `companyId`: the same. For an active company: 409 `company-not-eligible` with `companyStatus` `ACTIVE`. With a verification attestation of Beispiel Maschinenbau GmbH: 400 `attestation-mismatch` with `EBWOID`. `ONBOARDING` with `companyId`: 400 `validation`. C3-M1 not reachable: 502.
14. `TRANSFER` without reference: 201, new `companyId`, `existingCompany` `false`. `TRANSFER` with the `companyId` of Beispiel Gießerei GmbH (transferred away): 201, `existingCompany` `true`. `REACTIVATION` for Beispiel Gießerei GmbH: 409 `company-not-eligible`.
15. Max Mustermann creates a second open application for Beispiel Werkzeugbau GmbH: 409 `company-has-open-application`.
16. Verification path `DOCUMENTS` with data and VAT ID, without verification attestation: 201, `comparison` empty, identifiers with `source` `APPLICANT`. With `EBWOID` or with verification attestation: 400 `validation`. Upload a supporting document as PDF: 201. A file with extension `.pdf` whose first bytes are not a PDF, a supporting document over `DOCUMENT_MAX_BYTES` and the eleventh supporting document: each 400 `validation`. Submission without register extract or without `representationDeclared`: 400 `documents-incomplete`. With both: 200.
17. `DOCUMENTS_ENABLED=false`: creation in the verification path `DOCUMENTS` results in 400 `documents-disabled`.
18. Withdrawal in `CREATED`: `WITHDRAWN`, no event. In `SUBMITTED` with review case `OPEN` in the stand-in: `WITHDRAWN`, `ApplicationWithdrawn` with `kind` and `existingCompany`. With review case `APPROVED`: 409 `already-decided`. Stand-in not reachable: 502, nothing changed. Repeated withdrawal: 200 without a second event.
19. `GET /applications/{id}` in `CREATED`: `reviewStatus` `NONE`. In `SUBMITTED` with review case `OPEN` in the stand-in: `OPEN`. Stand-in not reachable: 200 with `UNKNOWN`.
20. OSP staff (`view_applications`) list all applications, including those of other people, read one and download a supporting document: 200; the retrieval of the supporting document is in the audit log. Erika Mustermann on `/osp/applications`: 403. OSP staff without `register_company` on `GET /applications/{id}`: 403.
21. The OSP Admin creates `osp-terms` in version `1.1` with `reconsentRequired`: 201, version `1.0` `INACTIVE`, `AgreementVersionActivated` in the outbox. Submission of an application in `CREATED` with consent to `1.0`: 400 `consents-incomplete` with expected version `1.1`. Version `1.1` again: 409 `conflict`. OSP Reviewer without `manage_agreements`: 403.
22. `complete-application` on `SUBMITTED`: `COMPLETED` and `ApplicationCompleted` with `kind`. Repetition: `DONE`, no second event. On `CREATED`: 409 `invalid-state` with `retryable` `false`.
23. `close-application` with reason: `DECLINED`, the person sees the reason in `declineReason`. Repetition: unchanged. On `WITHDRAWN`: `DONE` with status `WITHDRAWN`.
24. `/internal/applications/{id}` returns `signatureCheck` `VALID` and `applicant.sub`. After removing the key from `ATTESTATION_JWKS_FILE` and a restart: `INVALID`.
25. `GET /agreements?companyRole=ACTIVE_PARTICIPANT` returns `osp-terms`, `terms-active-participant`, `security-regulations` with party and `documentUrls`.
26. Every changing operation is in `audit_log` with `sub`, `company_id` and operation. Neither log nor events nor `audit_log` contain the JWT, content of supporting documents or names of people.

## 14. Test data

### 14.1 Test Portal IAM

Shared test IDs according to FRAMEWORK 7a; the installation is that of OSP A.

| Account | `sub` | Claims | Permissions |
|---|---|---|---|
| Erika Mustermann (applicant) | `5a1c0001-0000-4000-8000-000000000001` | `given_name` Erika, `family_name` Mustermann, without `company_id` | `register_company` |
| Max Mustermann (second applicant) | `5a1c0002-0000-4000-8000-000000000002` | `given_name` Max, `family_name` Mustermann, without `company_id` | `register_company` |
| bound account | `d4fef928-56fe-4c94-a6c1-ffc6ea43704c` | `company_id` `11111111-1111-4111-8111-111111111111` | `register_company` |
| OSP Reviewer | `bbede0f3-fbc4-433b-b7ca-1982e8724912` | | `view_applications` |
| OSP Admin | `5a1c0012-0000-4000-8000-000000000012` | | `view_applications`, `manage_agreements` |
| service account | – | | `execute_steps_c1_m2`, `query_c1_m2` |
| account without permissions | `08f42539-b270-4ca0-805d-c2c8500f23cd` | | – (cases with 403) |

### 14.2 Shared reference data (`masterDataFile`, extract)

Countries and the assignment of identifier types are generated from the seed files of portal 2.6.0 (`countries.json`, `country_long_names.json`, `country_assigned_identifiers.json` without `VIES`), extended by `EUID` and `EBWOID` for EU and EEA (AT, BE, BG, CY, CZ, DE, DK, EE, ES, FI, FR, GR, HR, HU, IE, IT, LT, LU, LV, MT, NL, PL, PT, RO, SE, SI, SK, IS, LI, NO).

```json
{
  "countries": [
    { "code": "DE", "name": { "de": "Deutschland", "en": "Germany" },
      "identifierTypes": ["EUID", "EBWOID", "COMMERCIAL_REG_NUMBER", "VAT_ID", "LEI_CODE", "EORI"] },
    { "code": "US", "name": { "de": "Vereinigte Staaten", "en": "United States" },
      "identifierTypes": ["EORI"] }
  ],
  "identifierTypes": [
    { "type": "EUID", "name": { "de": "EUID", "en": "EUID" },
      "patterns": { "Worldwide": "^[A-Z]{2}[A-Za-z0-9.\\-]{2,60}$" } },
    { "type": "EBWOID", "name": { "de": "EBWOID", "en": "EBWOID" },
      "patterns": { "Worldwide": "^\\S{1,200}$" } },
    { "type": "COMMERCIAL_REG_NUMBER", "name": { "de": "Handelsregisternummer", "en": "Commercial register number" },
      "patterns": {
        "Worldwide": "^(?!.*\\s$)([A-Za-zÀ-ÿ0-9.()](\\.|\\s|-|_)?){4,50}$",
        "DE": "^(?!.*\\s$)([A-Za-zÀ-ÿ])([A-Za-zÀ-ÿ0-9.()](\\s|-|_)?){4,50}$",
        "FR": "^\\d{9}$" } },
    { "type": "VAT_ID", "name": { "de": "USt-ID", "en": "VAT ID" },
      "patterns": {
        "Worldwide": "^(?!.*\\s$)([A-Za-z0-9](\\.|\\s|-|\\/)?){5,18}$",
        "DE": "^DE\\d{9}$", "IN": "^[a-zA-Z\\d-]{5,15}$", "MX": "^[a-zA-Z\\d-&]{12,13}$" } },
    { "type": "LEI_CODE", "name": { "de": "LEI", "en": "LEI" },
      "patterns": { "Worldwide": "^[A-Za-z0-9]{20}$" } },
    { "type": "EORI", "name": { "de": "EORI-Nummer", "en": "EORI number" },
      "patterns": { "Worldwide": "^[A-Z]{2}[A-Za-z0-9]{1,15}$" } }
  ]
}
```

### 14.3 Company roles and terms (`CATALOG_FILE`)

```json
{
  "companyRoles": [
    { "role": "ACTIVE_PARTICIPANT", "selectable": true,
      "name": { "de": "Teilnehmer", "en": "Participant" },
      "description": { "de": "Datenanbieter, Datenkonsument, App-Nutzer", "en": "Data provider, data consumer, app user" },
      "agreements": ["terms-active-participant", "security-regulations"] },
    { "role": "APP_PROVIDER", "selectable": true,
      "name": { "de": "App-Anbieter", "en": "App provider" },
      "description": { "de": "bietet Apps im Marktplatz an", "en": "offers apps in the marketplace" },
      "agreements": ["terms-app-provider"] },
    { "role": "SERVICE_PROVIDER", "selectable": true,
      "name": { "de": "Service-Anbieter", "en": "Service provider" },
      "description": { "de": "bietet Services im Marktplatz an", "en": "offers services in the marketplace" },
      "agreements": ["terms-service-provider"] },
    { "role": "ONBOARDING_SERVICE_PROVIDER", "selectable": false,
      "name": { "de": "Onboarding-Dienstleister", "en": "Onboarding service provider" },
      "description": { "de": "Unternehmen des OSP", "en": "Company of the OSP" }, "agreements": [] }
  ],
  "agreements": [
    { "key": "osp-terms", "version": "1.0", "party": "OSP", "scope": "ALL",
      "title": { "de": "Nutzungsbedingungen des OSP", "en": "Terms of use of the OSP" },
      "documentUrls": { "de": "https://osp-a.example.org/terms/osp-terms-1.0-de.pdf",
                        "en": "https://osp-a.example.org/terms/osp-terms-1.0-en.pdf" } },
    { "key": "terms-active-participant", "version": "1.0", "party": "DATASPACE", "scope": "COMPANY_ROLE",
      "title": { "de": "Nutzungsbedingungen Teilnehmer", "en": "Terms and conditions participant" },
      "documentUrls": { "de": "https://dataspace.example.org/terms/active-participant-1.0-de.pdf",
                        "en": "https://dataspace.example.org/terms/active-participant-1.0-en.pdf" } },
    { "key": "security-regulations", "version": "1.0", "party": "DATASPACE", "scope": "COMPANY_ROLE",
      "title": { "de": "Sicherheitsregeln", "en": "Security regulations" },
      "documentUrls": { "en": "https://dataspace.example.org/terms/security-1.0-en.pdf" } },
    { "key": "terms-app-provider", "version": "1.0", "party": "DATASPACE", "scope": "COMPANY_ROLE",
      "title": { "de": "Nutzungsbedingungen App-Anbieter", "en": "Terms and conditions app provider" },
      "documentUrls": { "en": "https://dataspace.example.org/terms/app-provider-1.0-en.pdf" } },
    { "key": "terms-service-provider", "version": "1.0", "party": "DATASPACE", "scope": "COMPANY_ROLE",
      "title": { "de": "Nutzungsbedingungen Service-Anbieter", "en": "Terms and conditions service provider" },
      "documentUrls": { "en": "https://dataspace.example.org/terms/service-provider-1.0-en.pdf" } }
  ]
}
```

### 14.4 Verification attestation

At start, the tests generate their own key pair (EC P-256), write the public key to `ATTESTATION_JWKS_FILE` and sign the example from C5-M1 14.5 and its variants (expired, other issuer, other company, other person, `representation.confirmed: false`). Settings in the test: `ATTESTATION_ISSUER=https://ebw-connector.example.org`, `ATTESTATION_AUDIENCE=portal-registration`.

Header:

```json
{ "alg": "ES256", "typ": "ebw-attestation+jwt", "kid": "ebw-test-2026-1" }
```

Payload (C5-M1 14.5):

```json
{
  "iss": "https://ebw-connector.example.org",
  "sub": "EBWOID-DE-0000000001",
  "aud": "portal-registration",
  "iat": 1793174400,
  "exp": 1795766400,
  "jti": "3f6c1e2a-7b4d-4e8f-9a51-2c0d6b8e4f17",
  "attestation_version": "0.2",
  "purpose": "onboarding",
  "verified_at": 1793174400,
  "client_id": "demo-osp",
  "person": { "given_name": "Erika", "family_name": "Mustermann" },
  "company": {
    "legal_name": "Beispiel Maschinenbau GmbH",
    "ebwoid": "EBWOID-DE-0000000001",
    "euid": "DEXX001T.HRB100001",
    "address": { "street_address": "Musterstraße 1", "postal_code": "70569",
                 "locality": "Stuttgart", "country": "DE" },
    "country": "DE"
  },
  "representation": {
    "confirmed": true, "result": "CONFIRMED",
    "type": "legal_representative", "role": "Geschäftsführerin", "valid_until": "2027-12-31",
    "issuer": "https://register.test.example.org", "trust": "trust_list"
  },
  "evidence": [
    { "type": "PID", "vct": "urn:eudi:pid:1", "issuer": "https://pid.test.example.org",
      "wallet": "EUDI_WALLET", "status_checked": false },
    { "type": "MANDATE", "vct": "urn:tx:test:mandate:1", "issuer": "https://register.test.example.org",
      "wallet": "EUDI_WALLET", "status_checked": true },
    { "type": "LPID", "vct": "urn:tx:test:lpid:1", "issuer": "https://register.test.example.org",
      "wallet": "BUSINESS_WALLET", "status_checked": false }
  ]
}
```

`iat` is 28 Oct 2026 08:00 UTC, `exp` 27 Nov 2026 08:00 UTC. Variants for 13: Beispiel Werkzeugbau GmbH, `EBWOID-DE-0000000003`, EUID `DEXX001T.HRB100003`; Beispiel Gießerei GmbH, `EBWOID-DE-0000000004`, EUID `DEXX001T.HRB100004`; each with its own `jti`. Companies and identifiers are fictitious.

Create (`ONBOARDING`, `EBW`):

```json
{
  "kind": "ONBOARDING",
  "verification": "EBW",
  "attestation": "<compact JWS serialisation of the verification attestation>",
  "companyData": {
    "shortName": "Beispiel Maschinenbau",
    "identifiers": [ { "type": "VAT_ID", "value": "DE123456789" } ]
  },
  "companyRoles": ["ACTIVE_PARTICIPANT"]
}
```

Create (`REACTIVATION`):

```json
{
  "kind": "REACTIVATION",
  "verification": "EBW",
  "companyId": "cc080671-720e-42b6-95b2-0784c8314242",
  "attestation": "<verification attestation of Beispiel Werkzeugbau GmbH>"
}
```

Create (`ONBOARDING`, `DOCUMENTS`):

```json
{
  "kind": "ONBOARDING",
  "verification": "DOCUMENTS",
  "companyData": {
    "legalName": "Beispiel Feinmechanik GmbH",
    "address": { "streetName": "Musterweg", "streetNumber": "5", "postalCode": "70569",
                 "city": "Stuttgart", "countryCode": "DE" },
    "identifiers": [ { "type": "COMMERCIAL_REG_NUMBER", "value": "HRB 100005" } ]
  },
  "companyRoles": ["ACTIVE_PARTICIPANT"]
}
```

Submit (`EBW`):

```json
{
  "consents": [
    { "agreementKey": "osp-terms", "version": "1.0", "language": "de" },
    { "agreementKey": "terms-active-participant", "version": "1.0", "language": "de" },
    { "agreementKey": "security-regulations", "version": "1.0", "language": "en" }
  ]
}
```

In the verification path `DOCUMENTS` additionally `"representationDeclared": true`.

### 14.5 Stand-in files

`ATTESTATION_JWKS_FILE`:

```json
{ "keys": [ { "kty": "EC", "crv": "P-256", "kid": "ebw-test-2026-1", "x": "<Base64url>", "y": "<Base64url>" } ] }
```

`stub-companies.json` (stand-in for C3-M1):

```json
[
  { "companyId": "cc080671-720e-42b6-95b2-0784c8314242", "name": "Beispiel Werkzeugbau GmbH",
    "countryCode": "DE", "status": "INACTIVE", "deactivationReason": "OTHER", "bpn": "BPNL00000000003C",
    "identifiers": [ { "type": "EBWOID", "value": "EBWOID-DE-0000000003" },
                     { "type": "EUID", "value": "DEXX001T.HRB100003" } ],
    "companyRoles": ["ACTIVE_PARTICIPANT"] },
  { "companyId": "6c71d54b-e689-4eb6-a1fa-ae196d8a5209", "name": "Beispiel Gießerei GmbH",
    "countryCode": "DE", "status": "INACTIVE", "deactivationReason": "TRANSFERRED", "bpn": "BPNL00000000004D",
    "identifiers": [ { "type": "EBWOID", "value": "EBWOID-DE-0000000004" } ],
    "companyRoles": ["ACTIVE_PARTICIPANT", "APP_PROVIDER"] },
  { "companyId": "11111111-1111-4111-8111-111111111111", "name": "OSP A Beispiel GmbH",
    "countryCode": "DE", "status": "ACTIVE", "bpn": "BPNL00000000OSPA",
    "identifiers": [ { "type": "EUID", "value": "DEXX001T.HRB100000" } ],
    "companyRoles": ["ONBOARDING_SERVICE_PROVIDER"] }
]
```

`deactivationReason` `OTHER` stands for any reason other than `TRANSFERRED`.

`stub-reviews.json` (stand-in for C1-M3): the tests write the file after they have submitted the application, e.g.

```json
[ { "applicationId": "<applicationId from case 18>", "status": "OPEN" } ]
```

## 15. Changes from version 0.1

- **OSP 2.0 instead of the PSP architecture.** The applicant calls with their own token (portal role `Applicant`, permission `register_company`, scope `own`). The application belongs to them (`applicant_sub`) instead of a PSP. Dropped: `psp_id`, `externalId`, the headers `X-Acting-Person` and `X-On-Behalf-Of`, the delegation scope `onboarding`, the permission `view_application` with `GET /companies/{companyId}/application`, the check at C2-M4 and the status report through C1-M1 `notify-psp`.
- New permissions for OSP staff (scope `osp`): `view_applications` (read all applications and supporting documents, `/osp/applications`) and `manage_agreements` (versions of the terms, `/osp/agreements`).
- Creation only with an account without a company (403 `account-bound`); at most one open application per person and per existing company. Repetition at creation via the `jti` instead of `externalId`.
- **Verification attestation version 0.2** (C5-M1 section 9): the only recognised issuer is the EBW-Connector of its own installation (`ATTESTATION_ISSUER` instead of `TRUSTED_ATTESTATION_ISSUERS_FILE`); new checks `attestation_version`, `purpose`, `sub`, optionally `client_id`; new binding to the signed-in person through first and last name (reason `person`). `representation` and `evidence` are stored in full. The binding to a PSP (`psp_id`, OP-17) is dropped.
- **Application kinds** `ONBOARDING`, `REACTIVATION` (existing `companyId` or `bpn`) and `TRANSFER` (transfer, with reference as return); query at C3-M1 for existing companies.
- **Verification path without Business Wallet** (OP-35): `verification` `DOCUMENTS` with supporting documents, declaration of representation, can be switched off (`DOCUMENTS_ENABLED`); new table `document` and operations for supporting documents.
- **Consents** are given by the person personally, with the language of the document read (OP-11). The terms `psp-delegation` are dropped, `osp-terms` are new. Terms have a party (`OSP`, `DATASPACE`), one document per language and details on re-consent; OSP staff create new versions (`AgreementVersionActivated`).
- **Withdrawal** by the person until the decision (OP-15): status `WITHDRAWN`, event `ApplicationWithdrawn`, query of the review case at C1-M3; `reviewStatus` in the application.
- **Events** with `kind` and `existingCompany` instead of `pspId`; `ApplicationSubmitted` carries `applicantSub` for C2-M2 `assign-company-admin`.
- Countries and identifier types from the shared file of the installation (`masterDataFile`, OP-13); company roles and terms from `CATALOG_FILE`. The role `PORTAL_SERVICE_PROVIDER` is dropped.
- Identifier source `PSP` is now called `APPLICANT`. Errors of the steps with `retryable`.
- Test IDs according to FRAMEWORK 7a; test companies and verification attestation as in C5-M1.
