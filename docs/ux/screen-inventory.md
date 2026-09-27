# Screen Inventory — ICE Services Project Lifecycle Platform

| Field | Value |
|---|---|
| Status | BMAD draft v1.0 (2026-09-26) |
| Scope | R1 screens (R2/R3 marked as extensions) |
| Companion | `docs/ux/ux-spec.md`, `docs/ux/workflow-map.md` |

Status: **R1** = required for release 1; **R2** = release 2; **R3** = release 3.

## 1. Authentication & Account

| # | Screen | Rel | Purpose | Key PP refs |
|---|---|---|---|---|
| S-01 | Login (SSO + username/password fallback) | R1 | Authenticate; MFA enforced | PP-049, PP-050 |
| S-02 | Password reset / first-login password set | R1 | Onboarding + reset | PP-049, PP-052 |
| S-03 | Session expiry notice | R1 | Timeout handling | PP-049 |
| S-04 | User profile / preferences (R2: notification prefs) | R1/R2 | Account info | PP-043 (R2) |

## 2. Work Queues & SOW List

| # | Screen | Rel | Purpose | Key PP refs |
|---|---|---|---|---|
| S-10 | My SOWs (owner queue, role-filtered) | R1 | Default landing for NS Admin/PM | PP-017 |
| S-11 | Ops Accounting queue (closeout/review/holds) | R1 | Primary Ops Accounting work surface | PP-026, PP-030, PP-069 |
| S-12 | SOW list (global, filterable) | R1 | Search/filter all SOWs | PP-005, PP-017 |
| S-13 | SOW comparison / portfolio (R2) | R2 | Executive cross-project view | PP-048 |

## 3. SOW Creation & Setup

| # | Screen | Rel | Purpose | Key PP refs |
|---|---|---|---|---|
| S-20 | SOW creation wizard (intake → budget → confirm) | R1 | Create SOW; pre-job checklist; activation | PP-005, PP-007, PP-009 |
| S-21 | SOW creation — Budget table | R1 | Category + labor line entry | PP-002, PP-003 |
| S-22 | SOW creation — Metadata & references | R1 | Customer, dept, PO, narrative | PP-005 |
| S-23 | SOW creation — Pre-job obligations checklist | R1 | Per-project-type checklist + waiver | PP-007 |
| S-24 | Activation result / T-Sheets provisioning status | R1 | Job + sub-code provisioning outcome | PP-010, PP-011, PP-012 |
| S-25 | SOW creation — Document attach (initial) | R1 | Attach estimate/AFE/PO/approval | PP-006 |

## 4. SOW Detail (tabs)

| # | Screen | Rel | Purpose | Key PP refs |
|---|---|---|---|---|
| S-30 | SOW detail — Overview (status, metadata, people) | R1 | Hub; status actions | PP-005, PP-017 |
| S-31 | SOW detail — Budget (with variance panel) | R1 | Budget + actual + drill-down | PP-002, PP-003, PP-018 |
| S-32 | SOW detail — Time (labor by discipline) | R1 | Actual hours, ST/OT | PP-014, PP-015 (min) |
| S-33 | SOW detail — Documents | R1 | Attach/view/download; closeout strip | PP-006, PP-046, PP-068 |
| S-34 | SOW detail — Costs (materials/POs, subs) | R1 | PO/receipt/vendor cost view | PP-024, PP-038 |
| S-35 | SOW detail — Billing (packet + billed-to-date) | R1 | Packet status, billing events | PP-028, PP-031 |
| S-36 | SOW detail — History (activity log) | R1 | Chronological feed | PP-022 |
| S-37 | SOW detail — Audit (field-level for this SOW) | R1 | Per-SOW audit drill | PP-042 |
| S-38 | SOW detail — Change orders (R2) | R2 | CO list + delta | PP-020 |
| S-39 | SOW detail — Equipment (R2) | R2 | Equipment/stock-issue | PP-025 |

## 5. Ops Accounting Workspaces

| # | Screen | Rel | Purpose | Key PP refs |
|---|---|---|---|---|
| S-40 | Reconciliation workspace — Summary | R1 | Budget vs actual, variance flags | PP-026 |
| S-41 | Reconciliation workspace — Labor | R1 | ST/OT, true-up by discipline | PP-023, PP-026 |
| S-42 | Reconciliation workspace — Materials/POs | R1 | PO lines, exceptions, true-up | PP-024, PP-026 |
| S-43 | Reconciliation workspace — Subs | R1 | Subcontractor costs | PP-026 |
| S-44 | Reconciliation workspace — Equipment (placeholder R1 / full R2) | R1/R2 | Equipment section | PP-025 |
| S-45 | Reconciliation workspace — Closeout documents | R1 | Required list, upload, gap list | PP-068 |
| S-46 | Reconciliation workspace — Actions (approve/reject/hold) | R1 | Finalise, PDF, handoff, hold | PP-026, PP-028, PP-029, PP-030 |
| S-47 | Packet preview (PDF) | R1 | Preview before approval | PP-028 |
| S-48 | Packet layout override (R2) | R2 | Client/project-type format | PP-027 |
| S-49 | Adjustments / write-offs (R2) | R2 | Record adjustments | PP-055 |

## 6. Closeout & AR Handoff

| # | Screen | Rel | Purpose | Key PP refs |
|---|---|---|---|---|
| S-50 | Closeout trigger confirmation | R1 | Mark work complete / request closeout | PP-069 |
| S-51 | Closeout task detail (Ops Accounting) | R1 | Task with SOW context | PP-069 |
| S-52 | Hold queue | R1 | Held SOWs, reason, expected date | PP-030 |
| S-53 | AR handoff status & log | R1 | Delivery mechanism, status, retry | PP-029, PP-040 |
| S-54 | Billed-to-date ledger (per SOW) | R1 | Billing events + cumulative | PP-031 |

## 7. Master Data & Configuration

| # | Screen | Rel | Purpose | Key PP refs |
|---|---|---|---|---|
| S-60 | Customer master (list + form) | R1 | CRUD + validation | PP-033 |
| S-61 | Department master | R1 | CRUD | PP-034 |
| S-62 | Labor category master (trade → role) | R1 | Two-level hierarchy | PP-035 |
| S-63 | Project type configuration | R1 | 5 dimensions per type | PP-037, PP-004 |
| S-64 | Closeout document register | R1 | Required docs per type/client/SOW | PP-068 |
| S-65 | OM code list | R1 | Overhead/internal codes | PP-019 |
| S-66 | OT rules configuration (R2) | R2 | Default/type/client/after-hours | PP-036, PP-015 |
| S-67 | Change order templates/rules (R2) | R2 | CO workflow config | PP-020 |

## 8. Reporting

| # | Screen | Rel | Purpose | Key PP refs |
|---|---|---|---|---|
| S-70 | Budget-to-actual report (per SOW / portfolio) | R1 | Drill-down report | PP-018 |
| S-71 | Billed-to-date report | R1 | Billing rollups | PP-031 |
| S-72 | Executive / portfolio dashboard (R2) | R2 | Cross-project filters | PP-048 |
| S-73 | Billing & AR report (R2) | R2 | Packet/invoice/exception report | PP-071 |
| S-74 | Accrual support export (R3) | R3 | Unbilled + completion % | PP-032, PP-072 |

## 9. Administration & Audit

| # | Screen | Rel | Purpose | Key PP refs |
|---|---|---|---|---|
| S-80 | Users & roles | R1 | CRUD users, assign roles | PP-051, PP-041 |
| S-81 | Onboarding / new-user status | R1 | Track enrollment | PP-052 |
| S-82 | Role/permission matrix (read-only) | R1 | Reference | PP-041 |
| S-83 | Integration status (adapter health) | R1 | Real/simulator, errors | PP-016, PP-038, PP-039, PP-053 |
| S-84 | Integration mode switch (admin) | R1 | Simulator ↔ real adapter | Config |
| S-85 | Audit explorer | R1 | Field-level audit, export | PP-042 |
| S-86 | Security events log | R1 | Login/session, export | PP-070 |
| S-87 | System settings | R1 | Session timeout, thresholds, etc. | PP-049, D-014 |
| S-88 | Migration console (run/validate/report) | R1 | One-time migration | PP-044 |
| S-89 | Migration validation report | R1 | Counts, exceptions, sign-off | PP-044, D-025 |

## 10. Coverage Check

| PP ID | Screens |
|---|---|
| PP-001 | S-24 (activation assigns number) |
| PP-002 | S-21, S-31 |
| PP-003 | S-21, S-31, S-41 |
| PP-004 | S-22, S-63 |
| PP-005 | S-22, S-30 |
| PP-006 | S-25, S-33, S-45 |
| PP-007 | S-20, S-23, S-24 |
| PP-008 | S-31, S-33 (edit controls) |
| PP-009 | S-20, S-21, S-22 |
| PP-010 | S-24, S-83 |
| PP-011 | S-24, S-83 |
| PP-012 | S-24, S-83 |
| PP-013 | S-50, S-53 |
| PP-014 | S-32, S-41 |
| PP-015 | S-41 (R1 min), S-66 (R2) |
| PP-016 | S-83, S-84 |
| PP-017 | S-10, S-12, S-30 |
| PP-018 | S-31, S-70 |
| PP-019 | S-65, S-31 (OM distinct) |
| PP-020 | S-38, S-67 |
| PP-021 | S-35, S-73 |
| PP-022 | S-36 |
| PP-023 | S-41, S-47 |
| PP-024 | S-34, S-42, S-47 |
| PP-025 | S-39, S-44 |
| PP-026 | S-40–S-46 |
| PP-027 | S-48, S-47 |
| PP-028 | S-47, S-53 |
| PP-029 | S-53 |
| PP-030 | S-46, S-52 |
| PP-031 | S-35, S-54, S-71 |
| PP-032 | S-74 |
| PP-033 | S-60 |
| PP-034 | S-61 |
| PP-035 | S-62 |
| PP-036 | S-66 |
| PP-037 | S-63 |
| PP-038 | S-34, S-83 |
| PP-039 | S-24, S-83, S-84 |
| PP-040 | S-53, S-84 |
| PP-041 | S-80, S-82 |
| PP-042 | S-37, S-85 |
| PP-043 | S-04 (R2) |
| PP-044 | S-88, S-89 |
| PP-045 | (non-visual; perf baseline) |
| PP-046 | S-33 |
| PP-047 | (spike; no screen) |
| PP-048 | S-72 |
| PP-049 | S-01, S-02, S-03 |
| PP-050 | S-01 |
| PP-051 | S-80 |
| PP-052 | S-81 |
| PP-053 | S-83 |
| PP-054 | S-53 (data), S-73 |
| PP-055 | S-49 |
| PP-056 | (CI/CD; no user screen) |
| PP-057 | all (brand tokens) |
| PP-068 | S-33, S-45, S-64 |
| PP-069 | S-50, S-51 |
| PP-070 | S-86 |
| PP-071 | S-73 |
| PP-072 | S-74 |

All 62 PP IDs are covered by at least one screen (or are non-visual: PP-045,
PP-047, PP-056).
