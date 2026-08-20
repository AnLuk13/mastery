# 9. Testing

Goal: xUnit fundamentals, the actual testing stack `HRNS.Tests` uses (xUnit + FluentAssertions + NSubstitute + EF Core InMemory + `WebApplicationFactory`), and when to reach for each style of test.

## 9.1 The stack, and why each piece is there

From `HRNS.Tests/HRNS.Tests.csproj`:

| Package | Role | Node/TS analogue |
|---|---|---|
| `xunit` | test framework — `[Fact]`, `[Theory]`, test discovery/execution | Jest/Vitest/Mocha |
| `FluentAssertions` | readable assertion syntax (`result.Should().Be(x)`) | `expect(result).toBe(x)`, or `chai` |
| `NSubstitute` | mocking/faking interfaces | `jest.fn()` / `sinon` |
| `Microsoft.EntityFrameworkCore.InMemory` | fake, in-process DB provider for fast unit/integration tests | an in-memory SQLite DB, or mocking your ORM entirely |
| `Microsoft.EntityFrameworkCore.Sqlite` | a *real* (if lightweight) relational engine for tests that need actual SQL semantics InMemory can't fake | `better-sqlite3` in tests |
| `Microsoft.AspNetCore.Mvc.Testing` | `WebApplicationFactory<T>` — spins up the whole app in-process for true end-to-end HTTP tests | `supertest` against an Express app instance |

## 9.2 xUnit basics

```csharp
public class CalculatorTests
{
    [Fact] // a single, parameterless test case
    public void Add_TwoPositiveNumbers_ReturnsSum()
    {
        var result = Calculator.Add(2, 3);
        result.Should().Be(5); // FluentAssertions
    }

    [Theory] // a parameterized test — runs once per InlineData row
    [InlineData(2, 3, 5)]
    [InlineData(-1, 1, 0)]
    [InlineData(0, 0, 0)]
    public void Add_VariousInputs_ReturnsExpectedSum(int a, int b, int expected)
    {
        Calculator.Add(a, b).Should().Be(expected);
    }
}
```

`[Fact]`/`[Theory]` are xUnit's `test()`/`test.each()`. Test method naming convention throughout this codebase (and common in .NET generally): `MethodOrScenario_Condition_ExpectedResult` — readable as a sentence, and visible directly in **9.4**'s real example.

FluentAssertions' real value over a plain `Assert.Equal(expected, actual)` is the failure message and the vocabulary — `.Should().ContainSingle("because ...")`, `.Should().BeEquivalentTo(...)`, `.Should().Throw<T>()` all read close to English and produce diagnostic-friendly failure output, which matters a lot once test suites get large.

## 9.3 Unit vs. integration tests — where HRNS draws the line

- **Unit test**: exercises one piece of logic in isolation, with everything around it faked/stubbed. Fast, no I/O.
- **Integration test**: exercises real collaboration between multiple real pieces (the actual DI container, the actual HTTP pipeline, a real-enough database) — slower, but catches wiring bugs a unit test structurally cannot (a service that isn't registered in DI, a route that doesn't match, a migration that doesn't apply).

`HRNS.Tests` has genuine examples of both, and — notably — most of its "unit-ish" tests skip mocking `DbContext` entirely and use the real `HRNSDbContext` backed by EF Core's InMemory provider instead. That's a deliberate, common modern choice: mocking `DbContext`/`DbSet<T>` by hand is painful (LINQ query translation doesn't work against a mock), and EF Core's InMemory provider gives you a real, LINQ-queryable `DbContext` without needing an actual PostgreSQL instance — fast enough to run in CI on every push, "real" enough to test actual query/save logic.

## 9.4 A real "unit-ish" test: EF Core InMemory, no HTTP, no mocking

```csharp
public class TicketingTicketTypeSoftDeleteTests
{
    private static HRNSDbContext CreateContext(string? dbName = null)
    {
        var options = new DbContextOptionsBuilder<HRNSDbContext>()
            .UseInMemoryDatabase(databaseName: dbName ?? Guid.NewGuid().ToString())
            .UseQueryTrackingBehavior(QueryTrackingBehavior.TrackAll)
            .Options;
        return new HRNSDbContext(options);
    }

    [Fact]
    public async Task RemovingTicketTypeFromIncomingList_SoftDeletesIt_AndHidesItFromDefaultQueries()
    {
        var dbName = Guid.NewGuid().ToString();
        using var ctx = CreateContext(dbName);

        // ... Arrange: build & save a contract, a team, and two ticket types

        await RemoveTicketTypesNotIn(ctx, contract.Id, new HashSet<Guid> { keptType.Id });

        using var verifyCtx = CreateContext(dbName); // fresh context, same in-memory "database" (same dbName)

        var visibleTypes = await verifyCtx.Set<TicketingTicketTypeEntity>()
            .Where(t => t.ContractId == contract.Id).ToListAsync();

        visibleTypes.Select(t => t.Id).Should().BeEquivalentTo(
            new[] { keptType.Id },
            "the removed type should no longer show up under the default (!IsDeleted) query filter"
        );

        var removedFromDb = await verifyCtx.Set<TicketingTicketTypeEntity>()
            .IgnoreQueryFilters().FirstAsync(t => t.Id == removedType.Id);
        removedFromDb.IsDeleted.Should().BeTrue("the row itself should be soft-deleted, not hard-deleted");
    }
}
```
(`HRNS.Tests/Ticketing/TicketingTicketTypeSoftDeleteTests.cs`, condensed — full file worth reading top to bottom, it's short.)

Three things worth internalizing from this real test:

1. **A fresh `DbContext` per phase, same `dbName`.** InMemory "databases" are keyed by name and live independently of any one `DbContext` instance — creating a second context with the same name against the same in-memory store, after the first is done and disposed, is exactly how you make sure you're testing what got *persisted*, not what's still sitting in the first context's change-tracker cache (which would pass even if the save silently failed).
2. **The `.Should().BeEquivalentTo(..., "because ...")` string argument** becomes part of the failure message if the assertion fails — cheap, and worth doing on any assertion whose *reason* isn't obvious from the code alone.
3. **The test doesn't call the real `CommandBaseHandler` upsert loop** (chapter 5 §5.4) — it re-implements the specific diff-and-soft-delete shape locally (`RemoveTicketTypesNotIn`), with a comment explaining exactly why: "*Mirrors the diff-and-soft-delete shape that CommandBaseHandler.DoHandle applies per item*." This is a real trade-off worth knowing about, not hiding: testing the *mechanism* (soft delete via a query filter) in isolation is valuable and fast, but doesn't prove the actual production handler behaves identically — that's what an integration test (§9.5) is for.

## 9.5 A real integration test: `WebApplicationFactory` — the whole app, in-process

```csharp
public class AuthIntegrationTests : IClassFixture<WebApplicationFactory<HRNS.WebApi.Program>>
{
    private readonly WebApplicationFactory<HRNS.WebApi.Program> _factory;

    public AuthIntegrationTests(WebApplicationFactory<HRNS.WebApi.Program> factory)
        => _factory = factory.WithWebHostBuilder(webAppBuilder => { /* override services if needed */ });

    [Fact]
    public async Task Login_ReturnsToken_ForValidUser_LocalDevPG()
    {
        var httpClient = _factory.CreateClient(); // a real in-memory HttpClient hitting the real pipeline
        httpClient.DefaultRequestHeaders.Add("Origin", "http://localhost");

        var loginModel = new LoginModel { Email = "...", Password = "..." };
        var response = await httpClient.PostAsJsonAsync("api/Users/LoginAsync", loginModel);
        // Assert: response status, token shape, etc.
    }
}
```
(`HRNS.Tests/AuthIntegrationTests.cs:39-70`, condensed.) `IClassFixture<WebApplicationFactory<Program>>` — note `Program` here is exactly the `Program` class from `HRNS.WebApi/Program.cs` (chapter 2 §2.4); `WebApplicationFactory<T>` boots that *exact* app (same DI registrations, same middleware pipeline) in-process, then hands you an `HttpClient` wired directly to it — no real network socket, no separately-running server, but genuinely the real ASP.NET Core pipeline end to end. `IClassFixture<T>` is xUnit's mechanism for sharing one expensive-to-create object (booting the whole app) across every test in a class instead of rebuilding it per test.

`HRNS.Tests/Infrastructure/TestWebApplicationFactory.cs` is a more heavily customized version of the same idea, purpose-built to make the whole app testable without any real infrastructure:

```csharp
public class TestWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");
        builder.ConfigureAppConfiguration((context, config) =>
        {
            config.AddInMemoryCollection(new Dictionary<string, string?>
            {
                ["ConnectionStrings:PGSQL"] = "InMemory",       // triggers HRNS.Database's InMemory branch — chapter 3 §3.3
                ["Authentification:Jwt:jwtSecretKey"] = TestSigningKey,
                ["Firebase:Enable"] = "false",                    // skip real Firebase wiring entirely
                // ...
            });
        });
        // ConfigureServices then swaps the real Npgsql DbContext registration for InMemory
    }
}
```
(condensed.) This is the concrete mechanism behind the comment you already saw in chapter 3's `AddPersistence` walkthrough — `"connString == 'InMemory' -> skip Npgsql/DataSource setup entirely, TestWebApplicationFactory will replace DbContext with InMemory."` Feature flags like `Firebase:Enable` (chapter 2 §2.5) pull double duty here: the same toggle that lets ops disable a real integration in production also lets tests disable it entirely, for free.

## 9.6 Mocking with NSubstitute

For a genuine unit test that needs to isolate one class from a *specific* collaborator (not a whole `DbContext`), NSubstitute creates fake implementations of interfaces at test time:

```csharp
var emailService = Substitute.For<IEmailService>();
emailService.SendEmailAsync(Arg.Any<string>(), Arg.Any<string>(), Arg.Any<string>())
    .Returns(Task.FromResult((true, string.Empty)));

var handler = new SomeHandler(emailService /* , other deps */);
await handler.Handle(request, CancellationToken.None);

await emailService.Received(1).SendEmailAsync(expectedTo, Arg.Any<string>(), Arg.Any<string>());
```

This only works because handlers depend on **interfaces** (`IEmailService`, not `EmailService` directly) — chapter 6 §6.2's dependency-inversion point paying off directly in testability, the same reason it mattered there.

## 9.7 What to reach for, when

| You're testing... | Reach for |
|---|---|
| A pure function / small piece of business logic | Plain `[Fact]`/`[Theory]`, no DB, no mocks |
| Query/save logic against entities, relationships, soft delete, query filters | EF Core InMemory `DbContext` (§9.4) — real LINQ, no real Postgres needed |
| "Does this whole feature work end-to-end, wired the way production wires it" | `WebApplicationFactory` (§9.5) |
| "Does my handler call this *specific* dependency correctly, without caring what the dependency actually does" | NSubstitute (§9.6) |
| Something that depends on real PostgreSQL-specific SQL semantics (e.g. the `vector`/pgvector extension, chapter 13) that InMemory can't emulate | `Microsoft.EntityFrameworkCore.Sqlite`, or a real Postgres test container — heavier, use sparingly |

## Checkpoint

1. Open `HRNS.Tests/Application/PolicyVersionManagement/PolicyVersionManagementTests.cs` — is it closer to the InMemory-DbContext style (§9.4) or the `WebApplicationFactory` style (§9.5)? What does that choice tell you about what the test is trying to prove?
2. Write (on paper or in a scratch file) a `[Theory]`/`[InlineData]` test skeleton for `CommandBaseValidator`'s "no items to save" rule (chapter 7 §7.1) — what inputs would you feed it, and what would you assert for each?
3. `TestWebApplicationFactory` disables Firebase via `Firebase:Enable = "false"` rather than mocking `IFirebaseNotificationService` directly. Given `Program.cs` already registers a `NullFirebaseNotificationService` when that flag is off (chapter 3 §3.3), explain why that's *more* useful for a test than injecting a hand-rolled mock would be.
