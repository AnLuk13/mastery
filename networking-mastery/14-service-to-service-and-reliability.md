# 14. Service-to-Service Networking & Reliability

This chapter didn't exist as its own topic in the source roadmap either — its concerns were split across "distributed systems networking," "network reliability & performance," and "observability," each just a bullet list. It deserves to be pulled together and grounded properly, because `HRNS.Platform.Server` is genuinely **not a monolith talking only to a database** — it's one node in a small network of services (a File Server, `HRNS.Gelocation.Server`, an Ollama AI server, SendGrid, Firebase, Google reCAPTCHA, ClamAV) and the patterns in this chapter are precisely what makes those calls survive real network conditions instead of falling over the first time a request is slow.

## 14.1 The foundational fact of distributed systems

> A network call can fail, be delayed, be duplicated, or succeed while your client thinks it failed.

Sit with that for a second — it's the single most important sentence in this entire guide for anyone building backend systems. Chapter 1 §1.3 introduced latency/jitter/packet loss as abstract vocabulary; this is what it actually *means* in practice: **every** HTTP call `HRNS.Platform.Server` makes to another service — not just ones across the public internet, even ones to `localhost` in local development — is subject to this. There is no version of "just call the other service" that's exempt from needing to handle failure, delay, and ambiguity explicitly.

## 14.2 Real service-to-service calls in this platform

**Geolocation, via `HRNS.Gelocation.Server`** — every login (`../dotnet-mastery/` chapter 8 §8.5) calls `UserLocationInfoFactory.FillUserLocationInfo(clientIpAddress)`, which makes a real HTTP call to a sibling microservice to resolve a client IP into country/region/city — this is what populates `UserLoginHistoryEntity`'s `CountryCode`/`CountryName`/`Latitude`/`Longitude` fields (`../dotnet-mastery/` chapter 8 §8.5). Worth reading closely, because it does something genuinely smart *before* even making the network call:

```csharp
public static readonly string[] LocalAddresses = ["::1", "127.0.0.1", "10.", "192.168.", "172.16.", ..., "172.31."];

public async Task<UserLocationInfo> FillUserLocationInfo(IPAddress clientIpAddress)
{
    var isLocal = LocalAddresses.Contains(clientIp)
        || LocalAddresses.Any(mask => clientIp.StartsWith(mask))
        || IPAddress.IsLoopback(clientIpAddress);

    if (isLocal && !_forceOnLocalhost)
        return null; // skip the network call entirely — a private/loopback IP has no meaningful geolocation
    // ... otherwise, call HRNS.Gelocation.Server
}
```
(`HRNS.Application/Implementation/Utils/UserLocationInfoFactory.cs:22, 57-70`, condensed.) This is chapter 3 §3.3's private-address ranges, used as a genuine business-logic filter: there's no reason to make a network call asking "where is `192.168.1.10`" — the answer is always "somewhere on a private network, geolocation is meaningless," so the code checks this *locally, with zero network involvement*, before ever reaching for `IHttpClientFactory`. A good general instinct worth generalizing: the cheapest, most reliable network call is the one you correctly determine you don't need to make at all.

**Authentication between services — an API key, not a JWT.** `HRNS.Gelocation.Server`'s own README documents generating an API key (`dotnet run --project ./HRNS.WebGeoApi/HRNS.WebGeoApi.csproj generate-api-key`), and `GeoServerHealthCheck` attaches it as `Authorization: Bearer {geoApiKey}` (`../dotnet-mastery/` chapter 10 §10.4 introduced this health check; here's the auth detail it glossed over). This is worth contrasting deliberately with `../dotnet-mastery/` chapter 8's user-facing JWT auth: a **long-lived, static API key** is a completely different, and completely appropriate, authentication model for **machine-to-machine** calls between two services you operate yourself — there's no "user," no login flow, no expiry-driven refresh cycle needed; just "does this caller possess the shared secret." Using the same `Bearer <token>` header *shape* as user-facing JWT auth is a nice, low-friction convention (both sides parse it identically), even though the token's actual nature (a rotatable static key vs. a short-lived, cryptographically-signed, per-user credential) is fundamentally different.

**Service discovery via the database, not a service registry.** `FileServerHealthCheck` doesn't call one hardcoded File Server address — it queries `FileStorageLocationEntity` records from `HRNSDbContext` first, then health-checks *each one* individually:
```csharp
// Queries ALL FileStorageLocation records from the database and checks
// each one by performing an HTTP GET to its /files-api/health endpoint.
```
(`HRNS.WebApi/HealthChecks/FileServerHealthCheck.cs:18-23`.) This is a real, working instance of **service discovery** — the general problem of "how does a caller find out *where* a callee actually is, given that could change (multiple instances, different regions/tenants, servers coming and going)" — solved here by treating the database as the source of truth for "which file storage locations currently exist," rather than a fixed config value or a dedicated service-registry product (Consul, etcd, Kubernetes' own service discovery — chapter 15). A perfectly reasonable choice for this platform's scale: no extra infrastructure component needed, and the set of file storage locations is exactly the kind of thing that's *already* naturally modeled as data in this app's own database.

## 14.3 Timeouts — the first, non-negotiable reliability primitive

Every network call needs an explicit upper bound on how long you're willing to wait — without one, a single slow/hung dependency can tie up your own resources (threads, connections, `../dotnet-mastery/` chapter 12 §12.1) indefinitely, turning one remote problem into your own outage. You've already seen this pattern repeated consistently across this codebase, now with the "why" made explicit:

```csharp
client.Timeout = TimeSpan.FromSeconds(10);                           // GeoServerHealthCheck
_dbContext.Database.SetCommandTimeout(TimeSpan.FromSeconds(...));      // every CQRS handler, ../dotnet-mastery/ ch.5 §5.4
var ollamaHttpClient = new HttpClient { Timeout = TimeSpan.FromMinutes(3) }; // Program.cs — deliberately longer, since LLM generation is slow
```
Note the *timeout value itself* is a real design decision, not a formality — 10 seconds for a health check (fail fast, this instance shouldn't be considered healthy if a dependency is this slow), 3 minutes for an LLM call (genuinely can take that long, a short timeout would just produce constant false failures). "What should this timeout actually be" is a judgment call specific to what's on the other end, not a number you copy from another call site unexamined.

## 14.4 Retries, backoff & idempotency — handling the "delayed or duplicated" part

A timed-out call might have actually succeeded on the other end (chapter 14.1) — blindly retrying a **non-idempotent** operation (chapter 8 §8.2 — a `POST` that creates something) risks doing it twice. Blindly retrying an **idempotent** one (a `GET`, or a `POST` designed with an idempotency key so the server can recognize and dedupe a retried request) is safe. **Exponential backoff** — waiting progressively longer between retry attempts (1s, 2s, 4s, 8s...) rather than retrying immediately and repeatedly — exists specifically to avoid making a struggling remote service's problem *worse* by hammering it with immediate retries from every failed caller simultaneously (a real, well-documented failure mode called a "retry storm").

`CommandBaseHandler`'s upsert-by-ID logic (`../dotnet-mastery/` chapter 5 §5.4) is, incidentally, naturally close to idempotent by construction: sending the exact same `Upsert...Command` twice looks up the entity by ID both times and updates it identically — a genuine, if accidental, benefit of that generic pattern's design, worth recognizing now that you have the vocabulary for why it matters.

## 14.5 Circuit breakers & backpressure — protecting yourself from a failing dependency

A **circuit breaker** tracks a dependency's recent failure rate and, once it crosses a threshold, stops even *attempting* calls to it for a cooldown period — "failing fast" locally instead of repeatedly waiting out a timeout against something that's clearly down, and giving the struggling dependency room to recover instead of continuing to pile load onto it. **Backpressure** is the related, broader idea of a system signaling "I'm at capacity, slow down" back to whatever's calling it, rather than silently queueing unbounded work until it runs out of memory/connections. Neither is visible as a dedicated library in this codebase's `.csproj` files — this is another honest gap worth naming plainly (in the same spirit as `../dotnet-mastery/` chapter 12 §12.5's caching gap): a library like Polly is the standard .NET choice for adding circuit-breaker/retry-with-backoff policies around `HttpClient` calls declaratively, and would be a natural, well-scoped addition around this platform's File Server/Geo Server/AI calls specifically if their failure/degradation behavior under real load ever became a problem worth solving deliberately rather than relying on the per-call timeouts already in place.

## 14.6 Connection pooling, revisited from the reliability angle

`../dotnet-mastery/` chapter 4 §4.4 covered connection pooling for Postgres; chapter 3 §3.3 covered `IHttpClientFactory` pooling for outbound HTTP; chapter 5 §5.2 of this guide gave you the transport-layer reason (`TIME_WAIT`, ephemeral port exhaustion) *why* pooling matters at all. Pulling this together: **every one of this platform's external dependencies is accessed through some form of pooled, reused connection** — never a fresh TCP handshake per call — which is both a real performance optimization (avoiding chapter 5 §5.2's handshake round-trip repeatedly) and a real reliability one (avoiding the specific, genuinely-seen-in-production failure mode of exhausting a machine's available ports under load).

## 14.7 Observability — knowing a failure happened at all

Structured logging (`../dotnet-mastery/` chapter 10 §10.3's Serilog coverage) is the foundation; in a system with multiple services calling each other, a single user-facing request can fan out into several downstream calls, and **correlation IDs** (this platform's `CorrelationId`, threaded through every `InternalRequest<T>`/`ServiceLoadResponse`/`ServiceSaveResponse`, `../dotnet-mastery/` chapter 5 §5.3) are what let you reconstruct "which of these log lines, across which services, all belong to the same original request" after the fact — genuinely essential once more than one process is involved, and effectively free to add if (as here) the request/response envelope already carries an ID end-to-end. **Distributed tracing** (tools like OpenTelemetry, Jaeger) formalizes this further — automatically propagating a trace ID across service boundaries and visualizing the full call graph and timing of one logical request — not present in this codebase today, but a natural next step if/when the number of inter-service calls grows enough that manually correlating log lines by ID stops being sufficient.

## Checkpoint

1. `UserLocationInfoFactory.FillUserLocationInfo` returns `null` early for local/private IPs (§14.2) without making any network call. If this check were removed, what would actually happen when a developer logs in from `localhost` during local development — would it fail, hang, or silently return meaningless data? (Consider what `HRNS.Gelocation.Server` would realistically do with a `192.168.x.x` address.)
2. `FileServerHealthCheck`'s "discover file servers via the database" approach (§14.2) means adding a new file storage location is a data change, not a code/config deploy. What's the reliability trade-off if that database query itself is slow or the database is briefly unavailable — does the health check's own health depend on the database's?
3. Sketch, in one paragraph, what a circuit breaker (§14.5) wrapped around the `AIServer`/Ollama `HttpClient` call would actually change about this platform's behavior during a period where the Ollama server is completely down — compare it to the current behavior (a 3-minute timeout on every call, per §14.3).
