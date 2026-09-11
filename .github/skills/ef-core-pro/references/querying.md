# EF Core — Querying

Version target: EF Core 10 (LTS) / .NET 10

---

## Tracking vs No-Tracking

```csharp
// ✅ Read-only query — no change-tracking overhead (~30% faster, less allocation)
var orders = await context.Orders.AsNoTracking()
    .Where(o => o.Status == OrderStatus.Open)
    .ToListAsync(ct);

// ✅ Query that will be modified and saved — keep tracking
var order = await context.Orders.FirstAsync(o => o.Id == id, ct);
order.Status = OrderStatus.Shipped;
await context.SaveChangesAsync(ct);
```

**Rules:**
- Every read-only query uses `AsNoTracking()` — or set `UseQueryTrackingBehavior(NoTracking)` context-wide and opt into tracking with `AsTracking()`.
- `AsNoTracking` disables identity resolution: the same row appearing twice materializes twice. Use `AsNoTrackingWithIdentityResolution()` if that matters.
- Projections to DTOs (`Select(x => new Dto {...})`) are never tracked — no `AsNoTracking` needed.

---

## Project Only What You Need

```csharp
// ❌ Fetches every column, tracks entities, then discards most data
var names = (await context.Products.ToListAsync()).Select(p => p.Name);

// ✅ SELECT name FROM products
var names2 = await context.Products.Select(p => p.Name).ToListAsync(ct);

// ✅ DTO projection including related data — single SQL statement, no Include needed
var summaries = await context.Orders
    .Where(o => o.CustomerId == customerId)
    .Select(o => new OrderSummaryDto(o.Id, o.Number, o.Lines.Sum(l => l.Total)))
    .ToListAsync(ct);
```

**Rules:**
- `Include` is for loading tracked entity graphs you'll modify. For read models, project with `Select` instead.
- Never call `ToList()` mid-query and continue with LINQ-to-Objects — filtering/joining must stay in SQL.

---

## Eager Loading & N+1

```csharp
// ❌ N+1: one query per order
foreach (var order in await context.Orders.ToListAsync())
    Console.WriteLine(order.Lines.Count);   // lazy-load per iteration

// ✅ Eager load
var orders = await context.Orders.Include(o => o.Lines).ToListAsync(ct);

// ✅ Filtered include
var recent = await context.Orders
    .Include(o => o.Lines.Where(l => l.Quantity > 0).OrderBy(l => l.Position))
    .ToListAsync(ct);
```

**Rules:**
- Avoid lazy-loading proxies in web apps — they invite N+1 and fail after context disposal.
- Multiple collection `Include`s produce a cartesian explosion in single-query mode — use `AsSplitQuery()`:

```csharp
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Contributors)
    .AsSplitQuery()          // one SQL statement per collection; EF 10 fixed ordering consistency
    .ToListAsync(ct);
```

- Split queries trade one roundtrip for consistency risk (no shared snapshot) — wrap in a transaction if consistency matters.

---

## Pagination

```csharp
// ⚠️ Offset pagination — degrades on deep pages, unstable with concurrent inserts
var page = await query.OrderBy(o => o.Id).Skip(page * size).Take(size).ToListAsync(ct);

// ✅ Keyset pagination — constant cost at any depth
var page2 = await context.Orders
    .OrderBy(o => o.Id)
    .Where(o => o.Id > lastSeenId)
    .Take(size)
    .ToListAsync(ct);
```

Always `OrderBy` a unique column before paginating — otherwise page contents are nondeterministic.

---

## Parameterized Collections (EF 10)

`Where(x => ids.Contains(x.Id))` now translates to individual scalar parameters (padded buckets) by default — cache-friendly and sniffable.

```csharp
// Default (EF 10): WHERE id IN (@p0, @p1, @p2, @p2)  — padded parameters
var found = await context.Orders.Where(o => ids.Contains(o.Id)).ToListAsync(ct);

// Force constant inlining for a specific query (e.g. to hit an index differently)
var found2 = await context.Orders.Where(o => EF.Constant(ids).Contains(o.Id)).ToListAsync(ct);

// Or globally:
options.UseNpgsql(cs, o => o.UseParameterizedCollectionMode(ParameterTranslationMode.Constant));
```

---

## LeftJoin / RightJoin (.NET 10)

```csharp
// ✅ First-class left join — no more GroupJoin/SelectMany/DefaultIfEmpty ceremony
var result = await context.Students
    .LeftJoin(context.Departments,
        s => s.DepartmentId, d => d.Id,
        (s, d) => new { s.Name, Department = d != null ? d.Name : "[none]" })
    .ToListAsync(ct);
```

---

## Compiled Queries

For extremely hot query paths (called thousands of times/sec):

```csharp
private static readonly Func<AppDbContext, int, CancellationToken, Task<Order?>> GetOrderById =
    EF.CompileAsyncQuery((AppDbContext ctx, int id, CancellationToken ct) =>
        ctx.Orders.FirstOrDefault(o => o.Id == id));

var order = await GetOrderById(context, id, ct);
```

Compiled queries are thread-safe; store them in `static readonly` fields. Measure first — regular query caching is already fast.

---

## Raw SQL Safety

```csharp
// ✅ FromSql — interpolated values become DbParameters automatically
var user = await context.Users
    .FromSql($"SELECT * FROM users WHERE email = {email}")
    .FirstOrDefaultAsync(ct);

// ❌ SQL injection — string concatenation into FromSqlRaw (EF 10 ships an analyzer that flags this)
var bad = context.Users.FromSqlRaw("SELECT * FROM users WHERE email = '" + email + "'");

// ✅ FromSqlRaw only for dynamic SQL fragments (e.g. column names) — values still parameterized
var ok = context.Users.FromSqlRaw($"SELECT * FROM users ORDER BY {ValidateColumn(col)}");
```

**Rules:**
- Prefer `FromSql` (interpolated, safe) over `FromSqlRaw`.
- Never concatenate user input into `FromSqlRaw` / `ExecuteSqlRaw` — Critical severity finding.
- Raw SQL must return all columns of the entity and can be composed with LINQ afterwards.

---

## Streaming vs Buffering

```csharp
// Buffering — default; fine for bounded result sets
var list = await query.ToListAsync(ct);

// ✅ Streaming for large result sets — constant memory
await foreach (var row in query.AsAsyncEnumerable().WithCancellation(ct))
    Process(row);
```

---

## Common Anti-Patterns

| Anti-pattern | Severity | Fix |
|---|---|---|
| String concatenation in `FromSqlRaw` | Critical | `FromSql` interpolation |
| Read-only queries without `AsNoTracking` | Medium | Add `AsNoTracking` or NoTracking default |
| `ToList()` then in-memory filter/join | High | Keep composition in IQueryable |
| Lazy-loading in loops (N+1) | High | `Include` or projection |
| Multiple collection Includes without `AsSplitQuery` | Medium | Split query |
| Unordered `Skip/Take` | High | Add unique `OrderBy` |
| `Count() > 0` | Low | `AnyAsync()` |
