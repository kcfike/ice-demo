# ADR-002 — Document Storage Backend (D-002)

| Field | Value |
|---|---|
| Decision ID | D-002 |
| Status | **DECIDED FOR DEVELOPMENT** |
| Owner | IT |
| Release | R1 |
| Blocks | Feature 005 (Document Management); affects Feature 001 foundation |
| Related ADRs | ADR-001, ADR-003 |
| Requirements | PP-006, PP-046 (NFR-DOC) |
| Date | 2026-09-26 |

## 1. Context

All user-uploaded and system-generated documents (PP-006, PP-046) must be
stored or linked in a controlled repository, linked to the SOW, and
searchable by SOW number, date, and type. Access is RBAC-scoped (PP-041).

The architecture already defines the `DocumentStore` port (architecture §5.4)
with a config switch `documents.provider = blob | sharepoint | sim`. The
decision here is which **real adapter** the production path uses. The simulator
adapter (`SimDocumentStore`) is fixed by the architecture and is unaffected by
this decision — it always exists for development and CI.

Constraints that matter most here:
- No production credentials or data available (constraint 2) → the real
  adapter cannot be built or tested until credentials arrive (D-033).
- Design must support replacing simulators with real adapters (constraint 4).
- Avoid infrastructure unnecessary for Phase 1 (constraint 11).
- The port is the only access path (PP-046 AC-4).

## 2. Options Considered

### Option A — Azure Blob Storage (recommended in existing artifacts)

Object storage with Entra-scoped (or SAS) URLs behind `DocumentStore`.

| Constraint | Assessment |
|---|---|
| 2. No production creds | Real adapter deferred; simulator carries dev/CI |
| 4. Replace sim → real | Same port; `documents.provider = blob` |
| 8. Entra production | Native: Entra-scoped SAS URLs, RBAC via role assignment |
| 10. Simple | No metadata-mapping complexity; content-addressed objects |
| 11. Avoid unnecessary infra | Blob is the minimal managed object store; no library/metadata schema to build |

**Strengths:** simplest object model; first-class Entra integration; cheap;
content-addressing + checksums map directly to the port contract; no need to
mirror library metadata; scales without re-architecture.

**Weaknesses:** does not reuse any existing SharePoint library metadata (if the
client's team currently curates packets in SharePoint, that metadata is not
automatically available); requires a container + role setup.

### Option B — SharePoint (existing curation location)

Store/linked documents in a SharePoint document library; mirror metadata to
the SOW.

| Constraint | Assessment |
|---|---|
| 4. Replace sim → real | Same port; `documents.provider = sharepoint` |
| 10. Simple | More complex: graph API, permission model, metadata mapping |
| 11. Avoid unnecessary infra | Adds a graph-permission + metadata-mapping surface for Phase 1 |

**Strengths:** reuses the location the field team already uses; native
permissions; familiar to end users.

**Weaknesses:** metadata must be mirrored or mapped (architecture §5.4 flags
this as an open question); graph auth is more involved than Blob SAS; slower
to integrate and to test; higher surface area for Phase 1.

### Option C — Local filesystem only

Not viable for the production path (no RBAC, no central access), but this is
exactly what `SimDocumentStore` does (`sim-state/documents/`) — and it is
correct for dev/CI. It is not a candidate for the production adapter.

## 3. Evaluation Summary

| Criterion | Option A (Blob) | Option B (SharePoint) |
|---|---|---|
| Simplicity (constraint 10) | **High** | Moderate |
| Entra integration (constraint 8) | **Native (SAS/Entra URLs)** | Good (graph) |
| Avoid extra infra (constraint 11) | **Minimal** | Adds metadata-mapping surface |
| Reuses existing SharePoint metadata | No | **Yes** |
| Phase 1 effort to integrate + test | **Lower** | Higher |
| Replace sim → real (constraint 4) | Same port | Same port |
| Cost (demo/Phase 1) | **Low** | Low |

## 4. Decision

**Select Option A — Azure Blob Storage behind `DocumentStore`, with
Entra-scoped (or SAS) URLs, as the production adapter.**

The `DocumentStore` port (architecture §5.4) is the only access path. The
SharePoint adapter remains a valid **alternate implementation of the same
port** and can be enabled later by flipping `documents.provider` if IT/OPS
prefer to keep curation in SharePoint. The port contract, contract tests,
checksum/content-addressing, and RBAC scoping are identical either way.

## 5. Practical Consequences

1. **Development/CI:** `SimDocumentStore` (local filesystem under
   `sim-state/documents/`, content-addressed checksums) is used. No Azure
   resources are needed to develop or test Feature 005. This satisfies
   constraint 2 (no production creds) and constraint 11 (no unnecessary infra).
2. **Production:** `BlobDocumentStore` adapter (a real adapter, written against
   the port contract first, verified once credentials arrive — D-033). It
   stores objects in a container and returns Entra-scoped/SAS URLs; metadata
   (docType, uploadedBy, checksum) is either in the object metadata or in a
   small DB table — the port abstracts which.
3. **RBAC:** access scoped per SOW via the app's RBAC (PP-041) issuing
   short-lived, SOW-scoped URLs (architecture §9). Blob container policy is
   the backstop; the app is the authority.
4. **Config:** `documents.provider = blob` (prod) / `sim` (dev/CI);
   `documents.container = …`. The startup validation that forbids `Sim*`
   document adapters in prod (architecture §7) is unchanged.
5. **No sequencing impact:** D-002 does not change feature order. Feature 005
   still depends on 002 + 003; Feature 001 scaffolds the port + simulator.
6. **Traceability preserved:** PP-006 / PP-046 / NFR-DOC remain mapped to
   Feature 005 with D-002 as the governing decision; now D-002 is decided
   (development default), so those rows resolve to a concrete default while
   the SharePoint alternative stays recorded.

## 6. Client / IT Confirmation Items

| Item | Who | Status |
|---|---|---|
| Azure Blob acceptable for production document store (vs SharePoint mandate) | IT/OPS | **Pending confirmation** — if SharePoint is mandated, flip `documents.provider = sharepoint`; port and contract tests unchanged |
| Blob container, account, and Entra role/scope model | IT | **Pending** — needed when the real adapter is verified (D-033) |
| Retention policy for stored documents | IT | **Pending** — tracked under D-030 (audit/doc retention) |
| Whether existing SharePoint library metadata must be reused or mirrored | OPS | **Pending** — if "reuse" is required, Option B may be re-evaluated at adapter-verification time |
| Large-file limit and supported MIME types for billing packets (PDF) | IT | **Pending** — confirm against PP-028 packet sizes |

## 7. Development Default vs Production Approach

| Aspect | Development default | Production approach |
|---|---|---|
| Adapter | `SimDocumentStore` (local FS, `sim-state/documents/`) | `BlobDocumentStore` (Azure Blob) |
| Config | `documents.provider = sim` | `documents.provider = blob` |
| Access control | RBAC in app; sim URLs local | RBAC in app + Entra-scoped/SAS URLs |
| Credentials | None required | Entra service identity / connection (D-033) |
| Alternative | — | `documents.provider = sharepoint` (same port) if IT/OPS mandate it |

> **Key distinction:** the development default uses the simulator (no cloud).
> The production approach uses Azure Blob behind the identical port. Choosing
> Blob now is a **development-default decision with a production intent**; it
> is not a claim that Blob is the final production system of record if IT later
> mandates SharePoint — the port keeps that switch available.

## 8. Relationship to Existing Artifacts

- **`architecture.md` §5.4:** unchanged; the "Real adapter: Azure Blob (R1
  recommendation, D-002)" note is now confirmed as the decided default.
- **`decision-register.md` D-002:** status changes from `Proposed` to
  **`DECIDED FOR DEVELOPMENT`**.
- **`synthetic-data-strategy.md` §3, §6.5:** unchanged; `SimDocumentStore`
  remains the dev/CI adapter; §6.5 "Blob vs SharePoint final choice" is now
  answered (Blob default, SharePoint alternate).
- **`requirements-traceability.md`:** PP-006 / PP-046 rows keep D-002; now
  resolved to a concrete development default.
