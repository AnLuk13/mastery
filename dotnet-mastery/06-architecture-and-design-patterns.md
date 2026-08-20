# 6. Architecture & Design Patterns

Goal: the theory behind *why* this codebase is shaped the way it is — layered/Clean/Onion architecture, SOLID, and a handful of GoF design patterns you've now already seen in the wild in chapters 3–5, named explicitly.

## 6.1 Layered architecture, in general

The classic idea, decades old and still the right default for most business apps: separate code into layers where **each layer only knows about the layer(s) "below" it**, never the ones "above." A common four-layer split:

```
Presentation   (HTTP controllers, GraphQL resolvers, CLI — the "how does the outside world talk to us" layer)
Application     (use-case orchestration — "what happens when X is requested")
Domain           (business entities and rules — the actual subject matter, independent of any framework)
Infrastructure    (databases, external APIs, file systems — the "how do we actually persist/fetch things" layer)
```

**Clean Architecture** (Robert C. Martin) and **Onion Architecture** (Jeffrey Palermo) are two well-known refinements of the same idea, both centered on one non-negotiable rule: **dependencies point inward, toward the Domain.** Infrastructure depends on Domain; Domain depends on nothing. This is what makes the Domain layer testable and framework-agnostic — you can unit-test business rules with zero database, zero HTTP, zero anything except plain objects.

## 6.2 How HRNS.Platform.Server actually maps onto this

Six main projects (chapter 2 §2.3 had the dependency graph; here's the same graph read as architecture):

| Project | Architectural role |
|---|---|
| `HRNS.Models` | Domain-adjacent: DTOs/STOs (chapter 5 §5.3) — the *shape* of data crossing boundaries, not business rules themselves |
| `HRNS.Database` | Infrastructure + a slice of Domain: EF Core entities (`Entity`, `*Entity` classes) live here, which is a deliberate deviation from textbook Clean Architecture (there, entities would live in a persistence-ignorant Domain project, with EF Core mapping them from the *outside*). Here, entities are EF Core-aware from the start — pragmatic, not "pure," and worth knowing that's the trade-off being made. |
| `HRNS.Application` | Application layer: CQRS handlers (chapter 5), plus most cross-cutting services (`IEmailService`, `IJwtGenerator`, `IAuthService`, ...) via an `Interfaces/` + `Implementation/` folder split *within the same project* |
| `HRNS.Common` | The lowest layer — shared primitives nearly everything depends on (`InternalRequest<T>`, base CQRS message types) |
| `HRNS.Web.CQRS` | A thin presentation-adjacent layer: the generic web request/response envelope types (`LoadDTOWebRequest<T,F>`, `SaveSTOWebResponse`, ...) that controllers speak in |
| `HRNS.WebApi` | Presentation + composition root: controllers, `Program.cs`, background services, middleware |
| `Extensions/*` | Four standalone libraries of extension methods (chapter 1 §1.8) — cross-cutting, framework-adjacent utilities, deliberately kept out of any one layer since several layers use them |

**The honest read**: this is a **pragmatic layered architecture**, not a textbook Clean/Onion Architecture. The give-away is `HRNS.Database` containing both persistence concerns (DbContext, migrations, Npgsql config) *and* the entity classes that in a strict Onion setup would sit in a persistence-ignorant Domain project of their own. That's a completely reasonable trade-off for a large, actively-evolving business app — a stricter separation buys you the ability to swap persistence technology without touching business logic, which is a real but rarely-exercised benefit; what you give up is exactly what you save on: no extra mapping layer between "the concept" and "the row," so entities can use EF Core's Fluent API and navigation properties directly (chapter 4) instead of being mapped from a separate ORM-aware shadow model.

Where **dependency inversion** *is* used deliberately, and matters: `IEmailService`/`IJwtGenerator`/`IAuthService`/... are interfaces (`HRNS.Application/Interfaces/`) consumed by CQRS handlers, with concrete implementations (`HRNS.Application/Implementation/`) registered against those interfaces in DI (chapter 3 §3.3). A handler depends on `IEmailService`, never on `EmailService` directly — which is what makes it possible to substitute a fake/mock in a unit test (chapter 9) without touching SendGrid or SMTP at all.

## 6.3 SOLID, with real examples from this codebase

- **S — Single Responsibility.** `QueryBaseHandler` owns "how to filter/sort/page/map an entity"; `CommandBaseHandler` owns "how to upsert/soft-delete an entity"; `BaseHandler` owns "authorization + audit logging" — three separate concerns, three separate classes, composed through inheritance rather than one class doing everything (chapter 5 §5.4).
- **O — Open/Closed.** `CommandBaseHandler<TCommand, TEntity>` is closed for modification (you never edit it to add a feature) but open for extension — every concrete command customizes behavior by overriding `PreHandle`, `SetAdditionalFieldsOnAdd`, `FilterItemsByPermissionsAndACL`, etc., exactly the virtual/override mechanism from chapter 1 §1.2.
- **L — Liskov Substitution.** Anywhere the framework expects a `QueryBase<TDTOModel, TPropsFilter>`, any concrete query (`GetTicketingTicketTypeQuery`, `LoginUserQuery`, ...) can be substituted without breaking the caller — `BaseController.LoadDTOsAsync<TQuery, ...>` works identically regardless of which concrete query type `TQuery` is.
- **I — Interface Segregation.** `IEmailService`, `IJwtGenerator`, `IEncryptor` are each narrow, single-purpose interfaces rather than one giant `IApplicationServices` grab-bag — a handler that needs to send email doesn't have to know about JWT generation.
- **D — Dependency Inversion.** Covered above: handlers depend on `IEmailService`, not `EmailService`. The interface is owned by the layer that *uses* it (`HRNS.Application`), and the implementation is a detail plugged in via DI — even though, pragmatically, both live in the same project here rather than being split across a project boundary.

## 6.4 Repository / Unit of Work — and why HRNS doesn't add an extra layer for it

A common pattern in older .NET tutorials: wrap `DbContext`/`DbSet<T>` behind your own `IRepository<T>`/`IUnitOfWork` interfaces, so "the database" is an abstraction your business logic depends on instead of EF Core directly.

**HRNS doesn't do this, and it's a defensible choice, not an oversight.** `DbContext` already *is* a Unit of Work (one `SaveChanges()` call commits everything changed since it was created — exactly the Unit-of-Work contract) and `DbSet<TEntity>` already *is* a generic repository (`_dbContext.Set<TEntity>()` gives you exactly the query surface a custom `IRepository<TEntity>` would reinvent). Wrapping EF Core in a hand-rolled repository interface was common advice a decade ago, largely to make code testable against an in-memory fake — but EF Core now ships its own in-memory/SQLite test providers that make the DbContext itself trivially fake-able (chapter 9), which removes most of the original justification. `QueryBaseHandler`/`CommandBaseHandler` (chapter 5) are, in effect, the generic-repository pattern already — just built around EF Core's `DbContext` directly instead of around an extra hand-written abstraction on top of it.

## 6.5 Named GoF patterns you've already read in this codebase

You've seen every one of these in chapters 3–5 without the name attached — naming them now so the vocabulary sticks:

| Pattern | Where |
|---|---|
| **Template Method** | `QueryBaseHandler.DoHandle()`/`CommandBaseHandler.DoHandle()` define a fixed algorithm skeleton (filter → order → page → map; or lookup → create-or-update → save) with individual steps left as `virtual` methods for subclasses to override. This is *the* pattern behind chapter 5 — worth re-reading §5.4 now with the name in hand. |
| **Mediator** | MediatR itself (chapter 5 §5.2) — decouples the sender (controller) from the receiver (handler). |
| **Builder** | `WebApplicationBuilder` (`WebApplication.CreateBuilder(args)` → configure → `.Build()`, chapter 2 §2.4) and EF Core's `DbContextOptionsBuilder`/`ModelBuilder`/`EntityTypeBuilder<T>` (chapter 4) — all fluent, step-by-step object construction. |
| **Factory** | `IGTNParserFactory`/`IGTNGeneratorFactory` (registered `AddTransient` in `Program.cs:188-189`) — produce a configured object without the caller knowing the concrete construction details. |
| **Decorator** (structural cousin) | Extension methods (chapter 1 §1.8) aren't the GoF Decorator pattern precisely (no wrapping object, no shared interface), but serve the same intent — adding behavior to a type without modifying or subclassing it. |
| **Strategy** (implicit) | Every `AccessDomainIds` override plus every `FilterItemsByPermissionsAndACL` override is effectively a pluggable authorization/filtering strategy selected at compile time by which concrete handler class you're in. |
| **Options pattern** (.NET-specific, not GoF) | `services.ConfigureJwtToken(configuration.GetSection(...))` (chapter 2 §2.5) — strongly-typed configuration objects bound from JSON and injected like any other service. |

## 6.6 A rule of thumb for reading (and writing) code in this repo

When you open a new feature folder under `HRNS.Application/CQRS/`, ask, in order:
1. What entity/DTO/STO is this about? (chapters 4 & 7)
2. Is this a `Query` or a `Command`? (§5.3 — what shape of work is it?)
3. What does it override on top of the base handler, and *why* — what does the base implementation get wrong for this specific feature? (§5.4–§5.5)

If you can answer all three in under a minute for an unfamiliar feature, the architecture is doing its job — that's the entire point of pushing 90% of the logic into two generic base classes.

## Checkpoint

1. Find one interface in `HRNS.Application/Interfaces/` with more than one concrete implementation registered conditionally in `Program.cs` (hint: `IFirebaseNotificationService` — look at the `firebaseEnabled` branch around `Program.cs:228-237`). What GoF-adjacent pattern is that conditional registration an example of, and why does it matter that handlers depend on the interface rather than either concrete class?
2. Pick one entity and trace it through all three "layer" concerns it touches: its `*Entity.cs` (Database/Infrastructure), its `*DTO`/`*STO` (Models, crossing into Application), and its Query/Command handler (Application). Which project does each live in, and does that match the table in §6.2?
3. In one paragraph, argue *for* adding a formal repository/unit-of-work abstraction on top of `DbContext` in this codebase, then argue *against* it (§6.4). Which side do you find more convincing for a codebase this size, and why?
