# C2-M5 Portal IAM

**Version:** 0.2, as of 2026-10-30
**Capability:** C2 Identity and Access
**Kind:** Configuration
**Files:** this README (content of the configuration, tokens, acceptance), [openapi.yaml](openapi.yaml) (endpoints of the product in use, for reference only), [open-points.md](open-points.md)

The module is not a program. It is the configuration of the Portal IAM of an OSP: Keycloak, one realm per installation. This README describes what the configuration contains, how it is deployed and which tokens result from it. The framework in [`../FRAMEWORK.md`](../FRAMEWORK.md) is also binding, above all sections 4.2 and 4.3.

The structure follows FRAMEWORK section 3, adapted for a configuration: section 5 describes the content of the configuration instead of database tables, section 7 the tokens and endpoints instead of operations.

## 1. Purpose

With OSP 2.0, every OSP runs its own installation and therefore its own Portal IAM. It has three tasks:

1. **Sign-in of people without a password.** People sign in to the OSP's portal and to apps with their EUDI Wallet. For this, the Portal IAM redirects to the EBW-Connector of the installation (C5-M1), the only identity provider. On the first login, the person's account is created automatically.
2. **Tokens for the modules.** The token of a person carries their identifier, their company (if bound), their name from the PID and the permissions of their portal roles. The modules check permission and scope (FRAMEWORK 4.3).
3. **Tokens for programs.** External systems (CSP, issuers, dataspace wallets) call back with service accounts. Technical users of the companies get tokens for central services.

For this, C2-M5 defines the fixed part: realm and basic settings, the identity provider EBW-Connector with its mappers, the sign-in flows, the clients `portal-core`, `portal-ui` and `technical_roles_management` with their roles, the portal roles, client scopes and mappers (content of the tokens) and the service accounts. What is created at runtime is created by C2-M2 and C2-M3 (6.5).

Who uses the configuration:

| Who | What for |
|---|---|
| People through the UI of the OSP | sign-in through the client `portal-ui` (7.2) |
| Modules of the installation through the token validation library | validate tokens: discovery, JWKS, claims (7.6) |
| C2-M2, C2-M3 | create and change runtime objects in the realm through their service accounts (5.9) |
| EBW-Connector of the installation | check the client authentication of the Portal IAM with `private_key_jwt`, using the JWKS of the realm (5.5) |
| CSP, issuers, dataspace wallets | get tokens for callbacks to C3-M2, C4-M1, C4-M2 (5.9, 7.4) |
| Apps | sign-in of their users through their app client (7.3) |
| Programs of the companies (technical users) | tokens for central services (7.4) |
| Teams at the hackathon | test installation with a stand-in for the EBW-Connector, test persons and test keys (14) |

## 2. Terms

| Term | Meaning |
|---|---|
| Portal IAM | Keycloak of the OSP with one realm for its installation (FRAMEWORK 8) |
| Permission | client role on the client `portal-core`, name `<verb>_<object>`; an endpoint of a module requires exactly one (`x-permission`) |
| Portal role | composite client role on the client `portal-core` that bundles permissions: `Applicant`, `Member`, `IT Admin`, `Company Admin`, `OSP Reviewer`, `OSP Admin` |
| Technical user role | client role on the client `technical_roles_management`, composed of roles on the clients of the central services |
| App role | role that a provider defines for its app; lives on the app clients (C2-M2), not part of this module |
| Identity provider (sign-in service) | identity provider in Keycloak; here exactly one, the EBW-Connector, alias `ebw-connector` |
| First login | flow ("First Broker Login") that Keycloak runs when a person arrives through the identity provider who is not yet linked to any account |
| Link | assignment of an account to the alias and the `sub` of the person at the EBW-Connector, in Keycloak "Federated Identity" |
| Client assertion | JWT with which a client authenticates itself at the token endpoint (`private_key_jwt`, RFC 7523) |
| Client scope | bundle of mappers and scope mappings in Keycloak; determines what is in the token of a client |
| Mapper | rule in Keycloak that writes a value as a claim into the token; on the identity provider: rule that copies a claim of the EBW-Connector into the account |
| Scope mapping | list of the roles that may get into the token of a client at all (with `fullScopeAllowed: false`) |
| Service account | client with `serviceAccountsEnabled` with which a program gets tokens through client credentials |
| Runtime object | object in the realm that C2-M2 or C2-M3 creates at runtime; the deployment of C2-M5 never changes it |

## 3. Not part of this module

| What | Where |
|---|---|
| Bind an account to a company, assign portal roles and app roles, invitations, bootstrap of the first OSP Admin | C2-M2 Users and App Access |
| App clients and app roles | C2-M2 |
| Technical users of the companies | C2-M3 Technical Users |
| Check the EUDI Wallet and the Business Wallet | C5-M1 EBW-Connector |
| Validate tokens | shared token validation library (provided) |
| Trust of central services and apps in the Portal IAMs of several OSPs | core services of the CSP, OP-37 |
| Operation of Keycloak: hosting, database, backup, high availability, realm `master` | OSP |
| Keycloak extensions | none (decision of 16 Oct 2026) |

## 4. Technical frame

- **Product:** Keycloak. The target is major version 26. The configuration only uses functions that also exist in version 25.
- **One realm per installation**, name `IAM_REALM`, default `CX-Central` as today. Every OSP has its own address and therefore its own issuer `{IAM_URL}/realms/{IAM_REALM}`; tokens of different OSPs differ in `iss`.
- **Path** with `/auth` (Keycloak option `http-relative-path=/auth`), as today.
- **Configuration only:** no own providers, authenticators, mapper types or themes. All building blocks used are included in Keycloak.
- **Only C2 modules write to the realm** (FRAMEWORK 1): C2-M5 at deployment, C2-M2 and C2-M3 at runtime through the Admin API.
- **No passwords** in the realm. People sign in only through the EBW-Connector.
- **Role descriptions** per language as role attributes `description.de` and `description.en`; the field `description` contains the English text.

### Deployment

- A realm file in the Keycloak export format, `realm-osp.json`, is the only source. It is versioned in the repository of the base skeleton and is the same for every OSP. Its realm attribute `portal.config.version` names the version of the configuration (`0.2.0`).
- What differs per OSP is a placeholder `${NAME}` in the file (table below). The OSP's Helm chart sets the values. On import with `--import-realm`, Keycloak replaces them from environment variables; for the seeding job, the Helm template replaces them.
- **Operation:** seeding job (`Keycloak.Seeding` from the portal releases, as a Helm job) with the realm file. It creates and aligns. It may only delete roles of the clients `portal-core` and `technical_roles_management`. It does not touch runtime objects (6.6).
- **Test and hackathon:** Keycloak starts with `--import-realm` and the test variant of the realm file (section 14).
- There are no manual changes in the admin console. Every change is a new version of the realm file.

### Configuration

| Variable | Meaning | Example |
|---|---|---|
| `IAM_URL` | base address of the Portal IAM | `https://iam.osp-a.example.org/auth` |
| `IAM_REALM` | realm | `CX-Central` |
| `IAM_SSL_REQUIRED` | `sslRequired` of the realm | `all`; test: `external` |
| `IAM_ACCESS_TOKEN_LIFESPAN` | validity of the access tokens in seconds | `300` |
| `IAM_SSO_IDLE_TIMEOUT` | session without activity in seconds | `1800` |
| `IAM_SSO_MAX_LIFESPAN` | longest session in seconds | `36000` |
| `IAM_EVENTS_EXPIRATION` | retention of the login and admin events in seconds | `7776000` (90 days) |
| `EBW_CONNECTOR_URL` | issuer of the EBW-Connector of the installation (`PUBLIC_URL` in C5-M1) | `https://ebw.osp-a.example.org` |
| `EBW_CONNECTOR_CLIENT_ID` | client ID of the Portal IAM at the EBW-Connector | `portal-iam` |
| `PORTAL_UI_URL` | address of the UI of the OSP | `https://portal.osp-a.example.org` |
| `CSP_JWKS_URL` | JWKS address of the CSP for its service accounts (from the trust list) | `https://csp.example.org/.well-known/jwks.json` |
| `CALLBACK_CLIENTS` | which service accounts for callbacks are created (5.9), comma-separated | `sa-csp,sa-credential-issuer,sa-ih-callback` |
| `CALLBACK_JWKS_URLS` | JWKS address per service account that does not belong to the CSP, as `client=url` | `sa-credential-issuer=https://issuer.osp-a.example.org/jwks` |
| `C2_M2_CLIENT_SECRET`, `C2_M3_CLIENT_SECRET` | secrets of the service accounts of the C2 modules, from the secret store | |
| `CALLBACK_CLIENT_SECRETS` | secrets of the service accounts for callbacks without `private_key_jwt` (transition, OP-22), as `client=secret` from the secret store | |

## 5. Content of the configuration

### 5.1 Realm settings

| Setting | Value | Reason |
|---|---|---|
| `realm` | `${IAM_REALM}` | one realm per installation |
| `sslRequired` | `${IAM_SSL_REQUIRED}` | operation over HTTPS only |
| `registrationAllowed` | `false` | no self-registration with a form; accounts are created through the first login (5.6) |
| `resetPasswordAllowed` | `false` | no passwords |
| `rememberMe` | `false` | as today |
| `loginWithEmailAllowed` | `false` | no sign-in with an e-mail address |
| `duplicateEmailsAllowed` | `true` | the PID provides no e-mail address; the field stays empty |
| `editUsernameAllowed` | `false` | the user name follows from the link |
| `verifyEmail` | `false` | no e-mails from the Portal IAM |
| `passwordPolicy` | empty | no passwords |
| `browserFlow` | `browser-ebw` (5.6) | sign-in only through the EBW-Connector |
| `directGrantFlow` | `direct-grant-denied` (5.6) | no sign-in with user name and password at the token endpoint |
| `requiredActions` | all with `enabled: false` | no forms of the Portal IAM, not even "complete profile" (`VERIFY_PROFILE`) |
| `defaultRoles` | only the Keycloak default (`offline_access`, `uma_authorization`), no role on `portal-core` | otherwise service accounts would get the default roles as well; `Applicant` is assigned by the identity provider (5.5) |
| `eventsEnabled` | `true`; types at least `LOGIN`, `LOGIN_ERROR`, `REGISTER`, `IDENTITY_PROVIDER_FIRST_LOGIN`, `IDENTITY_PROVIDER_LOGIN`, `IDENTITY_PROVIDER_LOGIN_ERROR`, `CLIENT_LOGIN`, `CLIENT_LOGIN_ERROR`, `CODE_TO_TOKEN`, `CODE_TO_TOKEN_ERROR`, `LOGOUT` | traceability of sign-ins and token requests |
| `eventsExpiration` | `${IAM_EVENTS_EXPIRATION}` | limited retention |
| `adminEventsEnabled` | `true` | trace changes made by C2-M2 and C2-M3 |
| `adminEventsDetailsEnabled` | `false` | representations may contain secrets |
| Realm attribute `adminEventsExpiration` | `${IAM_EVENTS_EXPIRATION}` | limited retention |
| `internationalizationEnabled` | `true`, languages `de`, `en`, default `en` | error pages in both languages |
| Realm attribute `portal.config.version` | `0.2.0` | version of the configuration |
| Organizations (Keycloak 26) | off | not used |

### 5.2 Token lifetimes and sessions

| Setting | Value | Note |
|---|---|---|
| `accessTokenLifespan` | `${IAM_ACCESS_TOKEN_LIFESPAN}` = 300 s | applies to all tokens |
| `ssoSessionIdleTimeout` | `${IAM_SSO_IDLE_TIMEOUT}` = 1800 s | session in the Portal IAM; during this time, the portal and the apps of the OSP sign in without a new wallet request |
| `ssoSessionMaxLifespan` | `${IAM_SSO_MAX_LIFESPAN}` = 36000 s | after that, the EBW-Connector asks the wallet again |
| `accessCodeLifespan` | 60 s | as today |
| `accessCodeLifespanLogin` | 1800 s | as today |
| Refresh tokens for client credentials | none (`client_credentials.use_refresh_token: false`) | service accounts and technical users get a new token |

The EBW-Connector itself has no session (C5-M1). The session in the Portal IAM is the only single sign-on, and only within one OSP.

### 5.3 Keys of the realm

- **Tokens** are signed by the realm with RS256, key provider `rsa-generated`, 3072 bit, priority 100. This is the algorithm that the library checks (7.6).
- **Client assertion towards the EBW-Connector:** the EBW-Connector only accepts ES256 (C5-M1). For this, the realm also has a key provider `ecdsa-generated` with curve P-256, priority 100. The identity provider signs its client assertion with it (5.5). Both keys are in the JWKS of the realm.
- `hmac-generated` and `aes-generated` stay as Keycloak sets them by default.
- **Rotation:** create a new key with a higher priority. The old one stays passive for at least 24 hours, that is in the JWKS but without new signatures; this is longer than the longest session (10 hours). Then it is disabled. Library and EBW-Connector reload the JWKS when an unknown `kid` appears.

### 5.4 User profile

Declarative user profile of Keycloak. Keycloak does not store undeclared attributes (default for "unmanaged attributes": off). This way, the birth date from the PID never gets into the account.

| Attribute | Required | View / edit | Validation | Source |
|---|---|---|---|---|
| `username` | yes | `admin` / `admin` | Keycloak default | first login: `ebw.<sub at the EBW-Connector>` (5.5) |
| `firstName`, `lastName` | no | `admin` / `admin` | up to 255 characters | `given_name`, `family_name` of the PID, updated on every sign-in (5.5) |
| `email` | no | `admin` / `admin` | e-mail, up to 255 characters | not used; the PID provides none |
| `company_id` | no | `admin` / `admin` | UUID (`pattern`), one value | C2-M2 (binding), C2-M3 (technical users) |
| `bpn` | no | `admin` / `admin` | `^BPNL[0-9A-Z]{12}$`, one value | C2-M2, C2-M3 |

`firstName` and `lastName` are not required, so that Keycloak never shows a form to complete them. Compared with version 0.1, `psp_id` is dropped.

### 5.5 Identity provider EBW-Connector

The only identity provider of the realm. It replaces the identity providers per PSP (previously C2-M1) and is fixed configuration.

| Setting | Value |
|---|---|
| `alias` | `ebw-connector` |
| `providerId` | `oidc` |
| `displayName` | `EUDI-Wallet` |
| `enabled` | `true` |
| `config.issuer` | `${EBW_CONNECTOR_URL}` |
| `config.authorizationUrl`, `tokenUrl`, `jwksUrl` | `${EBW_CONNECTOR_URL}/authorize`, `/token`, `/jwks` (C5-M1 discovery) |
| `config.useJwksUrl`, `validateSignature` | `true`; the EBW-Connector signs the ID token with ES256 |
| `config.disableUserInfo` | `true`; all needed claims are in the ID token |
| `config.clientId` | `${EBW_CONNECTOR_CLIENT_ID}` |
| `config.clientAuthMethod` | `private_key_jwt` |
| `config.clientAssertionSigningAlg` | `ES256` (key from 5.3) |
| `config.defaultScope` | `openid profile` |
| `config.pkceEnabled`, `pkceMethod` | `true`, `S256` (required in C5-M1) |
| `config.syncMode` | `FORCE`: names from the PID on every sign-in |
| `config.trustEmail`, `storeToken`, `addReadTokenRoleOnCreate` | `false` |
| `config.hideOnLoginPage` | `false` |
| `firstBrokerLoginFlowAlias` | `first-login-ebw` (5.6) |
| `postBrokerLoginFlowAlias` | empty |
| `linkOnly` | `false` |

Mappers on the identity provider:

| Mapper | Type | Effect | Sync |
|---|---|---|---|
| `username` | `oidc-username-idp-mapper`, template `ebw.${CLAIM.sub}` | user name from the `sub` of the EBW-Connector | `IMPORT` |
| `first-name` | `oidc-user-attribute-idp-mapper`, claim `given_name` → `firstName` | first name from the PID | `FORCE` |
| `last-name` | `oidc-user-attribute-idp-mapper`, claim `family_name` → `lastName` | last name from the PID | `FORCE` |
| `applicant` | `oidc-hardcoded-role-idp-mapper`, role `portal-core.Applicant` | every new account gets the portal role `Applicant` | `IMPORT` |

No further mappers. In particular, none for `company_id`, `bpn` or any role other than `Applicant`: only C2-M2 sets the binding to a company, after onboarding or invitation, never a claim of the sign-in. `Applicant` comes through the mapper and not as a default role of the realm, so that service accounts do not get it.

**Entry at the EBW-Connector** (client registry as in C5-M1, section 6.8), created by the OSP at installation:

| Field | Value |
|---|---|
| `clientId` | `${EBW_CONNECTOR_CLIENT_ID}` |
| `name` | display name of the portal, for example "Portal OSP A" |
| `redirectUris` | `{IAM_URL}/realms/{IAM_REALM}/broker/ebw-connector/endpoint` |
| `jwksUri` | `{IAM_URL}/realms/{IAM_REALM}/protocol/openid-connect/certs` |
| `scopes` | `openid profile` |

The UI of the OSP is a separate client at the EBW-Connector, for the verification attestation during onboarding (C5-M1 open points no. 15); the Portal IAM never requests `representation`, `company` or `attestation`.

### 5.6 Sign-in flows

Three own flows made of building blocks that are included in Keycloak:

| Flow | Steps | Bound to |
|---|---|---|
| `browser-ebw` | 1. `auth-cookie` (ALTERNATIVE): use an existing session. 2. `identity-provider-redirector` (ALTERNATIVE) with `defaultProvider` = `ebw-connector`: redirects to the EBW-Connector immediately. No form. | browser flow of the realm |
| `first-login-ebw` | subflow with two alternatives: 1. `idp-create-user-if-unique` (ALTERNATIVE): create and link the account. 2. `deny-access-authenticator` (ALTERNATIVE): only applies if the user name already exists. No `idp-review-profile`, no linking through e-mail, no confirmation. | first login of the identity provider |
| `direct-grant-denied` | 1. `deny-access-authenticator` (REQUIRED) | direct grant flow of the realm |

Consequences:

- **First login:** the person arrives with a `sub` of the EBW-Connector that is not yet linked to any account. Keycloak creates an account (user name `ebw.<sub>`, first and last name from the PID, portal role `Applicant`, without company, without password) and links it. No page of the Portal IAM appears (FRAMEWORK 5.2 step 1).
- **Every further sign-in:** Keycloak finds the account through the link and updates first and last name.
- **Without a session**, the Portal IAM always redirects to the EBW-Connector, even without `kc_idp_hint`. A password field never appears.
- **No sign-in with a password at the token endpoint**, even if a client had `directAccessGrantsEnabled`.
- **Identifier at the EBW-Connector:** the `sub` is pairwise per client and depends on the name and birth date of the PID (C5-M1). If the name changes, the person arrives with a new `sub` and gets a new account (open points).

### 5.7 Client scopes and mappers

| Client scope | Content | Who gets it |
|---|---|---|
| `basic`, `roles`, `web-origins`, `acr` | Keycloak default (`sub`, `auth_time`, `realm_access`, `resource_access`, mapper "audience resolve") | realm default for all clients |
| `profile`, `email`, `address`, `phone`, `offline_access`, `microprofile-jwt` | Keycloak default | optional in the realm; none of the clients of this module requests them |
| `portal-person` | mappers `given_name` (property `firstName`), `family_name` (`lastName`), `company_id` (attribute), `bpn` (attribute); in access and ID token, type String | `portal-ui` |
| `portal-core-api` | audience mapper `portal-core-audience` (`included.client.audience` = `portal-core`, access token only); scope mapping: all client roles of `portal-core`, listed individually | `portal-ui` |
| `portal-core-audience` | only the audience mapper as above, no scope mapping | service accounts for callbacks and for HTTP between modules (5.9); their permissions come through the client's own scope mapping |
| `catena` | mappers `company_id` and `bpn` (attributes) in access, ID and userinfo token | app clients (C2-M2), clients of technical users (C2-M3) |
| `technical-roles` | scope mapping: all roles of `technical_roles_management`, listed individually | clients of technical users (C2-M3) |

- Default scopes of the realm: `basic`, `roles`, `web-origins`, `acr`.
- Apps get neither the name nor the e-mail address of a person (FRAMEWORK 4.6): the scope `catena` contains no name, and app clients do not get `profile` and `email` (C2-M2).
- New permissions always also go into the scope mapping of `portal-core-api` (6.6).

### 5.8 Clients

#### `portal-core`

Audience (`aud`) of the tokens for the modules and holder of the permissions and portal roles. Nobody signs in with it.

| Setting | Value |
|---|---|
| `clientId` | `portal-core` |
| `name` | `Portal core` |
| `description` | `Audience of tokens for the portal modules. Holds the permissions and portal roles. Not for sign-in.` |
| `enabled` | `true` |
| `publicClient` | `false` |
| `standardFlowEnabled`, `implicitFlowEnabled`, `directAccessGrantsEnabled`, `serviceAccountsEnabled` | `false` |
| `clientAuthenticatorType` | `client-jwt` without a stored key (no sign-in possible) |
| `fullScopeAllowed` | `false` |
| Client roles | permissions from 6.2 and portal roles from 6.3, each with `description`, `description.de`, `description.en`; permissions also with the attributes `module` and `scope` |

#### `portal-ui`

Client of the UI of the OSP. People sign in through it.

| Setting | Value |
|---|---|
| `clientId` | `portal-ui` |
| `name` | `Portal` |
| `publicClient` | `true` (UI in the browser) |
| `standardFlowEnabled` | `true`; all other flows `false` |
| Attribute `pkce.code.challenge.method` | `S256` |
| `redirectUris` | `${PORTAL_UI_URL}/*` |
| `webOrigins` | `${PORTAL_UI_URL}` |
| Attribute `post.logout.redirect.uris` | `${PORTAL_UI_URL}/*` |
| `fullScopeAllowed` | `false` |
| `defaultClientScopes` | `basic`, `roles`, `web-origins`, `acr`, `portal-person`, `portal-core-api` |
| `optionalClientScopes` | none |

#### `technical_roles_management`

Holder of the technical user roles (6.4). Settings as for `portal-core`, `name` `Technical user roles`. All flows off.

#### Clients of the central services

`Cl3-CX-Semantic`, `Cl7-CX-BPDM`, `Cl16-CX-BPDMGate`, `Cl21-CX-DF`, `Cl22-CX-BPND`, `Cl23-CX-Policy-Hub`, `Cl24-CX-SSI-CredentialIssuer`, `Cl25-CX-BPDM-Orchestrator` are in every realm with their roles, so that technical users can get roles on them (6.4). All flows off, `directAccessGrantsEnabled: false`. The CSP runs the services themselves; whether they trust the tokens of several OSPs is open (OP-37). Service accounts that the central services use among each other do not belong in the realm of an OSP.

#### Keycloak's own clients

`account` and `account-console` are disabled: there is no self-service for accounts. `admin-cli`, `broker`, `realm-management` and `security-admin-console` stay as Keycloak sets them by default.

#### Defaults for runtime clients

C2-M5 does not create these clients. The creating modules set these values; the acceptance of C2-M5 checks them with the test objects from section 14.

- **Technical user** (C2-M3): `clientAuthenticatorType: client-secret` (as today), service account, all other flows off, `fullScopeAllowed: false`, `defaultClientScopes` `basic`, `roles`, `catena`, `technical-roles`. On the service account user, the attributes `company_id` and `bpn` and the roles on `technical_roles_management`.
- **App client** (C2-M2): settings as in C2-M2; `defaultClientScopes` with `catena`, without `profile` and `email`.

### 5.9 Service accounts

Every service account is a separate client: `serviceAccountsEnabled: true`, all other flows off, `fullScopeAllowed: false`, `client_credentials.use_refresh_token: false`.

**Service accounts of C2 for the Admin API** (roles of Keycloak's own client `realm-management`):

| Client | Module | Roles on `realm-management` | What for |
|---|---|---|---|
| `svc-c2-m2` | C2-M2 Users and App Access | `manage-users`, `view-users`, `query-users`, `manage-clients`, `view-clients`, `query-clients` | bind accounts, set the attributes `company_id` and `bpn`, assign portal roles and app roles, end sessions; app clients and app roles |
| `svc-c2-m3` | C2-M3 Technical Users | `manage-clients`, `view-clients`, `query-clients`, `manage-users`, `view-users`, `query-users` | clients of technical users; attributes and roles of their service accounts |

- Sign-in with the secret from `${C2_M2_CLIENT_SECRET}` or `${C2_M3_CLIENT_SECRET}`; scopes `basic`, `roles`.
- No other module has a service account for the Admin API. Nobody except C2-M5 changes the identity provider.
- `manage-users` allows `svc-c2-m2` to assign any role, including permissions. The limit "C2-M2 only assigns whole portal roles" is ensured by the acceptance of C2-M2, not by Keycloak.

**Service accounts for callbacks of external systems** (FRAMEWORK 4.2: one per system, exactly one permission):

| Client | System | Permission | Sign-in | Created if |
|---|---|---|---|---|
| `sa-csp` | core services of the CSP: notifications to C3-M2 | `report_csp_notification` | `private_key_jwt`, `jwks.url` = `${CSP_JWKS_URL}` | always |
| `sa-csp-issuer` | issuer of the CSP: status report to C4-M1 | `report_credential_status` | as `sa-csp` | C4-M1 uses the issuer of the CSP |
| `sa-csp-wallet` | dataspace wallet of the CSP: status report to C4-M1 | `report_credential_status` | as `sa-csp` | C4-M2 uses the dataspace wallet of the CSP |
| `sa-credential-issuer` | own credential issuer of the OSP: status report to C4-M1 | `report_credential_status` | `private_key_jwt` if the system supports it, otherwise secret as a transition (OP-22) | C4-M1 uses its own issuer |
| `sa-ih-callback` | callback adapter in the own IdentityHub: status report to C4-M1 | `report_credential_status` | as `sa-credential-issuer` | C4-M2 uses its own IdentityHub |
| `sa-div-callback` | SAP intermediate layer (SAP DIV): callbacks to C4-M2 | `submit_wallet_callback` | as `sa-credential-issuer` | C4-M2 uses SAP DIV |

- Which ones are created is set in `${CALLBACK_CLIENTS}`. The names are fixed; C4-M1 and C4-M2 check `azp` where they need to distinguish the sender.
- `private_key_jwt`: `clientAuthenticatorType: client-jwt`, attributes `use.jwks.url: true`, `jwks.url` from `${CSP_JWKS_URL}` or `${CALLBACK_JWKS_URLS}`. Secret: `client-secret` from `${CALLBACK_CLIENT_SECRETS}`.
- `defaultClientScopes`: `basic`, `roles`, `portal-core-audience`. The client's own scope mapping: exactly the one permission. On the service account user: exactly this permission. This way, only this one permission is in the token even if another role were assigned to the account by mistake.

**Service accounts for HTTP calls between modules:** only needed if a module runs as a separate service. Then the caller gets a client `svc-<id>` as above, with secret or `private_key_jwt`, scope `portal-core-audience` and the needed permissions `execute_steps_<id>` or `query_<id>` in its own scope mapping. In the modular monolith there are none.

### 5.10 What is dropped compared with version 0.1

| Version 0.1 (PSP architecture) | Version 0.2 (OSP 2.0) |
|---|---|
| one central Portal IAM of the operator, realm `CX-Central` | one Portal IAM per OSP, realm `${IAM_REALM}` |
| PSP access with `private_key_jwt`, claim `psp_id`, attribute `psp_id` | dropped; people call with their own token |
| role bundles `PSP` and `Operator` | replaced by the portal roles (6.3) |
| identity providers per PSP, created by C2-M1 | one fixed identity provider `ebw-connector` (5.5); C2-M1 is dropped |
| first login `first-broker-login-linked-only` (only accounts created in advance) | `first-login-ebw`: the account is created on the first login (5.6) |
| browser flow `browser-psp-only` with `kc_idp_hint` | `browser-ebw` with a fixed identity provider |
| service account `svc-c2-m1` | dropped |
| permissions on `portal-core` only through bundles for PSPs | permissions through portal roles for people and individually for service accounts |
| stand-in realm `TEST-PSP-A`, test PSPs | stand-in realm `TEST-EBW` for the EBW-Connector, test persons (14) |

Compared with today's installation (Portal 2.6.0), the following also applies: the Company IAM, the client `Cl2-CX-Portal` with its roles, the client `Cl1-CX-Registration`, the shared service account `sa-cl1-reg-2`, the service accounts for callbacks to today's portal and the claim `organisation` are dropped. Calls from the portal to central services run through C3-M2 and C4-M1 with the OSP's own keys; their service accounts of today (`sa-cl2-04`, `sa-cl7-cx-5`, `sa-cl2-03`) no longer belong in the realm of the OSP.

## 6. Rules

### 6.1 Permissions: structure

- A permission is a client role on the client `portal-core`.
- Name in English, lower case, `snake_case`, form `<verb>_<object>`. In addition, per module `execute_steps_<id>` and `query_<id>` with the module identifier in lower case with underscore, for example `execute_steps_c3_m2`.
- Attributes of every role: `description.de`, `description.en`, `module` (module identifier, for example `C3-M1`), `scope` (scope `company`, `osp` or `own`, FRAMEWORK 4.3). Field `description` = English text.
- An endpoint of a module requires exactly one permission (`x-permission`). Modules never check a portal role.
- People get permissions only through portal roles (6.3). Service accounts get their permissions individually (5.9) and never a portal role.
- Every module names its permissions in section 6 of its spec. C2-M5 enters them into the realm file, assigns them to portal roles and includes them in the scope mapping of `portal-core-api`.

### 6.2 List of permissions

**Status: aligned with all module specs (30 Oct 2026).** Every permission that an `x-permission` in an `openapi.yaml` requires is listed here.

**Permissions of people:**

| Permission | Module | Scope | `description.de` | `description.en` |
|---|---|---|---|---|
| `register_company` | C1-M2 | `own` | Eigenen Antrag eines Unternehmens anlegen, ändern, einreichen und zurückziehen | Create, change, submit and withdraw an own company application |
| `view_applications` | C1-M2 | `osp` | Alle Anträge und Belege lesen | Read all applications and documents |
| `manage_agreements` | C1-M2 | `osp` | Fassungen der Bedingungen lesen und anlegen | Read and create versions of the terms |
| `view_reviews` | C1-M3 | `osp` | Prüffälle lesen | Read review cases |
| `review_application` | C1-M3 | `osp` | Über Anträge entscheiden | Decide on applications |
| `accept_invitation` | C2-M2 | `own` | Einladung annehmen | Accept an invitation |
| `view_company_users` | C2-M2 | `company` | Personen des eigenen Unternehmens und ihre Rollen lesen | Read the people of the own company and their roles |
| `manage_company_users` | C2-M2 | `company` | Personen einladen, entfernen, Portal-Rollen im Unternehmen ändern | Invite and remove people, change portal roles in the company |
| `view_app_roles` | C2-M2 | `company` | App-Rollen der abonnierten Apps lesen | Read the app roles of subscribed apps |
| `assign_app_roles` | C2-M2 | `company` | App-Rollen an Personen des Unternehmens vergeben | Assign app roles to people of the company |
| `view_users` | C2-M2 | `osp` | Alle Konten der Installation lesen | Read all accounts of the installation |
| `manage_users` | C2-M2 | `osp` | Konten der Installation verwalten | Manage accounts of the installation |
| `manage_osp_staff` | C2-M2 | `osp` | Rollen des Personals des OSP vergeben und entziehen | Assign and remove staff roles of the OSP |
| `view_technical_users` | C2-M3 | `company` | Technische Benutzer und vergebbare Rollen lesen | Read technical users and assignable roles |
| `manage_technical_users` | C2-M3 | `company` | Technische Benutzer anlegen, ändern, Secret erneuern | Create and change technical users, rotate secrets |
| `delete_technical_users` | C2-M3 | `company` | Technische Benutzer löschen | Delete technical users |
| `view_all_technical_users` | C2-M3 | `osp` | Technische Benutzer aller Unternehmen lesen | Read the technical users of all companies |
| `view_company` | C3-M1 | `company` | Eigenes Unternehmen, Unternehmensrollen und Änderungsanträge lesen | Read the own company, its company roles and change requests |
| `edit_company` | C3-M1 | `company` | Unternehmensdaten ändern, Änderungsantrag stellen | Change company data, file a change request |
| `edit_company_roles` | C3-M1 | `company` | Unternehmensrollen mit Zustimmungen hinzufügen und entziehen | Add and remove company roles with consents |
| `view_companies` | C3-M1 | `osp` | Alle Unternehmen der Installation suchen und lesen | Search and read all companies of the installation |
| `manage_companies` | C3-M1 | `osp` | Unternehmensdaten berichtigen, Unternehmen deaktivieren | Correct company data, deactivate companies |
| `review_company_changes` | C3-M1 | `osp` | Änderungsanträge und Belege lesen und entscheiden | Read and decide change requests and documents |
| `view_csp_registration` | C3-M2 | `osp` | Stand eines Unternehmens beim CSP lesen | Read a company's registration at the CSP |
| `view_credentials` | C4-M1 | `company` | Nachweise des eigenen Unternehmens lesen | Read the own company's credentials |
| `manage_credentials` | C4-M1 | `osp` | Nachweise aller Unternehmen lesen, Widerruf von Hand bestätigen | Read all credentials, confirm manual revocation |
| `view_dataspace_wallet` | C4-M2 | `company` | Datenraum-Wallet und Wallet-Konten des Unternehmens lesen | Read the company's dataspace wallet and wallet accounts |
| `retrieve_wallet_credentials` | C4-M2 | `company` | Zugangsdaten eines Wallet-Kontos einmal abholen | Retrieve the credentials of a wallet account once |
| `prove_wallet_control` | C4-M2 | `own` | Kontrolle über eine eigene Datenraum-Wallet nachweisen (später) | Prove control of an own dataspace wallet (later) |
| `view_notifications` | C11-M1 | `own` | Eigene Benachrichtigungen im Portal lesen | Read own notifications in the portal |
| `view_processes` | C12-M1 | `osp` | Abläufe und ihre Schritte lesen | Read processes and their steps |
| `retrigger_process_steps` | C12-M1 | `osp` | Gescheiterte Schritte erneut anstoßen | Retrigger failed process steps |
| `view_marketplace` | C8-M1 | `company` | Marktplatz mit seinen Angeboten lesen | Read the marketplace and its offers |
| `view_connectors`, `manage_connectors` | C6-M1 | `company` | Connectoren des Unternehmens lesen bzw. verwalten | Read or manage the company's connectors |
| `view_subscriptions`, `manage_subscriptions` | C10-M1 | `company` | Abos des Unternehmens lesen bzw. abschließen und beenden | Read or manage the company's subscriptions |
| `view_offers`, `manage_offers` | C8-M1; `manage_offers` also on the provider side of C10-M1 | `company` | Angebote des Unternehmens als Anbieter lesen bzw. verwalten | Read or manage the company's offers as provider |
| `approve_offers` | C8-M1 | `osp` | Angebote prüfen und freigeben | Review and approve offers |

**Permissions of service accounts for callbacks** (in no portal role, only for the client from 5.9):

| Permission | Module | Scope | Granted to | `description.de` | `description.en` |
|---|---|---|---|---|---|
| `report_csp_notification` | C3-M2 | `osp` | `sa-csp` | Mitteilungen des CSP senden | Send notifications of the CSP |
| `report_credential_status` | C4-M1 | `osp` | `sa-csp-issuer`, `sa-csp-wallet`, `sa-credential-issuer`, `sa-ih-callback` | Ergebnis einer Ausstellung melden | Report the result of an issuance |
| `submit_wallet_callback` | C4-M2 | `osp` | `sa-div-callback` | Rückmeldungen der SAP-Zwischenschicht einliefern | Submit callbacks of the SAP middleware |

**Permissions for HTTP calls between modules** (in no portal role, only for service accounts `svc-<id>`, scope `osp`):

| Permissions | `description.de` | `description.en` |
|---|---|---|
| `execute_steps_c1_m2`, `execute_steps_c1_m3`, `execute_steps_c2_m2`, `execute_steps_c2_m3`, `execute_steps_c3_m1`, `execute_steps_c3_m2`, `execute_steps_c4_m1`, `execute_steps_c4_m2`, `execute_steps_c6_m1`, `execute_steps_c10_m1` | Schritte des Moduls `<Nr.>` über HTTP ausführen | Execute the steps of module `<no.>` via HTTP |
| `query_c1_m2`, `query_c1_m3`, `query_c2_m2`, `query_c2_m3`, `query_c3_m1`, `query_c3_m2`, `query_c4_m1`, `query_c4_m2`, `query_c6_m1`, `query_c8_m1`, `query_c10_m1`, `query_c12_m1` | Abfragen des Moduls `<Nr.>` über HTTP aufrufen | Call the queries of module `<no.>` via HTTP |
| `read_wallet_holder_access` | Zugang zur Holder-Wallet für die Ausstellung lesen | Read the holder wallet access for issuance |
| `rotate_subscription_credentials` | Secret eines Abo-Kontos erneuern | Rotate the secret of a subscription account |

`read_wallet_holder_access` (C4-M2) is granted only to the service account of C4-M1, `rotate_subscription_credentials` (C2-M3) only to the service account of C10-M1. C8-M1, C11-M1 and C12-M1 offer no steps, C11-M1 offers no queries.

C5-M1 (EBW-Connector) uses its own tokens and no permissions on the client `portal-core`.

### 6.3 Portal roles

| Portal role | For | Contains | Assigned by |
|---|---|---|---|
| `Applicant` | every account of a person | `register_company`, `accept_invitation`, `view_notifications`; later `prove_wallet_control` | identity provider on the first login (5.5); always stays |
| `Member` | person of a company | `view_company`, `view_app_roles`, `view_marketplace` | C2-M2 |
| `IT Admin` | person of a company | same as `Member`, plus `view_company_users`, `view_technical_users`, `manage_technical_users`, `delete_technical_users`, `view_credentials`, `view_dataspace_wallet`, `retrieve_wallet_credentials`, `view_connectors`, `manage_connectors`, `view_subscriptions` | C2-M2 |
| `Company Admin` | person of a company | same as `IT Admin`, plus `edit_company`, `edit_company_roles`, `manage_company_users`, `assign_app_roles`, `manage_subscriptions`, `view_offers`, `manage_offers` | C2-M2 (`assign-company-admin` during onboarding, otherwise invitation) |
| `OSP Reviewer` | OSP staff | `view_applications`, `view_reviews`, `review_application`, `view_companies`, `review_company_changes`, `view_users`, `view_all_technical_users` | C2-M2 (`manage_osp_staff`) |
| `OSP Admin` | OSP staff | `view_applications`, `manage_agreements`, `view_reviews`, `view_companies`, `manage_companies`, `review_company_changes`, `view_users`, `manage_users`, `manage_osp_staff`, `view_all_technical_users`, `view_csp_registration`, `manage_credentials`, `approve_offers`, `view_processes`, `retrigger_process_steps` | C2-M2 (bootstrap, then `manage_osp_staff`) |

- All are composite client roles on the client `portal-core` with `description.de` and `description.en`:
  - `Applicant`: „Antrag stellen, Einladung annehmen, Benachrichtigungen lesen“ / “File an application, accept an invitation, read notifications”;
  - `Member`: „Person eines Unternehmens“ / “Person of a company”;
  - `IT Admin`: „Technik des Unternehmens verwalten“ / “Manage the company's technical setup”;
  - `Company Admin`: „Unternehmen und seine Personen verwalten“ / “Manage the company and its people”;
  - `OSP Reviewer`: „Anträge und Änderungsanträge prüfen“ / “Review applications and change requests”;
  - `OSP Admin`: „Installation des OSP verwalten“ / “Manage the OSP's installation”.
- `OSP Reviewer` may decide, `OSP Admin` may not (`review_application` is missing): whoever manages the installation does not approve companies. A person can have both roles.
- Roles of the staff do not depend on a binding; the staff also work without a company (C2-M2).
- The scopes restrict the permissions: with `view_company`, a `Company Admin` sees only their own company; `register_company` only applies without a binding (otherwise C1-M2 responds with 403 `account-bound`).
- There are no further portal roles. Permissions of the service accounts (6.2) are in no portal role.

### 6.4 Technical user roles

Roles on the client `technical_roles_management`, as C2-M3 assigns them. Each is composed of roles on the clients of the central services. The basis is today's composition.

| Role | Contains | Compared with today |
|---|---|---|
| BPDM Orchestrator Admin, BPDM Orchestrator Processor Clean, BPDM Orchestrator Processor CleanAndSync, BPDM Orchestrator Processor PoolSync, BPDM Orchestrator Task Creator | roles on `Cl25-CX-BPDM-Orchestrator` | unchanged |
| BPDM Pool Admin, BPDM Pool Consumer, BPDM Pool Sharing Consumer | roles on `Cl7-CX-BPDM` | unchanged |
| BPDM Sharing Admin, BPDM Sharing Input Consumer, BPDM Sharing Input Manager, BPDM Sharing Output Consumer | roles on `Cl16-CX-BPDMGate` | unchanged |
| Dataspace Discovery | `Cl21-CX-DF`: `view_discovery_endpoint`; `Cl22-CX-BPND`: `add_bpn_discovery`, `delete_bpn_discovery`, `view_bpn_discovery` | without `Cl2-CX-Portal`: `view_connectors` |
| Semantic Model Management | `Cl3-CX-Semantic`: `view_semantic_model` | unchanged |
| Identity Wallet Management | nothing | remains as the wallet role that triggers wallet access (C2-M3, C4-M2); C2-M3 does not assign it in the Portal IAM |
| Connector Management, CX Membership Info, Offer Management, Registration External, Registration Internal | – | **dropped**: today they only contain rights on `Cl2-CX-Portal`. Technical users do not call the modules (FRAMEWORK 4.2). |

Every role gets `description.de` and `description.en`.

### 6.5 Who writes what into the realm

| Object | Created and changed by | When |
|---|---|---|
| Realm, basic settings, key providers, user profile, client scopes, flows | C2-M5 | deployment |
| Identity provider `ebw-connector` with mappers | C2-M5 | deployment |
| `portal-core` with permissions and portal roles, `portal-ui` | C2-M5 | deployment |
| `technical_roles_management` with roles, clients of the central services | C2-M5 | deployment |
| Service accounts `svc-c2-m2`, `svc-c2-m3` and the service accounts for callbacks | C2-M5 | deployment |
| Accounts of the people and their link | Keycloak itself (first login, 5.6) | first login |
| First and last name on the account | Keycloak itself (mapper, 5.5) | every sign-in |
| `Applicant` on the account | Keycloak itself (mapper, 5.5) | first login |
| Binding (`company_id`, `bpn`), portal roles except `Applicant`, app roles, ending sessions | C2-M2 | onboarding, invitation, removal, deactivation |
| App clients, app roles | C2-M2 | app approval, subscription |
| Technical users | C2-M3 | at runtime |

### 6.6 Changes and seeding

- Every change is a new version of the realm file with an increased `portal.config.version`.
- Seeder configuration (`Keycloak.Seeding`): create and align for all kinds of objects in the file. Delete only for client roles of the clients `portal-core` and `technical_roles_management`. No deletion of clients, users and links. Clients and users that are not in the file remain untouched. The seeder does not touch role assignments to accounts of people.
- A new permission: create the role, assign it to a portal role or to a service account, include it in the scope mapping of `portal-core-api`, descriptions in both languages, attributes `module` and `scope`.
- Removing a permission: only when no module requires it any more.
- The seeding run is repeatable: a second run with the same file changes nothing.

## 7. Tokens and endpoints

### 7.1 Endpoints

Endpoints of the product in use, Keycloak. [openapi.yaml](openapi.yaml) describes the first three as a reference.

| Purpose | Endpoint |
|---|---|
| OIDC discovery | `GET {IAM_URL}/realms/{IAM_REALM}/.well-known/openid-configuration` |
| Keys (JWKS) | `GET {IAM_URL}/realms/{IAM_REALM}/protocol/openid-connect/certs` |
| Token | `POST {IAM_URL}/realms/{IAM_REALM}/protocol/openid-connect/token` |
| Sign-in (UI, apps; not in the OpenAPI) | `GET {IAM_URL}/realms/{IAM_REALM}/protocol/openid-connect/auth` |
| Sign-out (not in the OpenAPI) | `GET {IAM_URL}/realms/{IAM_REALM}/protocol/openid-connect/logout` |
| Return from the EBW-Connector (only for it) | `GET {IAM_URL}/realms/{IAM_REALM}/broker/ebw-connector/endpoint` |

Issuer: `{IAM_URL}/realms/{IAM_REALM}`.

### 7.2 Sign-in to the portal

1. The UI redirects the browser to the sign-in endpoint: authorization code with PKCE (`S256`), `client_id=portal-ui`, `scope=openid`.
2. Flow `browser-ebw`: with a session in the Portal IAM, it continues directly with step 5. Otherwise the Portal IAM redirects to the EBW-Connector (`scope=openid profile`, with `state`, `nonce` and PKCE).
3. The EBW-Connector shows the QR code; the person presents the PID with their EUDI Wallet (C5-M1, step `person`). The EBW-Connector redirects back to `…/broker/ebw-connector/endpoint`.
4. The Portal IAM exchanges the code at the EBW-Connector, authenticated with a client assertion (ES256, key from 5.3, `aud` = token endpoint of the EBW-Connector). It checks the ID token with the JWKS of the EBW-Connector. First login: create the account (5.6). Otherwise: find the account through the link, update the names.
5. The Portal IAM redirects back to the UI with a code. The UI exchanges it for tokens (7.5) and calls the modules with them.

For the verification attestation during onboarding, the UI calls the EBW-Connector directly, with its own client there (FRAMEWORK 5.2 step 2), not through the Portal IAM.

### 7.3 Sign-in to apps

1. The app redirects the browser to the sign-in endpoint with the `client_id` of its app client (authorization code with PKCE). A `kc_idp_hint` is not needed.
2. With a session in the Portal IAM, for example after signing in to the portal, it continues without the wallet. Otherwise as in 7.2 steps 2 to 4.
3. The app exchanges the code for tokens. The access token contains `sub`, `company_id`, `bpn` and the app roles, no name (7.5).

Apps across the boundaries of an OSP are not in focus (OP-39).

### 7.4 Tokens for programs

- **Service accounts with `private_key_jwt`** (CSP, own issuers and wallets, if they support it): `grant_type=client_credentials` with `client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer` and `client_assertion`. The assertion: `iss` and `sub` = client ID, `aud` = issuer of the realm or token endpoint, `jti` unique, `exp` at most 60 seconds after `iat`, signed with a key from the stored JWKS (ES256, PS256 or RS256).
- **Service accounts and technical users with secret:** `grant_type=client_credentials` with `client_id` and `client_secret` in the form or through HTTP Basic.
- No refresh token. A token is valid for 5 minutes; the program uses it for all calls during this time.
- The CSP calls back to every OSP with its own token from that OSP's Portal IAM; it knows the token endpoint of every OSP from the trust list (FRAMEWORK 5.1).

### 7.5 Content of the tokens

**Person with company** (access token for `portal-ui`, Erika Mustermann as Company Admin, shortened):

```json
{
  "iss": "https://iam.osp-a.example.org/auth/realms/CX-Central",
  "aud": "portal-core",
  "azp": "portal-ui",
  "sub": "5a1c0001-0000-4000-8000-000000000001",
  "typ": "Bearer",
  "iat": 1793005200,
  "exp": 1793005500,
  "jti": "b0f9c1e2-6d1a-4f0e-9a51-0c7f3a2d9e18",
  "sid": "…",
  "given_name": "Erika",
  "family_name": "Mustermann",
  "company_id": "c0000001-0000-4000-8000-000000000001",
  "bpn": "BPNLIUK2REMLK3F4",
  "resource_access": {
    "portal-core": {
      "roles": ["Applicant", "Company Admin", "register_company", "accept_invitation",
                "view_company", "edit_company", "edit_company_roles", "view_company_users", "…"]
    }
  }
}
```

| Claim | Content |
|---|---|
| `aud` | contains `portal-core` (string or list) |
| `azp` | `portal-ui` |
| `sub` | ID of the account in the Portal IAM; stable as long as the link exists |
| `given_name`, `family_name` | from the PID, as of the last sign-in |
| `company_id`, `bpn` | only for a bound account (C2-M2) |
| `resource_access.portal-core.roles` | portal roles and all permissions they contain |
| not included | birth date, e-mail address, `sub` of the EBW-Connector |

**OSP staff without a company** (Otto Normalverbraucher as OSP Admin): as above, without `company_id` and `bpn`, with `Applicant`, `OSP Admin` and the permissions from 6.3.

**Service account for callbacks** (`sa-csp`):

```json
{
  "iss": "https://iam.osp-a.example.org/auth/realms/CX-Central",
  "aud": "portal-core",
  "azp": "sa-csp",
  "sub": "3c9e7b10-2f4d-4a8e-9b61-5d0c8e7f1a22",
  "typ": "Bearer",
  "resource_access": { "portal-core": { "roles": ["report_csp_notification"] } }
}
```

Without names, `company_id` and `bpn`. Exactly one permission.

**Technical user** (client `sa-test-u1`):

```json
{
  "iss": "https://iam.osp-a.example.org/auth/realms/CX-Central",
  "aud": ["Cl21-CX-DF", "Cl22-CX-BPND", "technical_roles_management"],
  "azp": "sa-test-u1",
  "company_id": "c0000001-0000-4000-8000-000000000001",
  "bpn": "BPNLIUK2REMLK3F4",
  "resource_access": {
    "technical_roles_management": { "roles": ["Dataspace Discovery"] },
    "Cl21-CX-DF": { "roles": ["view_discovery_endpoint"] },
    "Cl22-CX-BPND": { "roles": ["add_bpn_discovery", "delete_bpn_discovery", "view_bpn_discovery"] }
  }
}
```

`aud` contains the clients whose roles are in the token (mapper "audience resolve"), never `portal-core`.

**Person at an app** (access token for the app client `app-test`):

```json
{
  "iss": "https://iam.osp-a.example.org/auth/realms/CX-Central",
  "aud": "app-test",
  "azp": "app-test",
  "sub": "5a1c0001-0000-4000-8000-000000000001",
  "typ": "Bearer",
  "company_id": "c0000001-0000-4000-8000-000000000001",
  "bpn": "BPNLIUK2REMLK3F4",
  "resource_access": { "app-test": { "roles": ["Viewer"] } }
}
```

No name, no e-mail address, no permission on `portal-core` (FRAMEWORK 4.6).

### 7.6 Validation by the modules

The shared token validation library (provided) checks:

1. Signature with a key from the JWKS (`jwks_uri` from discovery), algorithm `RS256`; an unknown `kid` triggers one reload.
2. `iss` equal to `{IAM_URL}/realms/{IAM_REALM}` of the own installation. Tokens of the Portal IAM of another OSP fail here.
3. `exp` not exceeded; tolerance at most 30 seconds.
4. `aud` contains `portal-core`.
5. `typ` is `Bearer`.

It provides `sub`, `azp`, `company_id` and `bpn` (if present), `given_name` and `family_name` (if present) and the permissions from `resource_access.portal-core.roles`. Tokens of technical users and apps fail at step 4. The modules do not check whether a token belongs to a person or to a service account; only the permission with its scope counts (FRAMEWORK 4.3).

## 8. Steps for Process Orchestration

Not applicable. The module is a configuration without steps of its own. Runtime objects are created by C2-M2 and C2-M3 in their steps.

## 9. Queries for other modules

Not applicable. Others only use the endpoints from 7.1.

## 10. Events

No business events in the sense of FRAMEWORK 6. Login and admin events of Keycloak (5.1) are an operational log; they stay in Keycloak and do not go through an outbox. No module needs to know that an account has been created: C2-M2 reads accounts in the Portal IAM when needed.

## 11. Calls to external systems

| Who calls | Whom | When | Stand-in in the test installation |
|---|---|---|---|
| Browser through Keycloak | EBW-Connector: `/authorize` | sign-in without a session | realm `TEST-EBW` in the same Keycloak (14.3) or the reference implementation of the EBW-Connector |
| Keycloak | EBW-Connector: `/token` (with client assertion), `/jwks` | return from the EBW-Connector | as above |
| Keycloak | JWKS address of the CSP and of the other systems with `private_key_jwt` | token request of a service account; Keycloak caches the keys and reloads them for an unknown `kid` | file server `iam-test-jwks` with test JWKS (14.1) |

No calls to modules of the installation.

## 12. Errors

**Token endpoint (Keycloak):** errors as in RFC 6749 section 5.2 as JSON `{ "error", "error_description" }`, HTTP 400 or 401. Typical cases:

| `error` | When |
|---|---|
| `invalid_client` | assertion wrongly signed, expired, wrong `aud`, `jti` reused, client unknown, wrong secret |
| `unauthorized_client` | the client may not use this flow, for example client credentials without service account, or a disabled client |
| `invalid_request` | parameters missing or duplicated, PKCE missing for `portal-ui` |
| `unsupported_grant_type`, `invalid_grant` | other flow, for example `password` |

The exact code depends on the Keycloak version. Clients only evaluate: no `access_token`, HTTP 400 or 401.

**Sign-in:** if the EBW-Connector declines or the person cancels, Keycloak shows an error page in the person's language with a link back to the UI. The same applies if the EBW-Connector cannot be reached. If the user name already exists without a link (should not happen), `first-login-ebw` denies access.

**In the modules:** the library reports `urn:<id>:problem:unauthorized` (401) for a missing or invalid token; the module reports `urn:<id>:problem:forbidden` (403) if the permission is missing (FRAMEWORK 4.5).

## 13. Acceptance

The configuration is done when these cases work with the test installation (section 14). Cases 19 and 20 also run with the realm file for operation.

1. **Import:** Keycloak starts with the test variant without errors. The realm attribute `portal.config.version` is `0.2.0`.
2. **Discovery:** `issuer` is `{IAM_URL}/realms/CX-Central`; `token_endpoint_auth_methods_supported` contains `private_key_jwt`; `grant_types_supported` contains `authorization_code` and `client_credentials`; the JWKS contains an RSA key and an EC key (P-256).
3. **First login:** browser to the sign-in endpoint for `portal-ui` without a session: redirect to the stand-in `TEST-EBW` without a page of the Portal IAM; the request contains `scope=openid profile`, `code_challenge_method=S256` and `nonce`. After Nina Neu signs in: back to the UI. The realm has a new account `ebw.<sub>` with `firstName` "Nina", `lastName` "Neu", the role `Applicant`, a link with `ebw-connector`, without `company_id`, without credentials. Events `REGISTER` and `LOGIN`.
4. **Token after the first login:** the access token from case 3 has `aud` `portal-core`, `azp` `portal-ui`, `given_name`, `family_name`, under `resource_access.portal-core.roles` exactly `Applicant`, `register_company`, `accept_invitation` and `view_notifications`; no `company_id`, no `bpn`, no `birthdate`, no e-mail address.
5. **Further sign-in:** Nina Neu signs in again in a new browser session: the same account; the number of accounts stays the same. After the last name is changed to "Neu-Test" in the stand-in and she signs in again: `lastName` on the account and `family_name` in the token are "Neu-Test".
6. **Client assertion:** the token request of the Portal IAM at the stand-in uses `private_key_jwt`; the assertion is signed with ES256, its `kid` is in the JWKS of the realm. A stand-in that only allows a client secret rejects the Portal IAM (sign-in fails with an error page).
7. **No binding through claims:** the stand-in also delivers `company_id`, `bpn` and `email` in the ID token. After the sign-in, the account has none of these attributes.
8. **Bound account:** Erika Mustermann (test object, bound to Beispiel Maschinenbau, `Company Admin`) signs in: the token contains `company_id`, `bpn`, `Company Admin` and every permission of this portal role as in 6.3, plus `Applicant`.
9. **Staff:** Otto Normalverbraucher (`OSP Admin`, without company) and Rolf Prüfer (`OSP Reviewer`): tokens with the permissions as in 6.3, without `company_id`. Otto's token contains no `review_application`.
10. **Roles:** every role on `portal-core` has `description.de`, `description.en` and the English text in `description`; every permission has `module` and `scope`. Every portal role contains exactly the permissions from 6.3. Every role on `portal-core` is in the scope mapping of `portal-core-api`. No permission from the tables for service accounts is contained in a portal role.
11. **Service account with `private_key_jwt`:** `get-token.py sa-csp` (assertion with the CSP's test key): token with `aud` `portal-core`, `azp` `sa-csp`, under `resource_access.portal-core.roles` exactly `["report_csp_notification"]`, without `refresh_token`. The same assertion twice: no token the second time. Assertion with another key: no token. `sa-csp` with a secret: no token.
12. **Exactly one permission:** after `Company Admin` is assigned to the service account user of `sa-csp` (test only), its token still contains only `report_csp_notification`.
13. **Service account with secret:** `get-token.py sa-div-callback`: token with exactly `submit_wallet_callback`.
14. **Technical user:** `sa-test-u1` with secret: token with `company_id` and `bpn` of Beispiel Maschinenbau, `technical_roles_management` with `Dataspace Discovery` and the contained roles; `aud` does not contain `portal-core`.
15. **App sign-in with session:** after case 8, in the same browser, to the sign-in endpoint for `app-test`: no redirect to the stand-in, directly back to the app. The access token has `aud` `app-test`, `company_id`, `bpn`, `resource_access.app-test.roles` = `["Viewer"]`, no `given_name`, no `family_name`, no role on `portal-core`.
16. **No password:** `grant_type=password` at `portal-ui`, `app-test` and `sa-csp`: no token. For every account in the realm, `GET /admin/realms/CX-Central/users/{id}/credentials` returns an empty list. `passwordPolicy` is empty, `registrationAllowed` and `resetPasswordAllowed` are `false`, `account-console` is disabled. The sign-in endpoint never shows a password field of the Portal IAM.
17. **Rights of the C2 service accounts:** `svc-c2-m2` sets `company_id` on an account, assigns `Member` and ends the sessions; `PUT …/identity-provider/instances/ebw-connector` returns 403. `svc-c2-m3` creates a client with service account and sets attributes and roles there; `PUT …/identity-provider/instances/ebw-connector` returns 403. Neither changes the realm settings (`PUT /admin/realms/CX-Central` returns 403).
18. **User profile:** `PUT …/users/{id}` with `company_id` = `abc` returns 400. An attribute `birthdate` or `organisation` is not stored.
19. **Seeding:** two runs with the same file, one after the other: an export of the realm after the second run equals the one after the first (without timestamps and events). The account from case 3, its link and a portal role assigned by `svc-c2-m2` still exist unchanged afterwards; so does a client `sa999` that `svc-c2-m3` has created. A role on `portal-core` that is not in the file is deleted after the run.
20. **Operation without test objects:** the file for operation contains no client ending in `-test`, no realm `TEST-EBW`, no test persons and no secrets other than placeholders.
21. **Foreign issuer:** a token from the realm `TEST-EBW` and a token of a second test installation with a different `IAM_URL` fail the check as in 7.6 step 2.
22. **End to end (hackathon):** with `EBW_CONNECTOR_URL` pointing to the reference implementation of the EBW-Connector and an EUDI Wallet with a test PID, a person signs in to the portal; cases 3 and 4 apply accordingly.

## 14. Test data and test installation

### 14.1 Setup

Docker Compose in the repository of the base skeleton:

| Service | Content |
|---|---|
| `iam` | Keycloak 26 (image `quay.io/keycloak/keycloak`, fixed version), `start-dev --import-realm --http-relative-path=/auth`, port 8080, `KC_HOSTNAME=http://localhost:8080/auth`, `KC_HOSTNAME_BACKCHANNEL_DYNAMIC=true`. Imports `realm-osp.test.json` and `realm-test-ebw.json`. |
| `iam-test-jwks` | simple file server in the Compose network (`http://iam-test-jwks:8000/`) with the public test JWKS |

Files under `iam/`:

| File | Content |
|---|---|
| `realm-osp.json` | realm file for operation (sections 5 and 6) |
| `test/realm-osp.test.json` | the same file plus test objects (14.2); generated by `test/build-test-realm.py` from the operation file and `test/test-objects.json`, so that the operation file stays the only source |
| `test/realm-osp.hackathon.json` | the same, generated from `test/test-objects.hackathon.json`: the start state of a hackathon installation (14.2, FRAMEWORK 7a) |
| `test/realm-test-ebw.json` | realm `TEST-EBW`, stand-in for the EBW-Connector (14.3) |
| `test/keys/*.TEST-ONLY.pem` | private test keys (EC P-256) for `sa-csp`, `sa-csp-issuer`, `sa-csp-wallet` (shared test key of the CSP) and `sa-credential-issuer` |
| `test/jwks/*.json` | the matching public JWKS |
| `test/.env.test` | test secrets of the service accounts with secret and of the technical user, and test passwords of the stand-in |
| `test/get-token.py` | gets a token: for service accounts with a client assertion from the test key or with a secret; for test persons through the browser flow with the sign-in form of the stand-in; prints the access token |

Test keys, test secrets and test passwords are public in the repository and are only suitable for the test installation. The operation file contains none of them.

Values of the test installation: `IAM_URL=http://localhost:8080/auth`, `IAM_REALM=CX-Central`, `EBW_CONNECTOR_URL=http://localhost:8080/auth/realms/TEST-EBW` with the endpoints of the stand-in (the Keycloak paths differ from 5.5 and are set explicitly in the test variant), `PORTAL_UI_URL=http://localhost:3000`, `CALLBACK_CLIENTS=sa-csp,sa-csp-issuer,sa-csp-wallet,sa-credential-issuer,sa-ih-callback,sa-div-callback`.

### 14.2 Test objects in the realm `CX-Central`

IDs aligned with C2-M2 (test persons), C3-M1 (companies) and FRAMEWORK 7a. The values of `company_id` and `bpn` are placeholders with defaults, so that individual module tests can set their own IDs.

| Object | Kind | Values |
|---|---|---|
| Erika Mustermann | account | ID `5a1c0001-0000-4000-8000-000000000001`; linked with `ebw-connector` (`sub` of the stand-in account `erika`); `company_id` `c0000001-0000-4000-8000-000000000001`, `bpn` `BPNLIUK2REMLK3F4` (Beispiel Maschinenbau GmbH); `Applicant`, `Company Admin` |
| Max Mustermann | account | ID `5a1c0002-0000-4000-8000-000000000002`; linked (`max`); bound to Beispiel Maschinenbau GmbH; `Applicant`, `IT Admin` (as in C2-M3) |
| Hans Beispiel | account | ID `5a1c0003-0000-4000-8000-000000000003`; linked (`hans`); without company; only `Applicant` |
| Lieschen Müller | account | ID `5a1c0004-0000-4000-8000-000000000004`; linked (`lieschen`); without company; only `Applicant` |
| Otto Normalverbraucher | account | ID `5a1c0012-0000-4000-8000-000000000012`; linked (`otto`); without company; `Applicant`, `OSP Admin` |
| Rolf Prüfer | account | ID `bbede0f3-fbc4-433b-b7ca-1982e8724912` (R1 in C1-M3); linked (`rolf`); without company; `Applicant`, `OSP Reviewer` |
| Nina Neu | no account | only in the stand-in (`nina`); for the first login (case 3) |
| `sa-csp`, `sa-csp-issuer`, `sa-csp-wallet` | service accounts (5.9) | `private_key_jwt` with `http://iam-test-jwks:8000/csp.json` |
| `sa-credential-issuer` | service account | `private_key_jwt` with `http://iam-test-jwks:8000/credential-issuer.json` |
| `sa-ih-callback`, `sa-div-callback` | service accounts | secret from `.env.test` (transition, OP-22) |
| `svc-c2-m2`, `svc-c2-m3` | service accounts (5.9) | secrets from `.env.test` |
| `svc-test-internal` | service account for HTTP between modules | scope `portal-core-audience`; all `execute_steps_*` and `query_*` from 6.2; secret from `.env.test` |
| `sa-test-u1` | technical user (5.8) | `company_id` and `bpn` of Beispiel Maschinenbau, role `Dataspace Discovery`; secret from `.env.test` |
| `app-test` | app client as in C2-M2 | public, active, redirect `http://localhost:8081/*`, app roles `Viewer`, `Editor` with descriptions, audience mapper, scope `catena`; Erika has `Viewer` |

In the test variant, `portal-ui` also has the redirects `http://localhost:8081/*` and `http://localhost/callback` for `get-token.py`.

**Start state of a hackathon installation** (`test/test-objects.hackathon.json`, FRAMEWORK 7a): the same objects, but Erika and Max without company (no `company_id`, no `bpn`, no `Company Admin`, no `IT Admin`), without `sa-test-u1` and without `app-test`. So Erika can onboard Beispiel Maschinenbau GmbH herself (FRAMEWORK 5.2) and invite Max (5.3). The bindings in the table above serve the tests of this module (cases 8, 14, 15) and the tokens in 14.4; C2-M2 starts from the same unbound state in its tests.

### 14.3 Realm `TEST-EBW`

Stand-in for the EBW-Connector, only in the test installation. It stands for an external system; the rule "no passwords" applies to `CX-Central`, not to this stand-in.

| Object | Values |
|---|---|
| Users `erika`, `max`, `hans`, `lieschen`, `otto`, `rolf`, `nina` | first and last name as in 14.2; fixed IDs (`sub`); test passwords from `.env.test` |
| Client `portal-iam` | `client-jwt` with JWKS address `http://localhost:8080/auth/realms/CX-Central/protocol/openid-connect/certs`, signature algorithm `ES256`; PKCE `S256` required; redirect `http://localhost:8080/auth/realms/CX-Central/broker/ebw-connector/endpoint`; mappers `given_name`, `family_name` into the ID token |
| Client `portal-iam-claims` | like `portal-iam`, plus mappers with fixed values for `company_id`, `bpn` and `email` (only case 7; the test variant can switch the identity provider to it) |

The stand-in signs its ID token with RS256, the reference implementation of the EBW-Connector with ES256; the identity provider checks both through the JWKS.

### 14.4 Values for the tests of other modules

- Token of a person: `get-token.py erika` (Company Admin), `max` (IT Admin), `lieschen` (without company), `otto` (OSP Admin), `rolf` (OSP Reviewer).
- Token without the permissions of a module (cases with 403): `get-token.py lieschen`; she only has `Applicant`.
- Tokens for callbacks: `get-token.py sa-csp`, `sa-csp-issuer`, `sa-credential-issuer`, `sa-div-callback`.
- Token for HTTP calls between modules: `get-token.py svc-test-internal`.
- For modules running directly on the local machine, the library only needs `IAM_URL=http://localhost:8080/auth` and `IAM_REALM=CX-Central`.
- If a module runs in the Compose network, it gets discovery and JWKS through `http://iam:8080/auth`; the issuer in the tokens stays `http://localhost:8080/auth/realms/CX-Central`. For this, the library needs the address and the expected issuer as separate settings.

## 15. Changes from version 0.1

- **OSP 2.0:** one Portal IAM per OSP instead of a central one; PSP access, `psp_id`, role bundles `PSP` and `Operator` are dropped (5.10).
- **Identity provider EBW-Connector** as fixed configuration (5.5); C2-M1 is dropped and merged into this module.
- **Account on the first login** (5.6) instead of only accounts created and linked in advance; names from the PID on every sign-in.
- **Portal roles** `Applicant`, `Member`, `IT Admin`, `Company Admin`, `OSP Reviewer`, `OSP Admin` (6.3); permissions with the attribute `scope`.
- **New client** `portal-ui`, new client scopes `portal-person` and `portal-core-audience`.
- **Service accounts for callbacks** of the CSP, the issuers and the dataspace wallets with exactly one permission (5.9).
- **Second realm key** ES256 for the client assertion towards the EBW-Connector (5.3).
- **Test installation** with stand-in realm `TEST-EBW` and test persons instead of test PSPs.
- **Added on 29 Oct 2026:** permissions of the other part (`view_notifications` in `Applicant`, `view_marketplace` in `Member`, `view_processes` and `retrigger_process_steps` in `OSP Admin`; steps and queries of C6, C8, C10, C12).
- **Aligned on 30 Oct 2026** with C6-M1, C8-M1, C10-M1, C11-M1, C12-M1: module column of 6.2 without "proposal"; `execute_steps_` and `query_` with the full module identifier as in FRAMEWORK 4.2 (`query_c6_m1`). Test variant for the start state of a hackathon installation (14.2).
