# Feature 001 — Research & Assumptions

> Short research note. Feature 001 is scaffolding; the heavy research
> (T-Sheets API, Dataverse surface, HR source) belongs to DISC-001/003/006
> and Features 008/010/007 respectively. This file records only the
> tooling choices and assumptions that 001 itself must make.
> Stack baseline: **C# / .NET 10 LTS** (ADR-006).

## 1. Tool versions (pinned in `Directory.Packages.props` / `global.json` / `web/package.json`)

| Tool | Version floor | Why |
|---|---|---|
| .NET SDK | 10.x (LTS) | ADR-006 §2; `global.json` pins 10.x for dev + CI parity (A1: local env currently has 9.0.313 only) |
| ASP.NET Core | net10.0 (framework reference) | ADR-006 §2 (DI, config, logging, health checks) |
| Entity Framework Core | ≥ 10.0 (incl. `Microsoft.EntityFrameworkCore.Sqlite`) | ADR-006 §2; **A3** — EF Core's built-in migration tooling (`dotnet ef`) is the migration companion |
| xUnit | current (incl. `xunit.runner.visualstudio`, `Microsoft.NET.Test.Sdk`) | ADR-006 §2 (unit, contract, integration) |
| React | 18 | ADR-006 §2 |
| Vite | 5.x or 6.x (latest stable) | ADR-006 §2 |
| TypeScript | ≥ 5.4 | ADR-006 §2 |
| Node (CI + local) | 20 or 22 | ADR-004 §2.1 (GH runners); local 22.17.1 ✓ |

> Exact pins are set by Ralph in N1 using the latest compatible releases at
> implementation time (central package management via `Directory.Packages.props`);
> the *floors* above are the contract.

## 2. GitHub Actions setup actions

| Action | Purpose |
|---|---|
| `actions/checkout@v4` | source |
| `actions/setup-dotnet@v4` (dotnet 10.x) | .NET SDK from `global.json` |
| `actions/setup-node@v4` (node 22) | frontend |
| `actions/upload-artifact@v4` | package step (ADR-004 §2.3 item 7) |
| `rhysd/actionlint` (optional, in N13) | workflow YAML validation |

Azure DevOps is the IT-mandate fallback (ADR-004 §6, pending). The gate model
(branch protection + tag + approval) is substrate-independent, so a move to
Azure DevOps is a workflow-porting exercise, not a redesign.

## 3. Assumptions register (normative for 001)

| ID | Assumption | Source / evidence | Risk if wrong |
|---|---|---|---|
| A1 | 001/007 simulator split (identity sim in 001, other 7 in 007) | specification §4 reconciling ADR-006 §4.1 with feature-sequence §5#6 | 007 spec must restate; no code rework if split holds |
| A2 | C# projects live **under `src/`**; .NET 10 SDK (local env has 9.0.313 — install 10.x or rely on CI) | ADR-006 §3 (repo structure); specification §13 A1 | Trivial rename/TFM change if reversed |
| A3 | EF Core `dotnet ef` for migrations | ADR-006 §2 (Entity Framework Core); no ADR names a tool | Swap tool; 001 baseline migration re-generated |
| A4 | Deploy jobs ship **disabled/placeholder** | ADR-004 §5.1 (targets IT-provisioned, D-033) | Enable jobs when IT provisions; logic unchanged |
| A5 | 001 has **no domain tables** | specification §5 non-goals; traceability row 001 | 002 adds first tables |
| A6 | Frontend CI gate = `npm run build` only | ADR-004 §2.3 item 6 | Add `npm run lint` later; script stub in 001 |
| A7 | `environment ∈ {dev, dryrun, prod}` | ADR-004 §4 (normalizes architecture §7's `dev/test/prod`) | Config key rename only |
| A8 | No e2e framework in 001 | ADR-006 §2 (xUnit only); no ADR names Playwright/Cypress | 002+ may add; not a 001 gap |
| A9 | `audit` port defined in 001, **no audit fixtures, no audit table** | synthetic §3 note (sim audit = test harness only; prod always DB) | 006 owns audit table; 007 owns SimAudit if needed |
| A10 | Secrets via env vars in 001 (no Key Vault code) | ADR-004 §6 (secrets mgmt pending); architecture §7 | Key Vault integration is IT/002+ work |

## 4. Open questions 001 must NOT answer

These belong to later features or to IT; 001 carries them as open (see
specification §12.2):

- Real adapter choices per port (007–010, gated D-033).
- RBAC matrix (D-009), session timeout (D-027), MFA method (D-028) — 002.
- Document store (D-002): **DECIDED FOR DEVELOPMENT** (ADR-002) — Azure Blob
  behind the `DocumentStore` port. 001 defines the port + config key only; the
  real adapter is Feature 007. The following ADR-002 §6 items remain **Pending**
  and are carried in specification §12.2:
  1. Azure Blob acceptable for production (vs SharePoint mandate) — IT/OPS.
  2. Blob container, account, and Entra role/scope model — IT.
  3. Retention policy for stored documents — IT.
  4. Whether existing SharePoint library metadata must be reused or mirrored
     — OPS.
  5. Large-file limit and supported MIME types for billing packets (PDF) — IT.
- HR source (DISC-006) — 007 real adapter.
- AR mechanism (D-020, DISC-013) — 007 real adapter.
- Performance targets (D-026 / DISC-015) — 012 + R1 exit.
- Brand assets (DISC-016) — 001 ships placeholder only.

## 5. References

- `docs/architecture/decisions/ADR-006-platform-stack.md` §2 (stack table),
  §3 (repo structure), §7 (pending IT confirmations).
- `docs/architecture/decisions/ADR-004-ci-cd-and-release-controls.md` §2.1
  (GitHub Actions), §2.3 (CI gate).
- `docs/data/synthetic-data-strategy.md` §5 (shared error contract), §7
  (switching).
- `docs/architecture/architecture.md` §7 (config schema).
