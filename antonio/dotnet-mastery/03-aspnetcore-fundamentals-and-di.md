# 3. ASP.NET Core Fundamentals & Dependency Injection

Goal: understand the request pipeline (middleware), routing, controllers, and — critically — the built-in DI container, since almost every class in this codebase receives its dependencies through it rather than constructing them itself.

## 3.1 The middleware pipeline

ASP.NET Core handles every HTTP request by pushing it through an ordered chain of **middleware** — small pieces of code that each get a chance to inspect/modify the request, call the next one, then inspect/modify the response on the way back out. This is structurally identical to Express middleware (`app.use(...)`) — same "onion" model, same "call `next()` or short-circuit" mental model. **Order matters** exactly like it does in Express — a middleware can only affect requests/responses that pass through it, so authentication must run before authorization, CORS before routing decisions that need it, etc.

Real pipeline from `HRNS.WebApi/Program.cs:108-147` (condensed, comments added):

```csharp
var app = webAppBuilder.Build();

app.UseHealthChecks("/api/health", healthCheckOptions);   // 1. health probe short-circuits before anything else

if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();                      // 2. dev-only: full stack traces in responses
    app.UseHsts();
}

app.UseExceptionMiddleware();                             // 3. custom: catches unhandled exceptions -> JSON error response
app.UseApiSwaggerUi();                                     // 4. serves /api/docs (Swagger)
app.UseWebSockets();

app.UseRouting();                                          // 5. figures out WHICH endpoint matches this URL
app.MapHub<QrCodeHub>("/hub/qrcode");                       //    (SignalR hubs registered here)
app.MapHub<AIAssistantHub>("/hub/ai-assistant");

app.UseCors(_corsConfigurationName);                       // 6. must be after UseRouting, before auth
app.UseForwardedHeaders(forwardedHeadersOptions);           // 7. trust X-Forwarded-* from a reverse proxy

app.UseAuthentication();                                   // 8. WHO is this request from? (reads the JWT)
app.UseAuthorization();                                     // 9. is this WHO allowed to hit this endpoint?

app.MapControllers();                                       // 10. dispatch to the matched controller action
app.ApplyDBContextMigrations();                              // 11. custom: run pending EF Core migrations on boot

app.Run();                                                   // start listening (blocks until shutdown)
```

The comment already baked into the code at `Program.cs:132` — `"CORS middleware must be after UseRouting() but before UseAuthentication()"` — is the pipeline's own documentation of why order matters here; that's not incidental, it's load-bearing.

`app.Use...()` methods that don't map a URL (auth, CORS, exception handling) are true middleware. `app.Map...()` methods (`MapControllers`, `MapHub`) are **endpoint routing** — they register handlers reachable at specific URL patterns.

## 3.2 Routing & controllers

An ASP.NET Core MVC controller is conceptually an Express router file, but declared with a class and attributes instead of `router.get('/path', handler)` calls.

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet("{id:guid}")]
    public async Task<ActionResult<UserDto>> GetById(Guid id, CancellationToken ct)
    {
        var user = await _service.GetUserAsync(id, ct);
        return user is null ? NotFound() : Ok(user);
    }

    [HttpPost]
    public async Task<ActionResult<UserDto>> Create([FromBody] CreateUserRequest request)
    {
        var created = await _service.CreateUserAsync(request);
        return CreatedAtAction(nameof(GetById), new { id = created.Id }, created);
    }
}
```

- `[Route("api/[controller]")]` — `[controller]` is a token replaced with the class name minus "Controller" (`UsersController` → `api/Users`). Attribute routing = the route template lives directly on the class/method, unlike Express's separate `router.get(path, ...)` call.
- `[HttpGet]`/`[HttpPost]`/... pick the HTTP verb. `[ApiController]` turns on a bundle of API-friendly conventions (automatic 400 on invalid model state, automatic body-binding inference, etc — the .NET version of what a framework like NestJS's `@Controller()` gives you for free).
- **Model binding**: `[FromBody]`, `[FromRoute]`, `[FromQuery]` tell the framework where to pull a parameter's value from and deserialize it into — same job as manually reading `req.body`/`req.params`/`req.query` in Express, but declarative and type-checked.

**HRNS does something more specific than the generic example above.** Every controller inherits from `HRNS.WebApi/Areas/BaseController.cs`, and the class-level route is fixed for the whole app:

```csharp
[ApiController]
[Route("api/[controller]/[action]")]   // note the extra [action] token
[Authorize]                            // every endpoint requires a valid JWT unless overridden
public abstract class BaseController : ControllerBase
```

`[action]` means the **method name becomes part of the URL too** (`api/TicketingTicketType/GetTicketingTicketTypeAsync`), and every route requires the method to be named explicitly per action rather than relying on the HTTP verb alone to disambiguate — a deliberate, RPC-flavored convention (closer to how the UI's generated API client calls named endpoints than to strict REST-resource routing). A real controller built on top of it (`HRNS.WebApi/Areas/Ticketing/TicketingTicketType/TicketingTicketTypeController.cs`):

```csharp
public class TicketingTicketTypeController : BaseController
{
    public TicketingTicketTypeController(IHttpContextAccessor httpContextAccessor, IMediator mediator, IMapper mapper)
        : base(httpContextAccessor, mediator, mapper) { }

    [HttpPost]
    public async Task<LoadDTOWebResponse<TicketingTicketTypeDTO>> GetTicketingTicketTypeAsync(
        [FromBody] LoadDTOWebRequest<TicketingTicketTypeDTO, TicketingTicketTypeFilter> request)
        => await LoadDTOsAsync<GetTicketingTicketTypeQuery, TicketingTicketTypeDTO, TicketingTicketTypeFilter>(request);

    [HttpPost]
    public async Task<SaveSTOWebResponse> SaveTicketingTicketTypeAsync([FromBody] SaveSTOWebRequest<TicketingTicketTypeSTO> request)
        => await SaveSTOsAsync<UpsertTicketingTicketTypeCommand, TicketingTicketTypeSTO>(request);
}
```

Notice: **every controller action in this codebase is `[HttpPost]`, even the "get" ones.** That's a deliberate choice (queries here carry a rich filter/paging/sort body too large and structured for query-string encoding), not an oversight — filter/sort/paging objects go in the POST body via `LoadDTOWebRequest<TDto, TFilter>`. This is *not* how a typical REST tutorial does it, and it's the single most surprising routing convention you'll hit reading this codebase for the first time. `LoadDTOsAsync<TQuery, TDTOModel, TPropsFilter>` and `SaveSTOsAsync<TCommand, TSTOModel>` are generic helper methods on `BaseController` that translate the HTTP request into a MediatR message and send it — full breakdown in chapter 5, this chapter is just routing/DI.

## 3.3 Dependency Injection — the built-in container

.NET ships a DI container out of the box (`Microsoft.Extensions.DependencyInjection`) — no need for a third-party tool the way older .NET Framework or many Node setups (InversifyJS, tsyringe) require one. Every service the app needs gets **registered** once (in `Program.cs` / extension methods called from it) against an interface, and the container hands out instances wherever that interface appears in a constructor.

```csharp
// registration (composition root)
services.AddScoped<IEmailService, EmailService>();

// consumption — no "new EmailService()" anywhere in application code
public class SomeHandler
{
    private readonly IEmailService _emailService;
    public SomeHandler(IEmailService emailService) => _emailService = emailService; // constructor injection
}
```

This is the same idea as Nest's `@Injectable()` + constructor injection, minus the decorators — .NET's container works off explicit registration calls instead of reflection-scanned annotations (Nest does support a manual-registration style too, via modules — that's the closer analogy).

### Lifetimes — the part that actually causes bugs if you get it wrong

| Lifetime | New instance... | Typical use | Node/Nest analogue |
|---|---|---|---|
| `AddTransient<I, T>()` | every time it's requested | stateless, cheap services | a plain `new` on each use |
| `AddScoped<I, T>()` | once per HTTP request (or once per DI "scope") | anything touching the current request/user — **`DbContext` is always scoped** | Nest's default `REQUEST`-scoped provider |
| `AddSingleton<I, T>()` | once for the app's entire lifetime | caches, connection pools, config objects | Nest's default provider scope (singleton) |

**The classic bug**: injecting a `Scoped` service into a `Singleton` service's constructor. The singleton is built once at startup, before any request scope exists, so the DI container throws (or, worse, silently captures a scope that then never gets disposed — a real memory leak). You'll see the *correct* way to work around this in `HRNS.WebApi/BackgroundServices/ArchiveJob.cs` — a `BackgroundService` is registered as a singleton by `AddHostedService`, but it needs a `DbContext` (scoped). The fix: inject `IServiceScopeFactory` (itself safe to be a singleton dependency) and manually open a fresh scope each time you need scoped services:

```csharp
public class ArchiveJob : BackgroundService
{
    private readonly IServiceScopeFactory _serviceScopeFactory;
    public ArchiveJob(IMapper mapper, IHttpClientFactory httpClientFactory, IServiceScopeFactory serviceScopeFactory)
        => _serviceScopeFactory = serviceScopeFactory;

    private async Task ExecuteAsync()
    {
        using var scope = _serviceScopeFactory.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<DbContext>(); // fresh, scoped, disposed at end of `using`
        // ...
    }
}
```
(`HRNS.WebApi/BackgroundServices/ArchiveJob.cs:17-63`, condensed — full background-job pattern in chapter 10.)

### How HRNS organizes its registrations — the "extension method per layer" pattern

Rather than putting hundreds of `services.AddXxx()` calls directly in `Program.cs`, each project exposes its own `DependencyInjection` static class with one extension method that registers everything that project owns, then `Program.cs` calls each in sequence:

```csharp
// HRNS.Database/DependencyInjection.cs
public static IServiceCollection AddPersistence(this IServiceCollection services, IConfiguration configuration, ILoggerFactory loggerFactory, bool isDevEnv)
{
    // ... registers HRNSDbContext, Npgsql + pgvector data source
}

// HRNS.Application/DependencyInjection.cs
public static IServiceCollection AddApplication(this IServiceCollection services)
{
    services.AddAutoMapper(cfg => { cfg.AddMaps(/* every assembly that has mapping profiles */); });
    services.AddMediatR(cfg => { cfg.RegisterServicesFromAssembly(/* every assembly that has handlers */); });
    services.AddScoped<IApplicationSettings, ApplicationSettings>();
    // ... dozens more
    return services;
}

// HRNS.WebApi/Program.cs — the composition root, only place that calls both
services.AddPersistence(webAppBuilder.Configuration, loggerFactory, webAppBuilder.Environment.IsDevelopment());
services.AddApplication();
```

This keeps `HRNS.Database` self-contained (it knows how to register its own `DbContext`) without `HRNS.WebApi` needing to know Npgsql/pgvector setup details, and mirrors exactly what you'd do organizing an Express app's route/module registration into separate files instead of one giant `app.js`. Note the `AddAutoMapper`/`AddMediatR` calls scanning **multiple assemblies explicitly** (`cfg.AddMaps(typeof(UserEntity).Assembly)`, `cfg.AddMaps(Assembly.GetEntryAssembly())`, etc.) rather than one — this is defensive assembly-scanning to make sure mapping profiles and MediatR handlers defined in any of the six+ projects all get discovered and registered automatically, since both libraries work by reflecting over loaded assemblies rather than requiring every handler to be registered by hand.

### `IHttpClientFactory` — one more DI-adjacent thing worth knowing now

Registered via `services.AddHttpClient("SendGridHealth")` (`Program.cs:359`) and consumed as `IHttpClientFactory` — .NET's answer to "don't `new HttpClient()` per-request," which is a well-known .NET footgun (each raw `HttpClient` can exhaust the machine's sockets under load because of how `HttpClient` disposal interacts with the OS's `TIME_WAIT` socket state). The factory pools and reuses underlying connections for you — the equivalent lesson in Node is reusing a single `axios`/`fetch` agent instead of creating a new one per call, just enforced more strictly here.

## 3.4 Swagger / OpenAPI

`Swashbuckle.AspNetCore` auto-generates an OpenAPI spec from your controllers' attributes/types and serves an interactive UI — the .NET equivalent of `swagger-jsdoc`/`@nestjs/swagger`. In HRNS it's wired via `services.AddApiSwagger()` (registration) and `app.UseApiSwaggerUi()` (serving, `ApplicationBuilderExtension.cs:19-39`), reachable at `/api/docs` once the app is running — genuinely useful as a live, browsable map of every endpoint while you're learning the codebase.

## Checkpoint

1. Pick any controller under `HRNS.WebApi/Areas/` you haven't looked at yet. Identify its base class, its `[Route]`, and whether any action overrides `[Authorize]` (look for `[AllowAnonymous]`).
2. In `HRNS.WebApi/Program.cs`, find three services registered with `AddScoped` and one registered with `AddSingleton` (hint: `ConnectionTrackerService`). Explain in one sentence why each makes sense at that lifetime.
3. Run the app locally (or read the Swagger JSON if you can't run it) and open `/api/docs` — pick one endpoint and trace its `[HttpPost]` method name back to the controller class.
