# Feature 001 — Platform Foundation & CI/CD — Specification

| Field | Value |
|---|---|
| Feature | 001 Platform Foundation & CI/CD |
| Release | R1 |
| Sequence position | First (unblocks 002 and 007) |
| Governing decisions | D-001 (ADR-006), D-029 (ADR-004); ADR-003 §5.2 (identity scaffold), ADR-005 (delivery model) |
| Primary PP requirements | PP-056, PP-057 (tokens), PP-016 (port conventions), PP-045 (baseline) |
| Status | **Specified — pending human review before Ralph begins** |
| Spec Kit directory | `specs/feature-001-platform-foundation-cicd/` |

---

## 1. Purpose

Feature 001 builds the **platform substrate** that every later feature stands
on, per feature-sequence §3 ("Scaffolds app shell, DB, ports, sim harness,
CI/CD. Everything builds on this"):

1. A runnable **ASP.NET Core Web API app shell** with a two-command local dev
   loop (ADR-006 §4.2).
2. The **shared contracts** (the two independent result families
   `Result` / `Result<T>` with `Ok`/`Err`/`Ok<T>`/`Err<T>`, and the closed
   `AdapterError` enum) and all **eight port interfaces** in code (PP-016
   port conventions).
3. A **config model + prod-forbids-sim** startup validation (architecture §7).
4. An **Entity Framework Core + SQLite** foundation with a baseline migration
   (ADR-006 §2), *foundation only* — no domain tables (those arrive with 002+).
5. The **simulator harness skeleton** plus one reference simulator
   (`SimIdentityProvider`) and its fixtures and contract tests — the working
   model that Feature 007 will extend to all 8 ports (see §4 Scope boundary).
6. A **GitHub Actions CI pipeline** with the full gate: build, test (unit +
   contract + integration + architecture), frontend build (ADR-004 §4,
   feature-sequence §5 finding 7: "asserted in 001's CI gate").
7. A **release workflow** (tag → DRY-RUN → PROD approval) and a
   branch-protection runbook (ADR-004 §4, PP-056 AC-2).
8. A **React + Vite + TypeScript** frontend shell with the
   `brand/tokens.json` placeholder (PP-057 AC-1/AC-2 path, ADR-006 §2).
9. Developer documentation: dev loop, config reference, environment model,
   and a traceability pointer back to this spec.

## 2. Authoritative inputs

- `docs/product/feature-sequence.md` — §3 row 001, §5 findings 6–7, §7 handoff.
- `docs/product/requirements-traceability.md` — §3 row 001: *PP-056, PP-057 (tokens), PP-045 (baseline), PP-016 (port conventions)*.
- `docs/product/phase-1-prd.md` — PP-016 AC-1–3 (L284–296), PP-045 AC-1–3 (L833–843), PP-056 AC-1–3 (L921–932), PP-057 AC-1–2 (L933–942).
- `docs/architecture/architecture.md` — §3 layering, §5 ports, §6 simulator strategy, §7 config, §11 CI/CD, §12 testing.
- `docs/architecture/decisions/ADR-006-platform-stack.md` — D-001 decided; §2 stack table; §3 repo structure; §4 practical consequences; §7 pending IT confirmations.
- `docs/architecture/decisions/ADR-003-identity-and-access.md` — D-007 decided; §5.2 001 scaffolds port + `SimIdentityProvider` + config switch.
- `docs/architecture/decisions/ADR-004-ci-cd-and-release-controls.md` — D-029 decided; §2.3 CI gate; §4 release model; §6 pending IT confirmations.
- `docs/architecture/decisions/ADR-005-delivery-model.md` — D-031: Ralph primary implementer, relative timeline; §5 pending confirmations.
- `docs/data/synthetic-data-strategy.md` — §1 principles, §2 layout, §4.1 identity fixtures, §5 contract tests + shared error contract, §7 switching rules.

## 3. Requirements satisfied / enabled

| PP | What 001 delivers | ACs owned by 001 | ACs owned by later features |
|---|---|---|---|
| PP-056 CI/CD | Pipelines, branch-protection runbook, tag/approval release path | AC-1 (build+test+deploy pipeline definitions), AC-2 (source-control controls in-repo + runbook), AC-3 (all automated tests run in CI) | *Actual* DEV/PROD deploy execution (IT-provisioned targets, D-033) |
| PP-057 Theme & Branding | `brand/tokens.json` placeholder consumed by the branded shell | AC-1 (tokens applied application-wide — placeholder values) | AC-2 (real brand assets) — pending DISC-016; drop-in only |
| PP-016 Adapter Abstraction | Port interfaces in code, shared result/error contract (`Result`/`Result<T>`, `AdapterError`), architecture test, sim harness model | AC-1 (ports exist; one sim adapter implemented: identity), AC-2 (architecture test in CI), AC-3 (sim replaceable via config switch — demonstrated for identity) | Full 8-port sim/contract coverage — **Feature 007** |
| PP-045 Performance & Availability | Baseline only: platform runs, CI measures build/test duration; no targets set | *None of AC-1/2/3 are met by 001* — all pending D-026/DISC-015 | 012 (assembly perf), R1 exit (availability) |

> **Honesty rule:** 001 *enables* PP-045 (baseline infra) but does **not**
> satisfy any PP-045 acceptance criterion. No performance numbers are asserted
> or hard-coded (D-026 open; architecture §10's ≤30 s p95 is explicitly an
> *interim target, revise at DISC-015*).

## 4. Scope boundary — simulator split (001 vs 007)

ADR-006 §4.1 lists "simulator package, `fixtures/`" under 001's scaffolding,
while feature-sequence §5 finding 6 states 007 is first-class and
"must not be collapsed into 001". **Resolution adopted in this spec** (flagged
for human sign-off — see §13, Issue 1):

- **001 owns:** all 8 port interfaces (code, per-method typed results), the
  result/error contracts (`Result`/`Result<T>`, `Ok`/`Err`/`Ok<T>`/`Err<T>`,
  `AdapterError`, `AdapterHealth`), the `Simulators` facade (`Seed`/`Reset`),
  **`SimIdentityProvider` only**, `fixtures/identity/users.json` (incl.
  negative cases), the identity contract-test file, and the architecture test
  in CI.
- **007 owns:** the other 7 sim adapters, their fixtures
  (`timekeeping/`, `procurement/`, `employees/`, `documents/`, `ar-handoff/`),
  the full adapter-agnostic contract suite, and the remaining port contract
  tests.

Rationale: ADR-003 §5.2 explicitly assigns the identity sim to 001, and the
identity vertical slice (001→002) needs a working sim + contract test to
prove the PP-016 model end-to-end. The other ports have no 001 consumer.

## 5. Explicit non-goals

The following are **out of scope** for 001 and belong to the listed features:

| Non-goal | Owner |
|---|---|
| Login UI, session management, RBAC enforcement, role matrix (D-009), MFA flow UI, user admin UI, security-event logging (PP-070) | Feature 002 |
| Sim adapters for Timekeeping/Procurement/Employee/Document/AR/Notification; their fixtures; full contract suite | Feature 007 |
| Real (non-sim) adapters for any port (Entra OIDC, T-Sheets, on-prem SQL/Access, …) | Features 007–010 (gated D-033) |
| Any domain tables (SOW, Budget, roles table, audit table, …) | Feature 002+ (audit table with 002/006) |
| SOW form, state machine, packet assembly, billing, closeout, AR delivery, migration, reporting | Features 004–016 |
| Master data (customers, departments, project types, …) | Feature 003 |
| Performance targets, load testing, SLA instrumentation | pending D-026/DISC-015 (012) |
| Docker images, IIS, Kubernetes, Helm | none in R1 (ADR-006 §2) |
| Runtime sim↔real switching UI (S-84) | 007+ (admin action; audit-logged) |
| Azure DevOps migration, Key Vault setup | IT (ADR-004 §6 pending) |
| Committing calendar dates or person-day estimates | ADR-005 §3 (relative timeline only) |

## 6. Acceptance criteria (feature-level)

> Each AC cites its PP source and is testable by Ralph with the verification
> command given. "CI green" always means the full ADR-004 §2.3 gate.

**A1 — App shell (PP-056 AC-1 enablement).** `dotnet run --project
src/ICE.ProjectLifecycle.Api` boots from a clean checkout; `GET /health`
returns HTTP 200 with `environment` and the provider map; `GET /api/version`
returns version + git SHA. *Verify: `dotnet test
tests/ICE.ProjectLifecycle.UnitTests --filter Health` green.*

**A2 — Contracts (PP-016, ADR-006 §4.1).**
`src/ICE.ProjectLifecycle.Domain/` defines the two independent result
families — `Result`/`Ok`/`Err` (valueless) and `Result<T>`/`Ok<T>`/`Err<T>`
(value-bearing; `Result<T>` does **not** derive from `Result`) — plus the
closed `AdapterError` enum with `IsRetryable` and `AdapterHealth`; the 8
ports in `src/ICE.ProjectLifecycle.Domain/Ports/` are C# interfaces with the
per-method signatures in `contracts.md` §2 (value-producing methods →
`Task<Result<T>>`, valueless commands → `Task<Result>`). *Verify: `dotnet
build` clean; `dotnet test tests/ICE.ProjectLifecycle.UnitTests --filter
Contracts` green.*

**A3 — Config + prod-forbids-sim (architecture §7, ADR-004).** Config model
reads `appsettings.json` + env overrides; **starting with
`environment=prod` and any `sim` provider fails boot with a clear error.**
Config validation does **not** instantiate sim adapters (correction #1 — the
provider-factory forward dependency is eliminated; DI wiring is a separate
task). *Verify: `dotnet test
tests/ICE.ProjectLifecycle.UnitTests --filter Config` green (all branch
cases, incl. prod-forbidden-sim).*

**A4 — DB foundation (ADR-006 §2).** EF Core + SQLite wired from config;
a baseline migration with `schema_version` table only (no domain tables);
`dotnet ef database update` + `dotnet ef database update --target` succeed on
a temp SQLite file. **No domain tables in 001.** *Verify: `dotnet test
tests/ICE.ProjectLifecycle.IntegrationTests --filter Migration` green.*

**A5 — Identity sim + fixtures (ADR-003 §5.2, synthetic §4.1).**
`fixtures/identity/users.json` committed with the §4.1 shape incl. `negative`
block (`expiredToken`, `unknownSubject`, `mfaPendingUser`);
`SimIdentityProvider` serves it deterministically; `sim-state/` is
gitignored. *Verify: `dotnet test
tests/ICE.ProjectLifecycle.UnitTests --filter SimIdentity` green.*

**A6 — Contract test model (PP-016 AC-3, synthetic §5).**
`tests/ICE.ProjectLifecycle.ContractTests/IdentityContractTests.cs` is
adapter-agnostic (takes an `IIdentityProvider`, asserts the identity contract
incl. error mapping); green against `SimIdentityProvider` in CI. *Verify:
`dotnet test tests/ICE.ProjectLifecycle.ContractTests` green.*

**A7 — Architecture test (PP-016 AC-2, architecture §3).** CI fails if any
`src/ICE.ProjectLifecycle.Domain/**` file imports a vendor NuGet package or
simulator/implementation types. 001 ships the test + a
`src/ICE.ProjectLifecycle.Domain/` placeholder namespace it governs. *Verify:
`dotnet test
tests/ICE.ProjectLifecycle.UnitTests --filter Architecture` green
(deliberate-violation case fails, clean case passes).*

**A8 — Frontend shell (PP-057 AC-1).** React + Vite + TS app in
`src/ICE.ProjectLifecycle.Web/`; `npm ci && npm run build` succeeds; branded
shell page consumes `brand/tokens.json` (colors, typography, header/footer)
with **placeholder** values. *Verify: `npm run build` exit 0; build output
contains token-derived CSS custom properties.*

**A9 — CI pipeline (PP-056 AC-1/AC-3, ADR-004 §4).**
`.github/workflows/ci.yml` runs on push/PR: `dotnet restore` → `dotnet build`
→ `dotnet test` (unit + contract + integration + architecture) →
`npm ci && npm run build`, packaging step (artifact upload). A green run on
`main` is the merge gate. *Verify: green Actions run on a pushed branch.*

**A10 — Release workflow + release controls (PP-056 AC-2, ADR-004 §4).**
`.github/workflows/release.yml`: tag `v*` → package → DRY-RUN deploy job
(disabled/placeholder until IT provisions targets) → PROD deploy job behind
an Actions *environment* approval (named approvers pending). Branch-protection
runbook committed (owners: IT/PM, pending — ADR-004 §6). *Verify: workflow
YAML passes `actionlint`; runbook doc committed.*

**A11 — Dev-loop docs.** `README.md`: two-command loop, config reference,
DEV/DRY-RUN/PROD model, spec/traceability pointers, pending-IT list. *Verify:
a fresh developer can boot both servers following only the README.*

**A12 — Repo hygiene.** `git diff --check` clean; `.gitignore` includes
`sim-state/`, `bin/`, `obj/`, `*.db`, `node_modules/`, `dist/`; no secrets in
any committed file; `package-lock.json` committed for CI parity. *Verify:
`git diff --check`; `grep -r` scan for secret patterns (N15).*

## 7. Simulator boundaries (001)

- Deterministic, seeded, no network, no wall-clock dependence
  (synthetic strategy §1).
- `fixtures/` committed; `sim-state/` ephemeral + gitignored;
  `Simulators.Reset()` restores state (synthetic §2).
- Values use `SIM-` prefixes (synthetic §1 "Shape-stable, value-synthetic").
- Error paths exercised: expired token, unknown subject, MFA-pending
  (synthetic §4.1 `negative`).
- 001 sim surface = **identity port only** (§4). No other sim adapter code
  ships in 001.
- Simulators are **forbidden in prod** — enforced by A3 startup validation
  (synthetic §7).

## 8. Configuration requirements

- Schema: `contracts.md` §3 (architecture §7 keys verbatim + `database:ConnectionStrings`).
- Files: `config/appsettings.Development.json` (all sim, committed),
  `config/appsettings.DryRun.json` and `config/appsettings.Production.json`
  (templates, committed, **no secrets** — secrets via env/secret store per
  architecture §7), `.env.example` (committed).
- Env model: DEV (all sim allowed) / DRY-RUN (real identity + real DB + sim
  external adapters + real doc store) / PROD (real adapters only) (ADR-004 §4).
- Secrets: GitHub Actions secrets / Azure Key Vault per ADR-004 §6 (pending);
  **never in the repo**.
- Dev/CI default = sim everywhere; prod = real-only, boot-fail otherwise.

## 9. Repository / project structure

```
ice_demo/
├── ICE.ProjectLifecycle.sln
├── Directory.Build.props
├── Directory.Packages.props
├── global.json                       # pin .NET SDK 10.x
├── nuget.config
├── .editorconfig
├── .gitignore                        # + bin/ obj/ sim-state/ *.db web/node_modules web/dist
├── src/
│   ├── ICE.ProjectLifecycle.Domain/
│   │   ├── Contracts/                # Result, Ok, Err, Result<T>, Ok<T>, Err<T> (two families), AdapterError, AdapterHealth
│   │   ├── Ports/                    # 8 port interfaces (IIdentityProvider, …)
│   │   └── ValueObjects/             # port contract types
│   ├── ICE.ProjectLifecycle.Application/
│   │   └── Configuration/            # config POCOs, validation
│   ├── ICE.ProjectLifecycle.Infrastructure/
│   │   ├── Persistence/              # EF Core + SQLite + migrations
│   │   ├── Simulators/               # SimIdentityProvider (001), Simulators facade
│   │   └── Simulators/Seed/          # seed/reset logic
│   ├── ICE.ProjectLifecycle.Api/
│   │   ├── Program.cs                # app factory, DI wiring, startup validation
│   │   └── Endpoints/                # /health, /api/version
│   └── ICE.ProjectLifecycle.Web/     # React + Vite + TS
│       ├── src/
│       │   └── styles/tokens.ts      # imports brand/tokens.json
│       └── package.json, tsconfig.json, vite.config.ts
├── tests/
│   ├── ICE.ProjectLifecycle.UnitTests/
│   │   ├── ContractsTests.cs
│   │   ├── ConfigTests.cs
│   │   ├── SimIdentityTests.cs
│   │   ├── HealthTests.cs
│   │   └── ArchitectureTests.cs
│   ├── ICE.ProjectLifecycle.IntegrationTests/
│   │   └── MigrationTests.cs
│   └── ICE.ProjectLifecycle.ContractTests/
│       └── IdentityContractTests.cs  # (001) — 007 extends to 8 files
├── fixtures/
│   └── identity/users.json           # (001) — 007 adds other fixture dirs
├── brand/tokens.json                 # placeholder brand tokens (PP-057)
├── config/
│   ├── appsettings.Development.json
│   ├── appsettings.DryRun.json
│   └── appsettings.Production.json
├── .github/workflows/ci.yml, release.yml
├── README.md
└── docs/, specs/                     # unchanged BMAD + Spec Kit artifacts
```

> **Resolved (former Issue 2):** the repo previously had an empty `src/`
> directory that conflicted with a top-level `app/` Python package (ADR-001
> era). On the .NET baseline the C# projects live **under `src/`** as the
> natural MSBuild layout. The empty `src/` directory is no longer a conflict;
> it is the correct parent for the C# project tree.

## 10. CI/CD expectations (normative)

Per ADR-004 §2.3 + §4:

1. **Substrate:** GitHub Actions, Linux runners. Azure DevOps is the
   IT-mandate fallback only (ADR-004 §6, pending).
2. **Branch model:** `main` protected (PRs only, ≥1 review, required checks)
   — enforced in GitHub repo settings; the runbook in 001 documents the
   exact policy; IT/PM applies it (pending, ADR-004 §6).
3. **CI gate (`ci.yml`):** `dotnet restore` → `dotnet build` → `dotnet test`
   (unit + contract + integration + architecture) → `npm ci && npm run build`
   → package (backend publish + web dist) → upload artifacts.
4. **DEV deploy:** job defined, **disabled/placeholder** until IT provisions
   the target (D-033, ADR-004 §6). When enabled: auto on `main`, all sim.
5. **Release (`release.yml`):** tag `v*.*.*` from `main` → package →
   DRY-RUN (placeholder) → PROD behind Actions *environment* approval;
   approver names pending (ADR-004 §6).
6. **Migration console:** controlled operator job, **never** a pipeline step
   (ADR-004 §4.7, architecture §11) — documented in the runbook (N13).
7. **Determinism:** lockfiles (`package-lock.json`, NuGet restore from
   `Directory.Packages.props`) are the parity mechanism between Windows dev
   and Linux CI (ADR-004 §2.1 note).

## 11. Test expectations (001 slice)

| Level | 001 scope |
|---|---|
| Unit | `tests/ICE.ProjectLifecycle.UnitTests/` — contracts, config branches (incl. prod-forbids-sim), sim identity behavior, health endpoint, architecture (PP-016) |
| Contract | `tests/ICE.ProjectLifecycle.ContractTests/IdentityContractTests.cs` vs `SimIdentityProvider` (007: the other 7 files) |
| Integration | `tests/ICE.ProjectLifecycle.IntegrationTests/` — EF Core + SQLite migration up/down (no domain flows yet; WF-1…6 need 002+) |
| Architecture | `tests/ICE.ProjectLifecycle.UnitTests/ArchitectureTests.cs` — PP-016 import-boundary test (feature-sequence §5 finding 7) |
| Frontend | `npm run build` (compilation gate; no e2e framework in 001) |

All test runs deterministic and offline (no network in CI; sim only).

## 12. Dependencies & open confirmations

### 12.1 In-repo dependencies (already satisfied)

| Dependency | Status |
|---|---|
| D-001 platform stack | **DECIDED FOR DEVELOPMENT** (ADR-006) |
| D-029 CI/CD & release controls | **DECIDED FOR DEVELOPMENT** (ADR-004) |
| D-007 identity mechanism (001 scaffolds only) | **DECIDED FOR DEVELOPMENT** (ADR-003) |
| D-031 delivery model | **DECIDED FOR DEVELOPMENT** (ADR-005) |
| GitHub remote | Exists: `https://github.com/kcfike/ice-demo.git` |
| Local toolchain | .NET SDK 9.0.313 (note: .NET 10 SDK not yet installed locally — see §13 A1), Node 22.17.1 / npm 10.9.2 ✓ |

### 12.2 Open confirmations — **NOT requirements**

Carried verbatim from the ADRs; each must remain *pending* in 001's
implementation. None may be silently converted into a confirmed client
requirement.

**Platform (ADR-006 §7):**

| Item | Source | Owner | Affects |
|---|---|---|---|
| C# / .NET acceptable as the production application standard | ADR-006 §7 | IT | Dev/demo baseline is C#/.NET; if ICE mandates a different standard, the architecture (ports, simulators, domain model) is unchanged |
| Required .NET LTS version | ADR-006 §7 | IT | Dev/demo assumes .NET 10 LTS; a different LTS is a `global.json`/TFM change |
| Azure SQL vs SQL Server (or another approved database) as production DB | ADR-006 §7 | IT | Dev/demo uses SQLite; swap is connection-string + EF Core provider change |
| Azure App Service vs containers (or another hosting standard) | ADR-006 §7 | IT | Dev/demo uses ASP.NET Core dev server |
| GitHub Actions vs Azure DevOps | ADR-006 §7 | IT | Dev/demo uses GitHub Actions; the gate model is substrate-independent |
| Entra tenant + app-registration requirements | ADR-006 §7 | IT | Dev/demo uses `SimIdentityProvider`; see ADR-003 |
| Azure Blob vs SharePoint for document storage | ADR-006 §7 / ADR-002 §6 | IT/OPS | Dev/demo uses `SimDocumentStore`; see ADR-002 |

**Governance rule (standing, not a confirmation item):** no production
infrastructure choice is silently treated as confirmed. Every production
choice above stays *Pending* until ICE IT explicitly confirms it (ADR-006 §7).

**CI/CD & release (ADR-004 §6):**

| Item | Source | Owner | Affects |
|---|---|---|---|
| DEV/DRY-RUN/PROD deploy targets provisioned | ADR-004 §6 / D-033 | IT | Deploy jobs stay placeholder |
| Named approvers for PROD gate | ADR-004 §6 | IT/PM | Actions environment config |
| Branch-protection policy applied | ADR-004 §6 | IT/PM | GitHub repo settings |
| Artifact/registry location | ADR-004 §6 | IT | Default: GH Actions artifact |
| Secrets management (Actions secrets / Key Vault) | ADR-004 §6 | IT | Secret provisioning |

**Identity (ADR-003 §6):**

| Item | Source | Owner | Affects |
|---|---|---|---|
| RBAC matrix D-009, session timeout D-027, MFA method D-028 | ADR-003 §6 | IT/PM | Feature 002 values, not mechanism |

**Document store (ADR-002 §6 — all 5 items carried per correction #6):**

| Item | Source | Owner | Affects |
|---|---|---|---|
| Azure Blob acceptable for production (vs SharePoint mandate) | ADR-002 §6 | IT/OPS | D-002 decided for development (Blob default); port and contract tests unchanged if SharePoint is mandated |
| Blob container, account, and Entra role/scope model | ADR-002 §6 | IT | Needed when the real adapter is verified (D-033) |
| Retention policy for stored documents | ADR-002 §6 | IT | Tracked under D-030 (audit/doc retention) |
| Whether existing SharePoint library metadata must be reused or mirrored | ADR-002 §6 | OPS | If "reuse" is required, Option B may be re-evaluated at adapter-verification time |
| Large-file limit and supported MIME types for billing packets (PDF) | ADR-002 §6 | IT | Confirm against PP-028 packet sizes |

**Delivery model (ADR-005 §5 — all 5 items carried per correction #6):**

| Item | Source | Owner | Affects |
|---|---|---|---|
| Commit to input dates for DISC-001 (T-Sheets spike) evidence | ADR-005 §5 | PM/ICE | Until committed, Feature 008 stays gated and the timeline stays relative |
| Commit to input dates for DISC-012 (migration source files) + D-025 sign-off | ADR-005 §5 | PM/OPA/NSA | Until committed, the R1 exit date is not set |
| Confirm Ralph-as-primary-implementer is the intended delivery model | ADR-005 §5 | PM | ADR-005 assumes it (constraint 7) |
| Confirm no calendar commitment is being made in this run | ADR-005 §5 | PM | Relative timeline only |
| Any committed human developer capacity that would change the AI-track throughput | ADR-005 §5 | PM | If any, the AI track accelerates; gates unchanged |

**Other:**

| Item | Source | Owner | Affects |
|---|---|---|---|
| Real brand assets (DISC-016) | PRD PP-057 AC-2 | PM/IT | `brand/tokens.json` stays placeholder |
| Performance targets (D-026 / DISC-015) | architecture §10 | PM/ICE | No 001 numbers asserted |

## 13. Assumptions & issues to resolve before Ralph begins

**Assumptions (adopted in this spec, flagged for sign-off):**

1. **A1 — .NET 10 SDK availability.** The baseline targets .NET 10 LTS
   (ADR-006 §2). The local environment currently has .NET SDK 9.0.313 only.
   Ralph must install the .NET 10 SDK before building (or CI must use the
   .NET 10 SDK). This is an environment prerequisite, not an architectural
   choice. *Flag for sign-off.*
2. **A2 (Issue 1) — 001/007 simulator split** per §4: 001 ships identity sim
   + facade + contract-test model; 007 ships the other 7 sims + full suite.
   This reconciles ADR-006 §4.1 with feature-sequence §5 finding 6.
3. **A3 — EF Core migrations** for the data layer. ADR-006 §2 names Entity
   Framework Core; EF Core's built-in migration tooling (`dotnet ef`) is the
   natural companion. Recorded in `research.md`.
4. **A4 — Deploy jobs ship disabled/placeholder** (ADR-004 §5.1: real targets
   are IT-provisioned, D-033). Pipeline *logic* is complete; execution is
   gated on IT.
5. **A5 — 001 has no domain tables**; the DB layer is EF Core DbContext +
   baseline migration only.
6. **A6 — Frontend gate is `npm run build` only** (ADR-004 §2.3 item 6);
   lint/typecheck for web follows if ADR-004 §2.3 item 1's `npm run lint`
   is configured later — 001 sets it up as a no-op script.
7. **A7 — `environment` values are `dev | dryrun | prod`** (normalization of
   architecture §7's `dev | test | prod` to the ADR-004 environment names).

**Issues to resolve (human review gate):**

| # | Issue | Recommendation |
|---|---|---|
| 1 | ADR-006 §4.1 vs feature-sequence §5#6 conflict on sim scope | Adopt §4 split; human sign-off |
| 2 | ~~Empty `src/` vs ADR-001 `app/`~~ — **Resolved**: .NET projects live under `src/` as the natural MSBuild layout; the old conflict no longer applies | No action needed |
| 3 | `sim-state/` not in `.gitignore` | Add in N1 (synthetic §2 requires it) |
| 4 | `audit` fixture dir: 001 defines the AuditService *port* but audit's sim is test-harness-only (synthetic §3 note) — confirm 001 ships no audit fixture dir | Adopt: port only, no fixtures (006 owns audit table) |
| 5 | Branch protection & PROD approvers are repo-settings work, not code | Runbook in N13; IT/PM to apply (tracked as open) |

## 14. Traceability

| PP | AC | ADR/Decision | DISC | Task(s) |
|---|---|---|---|---|
| PP-056 AC-1 | A1, A9, A10 | ADR-004 §4 (D-029) | — | N2, N7, N12, N13 |
| PP-056 AC-2 | A10 | ADR-004 §4 (D-029) | — | N13 |
| PP-056 AC-3 | A6, A7, A9 | ADR-004 §2.3, §4 | — | N9, N10, N12 |
| PP-057 AC-1 | A8 | ADR-006 §2 (D-001) | — | N11 |
| PP-057 AC-2 | — (placeholder only) | — | DISC-016 (pending) | N11 (placeholder) |
| PP-016 AC-1 | A2, A5 | ADR-003 §5.2 (D-007); architecture §5 | — | N3, N4, N5 |
| PP-016 AC-2 | A7 | architecture §3; synthetic §5 | — | N10 |
| PP-016 AC-3 | A3, A6 | architecture §7; synthetic §7 | — | N6, N7, N9 |
| PP-045 (baseline) | A1 (enabling) | ADR-006 (D-001) | DISC-015 (pending) | N2, N7 (no targets asserted) |
| — | A4 | ADR-006 §2 (D-001) | — | N8 |
| — | A11, A12 | ADR-004 §5 (D-029) | — | N1, N14, N15 |
