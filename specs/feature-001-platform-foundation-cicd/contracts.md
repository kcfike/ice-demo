# Feature 001 — Contracts & Interfaces

> Feature 001 (Platform Foundation & CI/CD). These contracts are the
> **shared foundation**: the result/error model, the eight port interfaces,
> the configuration schema, and the `SimIdentityProvider` reference
> simulator. They are normative for Feature 002 onward — later features
> implement against these interfaces, not around them.
>
> Language mapping: **C# / .NET 10 LTS** (ADR-006). Ports are C# interfaces
> in `src/ICE.ProjectLifecycle.Domain/Ports/`; all methods are `async`. Each
> method declares its result type individually: value-producing operations
> return `Task<Result<TOutput>>`, valueless commands return `Task<Result>`
> (§1). Adapters never throw across the port boundary — expected business
> and adapter outcomes are encoded as `Err`/`Err<T>` values, never as
> exceptions.
> Source of truth for shapes: `docs/architecture/architecture.md` §5,
> `docs/data/synthetic-data-strategy.md` §4–§5.

## 1. Shared result & error model (`src/ICE.ProjectLifecycle.Domain/Contracts/`)

There are **two independent result families** — `Result<T>` does **not**
derive from `Result`:

- **`Result`** — commands whose successful execution produces **no value**
  (e.g. `UnassignEmployeeAsync`, `ActivateJobAsync`, `RecordAsync`,
  `DeleteAsync`): success is `Ok`, failure is `Err`.
- **`Result<T>`** — operations whose successful execution **produces a value**
  (e.g. `ValidateSessionAsync`, `GetUserAsync`, `CreateJobAsync`): success is
  `Ok<T>`, failure is `Err<T>`. The generic parameter always matches the
  method's documented successful output (architecture §5), so consumers read
  `Ok<T>.Value` directly — no narrowing or casting of a shared base.

Adapters **never throw across the port boundary** — expected business and
adapter outcomes are mapped to the closed `AdapterError` enum inside
`Err`/`Err<T>` at the adapter edge (synthetic-data-strategy §5).

```csharp
// AdapterError.cs
namespace ICE.ProjectLifecycle.Domain.Contracts;

/// <summary>Closed error taxonomy. Business code reacts to this, never to vendor exceptions.</summary>
public enum AdapterError
{
    Auth = 0,          // AdapterAuthError
    NotFound = 1,      // AdapterNotFound
    Timeout = 2,       // AdapterTimeout
    RateLimited = 3,   // AdapterRateLimited
    Other = 4,         // AdapterOther
}

public static class AdapterErrorExtensions
{
    private static readonly HashSet<AdapterError> _retryable =
        new() { AdapterError.Timeout, AdapterError.RateLimited };

    public static bool IsRetryable(this AdapterError code) => _retryable.Contains(code);
}
```

```csharp
// Result.cs
namespace ICE.ProjectLifecycle.Domain.Contracts;

// ── Valueless family ──────────────────────────────────────────────────

/// <summary>Result of a valueless port command (success: `Ok`; failure: `Err`).</summary>
public abstract record Result
{
    public abstract bool IsRetryable { get; }
}

/// <summary>Valueless success.</summary>
public sealed record Ok : Result
{
    public override bool IsRetryable => false;
}

/// <summary>Failure of a valueless command.</summary>
public sealed record Err(
    AdapterError Code,
    string Message = "",
    IReadOnlyDictionary<string, string> Detail = default!) : Result
{
    public override bool IsRetryable => Code.IsRetryable();
}

// ── Value-bearing family (independent of `Result`) ────────────────────

/// <summary>Result of a value-producing port operation (success: `Ok&lt;T&gt;`; failure: `Err&lt;T&gt;`).</summary>
public abstract record Result<T>
{
    public abstract bool IsRetryable { get; }
}

/// <summary>Success carrying the operation's value.</summary>
public sealed record Ok<T>(T Value) : Result<T>
{
    public override bool IsRetryable => false;
}

/// <summary>
/// Failure of a value-producing operation. `T` matches the method's
/// documented successful output type, so the error is typed at the call site.
/// </summary>
public sealed record Err<T>(
    AdapterError Code,
    string Message = "",
    IReadOnlyDictionary<string, string> Detail = default!) : Result<T>
{
    public override bool IsRetryable => Code.IsRetryable();
}
```

> **Usage:** each port method declares its result type individually —
> value-producing methods return `Task<Result<TOutput>>`, valueless commands
> return `Task<Result>` (exact signatures in §2). Callers pattern-match on
> the declared type: `if (result is Ok<AuthSession> ok) { … } else if
> (result is Err<AuthSession> err) { … }`. Expected failures are
> `Err`/`Err<T>` values, never exceptions.

```csharp
// AdapterHealth.cs
namespace ICE.ProjectLifecycle.Domain.Contracts;

public sealed record AdapterHealth(
    string Status,       // "healthy" | "degraded" | "unavailable"
    int LatencyMs,
    string Message = "")
{
    public const string Healthy = "healthy";
    public const string Degraded = "degraded";
    public const string Unavailable = "unavailable";
}
```

Rules:
- `IsRetryable` is derived from the enum (`Timeout`, `RateLimited` are
  retryable; `Auth`, `NotFound`, `Other` are not) — never set ad hoc.
- `Detail` carries adapter-specific diagnostics (e.g. HTTP status, vendor
  code) without leaking vendor types into the domain.
- The two families are **independent**: `Result<T>` does not derive from
  `Result`. A valueless command's result cannot be read as `Result<T>` and
  vice versa — the type system enforces the match per method.
- Expected business/adapter outcomes are **never exceptions**: failures are
  `Err` (valueless) or `Err<T>` (value-producing) values.

### Adapter health (the ports that define it)

See `AdapterHealth` above. The ports that define `HealthCheckAsync`
(`ITimekeepingService`, `IProcurementService`, `IDocumentStore`,
`IEmployeeDirectory`, `IARHandoffService` — architecture §5.2–§5.6) return
`Task<Result<AdapterHealth>>` (success `Ok<AdapterHealth>`, failure
`Err<AdapterHealth>`). `IIdentityProvider`, `IAuditService`, and
`INotificationService` define **no** health-check method (architecture
§5.1, §5.7, §5.8) — method sets follow architecture §5 per port, not a
uniform template.

## 2. Port interfaces (`src/ICE.ProjectLifecycle.Domain/Ports/`)

All methods `async`. Domain value objects used by ports live in
`src/ICE.ProjectLifecycle.Domain/ValueObjects/` (they are *port contract
types*, not framework types).

### 2.1 IIdentityProvider (`IIdentityProvider.cs`) — PP-049/050/051

```csharp
// ValueObjects/IdentityUser.cs
public sealed record IdentityUser(
    string Subject,
    string Name,
    string Email,
    string ExternalId,
    bool MfaEnrolled);

// ValueObjects/Role.cs
public sealed record Role(string Value);

public static class Roles
{
    public static readonly Role NSAdmin = new("NS_ADMIN");
    public static readonly Role OpsAccounting = new("OPS_ACCOUNTING");
    public static readonly Role ProjectManager = new("PROJECT_MANAGER");
    public static readonly Role Executive = new("EXECUTIVE");
    public static readonly Role AR = new("AR");
    public static readonly Role FinanceFpa = new("FINANCE_FPA");
    public static readonly Role SystemAdmin = new("SYSTEM_ADMIN");
}

// ValueObjects/LoginRequest.cs
public sealed record LoginRequest(string IdentityToken, string? MfaCode = null);

// ValueObjects/AuthSession.cs
public sealed record AuthSession(string Subject, string SessionToken, IReadOnlyList<Role> Roles);

// ValueObjects/IdentityContext.cs
public sealed record IdentityContext(IdentityUser User, IReadOnlyList<Role> Roles);

// ValueObjects/MfaChallenge.cs
public sealed record MfaChallenge(string ChallengeId, string Method);

public interface IIdentityProvider
{
    Task<Result<AuthSession>> AuthenticateAsync(LoginRequest request, CancellationToken ct = default);
    Task<Result<IdentityContext>> ValidateSessionAsync(string token, CancellationToken ct = default);
    Task<Result<IdentityUser>> GetUserAsync(string userSubject, CancellationToken ct = default);
    Task<Result<IReadOnlyList<Role>>> MapRolesAsync(IdentityUser identity, CancellationToken ct = default);
    Task<Result<MfaChallenge>> BeginMfaAsync(LoginRequest request, CancellationToken ct = default);
}
```

Error mapping (typed results; synthetic-data-strategy §5):
- `ValidateSessionAsync` success → `Ok<IdentityContext>`; unknown or expired
token → `Err<IdentityContext>(AdapterError.Auth)` — the `SessionExpired`
semantic (contract: unknown/expired token → `SessionExpired`).
- `GetUserAsync` with an unknown subject → `Err<IdentityUser>(AdapterError.NotFound)`
— a separate contract test, not a replacement for the
`ValidateSessionAsync` behavior above.
- `BeginMfaAsync` for a user with no pending MFA →
`Err<MfaChallenge>(AdapterError.Other)`.
- `AuthenticateAsync` failure (bad credentials/MFA code) →
`Err<AuthSession>(AdapterError.Auth)`.

> **001 vs 002 split:** 001 ships this interface + `SimIdentityProvider` +
> the `identity.provider` config switch (ADR-003 §5.2). The *real* Entra
> adapter, RBAC enforcement (D-009), session timeout (D-27), and MFA method
> (D-028) are **Feature 002** — out of scope here.

### 2.2 ITimekeepingService — PP-010–014, PP-039

```csharp
public interface ITimekeepingService
{
    Task<Result<JobRef>> CreateJobAsync(CreateJobRequest req, CancellationToken ct = default);
    Task<Result<SubCodeRef>> CreateSubCodeAsync(CreateSubCodeRequest req, CancellationToken ct = default);
    Task<Result<AssignmentRef>> AssignEmployeeAsync(JobRef jobRef, SubCodeRef subCodeRef,
        string employeeId, DateTime effectiveFrom, CancellationToken ct = default);
    Task<Result> UnassignEmployeeAsync(AssignmentRef assignmentRef,
        DateTime effectiveFrom, CancellationToken ct = default);   // valueless
    Task<Result> ActivateJobAsync(JobRef jobRef, CancellationToken ct = default);          // valueless
    Task<Result> DeactivateJobAsync(JobRef jobRef, CancellationToken ct = default);        // valueless
    Task<Result<IReadOnlyList<TimesheetEntry>>> GetTimesheetEntriesAsync(
        TimesheetQuery q, CancellationToken ct = default);
    Task<Result<AdapterHealth>> HealthCheckAsync(CancellationToken ct = default);
}
```

Value objects: `CreateJobRequest{SowNumber, Description, ClientRef,
ExternalRef?}`, `CreateSubCodeRequest`, `JobRef`, `SubCodeRef`,
`AssignmentRef`,
`TimesheetQuery{JobRef?|SubCodeRef?, EmployeeId?, From, To}`,
`TimesheetEntry{EmployeeId, SubCodeRef, Date, StartTs?, EndTs?, Hours,
SourceId}` (architecture §5.2).

Valueless commands (`UnassignEmployeeAsync`, `ActivateJobAsync`,
`DeactivateJobAsync` → architecture §5.2 `-> Ok`) return `Task<Result>`:
success `Ok`, failure `Err`. Failures of the value-producing methods use the
corresponding `Err<T>` (e.g. `Err<JobRef>(AdapterError.Auth)`).

### 2.3 IProcurementService — PP-038

```csharp
public interface IProcurementService
{
    Task<Result<IReadOnlyList<PO>>> GetPurchaseOrdersAsync(POQuery q, CancellationToken ct = default);
    Task<Result<IReadOnlyList<ReceivedGood>>> GetReceivedGoodsAsync(ReceivedGoodQuery q, CancellationToken ct = default);
    Task<Result<IReadOnlyList<StockIssue>>> GetStockIssuesAsync(StockIssueQuery q, CancellationToken ct = default);
    Task<Result<Vendor>> GetVendorAsync(string vendorId, CancellationToken ct = default);
    Task<Result<IReadOnlyList<CostLine>>> GetCostDataAsync(CostDataQuery q, CancellationToken ct = default);
    Task<Result<AdapterHealth>> HealthCheckAsync(CancellationToken ct = default);
}
```

`PO{PoNumber, Status, Vendor, Lines[POLine], SowNumber?,
ProjectIdentifier?, TotalCost}`; queries per architecture §5.3:
`POQuery{SowNumber?, ProjectIdentifier?, Status?}`,
`ReceivedGoodQuery{PoId?, SowNumber?, From, To}`,
`StockIssueQuery{SowNumber, From, To}`, `CostDataQuery{SowNumber}`.
Failures use the corresponding `Err<T>` (e.g. `Err<Vendor>(AdapterError.NotFound)`).

### 2.4 IDocumentStore — PP-006/046

```csharp
public interface IDocumentStore
{
    Task<Result<DocumentRef>> StoreAsync(StoredDocument doc, CancellationToken ct = default);
    Task<Result<DocumentStream>> OpenAsync(DocumentRef docRef, CancellationToken ct = default);
    Task<Result> DeleteAsync(DocumentRef docRef, CancellationToken ct = default);   // valueless; soft-delete preferred
    Task<Result<IReadOnlyList<DocumentMeta>>> ListAsync(DocumentQuery q, CancellationToken ct = default);
    Task<Result<IReadOnlyList<DocumentMeta>>> SearchAsync(DocumentQuery q, CancellationToken ct = default);  // PP-046
    Task<Result<AdapterHealth>> HealthCheckAsync(CancellationToken ct = default);
}
```

`StoredDocument{SowId, DocType, FileName, MimeType, Size, UploadedBy,
Tags[]}`, `DocumentMeta{Ref, SowId, DocType, UploadedAt, Size, Checksum,
Url}`, `DocumentStream(Stream Content, string MimeType, long Length)`,
`DocumentQuery{SowId, DocType?, From?, To?}` (architecture §5.4).
Failures use the corresponding `Err<T>` (e.g. `Err<DocumentMeta[]>(AdapterError.NotFound)` → `Err<IReadOnlyList<DocumentMeta>>`).

### 2.5 IEmployeeDirectory — PP-053

```csharp
public interface IEmployeeDirectory
{
    Task<Result<Employee>> GetEmployeeAsync(string employeeId, CancellationToken ct = default);
    Task<Result<IReadOnlyList<Employee>>> SearchAsync(EmployeeQuery q, CancellationToken ct = default);
    Task<Result<IReadOnlyList<Employee>>> ListDepartmentAsync(string departmentId, CancellationToken ct = default);
    Task<Result<AdapterHealth>> HealthCheckAsync(CancellationToken ct = default);
}
```

`Employee{EmployeeId, Name, Email, Department, JobTitle, Active}`;
`EmployeeQuery{Name?, Email?, Department?, ActiveOnly?}` (architecture §5.5).
Unknown ID → `Err<Employee>(AdapterError.NotFound)` (architecture §5.5
contract test).

### 2.6 IARHandoffService — PP-029/040

```csharp
public interface IARHandoffService
{
    Task<Result<HandoffResult>> DeliverAsync(HandoffBundle bundle, CancellationToken ct = default);
    Task<Result<HandoffStatus>> DeliveryStatusAsync(string handoffId, CancellationToken ct = default);
    Task<Result<IReadOnlyList<HandoffRecord>>> ListHandoffsAsync(HandoffQuery q, CancellationToken ct = default);
    Task<Result<AdapterHealth>> HealthCheckAsync(CancellationToken ct = default);
}
```

`HandoffBundle{SowId, PacketPdfRef, SupportingDocs[DocumentRef],
InvoiceDraftRef?, Metadata}`, `HandoffResult{HandoffId, Status,
DeliveredAt?, Error?}`, `HandoffStatus ∈ {QUEUED, DELIVERED, FAILED}`
(architecture §5.6). Failures use the corresponding `Err<T>`.

### 2.7 IAuditService — PP-042/070

```csharp
public interface IAuditService
{
    Task<Result> RecordAsync(AuditEvent e, CancellationToken ct = default);   // valueless; same UoW as the change
    Task<Result<AuditPage>> QueryAsync(AuditQuery q, CancellationToken ct = default);
    Task<Result<ExportJob>> ExportAsync(AuditQuery q, CancellationToken ct = default);
}
```

`RecordAsync` → `Ok` on success, `Err` on failure (architecture §5.7 `-> Ok`).
`AuditEvent{Actor, ActorType(USER|SYSTEM), Entity, EntityId, Field?,
OldValue?, NewValue?, Action, Reason?, CorrelationId, AtUtc}`.
**Production sink is always the DB** (synthetic strategy §3 note); the sim
audit exists only for isolated unit tests and is forbidden in prod.

### 2.8 INotificationService — PP-043(min)/052/069

```csharp
public interface INotificationService
{
    Task<Result<DeliveryRef>> SendAsync(Notification n, CancellationToken ct = default);
    Task<Result<TaskRef>> CreateTaskAsync(WorkflowTask t, CancellationToken ct = default);
    Task<Result<DeliveryStatus>> DeliveryStatusAsync(DeliveryRef deliveryRef, CancellationToken ct = default);
}
```

All three methods are value-producing (architecture §5.8):
success `Ok<T>`, failure `Err<T>` (e.g. `Err<DeliveryRef>(AdapterError.Timeout)`).

## 3. Configuration schema (`config/` + `src/ICE.ProjectLifecycle.Application/Configuration/`)

Single JSON config + env-var overrides for secrets (architecture §7).
Provider selection is **per-adapter, per-environment**.

```jsonc
// config/appsettings.Development.json (committed; all sim; no secrets)
{
  "environment": "dev",
  "identity":      { "provider": "sim", "tenant": "" },
  "timekeeping":   { "provider": "sim", "baseUrl": "", "apiVersion": "" },
  "procurement":   { "provider": "sim", "endpoint": "", "environment": "" },
  "employee":      { "provider": "sim", "source": "" },
  "documents":     { "provider": "sim", "container": "" },
  "arhandoff":     { "provider": "sim", "target": "" },
  "audit":         { "sink": "db", "retention": "" },
  "notifications": { "provider": "sim", "relay": "" },
  "database":      { "connectionString": "Data Source=./dev.db" }
}
```

Rules (enforced by startup validation in
`src/ICE.ProjectLifecycle.Application/Configuration/ConfigValidator.cs`):
- `environment == "prod"` ⇒ **every** provider must be non-`sim`
  (synthetic strategy §7; architecture §7). Violation throws at boot and
  fails CI if triggered.
- `environment == "dryrun"` ⇒ identity real, DB real, external adapters sim
  allowed, documents real (ADR-004 §4.4).
- Secrets (tokens, connection strings) come from the secret store / env vars,
  **never** from the config file (architecture §7).
- Dev/CI default config (`appsettings.Development.json`): all providers
  `sim`, `audit.sink = db`, SQLite connection string.

## 4. SimIdentityProvider contract
(`src/ICE.ProjectLifecycle.Infrastructure/Simulators/SimIdentityProvider.cs`)

Reference simulator (ADR-003 §5.2, synthetic strategy §4.1, §5);
all results are the port's typed results:
- Seeded from `fixtures/identity/users.json` (committed).
- `AuthenticateAsync`: known subject → `Ok<AuthSession>` with fixture roles
  (auth round-trip); failure → `Err<AuthSession>(AdapterError.Auth)`.
- `ValidateSessionAsync`: valid token → `Ok<IdentityContext>`;
  `negative.expiredToken` → `Err<IdentityContext>(AdapterError.Auth)`
  (`SessionExpired` semantic);
  `negative.unknownSubject` → `Err<IdentityContext>(AdapterError.Auth)`
  (`SessionExpired` semantic).
- `BeginMfaAsync`: `negative.mfaPendingUser` → `Ok<MfaChallenge>` (method
  `"simulated"`), followed by the MFA verification flow — `AuthenticateAsync`
  with the MFA code → `Ok<AuthSession>`; other users →
  `Err<MfaChallenge>(AdapterError.Other, "mfa not required")`.
- `GetUserAsync`: known subject → `Ok<IdentityUser>`; unknown subject →
  `Err<IdentityUser>(AdapterError.NotFound)` (a separate contract test; it
  does **not** replace the `ValidateSessionAsync` behaviors above).
- `MapRolesAsync`: stable — same `IdentityUser` → same role list across
  runs (`Ok<IReadOnlyList<Role>>`).
- Deterministic: identical inputs → identical outputs (synthetic strategy §1).
- Runtime state written to `sim-state/identity/` (gitignored);
  `Simulators.Reset()` restores it.
- **No network, no clock dependence** (seeded/deterministic).

## 5. Simulators facade
(`src/ICE.ProjectLifecycle.Infrastructure/Simulators/Simulators.cs`)

```csharp
public static class Simulators
{
    public static void Seed(string fixturesDir) { /* fixtures/ -> sim-state/ */ }
    public static void Reset() { /* clear sim-state/, re-seed */ }
}
```

001 ships the facade + `SimIdentityProvider` only. The remaining 7 sim
adapters are **Feature 007** (feature-sequence §5 finding 6 — do not collapse
007 into 001). The facade's `Seed`/`Reset` are defined now so 007's adapters
plug in without redesign.

## 6. HTTP surface (001 minimum)

| Route | Method | Purpose |
|---|---|---|
| `/health` | GET | liveness + `environment` + provider summary (no secrets) |
| `/api/version` | GET | app version + git SHA (from build metadata) |

No other routes in 001. All later endpoints land with their owning features.

## 7. Frontend tokens contract (`brand/tokens.json`)

`src/ICE.ProjectLifecycle.Web/src/styles/tokens.ts` imports
`../../../brand/tokens.json`.
Placeholder values are **obviously non-brand** (`#000000`-style, font
`"placeholder"`). When DISC-016 delivers real brand assets, only
`brand/tokens.json` changes — no code change (PP-057 AC-2 path, UX spec §2).
