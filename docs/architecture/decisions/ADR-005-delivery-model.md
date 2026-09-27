# ADR-005 — Staffing & Timeline Assumptions (D-031)

| Field | Value |
|---|---|
| Decision ID | D-031 |
| Status | **DECIDED FOR DEVELOPMENT** |
| Owner | PM |
| Release | R1 (delivery model for R1/R2/R3) |
| Blocks | Scheduling in `release-plan.md`; **not a code blocker** (register §10) |
| Related ADRs | ADR-001, ADR-004 |
| Date | 2026-09-26 |

## 1. Context

D-031 ("Staffing & timeline") governs the assumptions behind the relative
Gantt in `release-plan.md` and the delivery rhythm. The register marks it as
not a code blocker ("needed to schedule; not a code blocker"), and the release
plan already states the timeline is *relative only* pending this decision.

Two structural facts are fixed by the project context, not by this decision:
- **Ralph (an AI implementation agent) performs most implementation and
  automated testing** (constraint 7). This is the dominant delivery capacity.
- **The client/ICE supplies the human inputs** the AI cannot produce: business
  decisions (D-xxx values), discovery inputs (DISC-xxx evidence), credentials
  (D-033), and acceptance sign-off. These are the *critical path* items, and
  most of them are **unavailable at the start of development** (constraints 1,
  2).

So the delivery model is a **two-track model**: a continuous AI
implementation track (Ralph) and a client/IT input track that gates
specific features and the R1 exit. D-031 fixes the assumptions that make this
model schedulable and honest about what is *not* in the team's control.

## 2. Options Considered

### Option A — Single-track, calendar-committed plan
Treat the timeline as a fixed calendar (weeks 1–N to R1 exit) assuming a
constant human+AI throughput.

| Constraint | Assessment |
|---|---|
| 2. No production creds/data | **Fails** — many features/exit gates need client inputs that have no date; a fixed calendar would be false |
| 7. Ralph does most work | Under-weights the client-input dependency (the real critical path) |
| 10/11. Simplicity | A calendar is simpler, but it is *less* reliable given the known unknowns |

### Option B — Two-track, relative, input-gated plan (recommended)
- **Track 1 (AI/Ralph — continuous):** implement + auto-test features in the
  `feature-sequence.md` order against simulators, one at a time. Throughput is
  bounded by Ralph + the CI gate (ADR-004), not by headcount.
- **Track 2 (Client/ICE — gated):** supply decisions (D-xxx values), discovery
  evidence (DISC-xxx), credentials (D-033), and acceptance. These gate
  specific features (008, 010, 015) and the R1 exit.
- **Timeline is relative** (feature-order + gates), not calendar-committed.
  Calendar dates are attached **only when** the client commits to input
  dates. This is exactly what `release-plan.md` already assumes
  ("relative only; calendar dates require the D-031 staffing decision and
  spike outcomes").

| Constraint | Assessment |
|---|---|
| 2. No production creds/data | **Satisfies** — the model does not assume inputs that aren't there |
| 7. Ralph does most work | **Reflects reality** — AI track is the continuous engine; client track is the gate |
| 10. Simple | The model is simple to state and to run |
| 11. Avoid unnecessary infra | No extra delivery machinery beyond the existing BMAD/Spec Kit/Ralph flow |

### Option C — Staffing by headcount (N developers × weeks)
Not applicable: the implementation capacity is an AI agent, not a headcount.
A headcount model would misrepresent both the throughput and the risk (the
risk is client-input latency, not developer count).

**Assessment:** Option B is the only model that is both honest about the
no-production-data reality (constraint 2) and aligned with Ralph-as-primary
implementer (constraint 7). It also preserves the existing "relative timeline"
posture of the release plan, so it is a *confirmation + explicit assumption
set*, not a re-plan.

## 3. Decision — the explicit assumptions

**Select Option B (two-track, relative, input-gated delivery). The fixed
assumptions are:**

1. **Implementation capacity = Ralph (AI agent), primary.** Ralph implements
   one approved Spec Kit feature at a time, in `feature-sequence.md` order,
   validating against simulators + contract tests (ADR-004 gate) before the
   next. Human developers are *reviewers*, not the primary implementers.
2. **Client/ICE is the critical path for gated items.** The R1 exit and
   Features 008/010/015 depend on client inputs that are **not currently
   available** (constraints 1–2). The plan does not assume they will arrive by
   any date; the calendar attaches only when ICE commits.
3. **Relative timeline, not calendar.** Sequencing and dependencies
   (`feature-sequence.md`) are the schedule. Calendar weeks are added
   incrementally as client input dates firm up.
4. **Two 🟥 external items are the schedule's dominant risk** and are tracked
   as the gating inputs, not as "effort":
   - **DISC-001 / D-010 (T-Sheets spike 🔴):** gates Feature 008. Outcome
     (API-driven vs manual-mapping fallback) is unknown until the spike.
   - **DISC-012 / D-025 (migration inventory + cutover 🔴):** gates Feature
     015 and the R1 exit. "Cannot go live without it" (PP-044).
5. **No parallel human delivery teams are assumed.** The plan assumes one
   implementation track (Ralph) plus the client-input track. If ICE later
   commits human developers, the AI track accelerates but the *gates* (2 and
   4) are unchanged.
6. **Effort is expressed in feature-order + gates, not person-days.** A
   person-day estimate would falsely imply the bottleneck is labor; the
   bottleneck is client input latency and the two 🔴 spikes.
7. **Re-baselining rule:** when a gating client input lands (or a 🔴 spike
   resolves), the relative timeline is re-baselined at that point. No silent
   date promises are made before then.

## 4. Practical Consequences

1. **`release-plan.md` Gantt** stays *relative*; this ADR is the authority
   for *why* it is relative and which inputs convert it to a calendar.
2. **Feature order is unchanged** (no sequencing change). D-031 is not a
   code blocker, so Feature 001 can begin as soon as D-001/D-002/D-007/D-029
   are decided (all now decided for development).
3. **The critical path is stated explicitly:** DISC-001 → Feature 008, and
   DISC-012 → Feature 015 → R1 exit. Ralph's continuous track can proceed on
   001→007 and 009–014 (to the extent their own gates are met) while those
   two spikes are outstanding.
4. **Ralph's definition of done** is unchanged: simulator-backed, contract
   tests green, CI green (ADR-004), R1 boundary respected.
5. **Calendar dates:** none are committed in this run. The ADR defines the
   rule for when/ how they get committed (client input dates firm up).
6. **Traceability preserved:** D-031 maps to the release-plan timeline note;
   no PP requirement's traceability changes.

## 5. Client / IT Confirmation Items

| Item | Who | Status |
|---|---|---|
| Commit to input dates for DISC-001 (T-Sheets spike) evidence | PM/ICE | **Pending** — until committed, Feature 008 stays gated and the timeline stays relative |
| Commit to input dates for DISC-012 (migration source files) + D-025 sign-off | PM/OPA/NSA | **Pending** — until committed, the R1 exit date is not set |
| Confirm Ralph-as-primary-implementer is the intended delivery model | PM | **Pending confirmation** — this ADR assumes it (constraint 7) |
| Confirm no calendar commitment is being made in this run | PM | **Pending confirmation** — relative timeline only |
| Any committed human developer capacity that would change the AI-track throughput | PM | **Pending** — if any, the AI track accelerates; gates unchanged |

## 6. Development Default vs Production Approach

D-031 is a **delivery-model** decision, not a technical configuration, so the
"development default vs production" axis maps to *planning posture* vs
*operational reality*:

| Aspect | Planning posture (this run) | Operational reality (when inputs land) |
|---|---|---|
| Timeline | **Relative** (feature-order + gates) | Calendar dates attached as client input dates firm up |
| Critical path | DISC-001→008 and DISC-012→015→R1 exit (stated, not dated) | Dated once ICE commits to the two 🔴 items |
| Implementation | Ralph (AI) primary, humans as reviewers | Same; human devs (if committed) accelerate the AI track |
| Bottleneck | Client-input latency, not labor | Same |
| Exit | R1 exit gated by D-025 sign-off + DISC-012 | Dated at sign-off |

> **Key distinction:** the development/planning default is a *relative,
> input-gated* schedule that makes **no calendar commitment** and does **not**
> assume unavailable client inputs. The operational approach converts it to a
> calendar only when ICE commits to input dates — the assumptions above are
> what make that conversion honest.

## 7. Relationship to Existing Artifacts

- **`release-plan.md` §1 (Gantt note):** confirmed — "relative only; calendar
  dates require the D-031 staffing decision and spike outcomes" is now the
  decided posture, with the explicit assumption set above.
- **`decision-register.md` D-031:** status changes from `Open` to
  **`DECIDED FOR DEVELOPMENT`** (delivery-model assumptions fixed; calendar
  dates explicitly deferred to client input commitments).
- **`feature-sequence.md`:** **unchanged** — D-031 is not a sequencing
  decision and does not change any feature's order or dependencies.
- **`discovery-backlog.md` §3 (critical-path discovery):** consistent —
  DISC-001 and DISC-012 remain the two 🔴 gating items this ADR names as the
  schedule's dominant risk.
