# 5. CQRS & MediatR — the core of this codebase

This is the chapter everything else was building toward. `HRNS.Platform.Server` isn't "an ASP.NET Core app with some CQRS sprinkled in" — nearly every feature (every controller action, every business rule) is expressed as one Query or one Command flowing through one shared pipeline. Once this chapter clicks, the other ~95% of `HRNS.Application/CQRS/` — hundreds of files — stops looking like a wall of unfamiliar code and starts looking like the same six-piece pattern repeated with different types.

## 5.1 CQRS in theory

CQRS = **Command Query Responsibility Segregation**: reads (*Queries* — "give me data, change nothing") and writes (*Commands* — "change something, tell me if it worked") are modeled as distinct, separately-handled operations instead of one generic "service" class with a grab-bag of methods. Compare it to how you might already separate reads/writes conceptually — a GraphQL API's `Query` vs `Mutation` types are the same idea, or splitting a REST controller's `GET` handlers from its `POST`/`PUT`/`DELETE` handlers.

Why bother? Two real payoffs, both visible in this codebase:

1. **A query and a command genuinely have different shapes.** A query needs filtering, sorting, paging, and returns *many* items. A command needs items-to-save and returns success/failure. Trying to force both into one generic "repository service" method signature is where a lot of over-abstracted service layers go wrong. CQRS just admits up front that they're different and gives each its own base class (§5.3).
2. **Every query and every command can be pushed through the exact same pipeline** (validation → authorization → the actual work → audit logging) without that pipeline caring what the specific feature does. That's CQRS's real payoff *combined with* the mediator pattern, next.

## 5.2 MediatR — the mediator pattern

Without a mediator, a controller directly calls a service, which calls another service, and so on — same as calling into Nest providers directly. The **mediator pattern** inserts one indirection: the controller doesn't call a handler directly, it sends a *message* (a plain object describing "what I want") to a mediator, which looks up the one handler registered for that message's type and invokes it.

```csharp
public class GetUserQuery : IRequest<UserDto>          // 1. the message — just data
{
    public Guid UserId { get; set; }
}

public class GetUserQueryHandler : IRequestHandler<GetUserQuery, UserDto> // 2. the handler — the logic
{
    public async Task<UserDto> Handle(GetUserQuery request, CancellationToken cancellationToken)
    {
        // ... look up the user, map to DTO, return it
    }
}

// 3. usage — controller never references GetUserQueryHandler directly
var user = await _mediator.Send(new GetUserQuery { UserId = id }, cancellationToken);
```

MediatR (the library) does the wiring: at startup it scans your assemblies for every `IRequestHandler<,>` implementation and registers it in the DI container (`services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(...))`, see chapter 3 §3.3); at runtime, `_mediator.Send(message)` resolves the exactly-one handler whose generic type matches the message and calls it. This is the same one-message-one-handler idea as an event bus, minus the "many subscribers" part — MediatR's `Send` is strictly one-to-one (it also supports `Publish` for one-to-many notifications, which this codebase doesn't lean on for its core CRUD flows).

**Why go through a mediator instead of calling a service directly?** The controller becomes dumb (it only knows "send a message, get a response") and *all* cross-cutting behavior — validation, authorization, logging — can live in one shared place the mediator always passes through, instead of being duplicated at the top of every service method. That shared place, in this codebase, is `BaseHandler<TRequest, TResponse>` — the actual subject of this chapter.

## 5.3 The generic envelope — five layers of base types

Before looking at the pipeline, you need the vocabulary. Every query/command in this codebase is built from the same stack of generic base types, from the ground up:

```
HRNS.Models/Model/DTOModel.cs   — "Data Transfer Object": read-side shape, POCO, not persisted directly
HRNS.Models/Model/STOModel.cs   — "Storage/State Transfer Object": write-side shape, carries a `Recover` flag for undelete

HRNS.Common/CQRS/InternalRequest/InternalRequest.cs
    — base for every message: carries `User` (who's asking) + `UserId`, used for RBAC/ACL

HRNS.Application/CQRS/Generic/ServiceLoadRequest.cs   — adds PropsFilter, Sorts, Page (query-only concerns)
HRNS.Application/CQRS/Generic/ServiceSaveRequest.cs   — adds Items[] to save (command-only concern)

HRNS.Application/CQRS/Base/QueryBase.cs    — QueryBase<TDTOModel, TPropsFilter> : ServiceLoadRequest<...>
HRNS.Application/CQRS/Base/CommandBase.cs  — CommandBase<TSTOModel> : ServiceSaveRequest<...>
```

```csharp
// DTOModel — what a query returns: one item, shaped for reading
public class DTOModel : RefModel
{
    public DateTime CreatedAt { get; set; }
    public UserRef CreatedByUser { get; set; }
    public DateTime? ModifiedAt { get; set; }
    public UserRef ModifiedByUser { get; set; }
}

// STOModel — what a command receives: one item, shaped for writing (note: NOT the same shape as DTOModel)
public class STOModel : BaseModel
{
    public bool? Recover { get; set; } // set true to "undelete" a soft-deleted row
}
```

Every feature-specific DTO/STO (`TicketingTicketTypeDTO`, `TicketingTicketTypeSTO`, ...) inherits from these. This DTO/STO split is deliberate and worth sitting with: **the shape you read is not the shape you write**. A `DTO` carries resolved references (`UserRef CreatedByUser`, a full object) for display; an `STO` carries just enough to persist a change. Mixing them (one shape for both reading and writing) is a common shortcut in smaller apps — HRNS explicitly doesn't take it, because the two responsibilities genuinely diverge once ACLs, audit fields and partial updates enter the picture.

`InternalRequest<TResponse>` is the thing that makes authorization possible generically — because *every* message, query or command, carries `User`/`UserId`, `BaseHandler` can check authorization once, centrally, without knowing anything feature-specific about the message.

## 5.4 The two generic handlers: `QueryBaseHandler` and `CommandBaseHandler`

This is where the framework earns its keep — the entire read pipeline and the entire write pipeline are implemented **once**, generically, and every concrete feature only overrides the small pieces that are actually feature-specific.

**Reads** — `QueryBaseHandler<TQuery, TEntity>.DoHandle()` (`CQRS/Base/QueryBase.cs:48-79`) always does the same five steps, each overridable:

```
StructureItems()          →  base IQueryable<TEntity> (AsNoTracking + AsSplitQuery + Include CreatedByUser/ModifiedByUser)
FilterItems()              →  FilterItemsByPermissionsAndACL() + FilterItemsByFilter() (Ids, NotIds, date ranges, IsDeleted...)
CountTotalItems()          →  COUNT(*) for pagination metadata
OrderItems()                →  caller-supplied Sorts, or default ModifiedAt/CreatedAt descending
PageItems()                  →  Skip/Take from the request's Page
MapEntitiesAsync()            →  AutoMapper: TEntity -> TDTOModel, per row
```

**Writes** — `CommandBaseHandler<TCommand, TEntity>.DoHandle()` (`CQRS/Base/CommandBase.cs:45-123`) loops the incoming `Items[]` and, per item:

```
look up the existing entity BY ID, through the ACL filter    (GetEntityByIdAndACLFilter)
also look it up WITHOUT the ACL filter                        (GetEntityByIdNoACLFilter)
  if neither found  → this is a genuinely new row → map STO -> new entity, stamp CreatedAt/CreatedByUserId, Add()
  if only the no-ACL lookup found it → the row exists but the caller can't see/touch it → throw UnauthorizedAccessException
  if found (with ACL) → map STO onto the existing entity, stamp ModifiedAt/ModifiedByUserId, Update()
     (IsConstant rows can never be soft-deleted — the flag is force-reset to false even if the incoming STO asked for it)
SaveChangesAsync()
```

Notice there's no explicit "delete" branch — because there's no hard delete. Setting `sto.IsDeleted = true` on an existing item and sending it through the normal update path *is* the delete operation (`Entity.IsDeleted` gets mapped like any other field, and the global query filter from chapter 4 §4.5 makes it invisible afterward). "Recover" is the same trick in reverse — a soft-deleted row looked up with `Recover: true` bypasses the query filter, gets its `IsDeleted` flag reset to `false`, and is saved as an update.

Both generic handlers sit on top of one more shared base, `BaseHandler<TRequest, TResponse>` (`CQRS/Base/BaseHandler.cs`), which wraps `DoHandle()` with the parts that are common to *every* request regardless of read/write:

```csharp
public async Task<TResponse> Handle(TRequest request, CancellationToken cancellationToken)
{
    if (LocationInfoRequired)
        _userLocationInfo = await _userLocationInfoFactory.FillUserLocationInfo(request.ClientIpAddress);

    await LoadUserAuthorizations(request, cancellationToken);      // who is this user, what can they do

    if (!CheckUserAuthorization(request))                          // does AccessDomainIds allow this action
        throw new UnauthorizedAccessException(...);

    // (not shown: PreHandle() runs here — the extension point commands override for extra, per-feature authz)
    var response = await DoHandle(request, cancellationToken);      // <-- the actual read or write logic
    await FillAboutItems(request, response, cancellationToken);
    await LogUserAction(request, response, cancellationToken);       // writes a UserActivityHistoryEntity row
    return response;
}
```
(condensed/annotated from `CQRS/Base/BaseHandler.cs:130+`.) This is the concrete mechanism behind the claim in §5.2: validation, authorization, and audit logging are written **once**, here, and every one of the hundreds of Query/Command classes in this repo gets all three for free just by inheriting from `QueryBaseHandler`/`CommandBaseHandler`.

## 5.5 End-to-end trace: one HTTP request, start to finish

Follow a real save request (`SaveTicketingTicketTypeAsync`) all the way through:

1. **HTTP POST** `api/TicketingTicketType/SaveTicketingTicketTypeAsync` hits `TicketingTicketTypeController` (chapter 3 §3.2).
2. Controller action calls `SaveSTOsAsync<UpsertTicketingTicketTypeCommand, TicketingTicketTypeSTO>(request)` — a generic helper on `BaseController` (`Areas/BaseController.cs:211-305`) that builds a `new UpsertTicketingTicketTypeCommand()`, fills in `User`, `UserId`, `Items`, origin/host info, and calls `_mediator.Send(command, cancellationToken)`.
3. MediatR resolves `UpsertTicketingTicketTypeCommandHandler` (the *only* class implementing `IRequestHandler<UpsertTicketingTicketTypeCommand, ServiceSaveResponse>`) and invokes `Handle()`.
4. `Handle()` (inherited from `BaseHandler`) loads the caller's authorizations, checks `AccessDomainIds`, then calls the overridable `PreHandle()` hook **before** `DoHandle()`.
5. `UpsertTicketingTicketTypeCommandHandler` overrides `PreHandle()` with feature-specific authorization the generic framework can't know about — "is this user an admin of the specific ticketing contract these items belong to":

```csharp
protected override async Task PreHandle(UpsertTicketingTicketTypeCommand request, CancellationToken cancellationToken)
{
    var isSystemAdmin = CurrentUserAuthorizations?.IsAdmin ?? false;
    if (!isSystemAdmin)
    {
        var contractIds = request.Items.Select(sto => sto.Contract?.Id ?? Guid.Empty).Distinct().ToArray();
        var administeredContractIds = await _dbContext.Set<TicketingContractAdminEntity>()
            .AsNoTracking()
            .Where(admin => admin.UserId == userId && contractIds.Contains(admin.ContractId))
            .Select(admin => admin.ContractId)
            .ToListAsync(cancellationToken);

        if (contractIds.Any(id => !administeredContractIds.Contains(id)))
            throw new UnauthorizedAccessException("User is not authorized to administer ticket types on this ticketing contract.");
    }
    await base.PreHandle(request, cancellationToken);
}
```
(`HRNS.Application/CQRS/Ticketing/TicketingTicketType/UpsertTicketingTicketTypeCommand.cs:42-72`, in full.)

6. `DoHandle()` runs — inherited, unmodified, from `CommandBaseHandler<TCommand, TEntity>` — the upsert-or-create-or-soft-delete loop from §5.4 runs against `TicketingTicketTypeEntity`.
7. `SaveChangesAsync()` — EF Core generates the SQL (chapter 4).
8. `LogUserAction()` writes a `UserActivityHistoryEntity` audit row (inherited, unmodified).
9. `ServiceSaveResponse` flows back up through the mediator, into `SaveSTOsAsync`, gets wrapped in a `SaveSTOWebResponse`, and is serialized as the HTTP response.

**The entire feature-specific code for this endpoint is the 74-line file quoted above** (plus the equally-thin controller action from chapter 3, plus the entity/DbConfig/MappingProfile trio from chapter 4). Everything else — filtering, paging, sorting, soft delete, ACL enforcement, audit logging — is inherited. That's the whole point of the exercise.

## 5.6 What you get "for free" by following the pattern

When you add a brand-new feature that fits the standard CRUD shape, you genuinely only need to write:
1. An `Entity` + `EntityDbConfig<TEntity>` + `EntityMappingProfile` (chapter 4 + chapter 7).
2. A DTO and an STO (plain classes, chapter 7).
3. A `Query...` class (usually **zero lines of override code** — just `public class GetFooQuery : QueryBase<FooDTO, FooFilter> { public class GetFooQueryHandler : QueryBaseHandler<GetFooQuery, FooEntity> { public GetFooQueryHandler(...) : base(...) {} } }`).
4. A `Command...` class, same shape, overriding `PreHandle`/`SetAdditionalFieldsOnAdd`/etc. only if this feature needs authorization or default-value logic the base doesn't already provide.
5. A controller action that's two lines: call `LoadDTOsAsync<...>` or `SaveSTOsAsync<...>`.

## 5.7 Honest trade-offs — an architecture opinion worth having

This is a genuinely powerful pattern for a CRUD-heavy enterprise app like this one (dozens of near-identical "list/filter/page these records" and "upsert these records" features) — it eliminates enormous amounts of repetition and guarantees every feature gets ACL + audit logging + soft delete consistently, which is exactly the kind of thing that's easy to forget when hand-writing each endpoint.

The cost is real too, and worth naming rather than glossing over: **three levels of generics stacked** (`QueryBaseHandler<TQuery, TEntity>` where `TQuery : QueryBase<TDTOModel, TPropsFilter>`) is genuinely harder to hold in your head than a plain, one-off service method — that's exactly why this chapter exists. It also means the "interesting" logic for any given feature is often a *small diff* against a much larger inherited base, so reading a single Command file in isolation (without knowing `CommandBaseHandler`'s `DoHandle()` by heart) can be misleading — you have to know what's inherited to know what a given override is actually changing. Once you've internalized §5.3–§5.4, though, that cost mostly disappears — you're pattern-matching against a shape you already know instead of reading each handler from scratch.

## Checkpoint

1. Pick any `Upsert*Command.cs` file you haven't seen, under `HRNS.Application/CQRS/`. Does it override `PreHandle`? `SetAdditionalFieldsOnAdd`/`SetAdditionalFieldsOnUpdate`? Anything else? What's the *smallest possible* diff against the base class you can find in the whole `CQRS/` folder — and what's one of the largest (a handler that overrides most of `DoHandle` itself, like `LoginUserQuery`, chapter 8)?
2. Trace one `Get*Query.cs` file's `FilterItemsByPermissionsAndACL` override (if it has one) — what does it restrict, and why would the base `QueryBase` implementation (which returns the query untouched) be wrong for that specific feature?
3. Using `gitnexus_query({query: "..."})` or `graphify query "..."`, search for a feature name you're curious about and see whether the tool surfaces the Query/Command/Entity/Controller quartet as one connected execution flow — that's the graph doing exactly the pattern-recognition this chapter just taught you by hand.
