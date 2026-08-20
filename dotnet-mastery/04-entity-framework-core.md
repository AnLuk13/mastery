# 4. Entity Framework Core

Goal: understand EF Core's mental model (DbContext ≈ Prisma Client / TypeORM DataSource, entities ≈ models, migrations ≈ schema migrations you've used before) well enough to read and safely modify `HRNS.Database`.

## 4.1 The big picture

EF Core is a **code-first ORM**: you write C# classes (entities), EF Core maps them to tables, tracks changes to loaded objects in memory, and translates LINQ queries into SQL. If you've used Prisma or TypeORM, the roles map directly:

| EF Core | Prisma / TypeORM equivalent |
|---|---|
| `DbContext` | `PrismaClient` / `DataSource` |
| `DbSet<TEntity>` | `prisma.user` / a TypeORM `Repository<User>` |
| Entity class | Prisma model / TypeORM `@Entity()` class |
| `IEntityTypeConfiguration<T>` (Fluent API) | `schema.prisma` model block / TypeORM decorators |
| Migration | Prisma Migrate / TypeORM migration file |
| Change tracking | TypeORM's dirty-checking on tracked entities (Prisma has no equivalent — Prisma is always "explicit update calls") |

## 4.2 DbContext & DbSet

The `DbContext` is your single entry point to the database — one instance per unit of work (per HTTP request, in a web app: that's exactly why it's registered `Scoped`, see chapter 3). Each `DbSet<TEntity>` property is a queryable, in-code representation of a table.

```csharp
public class HRNSDbContext : DbContext, IHRNSDbContext
{
    public HRNSDbContext(DbContextOptions options) : base(options) { }

    public DbSet<UserEntity> Users { get; set; }
    public DbSet<CompanyAccountEntity> CompanyAccounts { get; set; }
    public DbSet<KnowledgeChunkEntity> KnowledgeChunks { get; set; }
    // ...

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.HasPostgresExtension("vector"); // enables pgvector, see chapter 13
        modelBuilder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
    }
}
```
(`HRNS.Database/DbContext/HrnsysDbContext.cs:17-67`, condensed.)

Two things worth noticing immediately:

- Not every entity gets an explicit `DbSet<T>` property here — most CQRS handlers instead call `_dbContext.Set<TEntity>()` generically (you saw this constantly in chapter 5's preview — `_dbContext.Set<UserEntity>()`). Both access the same underlying table; `Set<T>()` is just the generic-friendly way to reach a table when you only know `T` as a type parameter, which is exactly the situation `QueryBaseHandler<TQuery, TEntity>` is in.
- `ApplyConfigurationsFromAssembly(...)` is the key line: instead of configuring every entity's schema inline inside `OnModelCreating` (which would make this one method thousands of lines long), EF Core scans the assembly for every class implementing `IEntityTypeConfiguration<T>` and applies each automatically. That's what makes the pattern in §4.3 below work without any central registry to maintain.

## 4.3 Fluent API configuration — one class per entity

HRNS does **not** use Data Annotations (`[Required]`, `[MaxLength(50)]` attributes directly on entity properties) — it uses the **Fluent API** exclusively, one `IEntityTypeConfiguration<TEntity>` class per entity, colocated with the entity itself. This keeps the entity class itself a plain, readable POCO (Plain Old CLR Object) and puts all the schema/constraint/relationship logic in one dedicated, easy-to-find place.

Every entity inherits from a shared base:

```csharp
// HRNS.Database/Entities/Entity/Entity.cs
public abstract class Entity
{
    public Guid Id { get; set; }
    public Guid CreatedByUserId { get; set; }
    public UserEntity CreatedByUser { get; set; }
    public Guid? ModifiedByUserId { get; set; }
    public UserEntity ModifiedByUser { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? ModifiedAt { get; set; }
    public bool IsDeleted { get; set; } = false;
    public bool IsConstant { get; set; } = false;
}
```

...and every entity's configuration inherits from a matching base that wires up the shared parts once:

```csharp
// HRNS.Database/Entities/Entity/EntityDbConfig.cs
public abstract class EntityDbConfig<TEntity> : IEntityTypeConfiguration<TEntity> where TEntity : Entity
{
    public virtual void Configure(EntityTypeBuilder<TEntity> builder)
    {
        builder.HasKey(entity => entity.Id);
        builder.HasIndex(entity => entity.CreatedAt);
        builder.HasIndex(entity => entity.ModifiedAt);
        builder.Property(e => e.IsDeleted).HasDefaultValue(false);

        builder.HasOne(entity => entity.CreatedByUser).WithMany().HasForeignKey(entity => entity.CreatedByUserId);
        builder.HasOne(entity => entity.ModifiedByUser).WithMany().HasForeignKey(entity => entity.ModifiedByUserId);

        builder.HasQueryFilter(a => !a.IsDeleted);   // <-- global soft-delete filter, see §4.5
    }
}
```

A concrete entity then only has to declare what's *specific* to it:

```csharp
// entity — plain POCO, one relationship of its own on top of the inherited audit fields
public class TicketingTicketTypeEntity : Entity
{
    public Guid ContractId { get; set; }
    public TicketingContractEntity Contract { get; set; }
    public string Label { get; set; }
    public Guid DefaultAssignmentTeamId { get; set; }
    public TicketingAssignmentTeamEntity DefaultAssignmentTeam { get; set; }
}

// configuration — calls base.Configure() first, then adds what's specific to this entity
public class TicketingTicketTypeEntityDbConfig : EntityDbConfig<TicketingTicketTypeEntity>
{
    public override void Configure(EntityTypeBuilder<TicketingTicketTypeEntity> builder)
    {
        base.Configure(builder);
        builder.HasOne(e => e.Contract).WithMany().HasForeignKey(e => e.ContractId).OnDelete(DeleteBehavior.Restrict);
        builder.HasOne(e => e.DefaultAssignmentTeam).WithMany().HasForeignKey(e => e.DefaultAssignmentTeamId).OnDelete(DeleteBehavior.Restrict);
        builder.ToTable("TicketingTicketTypes");
    }
}
```
(`HRNS.Database/Entities/Ticketing/TicketingTicketType/TicketingTicketTypeEntity.cs` and `...EntityDbConfig.cs`, in full — they're short.)

`DeleteBehavior.Restrict` means "don't let the database cascade-delete this relationship; block the delete instead if a dependent row exists" — deliberate, given nothing in this codebase hard-deletes rows anyway (see §4.5).

## 4.4 Relationships — Fluent API vocabulary

| Fluent API call | Meaning |
|---|---|
| `.HasOne(e => e.Contract)` | this entity has (at most) one related `Contract` |
| `.WithMany()` | ...and `Contract` can have many of this entity pointing back at it (one-to-many). `WithMany()` with no argument means "no navigation property back on the other side" — `Contract` doesn't expose a `List<TicketingTicketType>` collection |
| `.WithOne(...)` | one-to-one instead |
| `.HasForeignKey(e => e.ContractId)` | which scalar property physically stores the FK |
| `.OnDelete(DeleteBehavior.Cascade\|Restrict\|SetNull\|NoAction)` | what happens to this row if the referenced row is deleted |
| `.HasMany(...).WithOne(...)` | declared from the "one" side instead — same relationship, other direction |

## 4.5 Global query filters — how soft delete actually works

`builder.HasQueryFilter(a => !a.IsDeleted)` in `EntityDbConfig<TEntity>` means **every** LINQ query against **every** entity in this database automatically has `WHERE "IsDeleted" = false` appended — silently, everywhere, unless you explicitly opt out. This is how HRNS implements soft delete: rows are never actually `DELETE`d, just flagged, and the ORM makes the flag invisible by default.

To see genuinely deleted-but-still-in-the-database rows, call `.IgnoreQueryFilters()`:

```csharp
// normal query — soft-deleted rows are invisible, automatically
var users = await _dbContext.Set<UserEntity>().Where(u => u.Email == email).ToListAsync();

// explicitly bypass the filter — used e.g. during login, to distinguish
// "no such user" from "user exists but was deleted"
var userEntity = await _dbContext.Set<UserEntity>()
    .IgnoreQueryFilters()
    .FirstOrDefaultAsync(e => e.Email.ToLower().Trim() == email.ToLower().Trim() && !e.IsDeleted);
```
(real usage from `HRNS.Application/CQRS/Users/LoginUserQuery.cs:82-90` — note it re-adds `!e.IsDeleted` manually right after ignoring the filter, because the handler also needs to inspect *locked* accounts which the automatic filter would otherwise hide identically to deleted ones — a subtle but deliberate distinction worth internalizing: `IgnoreQueryFilters()` isn't "show deleted rows," it's "stop applying the automatic filter, and now I decide what's visible.")

## 4.6 Tracking vs. no-tracking queries

By default, every entity EF Core loads is **tracked**: the `DbContext` keeps a snapshot and diffs it against the object's current state when you call `SaveChanges()`, to figure out what SQL `UPDATE` to generate. That's convenient for "load it, mutate it, save it" flows, but costs memory/CPU you don't need for read-only queries.

```csharp
// tracked (default) — EF Core watches this object; mutating + SaveChanges() will UPDATE it
var user = await _dbContext.Set<UserEntity>().FirstOrDefaultAsync(u => u.Id == id);
user.LastLoginAttempt = DateTime.UtcNow;
await _dbContext.SaveChangesAsync(); // generates UPDATE ... SET "LastLoginAttempt" = ...

// no-tracking — read-only, faster, cannot be mutated-and-saved this way
var users = await _dbContext.Set<UserEntity>().AsNoTracking().ToListAsync();
```

HRNS's convention, visible everywhere in `QueryBase.cs`/`CommandBase.cs`: **every read query uses `.AsNoTracking()`**, and writes go through an explicit `_dbContext.Add(entity)` / `_dbContext.Update(entity)` call rather than relying on tracked-mutation-then-save. This is a deliberate, consistent choice — it makes "is this a read or a write" obvious from the code shape alone, and avoids accidentally tracking (and holding in memory) every row a large paged query happens to touch.

## 4.7 Eager loading & the "cartesian explosion" problem

`.Include(e => e.Navigation)` eager-loads a related entity in the same query (SQL `JOIN`) instead of triggering a separate query later (which is the N+1 problem — see chapter 12). Chaining multiple `.Include()`s that each pull in a *collection* navigation, though, multiplies rows: if entity A has 3 related B's and 4 related C's, a single SQL `JOIN` across both produces 3×4 = 12 rows for what should be one A. EF Core calls this **cartesian explosion**, and its fix is `.AsSplitQuery()` — issue one SQL query per `Include()` instead of one giant join, then stitch results together in memory.

```csharp
var userLegalEntity = _dbContext
    .Set<EmployeeToUserEntity>()
    .AsNoTracking()
    .AsSplitQuery() // to eliminate the so-called "cartesian explosion"
    .Where(e => e.UserId == request.UserId)
    .Include(e => e.Employee.LegalEntity.CompanyAccount.AllowedProviders)
    .Include(e => e.Employee.LegalEntity)
    // ...
```
(`HRNS.Application/CQRS/Users/LoginUserQuery.cs:235-259`, condensed — and yes, that comment is verbatim in the real code, which tells you the author hit this exact problem and left the reasoning behind for the next reader.) `AsSplitQuery()` is used consistently across every `QueryBaseHandler`'s base implementation (`StructureItems()` in `CQRS/Base/QueryBase.cs`) — treat it as the default whenever you write a query with more than one `.Include()`.

## 4.8 Migrations

A migration is a versioned, incremental description of a schema change — EF Core's equivalent of a Prisma Migrate/TypeORM/Knex migration file, generated by diffing your current entity model against the last migration's snapshot.

```bash
# from HRNS.Database/ (the project holding the DbContext + Migrations/ folder)
dotnet ef migrations add AddTicketingTicketType   # generates a new file under Migrations/
dotnet ef database update                          # applies pending migrations to the connected DB
```

You'll find the accumulated history in `HRNS.Database/Migrations/` — hundreds of files, one per schema change ever made to this database, each with an `Up()` (apply) and `Down()` (revert) method the tool generates for you (you can hand-edit them, but the generated `Up`/`Down` pair is usually correct as-is for straightforward model changes).

Notably, HRNS applies pending migrations **automatically on app startup** rather than as a separate deploy step:

```csharp
app.ApplyDBContextMigrations(); // HRNS.WebApi/Program.cs:147
```
which — skipping the InMemory-provider guard used for tests — resolves to `dbContext.Database.Migrate()` (`HRNS.WebApi/Extensions/ApplicationBuilderExtension.cs:57-87`). This is a real, debatable architectural choice worth having an opinion on once you've read chapter 11 (DevOps): auto-migrate-on-boot is simple and guarantees the schema is never out of sync with the code that's running, but it also means a bad migration can take the app down at startup in production rather than failing a controlled, separate migration step.

## 4.9 `SaveChanges` — one more real-world wrinkle

`HRNSDbContext` overrides `SaveChangesAsync`/`SaveChanges` to translate a raw PostgreSQL unique-constraint violation into a friendlier exception:

```csharp
public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
{
    try
    {
        return await base.SaveChangesAsync(cancellationToken);
    }
    catch (Exception e)
    {
        if (e.InnerException != null && (e.InnerException.Message.Contains("23505:") || e.InnerException.Message.Contains("duplicate key value")))
            throw new Exception("Duplicate entry already exists");
        throw;
    }
}
```
(`HRNS.Database/DbContext/HrnsysDbContext.cs:74-100`, condensed. `23505` is PostgreSQL's own SQLSTATE error code for `unique_violation` — worth remembering, you'll see raw Postgres error codes surface like this elsewhere too.)

## Checkpoint

1. Pick any entity folder under `HRNS.Database/Entities/` you haven't seen yet. Find its `*EntityDbConfig.cs` and identify every relationship it declares (`HasOne`/`HasMany`) and each `OnDelete` behavior.
2. Explain, in your own words, what would happen if you queried `_dbContext.Set<UserEntity>()` **without** `.IgnoreQueryFilters()` for a user whose `IsDeleted` is `true`.
3. Find one more `.Include(...).Include(...)` chain elsewhere in `HRNS.Application/CQRS/` and confirm whether it also uses `.AsSplitQuery()`. If it doesn't, is it only a single-level `Include` (safe) or a real omission worth flagging?
