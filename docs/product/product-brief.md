# Product Brief — ICE Services Project Lifecycle Platform

| Field | Value |
|---|---|
| Project | ICE Services – Project Lifecycle Platform |
| Phase | Phase 1 — Project Setup, Tracking & Billing |
| Status | BMAD product definition draft |
| Authoritative source | `docs/source/phase-1-requirements-register-v2.2.md` (v2.2, 62 PP requirements, 8 epics) |
| Date | 2026-09-26 |

## 1. Vision

A single, auditable system of record for ICE Services SOWs that replaces the
current patchwork of Access databases, Excel logs, email intake, manual
T-Sheets administration, and manual SharePoint packet curation — so that a SOW
can be set up, tracked, and billed end-to-end with a small trained team and a
complete audit trail.

## 2. Problem Statement

Today, project setup, timekeeping administration, cost tracking, and billing
packet assembly are done manually across disconnected tools:

- SOW intake arrives by email/Excel; SOW numbers are assigned manually in Access.
- T-Sheets jobs, sub-codes, and employee assignments are created 100% manually.
- The field team tracks progress informally; accounting has limited mid-project
  visibility (PP-018 exists to replace that informal process).
- Billing packets are assembled by manual T-Sheets exports, Excel totaling, and
  SharePoint folder curation by the field team (PP-023, PP-028).
- Closeout depends on remembering which signed/approved documents exist
  (PP-068).
- There is no billings-to-date tracking, no hold loop, and no consistent audit
  trail (PP-031, PP-030, PP-042).

The cost is slow closeouts, billing errors, rework, and dependence on a few
individuals' institutional knowledge (e.g., Shannon's monthly accrual pull is
the only current mid-project accounting touchpoint — PP-032).

## 3. Target Users

| Role | Primary needs |
|---|---|
| NS Admin | Set up SOWs, master data, T-Sheets provisioning, user administration |
| Ops Accounting (e.g., Shannon) | True-up reconciliation, billing packet review, closeout validation, holds |
| Project Manager / Field Lead | Mark work complete, track budget-to-actual, attach documents |
| Executive | Portfolio visibility (R2: PP-048) |
| AR | Receive approved packets in a consistent format (R1: PP-029, PP-040) |
| Finance / FP&A | Billed-to-date, accrual support & project financial reporting (PP-031, R3: PP-032, PP-072) |
| System Admin | Users, roles, configuration, audit logs |

## 4. Phase 1 Scope

**In scope (per register v2.2):** Project setup (SOW), timekeeping
integration, project tracking & monitoring, billing/invoicing/closeout, master
data, integrations (T-Sheets, Procurement Power App, HR, AR handoff),
platform/security/infrastructure, and reporting.

**Out of scope for Phase 1 delivery (per register):** Estimating,
CRM/Opportunity, Procore, full GL posting, full progress-billing workflow
unless rules are provided.

**Release structure:**

- **R1 (49 requirements incl. Pre-R1 spike PP-047):** The complete
  SOW → track → bill → closeout → AR handoff path for labor + materials, with
  identity, master data, documents, audit, and one-time data migration.
- **R2 (11 requirements):** Change orders, full overtime rules, equipment
  section, client-specific packet formats, progress billing data, GL
  interface, notifications, executive & billing/AR reporting.
- **R3 (2 requirements):** Accrual data support (PP-032) and project
  financial / accrual-support reporting (PP-072), pending Shannon's accrual
  workbook/formula.

**Explicit R1 boundary rule:** R1 implements only R1-assigned requirements,
plus the *minimum* overtime behavior required by PP-023 (labor packet
assembly must be able to flag OT/straight time using one default rule).
Advanced overtime configuration (PP-036), full OT rule engine (PP-015),
equipment integration (PP-025), progress billing (PP-021), accrual reporting
(PP-032, PP-072), and executive reporting (PP-048/PP-071) are **excluded** from
the R1 implementation path. R1 also needs a *minimum* notification behavior for
PP-069 (closeout task + email); the configurable notification center (PP-043)
is R2.

## 5. Success Criteria (Phase 1 / R1)

1. A SOW can be created, activated (T-Sheets job + sub-codes provisioned),
   edited, and tracked from a single system with a complete audit trail.
2. Billing packet assembly for labor + materials is automated (no manual
   T-Sheets export/totaling), validated against required closeout documents,
   reviewed in a reconciliation workspace, and handed off to AR.
3. Billings-to-date and budget-to-actual are visible per SOW without Excel.
4. One-time migration from Access/Excel is validated and cutover completed.
5. All external systems sit behind stable service interfaces (PP-016), so a
   T-Sheets or Procurement replacement does not require business-logic
   rewrites.

## 6. Constraints & Assumptions

- **No production data is assumed available** during development. All
  integrations are built against interface contracts + synthetic fixtures
  (see `docs/data/synthetic-data-strategy.md`).
- T-Sheets API capabilities (sub-code creation, employee restriction,
  timestamps) are **unconfirmed** (PP-011, PP-047) — Pre-R1 spike is a blocker
  gate for the provisioning feature.
- Procurement Power App is backed by an **on-prem SQL Server** (not
  Dataverse); the PO↔SOW matching field and the on-prem read path are
  unconfirmed (PP-038, PP-024; D-018, D-035).
- Rates/billing live in an **Access DB**, not T-Sheets (D-017); the rate
  schema and on-prem read path are unconfirmed (DISC-018, D-035).
- HR source of truth is unconfirmed (PP-053).
- AR handoff mechanism is unconfirmed (PP-029, PP-040).
- Microsoft-centric environment is implied by the register (Entra ID,
  on-prem SQL/Power App, SharePoint, Access).
- Baker Tilly brand standards apply (PP-057).

## 7. Key Risks (from register)

| Risk | Ref | Mitigation |
|---|---|---|
| T-Sheets sub-code API unconfirmed | PP-011, PP-047 🔴 | Pre-R1 sandbox spike; adapter isolation (PP-016); fallback manual mapping workflow if API insufficient |
| Migration missing from estimate; cannot go live without it | PP-044 🔴 | Migration feature 014 with validation gate before cutover |
| Pre-job obligations checklist undefined | PP-007 | Decision D-003; start with per-project-type minimal checklist |
| Document storage decision open | PP-006, PP-046 | Decision D-002; DocumentStore port allows late switch |
| Change order design undefined | PP-020 | R2; decision D-008 before feature 015 |

## 8. Delivery Methodology

- **BMAD** (this run): product, UX, architecture, delivery context.
- **Spec Kit:** turns each approved bounded capability (feature) into an
  implementation-ready specification.
- **Ralph:** autonomously implements one approved Spec Kit feature at a time,
  in the order given by `docs/product/feature-sequence.md`.

Feature sequence, release plan, traceability, UX, and architecture are defined
in the sibling documents in `docs/`.
