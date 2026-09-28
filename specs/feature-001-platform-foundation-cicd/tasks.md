# Feature 001 — Tasks

> Ralph-executable task sequence for the **C# / .NET 10 LTS** baseline
> (ADR-006). Each task is independently executable: bounded file set, its own
> acceptance criteria, and an exact verification command. No task may leave
> the tree red.
>
> **Local full gate (from N2 onward):**
> `dotnet build && dotnet test`
> (from N11 onward add: `cd src/ICE.ProjectLifecycle.Web && npm ci && npm run build`)
>
> **Corrections carried (from the six consistency fixes):**
> 1. N6 (config) validates provider *selection* + prod-forbids-sim only — it
>    does **not** instantiate sim adapters. DI wiring is N7 (separate).
> 2. ADR-002 is DECIDED FOR DEVELOPMENT; only its §6 pending confirmations are
>    carried (specification §12.2).
> 3. `synthetic-data-strategy.md` §5 error mapping is enforced exactly in N9:
>    `expiredToken → validate_session → SessionExpired`;
>    `unknownSubject → validate_session → SessionExpired`;
>    `mfaPendingUser → begin_mfa → MFA verification flow`.
> 4. Runbook task reference: the branch-protection + release runbooks are
>    owned by **N13**; no other task references a different runbook task.
> 5. A8 (frontend) maps **only** to PP-057 AC-1 (specification §14) — it is
>    *not* in the PP-016 AC-3 row.
> 6. All five ADR-005 §5 pending confirmations are carried in specification
>    §12.2 and restated in N14's README.

---

## Phase A — Tooling & shell

### N1 — .NET solution scaffold & repo hygiene
**Files:** `ICE.ProjectLifecycle.sln`, `Directory.Build.props`,
`Directory.Packages.props`, `global.json`, `nuget.config`, `.editorconfig`,
`.gitignore` (edit), `src/ICE.ProjectLifecycle.Domain/` (empty project),
`src/ICE.ProjectLifecycle.Application/` (empty project),
`src/ICE.ProjectLifecycle.Infrastructure/` (empty project),
`src/ICE.ProjectLifecycle.Api/` (empty project),
`src/ICE.ProjectLifecycle.Web/` (empty dir placeholder),
`tests/ICE.ProjectLifecycle.UnitTests/` (empty project),
`tests/ICE.ProjectLifecycle.IntegrationTests/` (empty project),
`tests/ICE.ProjectLifecycle.ContractTests/` (empty project),
`config/` (empty directory for appsettings templates),
`fixtures/` (empty directory),
`brand/` (empty directory).
**Prereqs:** .NET 10 SDK installed locally (specification §13 A1); Node 22.
**Do:**
- `dotnet new sln -n ICE.ProjectLifecycle`.
- `dotnet new classlib` for each of the 5 `src/` projects; `dotnet new xunit`
  for each of the 3 `tests/` projects.
- Add all 8 projects to the solution; wire project references:
  Domain ← Application ← Infrastructure ← Api; tests reference the projects
  under test (UnitTests → Domain, Application; IntegrationTests →
  Infrastructure; ContractTests → Domain, Infrastructure).
- `global.json`: pin SDK `10.0.x`.
- `Directory.Build.props`: `<TargetFramework>net10.0</TargetFramework>`,
  `<Nullable>enable</Nullable>`, `<ImplicitUsings>enable</ImplicitUsings>`.
- `Directory.Packages.props`: central package management; add
  `Microsoft.EntityFrameworkCore.Sqlite`, `Microsoft.EntityFrameworkCore`,
  `xunit`, `xunit.runner.visualstudio`, `Microsoft.NET.Test.Sdk` with version
  floors from `research.md` §1.
- `.gitignore`: add `bin/`, `obj/`, `sim-state/`, `*.db`,
  `src/ICE.ProjectLifecycle.Web/node_modules/`,
  `src/ICE.ProjectLifecycle.Web/dist/`.
- `.editorconfig`: C# style (file-scoped namespaces, var preferences) +
  generic `trim_trailing_whitespace = true`.
- `nuget.config`: point at `https://api.nuget.org/v3/index.json`.
- Remove the empty default `Class1.cs` from each classlib; leave an empty
  `AssemblyInfo.cs` or a single placeholder type per project so the build
  succeeds.
**Acceptance:** `dotnet restore` succeeds; `dotnet build` succeeds with 0
errors; all 8 projects appear in the solution; `.gitignore` includes
`sim-state/`, `bin/`, `obj/`, `*.db`; `git diff --check` clean.
**Verify:** `dotnet restore && dotnet build`
**Satisfies:** A12 (partial)

### N2 — ASP.NET Core app shell (`/health`, `/api/version`)
**Files:** `src/ICE.ProjectLifecycle.Api/Program.cs`,
`src/ICE.ProjectLifecycle.Api/Endpoints/HealthEndpoint.cs`,
`src/ICE.ProjectLifecycle.Api/Endpoints/VersionEndpoint.cs`,
`src/ICE.ProjectLifecycle.Api/appsettings.json` (default),
`tests/ICE.ProjectLifecycle.UnitTests/HealthTests.cs`.
**Prereqs:** N1.
**Do:**
- `Program.cs`: minimal API; register health checks; map `/health` and
  `/api/version`.
- `HealthEndpoint`: return JSON `{ "environment": "dev", "providers": { ... } }`
  — read from `IConfiguration` (not yet fully wired; use a stub config
  provider that returns `environment=dev` and an empty provider map for now;
  N6/N7 will replace this with the real config model).
- `VersionEndpoint`: return JSON `{ "version": "0.1.0", "gitSha": "<sha>" }`
  — git SHA from build metadata (`git rev-parse HEAD` at build time via
  `Directory.Build.props` or a simple constant for now).
- Startup logging: log `environment` + provider map at `Information` level.
- `HealthTests`: use `Microsoft.AspNetCore.Mvc.Testing`
  (`WebApplicationFactory<Program>`) to assert both routes return 200 with
  the expected shape.
**Acceptance:** `dotnet run --project src/ICE.ProjectLifecycle.Api` boots;
`GET /health` returns 200 with `environment` + `providers`; `GET /api/version`
returns 200 with `version` + `gitSha`; `HealthTests` green.
**Verify:** `dotnet test tests/ICE.ProjectLifecycle.UnitTests --filter Health`
**Satisfies:** A1

### N3 — Domain shared contracts (two result families, `AdapterError`, `AdapterHealth`)
**Files:** `src/ICE.ProjectLifecycle.Domain/Contracts/Result.cs`,
`src/ICE.ProjectLifecycle.Domain/Contracts/AdapterError.cs`,
`src/ICE.ProjectLifecycle.Domain/Contracts/AdapterHealth.cs`,
`tests/ICE.ProjectLifecycle.UnitTests/ContractsTests.cs`.
**Prereqs:** N1.
**Do:** per `contracts.md` §1 — **two independent result families** (`Result<T>`
does **not** derive from `Result`):
- `AdapterError` enum: exactly 5 members (`Auth`, `NotFound`, `Timeout`,
  `RateLimited`, `Other`); `IsRetryable` extension derived from the enum
  (`Timeout` and `RateLimited` are retryable; the others are not).
- **Valueless family:** `Result` (abstract record, `IsRetryable` abstract);
  `Ok : Result` (sealed); `Err(AdapterError Code, string Message,
  IReadOnlyDictionary<string,string> Detail) : Result` (sealed).
- **Value-bearing family (independent):** `Result<T>` (abstract record,
  `IsRetryable` abstract); `Ok<T>(T Value) : Result<T>` (sealed);
  `Err<T>(AdapterError Code, string Message,
  IReadOnlyDictionary<string,string> Detail) : Result<T>` (sealed).
- `AdapterHealth`: `Status` (`"healthy" | "degraded" | "unavailable"`),
  `LatencyMs: int`, `Message: string = ""`.
- `ContractsTests`: assert enum has exactly 5 members; `Timeout`/`RateLimited`
  are retryable; `Auth`/`NotFound`/`Other` are not; `Ok`, `Ok<T>`, `Err`,
  `Err<T>` are all sealed; `Ok<T> : Result<T>` and `Err<T> : Result<T>`;
  `Ok : Result` and `Err : Result`; **`Result<T>` does NOT derive from
  `Result`** (`typeof(Result).IsAssignableFrom(typeof(Result<int>))` is false);
  `Ok<T>.Value` returns the value; `Err<T>.IsRetryable` reflects the enum;
  `Err.Detail` / `Err<T>.Detail` defaults to empty dict.
**Acceptance:** `dotnet build` clean; `ContractsTests` green (5 enum members,
retryability correct, 4 sealed types, two independent hierarchies,
`Result<T>` ≠ `Result` hierarchy, `Ok<T>.Value`, `Err`/`Err<T>` detail
default).
**Verify:** `dotnet test tests/ICE.ProjectLifecycle.UnitTests --filter Contracts`
**Satisfies:** A2 (partial)

---

## Phase B — Ports, sim & data foundation

### N4 — Port interfaces (8 C# interfaces) + value objects
**Files:** `src/ICE.ProjectLifecycle.Domain/Ports/IIdentityProvider.cs`,
`ITimekeepingService.cs`, `IProcurementService.cs`, `IDocumentStore.cs`,
`IEmployeeDirectory.cs`, `IARHandoffService.cs`, `IAuditService.cs`,
`INotificationService.cs`,
`src/ICE.ProjectLifecycle.Domain/ValueObjects/` (one file per port's value
objects: `IdentityUser.cs`, `IdentityContext.cs`, `AuthSession.cs`,
`MfaChallenge.cs`, `LoginRequest.cs`, `TimesheetEntry.cs`,
`CreateJobRequest.cs`, `PO.cs`, `DocumentRef.cs`, `Employee.cs`,
`HandoffRecord.cs`, `AuditEvent.cs`, `Notification.cs`, etc. —
shapes per `contracts.md` §2).
**Prereqs:** N3.
**Do:**
- Define all 8 port interfaces in C# with the **exact per-method signatures**
  in `contracts.md` §2. Each method declares its result type individually:
  value-producing methods return `Task<Result<TOutput>>` (success
  `Ok<TOutput>`, failure `Err<TOutput>`); valueless commands return
  `Task<Result>` (success `Ok`, failure `Err`). The generic parameter always
  matches the method's documented successful output type. All methods are
  `async`. **Do not apply one return type to an entire interface** — each
  method is chosen per its documented output.
- Domain value objects (port contract types) as C# records in
  `ValueObjects/`.
- **No vendor/NuGet references** in the Domain project (enforced by N10).
- Domain project references **only** BCL + its own `Contracts/` namespace.
**Acceptance:** `dotnet build` clean; all 8 interfaces present; every method
signature matches `contracts.md` §2 exactly (value-producing →
`Task<Result<T>>`, valueless → `Task<Result>`); value objects match
`contracts.md` §2 shapes; Domain project has **zero** NuGet package
references (verified by N10).
**Verify:** `dotnet build`
**Satisfies:** A2

### N5 — Identity fixtures + `SimIdentityProvider` + `Simulators` facade
**Files:** `fixtures/identity/users.json`,
`src/ICE.ProjectLifecycle.Infrastructure/Simulators/SimIdentityProvider.cs`,
`src/ICE.ProjectLifecycle.Infrastructure/Simulators/Simulators.cs`,
`src/ICE.ProjectLifecycle.Infrastructure/Simulators/Seed/IdentitySeeder.cs`,
`tests/ICE.ProjectLifecycle.UnitTests/SimIdentityTests.cs`.
**Prereqs:** N4.
**Do:** per `contracts.md` §4 + synthetic-data-strategy §4.1:
- `fixtures/identity/users.json`: §4.1 shape with `negative` block
  (`expiredToken`, `unknownSubject`, `mfaPendingUser`).
- `SimIdentityProvider` implements `IIdentityProvider` (typed results):
  - `AuthenticateAsync`: known subject → `Ok<AuthSession>` with fixture
    roles; failure (bad credentials/MFA code) →
    `Err<AuthSession>(AdapterError.Auth)`.
  - `ValidateSessionAsync`: `negative.expiredToken` →
    `Err<IdentityContext>(AdapterError.Auth)` (`SessionExpired` semantic);
    `negative.unknownSubject` →
    `Err<IdentityContext>(AdapterError.Auth)` (`SessionExpired` semantic);
    valid token → `Ok<IdentityContext>`.
  - `BeginMfaAsync`: `negative.mfaPendingUser` → `Ok<MfaChallenge>` (method
    `"simulated"`), followed by the MFA verification flow —
    `AuthenticateAsync` with the MFA code → `Ok<AuthSession>`; other users
    (no pending MFA) → `Err<MfaChallenge>(AdapterError.Other, "mfa not
    required")`.
  - `GetUserAsync`: known subject → `Ok<IdentityUser>`; unknown subject →
    `Err<IdentityUser>(AdapterError.NotFound)`.
  - `MapRolesAsync`: stable — same `IdentityUser` → same role list across
    runs (`Ok<IReadOnlyList<Role>>`).
  - Deterministic: identical inputs → identical outputs.
  - Runtime state written to `sim-state/identity/` (gitignored);
    `Simulators.Reset()` restores it.
  - **No network, no clock dependence.**
- `Simulators` facade: `Seed(fixturesDir)` (fixtures/ → sim-state/),
  `Reset()` (clear sim-state/, re-seed). 001 ships the facade +
  `SimIdentityProvider` **only**; the remaining 7 sim adapters are Feature 007.
- `SimIdentityTests`: exercise all 5 methods above; assert determinism (run
  twice, compare); assert `sim-state/` is created and gitignored.
**Acceptance:** `SimIdentityTests` green (AuthenticateAsync known →
Ok<AuthSession>; ValidateSessionAsync expired/unknown →
Err<IdentityContext>(Auth) [SessionExpired]; BeginMfaAsync → MFA flow;
GetUserAsync unknown → Err<IdentityUser>(NotFound); MapRolesAsync stable;
determinism); `sim-state/` created and in `.gitignore`.
**Verify:** `dotnet test tests/ICE.ProjectLifecycle.UnitTests --filter SimIdentity`
**Satisfies:** A5

### N6 — Configuration model + prod-forbids-sim validation
**Files:** `src/ICE.ProjectLifecycle.Application/Configuration/AppConfig.cs`,
`src/ICE.ProjectLifecycle.Application/Configuration/ConfigValidator.cs`,
`src/ICE.ProjectLifecycle.Application/Configuration/ProviderSelection.cs`,
`config/appsettings.Development.json`,
`config/appsettings.DryRun.json`,
`config/appsettings.Production.json`,
`.env.example`,
`tests/ICE.ProjectLifecycle.UnitTests/ConfigTests.cs`.
**Prereqs:** N1.
**Do:** per `contracts.md` §3 (architecture §7):
- `AppConfig`: C# POCO matching the config schema in `contracts.md` §3
  (`environment`, `identity`, `timekeeping`, `procurement`, `employee`,
  `documents`, `arhandoff`, `audit`, `notifications`, `database`).
- `ProviderSelection`: per-adapter, per-environment provider selection
  (`"sim" | "entra" | "tsheets" | …` per port).
- `ConfigValidator`:
  - `environment == "prod"` ⇒ **every** provider must be non-`sim`.
    Violation throws `ConfigurationValidationException` with a clear
    message naming the offending key.
  - `environment == "dryrun"` ⇒ identity real, DB real, external adapters
    sim allowed, documents real (ADR-004 §4.4).
  - Secrets (tokens, connection strings) come from env vars / secret store,
    **never** from the config file.
  - Dev/CI default (`appsettings.Development.json`): all providers `sim`,
    `audit.sink = db`, SQLite connection string.
- `appsettings.Production.json` / `appsettings.DryRun.json`: templates with
  **no secrets** (empty strings for secret fields).
- `.env.example`: placeholder names for secret env vars.
- **Correction #1:** `ConfigValidator` validates provider *selection* only.
  It does **not** instantiate sim adapters or any provider. DI wiring is N7.
- `ConfigTests`: assert prod + any sim → throws; dryrun validation; dev
  config loads with all sim; no secret values in committed config files.
**Acceptance:** `ConfigTests` green (prod-forbids-sim throws with named key;
dryrun validates per ADR-004 §4.4; dev loads with all sim; no secrets in
committed config); `ConfigValidator` does **not** reference `SimIdentityProvider`
or any other sim type.
**Verify:** `dotnet test tests/ICE.ProjectLifecycle.UnitTests --filter Config`
**Satisfies:** A3

### N7 — DI wiring (ASP.NET Core)
**Files:** `src/ICE.ProjectLifecycle.Api/Program.cs` (update),
`src/ICE.ProjectLifecycle.Api/DependencyInjection/ServiceCollectionExtensions.cs`,
`tests/ICE.ProjectLifecycle.UnitTests/DiWiringTests.cs`.
**Prereqs:** N2, N5, N6.
**Do:**
- `ServiceCollectionExtensions`: `AddProjectLifecycle(IServiceCollection, IConfiguration)`:
  - Bind `AppConfig` from `IConfiguration`.
  - Run `ConfigValidator.Validate(appConfig)` — boot-fail on prod + sim.
  - If `identity.provider == "sim"`: register `SimIdentityProvider` as
    `IIdentityProvider`.
  - If `identity.provider != "sim"`: register a stub that throws
    `NotImplementedException("Feature 002: real identity adapter")`.
  - For the other 7 ports: register a stub that throws
    `NotImplementedException("Feature 007: sim adapter not yet implemented")`
    — do **not** silently stub them.
  - Register `Simulators` facade (if any sim is active).
- Update `Program.cs` to call `AddProjectLifecycle` before `MapHealth` /
  `MapVersion`.
- `DiWiringTests`: assert that in dev mode, `IIdentityProvider` resolves to
  `SimIdentityProvider`; in prod mode with sim, the service collection throws
  at `Validate()` before any resolution.
**Acceptance:** `DiWiringTests` green; app boots in dev mode and resolves
`IIdentityProvider` to `SimIdentityProvider`; app fails boot in prod mode
with sim providers; the other 7 ports throw `NotImplementedException` with
the Feature 007 message when resolved.
**Verify:** `dotnet test tests/ICE.ProjectLifecycle.UnitTests --filter DiWiring`
**Satisfies:** A3 (DI wiring half), A1 (app boots)

### N8 — EF Core + SQLite + baseline migration
**Files:** `src/ICE.ProjectLifecycle.Infrastructure/Persistence/AppDbContext.cs`,
`src/ICE.ProjectLifecycle.Infrastructure/Persistence/Migrations/` (EF Core
migrations — generated by `dotnet ef`),
`tests/ICE.ProjectLifecycle.IntegrationTests/MigrationTests.cs`.
**Prereqs:** N6.
**Do:** per `data-model.md` §1–§2:
- `AppDbContext`: `DbContext` with `OnConfiguring` reading the SQLite
  connection string from `IConfiguration` (or injected).
- **Baseline migration `0001_baseline`**: contains *no domain tables*. It
  establishes:
  - `__EFMigrationsHistory` table (EF Core tooling, implicit), and
  - `schema_version` (single-row table: `version TEXT`, `created_at_utc
    TEXT`, `updated_at_utc TEXT`) — **infrastructure metadata only**, not a
    domain concept. It exists so later features' migrations have a first
    *named* schema object and the API can report schema state.
- `MigrationTests`: create a temp SQLite file; run `dotnet ef database update`
  (or `context.Database.Migrate()`); assert `schema_version` row exists;
  run `dotnet ef database update --target 0` (or manual downgrade); assert
  `schema_version` is removed.
**Acceptance:** `MigrationTests` green (migration up succeeds on temp SQLite;
`schema_version` row present after upgrade; downgrade removes it); no domain
tables in the baseline migration.
**Verify:** `dotnet test tests/ICE.ProjectLifecycle.IntegrationTests --filter Migration`
**Satisfies:** A4

### N9 — Identity contract tests (adapter-agnostic)
**Files:** `tests/ICE.ProjectLifecycle.ContractTests/IdentityContractTests.cs`.
**Prereqs:** N4, N5.
**Do:** per `contracts.md` §4 + synthetic-data-strategy §5:
- `IdentityContractTests` takes an `IIdentityProvider` as a constructor
  parameter (adapter-agnostic).
- Test cases:
  - `Authenticate_KnownSubject_ReturnsSession`.
  - `ValidateSession_ExpiredToken_ReturnsAuthError` (SessionExpired semantic).
  - `ValidateSession_UnknownSubject_ReturnsAuthError` (SessionExpired semantic).
  - `BeginMfa_MfaPendingUser_ReturnsChallenge` followed by
    `Authenticate_WithMfaCode_ReturnsSession`.
  - `GetUser_UnknownSubject_ReturnsNotFound`.
  - `MapRoles_StableAcrossRuns`.
- **The test class is adapter-agnostic:** `IdentityContractTests` takes an
  `IIdentityProvider` via constructor injection and references only
  `Domain.Ports` + `Domain.Contracts`. A separate fixture
  (`SimIdentityContractFixture`) is the only place that constructs
  `SimIdentityProvider` and passes it in — so the contract is proven against
  the port, not the sim.
- **Correction #3:** the error mapping is exactly (typed results):
  `expiredToken → ValidateSessionAsync → Err<IdentityContext>(AdapterError.Auth)`
  (SessionExpired);
  `unknownSubject → ValidateSessionAsync → Err<IdentityContext>(AdapterError.Auth)`
  (SessionExpired);
  `mfaPendingUser → BeginMfaAsync → Ok<MfaChallenge>` → MFA verification
  flow (`AuthenticateAsync` with code → `Ok<AuthSession>`);
  `GetUserAsync` unknown → `Err<IdentityUser>(AdapterError.NotFound)`.
**Acceptance:** `IdentityContractTests` green against `SimIdentityProvider`;
`IdentityContractTests.cs` contains **no** `using`/type reference to
`Infrastructure.Simulators` (only the separate fixture wires the sim); the
three error-path assertions match synthetic-data-strategy §5 exactly.
**Verify:** `dotnet test tests/ICE.ProjectLifecycle.ContractTests`
**Satisfies:** A6

### N10 — Architecture (PP-016) test
**Files:** `tests/ICE.ProjectLifecycle.UnitTests/ArchitectureTests.cs`.
**Prereqs:** N4.
**Do:**
- Scan `src/ICE.ProjectLifecycle.Domain/` source files (parse `*.cs` files
  for `using` directives and `#r` references):
  - FAIL if any `using` references a vendor NuGet package (maintain an
    explicit denylist in the test: `Microsoft.EntityFrameworkCore`,
    `Microsoft.AspNetCore`, `Microsoft.Identity.Client`, `System.Net.Http`
    (beyond BCL), `Azure`, etc.).
  - FAIL if any `using` references `Infrastructure.Simulators` or
    `Application` or `Api`.
  - Also check the Domain project's `.csproj` for `<PackageReference>` —
    assert there are **zero** package references (only the BCL is allowed).
- Include a **deliberate-violation** test: temporarily add a
  `<PackageReference Include="Microsoft.AspNetCore.Http" />` to a copy of
  the Domain `.csproj`, run the scanner, assert failure, clean up.
- The test itself is deterministic and offline.
**Acceptance:** clean Domain project → passes; injected violation → scanner
fails; Domain project `.csproj` has zero `<PackageReference>` entries.
**Verify:** `dotnet test tests/ICE.ProjectLifecycle.UnitTests --filter Architecture`
**Satisfies:** A7

---

## Phase C — Frontend

### N11 — React + Vite + TypeScript shell + `brand/tokens.json`
**Files:** `src/ICE.ProjectLifecycle.Web/package.json`,
`src/ICE.ProjectLifecycle.Web/vite.config.ts`,
`src/ICE.ProjectLifecycle.Web/tsconfig.json`,
`src/ICE.ProjectLifecycle.Web/index.html`,
`src/ICE.ProjectLifecycle.Web/src/main.tsx`,
`src/ICE.ProjectLifecycle.Web/src/App.tsx`,
`src/ICE.ProjectLifecycle.Web/src/styles/tokens.ts`,
`brand/tokens.json`.
**Prereqs:** N1.
**Do:**
- `npm create vite@latest src/ICE.ProjectLifecycle.Web -- --template react-ts`
  (or equivalent manual setup).
- `brand/tokens.json`: placeholder brand tokens (colors, typography,
  spacing) — values are **obviously non-brand** (`#000000`-style, font
  `"placeholder"`). When DISC-016 delivers real brand assets, only
  `brand/tokens.json` changes — no code change (PP-057 AC-2 path).
- `tokens.ts`: imports `../../../brand/tokens.json`; exports typed token
  constants.
- `App.tsx`: branded shell page (header, footer, body placeholder) consuming
  `tokens.ts` values as CSS custom properties.
- `npm ci && npm run build` succeeds.
- `package-lock.json` committed (CI parity).
**Acceptance:** `npm ci && npm run build` exit 0; build output contains
token-derived CSS custom properties; `brand/tokens.json` exists with
placeholder values; no `using`/`import` of any backend type.
**Verify:** `cd src/ICE.ProjectLifecycle.Web && npm ci && npm run build`
**Satisfies:** A8

---

## Phase D — CI/CD

### N12 — CI pipeline
**Files:** `.github/workflows/ci.yml`.
**Prereqs:** N2, N3, N4, N5, N6, N7, N8, N9, N10, N11.
**Do:**
- `ci.yml`: runs on `push` (to `main` + `develop`) and `pull_request`.
- Steps:
  1. `actions/checkout@v4`
  2. `actions/setup-dotnet@v4` (dotnet 10.x)
  3. `dotnet restore ICE.ProjectLifecycle.sln`
  4. `dotnet build ICE.ProjectLifecycle.sln --no-restore -c Release`
  5. `dotnet test ICE.ProjectLifecycle.sln --no-build -c Release`
     (unit + contract + integration + architecture)
  6. `actions/setup-node@v4` (node 22)
  7. `cd src/ICE.ProjectLifecycle.Web && npm ci && npm run build`
  8. `actions/upload-artifact@v4`: backend publish output + web dist
- Add a `concurrency` group cancelling superseded PR runs. No secrets used.
**Acceptance:** workflow YAML valid (actionlint passes); a push to a branch
produces a green run with all 8 steps; a deliberately broken test (throwaway
commit on a scratch branch) turns the run red.
**Verify:** `npx actionlint .github/workflows/ci.yml` exit 0; green Actions
run on a pushed branch (log link in PR)
**Satisfies:** A9

### N13 — Release workflow + branch-protection runbook
**Files:** `.github/workflows/release.yml`,
`docs/ops/branch-protection-runbook.md`,
`docs/ops/release-runbook.md`.
**Prereqs:** N12.
**Do:**
- `release.yml`: triggered on `push` with tag `v*`; steps: checkout →
  setup-dotnet → restore → build → test → setup-node → npm build → package
  (backend publish + web dist) → DRY-RUN deploy job (**disabled/placeholder**
  until IT provisions targets, ADR-004 §6) → PROD deploy job behind an
  Actions *environment* approval (named approvers pending, ADR-004 §6).
- **`branch-protection-runbook.md`:** exact GitHub Settings → Branches policy
  for `main` (PRs only, ≥1 approving review, required status checks: the ci
  job; no force-push; no deletes) + who applies it (IT/PM — pending,
  ADR-004 §6) + how to verify.
- **`release-runbook.md`:** tag naming (`vX.Y.Z`), DRY-RUN rehearsal steps,
  PROD approval flow (named approvers pending), rollback (re-deploy last
  known-good tag), and the note that the migration console is a controlled
  operator job, never a pipeline step (ADR-004 §4.7).
- **Correction #4:** this is **the** runbook task (N13). No other task
  references a different runbook task number.
**Acceptance:** both workflow files pass actionlint; runbooks contain the
exact policy text from ADR-004 §4 and label every pending item with its
ADR §6 row; no secrets in the files.
**Verify:** `npx actionlint .github/workflows/release.yml` exit 0;
`grep -ri "secret" docs/ops/` shows no values
**Satisfies:** A10

---

## Phase E — Docs & closeout

### N14 — Developer documentation
**Files:** `README.md` (root).
**Prereqs:** N13.
**Do:** README with:
- Project purpose (1 paragraph + pointer to `docs/product/phase-1-prd.md`).
- Prerequisites: .NET 10 SDK, Node 20+.
- **Two-command dev loop:** `dotnet run --project src/ICE.ProjectLifecycle.Api`
  (backend) + `cd src/ICE.ProjectLifecycle.Web && npm install && npm run dev`
  (frontend).
- Config reference: table of provider keys from `contracts.md` §3 +
  dev/dryrun/prod defaults + prod-forbids-sim rule.
- Environment model: DEV/DRY-RUN/PROD per ADR-004 §4.
- Testing guide: `dotnet test` (unit + contract + integration + architecture);
  contract-tests explanation; architecture test.
- CI/CD overview + runbook links (`docs/ops/branch-protection-runbook.md`,
  `docs/ops/release-runbook.md`).
- **Pending IT confirmations** table (copied from specification §12.2 —
  must stay labeled *pending*).
- Traceability pointer: "This codebase implements Feature 001 per
  `specs/feature-001-platform-foundation-cicd/`".
**Acceptance:** a developer with zero prior context can clone, `dotnet
restore`, `dotnet build`, `npm ci`, boot both servers, and hit `/health`
using only the README; the pending-confirmations table is present and
labeled pending.
**Verify:** walkthrough (manual; no automated command)
**Satisfies:** A11

### N15 — Final verification pass
**Files:** none new (verification + fixes only).
**Prereqs:** N14.
**Do:**
- Run the full local gate end-to-end from a **clean state**: delete `bin/`,
  `obj/`, `src/ICE.ProjectLifecycle.Web/node_modules/`, re-run
  `dotnet restore`, `dotnet build`, `dotnet test`,
  `cd src/ICE.ProjectLifecycle.Web && npm ci && npm run build`.
- `git diff --check` clean.
- `grep -ri` scan for secret patterns in the tree (no hardcoded tokens,
  connection strings, or API keys).
- Verify `.gitignore` includes `sim-state/`, `bin/`, `obj/`, `*.db`,
  `node_modules/`, `dist/`.
- Verify `package-lock.json` is committed.
- Verify no `sim-state/` directory exists in the tree (it's ephemeral).
- Verify traceability: every AC in specification §6 is referenced by at
  least one task in this file; every task's "Satisfies" line references
  at least one AC.
**Acceptance:** full gate green from clean state; `git diff --check` clean;
no secrets; `.gitignore` complete; traceability verified in both directions.
**Verify:** `dotnet restore && dotnet build && dotnet test && (cd
src/ICE.ProjectLifecycle.Web && npm ci && npm run build) && git diff --check`
**Satisfies:** A12 + all (final gate)

---

## Task summary

| Task | Phase | AC | Prereqs |
|---|---|---|---|
| N1 Solution scaffold + repo hygiene | A | A12 (partial) | — |
| N2 App shell (`/health`, `/api/version`) | A | A1 | N1 |
| N3 Domain contracts (two result families, `AdapterError`) | A | A2 (partial) | N1 |
| N4 Port interfaces (8) + value objects | B | A2 | N3 |
| N5 Identity sim + fixtures + facade | B | A5 | N4 |
| N6 Config model + prod-forbids-sim | B | A3 | N1 |
| N7 DI wiring | B | A3 (DI), A1 | N2, N5, N6 |
| N8 EF Core + SQLite + migration | B | A4 | N6 |
| N9 Identity contract tests | B | A6 | N4, N5 |
| N10 Architecture (PP-016) test | B | A7 | N4 |
| N11 Frontend + tokens | C | A8 | N1 |
| N12 CI pipeline | D | A9 | N2–N11 |
| N13 Release workflow + runbooks | D | A10 | N12 |
| N14 Dev docs | E | A11 | N13 |
| N15 Verification pass | E | A12 + all | N14 |
