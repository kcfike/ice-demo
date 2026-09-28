# ADR-006 — Platform Stack (C# / .NET) (D-001)

| Field | Value |
|---|---|
| Decision ID | D-001 |
| Status | **DECIDED FOR DEVELOPMENT** (development/demo default; production pending ICE IT confirmation) |
| Owner | PM/IT |
| Release | R1 |
| Blocks | Feature 001 (Platform Foundation & CI/CD) |
| Related ADRs | ADR-002, ADR-003, ADR-004, ADR-005 |
| **Supersedes** | **ADR-001** (marked Superseded; Python content retained as history) |
| Date | 2026-09-26 |

> **Why a new ADR and not an edit to ADR-001:** the decision is being
> corrected *before any implementation exists* because the expected client
> technology environment is Microsoft-aligned (ICE Services is a Microsoft
> technology shop). Architecture-decision history is preserved: ADR-001
> remains in the record as the decision that was made and why, and this
> ADR-006 records the corrected decision and its rationale. ADR-001 is
> marked **Superseded** and linked here; it is not erased.

## 1. Context

The platform stack decision determines the language, framework, database,
frontend, and hosting model for the entire Phase 1 application. Every other
decision (identity, document storage, CI/CD) builds on this choice.

ADR-001 selected a **Python 3.12+ / FastAPI** stack. Before Feature 001
implementation begins, the organization's expected client environment is
confirmed to be **Microsoft-aligned**: the ICE Services organization is a
Microsoft technology shop. Continuing on a Python baseline would place the
production stack outside the client's standard tooling, raising adoption,
support, and security-compliance friction.

Because **no application code exists yet**, this is the correct point to
correct the baseline. The architectural design — ports and adapters, the
simulator strategy, the domain model, the config model — is **language-
neutral by design** (architecture.md, PP-016). Changing the stack does **not**
change any port, simulator boundary, or domain concept. Only the
implementation language, frameworks, tooling, and the concrete project layout
change.

The decision here is the **development/demo default**. It is *not* a confirmed
production standard: ICE IT must still confirm each production item listed in
§7. Nothing in this ADR treats a production infrastructure choice as confirmed.

## 2. Decision

**Select a C# / .NET stack.**

| Layer | Choice | Rationale |
|---|---|---|
| Language | C# | Microsoft-aligned shop; strongly typed; single toolchain (SDK) for backend + tooling |
| Runtime / framework | **.NET 10 LTS**, ASP.NET Core Web API | LTS is the production-grade support baseline; ASP.NET Core gives DI, configuration, logging, and health checks out of the box |
| ORM / DB access | **Entity Framework Core** | First-party; works identically against SQLite (dev/demo) and SQL Server / Azure SQL (prod target); provider swap is a connection-string + provider change |
| Database (dev/demo) | **SQLite** | Zero-config; no server; ideal for a self-contained demo and CI |
| Database (prod target) | **Azure SQL / SQL Server** (pending IT) | Microsoft-aligned; confirmed only when IT validates |
| Frontend | React + TypeScript (Vite) | Same as ADR-001; rich SPA for the reconciliation workspace; standard tooling |
| Test framework | **xUnit** (unit, contract, integration) | .NET-standard test framework; deterministic; runs in CI |
| Hosting (dev/demo) | ASP.NET Core dev server + Vite dev server | Two commands; no IIS/Docker required for local dev |
| Hosting (prod target) | Azure App Service / containers (pending IT) | Confirmed only when IT validates |
| Package manager | NuGet (`nuget.config`, locked via `Directory.Packages.props` / `Directory.Build.props`) | First-party; reproducible restore in CI |
| Auth (dev) | `SimIdentityProvider` (simulator) | No Entra tenant needed for development (D-007 / ADR-003) |
| Auth (prod) | Entra ID OIDC via **MSAL for .NET** | Per D-007 / ADR-003; production architecture requirement |
| CI/CD | GitHub Actions (demo repo) | Per ADR-004; gate model substrate-independent (Azure DevOps fallback) |
| Repository layout | Clean-architecture solution under `src/` and `tests/` | See §3 |

> **LTS version note:** .NET 10 is the LTS target for this ADR. The *required*
> .NET LTS version is an **IT-confirmation item** (§7) — the dev/demo baseline
> assumes .NET 10 LTS, and a change to a different LTS is a `global.json` /
> target-framework adjustment, not an architectural change.

## 3. Repository structure (Feature 001 baseline)

The stack correction also resolves the ADR-001-era open issue that the repo's
empty `src/` directory conflicted with a top-level `app/` Python package. On a
.NET baseline the C# projects live **under `src/`** as the natural MSBuild
layout, and the old `src/` vs `app/` conflict no longer applies.

```
ice_demo/
├── ICE.ProjectLifecycle.sln
├── Directory.Build.props          # shared build settings, central package versions
├── Directory.Packages.props       # central package management (NuGet pins)
├── global.json                    # pin .NET SDK (10.x) for dev + CI parity
├── nuget.config
├── .editorconfig                  # C# + TS + generic style
├── .gitignore                     # + bin/ obj/ sim-state/ *.db web/node_modules web/dist
├── src/
│   ├── ICE.ProjectLifecycle.Domain/          # ports (8 interfaces), Result/Result<T> (two families), AdapterError, value objects
│   ├── ICE.ProjectLifecycle.Application/     # configuration POCOs, thin orchestration services
│   ├── ICE.ProjectLifecycle.Infrastructure/  # EF Core + SQLite, migrations, SimIdentityProvider, Simulators facade
│   ├── ICE.ProjectLifecycle.Api/             # ASP.NET Core host: /health, /api/version, DI/config/logging/health wiring, prod-forbids-sim startup validation
│   └── ICE.ProjectLifecycle.Web/             # React + TypeScript + Vite
├── tests/
│   ├── ICE.ProjectLifecycle.UnitTests/       # xUnit: contracts, config branches, sim identity, health
│   ├── ICE.ProjectLifecycle.IntegrationTests/ # xUnit: EF Core + SQLite migration up/down
│   └── ICE.ProjectLifecycle.ContractTests/   # xUnit: adapter-agnostic port contracts (identity in 001)
├── fixtures/
│   └── identity/users.json           # (001) — 007 adds other fixture dirs
├── brand/tokens.json                 # placeholder brand tokens (PP-057)
├── config/
│   ├── appsettings.Development.json  # all sim, committed
│   ├── appsettings.DryRun.json       # template, committed, no secrets
│   └── appsettings.Production.json   # template, committed, no secrets
├── .github/workflows/ci.yml, release.yml
├── README.md
└── docs/, specs/                     # unchanged BMAD + Spec Kit artifacts
```

**Layer mapping (001 scope):**

- **Domain** — the 8 port interfaces, the two independent result families
  (`Result`/`Result<T>`, `Ok`/`Err`/`Ok<T>`/`Err<T>`), the closed
  `AdapterError` enum (with `IsRetryable`), `AdapterHealth`, and port value
  objects.
  **Governed by the PP-016 architecture test:** no vendor/NuGet references.
- **Application** — configuration POCOs and thin services.
- **Infrastructure** — EF Core + SQLite + migrations, `SimIdentityProvider`,
  the `Simulators` facade (`Seed`/`Reset`).
- **Api** — the ASP.NET Core app shell, `/health` + `/api/version`,
  configuration→DI wiring, and the prod-forbids-sim startup validation.
- **Web** — React + Vite + TypeScript shell consuming `brand/tokens.json`.

## 4. Practical Consequences

1. **Feature 001 scaffolds:** .NET solution + 5 `src/` projects + 3 `tests/`
   projects; ASP.NET Core app shell (`/health`, `/api/version`); port
   interfaces + result/error contracts (`Result`/`Result<T>`/`AdapterError`,
   C#); EF Core + SQLite
   baseline migration (no domain tables); `SimIdentityProvider` +
   `Simulators` facade + `fixtures/identity/users.json`; xUnit harness (unit
   + contract + integration); PP-016 architecture test; React + Vite frontend
   + `brand/tokens.json` placeholder; GitHub Actions CI + release workflow.
2. **Windows dev loop:** `dotnet run --project src/ICE.ProjectLifecycle.Api`
   (backend) + `cd src/ICE.ProjectLifecycle.Web && npm install && npm run dev`
   (frontend). No Docker, no IIS, no SQL Server for local dev.
3. **CI loop:** GitHub Actions runs `dotnet restore` → `dotnet build` →
   `dotnet test` (unit + contract + integration + architecture) →
   `npm ci && npm run build` (frontend) → package (backend publish + web dist)
   → upload artifacts.
4. **Production swap:** SQLite → SQL Server / Azure SQL is a connection-string
   + EF Core provider change. The application code does not change.
5. **Ralph's implementation surface** remains highly reliable: C#/.NET + xUnit
   is a well-understood, strongly-typed target; the ports-and-adapters design
   is unchanged from ADR-001.

## 5. Development Default vs Production Approach

| Aspect | Development default | Production approach |
|---|---|---|
| Language/framework | C# / .NET 10 LTS / ASP.NET Core | C# / .NET LTS / ASP.NET Core (unchanged) |
| Database | SQLite (zero-config) | Azure SQL / SQL Server (pending IT confirmation) |
| Identity | `SimIdentityProvider` (fixture users) | Entra ID OIDC via MSAL for .NET |
| Frontend | React + Vite (dev server) | React + Vite (production build) |
| Hosting | ASP.NET Core dev server + Vite dev server | Azure App Service or container host (pending IT) |
| Test runner | xUnit locally + GitHub Actions | xUnit in GitHub Actions (CI gate) |
| Simulators | All 8 ports on simulators | Simulators **forbidden** (startup validation, per architecture §7) |

> **Key distinction:** the development default runs entirely on simulators +
> SQLite. The production approach uses real adapters (Entra ID, T-Sheets,
> Dataverse, etc.) + the IT-confirmed database. The application code is
> identical in both; only the config switch and database connection string /
> EF Core provider change.

## 6. Relationship to Existing Artifacts

- **`architecture.md`:** unchanged in design. The ports, simulators, domain
  model, and config model are language-neutral; only the stack/MSAL references
  that named Python are updated to C#/.NET.
- **`synthetic-data-strategy.md`:** unchanged. The simulator/fixture/
  contract-test model is language-agnostic; C# implementations follow the same
  fixture formats and contract assertions.
- **ADR-002 (document storage), ADR-003 (identity), ADR-004 (CI/CD), ADR-005
  (delivery model):** unchanged. They are decisions *about* ports and process,
  not about the language. ADR-003's "python-msal" reference is re-expressed as
  MSAL for .NET; the decision (Entra OIDC) is unchanged.
- **ADR-001:** **Superseded** by this ADR. Retained in the record as history.

## 7. Client / IT Confirmation Items (all **Pending**)

This is the **development/demo default**. The following **seven** items must
remain *Pending* until ICE IT confirms them.

| Item | Who | Status |
|---|---|---|
| C# / .NET acceptable as the production application standard (vs another mandate) | IT | **Pending confirmation** — the dev/demo baseline is C#/.NET; if ICE mandates a different standard, the architecture (ports, simulators, domain model) is unchanged |
| Required .NET LTS version | IT | **Pending confirmation** — dev/demo assumes .NET 10 LTS; a different LTS is a `global.json`/TFM change, not an architectural change |
| Azure SQL vs SQL Server (or another approved database) as production DB | IT | **Pending confirmation** — dev/demo uses SQLite; swap is connection-string + EF Core provider change |
| Azure App Service vs containers (or another hosting standard) | IT | **Pending confirmation** — dev/demo uses the ASP.NET Core dev server; see ADR-004 |
| GitHub Actions vs Azure DevOps | IT | **Pending confirmation** — dev/demo uses GitHub Actions; the gate model is substrate-independent (ADR-004) |
| Entra tenant + app-registration requirements | IT | **Pending confirmation** — dev/demo uses `SimIdentityProvider`; see ADR-003 |
| Azure Blob vs SharePoint for document storage | IT/OPS | **Pending confirmation** — dev/demo uses `SimDocumentStore`; see ADR-002 |

**Governance rule (standing, not a confirmation item):** no production
infrastructure choice is silently treated as confirmed. Every production
choice in the table above stays *Pending* until ICE IT explicitly confirms
it. This rule governs how the seven items above are handled; it is not
itself a client/IT confirmation.

## 8. Development Default vs Production Approach — Supersession Note

ADR-001's §7 "Development Default vs Production Approach" table is
superseded by §5 of this ADR. ADR-001's §6 confirmation items (Python vs .NET,
PostgreSQL vs SQL Server, `python-msal` vs .NET MSAL) are **retired as Python-
framed questions** and re-expressed as the C#/.NET confirmations in §7 above.
