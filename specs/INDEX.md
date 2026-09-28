# Spec Kit — ICE Services Project Lifecycle Platform

Spec Kit is the hand-off layer between the BMAD planning artifacts
(`docs/`) and the implementation agent (Ralph). Each feature gets a
self-contained directory under `specs/` with the artifacts below. This
directory is **documentation only** — Ralph does not write application code
until the human reviewer approves the spec.

## Directory layout

```
specs/
  INDEX.md                          # this file
  feature-NNN-<slug>/               # one directory per feature (NNN = 001…022)
    specification.md                # goals, scope, non-goals, acceptance criteria,
                                    # traceability to PP/D/DISC/ADRs
    plan.md                         # implementation plan, phase order, risks,
                                    # verification loop
    research.md                     # tooling/version pins, assumptions, open questions
                                    # (only if non-trivial choices are needed)
    data-model.md                   # database schema for this feature (only if the
                                    # feature owns domain tables)
    contracts.md                    # port interfaces, shared types, config schema
                                    # (only if the feature defines or extends them)
    tasks.md                        # small Ralph-executable tasks, each with its own
                                    # acceptance criteria + verification command
```

## Conventions

- **Authoritative inputs.** `docs/product/` (PRD, requirements register,
  traceability, feature sequence), `docs/architecture/` (architecture, ADRs,
  decision register), and `docs/data/synthetic-data-strategy.md` are the
  source of truth. A spec may add detail but must not contradict them.
  When a spec needs to resolve an apparent conflict, it says so explicitly
  and flags it for human sign-off — it never silently picks a side.
- **Pending confirmations stay pending.** ADR-006 §7, ADR-002 §6, ADR-003 §6,
  ADR-004 §6, and ADR-005 §5 list client/IT items that are *Pending
  confirmation*. A spec must carry these forward verbatim as open items. It
  must not convert them into confirmed requirements.
- **R1 boundary rule.** Ralph must not pull R2/R3 scope into R1 features
  (feature-sequence §7). A spec's non-goals section is the enforcement point.
- **Simulator-first.** Every port has a simulator adapter and a config switch.
  CI runs against simulators only. Real adapters are written against the
  contract, then verified against a sandbox once credentials arrive (D-033).
- **Task sizing.** Each task in `tasks.md` must be small enough for Ralph to
  execute and verify independently: a bounded set of file changes, its own
  acceptance criteria, and a concrete verification command (a test, a build,
  a `git diff --check`, etc.).
- **Traceability.** Every acceptance criterion in `specification.md` cites the
  PP requirement it satisfies and the ADR/decision that governs it.
  `tasks.md` tasks cite the acceptance criteria they satisfy.

## Feature index

| Feature | Directory | Status |
|---|---|---|
| 001 Platform Foundation & CI/CD | `feature-001-platform-foundation-cicd/` | **Specified** — pending human review |
| 002 Identity & Access | `feature-002-identity-access/` | Not yet specified |
| … | … | … |

> Feature 001 is the first spec written against this convention. If the
> convention needs to be amended, amend it here and in the affected feature
> directory, and note the change in that feature's `plan.md`.
