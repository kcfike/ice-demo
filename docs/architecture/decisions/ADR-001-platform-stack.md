# ADR-001 — Platform Stack (D-001)

| Field | Value |
|---|---|
| Decision ID | D-001 |
| Status | **SUPERSEDED** by [ADR-006](ADR-006-platform-stack.md) (2026-09-26) |
| Owner | PM/IT |
| Release | R1 |
| Blocks | Feature 001 (Platform Foundation & CI/CD) |
| Related ADRs | ADR-002, ADR-003, ADR-004, ADR-005 |
| Date | 2026-09-26 |

> **⚠ SUPERSEDED — do not use as the implementation baseline.**
> This ADR selected a **Python 3.12+ / FastAPI** stack. Before any
> implementation existed, the expected client environment (ICE Services is a
> Microsoft technology shop) was confirmed, and the decision was corrected to
> a **C# / .NET** baseline. See [ADR-006 — Platform Stack (C# / .NET)](ADR-006-platform-stack.md),
> which **supersedes** this ADR. The Python decision below is retained as
> decision history (what was decided, and why); it is **no longer** the
> development/demo or production baseline. D-001 in the decision register now
> points to ADR-006.

## 1. Context

The platform stack decision determines the language, framework, database,
frontend, and hosting model for the entire Phase 1 application. Every other
decision (identity, document storage, CI/CD) builds on this choice.

The architecture document (`architecture.md`) is deliberately **language-neutral**:
ports, simulators, config switches, and the domain model are all defined
independently of the implementation language. This is by design (PP-016) and
means the stack choice does not constrain the architectural decisions.

## 2. Options Considered

### Option A — .NET 8 (C#) / ASP.NET Core + React, SQL Server

The original register recommendation. Microsoft-first stack.

| Constraint | Assessment |
|---|---|
| 1. Demonstration/dev | Overkill for a demo; MSBuild + nuget + IIS add friction |
| 6. Windows local dev | Works, but requires .NET SDK + Visual Studio/VS Code + SQL Server Express |
| 7. Ralph reliability | C# is a valid target, but the repo is in a Python environment (`Documents/python/ice_demo`); Python is the path of least resistance |
| 10. Simple stack | More moving parts (MSBuild, nuget, IIS, SQL Server) |
| 11. Avoid unnecessary infra | SQL Server adds a dependency for no functional gain over SQLite/PostgreSQL at Phase 1 scale |

### Option B — Python 3.12+ / FastAPI + React (Vite), SQLite (dev) / PostgreSQL (prod target)

Boring, well-supported Python backend; standard React SPA frontend.

| Constraint | Assessment |
|---|---|
| 1. Demonstration/dev | Minimal setup; `pip install` and done |
| 6. Windows local dev | Python 3.13 confirmed working; pip/uv; no build server; no IIS |
| 7. Ralph reliability | Python is the most reliable implementation target for an AI coding agent; largest training corpus; `pytest` is the most well-understood test framework |
| 8. Entra ID production | `python-msal` / `authlib` provide OIDC support; well-documented |
| 9. Interfaces behind ports | Ports pattern is language-agnostic; Python `abc.ABC` + `Protocol` maps 1:1 |
| 10. Simple stack | FastAPI + SQLAlchemy + React + SQLite = minimal moving parts |
| 11. Avoid unnecessary infra | No IIS, no MSBuild, no nuget, no SQL Server; SQLite is zero-config |

### Option C — Node.js / Next.js

Full-stack TypeScript. Strong for rich UIs.

| Constraint | Assessment |
|---|---|
| 7. Ralph reliability | TypeScript is a valid target, but the repo is in a Python environment |
| 10. Simple stack | Node.js runtime + build tooling adds a second language |
| 8. Entra ID | `@azure/msal-node` works; equivalent to Python |

### Option D — Rust / Go

Not evaluated further: over-engineered for Phase 1, less reliable for Ralph,
no existing project context.

## 3. Evaluation Summary

| Criterion | Option A (.NET) | Option B (Python) | Option C (Node) |
|---|---|---|---|
| Matches repo environment (Python) | ✗ | **✓** | ✗ |
| Ralph implementation reliability | Moderate | **High** | Moderate |
| Windows local dev friction | High (SDK + IIS + SQL Server) | **Low** (pip) | Moderate |
| Simplicity (fewest moving parts) | Low | **High** | Moderate |
| Entra ID support | Native | Good (python-msal) | Good |
| Production portability | Azure App Service (Linux) | **Any** (container) | Any |
| Test ecosystem | xUnit | **pytest** (best for AI agent) | Jest |

## 4. Decision

**Select Option B — Python 3.12+ / FastAPI + React (Vite) + SQLAlchemy.**

| Layer | Choice | Rationale |
|---|---|---|
| Language | Python 3.12+ | Matches repo; Ralph reliability; no build step |
| Backend framework | FastAPI | Boring, async, Pydantic validation, OpenAPI auto-gen; minimal boilerplate |
| ORM / DB access | SQLAlchemy 2.0 | Works identically against SQLite (dev/CI) and PostgreSQL (prod target); no vendor lock-in |
| Database (dev/CI) | SQLite | Zero-config; no server; fast; perfect for Phase 1 scale |
| Database (prod target) | PostgreSQL | Boring, well-supported, no Windows-specific hosting; swap via SQLAlchemy |
| Frontend | React 18 + Vite + TypeScript | Rich SPA for the 7-tab reconciliation workspace, work queues, master data CRUD; standard tooling |
| Frontend build | Vite | Fast dev server; no webpack config; `npm run build` is the only build step |
| Auth (dev) | SimIdentityProvider (simulator) | No Entra tenant needed for development; per D-007 / ADR-003 |
| Auth (prod) | Entra ID OIDC via `python-msal` | Per D-007 / ADR-003; production architecture requirement |
| Process management (dev) | `uvicorn` (dev server) + `npm run dev` (Vite) | Two commands; no IIS, no Docker required for local dev |
| Package manager (Python) | `uv` (preferred) or `pip` | Fast; `uv lock` for reproducible installs; Windows-native |
| Test framework | `pytest` + `pytest-asyncio` | Ralph's most reliable test target; standard fixtures; `conftest.py` |
| Linting / type checking | `ruff` + `mypy` | Fast; single tool; optional but recommended |
| CI/CD | GitHub Actions | Per ADR-004; GitHub-hosted repo (constraint 5) |

## 5. Practical Consequences

1. **No application code is written in this run** (constraint 12). This ADR
   defines the target stack for Feature 001 (Platform Foundation & CI/CD)
   scaffolding by Ralph.
2. **Feature 001 scaffolds:** FastAPI app shell, SQLAlchemy models, SQLite
   connection, port interfaces (Python `Protocol`/`abc.ABC`), simulator
   package, `fixtures/` JSON, `pytest` harness, GitHub Actions workflow,
   React + Vite frontend, `brand/tokens.json` placeholder.
3. **Windows dev loop:** `uv sync && uv run uvicorn app.main:app --reload`
   (backend) + `npm install && npm run dev` (frontend). No Docker, no IIS,
   no SQL Server.
4. **CI loop:** GitHub Actions runs `ruff check`, `mypy`, `pytest` (unit +
   contract tests against simulators), `npm run build` (frontend), and
   deploys to a DEV App Service (or container) per ADR-004.
5. **Production swap:** SQLite → PostgreSQL is a connection-string change
   (SQLAlchemy handles it). The application code does not change.
6. **Ralph's implementation surface** is maximally familiar: Python + pytest
   + FastAPI is the most reliable AI-agent implementation target.

## 6. Client / IT Confirmation Items

| Item | Who | Status |
|---|---|---|
| Python 3.12+ acceptable for production (vs .NET mandate) | IT | **Pending confirmation** — if ICE mandates .NET, this decision reverts to Option A; the architecture (ports, simulators, domain model) is unchanged |
| PostgreSQL acceptable as production DB (vs SQL Server mandate) | IT | **Pending confirmation** — if SQL Server is mandated, SQLAlchemy supports it with no code change |
| Azure App Service (Linux) or container host acceptable for production | IT | **Pending confirmation** — see ADR-004 |
| `python-msal` acceptable for Entra ID integration (vs .NET MSAL) | IT | **Pending confirmation** — functionally equivalent OIDC |

## 7. Development Default vs Production Approach

| Aspect | Development default | Production approach |
|---|---|---|
| Language/framework | Python 3.12+ / FastAPI | Python 3.12+ / FastAPI (unchanged) |
| Database | SQLite (zero-config) | PostgreSQL (or SQL Server if mandated by IT) |
| Identity | SimIdentityProvider (fixture users) | Entra ID OIDC via python-msal |
| Frontend | React + Vite (dev server) | React + Vite (production build) |
| Hosting | Local `uvicorn` + Vite dev server | Azure App Service (Linux) or container host |
| Package manager | `uv` or `pip` | `uv` (locked via `uv.lock`) |
| Test runner | `pytest` locally + GitHub Actions | `pytest` in GitHub Actions (CI gate) |
| Simulators | All 8 ports on simulators | Simulators **forbidden** (startup validation, per architecture §7) |

> **Key distinction:** the development default runs entirely on simulators +
> SQLite. The production approach uses real adapters (Entra ID, T-Sheets,
> Dataverse, etc.) + PostgreSQL. The application code is identical in both;
> only the config switch and database connection string change.

## 8. Relationship to Existing Artifacts

- **`architecture.md` §1–§13:** unchanged. The architecture is language-neutral;
  all ports, domain model, simulator strategy, and config switches remain valid.
- **`decision-register.md` D-001:** status changes from `Proposed` to
  **`DECIDED FOR DEVELOPMENT`**. The .NET recommendation is superseded by this
  ADR; the rationale is recorded here.
- **`release-plan.md` entry criteria §2.1:** D-001 is now decided; Feature 001
  scaffolding can proceed.
- **`feature-sequence.md` Feature 001:** D-001 is no longer a blocker.
- **`synthetic-data-strategy.md`:** unchanged. The simulator/fixture/contract-test
  model is language-agnostic; Python implementations will follow the same
  fixture formats and contract assertions.
