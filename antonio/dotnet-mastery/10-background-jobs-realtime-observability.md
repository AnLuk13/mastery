# 10. Background Jobs, Real-Time & Observability

Goal: `IHostedService`/`BackgroundService` for recurring work, SignalR for server-push, Serilog for structured logging, health checks, and centralized exception handling — the cross-cutting machinery that keeps a long-running server observable and self-maintaining.

## 10.1 `BackgroundService` — recurring work outside the request/response cycle

ASP.NET Core hosts long-running background work as **hosted services**: classes registered with `services.AddHostedService<T>()` that the framework starts at app boot and stops at shutdown, running independently of any individual HTTP request. `BackgroundService` is the standard abstract base — you implement one method, `ExecuteAsync(CancellationToken stoppingToken)`, and the framework calls it once at startup; everything after that (looping, waiting, retrying) is on you. Conceptually the same job a `node-cron` job or a separate worker process does in a Node app — except here it runs in the *same process* as the web server, sharing its DI container.

`HRNS.WebApi/BackgroundServices/` has a dozen of these — `ArchiveJob`, `NotificationsBGService`, `ScheduledNotificationsBGService`, `PayrollCalendarReminderJob`, `TicketingEmailPollingBGService`, and more — each registered in `Program.cs`:

```csharp
services.AddHostedService<NotificationsBGService>();
services.AddHostedService<ArchiveJob>();
services.AddHostedService<CalculateUserStatsJob>();
var payrollRemindersEnabled = webAppBuilder.Configuration.GetValue<bool?>("PayrollCalendarReminders:Enable") ?? true;
if (payrollRemindersEnabled) services.AddHostedService<PayrollCalendarReminderJob>();
```
(`HRNS.WebApi/Program.cs:198-220`, condensed — note the same config-flag-gated registration pattern from chapter 2 §2.5/chapter 3 §3.3, applied here to whether a job runs at all.)

The recurring "poll on an interval" shape, from `ArchiveJob` (already introduced in chapter 3 §3.3 for its DI-scope trick — here's the full loop):

```csharp
protected override Task ExecuteAsync(CancellationToken stoppingToken) => ExecuteAsyncLoop(stoppingToken);

private async Task ExecuteAsyncLoop(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        try { await ExecuteAsync(); }
        catch { continue; }
        finally { await Task.Delay(TimeSpan.FromMinutes(1)); }
    }
}
```
(`HRNS.WebApi/BackgroundServices/ArchiveJob.cs:36-59`.) Worth reading critically, not just descriptively (same spirit as chapter 8's security review): the bare `catch { continue; }` swallows *every* exception silently, including ones you'd genuinely want to know about (a misconfiguration, a bug) — it optimizes for "never let one bad iteration kill the whole background loop forever," which is a real and reasonable goal for a long-running service, but at the cost of visibility. A stricter version would at least log the exception before continuing. Good instinct to develop: "this loop never crashes" and "this loop's failures are invisible" are two different properties, and code that guarantees the first doesn't automatically get you the second.

`stoppingToken` is the graceful-shutdown signal — ASP.NET Core cancels it when the app is stopping, and a well-behaved loop checks it (as this one does) so `Task.Delay` and any awaited work wind down promptly instead of the process being killed mid-operation.

## 10.2 SignalR — real-time, server-to-client push

SignalR is ASP.NET Core's real-time communication library — WebSockets with automatic fallback (long polling, etc.), and a higher-level RPC-style API on top, similar in spirit to `socket.io`. A **Hub** is the server-side endpoint clients connect to and call methods on (and that the server can call back on, per-client or broadcast).

```csharp
services.AddSignalR(options =>
{
    options.EnableDetailedErrors = true;
    options.ClientTimeoutInterval = TimeSpan.FromSeconds(30);
});
// ...
app.MapHub<QrCodeHub>("/hub/qrcode");
app.MapHub<AIAssistantHub>("/hub/ai-assistant");
```
(`HRNS.WebApi/Program.cs:190-194, 129-130`.) Two hubs exist in this codebase: `QrCodeHub` (pushes clock-in confirmation the instant a QR code is scanned — chapter 8 §8.6's AES-GCM QR codes are exactly what's being scanned) and `AIAssistantHub` (`HRNS.WebApi/Areas/AiAssistantHubs/AIAssistantHub.cs`, streams the AI assistant's response tokens to the browser as they're generated — chapter 13). Both are protected the normal `[Authorize]` way, verified explicitly by their own `SignalRHealthCheck` (§10.4) — worth remembering that a hub without `[Authorize]` is a real, easy-to-miss way to accidentally expose a push channel to anonymous clients.

## 10.3 Serilog — structured logging

The built-in `ILogger<T>` (Microsoft.Extensions.Logging) is the .NET logging abstraction (interface); Serilog is a concrete, more capable *provider* plugged in underneath it — richer sinks (console, rolling files, and commonly external aggregators like Seq/Elasticsearch, though this repo's configured sinks are console + file), and, crucially, **structured** logging: you log named properties, not just an interpolated string, so log data stays queryable instead of being flattened into unstructured text.

```csharp
Host.CreateDefaultBuilder(args)
    .ConfigureWebHostDefaults(webBuilder => webBuilder.UseStartup<Program>())
    .UseSerilog((webBuilder, configuration) =>
        configuration.ReadFrom.Configuration(configSerilog).Enrich.FromLogContext(),
        writeToProviders: true);
```
(`HRNS.WebApi/Program.cs:96-102`.) Configuration comes from `appsettings.json`'s `Logging` section (chapter 2 §2.5 — `PathFormat`, per-namespace `LogLevel`). Structured-logging discipline, seen throughout the codebase:

```csharp
startupLogger.LogInformation("Ollama model validation passed. Chat model: {ChatModel}, Embedding model: {EmbeddingModel}", ollamaModel, embeddingModel);
```
(`Program.cs:279`.) `{ChatModel}`/`{EmbeddingModel}` are **named placeholders**, not string interpolation (`$"...{ollamaModel}"`) — Serilog captures them as separate, queryable fields on the log event (so you could later query "every log line where `ChatModel = 'llama3'`"), not just bake them into one opaque message string. Always prefer this form over `$"..."` when logging through `ILogger`.

## 10.4 Health checks

`Microsoft.Extensions.Diagnostics.HealthChecks` exposes an endpoint (`/api/health` here) that reports whether the app and its dependencies are actually working — used by load balancers/orchestrators to decide whether to route traffic to an instance, or restart it. HRNS registers **ten** independent checks, one per external dependency:

```csharp
services.AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>("Database", failureStatus: HealthStatus.Degraded, tags: new[] { "critical" }, timeout: TimeSpan.FromSeconds(10))
    .AddCheck<FileServerHealthCheck>("File Server", ...)
    .AddCheck<ClamAvHealthCheck>("ClamAV Scanner", ...)     // malware-scans uploaded files — chapter 14
    .AddCheck<AiModelHealthCheck>("AI Model", ...)          // Ollama connectivity — chapter 13
    .AddCheck<SignalRHealthCheck>("SignalR Hubs", ...);     // verifies hub registration + [Authorize]
```
(`Program.cs:364-385`, condensed.) Each is a small class implementing `IHealthCheck`:

```csharp
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly IServiceScopeFactory _scopeFactory; // same singleton-needs-scoped-service trick as §10.1/chapter 3 §3.3

    public async Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken cancellationToken = default)
    {
        using var scope = _scopeFactory.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<HRNSDbContext>();
        var canConnect = await dbContext.Database.CanConnectAsync(cancellationToken);
        return canConnect ? HealthCheckResult.Healthy() : HealthCheckResult.Unhealthy("...");
    }
}
```
(`HRNS.WebApi/HealthChecks/DatabaseHealthCheck.cs`, condensed.) `failureStatus: HealthStatus.Degraded` (rather than the more severe `Unhealthy`) plus tagging each check `"critical"`/`"important"` lets ops distinguish "the whole app is down" from "one non-essential integration is down but the core app still works" — a real, deliberate signal-quality decision, not boilerplate.

## 10.5 Centralized exception handling

Rather than wrapping every controller action in `try/catch`, ASP.NET Core's `app.UseExceptionHandler(...)` middleware (chapter 3 §3.1) catches *any* unhandled exception anywhere downstream in the pipeline and routes it to one handler:

```csharp
app.UseExceptionHandler(x => x.UseMiddleware(typeof(HRNSExceptionMiddleware)));
```

```csharp
public class HRNSExceptionMiddleware
{
    public async Task Invoke(HttpContext httpContext)
    {
        var exceptionFeature = httpContext.Features.Get<IExceptionHandlerPathFeature>();
        if (exceptionFeature?.Error is HRNSException e)
        {
            httpContext.Response.StatusCode = (int)e.StatusCode;
            httpContext.Response.ContentType = "application/json";
            await httpContext.Response.WriteAsJsonAsync(new { status = (int)e.StatusCode, error = e.Errors.Select(x => x.Value).Join(","), message = e.Message });
        }
        else if (exceptionFeature?.Error is BadRequestException bre) { /* 400, similar shape */ }
        // ... further exception types mapped to their own status codes
    }
}
```
(`HRNS.WebApi/Middleware/Exception/HrnsysExceptionMiddleware.cs`, condensed.) A custom `HRNSException` base type carries its own `StatusCode` and structured `Errors` — meaning a handler deep in `HRNS.Application` can `throw new BadRequestException("Wrong user name or password")` (you saw this exact line in chapter 8's login walkthrough) with zero knowledge of HTTP at all, and this one middleware, at the very edge of the pipeline, is the single place that translates "a business-rule violation happened" into "the correct HTTP status code and JSON shape." This is the same principle as chapter 6's layering discussion applied to error handling specifically: business logic never needs to know it's running inside a web server.

## Checkpoint

1. Pick one background job you haven't read yet (`SignOffNotificationJob`, `SyncLoanDocumentLegalEntitiesJob`, or similar) and identify: does it follow the same poll-loop shape as `ArchiveJob`, and does it swallow exceptions the same way?
2. Find `HRNS.Models.Exceptions` and list every exception type that derives from `HRNSException`. What HTTP status code does each map to, and can you find a real `throw` site for at least two of them?
3. `AiModelHealthCheck` and `RecaptchaHealthCheck` are tagged `"important"` rather than `"critical"` in `Program.cs`. If AI or reCAPTCHA were unreachable, what should still work in this app, and does that match the tag?
