# C2-M5 Portal IAM: open points

**Version:** 0.2, as of 2026-10-28

For version 0.2, the README fixes each of these points so that the configuration can be created. The following decisions are provisional and still to be agreed. Decided on 28 Oct 2026 are: OSP 2.0 (one Portal IAM per OSP), the EBW-Connector as identity provider, accounts without a password.

## List of permissions

Aligned on 28 Oct 2026 with all module specs of our own part (README 6.2): C1-M2, C1-M3, C2-M2, C2-M3, C3-M1, C3-M2, C4-M1, C4-M2. Only the proposals for Connectors, Subscriptions and Offers (other part) remain open.

## Provisional decisions

| No. | Topic | Decision in the README | To be agreed with |
|---|---|---|---|
| 1 | Keycloak version | target 26; only functions that also exist in 25 (4) | OSPs, base skeleton |
| 2 | Deployment | one realm file for all OSPs with placeholders, set through the OSP's Helm chart; seeding job in operation, `--import-realm` in the test (4, 6.6) | base skeleton, OSPs |
| 3 | Realm name | `IAM_REALM`, default `CX-Central`; the issuer differs per OSP through the address (4) | OSPs, apps, central services |
| 4 | Account on the first login | the first login creates an account without a form and links it; user name `ebw.<sub>`; access is denied only if the user name already exists without a link (5.6). Cleaning up accounts that are never used is open (OP-40). | C2-M2, OSPs |
| 5 | `Applicant` through the identity provider | mapper with a fixed role, sync `IMPORT`, instead of a default role of the realm, so that service accounts do not get it (5.5) | C2-M2 |
| 6 | Names on every sign-in | sync `FORCE` for `firstName` and `lastName` (5.5) | C1-M2 (name check), C2-M2 |
| 7 | Client assertion with ES256 | second realm key `ecdsa-generated` P-256 only for the client assertion towards the EBW-Connector; tokens stay RS256 (5.3). Check in the version in use that the identity provider accepts `clientAssertionSigningAlg` `ES256`. | C5-M1, base skeleton |
| 8 | Portal roles | split as in 6.3; `OSP Admin` without `review_application`; `IT Admin` without rights on people except reading | project owner, OSPs |
| 9 | Role attributes | every permission with `module` and `scope` (6.1) | project owner |
| 10 | Client of the UI | `portal-ui`, public, PKCE `S256` (5.8) | UI |
| 11 | Scopes | `portal-person` (names, company) only for `portal-ui`; `catena` without names for apps; `portal-core-audience` for service accounts (5.7) | C2-M2, apps |
| 12 | Service accounts for callbacks | fixed names `sa-csp`, `sa-csp-issuer`, `sa-csp-wallet`, `sa-credential-issuer`, `sa-ih-callback`, `sa-div-callback`; created according to `CALLBACK_CLIENTS`; exactly one permission, secured through the scope mapping (5.9) | C3-M2, C4-M1, C4-M2, CSP |
| 13 | Sign-in of the CSP's service accounts | `private_key_jwt` with the JWKS address of the CSP from the trust list; one key of the CSP for all three service accounts (5.9) | CSP |
| 14 | Token lifetimes | as today: access token 300 s, session 1800 s without activity, at most 36000 s (5.2) | OSPs, apps |
| 15 | Technical user roles | roles that only contain rights on today's portal are dropped; `Dataspace Discovery` without `view_connectors`; `Identity Wallet Management` without content (6.4) | C2-M3, Connectors |
| 16 | Clients of the central services | are in every realm with their roles, names as today (5.8) | CSP, OP-37 |
| 17 | Stand-in for the EBW-Connector in the test | realm `TEST-EBW` in the same Keycloak, with test passwords only there; end to end with the reference implementation (14.3, case 22) | base skeleton |
| 18 | Validation rules of the library | `iss` of the own installation, `aud` contains `portal-core`, `typ` `Bearer`, tolerance at most 30 s (7.6) | library (provided) |
| 19 | Admin events | without representation, 90 days retention (5.1) | OSPs, data protection |

## Deviations from the framework

| No. | Framework | Deviation | Reason |
|---|---|---|---|
| A1 | FRAMEWORK 3: structure of the README and `openapi.yaml` with base `/api/<id>` | sections 5 and 7 adapted (content of the configuration, tokens and endpoints), 8 and 9 not applicable; `openapi.yaml` describes Keycloak paths instead of `/api/c2-m5` | The module is a configuration |
| A2 | FRAMEWORK 4.1: tags `Company`, `OSP`, `Queries`, `Steps` | in `openapi.yaml` only `Token` and `Queries` | The endpoints of the product do not fit the tags |

## Assumptions about other modules

| Module | Assumption |
|---|---|
| C5-M1 EBW-Connector | registers the Portal IAM as a client (5.5); accepts `private_key_jwt` with ES256 and the JWKS of the realm; returns `given_name` and `family_name` in the ID token with `openid profile`; `sub` pairwise per client |
| C2-M2 | binds accounts through the attributes `company_id` and `bpn`; assigns only whole portal roles; never assigns or removes `Applicant`; ends sessions on removal; creates the first OSP Admin through its bootstrap |
| C2-M3 | technical users as specified in 5.8; never assigns permissions on `portal-core` |
| C3-M2, C4-M1, C4-M2 | on callbacks, check only their permission and, where needed, `azp` with the fixed names from 5.9 |
| UI | signs people in through `portal-ui` (authorization code with PKCE); gets the verification attestation directly from the EBW-Connector with its own client |
| Library | validates as in 7.6 |
| Apps | read `sub`, `company_id`, `bpn` and the app roles; no names, not `organisation` |
| CSP | gets a token for callbacks from the Portal IAM of every OSP; token endpoint from the trust list |

## Open questions

Questions that affect several modules are listed with their number (OP-xx) in `../open-points.md`.

1. **New account after a change of name:** the `sub` of the EBW-Connector depends on the name and birth date of the PID. After a change of name, a new account is created without binding and roles. A remedy would be a stable personal identifier from the PID, or a transfer of the account by the staff (C2-M2). Cross-cutting OP-40.
2. **Tokens of several OSPs at central services and apps:** cross-cutting OP-37, OP-39.
3. **Callbacks of external systems:** `private_key_jwt` for all systems, secret only as a transition; cross-cutting OP-22.
4. **Replacement for `CX Membership Info`:** cross-cutting OP-24.
5. **Narrower rights for the C2 service accounts:** `manage-users` allows any role assignment; Keycloak has no finer limit without an extension; cross-cutting OP-26.
6. **Keycloak Organizations (from 26):** cross-cutting OP-28.
7. **Product question:** Keycloak stays for now; alternatives have not been assessed. The encapsulation through C2 keeps the product replaceable.
8. **Rate limiting** at the token endpoint and at sign-in: cross-cutting OP-27.

## Out of scope

- Operation: high availability, database, backup, monitoring, realm `master` and access for the OSP's administrators.
- Themes and texts of the error pages.
- Migration of existing accounts from today's installation.
