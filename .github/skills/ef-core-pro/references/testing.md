# EF Core — Testing

Version target: EF Core 10 (LTS) / .NET 10

---

## Test Double Decision Table

| Approach | Fidelity | Speed | Use for |
|---|---|---|---|
| **Real database (Testcontainers)** | Highest — real SQL, real provider | Medium | Integration tests; anything with raw SQL, JSON columns, provider functions, migrations |
| **SQLite in-memory** | Medium — relational but different SQL dialect | Fast | Lightweight relational tests when Docker is unavailable |
| **Repository fakes / mocked abstraction** | N/A — DB not exercised | Fastest | Pure unit tests of business logic behind a repository interface |
| **EF InMemory provider** | ❌ Low — not relational, no transactions, no raw SQL | Fast | **Avoid** — Microsoft explicitly discourages it |

**Rule of thumb:** test business logic against fakes; test queries and persistence against the real database engine your app uses (PostgreSQL → `PostgreSqlContainer`). See the `testcontainers-dotnet-pro` skill for container specifics.

---

## Testcontainers Integration Test (PostgreSQL)

```csharp
public class DatabaseFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container = new PostgreSqlBuilder()
        .WithImage("postgres:17-alpine")
        .Build();

    public string ConnectionString => _container.GetConnectionString();

    public async Task InitializeAsync()
    {
        await _container.StartAsync();
        await using var context = CreateContext();
        await context.Database.MigrateAsync();   // apply real migrations — also tests them
    }

    public AppDbContext CreateContext() =>
        new(new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(ConnectionString)
            .Options);

    public Task DisposeAsync() => _container.DisposeAsync().AsTask();
}

public class OrderRepositoryTests(DatabaseFixture fixture) : IClassFixture<DatabaseFixture>
{
    [Fact]
    public async Task GetOpenOrders_OnlyReturnsOpenStatus()
    {
        await using var arrange = fixture.CreateContext();
        arrange.AddRange(new Order { Status = OrderStatus.Open },
                         new Order { Status = OrderStatus.Closed });
        await arrange.SaveChangesAsync();

        await using var act = fixture.CreateContext();   // fresh context — no tracking bleed
        var result = await act.Orders.Where(o => o.Status == OrderStatus.Open).ToListAsync();

        result.Should().ContainSingle();
    }
}
```

**Rules:**
- Use **separate context instances** for arrange and act/assert — otherwise the change tracker serves cached entities and hides query bugs.
- Reset state between tests: transaction rollback, [Respawn](https://github.com/jbogard/Respawn) (see `respawn-pro` skill), or per-test schemas. Container-per-test is too slow.
- Apply real migrations in fixtures (`MigrateAsync`) rather than `EnsureCreated` — this continuously validates the migration chain.

---

## SQLite In-Memory (When Docker Isn't an Option)

```csharp
var connection = new SqliteConnection("DataSource=:memory:");
await connection.OpenAsync();               // keep open — DB lives with the connection

var options = new DbContextOptionsBuilder<AppDbContext>()
    .UseSqlite(connection)
    .Options;

await using var context = new AppDbContext(options);
await context.Database.EnsureCreatedAsync(); // migrations are provider-specific; don't reuse Npgsql migrations
```

**SQLite limitations to remember:** no schemas, limited `ALTER TABLE`, different date/decimal handling, no `xmin`-style row versions, many provider-specific functions untranslatable. If the code under test uses any of these — use Testcontainers.

---

## Unit Testing with Repository Fakes

```csharp
// Production code depends on an abstraction, not DbContext
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
}

// Test: NSubstitute fake — no EF involved
var repo = Substitute.For<IOrderRepository>();
repo.GetByIdAsync(42, Arg.Any<CancellationToken>()).Returns(OrderBuilder.Open().WithId(42).Build());

var handler = new ShipOrderCommandHandler(repo);
await handler.Handle(new ShipOrderCommand(42), CancellationToken.None);

await repo.Received(1).AddAsync(Arg.Any<Order>(), Arg.Any<CancellationToken>());
```

**Rules:**
- Never mock `DbContext`/`DbSet<T>` directly — mocking `IQueryable` providers is brittle and doesn't verify SQL translation. Put a repository interface in front, or run the query against a real database.
- LINQ that passes against in-memory fakes may fail SQL translation at runtime — every query shape needs at least one integration test.

---

## Testing Query Translation & Migrations

```csharp
// ✅ Assert a query translates (throws InvalidOperationException if untranslatable)
var sql = context.Orders.Where(o => ids.Contains(o.Id)).ToQueryString();

// ✅ Migration smoke test: full chain up
await context.Database.MigrateAsync();
```

- `ToQueryString()` is also the first tool for debugging generated SQL in tests.
- CI should run the full migration chain against a fresh containerized database on every build.

---

## Common Anti-Patterns

| Anti-pattern | Severity | Fix |
|---|---|---|
| EF InMemory provider for integration tests | High | Testcontainers with real provider |
| Mocking `DbSet<T>` / `IQueryable` | High | Repository abstraction or real DB |
| Shared context across arrange and assert | Medium | Fresh context per phase |
| `EnsureCreated` in fixtures when app uses migrations | Medium | `MigrateAsync` |
| No integration coverage for raw SQL / provider functions | High | Testcontainers test per query |
| Container-per-test lifecycle | Medium | Shared fixture + Respawn/transaction reset |
