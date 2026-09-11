---
name: ef-core-pro
description: >-
  Comprehensively reviews Entity Framework Core code for best practices on DbContext
  lifetime and pooling, entity modeling (complex types, owned types, value converters,
  JSON columns), LINQ querying (AsNoTracking, split queries, projections, compiled
  queries), change tracking, SaveChanges vs ExecuteUpdate/ExecuteDelete, migrations,
  concurrency tokens, interceptors, global query filters, performance tuning, and
  testing strategies. Use when reading, writing, or reviewing .NET projects that use
  EF Core for data access.
  Trigger keywords: EF Core, Entity Framework, DbContext, DbSet, AddDbContext,
  AddDbContextPool, OnModelCreating, ModelBuilder, IEntityTypeConfiguration,
  migrations, dotnet ef, SaveChanges, SaveChangesAsync, ExecuteUpdate, ExecuteDelete,
  AsNoTracking, AsSplitQuery, Include, ThenInclude, HasQueryFilter, owned entity,
  complex type, value converter, UseNpgsql, UseSqlServer, UseSqlite, FromSql.
  DO NOT USE FOR: Dapper or raw ADO.NET data access, EF6 /
  .NET Framework Entity Framework (use dotnet-migration), database-side SQL tuning
  (use postgres-pro), or test container setup (use testcontainers-dotnet-pro).
---

Review EF Core code for correctness, idiomatic patterns, and performance. Version target: **EF Core 10 (LTS) / .NET 10**.

## Review Process

1. **Load reference files** — identify which reference files apply to the change; read only those files.
2. **Check DbContext setup** — verify registration (`AddDbContext` vs `AddDbContextPool`), lifetime correctness, options configuration, and connection resiliency. See `references/dbcontext.md`.
3. **Check modeling** — verify entity configuration, complex types vs owned types, value converters, JSON mapping, relationships, and indexes. See `references/modeling.md`.
4. **Check queries** — verify tracking behavior, projections, eager loading vs N+1, split queries, pagination, and parameterization. See `references/querying.md`.
5. **Check writes** — verify SaveChanges patterns, ExecuteUpdate/ExecuteDelete usage, transactions, concurrency handling, and migrations discipline. See `references/saving-migrations.md`.
6. **Check testing** — verify test double strategy (SQLite in-memory vs real database vs fakes), fixture patterns, and migration testing. See `references/testing.md`.

## Reference Files

| File | Contents | When to load |
|------|----------|-------------|
| `references/dbcontext.md` | DbContext lifetime, AddDbContext vs AddDbContextPool, pooled factories, multi-tenancy state, thread safety, connection resiliency, interceptors, logging | When reviewing DI registration, context configuration, or startup code |
| `references/modeling.md` | Entity configuration, IEntityTypeConfiguration, complex types vs owned types (EF 10), JSON columns, value converters, relationships, indexes, named query filters, concurrency tokens | When reviewing entity models or OnModelCreating |
| `references/querying.md` | Tracking vs no-tracking, projections, Include/ThenInclude, split queries, N+1 avoidance, pagination, parameterized collections (EF 10), LeftJoin, compiled queries, raw SQL safety | When reviewing LINQ queries or read paths |
| `references/saving-migrations.md` | SaveChanges patterns, ExecuteUpdate/ExecuteDelete, transactions, optimistic concurrency, migrations workflow, migration bundles, idempotent scripts | When reviewing write paths, transactions, or migrations |
| `references/testing.md` | Test double decision table, SQLite in-memory limits, Testcontainers integration, repository fakes, fixture patterns, migration testing | When reviewing or writing tests |

## Output Format

For each finding:

1. **File and line(s)** affected
2. **Rule violated** (e.g. "Read-only query must use `AsNoTracking` to avoid change-tracking overhead")
3. **Severity:** Critical / High / Medium / Low
4. **Before/after code snippet**

End with a summary table of all findings sorted by severity.
