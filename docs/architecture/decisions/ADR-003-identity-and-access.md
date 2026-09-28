# ADR-003 — Identity, Authentication, MFA & Role Assignment (D-007)

| Field | Value |
|---|---|
| Decision ID | D-007 |
| Status | **DECIDED FOR DEVELOPMENT** |
| Owner | IT |
| Release | R1 |
| Blocks | Feature 002 (Identity & Access); scaffolds around Feature 001 |
| Related ADRs | ADR-001, ADR-005 |
| Requirements | PP-041 (RBAC), PP-049 (login/session), PP-050 (MFA), PP-051 (user mgmt), PP-070 (security log) |
| Date | 2026-09-26 |

## 1. Context

The application must support **Entra ID in the production architecture**
(constraint 8) and expose identity, roles, and MFA behind the
`IdentityProvider` port (architecture §5.1) so that development and CI can run
without a live tenant (constraints 2, 4).

Scope of this decision:
- The **authentication mechanism** (how a user proves who they are).
- The **role-assignment mechanism** (how PP-041's seven roles are granted).
- The **MFA/2FA approach** (PP-050) in line with ICE security policy (D-028).
- The **development path** (no production tenant available) vs the production
  path (Entra ID).

Explicitly **out of scope** (left open per the register): the exact RBAC
permission matrix per role (D-009), the session-timeout value (D-027), and the
final MFA method detail per ICE policy (D-028) — these are carried as open
decisions owned by their features. This ADR fixes the **mechanism**, not the
**values**.

## 2. Options Considered

### 2.1 Authentication mechanism

**Option A — Entra ID OIDC (SSO) as the single production auth path.**
OAuth2/OIDC against the ICE tenant; token issued and validated by the app.

**Option B — Local username/password (in-app user store) as primary, Entra
optional.**
App-managed credential store + password reset (PP-049 AC-1 lists a
username/password fallback).

**Option C — AIG Portal delegation.**
Not an authentication mechanism for the app itself; a role-source option (see
2.2). Rejected as a standalone auth path — the app still needs its own
AuthN.

**Assessment:** Entra OIDC is the production requirement (constraint 8) and
the register's own recommendation. A local password store is only acceptable as
a **development convenience / fallback**, not as the production primary. The
architecture's `IdentityProvider` port already abstracts this: `entra` in prod,
`sim` in dev/CI.

### 2.2 Role-assignment mechanism

**Option A — Entra group ↔ role mapping.** Roles derived from Entra group
membership (claims), authoritative in Entra.

**Option B — In-app role table.** App-side table mapping `(Entra subject ↔
role)`, managed through the PP-051 admin UI.

**Option C — Hybrid (recommended): Entra as the identity source (subject),
in-app role table as the assignment source.** Identity provisioning stays in
Entra; role assignment is a first-class app concern managed by System Admin /
NS Admin (D-009 boundary). This keeps RBAC testable against the simulator and
avoids a hard dependency on Entra group naming for every Phase 1 feature.

**Assessment:** Pure Entra-group mapping (Option A) couples business roles to
tenant group naming and is hard to exercise in CI without a tenant. The
register's recommendation ("Entra ID OIDC + in-app role table") is Option C.
Option C keeps the `mapRoles()` port method meaningful, lets the
`SimIdentityProvider` return fixture roles, and preserves auditability of role
changes (PP-042).

### 2.3 MFA / 2FA (PP-050, D-028)

**Option A — Entra Conditional Access enforces MFA** (phishing-resistant
where policy allows); the app consumes the resulting token and does not
implement its own MFA challenge.

**Option B — In-app MFA challenge** (TOTP/app) implemented by the application.

**Assessment:** In a Microsoft-centric shop (product brief §6), pushing MFA to
Entra Conditional Access is the boring, policy-driven, IT-governed choice and
matches the register recommendation. The app's `beginMfa`/`MfaChallenge`
port surface is retained for the **simulator** to exercise the MFA flow in
tests, but in production the MFA decision is enforced upstream by Entra — the
app does not re-implement it. This keeps the app thin and the security policy
in IT's hands (D-028 owns the exact method).

## 3. Evaluation Summary

| Aspect | Dev default | Production |
|---|---|---|
| AuthN | `SimIdentityProvider` (fixture users, `fixtures/identity/users.json`) | Entra ID OIDC (MSAL for .NET) |
| Role source | Fixture `roles[]` per user; in-app role table | In-app role table (subject from Entra) |
| MFA | Simulated challenge/verify in contract tests | Entra Conditional Access (D-028 method) |
| Session | App-issued session token (timeout per D-027) | App-issued session token (timeout per D-027) |
| User mgmt (PP-051) | In-app admin UI over role table | In-app admin UI over role table |

## 4. Decision

**Select: Entra ID OIDC as the production authentication mechanism + an
in-app role table as the role-assignment source (subject from Entra) +
Entra Conditional Access for MFA, all behind the `IdentityProvider` port.**

Concretely:
1. **AuthN:** production path is Entra ID OIDC (SSO). The app validates the
   OIDC token and extracts the stable `subject` + basic identity claims.
2. **AuthZ (roles):** roles are stored in an **in-app role table** keyed by
   the Entra subject, managed via the PP-051 admin UI by System Admin (and NS
   Admin per D-009 boundary). `IdentityProvider.mapRoles()` resolves subject →
   roles. This keeps RBAC testable and decouples business roles from tenant
   group naming.
3. **MFA:** enforced by **Entra Conditional Access** (method per D-028 / ICE
   policy). The app does not implement its own MFA in production; the
   `beginMfa`/`MfaChallenge` surface exists for the simulator to test the flow.
4. **Development/CI:** `SimIdentityProvider` returns fixture users + roles and
   supports the MFA challenge/verify and "force failure" paths so all
   acceptance criteria (PP-041, PP-049, PP-050, PP-051, PP-070) can be
   exercised without a tenant.

## 5. Practical Consequences

1. **Feature 002** builds the `IdentityProvider` real adapter (Entra OIDC via
   MSAL for .NET) + the in-app role table + user admin UI + security-event
   logging (PP-070). It is testable end-to-end against the simulator first.
2. **Feature 001** scaffolds the port + `SimIdentityProvider` + config switch
   `identity.provider = entra | sim` so identity is swappable (constraint 4).
3. **No tenant required for dev/CI:** satisfies constraint 2 (no production
   creds) and constraint 7 (Ralph can test the full identity path against the
   simulator).
4. **RBAC matrix (D-009), session timeout (D-027), and exact MFA method
   (D-028) remain open** — this decision fixes the *mechanism* and explicitly
   does not set those *values*, so no unrelated decisions are resolved.
5. **Audit:** role changes and login/security events flow through
   `AuditService` (PP-042) and the security log (PP-070), both already
   port-defined.
6. **Sequencing:** unchanged. Feature 002 still depends on 001; Feature 003
   still depends on 002.

## 6. Client / IT Confirmation Items

| Item | Who | Status |
|---|---|---|
| Entra tenant + app registration (client id, redirect URIs) | IT | **Pending** — needed to build/verify the real adapter (D-033) |
| Exact RBAC permission matrix (which role may do which action) | PM/OPA | **Open — D-009** (not resolved here) |
| Whether AR / Finance are internal identities or external (affects token flow) | IT | **Open** (register note under D-007) — affects token flow, not the mechanism |
| MFA method per ICE security policy | IT | **Open — D-028** (mechanism fixed to Entra CA; method value open) |
| Session timeout duration | IT | **Open — D-027** (value) |
| AIG Portal vs Entra groups as a secondary role source | IT | **Open** — in-app role table is the R1 default; AIG Portal can feed it later |

## 7. Development Default vs Production Approach

| Aspect | Development default | Production approach |
|---|---|---|
| Config | `identity.provider = sim` | `identity.provider = entra` |
| Identity source | `fixtures/identity/users.json` | ICE Entra tenant (OIDC) |
| Roles | Fixture roles + in-app role table | In-app role table (subject from Entra) |
| MFA | Simulated challenge/verify (contract tests) | Entra Conditional Access (D-028) |
| User mgmt | In-app admin UI | In-app admin UI (unchanged) |
| Security log | `AuditService` (DB sink) | `AuditService` (DB sink, retention D-030) |

> **Key distinction:** the development default uses the simulator (no tenant).
> The production approach uses Entra ID OIDC with in-app role assignment and
> Entra Conditional Access for MFA. The `IdentityProvider` port is the seam:
> identical interface, different adapter.

## 8. Relationship to Existing Artifacts

- **`architecture.md` §5.1, §8.1, §9:** unchanged; the "Real adapter: Entra
  ID (OAuth2/OIDC) per D-007" and "AuthN: IdentityProvider (Entra OIDC
  preferred, D-007)" notes are now confirmed.
- **`decision-register.md` D-007:** status changes from `Proposed` to
  **`DECIDED FOR DEVELOPMENT`**. D-009, D-027, D-028 remain open (values, not
  mechanism).
- **`synthetic-data-strategy.md` §6.1:** the identity questions (tenant, MFA
  method, role source) remain the real-adapter's open items; the mechanism is
  now decided.
- **`requirements-traceability.md`:** PP-041 / PP-049 / PP-050 / PP-051 /
  PP-070 rows keep D-007 (now decided for development); D-009 / D-027 / D-028
  stay open where listed.
