# Feature 001 — Data Model

> Foundation only. Per specification §5 (non-goals) and assumption A5,
> **Feature 001 owns no domain tables** — SOW, Budget, roles, audit,
> documents, and every other domain table arrive with their owning features
> (002+). This file defines the *data foundation* those tables will build on.
> Stack baseline: **C# / .NET 10 LTS** (ADR-006).

## 1. ORM & database (ADR-006 §2)

- **ORM:** **Entity Framework Core** — first-party, works identically against
  SQLite (dev/demo) and SQL Server / Azure SQL (prod target); provider swap is
  a connection-string + provider change (ADR-006 §2, §4.4).
- **Dev/CI database:** **SQLite** (connection string from config, default
  `Data Source=./dev.db`). Zero-config, no server (ADR-006 §2).
- **Production target:** **Azure SQL / SQL Server** — a **connection-string +
  EF Core provider change only**. Acceptance of a specific production database
  (Azure SQL vs SQL Server vs another) is **pending IT confirmation**
  (ADR-006 §7) and is *not* a confirmed requirement of this feature.
- **DbContext & scope policy:** a single `AppDbContext` in
  `src/ICE.ProjectLifecycle.Infrastructure/Persistence/AppDbContext.cs`.
  One context per request / unit of work, scoped via ASP.NET Core DI
  (registered in N7). The audit-same-UoW rule of architecture §1 applies to
  later features' writes.
- **No ORM types leak into ports** (architecture §3 layering): the Domain
  project references **no EF Core / NuGet package** (enforced by N10); port
  value objects in `src/ICE.ProjectLifecycle.Domain/ValueObjects/` are plain
  C# records.

## 2. Baseline migration (assumption A3 — EF Core `dotnet ef`)

- `dotnet ef migrations add 0001_baseline` generates the baseline into
  `src/ICE.ProjectLifecycle.Infrastructure/Persistence/Migrations/` (N8).
- **Baseline migration `0001_baseline`** contains *no domain tables*. It
  establishes:
  - the `__EFMigrationsHistory` table (EF Core tooling, implicit), and
  - `schema_version` (single-row table: `version` text, `created_at_utc`,
    `updated_at_utc`) — **infrastructure metadata only**, not a domain
    concept. It exists so later features' migrations have a first *named*
    schema object and the API can report schema state at `/api/version`.
- Verification: `dotnet ef database update` (or `context.Database.Migrate()`)
  then a downgrade to the empty target both succeed against a **temporary**
  SQLite file (specification A4; N8 `MigrationTests`).

## 3. Conventions for later features (normative, set now)

Set in 001 so 002+ do not re-debate them (architecture §4 "Conventions"):

| Convention | Rule |
|---|---|
| Money | integer minor units (cents) + currency code; **never float** (architecture §4) |
| Timestamps | UTC, ISO-8601; column suffix `_utc` |
| Identifiers | `Id` surrogate (int or `Guid` — 002 decides per table) + natural keys where the PRD names them (e.g. `sow_number`) |
| Audit columns | `created_at_utc`, `updated_at_utc` on mutable domain tables (002+) |
| Audit trail | separate append-only audit table owned by 002/006 (PP-042/PP-070) — **not** in 001 |
| FK discipline | domain tables reference by natural key where the PRD requires (SOW↔budget↔packet); EF-level enforcement in 002+ |
| SQLite↔SQL Server parity | only ANSI-safe types in the 001 baseline; later features avoid SQLite-only or SQL Server-only constructs without an adapter note |

## 4. What is explicitly NOT in this data model

| Table / concept | Owner |
|---|---|
| `roles` / `user_roles` (in-app role table, ADR-003 §4.2) | Feature 002 |
| Audit table (PP-042) + security events (PP-070) | Feature 002/006 |
| SOW, Budget, BillingPacket, BillingEvent, CloseoutRegister, ProjectTypeConfig (architecture §4) | Features 003–012 |
| Documents metadata (PP-046) | Feature 005 |
| Workflow task queue table (PP-043 min) | Feature 013 |
| Migration/ledger tables (PP-044) | Feature 015 |

## 5. Verification

- `dotnet test tests/ICE.ProjectLifecycle.IntegrationTests --filter Migration`
  — `AppDbContext` builds from config; `dotnet ef database update`
  (upgrade) then a downgrade to the empty target on a **temp** SQLite file
  both succeed; `schema_version` row present after upgrade, removed on
  downgrade (N8).
- `dotnet build` clean on `src/ICE.ProjectLifecycle.Infrastructure/`.
- The Domain project has **zero** EF Core / NuGet package references
  (enforced by N10, the PP-016 architecture test).
