# Decision Register — ICE Services Project Lifecycle Platform

| Field | Value |
|---|---|
| Status | BMAD draft v1.0 (2026-09-26) |
| Source | `docs/source/phase-1-requirements-register-v2.2.md` v2.2 (authoritative) |
| Purpose | Record unresolved requirements, technical unknowns, and open decisions. **Nothing is silently resolved.** |
| Discovery | Technical unknowns also tracked in `docs/discovery/discovery-backlog.md` (DISC-xxx) |

**Statuses:** `Open` (no decision), `Proposed` (BMAD recommendation, needs owner sign-off),
`Decided` (owner confirmed), `Decided for development` (development default fixed and
ready to build against; production approach and client/IT confirmation items
documented in the linked ADR — see §10), `Blocked` (waiting on external
party), `Deferred` (to R2/R3).

**Owner roles:** PM (project), OPA (Ops Accounting), NSA (NS Admin), IT (ICE IT/Security),
FIN (Finance/FP&A), EXE (Executive), VENDOR (T-Sheets/procurement/HR vendor).

> Decisions required **before Feature 001** are flagged 🔺. Everything else can wait
> for its owning feature. The full "decisions required before 001" list is in the
> final summary.

## 1. Platform & Infrastructure

| ID | Decision | Status | Owner | Rel | Blocks | Notes / Recommendation |
|---|---|---|---|---|---|---|
| D-001 | Platform stack (language, framework, hosting) | **Decided for development** | PM/IT | R1 | 🔺 001 | **DECIDED FOR DEVELOPMENT (ADR-006).** C# / .NET 10 LTS / ASP.NET Core + React (Vite) + Entity Framework Core; SQLite (dev/CI), Azure SQL / SQL Server (prod target). **Supersedes ADR-001** (Python, marked Superseded — retained as history); the baseline was corrected before implementation because ICE Services is a Microsoft-aligned shop. Rationale + IT-confirmation items in `decisions/ADR-006-platform-stack.md`. |
| D-002 | Document storage backend | **Decided for development** | IT | R1 | 🔺 005 | **DECIDED FOR DEVELOPMENT (ADR-002).** Azure Blob + Entra-scoped/SAS URLs behind `DocumentStore` (prod); `SimDocumentStore` (dev/CI). SharePoint remains a valid same-port alternate. Rationale + IT-confirmation in `decisions/ADR-002-document-storage.md`. |
| D-029 | Release controls & deployment gates | **Decided for development** | IT | R1 | 001 | **DECIDED FOR DEVELOPMENT (ADR-004).** GitHub Actions; branch protection + PR review + tag-based release + manual PROD approval gate; CI gate = lint/typecheck/unit/contract/integration/architecture(PP-016)+frontend build. Rationale + IT-confirmation in `decisions/ADR-004-ci-cd-and-release-controls.md`. |
| D-030 | Audit & security log retention + export format | Open | IT | R1 | 002 | ICE policy retention period; CSV export; SIEM requirement (register silent on SIEM). |
| D-031 | Staffing & timeline | **Decided for development** | PM | R1 | — | **DECIDED FOR DEVELOPMENT (ADR-005).** Two-track, relative, input-gated delivery: Ralph (AI) primary implementer; client/ICE input track is the critical path (DISC-001→008, DISC-012→015→R1 exit). Relative timeline — calendar dates attach only when ICE commits input dates. Not a code blocker. See `decisions/ADR-005-delivery-model.md`. |
| D-033 | Real-system verification & credentials | Open | IT/VENDOR | R1 | 007–010 | When do real sandbox creds (T-Sheets, on-prem SQL Server, HR, AR location) arrive? Gates real-adapter verification. No full demo env expected before build start — development proceeds on simulators (synthetic-data-strategy.md) and real adapters are verified when creds arrive. |
| D-035 | On-prem data connectivity (SQL + Access) | **Open 🔴** | IT | R1 | 008, 010, 015 | How does the app reach **on-prem** data with no public API — the Procurement **SQL Server** (PO read) and the **Access billing DB** (rates/billing, D-017)? Decide the read path: on-prem data gateway (Azure Data Gateway / self-hosted integration runtime), app co-located on-prem, linked server / read replica, or scheduled extract into the primary DB. Also the SQL/Access auth model. **Not solvable by the sim fallback** — gates the real `ProcurementService` adapter and the rates read. Record choice before building real adapters. |
| D-034 | Message broker for R2 GL sync | Deferred | PM/IT | R2 | — | Revisit if PP-054 introduces async volume. Not in R1. |

## 2. Identity, Security & Users

| ID | Decision | Status | Owner | Rel | Blocks | Notes / Recommendation |
|---|---|---|---|---|---|---|
| D-007 | Authentication/role-assignment mechanism | **Decided for development** | IT | R1 | 🔺 002 | **DECIDED FOR DEVELOPMENT (ADR-003).** Entra ID OIDC (prod authN) + in-app role table (role assignment, subject from Entra) + Entra Conditional Access for MFA, all behind `IdentityProvider`; `SimIdentityProvider` for dev/CI. Mechanism fixed; D-009/D-027/D-028 values remain open. See `decisions/ADR-003-identity-and-access.md`. |
| D-009 | RBAC permission matrix + edit/post-billing rules | Open | PM/OPA | R1 | 002, 006 | Which role may do which state transition (PP-017), edit budget (PP-008), true-up (PP-026). Also post-billing edit rules. |
| D-027 | Session timeout duration | Open | IT | R1 | 002 | ICE IT guidance (PP-049). |
| D-028 | MFA method per ICE policy | Open | IT | R1 | 002 | Recommend Entra Conditional Access (PP-050). |
| D-022 | Customer creation validation role | Open | NSA | R1 | 003 | Who validates a new customer before activation (PP-033). |

## 3. SOW, Budget & Configuration

| ID | Decision | Status | Owner | Rel | Blocks | Notes / Recommendation |
|---|---|---|---|---|---|---|
| D-003 | Pre-job obligations checklist definition | Open | PM/NSA | R1 | 004, 006 | Register: undefined, varies by project type (PP-007). Recommend: per-project-type checklist, admin-maintained, waivable with reason. See DISC-002. |
| D-004 | SOW number format + starting sequence | Open | NSA | R1 | 004, 015 | Confirm starting value from Access history (PP-001) and format. See DISC-012. |
| D-005 | Project type configuration content | Proposed | PM/NSA | R1 | 003, 004, 012 | Seed config for Hard Dollar/T&M/OMR/Camp: billing model, OT rule set, packet format, required budget categories, required closeout docs (PP-037, PP-068). Not hard-coded (PP-004). |
| D-006 | Requester identity model | Open | PM | R1 | 004 | Is Requester an internal user or external party (PP-005)? Affects IdentityProvider use. |
| D-013 | Full state machine transition matrix + hold-resume semantics | Proposed | PM/OPA | R1 | 006, 013 | Register fixes the 8 statuses (PP-017) but not all transitions. Proposed matrix in workflow-map §1. Confirm hold-release state and post-billing edit interplay. |
| D-014 | Variance threshold values (flags) | Open | OPA | R1 | 011, 012 | What % / amount triggers amber/red (PP-018, PP-026). |
| D-017 | Rate source for billing charges | **Decided for development** | OPA | R1 | 009, 012 | **RATES COME FROM THE ACCESS BILLING DB / RATES MASTER, NOT T-SHEETS** (confirmed 2026-09-30: T-Sheets = hours only). Billing rates per trade/discipline → role-level incl. OT rates read from the Access DB (D-035 connectivity). Budget rates are a separate thing (budgeted estimate); actual labor charge calc (PP-023, PP-026) uses the Access rate source. DISC-007 reconciles the labor-category master; DISC-012 captures the rate schema. |
| D-019 | Relevance filter defaults (packet) | Open | OPA | R1 | 012 | Which document types/data sections are included per type/client (PP-028). |
| D-021 | Notification recipients & channel for R1 minimum events | Proposed | OPA/PM | R1 | 002, 013 | R1 minimum (PP-069, PP-052): in-app task + email to configured Ops Accounting. Recommend that default. |

## 4. Timekeeping (T-Sheets)

| ID | Decision | Status | Owner | Rel | Blocks | Notes / Recommendation |
|---|---|---|---|---|---|---|
| D-010 | T-Sheets capability outcome + fallback | Blocked | VENDOR/PM | R1 | 🔺 008 | Depends on DISC-001 spike (PP-047 🔴). If sub-code creation/employee-restriction unsupported → manual sub-code + mapping table fallback. Decide after spike. |
| D-011 | Late-cost / reopen handling at close | Open | OPA/PM | R1 | 008, 013 | What happens when a closed SOW needs more time (PP-013, PP-030). Reopen path in state machine. |

## 5. Overtime

| ID | Decision | Status | Owner | Rel | Blocks | Notes / Recommendation |
|---|---|---|---|---|---|---|
| D-012 | OT rule model + rules source | Open | OPA/FIN | R2 | 018 | Need structured rules list from client (PP-015, PP-036). R1 keeps only the minimum rule for PP-023 (default >40h/week pending DISC-005). |

## 6. Procurement & Costs

| ID | Decision | Status | Owner | Rel | Blocks | Notes / Recommendation |
|---|---|---|---|---|---|---|
| D-018 | PO↔SOW matching convention | Open | PM/IT | R1 | 010, 012 | Procurement backend is **on-prem SQL Server** (not Dataverse) — matching field confirmed against the SQL PO table (PP-024, PP-038). See DISC-003; connectivity model in D-035. |
| D-016 | OM code billing treatment | Open | OPA | R1 | 011 | How OM codes are billed (PP-019). Recommend: not customer-billed; time/cost allocation only. |

## 7. Billing, Closeout & AR

| ID | Decision | Status | Owner | Rel | Blocks | Notes / Recommendation |
|---|---|---|---|---|---|---|
| D-020 | AR handoff mechanism | Proposed | OPA/IT | R1 | 013, 014 | Register: portal / shared folder / email / S2S (PP-029, PP-040). Recommend R1 default: shared-folder deposit of packet bundle; portal/email deferred. See DISC-013. |
| D-025 | Migration validation + cutover sign-off criteria | Open | OPA/NSA | R1 | 015 | What "validated" means for PP-044 🔴 (counts, sample SOWs, exception tolerance). |
| D-015 | Budget-to-actual refresh cadence | Proposed | OPA | R1 | 011 | Recommend on-demand + daily off-peak refresh. |

## 8. Reporting, GL, Accrual (R2/R3)

| ID | Decision | Status | Owner | Rel | Blocks | Notes / Recommendation |
|---|---|---|---|---|---|---|
| D-008 | Change order design (record vs budget edit) | Deferred | PM/OPA | R2 | 017 | Register: undefined (PP-020). Recommend first-class CO record with budget delta + approval. See DISC-009. |
| D-024 | GL interface mechanism | Deferred | FIN/IT | R2 | 021 | Register: "TBC" major risk (PP-054). Basic write-back first; GL posting out of scope. See DISC-014. |
| D-032 | Accrual tolerance / formula | Deferred | OPA/FIN | R3 | 022 | Shannon's accrual workbook needed (PP-032, PP-072). See DISC-011. |

## 9. Performance & Data

| ID | Decision | Status | Owner | Rel | Blocks | Notes / Recommendation |
|---|---|---|---|---|---|---|
| D-026 | User count & SLA / perf targets | Open | PM/OPA | R1 | 016 | Register: need confirm user count and SLA (PP-045). Interim target in architecture §10. See DISC-015. |
| D-023 | Department source of truth | Open | NSA/IT | R1 | 003 | Local list vs Paylocity vs other (PP-034). See DISC-006. |

## 10. Decision Status Summary

### 🔺 Required before Feature 001 (Platform Foundation & CI/CD) — **ALL DECIDED FOR DEVELOPMENT (2026-09-26)**
- **D-001** — platform stack → **DECIDED FOR DEVELOPMENT** ([ADR-006](decisions/ADR-006-platform-stack.md)): C# / .NET 10 LTS / ASP.NET Core + React (Vite) + Entity Framework Core; SQLite (dev/CI) / Azure SQL / SQL Server (prod target). **Supersedes [ADR-001](decisions/ADR-001-platform-stack.md)** (Python, marked Superseded — retained as history).
- **D-002** — document storage → **DECIDED FOR DEVELOPMENT** ([ADR-002](decisions/ADR-002-document-storage.md)): Azure Blob behind `DocumentStore` (prod); `SimDocumentStore` (dev/CI)
- **D-007** — identity mechanism → **DECIDED FOR DEVELOPMENT** ([ADR-003](decisions/ADR-003-identity-and-access.md)): Entra OIDC + in-app role table + Entra CA (MFA); `SimIdentityProvider` (dev/CI)
- **D-029** — release controls → **DECIDED FOR DEVELOPMENT** ([ADR-004](decisions/ADR-004-ci-cd-and-release-controls.md)): GitHub Actions; branch protection + tag + PROD approval gate
- **D-031** — staffing/timeline → **DECIDED FOR DEVELOPMENT** ([ADR-005](decisions/ADR-005-delivery-model.md)): two-track, relative, input-gated delivery; not a code blocker

> **Result:** every decision that previously blocked Feature 001 is now decided
> for development (with development defaults vs production approach made
> explicit in each ADR). Feature 001 scaffolding may proceed to Spec Kit.
> Client/IT confirmation items are captured in each ADR's §6 and remain open
> (not silently resolved); they gate real-adapter verification (D-033) and
> calendar commitment (D-031), not the development path.

> Note: D-010 (T-Sheets) and D-018 (PO matching) do **not** block Feature 001, but the
> DISC-001 spike should be **started immediately** (it is the critical path and gates
> Feature 008). See "Decisions that can wait" below.

### Decisions that can wait until their owning feature
| Feature | Decisions that can wait |
|---|---|
| 002 Identity & Access | D-009 (RBAC matrix detail), D-027, D-028, D-022 |
| 003 Master Data | D-003, D-004, D-005, D-006, D-016, D-023 |
| 006 Lifecycle | D-013, D-014 |
| 008 T-Sheets | D-010, D-011 (after DISC-001 spike), D-035 (on-prem connectivity for rates read) |
| 009/012 Labor & Packet | D-017, D-019, D-012 (R1 min only) |
| 010/013 Procurement & Closeout | D-018, D-020, D-021, D-025, D-035 (on-prem SQL connectivity) |
| 016 Reporting | D-014, D-015, D-026 |
| 017+ (R2) | D-008, D-012, D-024 |
| 022 (R3) | D-032 |

### Decisions that are **Blocked** on external parties
| ID | Waiting on | Impact |
|---|---|---|
| D-010 | DISC-001 spike / T-Sheets vendor | Feature 008 design |
| D-035 | ICE IT — on-prem SQL + Access read path | Real `ProcurementService` adapter + rates read |
| D-033 | IT/vendor sandbox creds | Real-adapter verification (007–010) |

## 11. Explicitly NOT decided (carried as open, per "do not silently resolve")

The following register ⚠️/🔴 items are **intentionally left open** and tracked here
or in discovery — none have been assumed away:

PP-006 (storage), PP-007 (pre-job checklist), PP-011 (sub-code API), PP-013
(reopen), PP-015 (OT rules), PP-017 (transitions), PP-018 (cadence/threshold),
PP-019 (OM treatment), PP-020 (CO design), PP-021 (progress billing), PP-024
(matching), PP-025 (equipment source), PP-026 (true-up), PP-027 (layouts),
PP-028 (filter), PP-029 (AR mechanism), PP-030 (hold), PP-032 (accrual), PP-034
(dept source), PP-036 (OT rules), PP-038 (API/matching), PP-040 (AR), PP-041
(role list), PP-044 (migration 🔴), PP-045 (SLA), PP-046 (storage), PP-047
(spike 🔴), PP-051 (mgmt mechanism), PP-054 (GL).
