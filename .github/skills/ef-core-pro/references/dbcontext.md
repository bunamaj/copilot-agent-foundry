# EF Core — DbContext Lifetime, Registration & Configuration

Version target: EF Core 10 (LTS) / .NET 10

---

## Registration Options

| Method | Lifetime | Use when |
|--------|----------|----------|
| `AddDbContext<T>` | Scoped | Default for web apps — one context per request |
| `AddDbContextPool<T>` | Scoped (pooled instances) | High-throughput apps; avoids per-request context setup cost (~2x faster, far less allocation) |
| `AddDbContextFactory<T>` | Singleton factory | Blazor, background services, parallel work — create/dispose contexts on demand |
| `AddPooledDbContextFactory<T>` | Singleton pooled factory | Factory + pooling combined; needed for pooled multi-tenant patterns |

```csharp
// ✅ Standard web app registration
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("Default")));

// ✅ High-throughput: pooling (poolSize default 1024)
builder.Services.AddDbContextPool<AppDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("Default")));
```

**Rules:**
- A `DbContext` is **not thread-safe**. Never share an instance across threads or parallel tasks. EF throws `InvalidOperationException` when it detects concurrent use.
- Never register `DbContext` as Singleton.
- Never inject `DbContext` into a Singleton service — inject `IDbContextFactory<T>` instead.
- With pooling, `OnConfiguring` runs only once per pooled instance — never use it for per-request state (tenant ID, user ID).

---

## Pooled Contexts with Per-Request State (Multi-Tenancy)

Pooled instances are reused across requests, so mutable state must be injected via a scoped factory wrapper:

```csharp
// 1. Singleton pooled factory
builder.Services.AddPooledDbContextFactory<AppDbContext>(o =>
    o.UseNpgsql(connectionString));

// 2. Scoped factory that stamps tenant state onto each instance
public class TenantScopedFactory(IDbContextFactory<AppDbContext> pooledFactory, ITenant tenant)
    : IDbContextFactory<AppDbContext>
{
    public AppDbContext CreateDbContext()
    {
        var context = pooledFactory.CreateDbContext();
        context.TenantId = tenant?.TenantId ?? -1;
        return context;
    }

    public async Task<AppDbContext> CreateDbContextAsync(CancellationToken cancellationToken = default)
    {
        var context = await pooledFactory.CreateDbContextAsync(cancellationToken);
        context.TenantId = tenant?.TenantId ?? -1;
        return context;
    }
}

// 3. Register scoped factory + scoped context resolution
builder.Services.AddScoped<TenantScopedFactory>();
builder.Services.AddScoped(sp => sp.GetRequiredService<TenantScopedFactory>().CreateDbContext());
```

**Rules:**
- EF resets its own state when returning a context to the pool, but NOT ADO.NET state — if you manually open a `DbConnection`, close it before disposal.
- Custom mutable fields on the context must be reset/re-set on every checkout.

---

## Options Configuration

```csharp
builder.Services.AddDbContextPool<AppDbContext>(options => options
    .UseNpgsql(connectionString, npgsql => npgsql
        .EnableRetryOnFailure(maxRetryCount: 3)      // connection resiliency
        .CommandTimeout(30))
    .UseSnakeCaseNamingConvention()                   // EFCore.NamingConventions for PostgreSQL
    .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking)); // default for read-heavy apps
```

**Rules:**
- Enable `EnableRetryOnFailure` for cloud databases — transient faults are expected.
- With a retrying execution strategy, user-initiated transactions must be wrapped in `strategy.ExecuteAsync(...)` (see saving-migrations.md).
- `EnableSensitiveDataLogging()` and `EnableDetailedErrors()` — development only, NEVER in production (leaks PII into logs).
- EF 10 redacts inlined constants from logs by default (shown as `?`).

---

## Interceptors

```csharp
// ✅ Audit interceptor — set audit fields on save
public class AuditInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result,
        CancellationToken cancellationToken = default)
    {
        foreach (var entry in eventData.Context!.ChangeTracker.Entries<IAuditable>())
        {
            if (entry.State == EntityState.Added)
                entry.Entity.CreatedAt = DateTimeOffset.UtcNow;
            if (entry.State == EntityState.Modified)
                entry.Entity.UpdatedAt = DateTimeOffset.UtcNow;
        }
        return base.SavingChangesAsync(eventData, result, cancellationToken);
    }
}

// Registration
options.AddInterceptors(new AuditInterceptor());
```

**Rules:**
- Interceptors registered via `AddInterceptors` in pooled contexts must be stateless (they are shared).
- Prefer interceptors over overriding `SaveChangesAsync` in the context — composable and testable.

---

## Async Discipline

```csharp
// ✅ Always async for I/O
var order = await context.Orders.FirstOrDefaultAsync(o => o.Id == id, ct);
await context.SaveChangesAsync(ct);

// ❌ Sync-over-async or sync APIs in web apps
var order2 = context.Orders.FirstOrDefault(o => o.Id == id); // blocks thread
```

**Rules:**
- Always use async EF APIs (`ToListAsync`, `SaveChangesAsync`, `FirstOrDefaultAsync`) in server apps.
- Always flow `CancellationToken` from the request.
- Never mix sync and async EF calls in the same code path — thread-pool starvation risk.

---

## Common Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| `DbContext` field in a Singleton service | Inject `IDbContextFactory<T>` |
| Sharing one context across `Task.WhenAll` branches | One context per parallel operation via factory |
| Per-request state set in `OnConfiguring` with pooling | Scoped factory wrapper pattern |
| `EnableSensitiveDataLogging` in production config | Guard with `env.IsDevelopment()` |
| Long-lived context held for the app lifetime (memory leak — tracked entities accumulate) | Short-lived scoped/factory contexts |
