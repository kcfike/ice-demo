# Phase 1 PRD — ICE Services Project Lifecycle Platform

| Field | Value |
|---|---|
| Phase | Phase 1 — Project Setup, Tracking & Billing |
| Status | BMAD draft v1.0 (2026-09-26) |
| Authoritative source | `docs/source/phase-1-requirements-register-v2.2.md` (v2.2) |
| Traceability | `docs/product/requirements-traceability.md` |
| Decisions | `docs/architecture/decision-register.md` (IDs cited as D-xxx) |
| Discovery | `docs/discovery/discovery-backlog.md` (IDs cited as DISC-xxx) |

> The requirements register v2.2 is authoritative. This PRD restates each PP
> requirement, adds acceptance criteria that are directly implied by the
> requirement statement, and links open issues. Where the register marks a
> requirement ⚠️ or 🔴, the open item is carried into the decision register or
> discovery backlog — it is **not** silently resolved here.

## 1. Product Summary

Phase 1 delivers a web-based SOW lifecycle platform covering: SOW setup,
T-Sheets timekeeping provisioning and retrieval, project tracking, billing
packet assembly and review, closeout validation, AR handoff, master data,
security/RBAC, one-time migration, and (R2/R3) change orders, advanced OT,
equipment, reporting, and accrual support.

## 2. Release Definitions

| Release | Contents | Gate |
|---|---|---|
| **R1** | All R1-assigned PP requirements + Pre-R1 spike (PP-047) + minimum OT behavior required by PP-023 + minimum closeout notification required by PP-069 | T-Sheets spike passed; migration validated; dry-run billing for real SOWs accepted by Ops Accounting |
| **R2** | PP-015, PP-020, PP-021, PP-025, PP-027, PP-036, PP-043, PP-048, PP-054, PP-055, PP-071 | R1 in production; OT rules list and CO design decisions closed |
| **R3** | PP-032, PP-072 | Accrual workbook/formula obtained from Ops Accounting |

**R1 exclusion rule (binding):** R1 does **not** implement: advanced overtime
configuration (PP-036), the full OT rule engine (PP-015), equipment billing
integration (PP-025), progress billing (PP-021), accrual reporting (PP-032,
PP-072), and executive/portfolio reporting (PP-048, PP-071). R1's only overtime
behavior is
the minimum required for PP-023 packet labor assembly (a single default
OT/straight-time determination stored on labor lines).

## 3. Functional Requirements

### 3.1 Epic 1 — SOW / Project Setup

#### PP-001 — Auto SOW Number Generation (R1, P1, ✅)
Upon successful submission of a new SOW record, the system shall automatically
assign a unique, sequential, immutable SOW number. Concurrent creation shall
not produce duplicate numbers.

**Acceptance criteria**
- AC-1: SOW number is assigned exactly once, at the moment the SOW exits
  Draft activation and is committed.
- AC-2: Numbers are sequential with no gaps caused by the application's normal
  flow (a failed/rolled-back submission does not consume a number).
- AC-3: Two concurrent submissions cannot receive the same number.
- AC-4: SOW number is immutable after assignment; no UI or API path rewrites it.
- AC-5: Number format and starting value follow D-004 (starting sequence from
  Access history).

**Open:** starting sequence value (register note) → D-004.

#### PP-002 — Project Budget Capture (R1, P1, ✅)
The SOW creation form shall capture the approved project budget by at least:
Labor, Materials, Equipment, and Subcontractor costs. A total budget field
shall auto-calculate.

**Acceptance criteria**
- AC-1: Budget has category structure at minimum {Labor, Materials, Equipment,
  Subcontractor}; category list is data-driven, not hard-coded (supports
  per-project-type required-category configuration, PP-037).
- AC-2: Total = sum of category budgets, auto-calculated, never user-editable
  as an independent value.
- AC-3: Each category budget is the sum of its line items.
- AC-4: Existing Access records where labor is a single lump sum are
  representable (a single-line labor budget is valid).

#### PP-003 — Labor Budget by Discipline (R1, P1, ✅)
Within the Labor budget category, the system shall support line-item entries
by trade/discipline and role level, each with hours and rate where applicable.

**Acceptance criteria**
- AC-1: Labor budget lines carry (trade/discipline, role level, hours, rate,
  amount).
- AC-2: Trade and Role Level come from the Labor Category master (PP-035).
- AC-3: Lines support the two-axis trade × role classification.
- AC-4: These lines are the unit that maps to T-Sheets sub-codes (PP-011).

#### PP-004 — Project Type Classification (R1, P1, ✅)
The SOW record shall include a required, configurable Project Type field.
Initial values: Hard Dollar, T&M, OMR, Camp. Project Type shall drive billing
rules, OT treatment, packet format, and closeout document requirements.

**Acceptance criteria**
- AC-1: Project Type is a required dropdown on SOW creation, sourced from
  configuration (PP-037), not hard-coded.
- AC-2: All four initial values (Hard Dollar, T&M, OMR, Camp) are present in
  seed configuration.
- AC-3: Project Type is a first-class key for: OT treatment, packet format,
  required budget categories, required closeout documents (PP-068).
- AC-4: Adding a new project type requires no code deployment (D-005).

#### PP-005 — SOW Metadata Capture (R1, P1, ✅)
The SOW creation form shall capture: Request Date, Requester, Customer,
Department, Customer PO Number, Customer Reference / Job Description, and SOW
Narrative / Scope.

**Acceptance criteria**
- AC-1: All listed fields are captured on the SOW record.
- AC-2: Customer is a controlled reference to the Customer master (PP-033);
  Department to the Department master (PP-034).
- AC-3: Requester is an identity reference (user or external party per
  D-006).
- AC-4: Free-text fields (Narrative/Scope, Job Description) have sensible
  length limits and are searchable.

#### PP-006 — Supporting Document Attachment (R1, P2, ⚠️)
The SOW record shall support attachment of supporting documents, including
approved estimate, AFE, customer PO, approval email, and other support. Each
document shall be tagged with a document type and be viewable/downloadable
from the SOW record.

**Acceptance criteria**
- AC-1: Users can upload documents to an SOW; each upload carries a required
  document type from a configurable type list.
- AC-2: Documents are viewable and downloadable from the SOW record.
- AC-3: Document type feeds closeout validation (PP-068) and packet relevance
  filtering (PP-028).
- AC-4: Storage backend is behind the DocumentStore port (D-002); switching
  backend requires no change to business logic.

**Open:** SharePoint vs Blob/app storage → D-002.

#### PP-007 — SOW Creation Workflow (R1, P1, ⚠️)
The system shall enforce a SOW creation workflow: request received, pre-job
obligations validated, budget/metadata entered, submitted for activation, SOW
number generated, T-Sheets job created. SOW remains Draft until activation
completes.

**Acceptance criteria**
- AC-1: SOW status is Draft from creation until activation completes.
- AC-2: Activation is a single atomic operation that (a) assigns the SOW number
  (PP-001) and (b) provisions the T-Sheets job (PP-010); failure of (b) rolls
  back or clearly marks activation as failed/retryable — no half-activated SOW.
- AC-3: The pre-job obligations checklist is per-project-type and
  admin-maintained (D-003); it must pass (or be explicitly waived with reason
  per D-003) before activation can be submitted.
- AC-4: Every workflow step is recorded in the activity log (PP-022) and audit
  trail (PP-042).

**Open:** pre-job obligations checklist undefined and varies by project type →
D-003, DISC-002.

#### PP-008 — SOW Edit / Amend (R1, P2, ❌)
After creation, an authorized user shall be able to modify budget lines,
metadata, and attachments on an active SOW. All edits shall be captured in the
audit trail. Minor corrections shall be distinct from formal change orders.

**Acceptance criteria**
- AC-1: Authorized roles (NS Admin, PM per D-009) can edit budget lines,
  metadata, and attachments on an Active SOW.
- AC-2: Every field-level change is written to the immutable audit log
  (PP-042) with user, timestamp, old/new values.
- AC-3: The UI clearly distinguishes "edit/correction" from "change order"
  (R2, PP-020); an edit never silently creates a CO.
- AC-4: Edits to fields that have already been billed (e.g., after Invoiced)
  require the higher-privilege path defined by D-009.

**Open:** edit permissions and post-billing edit rules → D-009.

#### PP-009 — Standardised SOW Form (R1, P2, ✅)
The SOW form shall use validated dropdowns for Customer, Department, Project
Type, and Labor Category. Required fields shall be marked. Budget lines shall
be presented in a structured table.

**Acceptance criteria**
- AC-1: The four named fields are dropdowns bound to their masters/config.
- AC-2: Required fields are marked and validated client- and server-side.
- AC-3: Budget entry is a structured table (rows = lines, columns = hours,
  rate, amount), with auto-totals per category and overall (PP-002).
- AC-4: Form is usable by a trained non-technical user without reference
  material for common fields (Baker Tilly brand applied, PP-057).

---

### 3.2 Epic 2 — Timekeeping Integration

#### PP-010 — Auto-Provision T-Sheets Job (R1, P1, ✅)
Upon SOW activation, the system shall create the corresponding T-Sheets
job/project code using the SOW number, description, and client reference. The
T-Sheets job ID shall be stored on the SOW record.

**Acceptance criteria**
- AC-1: Activation triggers job creation via TimekeepingService only (PP-016).
- AC-2: Job name/description derives from SOW number, description, client
  reference (format per DISC-001).
- AC-3: The external T-Sheets job ID is persisted on the SOW and shown in the
  SOW detail view.
- AC-4: If job creation fails, activation is marked failed with a retry action;
  no orphan job state that cannot be reconciled.
- AC-5: Provisioning is idempotent: re-running activation for the same SOW does
  not create a second job.

#### PP-011 — Discipline Sub-Code Creation (R1, P1, ⚠️🔴)
For each labor discipline/role line on the SOW budget, the system shall create
or map a corresponding T-Sheets sub-code/cost assignment so time can be charged
by discipline rather than one blended job.

**Acceptance criteria**
- AC-1: Every labor budget line (PP-003) results in a T-Sheets sub-code (or a
  recorded manual mapping) before the SOW is Active.
- AC-2: Sub-code names follow a convention that round-trips to
  (trade, role level) (DISC-001).
- AC-3: The sub-code mapping is stored on the SOW (line → external sub-code ID).
- AC-4: If the T-Sheets API cannot create sub-codes (spike outcome), the
  fallback workflow (manual creation + admin-entered mapping) is available and
  logged — **blocker: spike PP-047 outcome** (DISC-001, D-010).

**Open:** API support for sub-code creation and employee restriction
unconfirmed → Pre-R1 spike PP-047; DISC-001; D-010.

#### PP-012 — Employee Assignment to Sub-Codes (R1, P1, ✅)
The system shall allow the NS Admin to assign employees to specific discipline
sub-codes at setup. Assignment shall be reflected in T-Sheets so the employee
can charge time.

**Acceptance criteria**
- AC-1: NS Admin can assign any active employee (EmployeeDirectory) to any
  sub-code of a SOW.
- AC-2: Assignment is pushed to T-Sheets via TimekeepingService and verified
  (read-back) before being marked complete.
- AC-3: Employee identity and project role are independent (PP-053): assignment
  is by employee ID, role label is descriptive only.
- AC-4: Assignment changes are audited (PP-042).

#### PP-013 — Employee Unassignment at Close (R1, P1, ✅)
When a SOW reaches Ready for Billing or Closed status, the system shall
unassign employees from the job/sub-codes or deactivate charging, preventing
further time entry.

**Acceptance criteria**
- AC-1: Transition to Ready for Billing (or Closed) triggers unassignment/deactivation
  via TimekeepingService.
- AC-2: Deactivation is verified by read-back.
- AC-3: A SOW that is reopened (On Hold released / billing rejected) restores
  charging capability per D-011 (late-cost/reopen scenarios).
- AC-4: The event and its outcome are logged (PP-022) and auditable (PP-042).

**Open:** late-cost/reopen handling needs confirmation → D-011.

#### PP-014 — Labor Data Retrieval for Billing (R1, P1, ✅)
The system shall retrieve labor hours by employee, sub-code, and date range,
including start/end timestamps where required to support OT determination.

**Acceptance criteria**
- AC-1: A query returns hours by (employee, sub-code, date range) with
  start/end timestamps per entry where the time source provides them.
- AC-2: Results are attributable to a SOW via the sub-code mapping (PP-011).
- AC-3: Retrieval is deterministic and re-runnable (same inputs → same data)
  so packet assembly can be regenerated.
- AC-4: All access is via TimekeepingService (PP-016).

#### PP-015 — OT / Straight-Time Flagging (R2, P1, ⚠️)
The system shall flag time as straight time or overtime based on configurable
rules, including client/project-type overrides. OT determination shall be
stored in the new system and available for billing.

**R1 note:** R1 implements only the *minimum* behavior this requirement needs
for PP-023: a default rule (e.g., >40h/week per DISC-005 outcome) applied at
packet assembly, stored on the labor line as the OT/ST flag. The configurable
rule engine, client/project-type overrides, after-hours rates, and special
thresholds are R2 (with PP-036).

**Acceptance criteria (R2 scope)**
- AC-1: Rules are configurable per D-012: default rule set, project-type
  override, client override, after-hours rates, special rate thresholds.
- AC-2: Determination is computed from raw time data (T-Sheets does not
  provide the flag — register note), stored, and visible in the billing packet.
- AC-3: Rules are admin/finance-maintained (PP-036) and versioned so past
  packets remain explainable.

**Open:** structured rules list needed from client → D-012, DISC-005.

#### PP-016 — T-Sheets Adapter Abstraction (R1, P2, ✅)
All T-Sheets API calls shall be isolated behind a TimekeepingService
interface. Business logic shall not call T-Sheets endpoints directly.

**Acceptance criteria**
- AC-1: A `TimekeepingService` port exists (architecture §Interfaces); only
  one adapter implements it in R1.
- AC-2: No business-domain code references T-Sheets types, endpoints, or SDKs
  (enforced by an architecture test / dependency rule).
- AC-3: A simulator adapter (synthetic data) can replace the real adapter
  without code changes (configuration switch, see
  `docs/data/synthetic-data-strategy.md`).

---

### 3.3 Epic 3 — Project Tracking & Monitoring

#### PP-017 — SOW Status State Machine (R1, P1, ⚠️)
The system shall enforce a defined SOW status lifecycle, including at minimum:
Draft, Active, On Hold, Ready for Billing, Billing in Review, Approved for AR,
Invoiced, Complete. Each transition shall have an assigned role and be logged.

**Acceptance criteria**
- AC-1: The eight listed statuses exist; the state machine permits only
  defined transitions (table in architecture §Domain Model).
- AC-2: Each transition declares allowed roles (per D-009 mapping).
- AC-3: Every transition is logged with user, timestamp, reason/notes
  (PP-022) and written to the audit trail (PP-042).
- AC-4: Illegal transitions are rejected with a clear error; no status can be
  set by direct record update outside the state machine.
- AC-5: Hold (On Hold) carries reason code + expected resolution date (PP-030)
  and a resume path.

**Open:** exact transition matrix and hold-resume semantics → D-013 (state
machine is proposed in architecture; register does not fix all transitions).

#### PP-018 — Budget-to-Actual Dashboard (R1, P2, ⚠️)
For each active SOW, the system shall display budgeted vs actuals for Labor,
Materials, Equipment, and Subs, with variance indicators and drill-down by
discipline where available.

**Acceptance criteria**
- AC-1: Per-SOW view shows budget vs actual for the four categories.
- AC-2: Variance indicators (e.g., % and absolute) with threshold coloring
  (thresholds configurable, D-014).
- AC-3: Labor drills down by discipline (from sub-code actuals, PP-014);
  materials/equipment/subs drill to source documents/PO lines where data
  exists (R1: labor + materials; equipment actuals arrive with PP-025 in R2 —
  the dashboard shows "no data" rather than omitting the category).
- AC-4: Data is refreshed on a defined cadence (D-015: on-demand refresh +
  daily refresh).

**Open:** refresh cadence and threshold values → D-014, D-015.

#### PP-019 — OM Code Management (R1, P2, ✅)
The system shall support creation, assignment, and tracking of OM codes for
overhead/internal work. OM codes shall be distinguishable from customer-billed
SOW codes and usable for time/cost allocation.

**Acceptance criteria**
- AC-1: An OM code reference list exists, admin-maintained.
- AC-2: OM codes are a distinct type from SOW codes in every report and view
  (they never appear as customer-billed SOWs).
- AC-3: OM codes are usable for time/cost allocation (accept time or cost
  entries/links in R1 per D-016; billing treatment defined in D-016).
- AC-4: Creation/assignment/tracking are audited (PP-042).

**Open:** OM code list source and billing treatment → D-016, DISC-008.

#### PP-020 — Change Order Management (R2, P2, ⚠️)
The system shall support change orders against an existing SOW. A change order
shall be linked to the parent SOW, carry its own budget delta, support approval
workflow, and be visible in billing.

**Acceptance criteria**
- AC-1: CO is a first-class record linked to the parent SOW (D-008).
- AC-2: CO carries its own budget delta (category + line structure, PP-002).
- AC-3: CO has an approval workflow (request → approve/reject) with roles per
  D-008.
- AC-4: Approved COs adjust the SOW budget and are visible in billing packet
  and budget-to-actual.
- AC-5: CO history is immutable and auditable.

**Open:** CO as first-class record vs budget edit, approval rules → D-008,
DISC-009.

#### PP-021 — Progress Billing Data (R2, P3, ⚠️)
The system shall track cumulative hours, POs, billings, and payments to date
for active SOWs. The progress-billing workflow may be deferred until rules are
defined.

**Acceptance criteria**
- AC-1: Cumulative hours, PO totals, billings, and payments-to-date are
  tracked per SOW (payments depend on GL data availability, PP-054).
- AC-2: Progress-billing *workflow* is only built if rules are provided
  (register scope note); otherwise data tracking only.

**Open:** rules for progress billing → DISC-010 (scope disagreement with
estimate noted in register).

#### PP-022 — Project Activity Log (R1, P2, ❌)
Each SOW shall maintain a chronological activity log capturing status changes,
budget edits, document changes, system events, and user actions.

**Acceptance criteria**
- AC-1: A per-SOW chronological feed captures the five listed event classes.
- AC-2: Each entry has actor (user or system), timestamp, and human-readable
  summary; field-level detail is resolved by linking to the audit record
  (PP-042).
- AC-3: Log is append-only; no edit/delete UI or API.
- AC-4: Distinct from, but linked to, the immutable field-level audit trail
  (PP-042) and the security log (PP-070).

---

### 3.4 Epic 4 — Billing / Invoicing / Closeout

#### PP-023 — Auto-Assemble Packet — Labor (R1, P1, ✅)
Upon closeout/billing trigger, the system shall retrieve labor hours by
discipline/role, apply OT flagging, calculate labor charges, and populate the
labor section of the billing packet.

**Acceptance criteria**
- AC-1: On trigger (PP-069 closeout request or Ops Accounting initiation),
  labor data is retrieved per PP-014.
- AC-2: Hours are grouped by discipline/role (sub-code mapping, PP-011).
- AC-3: OT/straight-time flag is applied using the R1 minimum rule (PP-015 R1
  scope, D-012) and stored on the labor line.
- AC-4: Charges = hours × applicable rate (budget rate or actual rate per
  D-017); calculations are itemized and re-derivable.
- AC-5: The labor section is part of the packet review workspace (PP-026) and
  regeneratable if data changes before approval.

**Decided (D-017, 2026-09-30):** the rate source for billing charges is the
**Access billing DB / rates master** (trade/discipline → role-level, incl. OT
rates) — **not T-Sheets** (T-Sheets is hours-only) and not the budget.
Open: exact rate schema + the on-prem read path → DISC-018, D-035.

#### PP-024 — Auto-Assemble Packet — Materials / POs (R1, P1, ⚠️)
The system shall retrieve approved POs, received materials, stock issues, and
related vendor/cost data associated with the SOW and populate the materials
section.

**Acceptance criteria**
- AC-1: PO headers/lines, received goods, and vendor/cost data are retrieved
  via ProcurementService, filtered by SOW number or mapped identifier
  (PP-038).
- AC-2: Only *approved* POs enter the materials section (status filter).
- AC-3: Each material line is traceable to its source PO/receipt.
- AC-4: If PO↔SOW matching cannot be resolved for a record, it is surfaced as
  an exception in the review workspace (PP-026), not silently included or
  dropped.

**Open:** which departments are fully on the Procurement Power App; PO↔SOW
matching convention (against the on-prem SQL PO table); on-prem SQL
connectivity model → DISC-003, D-018, D-035.

#### PP-025 — Auto-Assemble Packet — Equipment (R2, P2, ⚠️)
The system shall retrieve equipment charges and stock-issue reports associated
with the SOW and populate the equipment section.

**Acceptance criteria (R2)**
- AC-1: Equipment charges and stock-issue reports retrieved from the source
  identified in DISC-004.
- AC-2: Equipment section follows the same traceability and review rules as
  materials.

**Open:** source system for equipment/stock-issue data unidentified → DISC-004.

#### PP-026 — Ops Accounting Review & True-Up (R1, P1, ⚠️)
The assembled billing packet shall be presented to Ops Accounting in a
reconciliation workspace showing budget vs actual by category/discipline,
source documents, and flagged variances. Ops Accounting shall be able to
true-up line items, add adjustments/notes, approve, or reject with comments.
Before finalisation, the system shall validate required closeout documents per
PP-068 and block AR submission if required items are missing.

**Acceptance criteria**
- AC-1: Reconciliation workspace shows, per category (and discipline for
  labor): budget, actual, variance, source document links.
- AC-2: Ops Accounting can adjust line items (true-up), add notes/adjustments
  (amount, reason, user, date — same shape as PP-055 records), approve, or
  reject with mandatory comments.
- AC-3: Variances beyond threshold are flagged (D-014).
- AC-4: Before finalisation, required closeout document validation (PP-068)
  runs; missing items block AR submission with a visible gap list.
- AC-5: Every true-up action is audited (PP-042) and visible in the activity
  log (PP-022).
- AC-6: Rejection returns the SOW to a defined state (D-013) with the comment
  recorded.

**Open:** true-up adjustment rules (what may be adjusted, by how much) → D-017
partially; register note: reconciliation step, not rubber-stamp.

#### PP-027 — Client-Specific & Project-Type Formatting (R2, P2, ⚠️)
The system shall support client-specific and project-type-specific billing
packet layouts. For T&M and OMR types, the packet shall include both the
original approved estimate and a final "actuals" summary showing completed work
vs original scope. Ops Accounting shall be able to override formatting before
finalisation.

**Acceptance criteria (R2)**
- AC-1: Layout is selected by (client, project type) configuration.
- AC-2: T&M and OMR packets include original approved estimate + final actuals
  summary side-by-side.
- AC-3: Ops Accounting can override layout choices before finalisation; the
  override is recorded.

#### PP-028 — PDF Generation & Relevance Filtering (R1, P1, ⚠️)
The finalised packet shall be generated as a PDF containing cover/invoice page,
labor detail, materials/PO detail, equipment detail, subs detail, and required
supporting documents. The system shall apply a configurable relevance filter
to include only documents/data required for invoicing and exclude internal noise.

**Acceptance criteria**
- AC-1: R1 PDF contains: cover/invoice page, labor detail, materials/PO detail,
  subs detail, and required supporting documents. (Equipment detail appears
  when equipment data exists; in R1 that section is present only if data is
  available, per PP-025 being R2 — see note below.)
- AC-2: Relevance filter is configurable per project type/client (D-019):
  which document types and data sections are included.
- AC-3: Excluded items remain attached to the SOW (nothing is deleted); only
  inclusion in the packet is filtered.
- AC-4: Generated PDF is stored, linked to the SOW, and retrievable (PP-046).

**Note:** Register lists equipment detail in the PDF. R1 excludes the equipment
*integration* (PP-025); the R1 PDF template includes an equipment section
placeholder that is populated if equipment data is available, to keep the
template stable when PP-025 lands in R2.

#### PP-029 — AR Handoff (R1, P1, ⚠️)
Upon approval, the system shall update the SOW status to Approved for AR, make
the packet and supporting documents available to AR, and record the handoff
timestamp/user. AR downstream processing remains out of Phase 1 scope.

**Acceptance criteria**
- AC-1: Approval of the packet (PP-026) transitions SOW to Approved for AR.
- AC-2: Packet + supporting documents are delivered to AR via the mechanism
  chosen in D-020 (R1 default: shared folder deposit of the packet bundle;
  portal/email possible per D-020).
- AC-3: Handoff record stores timestamp, user, and delivery confirmation.
- AC-4: No AR system write-back occurs in Phase 1 (register scope).

**Open:** handoff mechanism (folder drop / portal / email / S2S) → D-020.

#### PP-030 — Hold / Follow-Up Loop (R1, P2, ⚠️)
If the packet cannot be completed, Ops Accounting shall be able to place the
SOW in On Hold with reason code and expected resolution date. The SOW shall
remain visible in a hold queue and resume at review when resolved.

**Acceptance criteria**
- AC-1: Ops Accounting can place a SOW in On Hold with required reason code
  (configurable list) and expected resolution date.
- AC-2: A hold queue view lists held SOWs with age, reason, expected date.
- AC-3: Resolution returns the SOW to the appropriate workflow state (D-013).
- AC-4: Hold/release events are logged (PP-022) and audited (PP-042).

#### PP-031 — Billed-Amount Tracking (R1, P2, ✅)
The system shall record billed amount by category, billing date, invoice
number where available, adjustments, and cumulative billed-to-date on the SOW
record.

**Acceptance criteria**
- AC-1: Each billing event records: category breakdown, billing date, invoice
  number (optional), adjustments.
- AC-2: Billed-to-date is the cumulative sum, always derivable from events
  (no independent counter).
- AC-3: Billed-to-date feeds budget-to-actual (PP-018) and R2 billing reports
  (PP-071).

#### PP-032 — Accrual Data Support (R3, P3, ⚠️)
For each active SOW, the system shall provide accrual-support data, including
unbilled labor, unbilled materials, and estimated completion percentage.
Accrual calculation and journal posting remain in the existing finance process
in Phase 1.

**Acceptance criteria (R3)**
- AC-1: Per SOW: unbilled labor, unbilled materials, estimated completion %
  available for export/view.
- AC-2: No journal posting in Phase 1 (register scope).

**Open:** Shannon's accrual workbook/formula needed → DISC-011.

#### PP-055 — Write-Offs & Cost Adjustments (R2, P2, ➕)
The system shall support recording write-offs, cost adjustments, and
corrections against an active SOW. Each adjustment shall capture amount,
category, reason, authorizing user, and date. Adjustments shall be visible in
budget-to-actual and billing.

**Acceptance criteria (R2)**
- AC-1: Adjustment record: amount, category, reason, authorizing user, date.
- AC-2: Adjustments affect budget-to-actual (PP-018) and billing (PP-023/024
  as applicable).
- AC-3: Full audit (PP-042); adjustments are never silently deleted.

#### PP-068 — Required Closeout Document Validation (R1, P1, 🆕)
The system shall maintain a configurable required closeout document register
per project type/client/SOW and validate required documents before billing
packet finalisation or AR handoff. Missing required documents shall be flagged
and block AR submission until resolved.

**Acceptance criteria**
- AC-1: A register defines required document types per (project type, client,
  SOW override), sourced from PP-037 project-type configuration with client
  and SOW overrides.
- AC-2: Validation runs before finalisation and before AR handoff (PP-026,
  PP-029).
- AC-3: Missing documents produce a visible gap list (document type, expected
  source, owner if known) and block AR submission.
- AC-4: Resolution (document attached) clears the block; the event is audited.
- AC-5: Signed/approved copies (register note) are first-class document types
  in the seed register.

#### PP-069 — Job Completion / Closeout Workflow (R1, P1, 🆕)
The field/project team or NS Admin shall be able to mark a SOW as work-complete
or request closeout. The system shall transition the SOW to Ready for Billing,
create an Ops Accounting task, send notification, and track closeout age.

**Acceptance criteria**
- AC-1: PM/Field Lead or NS Admin can trigger "mark work complete / request
  closeout" on an Active SOW.
- AC-2: The SOW transitions to Ready for Billing (PP-017 state machine).
- AC-3: An Ops Accounting task (queue item) is created with SOW reference.
- AC-4: A notification is sent to the configured Ops Accounting recipient(s)
  (R1 minimum of PP-043, per D-021: in-app task + email).
- AC-5: Closeout age (time in Ready for Billing / awaiting review) is tracked
  and visible in the queue.
- AC-6: The trigger event is logged (PP-022) and audited (PP-042).

---

### 3.5 Epic 5 — Master Data & Reference Data

#### PP-033 — Customer Master (R1, P1, ✅)
The system shall maintain a customer master with ID, Name, Billing Contact,
Address, Payment Terms, Active/Inactive status, and default billing format
reference. New customers shall require validation before activation.

**Acceptance criteria**
- AC-1: All listed attributes present; customer ID is stable.
- AC-2: Inactive customers remain referenceable by existing SOWs but are not
  selectable for new SOWs.
- AC-3: New customer creation requires validation step by an authorized role
  (D-022) before activation.
- AC-4: Default billing format reference is the hook for R2 packet layouts
  (PP-027).
- AC-5: Changes are audited (PP-042).

**Open:** actual customer master file needed for migration → DISC-012.

#### PP-034 — Department Master (R1, P1, ⚠️)
The system shall maintain a department reference list usable across SOW setup,
reporting, and cost allocation. The list shall be admin-maintainable without
code deployment.

**Acceptance criteria**
- AC-1: Department reference list CRUD via admin UI (no deployment).
- AC-2: Used by SOW setup (PP-005), reporting (PP-048), cost allocation
  (PP-019).
- AC-3: Inactive departments remain referenceable by historical SOWs.

**Open:** source of truth (local list vs Paylocity vs other) → D-023, DISC-006.

#### PP-035 — Labor Category / Discipline Master (R1, P1, ✅)
The system shall maintain a two-level labor classification: Trade/Discipline
and Role Level. This master shall drive budget entry, T-Sheets sub-code naming,
and billing detail.

**Acceptance criteria**
- AC-1: Two-level hierarchy (Trade/Discipline → Role Level), admin-maintained.
- AC-2: Drives labor budget entry (PP-003), sub-code naming convention
  (PP-011), and billing detail grouping (PP-023).
- AC-3: Rate defaults may be attached (per D-017) but rates remain
  configurable per SOW line.

**Open:** full list from rates/bid sheets → DISC-007.

#### PP-036 — Overtime Rules Configuration (R2, P2, ⚠️)
The system shall support configurable OT rules, including default rules,
project-type overrides, client overrides, after-hours rates, and special rate
thresholds. Rules shall be maintained by an admin/finance role.

**Acceptance criteria (R2)**
- AC-1: Rule model supports: default, project-type override, client override,
  after-hours rates, special rate thresholds.
- AC-2: Admin/finance role maintains rules; changes are versioned and audited.
- AC-3: Rules consumed by the OT determination in PP-015.

**Open:** structured rules list from client → D-012, DISC-005.

#### PP-037 — Project Type Configuration (R1, P2, ✅)
The system shall support a project-type configuration table defining default
billing model, OT rule set, packet format, required budget categories, and
required closeout documents.

**Acceptance criteria**
- AC-1: Configuration table with the five listed dimensions per project type.
- AC-2: Consumed by PP-004 (values), PP-002/AC-1 (required categories),
  PP-068 (required closeout documents), PP-028 (packet format), and the R1
  minimum OT rule (PP-015 R1 scope).
- AC-3: Admin-maintainable without code deployment; changes audited.
- AC-4: Seed configuration ships for Hard Dollar, T&M, OMR, Camp.

---

### 3.6 Epic 6 — Integrations

#### PP-038 — Procurement Power App / on-prem SQL Read (R1, P1, ⚠️)
The system shall read PO headers/lines, received goods, stock issues, vendor
info, and cost data from the Procurement Power App (backed by an **on-prem
SQL Server**, not Dataverse), filtered by SOW number or mapped project
identifier.

**Acceptance criteria**
- AC-1: All reads go through the `ProcurementService` port.
- AC-2: Query surface: PO headers/lines, received goods, stock issues, vendor
  info, cost data; filter by SOW number or mapped project identifier.
- AC-3: Read-only in Phase 1 (no writes to the on-prem SQL Server).
- AC-4: Unmatched/ambiguous records surface as review exceptions (PP-026),
  never silently included.
- AC-5: Simulator adapter available with synthetic fixtures (see
  `docs/data/synthetic-data-strategy.md`).

**Open:** the SQL PO table/fields; PO records carrying SOW/job number (D-018);
**how the app reaches the on-prem SQL Server** (connectivity model — D-035 🔴)
→ DISC-003.

#### PP-039 — T-Sheets Bidirectional Integration (R1, P1, ✅)
The system shall integrate with T-Sheets for job/sub-code creation, employee
assignment/unassignment, time retrieval, and job activation/deactivation. All
calls shall use the TimekeepingService adapter.

**Acceptance criteria**
- AC-1: The five capability groups are implemented behind TimekeepingService
  (see architecture §Interfaces for the contract).
- AC-2: All operations are idempotent or carry client-side correlation keys.
- AC-3: Errors are surfaced with retry/inspect paths; no silent failure.
- AC-4: Sandbox spike (PP-047) results are the acceptance gate for the
  sub-code and assignment capabilities.

#### PP-040 — AR / Payables Handoff (R1, P2, ⚠️)
Upon AR handoff, the system shall make the packet and supporting documents
available in the required format/location. Phase 1 mechanism may be portal,
shared folder deposit, or email attachment.

**Acceptance criteria**
- AC-1: Mechanism per D-020 is implemented and confirmed with AR.
- AC-2: Delivery includes the packet PDF + supporting documents in a stable
  naming/location convention.
- AC-3: Delivery is logged (PP-029) and failure is visible/retryable.

**Open:** AR mechanism confirmation → D-020, DISC-013.

#### PP-053 — HR / Employee Master Data (R1, P2, ➕)
The system shall consume employee, department, active/inactive status, and
job-title data from the authoritative HR/payroll source where available.
Project role on a SOW shall remain independent of HR job title.

**Acceptance criteria**
- AC-1: Employee data consumed via the `EmployeeDirectory` port.
- AC-2: Fields: employee ID, name, email, department, active/inactive,
  job title.
- AC-3: Project role assignment (PP-012) is independent of HR job title.
- AC-4: Simulator adapter with synthetic fixtures; real adapter behind config
  switch (source unconfirmed — DISC-006).
- AC-5: Inactive employees cannot be newly assigned to sub-codes but remain
  visible on historical data.

**Open:** authoritative source (Paylocity / Workday / local / other) → DISC-006.

#### PP-054 — Accounting / GL Integration (R2, P2, ⚠️)
The system shall exchange data with Accounting/GL/AR as required, including
billing identifiers, accrual data, invoice numbers, and payment status where
available. Interface mechanism shall be confirmed.

**Acceptance criteria (R2)**
- AC-1: Mechanism per D-024 (basic write-back first; GL posting is out of
  Phase 1 scope per register).
- AC-2: Data exchanged: billing identifiers, invoice numbers, payment status
  where available.

**Open:** interface mechanism "TBC" — major risk → D-024, DISC-014.

---

### 3.7 Epic 7 — Platform, Security & Infrastructure

#### PP-041 — Role-Based Access Control (R1, P1, ⚠️)
The system shall support role-based access control with minimum roles:
NS Admin, Ops Accounting, Project Manager/Field Lead, Executive, AR,
Finance/FP&A, and System Admin. Roles shall be assignable via Entra ID, AIG
Portal, or in-app mechanism.

**Acceptance criteria**
- AC-1: The seven minimum roles exist; authorization is enforced server-side
  on every action (including state transitions, PP-017).
- AC-2: Role assignment mechanism per D-007 (Entra ID group ↔ role mapping is
  the R1 recommendation; in-app assignment is the fallback).
- AC-3: A user may hold multiple roles (union of permissions).
- AC-4: Role changes take effect on next session/token refresh; no stale
  elevated access.

**Open:** exact role list and whether AR/Finance need direct logins → D-007.

#### PP-042 — Field-Level Audit Trail (R1, P2, ⚠️)
The system shall maintain an immutable audit log for SOW records capturing
field-level changes, status transitions, billing edits, document changes, and
configuration changes.

**Acceptance criteria**
- AC-1: Audit events for: field-level SOW changes (old/new), status
  transitions, billing true-ups, document attach/delete, and configuration
  changes (master data, project-type config, closeout register, OT rule
  defaults).
- AC-2: Log is append-only; no application path can update or delete rows.
- AC-3: Each event: actor, timestamp (UTC), entity, field, old/new (or
  action), session/correlation ID.
- AC-4: Distinct from security event logging (PP-070); both exportable.
- AC-5: Audit writes are in the same transaction as the change (no
  orphaned changes without audit).

#### PP-043 — Notification / Alerting (R2, P3, ✅)
The system shall generate in-app and/or email notifications at key workflow
events, including SOW created, ready for billing, packet awaiting review,
packet approved, SOW on hold, and budget threshold exceeded.

**Acceptance criteria (R2)**
- AC-1: The six listed event types trigger notifications.
- AC-2: Recipients and thresholds are configurable (per role/user).
- AC-3: Channels: in-app and email.

**R1 note:** R1 implements only the minimum closeout notification required by
PP-069 (in-app task + email to configured Ops Accounting recipient) via the
NotificationService port, so the port and delivery path exist before the R2
notification center.

#### PP-044 — Access / Excel Data Migration (R1, P1, ❌🔴)
The system shall provide a one-time migration from the Access database,
Excel logs, customer master, and relevant SharePoint job list/document
references into the new system. Migration shall be validated before cutover.

**Acceptance criteria**
- AC-1: Migration tooling ingests: Access DB (SOWs, numbers), Excel logs,
  customer master, SharePoint job list + document references.
- AC-2: Migration is re-runnable (idempotent per source record key).
- AC-3: A validation report is produced: record counts, field-level mapping
  results, exceptions, referential integrity checks.
- AC-4: A cutover gate: Ops Accounting signs off on a validation report
  covering a sample of live SOWs before go-live (D-025).
- AC-5: Post-migration, existing SOW numbers continue the sequence (PP-001,
  D-004).

**Open:** source file access and schema mapping → DISC-012, D-025. 🔴 Cannot
go live without it.

#### PP-045 — Performance & Availability (R1, P2, ❌)
The system shall support concurrent use by the expected user population,
complete billing-packet assembly within acceptable performance targets, and
meet agreed availability during business hours.

**Acceptance criteria**
- AC-1: Concurrent users: per D-026 (register: need confirm user count and SLA).
- AC-2: Billing packet assembly (labor + materials) completes within the
  target in D-026 for the largest known SOW (target set after DISC-015 sizing).
- AC-3: Availability target for business hours per D-026.

#### PP-046 — Document Storage (R1, P2, ⚠️)
All user-uploaded and system-generated documents shall be stored or linked in
a controlled document repository, linked to the SOW record, and
retrievable/searchable by SOW number, date, and type.

**Acceptance criteria**
- AC-1: Both user-uploaded and system-generated (e.g., packet PDFs) documents
  are in the controlled repository.
- AC-2: Every document is linked to its SOW; searchable by SOW number, date,
  type.
- AC-3: Access to documents is RBAC-scoped (PP-041).
- AC-4: Backend per D-002; the DocumentStore port is the only access path.

**Open:** SharePoint vs app/Blob storage → D-002.

#### PP-047 — T-Sheets API Validation Spike (Pre-R1, P1, 🔴)
Before design lock, the team shall complete a T-Sheets sandbox spike
validating job creation, sub-code creation, employee assignment, time-entry
timestamps, activation/deactivation, and error handling.

**Acceptance criteria**
- AC-1: A written spike report covering the six capability areas with pass/
  fail/limitations.
- AC-2: Any failed capability has a documented fallback (e.g., manual sub-code
  creation + mapping) and an updated risk entry.
- AC-3: Spike outcome is recorded in the decision register (D-010) and
  discovery backlog (DISC-001) and gates Feature 008 (T-Sheets Provisioning).

**Status:** 🔴 Blocker for Epic 2. Not a code deliverable; a time-boxed
investigation.

#### PP-049 — Login Screen & Session Management (R1, P2, ➕)
The system shall provide a central login screen with username/password input,
password reset, and session timeout. SSO via Entra ID shall be supported as
the preferred authentication method.

**Acceptance criteria**
- AC-1: Entra ID SSO is the primary path (D-007); username/password fallback
  with password reset exists.
- AC-2: Session timeout enforced (duration per D-027, ICE IT guidance).
- AC-3: Login screen branded (PP-057).
- AC-4: Login attempts feed the security log (PP-070).

#### PP-050 — Two-Factor Authentication (R1, P2, ➕)
The system shall support MFA/2FA in accordance with ICE security policy. The
method shall follow ICE IT/security guidance.

**Acceptance criteria**
- AC-1: MFA enforced per ICE security policy (D-028); method per ICE IT
  guidance (Entra ID Conditional Access is the R1 recommendation).
- AC-2: MFA state recorded in security log (PP-070) where relevant.

#### PP-051 — User Management (R1, P2, ⚠️)
Admin users shall be able to create, modify, and remove users and assign
roles. User list shall show name, email, role, status, and last login.

**Acceptance criteria**
- AC-1: System Admin (and NS Admin for role assignment per D-009) can create,
  modify, deactivate users and assign roles.
- AC-2: User list: name, email, role(s), status, last login.
- AC-3: Deactivation is soft (preserves audit attribution).
- AC-4: Management mechanism per D-007 (in-app admin UI; identity source
  Entra ID).

**Open:** in-app vs AIG Portal / Entra ID management → D-007.

#### PP-052 — New User Onboarding Notification (R1, P3, ➕)
Upon user creation, the system shall send an email with temporary password or
SSO enrollment link. User shall be required to set a permanent password or
complete enrollment on first login.

**Acceptance criteria**
- AC-1: On user creation, an email is sent (temporary password or SSO
  enrollment link, per D-007 path).
- AC-2: First login enforces password change / enrollment completion.
- AC-3: Sent via NotificationService port (simulator-capable).

#### PP-056 — CI/CD Pipelines (R1, P2, ➕)
The system shall use automated build, test, and deployment pipelines across
DEV and PROD environments, with source control and release controls.

**Acceptance criteria**
- AC-1: Build + test + deploy pipelines for DEV and PROD (architecture
  §Deployment).
- AC-2: Source control with branch protection and release controls (tag/
  approval per D-029).
- AC-3: All automated tests (unit, integration against simulators, contract
  tests) run in CI.

#### PP-057 — Theme & Branding (R1, P3, ➕)
The system shall apply Baker Tilly brand standards, including header/footer,
color scheme, typography, and logo placement.

**Acceptance criteria**
- AC-1: Brand tokens (colors, typography, logo, header/footer) applied
  application-wide (see `docs/ux/ux-spec.md` §Branding).
- AC-2: Brand asset source confirmed (DISC-016).

#### PP-070 — Login / Security Event Logging (R1, P2, 🆕)
The system shall log security and access events, including login attempts,
success/failure, user, timestamp, source, session duration, and logout. Logs
shall be exportable and retained per ICE policy.

**Acceptance criteria**
- AC-1: Events: login attempt, success, failure, user, timestamp, source
  (IP/UA), session duration, logout.
- AC-2: Distinct from field-level audit (PP-042); both exportable.
- AC-3: Retention per ICE policy (D-030).

---

### 3.8 Epic 8 — Reporting

#### PP-048 — Portfolio / Executive Dashboard (R2, P2, ➕)
The system shall provide a cross-project view filterable by department,
project type, status, customer, and owner, showing budget, actuals, variance,
and billing progress.

**Acceptance criteria (R2)**
- AC-1: Filters: department, project type, status, customer, owner.
- AC-2: Columns/metrics: budget, actuals, variance, billing progress
  (billed-to-date per PP-031).
- AC-3: OM codes excluded from customer-billed views (PP-019).

#### PP-071 — Billing & AR Reporting (R2, P2, 🆕)
The system shall provide billing and AR reports, including billed-to-date by
SOW/client, unbilled activity, packet status, invoice number where available,
AR handoff date, payment status where available, and open billing exceptions.

**Acceptance criteria (R2)**
- AC-1: All listed fields present, SOW-level and client-level rollups.
- AC-2: Payment status shown when available from PP-054 source, else marked
  "not received".
- AC-3: Open billing exceptions (holds, validation gaps) listed with age.

#### PP-072 — Project Financial / Accrual Reporting (R3, P3, 🆕)
The system shall provide project financial and accrual-support reports,
including incurred labor/materials/equipment/subs, unbilled incurred,
estimated completion, accrual inputs by period, and variance to budget.
(Not full GL journal posting.)

**Acceptance criteria (R3)**
- AC-1: Reports cover: incurred by category (labor/materials/equipment/subs),
  unbilled incurred, estimated completion, accrual inputs by period, variance
  to budget.
- AC-2: Data derived from the same financial tracking substrate (PP-031,
  PP-018, PP-032) — no separate ledger.
- AC-3: No GL journal posting (register scope note).

## 4. Non-Functional Requirements (cross-cutting)

| ID | Requirement | Release | Notes |
|---|---|---|---|
| NFR-AUDIT | Immutable audit (PP-042) + security log (PP-070) | R1 | See Epic 7 |
| NFR-PERF | Performance & availability (PP-045) | R1 | Targets pending D-026/DISC-015 |
| NFR-ADAPTER | Integration isolation (PP-016) | R1 | Ports in architecture |
| NFR-DOC | Document storage (PP-046) | R1 | D-002 |
| NFR-CICD | Pipelines (PP-056) | R1 | Feature 001 |
| NFR-BRAND | Baker Tilly branding (PP-057) | R1 | UX spec |
| NFR-SEC | SSO + MFA (PP-049, PP-050) | R1 | D-007, D-028 |

## 5. Out of Scope (Phase 1)

Per register: Estimating, CRM/Opportunity, Procore, full GL posting, full
progress-billing workflow unless rules are provided. Additionally excluded
from R1 by the R1 boundary rule: PP-015 (full), PP-020, PP-021, PP-025, PP-027,
PP-036, PP-043 (full), PP-048, PP-054, PP-055, PP-071, PP-032 (R3), PP-072 (R3).

## 6. Release Gate Definitions

**R1 exit gate**
- Pre-R1 spike (PP-047) completed and recorded.
- All R1 requirements implemented and tested against simulators.
- Migration (PP-044) validated; cutover sign-off (D-025).
- At least one full dry-run billing (create → activate → bill → closeout →
  handoff) accepted by Ops Accounting on a real historical SOW.

**R2 exit gate**
- OT rules list (DISC-005) and CO design (D-008) closed.
- Equipment source (DISC-004) identified and integrated.
- Executive + billing/AR reporting accepted by Finance/FP&A.

**R3 exit gate**
- Accrual workbook (DISC-011) obtained; accrual-support data (PP-032) and
  project financial / accrual-support reports (PP-072) match Shannon's manual
  pull on a sample month (within tolerance D-032).
- No GL journal posting implemented.
