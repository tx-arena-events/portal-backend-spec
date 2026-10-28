# Cross-module open points

**As of:** 2026-10-28, after the switch to OSP 2.0

This file lists the open questions that concern several modules or need a fundamental decision. Questions that concern only one module stay in that module's `open-points.md`. There, cross-module questions are referred to only by their number (OP-xx).

Once a question is decided, it applies to all modules through [FRAMEWORK.md](FRAMEWORK.md). Here it gets the status "decided" with a date, so that its number is kept. Questions that became moot with OSP 2.0 keep their number with the status "dropped". The version for the PSP architecture is in the git history of this repository.

**Status:** open · proposal (a proposal exists, confirmation is missing) · partly · decided (date) · dropped (date)

## Clarify first

Important for the hackathon on 6 Nov 2026: **OP-36**, **OP-37**, **OP-40**, **OP-41**, **OP-42**, **OP-45**, **OP-18**, **OP-22**.

## J. OSP 2.0 (new on 28 Oct 2026)

| No. | Question | Modules | Decided by | Status |
|---|---|---|---|---|
| OP-36 | **Installation settings:** a common file with `ospId`, `ospName`, `ospCompanyId`, dataspace and reference data that every module reads at start-up (FRAMEWORK 4.8). | all | project owner, other part | proposal |
| OP-37 | **Tokens of several OSPs at apps and central services:** Every OSP has its own Portal IAM. Proposal: the issuer of the Portal IAM of every admitted OSP is in the trust list with its JWKS; apps and central services accept tokens of all admitted OSPs. Alternative: credentials (DCP) instead of tokens. | C2-M2, C2-M3, C2-M5, C3-M2, central services | project owner, CSP | proposal |
| OP-38 | **Transfer of a company to another OSP:** application kind `TRANSFER`, the CSP changes the home OSP, the old OSP deactivates the company; data, technical users and dataspace wallet do not move along (FRAMEWORK 5.6). Open: handover of the data, credentials of the old issuer. | C1-M2, C1-M3, C3-M1, C3-M2, C4-M1, C4-M2 | project owner, CSP | proposal |
| OP-39 | **Apps and marketplace across the boundaries of an OSP:** Where are apps offered and subscribed to when every OSP has its own installation? Not in focus of the first version. | C2-M2, Offers, Subscriptions | project owner, other part | open |
| OP-40 | **Accounts of people:** The account is created at the first login through the EBW-Connector, without a company, with the portal role `Applicant` (mapper of the identity provider, C2-M5 5.5). It is bound during onboarding (the applicant becomes `Company Admin`) or through an invitation (FRAMEWORK 5.2, 5.3). Names come from the PID at every sign-in. Open: cleanup of accounts that were never used, deletion on request, new account after a change of name (the `sub` of the EBW-Connector depends on name and birth date), name check with several given names. | C1-M2, C2-M2, C2-M5, C5-M1 | project owner | proposal |
| OP-41 | **Roles of OSP staff:** portal roles `OSP Reviewer` (review and decide applications and change requests) and `OSP Admin` (manage companies, staff, terms, status at the CSP, credentials; does not decide on applications), split defined in C2-M5 6.3. Staff work without a company or in the company of the OSP (`ospCompanyId`); the bootstrap of C2-M2 creates the first OSP Admin. Nobody decides on their own application (C1-M3). Open: whether staff may submit applications themselves. | C1-M2, C1-M3, C2-M2, C2-M5, C3-M1, C3-M2, C4-M1 | project owner | proposal |
| OP-42 | **Interfaces to the CSP:** core services API (proposal in C3-M2), issuer of the CSP (proposal in C4-M1) and dataspace wallet of the CSP (proposal in C4-M2). Who defines them bindingly (association, CSP, community)? Also: how do connectors learn which issuers to trust when OSPs issue credentials themselves? Proposal: through the trust list of the CSP. | C3-M2, C4-M1, C4-M2, Connectors | association, CSP | open |
| OP-43 | **Presentations as strings:** For the hackathon, C5-M1 is lenient: `vp_token` per credential also as a string, several presentations per credential (the first valid one counts), `"apu": null` in the JWE header (C5-M1 6.3, 7.8). Later perhaps strict again according to OpenID4VP 1.0. | C5-M1 | project owner | decided 28 Oct 2026 (for the hackathon) |
| OP-44 | **Several CSPs:** federation of trust lists and company identifier, connection between dataspaces. | C3-M2, C5-M1 | association | later |
| OP-45 | **Accounts on deactivation and reactivation:** The accounts stay active but, for the duration of the deactivation, lose the company, the portal roles in the company and the app roles; C2-M2 remembers them and restores them on reactivation. This way a person with a mandate can apply for the reactivation themselves; after the approval they become `Company Admin` (FRAMEWORK 5.4, 5.5; C2-M2 8.9, 8.10). Replaces "accounts deactivated" in OP-07. | C1-M2, C2-M2, Process Orchestration | project owner | proposal |
| OP-46 | **One catalogue of company roles:** C1-M2 (selection in the application, terms) and C3-M1 (roles and consents on the company) both keep a role catalogue. Proposal: a common file in the installation settings (OP-36) that both read; the versions of the terms stay in C1-M2. | C1-M2, C3-M1 | project owner | proposal |

## A. Portal Service Provider (PSP)

With OSP 2.0, PSPs, PSP admission and delegation are dropped. The points keep their numbers.

| No. | Question | Status |
|---|---|---|
| OP-01 | Today's onboarding service providers as PSPs | dropped 28 Oct 2026; the transition from today's portal stays open (OP-32) |
| OP-02 | Criteria for the admission of a PSP | dropped 28 Oct 2026; the governance rules the admission of an OSP through the trust list (FRAMEWORK 5.1) |
| OP-03 | Application without an operator PSP | dropped 28 Oct 2026 |
| OP-04 | Final end of a PSP | dropped 28 Oct 2026; the end of an OSP (change and data handover of its companies) is open, see OP-38 |
| OP-05 | Further consequences of the suspension of a PSP | dropped 28 Oct 2026 |
| OP-06 | Companies without a PSP | dropped 28 Oct 2026; every company has exactly one home OSP |

## A2. Business Wallet as an option

| No. | Question | Modules | Decided by | Status |
|---|---|---|---|---|
| OP-35 | **Onboarding without Business Wallet:** Both ways run in parallel; the OSP can switch off the way without Business Wallet. Without Business Wallet, the applicant uploads supporting documents and assures the representation; OSP staff review manually in C1-M3. Also applies to reactivation and transfer. | C1-M2, C1-M3, C3-M1 | project owner | decided 26 Oct 2026, carried over to OSP 2.0 on 28 Oct 2026 |

## B. Companies

| No. | Question | Modules | Decided by | Status |
|---|---|---|---|---|
| OP-07 | **Deactivation of a company:** technical users are deactivated, not deleted; accounts of people lose the company and roles (OP-45); the dataspace wallet stays, without new accounts; only the membership credential is revoked; the membership is deregistered at the CSP (FRAMEWORK 5.4). | C2-M2, C2-M3, C3-M1, C3-M2, C4-M1, C4-M2 | project owner | decided 26 Oct 2026, carried over to OSP 2.0 on 28 Oct 2026; accounts changed on 28 Oct 2026 (OP-45) |
| OP-08 | **Reactivation:** application of kind `REACTIVATION` at the OSP with a verification attestation or supporting documents, approval by OSP staff (FRAMEWORK 5.5). | C1-M2, C1-M3, C2-M2, C2-M3, C3-M1, C3-M2, C4-M1 | project owner | decided 26 Oct 2026, carried over to OSP 2.0 on 28 Oct 2026 |
| OP-09 | **Authoritative master data:** The source through which the company was verified leads: with Business Wallet the LPID, without it the supporting documents. The data goes through C3-M2 to the CSP, which assigns the BPN; the master data service sits behind the CSP. | C3-M1, C3-M2, C1-M2 | project owner, CSP | decided 26 Oct 2026, carried over to OSP 2.0 on 28 Oct 2026 |
| OP-10 | **Change of core data:** with a new verification attestation (same EBWOID, at most 24 hours old) immediately; without Business Wallet with a change request, a supporting document and approval by OSP staff (C3-M1). Changed core data goes to the CSP through C3-M2 `update-company` (FRAMEWORK 5.6a). | C3-M1, C3-M2, C5-M1 | project owner | proposal |
| OP-11 | **Terms:** versions of the terms of the OSP and of the dataspace, re-consent with a deadline; the person gives consent themselves. | C1-M2, C3-M1 | OSP | decided 26 Oct 2026, carried over to OSP 2.0 on 28 Oct 2026 |
| OP-12 | **Formats and schemas** of LPID, EBWOID and mandate: only C5-M1 knows them; EUID and EBWOID as text. | C5-M1, C1-M2, C3-M1 | external, project owner | decided 26 Oct 2026 (handling) |
| OP-13 | **Reference data:** a common file (countries, identifier types), part of the installation settings (OP-36). | C1-M2, C3-M1 | project owner | decided 26 Oct 2026 |
| OP-14 | **One dataspace per installation:** `memberOf` and name in the installation settings. | C4-M1, C3-M1 | project owner | decided 26 Oct 2026 |

## C. Onboarding and review

| No. | Question | Modules | Decided by | Status |
|---|---|---|---|---|
| OP-15 | **Withdraw an application:** by the applicant until the decision; drafts do not expire. | C1-M2, C1-M3, C3-M1 | project owner | decided 26 Oct 2026, carried over to OSP 2.0 on 28 Oct 2026 |
| OP-16 | **Four-eyes principle:** configurable cases in C1-M3; the `sub` in the token is compared. PSP admission is dropped. | C1-M3 | OSP | decided 26 Oct 2026, carried over to OSP 2.0 on 28 Oct 2026 |
| OP-17 | **Verification attestation bound to the PSP:** dropped 28 Oct 2026. The verification attestation comes from the EBW-Connector of the own installation; the person gives the consents themselves on submission. | C1-M2, C5-M1 | – | dropped 28 Oct 2026 |
| OP-18 | **Retry after errors:** steps report `retryable`; Process Orchestration retries with growing wait times, then notifies OSP staff. | all modules with steps | other part | proposal |
| OP-19 | **Notifications:** notice to OSP staff about new review cases, invitations to people, notice to providers about new app roles. | C1-M3, C2-M2 | other part, project owner | open |
| OP-20 | **Retention and deletion periods:** verification attestations, supporting documents, audit logs, audit log of the EBW-Connector. | C1-M2, C2-M2, C2-M3, C5-M1, others | OSP, data protection | open |

## D. Delegation and change of PSP

| No. | Question | Status |
|---|---|---|
| OP-21 | Consequences of a change of PSP | dropped 28 Oct 2026; replaced by the transfer to another OSP (OP-38) |

## E. Sign-in and Portal IAM

| No. | Question | Modules | Decided by | Status |
|---|---|---|---|---|
| OP-22 | **Callbacks from external systems:** one service account per system (issuer, dataspace wallet, CSP) in the Portal IAM of the OSP with exactly one permission; `private_key_jwt` where possible. | C2-M5, C3-M2, C4-M1, C4-M2 | OSP, operators of the systems | decided 26 Oct 2026, CSP added on 28 Oct 2026 |
| OP-23 | **UIs of central services** (Semantic Hub, Policy Hub, issuer): treat them like apps, or stop offering them? With OSP 2.0 also: sign-in of people of several OSPs (OP-37). | C2-M5, C2-M2 | project owner, CSP | open |
| OP-24 | **Technical users that call the portal today** (member list, search for connectors): with OSP 2.0, membership and discovery are at the CSP. | C2-M3, C2-M5, C3-M2, Connectors | project owner, CSP | open |
| OP-25 | **Identity provider of a person for the app** (`kc_idp_hint`): the Portal IAM of an OSP has exactly one identity provider. | C2-M2, C2-M5 | – | dropped 28 Oct 2026 |
| OP-26 | **Narrower permissions of the service accounts in Keycloak:** check Fine-Grained Admin Permissions. | C2-M2, C2-M3, C2-M5 | this part | open |
| OP-27 | **Securing tokens:** bound tokens (DPoP, mTLS), rate limiting at the token endpoint. The PSP access is dropped; the topic stays open for service accounts and technical users. | C2-M3, C2-M5 | OSP | open |
| OP-28 | **Keycloak Organizations** as an alternative to attributes on the account. | C2-M2, C2-M5 | this part | later |

## F. Dataspace

| No. | Question | Modules | Decided by | Status |
|---|---|---|---|---|
| OP-29 | **Own dataspace wallet (`BRING_YOUR_OWN`):** later; the first version has hosted wallets and the dataspace wallet of the CSP. | C4-M1, C4-M2 | project owner | decided 26 Oct 2026 |
| OP-30 | **Operation of the dataspace wallets:** one per OSP, the one of the CSP, or both; exchange between connectors with different wallet providers is not shown in any test. | C4-M1, C4-M2 | OSP, CSP, project owner | open |

## G. EBW-Connector

| No. | Question | Modules | Decided by | Status |
|---|---|---|---|---|
| OP-31 | **Kind and operation of the EBW-Connector:** provided (for the hackathon); every operator runs its own instance (OSP and Business Wallet); trust through the trust list of the core services. Registration as a relying party in production. | C5-M1, C1-M2 | project owner | decided 28 Oct 2026 |

## H. Transition from today's installation

| No. | Question | Modules | Decided by | Status |
|---|---|---|---|---|
| OP-32 | **Transition:** With OSP 2.0, today's central portal becomes one installation per OSP and a CSP. Open: who runs the first installation, how do today's companies get a home OSP, what happens to existing apps, technical users, accounts and the Company IAM? | all | project owner, CSP | open |

## I. Module split and alignment with the other part

| No. | Question | Modules | Decided by | Status |
|---|---|---|---|---|
| OP-33 | **Numbers for external systems** | all | project owner | open |
| OP-34 | **Questions to the other part of 21 Oct 2026** (R1 to R7); new in addition: Process Orchestration per installation, offers and subscriptions under OSP 2.0 (OP-39). | all | other part | asked 21 Oct 2026 |

## Follow-up work

The list of follow-up work from the PSP architecture is done or moot with the switch to OSP 2.0. Cross-check of all modules against the framework on 28 Oct 2026: permissions with scope, steps, events and queries match; the additions of the modules are in the framework (sections 4.1, 4.3, 5, 6, 7).

| No. | What | Where |
|---|---|---|
| N1 | English version of the German specs (version 0.2) for the public repository; C5-M1 is already at version 0.2 there, only the separate clients of the OSP (C5-M1 open points No. 15) still need to follow | all modules, framework |
| N2 | Shared CSP stand-in for several installations in the test tools (FRAMEWORK 4.7) | hackathon preparation |
| N3 | Client `portal-iam` as an example in the client directory of the reference implementation of the EBW-Connector (C2-M5 5.5) | reference implementation of C5-M1 |
| N4 | Test installation of the Portal IAM with the stand-in realm `TEST-EBW` and `get-token.py` (C2-M5 14) | base skeleton |
