# Synthetic Data Strategy — ICE Services Project Lifecycle Platform

| Field | Value |
|---|---|
| Status | BMAD draft v1.0 (2026-09-26) |
| Source | `docs/source/phase-1-requirements-register-v2.2.md` v2.2 (authoritative) |
| Companion | `docs/architecture/architecture.md` §5 (ports), §6–§7 (simulator, config) |
| Rule | **Do not assume production data is available.** All R1 development runs on synthetic fixtures. |

## 1. Purpose & Principles

Production data (T-Sheets, Procurement/Dataverse, HR, AR, documents) is
**not available** during development. Rather than block, we:

1. Define a **stable interface contract** per external system (the port,
   architecture §5).
2. Define a **synthetic fixture format** per system (this document §4).
3. Build a **deterministic simulator adapter** per port that satisfies the
   contract from fixtures.
4. Write **adapter-agnostic contract tests** that run against both the
   simulator and the (later) real adapter — so "does the contract hold?" is
   the same question for both.
5. Provide a **config switch** to flip simulator ↔ real adapter per
   environment with **zero code change**.
6. Record **unresolved authentication & API questions** per system (this
   document §6) so the real adapter is written against known unknowns.

**Principles**
- **Deterministic:** seeded generation; identical inputs → identical outputs,
  so tests are reproducible and packet assembly is re-derivable (PP-014 AC-3).
- **Negative by design:** every fixture set includes failure cases (auth
  expiry, not-found, timeout, unmatched record, inactive employee, failed
  delivery) so error paths are exercised in CI, not just happy paths.
- **Shape-stable, value-synthetic:** fixtures match the *real* record shapes
  (so the contract is honest) but contain obviously synthetic values
  (`SIM-` prefixes, fictional customers/employees) to prevent confusion with
  real data.
- **SIM-visible in UI:** any record produced by a simulator carries a `SIM`
  chip (UX spec §4.6) so no one mistakes synthetic billing numbers for real.

## 2. Layout

```
fixtures/                          # source-of-truth synthetic data (JSON)
  identity/users.json
  timekeeping/jobs.json
  timekeeping/subcodes.json
  timekeeping/timesheets.json
  procurement/purchase-orders.json
  procurement/received-goods.json
  procurement/vendors.json
  employees/employees.json
  documents/seed-documents.json
  ar-handoff/target-config.json
  masters/customers.json
  masters/departments.json
  masters/labor-categories.json
  masters/project-types.json
  masters/om-codes.json
  masters/closeout-register.json

sim-state/                         # runtime simulator state (gitignored)
  timekeeping/procurement/employees/documents/ar-handoff/notifications/

simulator/                         # the sim adapters (one per port)
  SimIdentityProvider, SimTimekeepingService, SimProcurementService,
  SimEmployeeDirectory, SimDocumentStore, SimARHandoff,
  AuditServiceDb (fixed sink), SimNotificationService
  Simulators (facade: seed + reset)

contract-tests/                    # adapter-agnostic suite, parameterized per port
  identity.test, timekeeping.test, procurement.test, employee.test,
  document.test, arhandoff.test, audit.test, notification.test
```

- `fixtures/` is **committed** (it is the data contract).
- `sim-state/` is **ephemeral** (gitignored); `Simulators.reset()` restores it.
- `simulator/` + `contract-tests/` are **committed** (they are the test
  harness + adapters).

## 3. Adapter Matrix (all 8 ports)

| Port | Real adapter | Simulator adapter | Fixture file(s) | Config key |
|---|---|---|---|---|
| IdentityProvider | Entra OIDC (D-007) | `SimIdentityProvider` | `identity/users.json` | `identity.provider` |
| TimekeepingService | T-Sheets REST | `SimTimekeepingService` | `timekeeping/*.json` | `timekeeping.provider` |
| ProcurementService | Dataverse OData | `SimProcurementService` | `procurement/*.json` | `procurement.provider` |
| EmployeeDirectory | HR source (DISC-006) | `SimEmployeeDirectory` | `employees/employees.json` | `employee.provider` |
| DocumentStore | Azure Blob / SharePoint (D-002) | `SimDocumentStore` | `documents/seed-documents.json` | `documents.provider` |
| ARHandoffService | folder/portal/email (D-020) | `SimARHandoff` | `ar-handoff/target-config.json` | `arhandoff.provider` |
| AuditService | DB append-only sink (fixed) | `SimAudit` (test harness only) | — | `audit.sink` |
| NotificationService | in-app + SMTP relay | `SimNotificationService` | — | `notifications.provider` |

> Audit is the exception: production **always** uses the DB sink. The sim
> audit exists only for isolated unit tests and is **forbidden in prod**
> (startup validation).

## 4. Synthetic Fixture Formats

> Schemas below are the **contract shapes** (they mirror the real system
> records). Values are synthetic. All timestamps ISO-8601 UTC. Money in
> integer minor units + currency.

### 4.1 Identity — `identity/users.json`
```json
{
  "users": [
    { "subject": "SIM-user-001", "name": "Sim NS Admin", "email": "sim.nsa@sim.ice",
      "externalId": "entra-sim-001", "mfaEnrolled": true,
      "roles": ["NS_ADMIN", "SYSTEM_ADMIN"] },
    { "subject": "SIM-user-002", "name": "Sim Ops Accounting", "email": "sim.ops@sim.ice",
      "externalId": "entra-sim-002", "mfaEnrolled": true, "roles": ["OPS_ACCOUNTING"] },
    { "subject": "SIM-user-003", "name": "Sim PM", "email": "sim.pm@sim.ice",
      "externalId": "entra-sim-003", "mfaEnrolled": false, "roles": ["PROJECT_MANAGER"] }
  ],
  "negative": {
    "expiredToken": "sim-expired-token",
    "unknownSubject": "SIM-user-404",
    "mfaPendingUser": "SIM-user-003"
  }
}
```

### 4.2 Timekeeping — `timekeeping/*.json`
`jobs.json`
```json
{ "jobs": [
  { "jobRef": "SIM-job-1001", "sowNumber": "SOW-1001", "description": "Sim Hard Dollar build",
    "clientRef": "SIM-cust-01", "active": true }
] }
```
`subcodes.json`
```json
{ "subcodes": [
  { "subCodeRef": "SIM-sub-1001-01", "jobRef": "SIM-job-1001", "trade": "Electrical",
    "roleLevel": "Journeyman", "employeeIds": ["SIM-emp-01", "SIM-emp-02"] },
  { "subCodeRef": "SIM-sub-1001-02", "jobRef": "SIM-job-1001", "trade": "Mechanical",
    "roleLevel": "Apprentice", "employeeIds": ["SIM-emp-03"] }
] }
```
`timesheets.json` (deterministic per SOW/discipline/date-range)
```json
{ "timesheets": [
  { "sourceId": "SIM-ts-0001", "employeeId": "SIM-emp-01", "subCodeRef": "SIM-sub-1001-01",
    "date": "2026-09-01", "startTs": "2026-09-01T06:30:00Z", "endTs": "2026-09-01T15:15:00Z",
    "hours": 8.75, "overtimeHours": 0.0 },
  { "sourceId": "SIM-ts-0002", "employeeId": "SIM-emp-01", "subCodeRef": "SIM-sub-1001-01",
    "date": "2026-09-01", "startTs": null, "endTs": null, "hours": 9.5, "overtimeHours": 1.5 }
] }
```
> Note one entry has `null` timestamps — this is the **negative case** for the
> DISC-001 unknown (timestamp availability) and exercises the OT min-rule
> degradation path (architecture §8.2).

### 4.3 Procurement — `procurement/*.json`
`purchase-orders.json`
```json
{ "purchaseOrders": [
  { "poNumber": "SIM-PO-5001", "status": "APPROVED", "vendor": "SIM-vendor-01",
    "sowNumber": "SOW-1001", "projectIdentifier": null, "totalCost": 1250000,
    "currency": "USD",
    "lines": [ { "lineId": "SIM-pol-5001-1", "description": "Copper conduit",
      "qty": 40, "unitCost": 31250, "receivedQty": 40 } ] },
  { "poNumber": "SIM-PO-5002", "status": "APPROVED", "vendor": "SIM-vendor-02",
    "sowNumber": null, "projectIdentifier": null, "totalCost": 800000, "currency": "USD",
    "lines": [ { "lineId": "SIM-pol-5002-1", "description": "Unmatched material",
      "qty": 10, "unitCost": 80000, "receivedQty": 10 } ] }
] }
```
> `SIM-PO-5002` has `sowNumber: null` — the **negative case** for the DISC-003
> matching unknown (PO not linked to SOW) that must surface as a PP-026
> exception, not be silently included or dropped.

`received-goods.json` / `vendors.json` follow the same shape (received goods
reference `poNumber`+`lineId`; vendors carry `vendorId`, `name`, `terms`).

### 4.4 EmployeeDirectory — `employees/employees.json`
```json
{ "employees": [
  { "employeeId": "SIM-emp-01", "name": "Sim Journeyman Elec", "email": "sim.e1@sim.ice",
    "department": "SIM-dept-field", "jobTitle": "Electrician J1", "active": true },
  { "employeeId": "SIM-emp-04", "name": "Sim Former Mech", "email": "sim.e4@sim.ice",
    "department": "SIM-dept-field", "jobTitle": "Mechanic A2", "active": false }
] }
```
> `SIM-emp-04` is **inactive** — negative case: cannot be newly assigned
> (PP-053 AC-5) but remains on historical data.

### 4.5 Documents — `documents/seed-documents.json`
```json
{ "documents": [
  { "ref": "SIM-doc-1001-est", "sowId": "SOW-1001", "docType": "APPROVED_ESTIMATE",
    "fileName": "sim-estimate.pdf", "mimeType": "application/pdf", "size": 48213,
    "checksum": "sha256:sim…", "uploadedBy": "SIM-user-001" },
  { "ref": "SIM-doc-1001-signed", "sowId": "SOW-1001", "docType": "SIGNED_COPY",
    "fileName": "sim-signed.pdf", "mimeType": "application/pdf", "size": 22001,
    "checksum": "sha256:sim…", "uploadedBy": "SIM-user-003" }
] }
```
> `SIGNED_COPY` is a first-class doc type (PP-068 note) so closeout validation
> has something to require/miss.

### 4.6 AR Handoff — `ar-handoff/target-config.json`
```json
{ "target": { "mode": "folder", "path": "sim-state/ar-handout-inbox/",
  "naming": "{sowNumber}_{date}_{stage}.zip" } }
```

### 4.7 Masters (seed, PP-033…037, PP-019, PP-068)
`masters/project-types.json` (drives PP-037/PP-004/PP-068)
```json
{ "projectTypes": [
  { "code": "HARD_DOLLER", "billingModel": "LUMP_SUM", "otRuleSet": "DEFAULT",
    "packetFormat": "STD", "requiredBudgetCategories": ["LABOR","MATERIALS","EQUIPMENT","SUBS"],
    "requiredCloseoutDocs": ["SIGNED_COPY","APPROVED_ESTIMATE"] },
  { "code": "T_AND_M", "billingModel": "TIME_MATERIALS", "otRuleSet": "DEFAULT",
    "packetFormat": "TM_ACTUALS", "requiredBudgetCategories": ["LABOR","MATERIALS"],
    "requiredCloseoutDocs": ["SIGNED_COPY","APPROVED_ESTIMATE","ACTUALS_SUMMARY"] },
  { "code": "OMR", "billingModel": "OMR", "otRuleSet": "DEFAULT", "packetFormat": "OMR_ACTUALS",
    "requiredBudgetCategories": ["LABOR","MATERIALS","SUBS"],
    "requiredCloseoutDocs": ["SIGNED_COPY","APPROVED_ESTIMATE","ACTUALS_SUMMARY"] },
  { "code": "CAMP", "billingModel": "CAMP", "otRuleSet": "DEFAULT", "packetFormat": "STD",
    "requiredBudgetCategories": ["LABOR","MATERIALS","EQUIPMENT","SUBS"],
    "requiredCloseoutDocs": ["SIGNED_COPY"] }
] }
```
> Seed values are **proposed** (D-005) and must be confirmed by NSA/OPA — they
> are synthetic until the real configuration is supplied.

`masters/labor-categories.json` (PP-035, two-level; seed from DISC-007)
```json
{ "laborCategories": [
  { "trade": "Electrical", "roles": [ { "roleLevel": "Journeyman" }, { "roleLevel": "Apprentice" } ] },
  { "trade": "Mechanical", "roles": [ { "roleLevel": "J1" }, { "roleLevel": "A2" } ] }
] }
```
`masters/customers.json`, `departments.json`, `om-codes.json`,
`closeout-register.json` follow the same "seed + `SIM-` values + one negative
case" pattern.

## 5. Contract Tests (per port)

The suite is **adapter-agnostic**: each test file takes a `Port` under test and
asserts the contract. It runs against the **simulator** in CI and against the
**real** adapter in the DEV/dry-run environment (when credentials exist,
D-033). Same assertions, different adapter.

| Port | Contract assertions (minimum) |
|---|---|
| IdentityProvider | auth round-trip; `validateSession` known→context, expired→`SessionExpired`, unknown→`SessionExpired`; role mapping stable; MFA challenge→verify flow |
| TimekeepingService | `createJob` idempotent (same `sowNumber`+`externalRef` → same `jobRef`); sub-code create→read-back; `assignEmployee` visible; `deactivateJob` blocks new `timesheet` entries; `getTimesheetEntries` filters by subCode/employee/range and is deterministic; error mapping (auth→`AdapterAuthError`, notFound→`AdapterNotFound`, timeout→`AdapterTimeout`, all retryable-classed) |
| ProcurementService | `getPurchaseOrders` by `sowNumber` returns only matches; **unmatched PO returns with `sowNumber=null`** (not dropped); PO status enum stable; `getReceivedGoods`/`getVendor`/`getCostData` shapes; error mapping |
| EmployeeDirectory | `getEmployee` known→record, unknown→`AdapterNotFound`; `activeOnly` filter excludes inactive; search deterministic; project-role independence (never returns project roles) |
| DocumentStore | store→open round-trip (content matches checksum); `list`/`search` by sowId/docType/date; concurrent store of same name → unique refs; metadata integrity |
| ARHandoffService | `deliver`→`deliveryStatus` QUEUED→DELIVERED; retry after FAILED; **bundle integrity** (all listed docs present in destination); handoff record immutable |
| AuditService | record→query round-trip; **immutability** (no API mutates/deletes rows); `export` completeness; `correlationId` joins audit ↔ activity log (PP-022) |
| NotificationService | `send`→`deliveryStatus` round-trip; recipient resolution via IdentityProvider; `createTask` visible in queue; email templating deterministic |

**Shared error contract:** every port returns `Result<T>`; adapter errors are
mapped to a closed enum (`AdapterAuthError | AdapterNotFound | AdapterTimeout |
AdapterRateLimited | AdapterOther`) with an `isRetryable` flag. Business code
reacts to this enum, never to vendor exceptions (this is the testable core of
PP-016).

**Architecture test (CI gate):** fail the build if any `domain/**` file imports
a vendor package or a `simulator/**` type (PP-016 isolation).

## 6. Unresolved Authentication & API Questions (per system)

> These are the **open questions** that the real adapter must answer before it
> can be marked done. They are cross-referenced to DISC items; none are assumed.

### 6.1 Identity (Entra) — DISC: — / D-007, D-027, D-028
- Exact tenant + app registration (client id, redirect URIs).
- MFA/Conditional Access method per ICE policy (D-028).
- Whether AIG Portal vs Entra groups is the role source (D-007).
- Session timeout policy (D-027).
- Whether AR/Finance are external identities or internal (affects token flow).

### 6.2 Timekeeping (T-Sheets) — DISC-001 🔴 / D-010, D-011
- **Sub-code creation via API — unconfirmed** (the 🔴 risk).
- Employee restriction per sub-code — unconfirmed.
- Time-entry **start/end timestamps — unconfirmed** (blocks OT precision).
- Auth model (API key / OAuth / basic), rate limits, pagination.
- Activate/deactivate semantics; behavior on **reopen** (D-011).
- Sandbox credentials + environment (D-033).
- Error/response shapes (for the shared error contract mapping).

### 6.3 Procurement (Dataverse) — DISC-003 / D-018
- Stable entity set + field names (PO, received goods, stock issues, vendor).
- **Which field carries the SOW/job number** for matching (D-018) — if absent,
  CSV-import fallback (release-plan contingency).
- Auth: service principal (app-only) vs delegated; which is allowed by IT.
- Which departments are fully on the Power App (coverage of PP-024 data).
- OData pagination / `$filter` support; rate limits.

### 6.4 EmployeeDirectory (HR) — DISC-006 / D-023
- Authoritative source (Paylocity / Workday / local) — **unconfirmed**.
- Read API + auth; data freshness SLA.
- Whether department master (PP-034) is derived from HR or maintained locally
  (D-023).

### 6.5 DocumentStore — D-002 / D-030
- Blob vs SharePoint final choice (D-002).
- Auth for scoped URLs (Entra vs SAS); large-file limits.
- Retention policy (D-030).
- If SharePoint: can library metadata be reused or must be mirrored?

### 6.6 ARHandoff — DISC-013 / D-020
- Mechanism (folder / portal / email / S2S) — **unconfirmed** (D-020).
- Target location, naming convention, credentials.
- Fire-and-forget vs read-back confirmation.
- Whether AR expects a specific packet format (ties to PP-028 filter, D-019).

### 6.7 NotificationService — D-021
- SMTP relay address + sender identity; SPF/DKIM requirements.
- Recipient lists per event (configurable in R2).
- Bounce/undeliverable handling.

## 7. Switching Simulator ↔ Real Adapter

- Config key per port (architecture §7); select at **environment** level.
  - DEV: all simulators allowed (fast, deterministic CI).
  - DRY-RUN: real identity + real DB + **sim external adapters** + real doc
    store (cutover rehearsal, PP-044).
  - PROD: real adapters only; `Sim*` external adapters **forbidden** (startup
    validation fails the boot).
- Runtime switch (S-84, admin action) is allowed in DEV/DRY-RUN and is
  **audit-logged** (PP-042) as a config-change event.
- Because the contract tests run against both, a real adapter that "passes the
  contract" is expected to be drop-in — the business code is unchanged by the
  switch (this is the whole point of PP-016).

## 8. What Synthetic Data Does NOT Cover

- It does **not** validate real data volumes/performance (PP-045) — that needs
  the DRY-RUN environment with real data (D-026/DISC-015).
- It does **not** confirm real API capabilities — that is DISC-001/003/006 and
  the real-adapter verification (D-033).
- It does **not** replace the migration (PP-044) — migration uses the *real*
  source files (DISC-012), not these fixtures.
