# ADR-004 — CI/CD Pipelines & Release Controls (D-029)

| Field | Value |
|---|---|
| Decision ID | D-029 |
| Status | **DECIDED FOR DEVELOPMENT** |
| Owner | IT |
| Release | R1 |
| Blocks | Feature 001 (Platform Foundation & CI/CD — CI/CD *is* Feature 001) |
| Related ADRs | ADR-001, ADR-005 |
| Requirements | PP-056 (CI/CD pipelines), PP-016 (architecture test in CI) |
| Date | 2026-09-26 |

## 1. Context

PP-056 requires automated build, test, and deployment pipelines across DEV and
PROD environments, with source control and release controls. The repository is
hosted on **GitHub** (constraint 5), so the natural CI/CD substrate is
**GitHub Actions**. Ralph performs most implementation and automated testing
(constraint 7), so the pipeline must be a reliable, deterministic, low-ceremony
gate that an AI agent can run and debug.

The architecture (architecture §11) already defines the pipeline shape and the
three environments:

```
repo → build (deps, typecheck, unit) → test (contract vs simulators,
      integration vs sim stack) → package → deploy DEV → [staging/dry-run]
      → release controls (D-029) → PROD
```

Environments: **DEV** (all simulators allowed), **DRY-RUN** (real identity +
real DB + sim external adapters + real doc store, for cutover rehearsal),
**PROD** (real adapters only). This decision fixes the tooling and the gate
model; it does not set environment-specific credential values (those are
IT-managed, D-033).

Constraints that matter:
- GitHub hosting (constraint 5) → GitHub Actions is the default substrate.
- Windows local dev (constraint 6) → the *pipeline* runs on Linux runners;
  local dev on Windows uses the same lockfiles, so CI parity holds.
- Ralph reliability (constraint 7) → minimal, declarative YAML; no bespoke
  tooling; tests are the gate.
- Avoid unnecessary infrastructure (constraint 11) → no separate CD engine,
  no artifact registry beyond what GitHub Actions provides, no staging
  infrastructure beyond what Phase 1 needs.

## 2. Options Considered

### 2.1 CI/CD substrate

**Option A — GitHub Actions (recommended).** Native to the GitHub host; free
for public/internal use; Linux runners; `dotnet` + `npm` available via
setup actions; no extra infrastructure.

**Option B — Azure DevOps Pipelines.** Microsoft-native; would be the choice
if ICE mandates Azure DevOps. Heavier; not required by any constraint.

**Option C — Local scripts only (no CI).** Rejected: PP-056 AC-1/AC-3 require
CI build+test; and Ralph needs an automated gate.

**Assessment:** GitHub Actions (Option A) is the boring, zero-extra-infra,
constraint-aligned choice. Azure DevOps is recorded as the IT-mandate
fallback.

### 2.2 Release-control model

**Option A — Branch protection + PR review + tag-based deploy + PROD approval gate (recommended).**
- `main` protected: PRs only, required review, required status checks.
- Feature work on short-lived branches; PR to `main`.
- Deploy to **DEV** on every push/PR to `main` (fast feedback, all simulators).
- Release = **git tag** (`vX.Y.Z`); a tag-triggered job packages and deploys to
  **DRY-RUN**, then to **PROD** behind a manual approval gate.
- PROD approval is a human (IT/PM) sign-off in the Actions "Environments"
  approval feature — this is the release control PP-056 AC-2 asks for.

**Option B — Deploy on every push to `main` (no tag, no approval).** Rejected:
no PROD gate; violates "release controls (tag/approval per D-029)".

**Option C — Environment-promotion with manual promotion buttons on every env.**
Heavier ceremony than Phase 1 needs (constraint 11); the DEV→DRY-RUN→PROD
tag+approval model is simpler and sufficient.

**Assessment:** Option A satisfies PP-056 (AC-1 build/test/deploy DEV+PROD,
AC-2 branch protection + tag + approval, AC-3 all automated tests in CI) with
the least infrastructure. It is also the model Ralph can verify: the CI gate
is the test suite; the gate to PROD is a human approval, not code.

### 2.3 What runs in CI (the gate)

Fixed by the architecture + synthetic strategy, and now confirmed:
1. **Restore + build:** `dotnet restore` → `dotnet build --configuration
   Release --no-restore` (backend solution; frontend compiles in step 6).
2. **Unit tests:** `dotnet test --configuration Release --no-build`
   (domain, pure logic).
3. **Contract tests:** the adapter-agnostic suite run against **every
   simulator** adapter (synthetic strategy §5). This is the PP-016 gate.
4. **Integration tests:** `dotnet test` against the full simulator stack
   (workflows WF-1…WF-6 on fixtures).
5. **Architecture test (PP-016):** fail the build if any `Domain` project
   file imports a vendor package or a `Simulators` type (synthetic strategy
   §5; architecture §3). This is the "no vendor imports in domain" gate.
6. **Frontend build:** `npm ci && npm run build` (ensures the SPA compiles).
7. **Packaging:** produce a deployable (backend artifact + frontend static
   bundle).

> **Amendment (2026-09-27):** the concrete CI commands were updated when
> **ADR-006** superseded **ADR-001** (Python 3.12 / FastAPI → C# / .NET 10
> platform). The gate *model* is unchanged: restore → build → test
> (unit + contract + integration + architecture) → frontend build →
> package. Only the tooling references are language-specific: the Python-era
> commands (`ruff`, `mypy`, `pytest`, `uv run`) were replaced with the
> `dotnet restore` / `dotnet build` / `dotnet test` steps above, aligned with
> ADR-006 §4.3 and Feature 001 (specification §10, tasks N12). The Python-era
> wording is retained in ADR-001 as history.

## 3. Evaluation Summary

| Criterion | Option A (GH Actions + tag/approval) | Option B (push-deploy) |
|---|---|---|
| PP-056 AC-1 (DEV+PROD build/test/deploy) | **Meets** | Meets |
| PP-056 AC-2 (branch protection, tag, approval) | **Meets** | **Fails** (no PROD gate) |
| PP-056 AC-3 (all automated tests in CI) | **Meets** | Meets |
| Ralph reliability (constraint 7) | **High** (tests = gate) | High |
| Avoid extra infra (constraint 11) | **Minimal** | Minimal |
| GitHub-hosted (constraint 5) | **Native** | Native |
| PROD is protected by a human gate | **Yes** | No |

## 4. Decision

**Select GitHub Actions as the CI/CD substrate, with branch protection +
PR review + tag-based release + a manual approval gate to PROD, running the
full test/contract/architecture suite as the CI gate.**

Concretely:
1. **Substrate:** GitHub Actions (Linux runners). Azure DevOps recorded as the
   IT-mandate fallback (see §6).
2. **Branch model:** `main` protected (PRs only, ≥1 review, required status
   checks). Feature work on short-lived branches → PR to `main`.
3. **DEV deploy:** automatic on push to `main` and on PR (all simulators;
   `identity.provider = sim`, etc.). Fast, deterministic, no creds.
4. **Release:** cut a **git tag** from `main`. The tag triggers package →
   deploy **DRY-RUN** (cutover rehearsal; real identity + real DB + sim
   external adapters + real doc store) → **PROD** behind a manual approval.
5. **PROD gate:** GitHub Actions *Environments* approval — a named IT/PM
   approver must approve before the PROD deploy job runs. This is the
   "release control" PP-056 AC-2 requires.
6. **CI gate = tests:** lint, typecheck, unit, contract (vs simulators),
   integration (vs sim stack), architecture (PP-016), frontend build. A red
   gate blocks merge to `main` and blocks any release tag from being promoted.
7. **Migration console (PP-044, S-88):** runs as a **controlled job**, not a
   pipeline step, in DRY-RUN and PROD (architecture §11). It is triggered by
   an operator with the migration inputs (DISC-012), not by CI.

## 5. Practical Consequences

1. **Feature 001** scaffolds: `.github/workflows/ci.yml` (build + test +
   architecture gate), a DEV deploy job, and the branch-protection
   recommendation (enforced via GitHub repo settings, documented in the
   workflow). The tag→DRY-RUN→PROD-approval path is defined but its real
   deploy targets are IT-provisioned (D-033).
2. **Ralph's loop:** implement → run `dotnet test` + architecture test
   locally → push → CI re-runs the same suite → merge green → (for a release)
   tag → DRY-RUN → human approves PROD. Ralph is responsible for keeping the CI gate
   green; the PROD gate is a human.
3. **Determinism:** because all external systems are simulators in CI
   (constraints 2, 4, 7), the gate is fully reproducible on any runner —
   including a Linux runner for a Windows-developed repo.
4. **No new infrastructure** is required to *build and test* (constraint 11):
   GitHub Actions runners + the app's own containers/artifacts. The DEV and
   PROD *deploy targets* are IT-provisioned App Service / containers (see
   ADR-005 / ADR-001), not part of this decision's infra burden.
5. **Sequencing:** unchanged. Feature 001 already owns CI/CD (PP-056); this
   decision removes D-029 from its blocker list.
6. **Traceability preserved:** PP-056 remains mapped to Feature 001 with D-029
   as the governing decision; now D-029 is decided (development default).

## 6. Client / IT Confirmation Items

| Item | Who | Status |
|---|---|---|
| GitHub Actions acceptable as the CI/CD substrate (vs Azure DevOps mandate) | IT | **Pending confirmation** — if Azure DevOps is mandated, the pipeline moves; the gate model (branch protection + tag + approval) is unchanged |
| DEV / DRY-RUN / PROD deploy targets (App Service Linux or container host) | IT | **Pending** — provisioned by IT; needed when the real deploy jobs run (D-033) |
| Named approver(s) for the PROD approval gate | IT/PM | **Pending** — required to configure the Actions environment approval |
| Branch-protection policy (reviewers, required checks) | IT/PM | **Pending** — set in GitHub repo settings |
| Artifact/registry location for the deployable | IT | **Pending** — default is GitHub Actions artifact / App Service zip; no extra registry required |
| Secrets management for deploy (connection strings, Entra client secret) | IT | **Pending** — stored in GitHub Actions secrets / Azure Key Vault; never in the repo (architecture §7) |

## 7. Development Default vs Production Approach

| Aspect | Development default | Production approach |
|---|---|---|
| CI substrate | GitHub Actions | GitHub Actions (or Azure DevOps if mandated) |
| Local loop (Windows) | `dotnet test` + `npm run build` | same lockfiles; CI is the authoritative gate |
| DEV deploy | Automatic on `main` (all simulators) | DEV target provisioned by IT; same pipeline |
| Release | git tag | git tag |
| DRY-RUN | cutover rehearsal env (real identity + real DB + sim adapters + real doc store) | same, IT-provisioned |
| PROD gate | manual approval (Actions environment) | manual approval by named IT/PM approver |
| Migration (PP-044) | controlled job (not a CI step) | controlled job in DRY-RUN/PROD (operator-triggered) |

> **Key distinction:** the development default means CI/DEV run entirely on
> simulators and need no production credentials. The production approach adds
> the IT-provisioned DRY-RUN and PROD deploy targets and the human approval
> gate — the *pipeline logic* (build → test → package → deploy) is identical.

## 8. Relationship to Existing Artifacts

- **`architecture.md` §11:** unchanged; the pipeline shape, the three
  environments, and the "release controls: branch protection, PR review,
  tag-based deploy, approval gate to PROD (D-029)" note are now confirmed as
  the decided model.
- **`decision-register.md` D-029:** status changes from `Open` to
  **`DECIDED FOR DEVELOPMENT`**.
- **`feature-sequence.md` Feature 001:** D-029 is no longer a blocker;
  Feature 001 now has a concrete CI/CD model to scaffold.
- **`release-plan.md` §5:** the "real-system verification after credentials
  (D-033)" note is consistent — the PROD/DRY-RUN deploy targets are
  IT-provisioned and are the point at which real adapters are verified.
