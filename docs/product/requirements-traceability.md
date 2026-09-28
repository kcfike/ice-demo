# Requirements Traceability — ICE Services Project Lifecycle Platform

| Field | Value |
|---|---|
| Status | BMAD draft v1.0 (2026-09-26) |
| Authoritative source | `docs/source/phase-1-requirements-register-v2.2.md` v2.2 |
| Features | `docs/product/feature-sequence.md` (R1: 001–016; R2/R3: 017–022) |
| Decision register | `docs/architecture/decision-register.md` |
| Discovery backlog | `docs/discovery/discovery-backlog.md` |

**Completeness invariant:** all 62 PP requirement IDs from register v2.2 must
appear in §1. IDs not mapped to a feature are listed in §4 (none expected).

**Feature numbering note:** R1 features 001–016 follow the proposed sequence
in the run brief. R2/R3 features are numbered 017–022 (recommended in
`docs/product/feature-sequence.md` §4). The proposed R1 slot "014 Change Orders
and Billing Extensions" is R2 scope per the R1 boundary rule and is delivered
as Feature 017.

## 1. Full Traceability Matrix

| PP ID | Rel | Pri | Feature(s) | Decision(s) | Discovery | Notes |
|---|---|---|---|---|---|---|
| PP-001 | R1 | P1 | 004, 015 | D-004 | DISC-012 | Auto SOW number; sequence continuity at migration |
| PP-002 | R1 | P1 | 004 | D-005 | — | Budget capture, 4 categories, auto-total |
| PP-003 | R1 | P1 | 004 | D-005 | DISC-007 | Labor budget by discipline × role |
| PP-004 | R1 | P1 | 003, 004 | D-005 | — | Project type drives rules |
| PP-005 | R1 | P1 | 004 | D-006 | — | SOW metadata capture |
| PP-006 | R1 | P2 | 005 | D-002 | — | Document attachment + tagging |
| PP-007 | R1 | P1 | 004, 006 | D-003 | DISC-002 | Creation workflow + pre-job checklist |
| PP-008 | R1 | P2 | 006 | D-009 | — | SOW edit/amend (distinct from CO) |
| PP-009 | R1 | P2 | 004 | — | — | Standardised SOW form |
| PP-010 | R1 | P1 | 008 | D-010 | DISC-001 | T-Sheets job provisioning |
| PP-011 | R1 | P1 | 008 | D-010 | DISC-001 🔴 | Sub-code creation (spike-gated) |
| PP-012 | R1 | P1 | 008 | D-010 | DISC-001, DISC-006 | Employee→sub-code assignment |
| PP-013 | R1 | P1 | 008, 013 | D-011 | DISC-001 | Unassignment at close / reopen |
| PP-014 | R1 | P1 | 009 | D-010 | DISC-001 | Labor retrieval for billing |
| PP-015 | R2 | P1 | 018 (full); 009 (R1 minimum) | D-012 | DISC-005 | OT/ST flagging; R1 = minimum rule for PP-023 |
| PP-016 | R1 | P2 | 007, 008, 009, 010 | — | — | Service-interface isolation |
| PP-017 | R1 | P1 | 006 | D-013 | — | SOW status state machine |
| PP-018 | R1 | P2 | 011 | D-014, D-015 | — | Budget-to-actual dashboard |
| PP-019 | R1 | P2 | 003 (list), 011 (usage) | D-016 | DISC-008 | OM code management |
| PP-020 | R2 | P2 | 017 | D-008 | DISC-009 | Change orders |
| PP-021 | R2 | P3 | 017 | — | DISC-010 | Progress billing data |
| PP-022 | R1 | P2 | 006 | — | — | Project activity log |
| PP-023 | R1 | P1 | 009 (labor calc), 012 (packet) | D-017 | — | Packet labor assembly |
| PP-024 | R1 | P1 | 010 (retrieval), 012 (packet) | D-018 | DISC-003 | Packet materials/POs |
| PP-025 | R2 | P2 | 019 | — | DISC-004 | Packet equipment section |
| PP-026 | R1 | P1 | 012 | D-014, D-017 | — | Ops Accounting review & true-up |
| PP-027 | R2 | P2 | 020 | — | DISC-017 | Client/project-type formatting |
| PP-028 | R1 | P1 | 012 | D-019 | — | PDF generation + relevance filter |
| PP-029 | R1 | P1 | 013, 014 | D-020 | DISC-013 | AR handoff + delivery |
| PP-030 | R1 | P2 | 013 | D-013 | — | Hold / follow-up loop |
| PP-031 | R1 | P2 | 011 (data model), 013 (event), 014 (tracking) | — | — | Billed-amount tracking |
| PP-032 | R3 | P3 | 022 | D-032 | DISC-011 | Accrual data support |
| PP-072 | R3 | P3 | 022 | D-032 | DISC-011 | Project financial / accrual-support reports (not GL posting) |
| PP-033 | R1 | P1 | 003, 015 | D-022 | DISC-012 | Customer master + migration |
| PP-034 | R1 | P1 | 003 | D-023 | DISC-006 | Department master |
| PP-035 | R1 | P1 | 003 | D-017 | DISC-007 | Labor category master |
| PP-036 | R2 | P2 | 018 | D-012 | DISC-005 | OT rules configuration |
| PP-037 | R1 | P2 | 003 | D-005 | — | Project type configuration |
| PP-038 | R1 | P1 | 007 (port), 010 (usage) | D-018 | DISC-003 | Procurement Power App read |
| PP-039 | R1 | P1 | 008, 009 | D-010 | DISC-001 | T-Sheets bidirectional integration |
| PP-040 | R1 | P2 | 014 | D-020 | DISC-013 | AR/Payables handoff mechanism |
| PP-041 | R1 | P1 | 002 | D-007 | — | RBAC, 7 minimum roles |
| PP-042 | R1 | P2 | 002 (port), 003–013 (usage) | D-030 | — | Field-level audit trail |
| PP-043 | R2 | P3 | 021 (full); 013 (R1 minimum) | D-021 | — | Notifications; R1 = PP-069 minimum |
| PP-044 | R1 | P1 | 015 | D-025 | DISC-012 🔴 | Access/Excel migration + cutover |
| PP-045 | R1 | P2 | 001 (baseline), 012 (assembly perf) | D-026 | DISC-015 | Performance & availability |
| PP-046 | R1 | P2 | 005 | D-002 | — | Document storage |
| PP-047 | Pre-R1 | P1 | 008 (gate) | D-010 | DISC-001 🔴 | T-Sheets spike — blocker |
| PP-048 | R2 | P2 | 021 | — | — | Portfolio/executive dashboard |
| PP-049 | R1 | P2 | 002 | D-007, D-027 | — | Login screen & session |
| PP-050 | R1 | P2 | 002 | D-028 | — | MFA/2FA |
| PP-051 | R1 | P2 | 002 | D-007, D-009 | — | User management |
| PP-052 | R1 | P3 | 002 | D-007, D-021 | — | New user onboarding notification |
| PP-053 | R1 | P2 | 007 (port), 008 (usage) | — | DISC-006 | HR/employee master data |
| PP-054 | R2 | P2 | 021 | D-024 | DISC-014 | Accounting/GL integration |
| PP-055 | R2 | P2 | 017 | — | — | Write-offs & cost adjustments |
| PP-056 | R1 | P2 | 001 | D-029 | — | CI/CD pipelines |
| PP-057 | R1 | P3 | 001 (tokens), 002 (login) | — | DISC-016 | Branding |
| PP-068 | R1 | P1 | 003 (register config), 012 (validation) | D-005, D-019 | — | Required closeout document validation |
| PP-069 | R1 | P1 | 013 | D-021 | — | Closeout workflow + task + notification |
| PP-070 | R1 | P2 | 002 | D-030 | — | Security event logging |
| PP-071 | R2 | P2 | 021 | — | DISC-014 | Billing & AR reporting |

**Total PP IDs: 62.**

## 2. Release Counts (verification)

| Release | Count | PP IDs |
|---|---|---|
| Pre-R1 | 1 | PP-047 |
| R1 | 48 | PP-001–PP-014, PP-016–PP-019, PP-022–PP-024, PP-026, PP-028–PP-031, PP-033–PP-035, PP-037–PP-042, PP-044–PP-046, PP-049–PP-053, PP-056, PP-057, PP-068, PP-069, PP-070 |
| R2 | 11 | PP-015, PP-020, PP-021, PP-025, PP-027, PP-036, PP-043, PP-048, PP-054, PP-055, PP-071 |
| R3 | 2 | PP-032, PP-072 |
| **Total** | **62** | |

## 3. Feature Coverage (R1: 001–016; R2/R3: 017–022)

| Feature | PP IDs covered | Rel |
|---|---|---|
| 001 Platform Foundation & CI/CD | PP-056, PP-057 (tokens), PP-045 (baseline), PP-016 (port conventions) | R1 |
| 002 Identity & Access | PP-041, PP-042 (audit port), PP-049, PP-050, PP-051, PP-052, PP-057 (login), PP-070 | R1 |
| 003 Master Data & Configuration | PP-033, PP-034, PP-035, PP-037, PP-019 (OM code list), PP-068 (closeout register config) | R1 |
| 004 SOW Core Setup | PP-001, PP-002, PP-003, PP-004, PP-005, PP-007 (intake side), PP-009 | R1 |
| 005 Document Management | PP-006, PP-046 | R1 |
| 006 SOW Lifecycle & Audit | PP-007 (activation), PP-008, PP-017, PP-022, PP-042 (usage) | R1 |
| 007 Integration Ports & Simulators | PP-016, PP-038 (port), PP-053 (port), NotificationService/ARHandoff/Audit ports | R1 |
| 008 T-Sheets Provisioning | PP-010, PP-011, PP-012, PP-013 (deactivate), PP-039 (provisioning side), PP-047 (gate) | R1 |
| 009 Labor Retrieval & Basic OT | PP-014, PP-015 (R1 minimum), PP-023 (labor calc), PP-039 (retrieval side) | R1 |
| 010 Procurement & Project Costs | PP-038 (usage), PP-024 (retrieval) | R1 |
| 011 Project Financial Tracking | PP-018, PP-019 (usage), PP-031 (billed-to-date data model) | R1 |
| 012 Billing Packet Assembly | PP-023, PP-024, PP-026, PP-028, PP-068 (validation), PP-045 (assembly perf) | R1 |
| 013 Closeout & Reconciliation | PP-013 (reopen), PP-029 (status), PP-030, PP-031 (billing event), PP-043 (R1 min), PP-069 | R1 |
| 014 AR Handoff & Billing Tracking | PP-029 (delivery), PP-031 (tracking), PP-040 | R1 |
| 015 Data Migration & Cutover | PP-044, PP-001 (sequence), PP-033 (customer master data) | R1 |
| 016 Reporting & Accrual (R1 slice) | PP-018 (dashboard views), PP-031 (billed-to-date report) | R1 |
| 017 Change Orders & Billing Extensions | PP-020, PP-021, PP-055 | R2 |
| 018 Overtime Rules Engine | PP-015 (full), PP-036 | R2 |
| 019 Equipment Integration | PP-025 | R2 |
| 020 Packet Formatting | PP-027 | R2 |
| 021 GL, Reporting & Notification Center | PP-048, PP-054, PP-071, PP-043 (full) | R2 |
| 022 Accrual Support | PP-032, PP-072 | R3 |

**Every PP ID is mapped to at least one feature.**

## 4. Requirement IDs Not Mapped to a Feature

**None.** All 62 PP IDs from register v2.2 appear in §1 and §3.

## 5. Open-Item Index (requirements with ⚠️/❌/🔴 in register)

| PP ID | Register flag | Carried in |
|---|---|---|
| PP-006 | ⚠️ document storage | D-002 |
| PP-007 | ⚠️ pre-job obligations | D-003, DISC-002 |
| PP-011 | ⚠️🔴 sub-code API | DISC-001, D-010 |
| PP-013 | ⚠️ late-cost/reopen | D-011 |
| PP-015 | ⚠️ OT rules source | D-012, DISC-005 |
| PP-017 | ⚠️ transitions implicit | D-013 |
| PP-018 | ⚠️ cadence/thresholds | D-014, D-015 |
| PP-019 | ⚠️ OM list/treatment | D-016, DISC-008 |
| PP-020 | ⚠️ CO design | D-008, DISC-009 |
| PP-021 | ⚠️ scope disagreement | DISC-010 |
| PP-024 | ⚠️ PO↔SOW matching | D-018, DISC-003 |
| PP-025 | ⚠️ source unidentified | DISC-004 |
| PP-026 | ⚠️ true-up rules | D-014, D-017 |
| PP-027 | ⚠️ layouts | DISC-017 |
| PP-028 | ⚠️ filter config | D-019 |
| PP-029 | ⚠️ AR mechanism | D-020, DISC-013 |
| PP-030 | ⚠️ hold semantics | D-013 |
| PP-032 | ⚠️ accrual formula | D-032, DISC-011 |
| PP-033 | ✅ (needs master file) | DISC-012 |
| PP-034 | ⚠️ department source | D-023, DISC-006 |
| PP-036 | ⚠️ rules list | D-012, DISC-005 |
| PP-038 | ⚠️ API/matching | D-018, DISC-003 |
| PP-040 | ⚠️ AR mechanism | D-020, DISC-013 |
| PP-041 | ⚠️ role list/AR logins | D-007 |
| PP-042 | ⚠️ vs PP-070 | D-030 |
| PP-044 | ❌🔴 missing from estimate | D-025, DISC-012 |
| PP-045 | ❌ user count/SLA | D-026, DISC-015 |
| PP-046 | ⚠️ storage | D-002 |
| PP-047 | 🔴 blocker | DISC-001, D-010 |
| PP-051 | ⚠️ management mechanism | D-007 |
| PP-054 | ⚠️ TBC mechanism | D-024, DISC-014 |

## 6. Epic → Feature Map

Derived table: epics and their PP IDs come from the authoritative register
(`docs/source/phase-1-requirements-register-v2.2.md`, 8 epics, 62 PPs); the
PP → feature mapping comes from the §1 matrix. The register, PRD, and
release-plan map epics to PP IDs only — this section is the single place that
closes the epic → feature gap.

| Epic (register) | PP IDs | Feature(s) | Release |
|---|---|---|---|
| Epic 1 — SOW / Project Setup | PP-001–009 | 004 (core setup); 003 (PP-004 project type); 005 (PP-006 documents); 006 (PP-007 activation, PP-008 edit/amend); 015 (PP-001 number sequence) | R1 |
| Epic 2 — Timekeeping Integration | PP-010–016 | 008 (provisioning); 009 (retrieval; PP-015 R1 minimum); 007, 010 (PP-016 adapter convention); 013 (PP-013 reopen); 018 (PP-015 full) | R1; R2 (PP-015 full) |
| Epic 3 — Project Tracking & Monitoring | PP-017–022 | 006 (PP-017 state machine, PP-022 activity log); 011 (PP-018, PP-019 usage); 003 (PP-019 OM code list); 017 (PP-020, PP-021) | R1; R2 (PP-020, PP-021) |
| Epic 4 — Billing / Invoicing / Closeout | PP-023–032, PP-055, PP-068, PP-069 | 012 (packet core); 009 (PP-023 labor calc); 010 (PP-024 retrieval); 011 (PP-031 data model); 013 (PP-029 status, PP-030, PP-069); 014 (PP-029 delivery, PP-031 tracking); 003 (PP-068 register config); 017 (PP-055); 019 (PP-025); 020 (PP-027); 022 (PP-032) | R1; R2 (PP-025, PP-027, PP-055); R3 (PP-032) |
| Epic 5 — Master Data & Reference Data | PP-033–037 | 003 (core masters); 015 (PP-033 customer master data); 018 (PP-036 OT rules) | R1; R2 (PP-036) |
| Epic 6 — Integrations | PP-038–040, PP-053, PP-054 | 007 (ports); 008, 009 (PP-039 T-Sheets); 010 (PP-038 procurement usage); 014 (PP-040 AR handoff); 021 (PP-054 GL) | R1; R2 (PP-054) |
| Epic 7 — Platform, Security & Infrastructure | PP-041–047, PP-049–052, PP-056, PP-057, PP-070 | 002 (identity/security core); 001 (PP-045 baseline, PP-056, PP-057 tokens); 005 (PP-046 storage); 008 (PP-047 spike gate); 012 (PP-045 assembly perf); 013 (PP-043 R1 minimum); 015 (PP-044 migration); 021 (PP-043 full); 003–013 (PP-042 audit usage) | Pre-R1 (PP-047); R1; R2 (PP-043) |
| Epic 8 — Reporting | PP-048, PP-071, PP-072 | 021 (PP-048, PP-071); 022 (PP-072) | R2; R3 |

> **Notes.**
>
> - **PP-042 (Epic 7)** is defined in Feature 002 and used by Features 003–013
>   per §1; the epic map lists it once under 002 (range notation follows §1).
> - The **R1 reporting slice (Feature 016)** is delivered from Epic 3 (PP-018)
>   and Epic 4 (PP-031) PPs, so it does not appear under Epic 8 — Epic 8
>   itself (PP-048, PP-071, PP-072) is R2/R3 scope.
> - Release column reflects the PP-level release assignments: an epic can be
>   satisfied across releases when some of its PPs are R2/R3 (see §2).
