# EF Core — Saving, Transactions & Migrations

Version target: EF Core 10 (LTS) / .NET 10

---

## SaveChanges Patterns

```csharp
// ✅ Unit of work: mutate tracked entities, save once
var order = await context.Orders.Include(o => o.Lines).FirstAsync(o => o.Id == id, ct);
order.Ship(shippedAt);
order.Lines.Add(new OrderLine(...));
await context.SaveChangesAsync(ct);   // single transaction, batched statements

// ❌ SaveChanges inside a loop — one roundtrip + transaction per iteration
foreach (var item in items)
{
    context.Add(item);
    await context.SaveChangesAsync(ct);
}

// ✅ Add all, save once (EF batches inserts automatically)
context.AddRange(items);
await context.SaveChangesAsync(ct);
```

**Rules:**
- One `SaveChangesAsync` per business operation. EF wraps it in a transaction automatically — no explicit transaction needed for a single save.
- Don't call `Update(entity)` on tracked entities — mutation is detected automatically. `Update` marks ALL properties modified.
- For disconnected updates (API PUT), prefer fetch-then-mutate; use `Attach` + set modified properties only when avoiding the read is measured as necessary.

---

## ExecuteUpdate / ExecuteDelete

Bulk set-based writes without loading entities:

```csharp
// ✅ Single UPDATE statement — no entities loaded, no change tracking
await context.Orders
    .Where(o => o.Status == OrderStatus.Draft && o.CreatedAt < cutoff)
    .ExecuteDeleteAsync(ct);

// ✅ Expression-bodied setter chain — keep conditionals inside expressions
await context.Blogs.ExecuteUpdateAsync(s => s
    .SetProperty(b => b.Views, b => b.Views + 1)
    .SetProperty(b => b.Name, b => nameChanged ? newName : b.Name), ct);

// ✅ EF 10: works on complex-type JSON columns too
await context.Customers.ExecuteUpdateAsync(s =>
    s.SetProperty(c => c.Address.City, "Berlin"), ct);
```

**Rules:**
- `ExecuteUpdate`/`ExecuteDelete` bypass the change tracker, interceptors, and concurrency tokens — tracked in-memory entities become stale.
- Don't mix ExecuteUpdate and SaveChanges on the same rows in one operation without a transaction.
- Prefer these over load-modify-save for bulk operations (thousands of rows).

---

## Transactions & Execution Strategies

```csharp
// ✅ Explicit transaction spanning multiple saves / raw SQL
await using var tx = await context.Database.BeginTransactionAsync(ct);
await context.SaveChangesAsync(ct);
await context.Database.ExecuteSqlAsync($"REFRESH MATERIALIZED VIEW order_stats", ct);
await tx.CommitAsync(ct);

// ✅ With EnableRetryOnFailure, user transactions MUST go through the strategy
var strategy = context.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    await using var tx = await context.Database.BeginTransactionAsync(ct);
    // ... work ...
    await tx.CommitAsync(ct);
});
```

**Rules:**
- `BeginTransaction` without the execution strategy throws when retry-on-failure is enabled.
- Keep transactions short — no external HTTP calls inside a transaction scope.

---

## Optimistic Concurrency

```csharp
try
{
    await context.SaveChangesAsync(ct);
}
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    var dbValues = await entry.GetDatabaseValuesAsync(ct);
    if (dbValues is null)
        throw new NotFoundException("Row was deleted by another user.");

    // Choose: fail, client-wins, database-wins, or merge
    entry.OriginalValues.SetValues(dbValues);   // database-wins baseline, then retry
}
```

**Rules:**
- Configure a concurrency token (`IsRowVersion` — `xmin` on PostgreSQL) on aggregates edited concurrently.
- Never silently swallow `DbUpdateConcurrencyException` — surface a 409 or apply a deliberate merge policy.

---

## Migrations Workflow

```bash
dotnet ef migrations add AddOrderShippingAddress
dotnet ef migrations script --idempotent -o migrate.sql   # for review / DBA pipelines
dotnet ef migrations bundle --self-contained -r linux-x64 # single-file executable for CI/CD
```

**Rules:**
- **Never** call `Database.Migrate()`/`EnsureCreated()` at app startup in production — race conditions with multiple instances, and the app needs excessive DB permissions. Apply migrations in the deployment pipeline (bundles or idempotent scripts).
- `EnsureCreated` and migrations are mutually exclusive — `EnsureCreated` bypasses migration history.
- Review every generated migration file — EF can't detect renames (it scaffolds drop+add, losing data). Fix with `migrationBuilder.RenameColumn`.
- Destructive changes (column drops, type narrowing) need an explicit data-preservation step in `Up()`.
- Never edit an already-applied migration; add a new one.
- Migrations are per-provider — keep separate migration sets if targeting both Npgsql and SQL Server.
- Seed static reference data via `UseSeeding`/`UseAsyncSeeding` (EF 9+) or `HasData` in the model for small lookup tables.

---

## Common Anti-Patterns

| Anti-pattern | Severity | Fix |
|---|---|---|
| `Database.Migrate()` on startup in production | High | Migration bundles / idempotent scripts in pipeline |
| `SaveChangesAsync` in a loop | High | Batch with `AddRange`, save once |
| Load-modify-save for bulk updates | Medium | `ExecuteUpdateAsync` |
| `context.Update(entity)` on tracked entities | Medium | Rely on change detection |
| `BeginTransaction` without execution strategy (retries enabled) | High | `CreateExecutionStrategy().ExecuteAsync` |
| Swallowed `DbUpdateConcurrencyException` | High | Explicit resolution policy |
| Edited already-applied migration | Critical | New migration instead |
