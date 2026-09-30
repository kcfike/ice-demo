# UX Specification — ICE Services Project Lifecycle Platform

| Field | Value |
|---|---|
| Status | BMAD draft v1.0 (2026-09-26) |
| Scope | R1 UX (with R2/R3 extension points noted) |
| Branding | Baker Tilly brand standards (PP-057) |
| Screen inventory | `docs/ux/screen-inventory.md` |
| Workflows | `docs/ux/workflow-map.md` |

## 1. Design Principles

1. **Auditability is visible, not hidden.** Every consequential action
   (status change, true-up, hold, approval) produces a visible trace in the
   SOW activity log and is confirmable by the user before commit.
2. **One SOW, one page of truth.** The SOW record is the hub; everything
   (budget, documents, time, packet, billing) is reachable from it within two
   clicks.
3. **Fail loudly at the right gate.** Validation (required fields, closeout
   documents, unmatched POs) surfaces *before* handoff, with a concrete gap
   list and owner hints — never after AR has the packet.
4. **Degrade gracefully, visibly.** When an integration is on simulator or a
   data source has no data (e.g., equipment in R1), the UI says so explicitly
   ("Simulator data" badge / "No data available from source") instead of
   showing blanks or omitting sections.
5. **Small trained team, low context switching.** Keyboard-friendly, minimal
   modal nesting, queue-based work views for Ops Accounting (the primary
   heavy user).
6. **Baker Tilly brand** applied consistently (PP-057).

## 2. Branding & Visual Language (PP-057)

- Header: Baker Tilly logo (top-left), product name "ICE Services — Project
  Lifecycle", user menu (top-right).
- Footer: environment indicator (DEV/SIM/PROD — critical while running on
  simulators), version, support link.
- Color/typography: Baker Tilly brand tokens (asset source pending DISC-016).
  R1 ships with a token file (`brand/tokens.json`) so assets can be dropped in
  without code changes.
- Status colors (consistent everywhere):
  - Draft — neutral grey
  - Active — blue
  - On Hold — amber
  - Ready for Billing — teal
  - Billing in Review — purple
  - Approved for AR — green outline
  - Invoiced — green
  - Complete — dark grey
- Variance indicators: green (under budget) / amber (within threshold) / red
  (over threshold, threshold per D-014).

## 3. Navigation Model

```
App shell
├── Work queues (role-filtered)
│   ├── My SOWs (owner view)
│   ├── Ops Accounting queue (closeout tasks, holds, reviews)
│   └── Admin queue (pending validations, onboarding)
├── SOWs
│   ├── SOW list (filters: status, customer, project type, department, owner)
│   └── SOW detail (tabs)
├── Master Data (NS Admin / System Admin)
│   ├── Customers · Departments · Labor categories · Project types
│   ├── Closeout document register · OM codes · (R2) OT rules
├── Reports (R2: executive, billing/AR)
├── Audit & Security (System Admin)
│   ├── Audit explorer · Security events · Users
└── Administration
    ├── Users & roles · Settings · Integration status (adapter health)
```

- Primary navigation is a left sidebar (collapsible), breadcrumbs on detail
  pages.
- Queue items are the default landing page per role (NS Admin → My SOWs;
  Ops Accounting → Ops Accounting queue).

## 4. Global UX Patterns

### 4.1 Forms
- Required fields marked with `*`; validation errors inline + summary banner
  at top on submit failure.
- Dropdowns for controlled vocabularies (Customer, Department, Project Type,
  Labor Category, Document Type, Reason code) — PP-009.
- Long free-text (SOW Narrative) as full-width textarea with character count.
- Draft autosave on the SOW creation form (recovery on tab close).
- Two-step confirmation only for destructive or irreversible actions
  (activation, AR submission, reject with comment).

### 4.2 Budget table (PP-002, PP-003)
- Grouped by category (Labor / Materials / Equipment / Subcontractor).
- Labor rows: discipline, role level, hours, rate, amount (auto = hours ×
  rate). Category subtotal + grand total auto-calculated, read-only.
- Add/remove rows via row actions; reordering via drag (optional nicety, not
  required).
- Post-activation edits restricted per D-009; UI shows lock state with reason.

### 4.3 Tables & lists
- Column sorting, text filter, status chips.
- Bulk actions only where safe (e.g., "export list"); no bulk state changes
  without explicit per-item confirmation.
- Pagination or virtualization for >50 rows.

### 4.4 Notifications & toasts
- Toasts for action confirmations (non-blocking).
- In-app task list for workflow items (closeout tasks, hold resolutions).
- R1: closeout notification (PP-069) + user onboarding email (PP-052) use
  the same task + email pattern. R2: full notification center (PP-043).

### 4.5 Empty / error / loading states
- Every async panel has loading skeleton, empty state with next-action
  guidance, and error state with retry.
- Integration errors (T-Sheets, Procurement) show: what failed, which adapter
  (real/simulator), retry button, and link to Integration status page.

### 4.6 Simulator visibility (synthetic data rule)
- When any adapter is in simulator mode, a persistent banner:
  "⚠ DEV — [Timekeeping/Procurement/HR/Document] data is synthetic".
- Records touched by simulator adapters carry a `SIM` chip in lists.
  This prevents accidental confidence in synthetic billing numbers.

### 4.7 Auditability in UI
- "History" tab on SOW detail (activity log, PP-022).
- Every audit-eligible field shows a "last changed by / when" tooltip on hover.
- True-up adjustments show adjuster, reason, and amount inline.

## 5. Visual Concept Mockups (draft, 2026-09-29)

Three high-fidelity concept screens were produced and are stored in
`docs/ux/wireframes/`. They are consistent with this spec's substance; where
they introduce a pattern, that pattern is captured in §6 below.

| Mockup | Screen | Maps to | Notes |
|---|---|---|---|
| `wireframes/concept-01-sow-detail.png` | SOW detail — Overview | S-30 | Introduces the status stepper, condensed Recent Activity feed and Assigned Employees panel (§6.1) |
| `wireframes/concept-02-billing-reconciliation.png` | Billing & Closeout — Reconciliation | S-40–S-46 | Matches the §6.5 workspace; adds a "Notes" tab (new, not in S-40–S-49) |
| `wireframes/concept-03-portfolio-dashboard.png` | Project Portfolio dashboard | S-72 (PP-048) | **R2 scope** — shown for reference only; not an R1 screen |

**Open items to reconcile before the concepts are canonical:**
- **Status vocabulary:** concept-01's header chip reads "In Progress" and the
  stepper shows 5 states, but §2 defines 8 statuses and "Active" is the
  canonical label. The stepper and chip must be re-based onto the §2 set.
- **Sidebar IA:** the three concepts each use a different sidebar
  (Projects/My Work/Time · Home/SOWs/Planning/Execution · Dashboard/Billing/
  Reports). §3's navigation tree is the single source of truth; the mockups
  must converge on it rather than the spec adopting three variants.
- **Draft data artifacts:** the two SOW concepts disagree on customer
  (Riverside vs Riverton) and budget ($420k vs $2.48M), and concept-03 is
  dated 2025 while the SOWs are 2026. Tidy before treating as canonical.

## 6. Screen-Level Specifications (R1)

> Full inventory in `docs/ux/screen-inventory.md`; flows in
> `docs/ux/workflow-map.md`. Key screens with behavior notes:

### 6.1 SOW Detail — Overview & status stepper (S-30)
- **Status stepper (header):** horizontal progress rail showing the SOW's
  lifecycle position, with dated milestones under each completed step. The
  concept shows a 5-step happy path (Draft → Approved → Active → Ready to Bill
  → Closed). The §2 status set is the authoritative model — every §2 status
  (including On Hold, Billing in Review, Approved for AR, Invoiced) must map
  to a defined position on the rail. Dates come from the SOW activity log
  (PP-022). The header status chip label must use §2 vocabulary ("Active",
  not "In Progress").
- **Condensed Recent Activity feed:** last ~5 activity events (date, type,
  description, actor) with a "View All" link to the full History tab (S-36,
  PP-022). The History tab remains the complete chronological log; the feed is
  the hub-level summary (principle 2 — one page of truth).
- **Assigned Employees panel:** name, role, allocation %, status, with an
  "Add" action. Data from T-Sheets provisioning (PP-010/PP-012) / HR source.
- **Budget snapshot cards:** Approved Budget / Actual Cost / Remaining with
  % of budget — a read-only summary linking into the Budget tab (S-31, PP-018).
- **Missing-data chips:** e.g. "Customer PO ⚠ Missing" on required fields —
  surfaced inline on the hub per principle 4 / §4.6.

### 6.2 SOW Detail — Budget tab
- State-dependent editability (D-009): Draft = full edit; Active = restricted
  fields; post-Ready-for-Billing = read-only (corrections via adjustment
  records in R2, PP-055).
- Variance panel (budget vs actual) per category with drill-down for labor
  (by discipline) — PP-018.

### 6.3 SOW Detail — Documents tab
- Upload with mandatory document type (from closeout register types).
- Closeout readiness strip: "3 of 5 required documents present" with missing
  list (PP-068) — this strip is also on the Billing tab.
- Download single / all; preview inline for PDF/Office where supported.

### 6.4 Ops Accounting Queue
- Columns: SOW, customer, project type, stage (Ready for Billing / In Review /
  On Hold / Approved), age, variance flag, missing docs count.
- Row action: "Open review" → reconciliation workspace.
- Age thresholds highlight (closeout age per PP-069; hold age per PP-030).

### 6.5 Reconciliation Workspace (PP-026) — the core R1 screen
Layout (single screen, tabbed sections):
1. **Summary** — budget vs actual by category, totals, variance flags.
2. **Labor** — by discipline/role: budgeted hours/rate, actual hours, ST/OT
   split, computed charges; per-line true-up control.
3. **Materials/POs** — PO, vendor, lines, received qty/cost, source doc link;
   unmatched/exception rows pinned at top with "resolve" actions.
4. **Subs** — same shape as materials (R1: data present only if available).
5. **Equipment** — placeholder section in R1 (PP-025 is R2); shows "Not
   configured in R1".
6. **Closeout documents** — required list, present/missing, upload inline.
7. **Actions bar** — Approve (→ finalise + PDF), Reject (mandatory comment,
   returns to state per D-013), Hold (reason code + expected date, PP-030).
- True-up controls: adjust amount, mandatory reason, shows running totals.
- "Generate preview PDF" available before approval (PP-028) so layout issues
  are caught pre-approval.

### 6.6 AR Handoff view (PP-029, PP-040)
- Packet bundle contents list (PDF + documents), delivery mechanism (D-020),
  delivery status (queued/sent/failed), timestamp, user.
- Retry on failure; handoff log (append-only) shown below.

### 6.7 Master Data screens
- Generic CRUD shell (list + form) per entity; validation per PRD.
- Project Type configuration screen: the five dimensions (billing model, OT
  rule set, packet format, required budget categories, required closeout docs)
  as one coherent form per type — this is where PP-037 + PP-068 config lives.

### 6.8 Integration status (Administration)
- Per adapter: real/simulator mode, last health check, last call success/fail,
  open errors, "switch mode" (admin action, config change per
  `docs/architecture/architecture.md` §Configuration).
- Critical for support while R1 runs on simulators.

### 6.9 Audit explorer & Security events (Administration)
- Filter: entity, user, date range, event class; export to CSV (PP-042,
  PP-070).
- Read-only; no delete; row detail shows old/new values (audit) or session
  data (security).

## 7. Accessibility & Usability Targets

- WCAG 2.1 AA: contrast, keyboard navigation, focus order, screen-reader
  labels on all interactive elements.
- All workflow actions reachable by keyboard; no color-only state encoding
  (status chips include text/icon).
- Target: a trained user completes SOW creation (intake → activation) in
  under 10 minutes without help (measure in dry-run, PP-045 adjacent).

## 8. R2/R3 UX Extension Points (noted, not designed here)

- OT rules admin screen (PP-036) — new Master Data entity.
- Change order screens (PP-020) — CO list, CO detail with delta + approval.
- Equipment tab in reconciliation workspace (PP-025) — replaces placeholder.
- Packet layout override controls (PP-027) — in reconciliation Actions.
- Executive dashboard (PP-048) + Billing/AR reports (PP-071) — Reports area.
- Notification preferences (PP-043) — user settings.
- Accrual export view (PP-032) — Reports area (R3).
