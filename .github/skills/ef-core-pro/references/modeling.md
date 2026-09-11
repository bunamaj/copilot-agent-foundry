# EF Core — Entity Modeling

Version target: EF Core 10 (LTS) / .NET 10

---

## Configuration Style

Prefer `IEntityTypeConfiguration<T>` classes over a monolithic `OnModelCreating`:

```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("orders");
        builder.HasKey(o => o.Id);

        builder.Property(o => o.Number)
            .HasMaxLength(32)
            .IsRequired();

        builder.HasIndex(o => o.Number).IsUnique();

        builder.HasMany(o => o.Lines)
            .WithOne(l => l.Order)
            .HasForeignKey(l => l.OrderId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}

// In DbContext:
protected override void OnModelCreating(ModelBuilder modelBuilder)
    => modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
```

**Rules:**
- One configuration class per aggregate/entity, discovered via `ApplyConfigurationsFromAssembly`.
- Always set `HasMaxLength` on string properties — unbounded `nvarchar(max)`/`text` columns hurt indexing.
- Define indexes in the model (`HasIndex`) so migrations own the schema.

---

## Complex Types vs Owned Types (EF 10)

**Complex types are now the recommended choice** for value objects — owned entity types have identity/reference semantics that cause subtle bugs.

```csharp
// ✅ EF 10: complex type — value semantics, supports ExecuteUpdate, struct support
public class Customer
{
    public int Id { get; set; }
    public required Address ShippingAddress { get; set; }
    public Address? BillingAddress { get; set; }   // optional complex types: new in EF 10
}

modelBuilder.Entity<Customer>(b =>
{
    b.ComplexProperty(c => c.ShippingAddress);              // table splitting (columns)
    b.ComplexProperty(c => c.BillingAddress, c => c.ToJson()); // or JSON column
});

// ✅ Assignment copies values — works as expected with complex types
customer.BillingAddress = customer.ShippingAddress; // fine
// ❌ Same code with owned types throws (identity conflict)
```

| Aspect | Complex type | Owned entity type |
|---|---|---|
| Semantics | Value | Reference/identity |
| Assign same instance twice | ✅ Copies values | ❌ Throws |
| `ExecuteUpdate` support | ✅ | ❌ |
| LINQ comparison | By contents | By identity |
| Collections | JSON mapping only; no struct collections | ✅ Supported |
| Recommendation | **Default choice** | Only when a collection of dependents with keys is needed |

---

## JSON Columns

```csharp
// ✅ Map complex type (or collection) to a JSON column
modelBuilder.Entity<Blog>().ComplexProperty(b => b.Details, d => d.ToJson());

// Query into JSON transparently:
var popular = await context.Blogs.Where(b => b.Details.Viewers > 3).ToListAsync();

// Bulk update inside JSON (EF 10, complex types only):
await context.Blogs.ExecuteUpdateAsync(s =>
    s.SetProperty(b => b.Details.Views, b => b.Details.Views + 1));
```

**Rules:**
- Primitive collections (`string[] Tags`) are mapped to JSON automatically since EF 8.
- On PostgreSQL, JSON maps to `jsonb`; use `EF.Functions.JsonContains` etc. for provider-specific operators.
- Use JSON mapping for document-shaped data read/written as a unit; use columns (table splitting) when properties are queried/indexed individually.

---

## Value Converters & Comparers

```csharp
// ✅ Strongly-typed ID
builder.Property(o => o.Id)
    .HasConversion(id => id.Value, value => new OrderId(value));

// ✅ Enum as string (readable, but index-size tradeoff)
builder.Property(o => o.Status)
    .HasConversion<string>()
    .HasMaxLength(20);
```

**Rules:**
- Value-converted properties can't be translated into SQL beyond equality — avoid converters on columns needing range/LIKE queries.
- Mutable converted types (e.g. collections) need a `ValueComparer` or change tracking misses mutations.

---

## Global Query Filters (Named — EF 10)

```csharp
// ✅ EF 10: multiple named filters per entity
modelBuilder.Entity<Order>()
    .HasQueryFilter("SoftDelete", o => !o.IsDeleted)
    .HasQueryFilter("Tenant", o => o.TenantId == _tenantId);

// Selectively disable one filter
var all = await context.Orders.IgnoreQueryFilters(["SoftDelete"]).ToListAsync();
```

**Rules:**
- Reference DbContext fields (not locals) in filters so per-instance state (tenant ID) works.
- Navigations to filtered entities in required relationships can silently drop rows — make such relationships optional or filter both sides.
- `IgnoreQueryFilters()` (no args) disables ALL filters — prefer named disabling in EF 10.

---

## Relationships & Delete Behavior

```csharp
builder.HasMany(o => o.Lines)
    .WithOne(l => l.Order)
    .HasForeignKey(l => l.OrderId)
    .IsRequired()
    .OnDelete(DeleteBehavior.Cascade);    // dependents die with principal

builder.HasOne(o => o.Customer)
    .WithMany(c => c.Orders)
    .HasForeignKey(o => o.CustomerId)
    .OnDelete(DeleteBehavior.Restrict);   // prevent accidental cascade across aggregates
```

**Rules:**
- Cascade delete **within** an aggregate; `Restrict`/`NoAction` **across** aggregates.
- Always declare FK properties explicitly (`OrderId`) rather than relying on shadow FKs — clearer and queryable.
- Many-to-many: let EF create the join table implicitly unless the join has payload columns — then model it explicitly.

---

## Concurrency Tokens

```csharp
// PostgreSQL: use xmin system column
builder.Property(o => o.Version).IsRowVersion();  // maps to xmin with Npgsql

// SQL Server: rowversion
public byte[] Version { get; set; }
builder.Property(o => o.Version).IsRowVersion();
```

Handle `DbUpdateConcurrencyException` on save (see saving-migrations.md).

---

## Common Anti-Patterns

| Anti-pattern | Fix |
|---|---|
| All configuration inline in a 500-line `OnModelCreating` | `IEntityTypeConfiguration<T>` per entity |
| Owned types for simple value objects | Complex types (EF 10) |
| Strings without `HasMaxLength` | Explicit lengths |
| Cascade delete across aggregate boundaries | `DeleteBehavior.Restrict` |
| Data annotations + fluent config for the same property (drift) | Fluent API as single source of truth |
