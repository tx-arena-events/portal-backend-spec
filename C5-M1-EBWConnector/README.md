# C5-M1 EBW-Connector

**Version:** 0.2, as of 2026-10-28
**Capability:** C5 Business Wallet
**Kind:** Provided (built before the hackathon; a reference implementation is provided for the hackathon)
**Files:** this README (behaviour), [openapi.yaml](openapi.yaml) (interface)

This README and the OpenAPI description are sufficient to build the module. Modules from other parts of the spec are referred to by name.

## 1. Purpose

The EBW-Connector identifies a person and, where needed, the company the person represents. It uses two wallets:

| Wallet | Belongs to | Runs | Holds |
|---|---|---|---|
| EUDI Wallet | the person | on the phone | PID, mandate |
| Business Wallet | the company | in the browser | LPID; issues mandates to employees |

Clients use the EBW-Connector through OpenID Connect. The scopes of the request decide which credentials are requested from which wallet. For onboarding, the EBW-Connector also issues a verification attestation that the registration (C1-M2) accepts.

**Operation.** Every operator that needs wallet-based identification runs its own instance: an onboarding service provider (OSP) for its portal, a Business Wallet for the sign-in of its users. One instance may also serve several clients. Instances do not talk to each other. They trust the same issuers through the trust list of the core services (11.3). The module calls no other portal module.

Callers:

| Who | What for |
|---|---|
| OSP portal (OIDC client) | sign-in of people; onboarding of a company with verification attestation |
| Business Wallet (OIDC client) | sign-in of the company's employees |
| EUDI Wallet of the person | fetch the request, send the response (OpenID4VP) |
| Business Wallet of the company (as wallet) | fetch the request, send the response (OpenID4VP) |
| Browser of the person | pages of a transaction |
| Registration (C1-M2) | read the JWKS to check verification attestations |

## 2. Terms

| Term | Meaning |
|---|---|
| EUDI Wallet | the person's wallet for identification (European Digital Identity Wallet), usually on the phone |
| Business Wallet | the company's wallet (European Business Wallet), used in the browser; holds the LPID and issues mandates |
| Dataspace wallet | the company's wallet for data exchange in the dataspace (C4-M2); not used by this module |
| PID | person identification data |
| LPID | legal person identification data of the company |
| EBWOID | identifier of the owner of a European Business Wallet; format not fixed yet |
| EUID | European unique identifier of a company from the business registers |
| Mandate | credential that a person may act for a company: legal representation (issued by a register) or power of attorney (issued by the company's Business Wallet) |
| Credential | SD-JWT VC (`dc+sd-jwt`) |
| Issuer | who signs a credential: PID provider, register, Business Wallet |
| Trust list | list of issuers whose credentials are accepted, per credential type; provided by the core services (11.3) |
| Transaction | one sign-in or onboarding, from the authorization request of the client to the redeemed code |
| Step | one presentation from one wallet within a transaction: `person` (EUDI Wallet) or `company` (Business Wallet) |
| Verification attestation | JWT signed by the EBW-Connector about a successful onboarding transaction (section 9) |
| Client | registered OIDC client: OSP portal or Business Wallet |
| Same-device, cross-device | wallet on the same device as the browser (link) or on another device (QR code) |
| DCQL | Digital Credentials Query Language of OpenID4VP |
| KB-JWT | key binding JWT: proof of the wallet that it holds the key the credential is bound to |

## 3. Not part of this module

| What | Where |
|---|---|
| Dataspace wallet, DID, BDRS | C4-M2 Dataspace Wallet Interface |
| Create an application, check and accept the attestation | C1-M2 Registration |
| Review of applications | C1-M3 Application Review |
| User accounts of people at the OSP | OSP |
| Issue PID, LPID and mandates for legal representation | member states, registers; for tests the test issuer (14.1) |
| Issue powers of attorney to employees | Business Wallet |
| Keycloak extensions | none |
| Qualified signatures, seals, registered delivery | not in scope |
| Single sign-on session | none; every transaction needs the wallet |

## 4. Technical frame

- Standalone service in Python with FastAPI.
- State: transactions are short-lived (minutes) and kept in the memory of one instance. An audit log without personal data is written as JSON lines. A database for several instances is a later step.
- Standards:
  - OpenID for Verifiable Presentations (OpenID4VP) 1.0 with DCQL; request by reference (`request_uri`, signed request object); client identifier prefix `x509_san_dns`; response mode `direct_post.jwt` (encrypted response), `direct_post` configurable for wallets without encryption;
  - SD-JWT VC; IETF Token Status List;
  - OpenID Connect Core 1.0, authorization code flow, PKCE with `S256` required, RFC 7523 `private_key_jwt`, RFC 9207 (`iss` in the authorization response);
  - guideline: OpenID4VC High Assurance Interoperability Profile (HAIP).
- Signing key ES256 (P-256) for ID tokens, attestations and request objects. Request objects carry a certificate chain (`x5c`) whose leaf names the host of `PUBLIC_URL` as DNS name. A self-signed certificate is accepted for tests only.
- Logs without personal data: no names, birth dates or identifiers from PID, LPID or mandate, no `vp_token`, no attestations. Allowed: transaction ID, client, status, failure reason, issuer, `vct`.
- Pages: plain HTML in German and English (`ui_locales`).

### Configuration

| Variable | Meaning | Default |
|---|---|---|
| `PUBLIC_URL` | address of the instance; `iss` of ID tokens and attestations; basis of all endpoints | `http://localhost:8077` |
| `KEY_DIR` | directory for the signing key, the subject key and the self-signed certificate; created on first start | `.keys` |
| `REQUEST_CERT_FILE` | certificate chain (PEM) for `x5c`; empty: self-signed for the host of `PUBLIC_URL` (tests only) | |
| `CLIENTS_FILE` | registered clients (6.8) | `clients.json` |
| `TRUST_LIST_URL` | trust list of the core services, signed (11.3) | |
| `TRUST_LIST_SIGNER_FILE` | JWKS with the keys that may sign the trust list | |
| `TRUST_LIST_FILE` | local, unsigned trust list; used when `TRUST_LIST_URL` is empty (tests) | |
| `TRUST_LIST_CACHE_SECONDS` | cache for the trust list | `300` |
| `PID_VCT`, `LPID_VCT`, `MANDATE_VCT` | accepted `vct` per credential type | `urn:eudi:pid:1`, `urn:tx:test:lpid:1`, `urn:tx:test:mandate:1` |
| `ACCEPTED_REPRESENTATION_TYPES` | accepted kinds of representation | `legal_representative,power_of_attorney` |
| `RESPONSE_MODE` | `direct_post.jwt` or `direct_post` | `direct_post.jwt` |
| `WALLET_URL_SCHEME` | scheme of the link to the EUDI Wallet | `openid4vp://` |
| `TRANSACTION_TTL_SECONDS` | time for the whole transaction until the code is issued | `600` |
| `AUTH_CODE_TTL_SECONDS` | validity of an authorization code | `60` |
| `ACCESS_TOKEN_TTL_SECONDS` | validity of access tokens (for `userinfo`) | `300` |
| `ATTESTATION_AUDIENCE` | `aud` of the verification attestation | `portal-registration` |
| `ATTESTATION_TTL_HOURS` | validity of the verification attestation | `720` |
| `STATUS_LIST_CACHE_SECONDS` | cache for status lists | `300` |
| `AUDIT_LOG_FILE` | audit log (JSON lines) | `audit.log` |

## 5. Data

Transactions live in memory only.

| Field | Meaning |
|---|---|
| `id` | transaction ID, 128 bit random |
| `client_id`, `redirect_uri`, `scope`, `state`, `nonce`, `code_challenge` | from the authorization request |
| `browser_binding_hash` | SHA-256 of the value of the cookie that binds the transaction to the browser |
| `status` | 6.2 |
| `failure_reason` | 6.2 |
| `steps` | list of steps (6.1), each with `name`, `request_id`, `nonce`, `state`, private key for the response (ECDH-ES, P-256), `status`, DCQL query |
| `result` | claims of the person and the company, result of the representation check; deleted after 6.9 |
| `auth_code_hash`, `access_token_hash` | SHA-256 of code and access token |
| `created_at`, `expires_at`, `verified_at` | |

Audit entry per finished transaction, without personal data: time, `client_id`, transaction ID, scopes, status, failure reason, per checked credential `type`, `vct` and issuer, representation result and how the mandate's issuer was trusted. No IP addresses.

**Not stored:** `vp_token`, credentials, disclosures and KB-JWT (only in memory during the check); attestations after they were handed out; status lists beyond the cache.

## 6. Rules

### 6.1 Scopes and steps

| Scope | Effect |
|---|---|
| `openid` | required. PID from the EUDI Wallet; `sub` (6.10) |
| `profile` | `given_name`, `family_name`, `birthdate` in the ID token |
| `representation` | mandate from the EUDI Wallet, required; representation claims (6.5) |
| `company` | LPID from the Business Wallet in a second step; company claims |
| `attestation` | verification attestation in the token response (section 9); requires `profile`, `representation` and `company` |

Steps of a transaction:

| Step | Wallet | Requested | When |
|---|---|---|---|
| `person` | EUDI Wallet | PID; mandate with scope `representation` | always, first |
| `company` | Business Wallet | LPID | with scope `company`, after `person` |

Typical requests:

| Use | Scopes | Steps |
|---|---|---|
| Sign-in at an OSP portal | `openid profile representation` | `person` |
| Sign-in at a Business Wallet | `openid profile representation` | `person` |
| Onboarding a company at an OSP | `openid profile representation company attestation` | `person`, `company` |

Each step sends one DCQL query to one wallet. Every credential in the query is required: the query has no `credential_sets` and no optional credentials, and claim paths name top-level claims only. Wallets differ in how they handle optional sets, combinations and nested paths; a fixed query per step works with all of them. Examples are in 14.4.

### 6.2 Status

| From | Event | To |
|---|---|---|
| – | authorization request accepted (7.2) | `PENDING` (step `person`) |
| `PENDING` | step verified, next step exists | `PENDING` (next step) |
| `PENDING` | last step verified | `VERIFIED` |
| `PENDING` | wallet reports an error, or a check fails | `FAILED` |
| `PENDING`, `VERIFIED` | `TRANSACTION_TTL_SECONDS` passed without a code | `EXPIRED` |
| `VERIFIED` | code issued to the client (7.4) | `CODE_ISSUED` |
| `CODE_ISSUED` | code redeemed (7.6) | `COMPLETED` |
| `CODE_ISSUED` | `AUTH_CODE_TTL_SECONDS` passed | `EXPIRED` |

Final: `FAILED`, `EXPIRED`, `COMPLETED`. A step has the status `CREATED`, `REQUEST_RETRIEVED`, `VERIFIED` or `FAILED`.

Failure reasons (`failure_reason`):

| Reason | Meaning |
|---|---|
| `WALLET_ERROR` | the wallet reports an error, for example the person declined |
| `INVALID_RESPONSE` | response cannot be decrypted, wrong `state`, wrong form, unknown credential ID, more than one presentation for one query |
| `UNSUPPORTED_FORMAT` | format is not `dc+sd-jwt` |
| `MISSING_CREDENTIAL` | a requested credential is missing or has another `vct` |
| `UNTRUSTED_ISSUER` | issuer of a PID or LPID is not in the trust list |
| `INVALID_SIGNATURE` | signature of the issuer is invalid |
| `CREDENTIAL_EXPIRED` | credential expired or not yet valid |
| `CREDENTIAL_REVOKED` | credential revoked or suspended according to its status list |
| `STATUS_UNAVAILABLE` | status list cannot be fetched or is invalid |
| `HOLDER_BINDING_FAILED` | KB-JWT missing or wrong (key, `nonce`, `aud`, `sd_hash`, time) |
| `MISSING_CLAIM` | a requested claim is missing |

### 6.3 Checking a credential

For every presented credential, in this order. The first error ends the check and the transaction becomes `FAILED` with the reason.

1. Exactly one presentation per credential in the query, format `dc+sd-jwt`; otherwise `INVALID_RESPONSE` or `UNSUPPORTED_FORMAT`.
2. Header: `typ` `dc+sd-jwt` (for a transition also `vc+sd-jwt`), `alg` `ES256`; never `none`.
3. `vct` is the one configured for the credential type; otherwise `MISSING_CREDENTIAL`.
4. Signature and trust: the signature verifies with a key of a trust list entry for this credential type (11.3). If the entry names `iss`, the credential's `iss` must be equal. Without a matching entry: `UNTRUSTED_ISSUER`; signature wrong: `INVALID_SIGNATURE`. For the mandate, see below.
5. Times with 60 seconds tolerance: `nbf` and `iat` not in the future, `exp` not passed; otherwise `CREDENTIAL_EXPIRED`.
6. Disclosures: every disclosure matches a digest in `_sd` (algorithm from `_sd_alg`, default `sha-256`), none twice; the claims are built from them.
7. Status: if the credential has `status.status_list` (`idx`, `uri`), fetch the list (11.4), check its signature with the issuer's key, `sub` equals `uri`, read the value at `idx`. Value 0: valid; otherwise `CREDENTIAL_REVOKED`. Not available or invalid: `STATUS_UNAVAILABLE`.
8. Key binding: `cnf.jwk` present; KB-JWT present with `typ` `kb+jwt`, signed with `cnf.jwk`, `aud` equal to the client identifier of the request (`x509_san_dns:{host}`), `nonce` equal to the nonce of the step, `iat` at most 5 minutes old, `sd_hash` correct; otherwise `HOLDER_BINDING_FAILED`.
9. Every requested claim is present; otherwise `MISSING_CLAIM`.

**Mandate.** Its issuer may be the company's Business Wallet, whose key is known only from the LPID of a later step. Therefore checks 1 to 3, 6 and 8 run when the mandate arrives; checks 4, 5, 7 and 9 run after the last step, with the key that 6.5 finds. If no key of 6.5 verifies the signature, the mandate counts as not trusted and its content is not used; the transaction does not fail for that reason. A trusted mandate that fails check 5, 7 or 9 fails the transaction.

### 6.4 Requested claims

| Credential | `vct` (configurable) | Claims |
|---|---|---|
| PID | `urn:eudi:pid:1` | `given_name`, `family_name`, `birthdate`, `issuing_country` |
| LPID | `urn:tx:test:lpid:1` (placeholder) | `legal_name`, `ebwoid`, `euid`, `address`, `country` |
| Mandate | `urn:tx:test:mandate:1` (placeholder) | `representative` (`given_name`, `family_name`, `birthdate`), `represented` (`identifier`, `legal_name`), `representation_type`, `role`, `valid_until` |

The `vct` values and claim names of LPID and mandate are placeholders until the schemas are agreed.

### 6.5 Representation check

**Trust in the mandate's issuer.** A mandate is trusted if one of these applies, checked in this order:

| Order | Rule | `representation_trust` |
|---|---|---|
| 1 | the issuer is in the trust list for credential type `MANDATE` (registers) | `trust_list` |
| 2 | the mandate is signed with the key the company's LPID in the same transaction is bound to (`cnf.jwk` of the LPID): the company's Business Wallet issued it; the company check below compares `represented.identifier` with the LPID | `company_key` |
| 3 | the mandate is signed with a key from the JWKS of the client: a Business Wallet trusts the mandates it issued itself | `client_key` |

For each rule, the EBW-Connector checks the mandate's signature with the respective keys; the first rule whose key verifies the signature applies. A Business Wallet signs mandates with the key its LPID is bound to (11.2). If no rule applies, the result is `NOT_CONFIRMED` with the only reason `ISSUER_NOT_TRUSTED`, and no further checks run.

**Checks**, after trust:

1. **Person:** given name, family name and birth date of `representative` equal those of the PID, after normalisation (Unicode NFC, outer spaces removed, case ignored). Otherwise `PERSON_MISMATCH`.
2. **Company:** with step `company`, `represented.identifier` equals `ebwoid` or `euid` of the LPID. Otherwise `COMPANY_MISMATCH`.
3. **Kind:** `representation_type` is in `ACCEPTED_REPRESENTATION_TYPES`. Otherwise `TYPE_NOT_ACCEPTED`.
4. **Validity:** `valid_until` is missing or not before today (UTC). Otherwise `EXPIRED`.

| Result | When |
|---|---|
| `CONFIRMED` | trusted and all checks passed |
| `NOT_CONFIRMED` | otherwise; reasons in `representation_reasons`, including `ISSUER_NOT_TRUSTED` |

A mandate that is not confirmed does not fail the transaction. The client decides: the registration (C1-M2) accepts an attestation with `representation.confirmed: false` and leaves the decision to the review (C1-M3); a Business Wallet refuses the sign-in.

### 6.6 Invoking the wallets

**EUDI Wallet (step `person`).** The page shows a QR code with the link `{WALLET_URL_SCHEME}?client_id=…&request_uri=…` (cross-device) and a button with the same link (same-device).

**Business Wallet (step `company`).** A browser does not open a web application through a custom scheme. The page therefore asks for the address of the person's Business Wallet, its *wallet address* (11.2), and shows a button that opens `{wallet address}?client_id=…&request_uri=…`. The page keeps the address in the browser's local storage, so the person enters it only once; the server does not store it. As a fallback, the page shows the request link to copy into the Business Wallet.

After the response, the wallet follows the `redirect_uri` from the EBW-Connector's answer (7.8). It leads to the page of the transaction. With the cookie of the transaction (same browser), the page continues; without it (another device), it shows only that the step is done.

### 6.7 Claims in the ID token and in `userinfo`

| Claim | Scope | Source |
|---|---|---|
| `iss`, `sub`, `aud`, `exp`, `iat`, `auth_time`, `nonce` | `openid` | `sub` per 6.10 |
| `verification_id` | `openid` | ID of the check |
| `given_name`, `family_name`, `birthdate` | `profile` | PID |
| `representation_result` | `representation` | `CONFIRMED` or `NOT_CONFIRMED` (6.5) |
| `representation_reasons` | `representation` | list, only with `NOT_CONFIRMED` |
| `representation_type`, `representation_role`, `representation_valid_until` | `representation` | mandate |
| `representation_company_id`, `representation_company_name` | `representation` | `represented.identifier`, `represented.legal_name` of the mandate |
| `representation_issuer`, `representation_trust` | `representation` | `iss` of the mandate; rule from 6.5 |
| `company_legal_name`, `company_ebwoid`, `company_euid`, `company_country` | `company` | LPID |
| `company_address` | `company` | LPID, object with `street_address`, `postal_code`, `locality`, `country` |

Missing values are left out, never `null`. With `ISSUER_NOT_TRUSTED`, only `representation_result`, `representation_reasons` and `representation_issuer` are given, because the other values are not verified. `userinfo` returns the same claims without `iss`, `aud`, `exp`, `iat`, `auth_time` and `nonce`.

### 6.8 Clients

Registered in `CLIENTS_FILE` (example 14.2):

| Field | Meaning |
|---|---|
| `clientId` | identifier |
| `name` | display name on the pages |
| `jwksUri` or `jwks` | public keys of the client for `private_key_jwt`; for a Business Wallet also the key it signs mandates with (6.5 rule 3) |
| `redirectUris` | allowed `redirect_uri`, exact match |
| `scopes` | allowed scopes |

**Client assertion** (RFC 7523): `iss` and `sub` = `clientId`, `aud` = `{PUBLIC_URL}/token` or `PUBLIC_URL`, `exp` at most 5 minutes in the future, `jti` used once, signed with `ES256` and a key of the client. There are no client secrets.

### 6.9 Retention

| What | How long |
|---|---|
| transaction without code | until `TRANSACTION_TTL_SECONDS` (10 minutes) |
| claims for the token response | until the code is redeemed, at most `AUTH_CODE_TTL_SECONDS` |
| claims for `userinfo` | until the access token expires (`ACCESS_TOKEN_TTL_SECONDS`) |
| `vp_token`, credentials, disclosures | only in memory during the check; the mandate until the last step (6.3) |
| audit log | as decided by the operator; it holds no personal data |

### 6.10 Identifier of the person (`sub`)

`sub` is pairwise per client:

`sub` = base64url(HMAC-SHA256(subject key, `clientId` + "|" + `issuing_country` + "|" + family name + "|" + given name + "|" + birth date))

Names and birth date are normalised as in 6.5. `sub` stays the same as long as these values stay the same; a change of name gives a new `sub`.

### 6.11 Uniqueness

- A wallet may fetch a request several times while the step is open; it gets the same request object.
- Only the first response of a wallet counts per step; every further one gets 400 `invalid_request`.
- A code is valid once. A second attempt gives `invalid_grant` and invalidates the access token issued from it.
- A client assertion (`jti`) is accepted once.

## 7. Operations

### 7.1 `GET /.well-known/openid-configuration`

`issuer` = `PUBLIC_URL`, `authorization_endpoint` `{PUBLIC_URL}/authorize`, `token_endpoint` `{PUBLIC_URL}/token`, `userinfo_endpoint` `{PUBLIC_URL}/userinfo`, `jwks_uri` `{PUBLIC_URL}/jwks`, `response_types_supported` `["code"]`, `grant_types_supported` `["authorization_code"]`, `subject_types_supported` `["pairwise"]`, `id_token_signing_alg_values_supported` `["ES256"]`, `token_endpoint_auth_methods_supported` `["private_key_jwt"]`, `token_endpoint_auth_signing_alg_values_supported` `["ES256"]`, `code_challenge_methods_supported` `["S256"]`, `scopes_supported`, `claims_supported`, `authorization_response_iss_parameter_supported` `true`, `ui_locales_supported` `["de", "en"]`.

### 7.2 `GET /authorize`

Parameters: `response_type=code`, `client_id`, `redirect_uri`, `scope`, `state`, `nonce`, `code_challenge`, `code_challenge_method=S256`, optional `ui_locales`.

1. Client unknown or `redirect_uri` not registered: error page 400, no redirect.
2. Other errors: redirect to `redirect_uri` with `error` (`invalid_request` without `nonce` or PKCE; `invalid_scope` for scopes not allowed for the client, without `openid`, or `attestation` without `profile`, `representation` and `company`), `state`, `iss`.
3. Create the transaction (status `PENDING`, steps per 6.1, `expires_at` = now + `TRANSACTION_TTL_SECONDS`). Set the cookie `ebw_tx` (random value, `HttpOnly`, `SameSite=Lax`, path `/`, `Secure` with HTTPS); store its hash.
4. Redirect 302 to `{PUBLIC_URL}/transactions/{id}`.

### 7.3 `GET /transactions/{id}`

HTML page of the transaction. Without the matching cookie: a neutral page "This step is done; continue on your other device", no data, status 200.

With the cookie:

- `PENDING`: name of the client, what is requested, the open step with its invocation (6.6); the page asks `GET /transactions/{id}/status` every 2 seconds and reloads when the step or status changes.
- `VERIFIED`: redirect to `…/complete`.
- `FAILED`, `EXPIRED`: redirect to `…/complete`.

### 7.4 `GET /transactions/{id}/status` and `GET /transactions/{id}/complete`

`status`: `{ "status", "step" }` for the page. Only with the matching cookie, otherwise 404.

`complete`: only with the matching cookie, otherwise 404.

- `VERIFIED`: create the code (256 bit, store its hash, `AUTH_CODE_TTL_SECONDS`), status `CODE_ISSUED`, 302 to `redirect_uri?code=…&state=…&iss=…`.
- `FAILED`, `EXPIRED`: 302 to `redirect_uri?error=access_denied&error_description=<reason>&state=…&iss=…`.
- `PENDING`: 302 back to the page.

### 7.5 `GET /openid4vp/requests/{requestId}` and `POST /openid4vp/requests/{requestId}`

Returns the signed request object (`application/oauth-authz-req+jwt`) of the step. With POST (`request_uri_method=post`) the wallet may send `wallet_nonce`; it is then included in the request object.

Header: `alg` `ES256`, `typ` `oauth-authz-req+jwt`, `x5c`. Payload:

| Claim | Value |
|---|---|
| `client_id` | `x509_san_dns:{host of PUBLIC_URL}` |
| `response_type` | `vp_token` |
| `response_mode` | `RESPONSE_MODE` |
| `response_uri` | `{PUBLIC_URL}/openid4vp/responses/{requestId}` |
| `nonce`, `state` | of the step |
| `dcql_query` | per 6.1 and 6.4 (examples 14.4) |
| `client_metadata` | `client_name`; `vp_formats_supported` for `dc+sd-jwt` with `sd-jwt_alg_values` and `kb-jwt_alg_values` `["ES256"]`; with `direct_post.jwt` also `jwks` with the public key of the step (`use` `enc`, `alg` `ECDH-ES`, `kid`) and `encrypted_response_enc_values_supported` `["A128GCM", "A256GCM"]` |
| `aud` | `https://self-issued.me/v2` |
| `iat`, `exp` | now, end of the transaction |

Step `CREATED` becomes `REQUEST_RETRIEVED`. Unknown step, step not open or transaction not `PENDING`: 404.

### 7.6 `POST /token`

Form with `grant_type=authorization_code`, `code`, `redirect_uri`, `code_verifier`, `client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer`, `client_assertion`.

1. Client assertion per 6.8; otherwise 401 `invalid_client`.
2. Code valid, not used, issued to this client and this `redirect_uri`, PKCE matches; otherwise `invalid_grant`.
3. Response 200, `Cache-Control: no-store`:

| Field | Content |
|---|---|
| `access_token` | random, for `userinfo`; valid `ACCESS_TOKEN_TTL_SECONDS` |
| `token_type` | `Bearer` |
| `expires_in` | seconds |
| `id_token` | ES256, `kid` of the signing key, claims per 6.7 |
| `ebw_attestation` | only with scope `attestation`: verification attestation (section 9) |

Status `COMPLETED`. Errors per RFC 6749 as JSON: `invalid_request`, `invalid_client` (401), `invalid_grant`, `unsupported_grant_type`.

### 7.7 `GET /userinfo`

With `Authorization: Bearer <access_token>`: claims per 6.7. Unknown or expired token: 401 with `WWW-Authenticate: Bearer error="invalid_token"`.

### 7.8 `POST /openid4vp/responses/{requestId}`

Form (`application/x-www-form-urlencoded`) with `response` (JWE, with `direct_post.jwt`), or `vp_token` and `state` (with `direct_post`), or on wallet errors `error`, `error_description`, `state`.

1. Step of `requestId` open and transaction `PENDING`; otherwise 400 `invalid_request`.
2. `error` and matching `state`: `FAILED`, `WALLET_ERROR`.
3. With `direct_post.jwt`: decrypt `response` with the private key of the step (ECDH-ES with `A128GCM` or `A256GCM`); check `state`. Errors: `FAILED`, `INVALID_RESPONSE`.
4. `vp_token`: JSON object with the credential IDs of the query (`pid`, `mandate`, `lpid`) as keys and a list with one presentation each.
5. Check every credential per 6.3.
6. After the last step: representation check per 6.5, build the result, status `VERIFIED`, write the audit entry. Otherwise the next step becomes open. `vp_token` is discarded.
7. Response 200 `{ "redirect_uri": "{PUBLIC_URL}/transactions/{id}" }`, also after `FAILED`.

### 7.9 `GET /jwks`

Public signing key (`kty`, `crv`, `x`, `y`, `kid`, `use` `sig`, `alg` `ES256`). `Cache-Control: max-age=3600`.

## 8. Steps for Process Orchestration

None. Clients call the EBW-Connector through OpenID Connect; Process Orchestration does not call it.

## 9. Verification attestation

### 9.1 Format

Compact JWS, signed with the signing key, handed out in the token response as `ebw_attestation` (7.6).

Header: `alg` `ES256`, `typ` `ebw-attestation+jwt`, `kid`.

Claims (snake_case; times as seconds since 1970; dates as `YYYY-MM-DD`; missing optional values are left out, never `null`):

| Claim | Required | Content |
|---|---|---|
| `iss` | yes | `PUBLIC_URL` |
| `sub` | yes | `company.ebwoid` |
| `aud` | yes | `ATTESTATION_AUDIENCE` |
| `iat`, `exp` | yes | issued; `iat` + `ATTESTATION_TTL_HOURS` |
| `jti` | yes | ID of the check (`verification_id`, UUID) |
| `attestation_version` | yes | `"0.2"` |
| `purpose` | yes | `"onboarding"` |
| `verified_at` | yes | time of the last step |
| `client_id` | yes | client of the transaction |
| `person` | yes | `given_name`, `family_name` |
| `company` | yes | `legal_name`, `ebwoid`, `country` required; `euid`, `address` (`street_address`, `postal_code`, `locality`, `country`) optional; from the LPID |
| `representation` | yes | `confirmed` (`true` exactly with `CONFIRMED`), `result`; `type`, `role`, `valid_until`, `issuer`, `trust` (6.5); `reasons` with `NOT_CONFIRMED` |
| `evidence` | yes | per checked credential: `type` (`PID`, `LPID`, `MANDATE`), `vct`, `issuer`, `wallet` (`EUDI_WALLET` or `BUSINESS_WALLET`), `status_checked` |

Not included: birth date, nationality, address of the person, the credentials themselves.

### 9.2 Check by the recipient

The registration (C1-M2) checks: `iss` is a recognised instance with its JWKS (`{iss}/jwks`); `typ`, `alg`, `kid` and signature; `aud`, `exp`, `iat`; `jti` used once; then its own rules. Unchanged from version 0.1, except that `psp_id` is gone and `evidence` names the wallet.

## 10. Events

None.

## 11. Calls to external systems

### 11.1 EUDI Wallet

The wallet calls the EBW-Connector (7.5, 7.8). Requirements: OpenID4VP 1.0 with DCQL, `request_uri`, `x509_san_dns`, `direct_post.jwt` (or `direct_post` if configured), SD-JWT VC with KB-JWT.

### 11.2 Business Wallet

Same requirements as 11.1, and in addition:

- **Wallet address:** an HTTPS address that accepts `client_id` and `request_uri` as query parameters, shows the request to the signed-in user and sends the response after consent. The Business Wallet shows this address to its users, for example on its start page.
- **Redirect:** after sending the response it opens the `redirect_uri` from the EBW-Connector's answer in the browser.
- **Key:** it signs the mandates it issues with the key its LPID is bound to (`cnf.jwk`), and publishes this key in its JWKS as a client (6.5 rules 2 and 3).

### 11.3 Trust list of the core services

`GET {TRUST_LIST_URL}` returns a compact JWS (`typ` `trust-list+jwt`, `alg` `ES256`) whose signature verifies with a key from `TRUST_LIST_SIGNER_FILE`. Payload:

```json
{
  "iss": "https://core.example.org",
  "iat": 1793174400,
  "exp": 1793260800,
  "entries": [
    { "name": "test-pid-provider", "credential_types": ["PID"],
      "iss": "https://pid.test.example.org", "jwks": { "keys": [ "…" ] } },
    { "name": "test-register", "credential_types": ["LPID", "MANDATE"],
      "iss": "https://register.test.example.org", "jwks": { "keys": [ "…" ] } }
  ]
}
```

`iss` in an entry is optional; without it, only the key decides (tests with changing addresses). Cache `TRUST_LIST_CACHE_SECONDS`; if the list is not available and the cache is empty, every check of a PID or LPID fails with `UNTRUSTED_ISSUER`. `TRUST_LIST_FILE` holds the same payload as plain JSON.

### 11.4 Status lists

`GET {uri}` from `status.status_list` with `Accept: application/statuslist+jwt`. Timeout 5 seconds, HTTPS (HTTP only for `localhost`), cache `STATUS_LIST_CACHE_SECONDS`.

### 11.5 JWKS of the clients

`GET {jwksUri}` for client assertions and for 6.5 rule 3. Timeout 5 seconds, cache 5 minutes; an unknown `kid` triggers one reload.

## 12. Errors

**OIDC endpoints:** errors per RFC 6749 and OpenID Connect Core (`error`, `error_description`), see 7.2 and 7.6.

**OpenID4VP endpoints:** 400 with `{ "error": "invalid_request", "error_description" }`; unknown request 404.

**Pages:** HTML error page with status 400 or 404.

## 13. Acceptance

The module is done when these cases work with the test issuer (14.1) and a test wallet that plays the EUDI Wallet and the Business Wallet; cases 2, 5 and 9 in addition with a real EUDI Wallet on a phone and the Business Wallet.

1. Discovery and JWKS are valid. `authorize` without PKCE: redirect with `invalid_request`. Unknown `redirect_uri`: error page without redirect.
2. Sign-in with `openid profile representation`, mandate of the register: one step; ID token with `aud` = client, matching `nonce`, `representation_result` `CONFIRMED`, `representation_trust` `trust_list`; `userinfo` returns the same claims.
3. The request object is signed with the certificate, has `client_id` `x509_san_dns:{host}`, `response_mode` `direct_post.jwt` and a DCQL query without `credential_sets`.
4. The response is encrypted (`direct_post.jwt`); a response that cannot be decrypted gives `INVALID_RESPONSE`.
5. Onboarding with `openid profile representation company attestation`: steps `person` and `company`; the token response contains `ebw_attestation`, which verifies against `/jwks` and contains the values of 14.5 with `representation.confirmed` `true`.
6. Mandate issued by the Business Wallet with the key of its LPID: onboarding gives `CONFIRMED` with `representation_trust` `company_key`; sign-in at the Business Wallet gives `CONFIRMED` with `client_key`; sign-in at another client gives `NOT_CONFIRMED` with `ISSUER_NOT_TRUSTED`.
7. Mandate for another company: `NOT_CONFIRMED` with `COMPANY_MISMATCH` (onboarding). Mandate for another person: `PERSON_MISMATCH`.
8. PID of an issuer not in the trust list: `FAILED`, `UNTRUSTED_ISSUER`; the client receives `error=access_denied`.
9. Mandate revoked in its status list: `FAILED`, `CREDENTIAL_REVOKED`.
10. KB-JWT with a wrong `nonce`: `FAILED`, `HOLDER_BINDING_FAILED`. A second response to the same `response_uri`: 400.
11. Wallet answers `access_denied`: `FAILED`, `WALLET_ERROR`.
12. `sub` is the same for the same person at the same client in two transactions, and different at a second client.
13. Code used twice: `invalid_grant`; the access token issued before no longer works for `userinfo`. Client assertion with the same `jti` twice: `invalid_client`.
14. `complete` from another browser without the cookie: 404. The page without the cookie shows no data.
15. Privacy: the audit log contains no given name, family name, birth date or identifier of the person.

## 14. Test data

### 14.1 Test issuer and trust list

A separate test tool plays the issuers and the core services. It is not part of the module.

| Role | Issues | To |
|---|---|---|
| test PID provider | PID | EUDI Wallet |
| test register | LPID; mandate for legal representation | Business Wallet; EUDI Wallet |
| test core services | signed trust list with both issuers (11.3) | EBW-Connector |

All with the person Erika Mustermann, born 1984-01-26, PID from Germany, and the company Beispiel Maschinenbau GmbH, `EBWOID-DE-0000000001`, EUID `DEXX001T.HRB100001`, Musterstraße 1, 70569 Stuttgart, DE. Company and identifiers are invented.

### 14.2 Clients (`CLIENTS_FILE`)

```json
[
  { "clientId": "demo-osp", "name": "Demo-OSP",
    "jwksUri": "https://osp.example.org/jwks",
    "redirectUris": ["https://osp.example.org/callback"],
    "scopes": ["openid", "profile", "representation", "company", "attestation"] },
  { "clientId": "business-wallet", "name": "Business Wallet Beispiel Maschinenbau GmbH",
    "jwksUri": "https://bw.example.org/jwks",
    "redirectUris": ["https://bw.example.org/login/callback"],
    "scopes": ["openid", "profile", "representation"] }
]
```

### 14.3 Test trust list (`TRUST_LIST_FILE`)

```json
{
  "iss": "https://core.test.example.org",
  "iat": 1793174400,
  "entries": [
    { "name": "test-pid-provider", "credential_types": ["PID"], "jwks": { "keys": [ "…" ] } },
    { "name": "test-register", "credential_types": ["LPID", "MANDATE"], "jwks": { "keys": [ "…" ] } }
  ]
}
```

### 14.4 DCQL queries

Step `person` with scope `representation`:

```json
{
  "credentials": [
    { "id": "pid", "format": "dc+sd-jwt", "meta": { "vct_values": ["urn:eudi:pid:1"] },
      "claims": [ { "path": ["given_name"] }, { "path": ["family_name"] },
                  { "path": ["birthdate"] }, { "path": ["issuing_country"] } ] },
    { "id": "mandate", "format": "dc+sd-jwt", "meta": { "vct_values": ["urn:tx:test:mandate:1"] },
      "claims": [ { "path": ["representative"] }, { "path": ["represented"] },
                  { "path": ["representation_type"] }, { "path": ["role"] }, { "path": ["valid_until"] } ] }
  ]
}
```

Step `company`:

```json
{
  "credentials": [
    { "id": "lpid", "format": "dc+sd-jwt", "meta": { "vct_values": ["urn:tx:test:lpid:1"] },
      "claims": [ { "path": ["legal_name"] }, { "path": ["ebwoid"] }, { "path": ["euid"] },
                  { "path": ["address"] }, { "path": ["country"] } ] }
  ]
}
```

### 14.5 Example verification attestation (payload)

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

## 15. Changes from version 0.1

- Two wallets: the LPID comes from the company's Business Wallet in a second step; PID and mandate come from the person's EUDI Wallet. The mandate is required for onboarding.
- Clients use OpenID Connect only. The separate API for verification sessions is gone; the verification attestation comes with the token response (scope `attestation`).
- Every operator runs its own instance; trust comes from the trust list of the core services. No dependency on PSP admission (C1-M1) or the Portal IAM; `psp_id` and the purpose `delegation` are gone.
- Mandates from a Business Wallet are trusted through the key of the company's LPID or of the client (6.5).
- DCQL without `credential_sets`, optional credentials and nested paths (6.1), after tests with a wallet that does not support them.
- The wallet stub and test issuer are a separate test tool.

## 16. Open questions

1. **Schemas** of LPID, EBWOID and mandate: placeholders until aligned, for example with the rulebooks of the European Business Wallet pilots.
2. **`sub` without a personal identifier:** the PID often has no stable personal number; `sub` depends on name and birth date.
3. **Business Wallet as issuer:** binding the issuer key to the LPID (6.5 rule 2) is a proposal; the European Business Wallet may define its own way, for example with certificates.
4. **Several instances:** transactions in a database instead of memory.
5. **Format of the trust list:** own JSON for now; later ETSI trusted lists of the EU.
6. **Wallet address:** discovery of a Business Wallet without entering its address, for example through the Digital Credentials API of the browsers.
