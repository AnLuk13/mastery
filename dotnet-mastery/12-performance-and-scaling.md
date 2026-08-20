# 12. Performance & Scaling

Goal: the recurring performance pitfalls in async .NET web apps and EF Core specifically, tied back to patterns you've already seen (and one honest gap) in this codebase.

## 12.1 The thread pool, and why blocking it is expensive

ASP.NET Core (via Kestrel) handles concurrent requests using a shared **thread pool** — a fixed-ish set of worker threads reused across many in-flight requests, not one thread per request. `async`/`await` (chapter 1 §1.5) exists specifically so that while a request is waiting on I/O (a database round-trip, an HTTP call to another service), its thread is released back to the pool to serve *other* requests, instead of sitting idle. This is the same performance argument behind Node's single-threaded, non-blocking event loop — different mechanism (a real thread pool vs. one event loop thread), same underlying principle: **don't block a worker on I/O you could be waiting on asynchronously.**

Two ways this gets violated, both real, both already seen in earlier chapters:

- **Sync-over-async**: calling `.Result` or `.Wait()` on a `Task` instead of `await`ing it — forces the calling thread to block until the async work finishes, defeating the entire point and risking deadlocks in some hosting contexts (chapter 1 §1.5 already flagged this).
- **A genuinely synchronous blocking call inside async code**: `Thread.Sleep(5000)` in the login flow's brute-force throttle (chapter 8 §8.5) and in `ArchiveJob`'s poll loop's `catch` path — wait, that one's actually `Task.Delay` in the `finally`, correctly async; but `Encryptor.VerifyPasswordHash`'s `Thread.Sleep(1000)` (chapter 8 §8.5/§8.6) and `LoginUserQuery`'s several `Thread.Sleep(5000)` calls genuinely tie up a request-handling thread for the full duration, on every login attempt (successful or not — `VerifyPasswordHash` sleeps unconditionally at the top, before even checking the password). Under real concurrent login traffic, that's real thread-pool pressure that `await Task.Delay(...)` wouldn't cause. A good exercise, now that you've read three chapters that each independently surfaced this: that's not three unrelated observations, it's the same category of issue (blocking calls inside an async pipeline) showing up in three different files — recognizing the *pattern*, not just each individual instance, is the actual skill.

## 12.2 The N+1 query problem

The single most common EF Core (and ORM-in-general) performance bug: loading a list of parent entities, then triggering a *separate* database round-trip for each parent's related data, inside a loop — N+1 queries where 1 (or a handful, with proper `Include`) would do.

```csharp
// BAD — N+1: one query for contracts, then one MORE query per contract inside the loop
var contracts = await _dbContext.Set<TicketingContractEntity>().ToListAsync();
foreach (var contract in contracts)
{
    var admins = await _dbContext.Set<TicketingContractAdminEntity>()
        .Where(a => a.ContractId == contract.Id).ToListAsync(); // <-- fires once per contract!
}

// GOOD — one query, admins eager-loaded via a JOIN
var contracts = await _dbContext.Set<TicketingContractEntity>()
    .Include(c => c.Admins) // assuming a navigation property exists
    .ToListAsync();
```

This is exactly why `QueryBaseHandler.StructureItems()` (chapter 4/5) eager-loads `CreatedByUser`/`ModifiedByUser` up front for *every* query, rather than each DTO resolving its creator lazily later — and why `.AsSplitQuery()` exists at all (chapter 4 §4.7): once you've correctly avoided N+1 by using `.Include()`, multiple *collection* includes in one single SQL join can multiply row counts (the "cartesian explosion" problem) — `.AsSplitQuery()` is the fix for the fix, still avoiding N+1 while avoiding the join-multiplication cost too.

## 12.3 EF Core performance checklist, cross-referenced to earlier chapters

| Practice | Why | Where you've seen it |
|---|---|---|
| `.AsNoTracking()` on every read query | skips change-tracking bookkeeping for data you're not going to mutate/save | chapter 4 §4.6 — used consistently in `QueryBase`/`CommandBase` |
| `.AsSplitQuery()` when `Include`-ing more than one collection navigation | avoids cartesian-explosion row multiplication | chapter 4 §4.7 |
| Indices on frequently-filtered/sorted columns | lets Postgres use an index scan instead of a full table scan | `EntityDbConfig<TEntity>`'s `HasIndex(e => e.CreatedAt)`/`HasIndex(e => e.ModifiedAt)` (chapter 4 §4.3) — every entity gets these two for free; a feature-specific `EntityDbConfig` can add more (e.g. an index on a foreign key that's filtered on heavily) |
| Explicit command timeouts | a slow/hung query shouldn't hold a connection (and a thread, transitively) forever | `_dbContext.Database.SetCommandTimeout(...)`, set per-request from `request.Timeout` in every `QueryBaseHandler`/`CommandBaseHandler.DoHandle()` (chapter 5 §5.4) |
| Pagination with a sane default page size | an unbounded `SELECT *` on a large table is a real production incident waiting to happen | `QueryBaseHandler.PageItems()`: `request.Page.PageSize = request.Page.PageSize <= 0 ? 10 : request.Page.PageSize` (chapter 5 §5.4) — every list endpoint in this app is paginated *by construction*, not by each feature remembering to add `.Take(...)` |
| `Select()` projection instead of loading full entities, when you only need a few columns | less data over the wire, less mapping work | **not** how this codebase's generic query pipeline works — `MapEntitiesAsync` maps the *whole* entity via AutoMapper after loading it fully (chapter 5 §5.4's comment even explicitly rejects `ProjectTo` "because of performance issues and circular dependencies") — a deliberate trade-off (simplicity/consistency of the generic pipeline) over maximum per-query efficiency, worth knowing about rather than assuming every query here is maximally optimized |

## 12.4 Connection pooling

Opening a new physical database connection per query is expensive (TCP handshake, auth, session setup). ADO.NET providers (Npgsql here) pool connections transparently underneath `DbContext` — `AddDbContext<T>` registers a `DbContext` as `Scoped` (chapter 3 §3.3), but the underlying physical connections are reused across requests via the pool, not opened/closed from scratch each time. `HRNS.Database/DependencyInjection.cs` builds an `NpgsqlDataSourceBuilder` explicitly (rather than letting `UseNpgsql(connectionString)` build one implicitly) specifically to register the `pgvector` type handler once at the data-source level (chapter 13) — worth noting as an example of a library needing to hook in *below* the per-request `DbContext` layer, at the shared connection-pool layer.

## 12.5 Caching — a genuine gap worth naming honestly

Searching this codebase for `IMemoryCache`/`IDistributedCache` (.NET's built-in in-process and distributed caching abstractions — the .NET equivalent of an in-memory `Map` with TTLs, or reaching for Redis) turns up **nothing** — there's no caching layer in `HRNS.Application` today. That's not necessarily wrong (premature caching is a real anti-pattern — cache invalidation is famously one of the two hard problems in computer science, and a paginated, filtered, per-user-ACL-scoped query result is often a poor caching candidate to begin with, since the cache key would need to encode the filter + page + sort + current user's authorizations to be safe), but it is a real, honest gap to be aware of if you're ever asked to improve read performance on a hot endpoint — `IMemoryCache` (single-instance) or `IDistributedCache` (Redis-backed, safe across multiple app instances) are the standard first tools to reach for, layered in at the `Implementation`/`Interfaces` split point from chapter 6 §6.2 rather than inside the generic `QueryBaseHandler` itself.

## 12.6 `Task.WhenAll` for independent parallel work

Chapter 1 §1.6 introduced this; worth reinforcing with the actual cost model. If a handler needs to await two *independent* pieces of I/O (e.g., fetch a user record and, separately, fetch that provider's settings — nothing about the second depends on the result of the first), sequential awaiting pays the latency of both, back to back:

```csharp
var user = await GetUserAsync(id);           // e.g. 50ms
var settings = await GetProviderSettingsAsync(providerId); // e.g. 40ms
// total: ~90ms, even though nothing here depended on 'user'
```
```csharp
var userTask = GetUserAsync(id);
var settingsTask = GetProviderSettingsAsync(providerId);
await Task.WhenAll(userTask, settingsTask);   // total: ~50ms — bounded by the SLOWER of the two, not the sum
var user = await userTask; var settings = await settingsTask;
```
This codebase's generic CQRS pipeline (chapter 5) is largely sequential-by-construction (one `DbContext`, not safe to run multiple concurrent queries against from multiple threads simultaneously — EF Core's `DbContext` is explicitly **not thread-safe** for concurrent operations), so this specific optimization applies more to independent calls to *external* services (email, AI, file storage) than to database queries within one handler — a distinction worth keeping straight: parallelizing two awaits against the *same* `DbContext` instance concurrently is a bug, not an optimization, and will throw at runtime.

## Checkpoint

1. Find one more `.Include(...)` chain anywhere in `HRNS.Application/CQRS/` with two or more collection-valued includes, and confirm it uses `.AsSplitQuery()`. If you find one that doesn't, is that a real N+1/cartesian-explosion risk, or a false alarm (e.g. both includes are single-valued references, not collections)?
2. If you were asked to add caching to one read-heavy, rarely-changing endpoint in this app (a good candidate: something like the list of `AvailableApplications` or `CompanyAccountTypes` seeded once and rarely modified — chapter 4 §4.2's `HRNSDbContext.DbSet` list), sketch (in words, not code) what the cache key would need to include to stay correct.
3. Explain in your own words why running two `await someDbContext.Set<T>().ToListAsync()` calls concurrently via `Task.WhenAll` against the *same* injected `DbContext` would be a bug, even though the same pattern against two different `HttpClient` calls is a legitimate optimization.
