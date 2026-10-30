# C5-M1 EBW-Connector: open points

**As of:** 2026-10-30, version 0.2

The README fixes each of these points for version 0.2 so that the module can be built. The following decisions are provisional and still to be agreed. Decided on 28 Oct 2026: OSP 2.0 (every OSP runs the portal modules itself, the CSP provides the core services), two wallets, the mandate is required at onboarding, the EBW-Connector and a Business Wallet are provided for the hackathon.

## Provisional decisions

| No. | Topic | Decision in the README | To be agreed with |
|---|---|---|---|
| 1 | Operation | every operator runs its own instance (an OSP for its portal, a Business Wallet for its sign-in); one instance may serve several clients; no calls to other modules (README 1) | project owner, OSPs |
| 2 | Interface to clients | OpenID Connect only: authorization code flow with PKCE and `private_key_jwt`; verification attestation in the token response as `ebw_attestation` (README 6.1, 7.6). The own interface for verification sessions from 0.1 is dropped. | C1-M2, OSPs |
| 3 | Two wallets | step `person` (EUDI Wallet: PID, mandate), step `company` (Business Wallet: LPID) (README 6.1) | decided 28 Oct 2026 |
| 4 | Invoking the Business Wallet | wallet address, entered once and remembered in the browser; link to copy as fallback (README 6.6, 11.2) | Business Wallet providers |
| 5 | Trust in mandates | three rules: trust list (register), key of the LPID in the same transaction, key of the client (README 6.5) | project owner, European Business Wallet pilots |
| 6 | DCQL | no `credential_sets`, no optional credentials, top-level claims only (README 6.1) | wallet providers |
| 7 | Format of the verification attestation | version `0.2`: without `psp_id`, with `evidence[].wallet` and `representation.trust`; otherwise as 0.1 (README 9.1) | C1-M2 |
| 8 | Trust list of the core services | own JSON, signed as `trust-list+jwt`; entries with keys, `iss` optional (README 11.3) | core services (CSP) |
| 9 | Profiles of LPID and mandate | `vct` and claim names are placeholders (`urn:tx:test:lpid:1`, `urn:tx:test:mandate:1`) (README 6.4) | project owner, schemas |
| 10 | Claims of the PID | `given_name`, `family_name`, `birthdate`, `issuing_country`; no personal identifier (README 6.4, 6.10) | check against the EU PID rulebook |
| 11 | State | transactions in the memory of one instance, no database; audit log as JSON lines (README 4) | operations |
| 12 | Response mode | `direct_post.jwt`, `direct_post` configurable (README 4) | wallet providers |
| 13 | Claims in the ID token | flat names (`representation_*`, `company_*`) (README 6.7) | OSPs |
| 14 | Single sign-on | none; every transaction asks the wallet | OSPs |
| 15 | Clients of an OSP | Portal IAM (sign-in, scopes `openid profile`) and the server part of the user interface (onboarding with verification attestation; client ID `osp-a-ui`, `osp-b-ui`, FRAMEWORK 7a) are separate clients (README 1, 6.1; FRAMEWORK 5.2) | C2-M5, user interface |

## Deviations from the framework

| No. | Deviation | Reason |
|---|---|---|
| A1 | No tokens of the Portal IAM; the EBW-Connector is an OIDC provider itself with its own tokens | a standalone module that every operator runs itself |
| A2 | Paths `/authorize`, `/token`, `/userinfo`, `/jwks`, `/.well-known/...`, `/transactions/...`, `/openid4vp/...` instead of `/api/c5-m1`; tags `OpenID Connect`, `Wallet`, `Pages` | standards and the role as OIDC provider and verifier |
| A3 | snake_case in JWT claims, OIDC and OpenID4VP parameters | JWT and OIDC convention |
| A4 | Points in time in the verification attestation and the ID token as seconds since 1970 | JWT convention (NumericDate) |
| A5 | Errors of the OIDC and OpenID4VP endpoints follow their standards instead of Problem Details | clients and wallets expect the standard form |
| A6 | No events through an outbox | clients call the EBW-Connector through OIDC; there is nothing to report |
| A7 | Outline: section 8 (steps) is empty; section 9 describes the verification attestation | the verification attestation is the contract with C1-M2 |
| A8 | No own database; transactions in memory | transactions last minutes; a database follows when several instances are needed |

## Assumptions about other modules

| Module | Assumption |
|---|---|
| C1-M2 Registration | checks the verification attestation according to README 9.2; accepts `attestation_version` `0.2`; does not evaluate `evidence` and `representation.trust`; also accepts `representation.confirmed: false` and leaves the assessment to C1-M3 |
| Portal of an OSP | is an OIDC client of the EBW-Connector of its choice; registers `redirect_uri` and JWKS address; shows no QR codes itself |
| Business Wallet | offers a wallet address; signs mandates with the key of its LPID; publishes this key in its JWKS as a client |
| Core services (CSP) | provide a signed trust list with the keys of the PID providers and registers |
| C2-M5 Portal IAM | includes the EBW-Connector as its only identity provider (C2-M5 5.5): client `portal-iam`, `private_key_jwt` with ES256 and the JWKS of the realm, scopes `openid profile`, PKCE `S256`; takes over only `given_name` and `family_name` |

## Open questions

Questions that concern several modules are listed with their number (OP-xx) in `../open-points.md`.

1. **Schemas** of LPID, EBWOID and mandate: placeholders until agreed, for example with the rulebooks of the European Business Wallet pilots (cross-module OP-12).
2. **`sub` without a personal identifier:** the PID often has no stable personal identifier; `sub` depends on name and birth date.
3. **Business Wallet as issuer:** binding the issuer key to the LPID (README 6.5 rule 2) is a proposal; the European Business Wallet may define its own way, for example with certificates.
4. **Several instances:** transactions in a database instead of memory.
5. **Format of the trust list:** own JSON for now; later EU trusted lists according to ETSI.
6. **Wallet address:** finding a Business Wallet without entering its address, for example through the Digital Credentials API of the browsers.
7. ~~**Deviation of the reference implementation from the spec:**~~ Decided on 28 Oct 2026 (OP-43): for the hackathon the spec is as lenient as the reference implementation. It also accepts presentations as a string, with several per credential the first valid one (the mandate according to 6.5), and `"apu": null` in the JWE header (README 6.3 check 1, 7.8 step 4). Reason: a tested EUDI Wallet answers this way. Later the spec may become strict again (exactly one presentation, always as a list), once wallets follow OpenID4VP 1.0.
8. **Access to apps and central services:** with OSP 2.0 there are tokens of several OSPs; whether apps and central services trust all OSP IAMs or switch to credentials is open.
9. **Several CSPs:** federation of trust lists and connectivity between dataspaces are later work.

## Out of scope

- Qualified signatures, seals, electronic delivery.
- Issuing credentials; registers, PID providers and Business Wallets do this, in the test a test tool.
- Revoking an issued verification attestation.
- Design and accessibility of the pages; languages other than German and English.
- Delegation to PSPs; it is dropped with OSP 2.0.
