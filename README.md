# Portal Backend Spec

Module specifications for a modernized Eclipse Tractus-X portal backend, prepared for the hackathon.

> **Status:** work in progress. Feedback and proposals are welcome via issues and pull requests.

## Architecture: OSP 2.0, every OSP runs its portal, the CSP provides trust

![OSP 2.0: every OSP runs its portal, the CSP provides trust](architecture-overview.png)

Decided on 28 Oct 2026. This replaces the earlier picture of Portal Service Providers in front of a central portal core with APIs.

### Why

Today's portal is a monolith: the user interface owns orchestration and business rules, and a third party cannot use a capability without the portal UI. In practice, participants use it mostly once, during onboarding. The new architecture splits the portal into modules with clear interfaces and makes these interfaces the only way in. No single party should run the portal for all participants of a dataspace.

### OSP 2.0 and the CSP

- **Every onboarding service provider (OSP) runs the portal modules itself** and configures its own portal. Its Helm chart decides which service runs where.
- **The core service provider (CSP) provides trust** through central core services: the trust list of admitted OSPs and issuers, the company identifier (BPN) with the home OSP of each company, the directory from BPN to DID (BDRS), discovery, and common rules. The OSPs feed these services through the OSP–CSP interface.
- **The CSP may offer further services**, such as an issuer or a dataspace wallet. An OSP uses them through a facade or runs its own.
- **Hackathon scenario:** one CSP and two OSPs. OSP A issues credentials itself, OSP B uses the CSP's issuer.
- **Later:** several CSPs with federation, and connectivity between dataspaces.

### Identity

- **Two wallets.** The EUDI Wallet of the person, on the phone, holds the PID and the mandate. The Business Wallet of the company, in the browser, holds the LPID and issues mandates to employees.
- **EBW-Connector (C5-M1)** at every OSP and at the Business Wallet. Onboarding asks the EUDI Wallet for PID and mandate, then the Business Wallet for the LPID; the mandate is required. The EBW-Connector and a Business Wallet are provided for the hackathon.
- **Sign-in** at the OSP's portal with the EUDI Wallet. There are no passwords; today's Company IAM with one realm per company is removed.

### Dropped

- the central portal core with APIs for all OSPs;
- PSP Admission (C1-M1): OSPs are admitted through the trust list of the CSP;
- Delegation (C2-M4): an OSP and its portal are one;
- Sign-in Services (C2-M1): the Portal IAM of an OSP has exactly one identity provider, the EBW-Connector.

### New

- CSP Connection (C3-M2): the only module that calls the core services of the CSP.

### Flows and data

- Every step of a flow, such as onboarding, runs through Process Orchestration and is event-driven. Modules call each other directly only for queries.
- Each module has its own database. No module reads or writes the database of another module; it uses that module's interface. Events go out through an outbox per module and never carry secrets.

## Framework and open points

- [FRAMEWORK.md](FRAMEWORK.md): binding rules for all module specs: architecture, conventions for paths, tokens, permissions and scopes, steps and errors, the flows across modules (onboarding, persons, deactivation, reactivation, transfer, technical users, apps), and the catalogues of events and steps.
- [open-points.md](open-points.md): open questions that concern several modules (OP-xx), with their status.

## Modules

All modules are at version 0.2 for OSP 2.0, as of 28 Oct 2026. Every decision that is not yet agreed is marked as provisional in the module's `open-points.md`.

| Module | Kind | Status |
|---|---|---|
| [C1-M1 PSP Admission](C1-M1-PSPAdmission/) | – | dropped with OSP 2.0: OSPs are admitted through the trust list of the CSP |
| [C1-M2 Registration](C1-M2-Registration/) | Hackathon | draft 0.2 |
| [C1-M3 Application Review](C1-M3-ApplicationReview/) | Hackathon | draft 0.2 |
| [C2-M1 Sign-in Services](C2-M1-SignInService/) | – | dropped with OSP 2.0: the only identity provider (EBW-Connector) is configuration in C2-M5 |
| [C2-M2 Users and App Access](C2-M2-AppAccess/) | Hackathon | draft 0.2 |
| [C2-M3 Technical Users](C2-M3-TechnicalUser/) | Hackathon | draft 0.2 |
| [C2-M4 Delegation](C2-M4-Delegation/) | – | dropped with OSP 2.0 |
| [C2-M5 Portal IAM](C2-M5-PortalIAM/) | Configuration | draft 0.2 |
| [C3-M1 Company Profile](C3-M1-CompanyProfile/) | Hackathon | draft 0.2 |
| [C3-M2 CSP Connection](C3-M2-CSPConnection/) | Hackathon | draft 0.1 (new with OSP 2.0) |
| [C4-M1 Credentials](C4-M1-Credentials/) | Hackathon | draft 0.2 |
| [C4-M2 Dataspace Wallet Interface](C4-M2-DataspaceWalletInterface/) | Hackathon | draft 0.2 |
| [C5-M1 EBW-Connector](C5-M1-EBWConnector/) | Provided | draft 0.2 (two wallets, OSP 2.0; lenient for the hackathon, OP-43) |
| [C6-M1 Connectors](C6-M1-Connector/) | Hackathon | draft 0.2 (other part of the spec) |
| [C8-M1 Offers](C8-M1-Offers/) | Hackathon | draft 0.2 (other part of the spec) |
| [C10-M1 Subscriptions](C10-M1-Subscriptions/) | Hackathon | draft 0.2 (other part of the spec) |
| [C11-M1 Notifications](C11-M1-Notifications/) | Hackathon | draft 0.2 (other part of the spec; inbox in the portal only) |
| [C12-M1 Process Orchestration](C12-M1-ProcessOrchestration/) | Provided | draft 0.2 (other part of the spec; the flows of FRAMEWORK 5 in `flows.yaml`) |

C6 to C12 are the other part of the spec. The framework refers to them by name (Connectors, Offers, Subscriptions, Notifications, Process Orchestration). The MCP server is not specified yet.

The module numbers changed with the Portal Service Provider architecture. The former C2-M6 App Clients and Roles is now part of C2-M2. The versions written for the Portal Service Provider architecture are in the git history of this repository.

## Format

Each module has its own folder, named after its identifier, with three files:

- `README.md` – the behaviour: purpose, data, rules, every operation step by step, steps for Process Orchestration, queries, events, calls to other modules, errors, acceptance criteria and test data
- `openapi.yaml` – the interface (OpenAPI 3.1)
- `open-points.md` – provisional decisions, deviations from the framework, assumptions about other modules and open questions

A team should be able to implement a module from these files and [FRAMEWORK.md](FRAMEWORK.md). A dropped module keeps its folder with a short README.

Modules are grouped into capabilities: `C2-M3` is module 3 of capability C2.

## License

Licensed under the [Apache License 2.0](LICENSE).
