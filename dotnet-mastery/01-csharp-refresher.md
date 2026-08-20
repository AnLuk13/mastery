# 1. C# Refresher

Goal: get OOP fundamentals and modern C# syntax back to fluency, with the specific vocabulary (`generics`, `LINQ`, `async/await`, `nullable reference types`, `records`, `pattern matching`) that shows up on nearly every page of `HRNS.Platform.Server`.

## 1.1 Classes, objects, encapsulation

Same concept as TS classes, different defaults. The big difference: in C#, **members are `private` by default** (TS class fields are `public` by default unless you write `private`/`#field`).

```csharp
public class BankAccount
{
    // fields: private by convention, prefixed with _
    private decimal _balance;

    // auto-property: compiler generates a hidden backing field for you
    public string Owner { get; private set; }

    // constructor
    public BankAccount(string owner, decimal openingBalance)
    {
        Owner = owner;
        _balance = openingBalance;
    }

    // method
    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentOutOfRangeException(nameof(amount));
        _balance += amount;
    }

    public decimal Balance => _balance; // expression-bodied read-only property
}
```

`{ get; private set; }` is the C# equivalent of exposing a getter but keeping the setter private to the class — same intent as a TS class with a private `#balance` and a public getter.

## 1.2 Interfaces, abstract classes, inheritance, polymorphism

- `interface`: pure contract, no implementation (like TS `interface`, but C# interfaces can also carry default method bodies since C# 8 — rarely used, don't reach for it).
- `abstract class`: a base class that can have real implementation *and* members that MUST be overridden (`abstract` members) — TS has no direct equivalent; closest is an abstract class simulated via `protected` constructor + `throw new Error("not implemented")`.
- `virtual` / `override`: opt-in polymorphism. Unlike TS/JS where every method is overridable by default, C# methods are **sealed by default** — you must mark a method `virtual` in the base class before a subclass can `override` it.

```csharp
public interface INotifier
{
    Task SendAsync(string to, string message);
}

public abstract class NotifierBase : INotifier
{
    public abstract Task SendAsync(string to, string message); // must be implemented

    protected virtual string FormatMessage(string message) => message.Trim(); // can be overridden
}

public class EmailNotifier : NotifierBase
{
    public override Task SendAsync(string to, string message)
    {
        var formatted = FormatMessage(message);
        // ... send email
        return Task.CompletedTask;
    }
}
```

This exact shape — `interface` for the contract, `abstract class` for shared base behavior, concrete class for the specific implementation — is how almost every service in HRNS is structured (see §1.6 below).

## 1.3 Generics

Generics in C# are much closer to TypeScript generics than to Java's type-erased generics — C# generics are **reified**: the runtime knows the concrete type at execution time (`List<Guid>` and `List<string>` are genuinely different types at runtime, unlike TS where generics vanish at compile time).

```csharp
public class Repository<TEntity> where TEntity : class
{
    private readonly List<TEntity> _items = new();
    public void Add(TEntity item) => _items.Add(item);
    public IEnumerable<TEntity> All() => _items;
}

// constraints: "where TEntity : class" restricts T to reference types
// other common constraints: "where T : new()" (must have parameterless ctor),
// "where T : SomeBaseClass", "where T : ISomeInterface"
```

You'll see multi-parameter generics with several constraints stacked, e.g. `class Foo<TA, TB> where TA : Entity where TB : DTOModel, new()` — read each `where` clause independently; they don't interact with each other.

## 1.4 LINQ

LINQ (Language Integrated Query) is C#'s built-in, statically-typed equivalent of chaining `.filter()/.map()/.reduce()` in JS — but it works uniformly over in-memory collections (`IEnumerable<T>`, "LINQ to Objects") **and** database queries (`IQueryable<T>`, EF Core translates it to SQL — see chapter 4).

```csharp
var activeAdults = people
    .Where(p => p.IsActive && p.Age >= 18)   // .filter()
    .OrderBy(p => p.LastName)                // .sort()
    .Select(p => new { p.FirstName, p.LastName }) // .map()
    .ToList();                               // materialize (like calling the iterator to completion)

var total = orders.Sum(o => o.Amount);       // .reduce() shortcut
var any = orders.Any(o => o.Amount > 1000);  // .some()
var first = orders.FirstOrDefault(o => o.Id == id); // .find(), returns null/default if not found
```

**Deferred execution** is the one truly non-obvious part: a LINQ chain (`Where`, `Select`, `OrderBy`, ...) builds up a lazy pipeline and does **not** run until you enumerate it (`.ToList()`, `foreach`, `.First()`, ...). This matters enormously with EF Core: everything before `.ToListAsync()` is still "a query being built," not "data."

```csharp
var query = dbContext.Users.Where(u => u.IsActive); // no SQL executed yet
query = query.Where(u => u.Email.Contains("@acme.com")); // still building
var users = await query.ToListAsync(); // NOW it runs, as one SQL statement
```

## 1.5 async/await

Conceptually identical to JS/TS `async/await` — `Task` is C#'s `Promise`, `Task<T>` is `Promise<T>`, `await` unwraps it the same way. The differences that bite people coming from Node:

- **Never block on async code** with `.Result` or `.Wait()` — that's the C# equivalent of a synchronous `await` that can deadlock under ASP.NET Core's synchronization context in some hosting scenarios. Always `await` all the way up.
- `Task` (non-generic) is like `Promise<void>`.
- `CancellationToken` is threaded explicitly through async method signatures — C#'s idiomatic way of doing what `AbortController`/`AbortSignal` does in JS. You'll see it as the last parameter almost everywhere: `Task<T> DoWorkAsync(Request r, CancellationToken cancellationToken)`.
- Naming convention: async methods are suffixed `Async` (`GetUserAsync`, `SaveChangesAsync`) — a convention, not a compiler rule, but followed almost universally, including throughout HRNS.

```csharp
public async Task<UserDto> GetUserAsync(Guid id, CancellationToken cancellationToken)
{
    var user = await _dbContext.Users
        .FirstOrDefaultAsync(u => u.Id == id, cancellationToken);

    if (user is null) throw new NotFoundException($"User {id} not found");

    return _mapper.Map<UserDto>(user);
}
```

`Task.WhenAll` is the equivalent of `Promise.all`:

```csharp
var (a, b) = (await taskA, await taskB); // sequential — bad if independent
await Task.WhenAll(taskA, taskB);        // parallel — good when independent
```

## 1.6 Nullable reference types

A compile-time-only opt-in feature (`<Nullable>enable</Nullable>` in the `.csproj`) that makes the C# type system distinguish `string` (never null, compiler warns if you assign null) from `string?` (nullable, compiler forces you to null-check before use). This is C#'s answer to TypeScript's `strictNullChecks`.

**HRNS specifically only enables it in `HRNS.WebApi`** (`HRNS.WebApi/HRNS.WebApi.csproj:4`), not in `HRNS.Application`, `HRNS.Database`, or `HRNS.Models`. Practically: when you're editing files in the WebApi project, the compiler will nag you about nullability; everywhere else in the codebase it won't, and older, more "classic C#" nullable handling (checking `!= null` defensively) is the norm — that's why you'll see `IsNull()` / `IsNullSafeAny()` extension-method helpers instead of `?.` chains in a lot of the `Application`/`Database` code (more in chapter 2 on extension methods).

## 1.7 Records & pattern matching (modern C#, used sparingly in HRNS)

`record` is a reference type with built-in value equality and a concise declaration — closest analogue is a TS `interface` plus an auto-generated deep-equals and a `with`-style copy constructor:

```csharp
public record PageRequest(int PageNumber, int PageSize);

var p1 = new PageRequest(0, 10);
var p2 = p1 with { PageSize = 20 }; // immutable "copy with change", like { ...p1, pageSize: 20 }
```

HRNS mostly uses plain mutable classes for DTOs (`DTOModel`, `STOModel` subclasses — see chapter 5) rather than `record`, because those objects get built up field-by-field by the controller/handler pipeline before being sent to AutoMapper. You'll still encounter `record` occasionally in newer, smaller pieces of the codebase (e.g., request/response shapes for AI services).

Pattern matching — `switch` expressions and `is` patterns replace long `if/else if` chains:

```csharp
string Describe(object value) => value switch
{
    null => "nothing",
    int n when n < 0 => "negative number",
    int n => $"number {n}",
    string s when s.Length == 0 => "empty string",
    string s => $"string of length {s.Length}",
    _ => "something else"
};
```

## 1.8 Extension methods

C#'s way of "adding a method" to a type you don't own, without subclassing — the rough equivalent of monkey-patching `Array.prototype` in JS, but statically typed and scoped to files that `using` the namespace it's declared in.

```csharp
public static class StringExtensions
{
    public static bool IsNullOrEmptySafe(this string? value) =>
        string.IsNullOrEmpty(value);
}

// usage — looks like an instance method, but it's just sugar for StringExtensions.IsNullOrEmptySafe(name)
if (name.IsNullOrEmptySafe()) { ... }
```

HRNS leans on this pattern heavily — see `Extensions/System.Extensions`, `Extensions/AutoMapper.Extensions`, and `Extensions/Microsoft.EntityFrameworkCore.Extensions` at the repo root. Those are entire *projects* dedicated to extension methods layered onto framework types. You already saw two of them in chapter-5 territory: `request.PropsFilter.Ids.IsNullSafeAny()` and `entity.IsNull()` used throughout `CQRS/Base/QueryBase.cs` are custom extension methods from `HRNS.Common`/`System.Extensions`, not framework built-ins.

## Checkpoint

Before moving to chapter 2, go find and read (don't just skim):

1. `HRNS.Database/Entities/Entity/Entity.cs` — the abstract base class every database entity inherits from. Identify: which members are auto-properties, which are nullable (`Guid?`, `DateTime?`), and why `ModifiedByUserId` is nullable while `CreatedByUserId` isn't.
2. `HRNS.Application/CQRS/Base/QueryBase.cs` — find the generic constraint list on `QueryBaseHandler<TQuery, TEntity>`. Write out in plain English what each `where` clause requires.
3. Open any file under `Extensions/System.Extensions/` and identify one extension method. What framework type does it extend, and what LINQ-like convenience does it add?
