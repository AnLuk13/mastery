# 12. Proxies, Load Balancing & CDNs

Goal: the infrastructure layer sitting between "a client" and "your application" in essentially every real production deployment — critical for understanding how a request from a browser actually reaches `HRNS.Platform.Server`, and why several things you've already read (TLS termination, `ForwardedHeadersOptions`, health checks) exist at all.

## 12.1 Forward proxy vs. reverse proxy — the distinction that confuses almost everyone at first

Both are "a server that sits between a client and a destination server, forwarding traffic" — the difference is *whose interests* the proxy represents:

- **Forward proxy** — represents the **client**. Sits in front of a group of clients, forwarding *their* outbound requests to arbitrary destinations on the internet, often for filtering, caching, or anonymity (a corporate network's outbound web proxy is the classic example — the destination server often doesn't even know a proxy was involved).
- **Reverse proxy** — represents the **server**. Sits in front of a group of backend servers, receiving inbound requests from the internet and forwarding them to whichever backend should actually handle each one — the *client* often doesn't know a proxy was involved at all, and only ever sees the reverse proxy's own address.

```
Forward proxy:                          Reverse proxy:

Client ──► Forward Proxy ──► Internet     Client ──► Reverse Proxy ──► Backend server(s)
(proxy represents the client)              (proxy represents the server)
```

`HRNS.Platform.Server` is deployed behind a **reverse proxy** in any real environment (chapter 11's `ApplicationBuilderExtension`/DevOps chapter of `../dotnet-mastery/` already assumed this) — this is exactly why `ForwardedHeadersOptions`/`X-Forwarded-*` (chapter 4 §4.2) and TLS termination at the proxy (chapter 9 §9.5) both exist: the app never sees the real client directly, it sees the reverse proxy, and relies on headers the proxy adds to recover the original request's true details.

A **transparent proxy** is a forward proxy that intercepts traffic without the client being configured to use it at all (common at the ISP or corporate-firewall level) — the client believes it's talking directly to the destination. An **HTTP proxy** operates at the application layer, understanding HTTP well enough to inspect/modify requests; a **SOCKS proxy** operates lower, at roughly the transport layer, simply relaying bytes for *any* protocol without understanding HTTP specifically — SSH's dynamic port forwarding (chapter 16) is literally a SOCKS proxy.

## 12.2 Load balancing — a reverse proxy's most common job

A **load balancer** is a reverse proxy specialized for one purpose: distributing incoming traffic across *multiple* backend server instances, so no single instance is overwhelmed and so the system keeps working if one instance fails.

```
                Load Balancer
                /     |     \
               /      |      \
          Server1  Server2  Server3
```

- **Layer 4 load balancing** — operates at the transport layer (chapter 5): distributes raw TCP/UDP connections based on IP/port alone, with no understanding of HTTP at all. Fast, simple, but can't make routing decisions based on URL path, headers, or content.
- **Layer 7 load balancing** — operates at the application layer (chapter 8): understands HTTP, and can route based on the actual request — path-based routing (`/api/*` to one backend, `/static/*` to another), header-based routing, and more. Almost every modern web load balancer/API gateway operates here.
- **Round robin** — cycle through backends in order, one request each.
- **Least connections** — send the next request to whichever backend currently has the fewest active connections — better than round robin when requests have wildly varying processing times.
- **Health checks** — the load balancer periodically probes each backend (often that backend's own dedicated health endpoint) and stops routing traffic to any instance that's failing. This is *exactly* what `HRNS.Platform.Server`'s `/api/health` endpoint and its ten individual `IHealthCheck` implementations (`../dotnet-mastery/` chapter 10 §10.4) exist to be consumed by — you already understand the *application* side of this deeply; this chapter supplies the missing *infrastructure* side: a load balancer/orchestrator hitting that exact endpoint on a timer is the actual consumer of the "Database: Degraded" / "critical" vs. "important" tag distinction that chapter explained.
- **Sticky sessions** — routing all of one client's requests to the *same* backend instance for the duration of a session, typically via a cookie the load balancer sets — necessary if a backend holds in-memory state per-session (this platform's `ConnectionTrackerService`, `../dotnet-mastery/` chapter 3 §3.3, registered as a singleton, is exactly the kind of in-process state that would make sticky sessions relevant for SignalR connections specifically, once you're running more than one app instance — chapter 14 revisits this concern from the "distributed state" angle).

## 12.3 API gateways — a reverse proxy with more application-aware jobs

An **API gateway** is a reverse proxy that's grown additional, application-layer responsibilities beyond plain routing/load-balancing: authentication/authorization enforcement, rate limiting, request/response transformation, aggregating calls to multiple backend services into one client-facing endpoint. Whether a given deployment uses a dedicated API gateway product or handles these concerns inside the application itself (as `HRNS.Platform.Server` does — its own JWT validation and `AccessDomainIds` authorization, `../dotnet-mastery/` chapter 8 §8.4, run *inside* the app rather than at a gateway in front of it) is an architectural choice with real trade-offs: gateway-level enforcement centralizes policy across many backend services at the cost of an extra network hop and a separate system to operate; in-app enforcement keeps everything in one place at the cost of every service having to implement its own (though this platform's `BaseHandler`/`BaseController` pattern, `../dotnet-mastery/` chapter 5–6, is itself exactly the kind of shared-base-class centralization that mitigates *that* particular downside without needing a separate gateway process.)

## 12.4 CDNs — reverse proxies distributed geographically

A **CDN** (Content Delivery Network) is, structurally, a reverse proxy — but deliberately deployed at many **edge** locations around the world, geographically close to end users, caching content so a user's request never has to travel all the way to the **origin server** (the actual backend) if a nearby edge node already has a cached copy.

```
User (São Paulo)
      │
      ▼
Nearest edge node (also in/near São Paulo)
      │
      ├── cache HIT  → response served from the edge, low latency, origin never touched
      │
      └── cache MISS → edge fetches from origin, caches it, THEN responds
```

This directly attacks the **latency** problem from chapter 1 §1.3 — physical distance has a real, unavoidable speed-of-light floor on round-trip time, and a CDN's entire value proposition is putting a cached copy of your content physically closer to the request. **Anycast** (the same IP address announced from many physical locations, with routing — chapter 4 §4.1 — naturally directing each user to the topologically nearest one) is the networking trick that makes "nearest edge node" work without the client needing to know anything about geography itself.

`HRNS.Platform.Server`'s API responses (mostly per-user, authenticated, frequently-changing business data) are largely *not* good CDN-caching candidates — but `HRNS.Platform.UI`'s built, static frontend bundle (JS/CSS/images, produced by `npm run build`, per the UI project's own tooling) is a textbook CDN use case: identical for every user until the next deploy, safe to cache aggressively and serve from edge locations worldwide.

## 12.5 HTTP caching — the mechanism CDNs (and browsers) actually use

**`Cache-Control`** is the primary response header controlling caching behavior: `max-age=3600` (cacheable for an hour), `no-store` (never cache at all — appropriate for this platform's authenticated API responses), `private` (cacheable only by the end user's own browser, not by a shared cache/CDN in between — relevant for anything containing per-user data), `public` (cacheable by anyone in the chain, including CDNs).

**Validation, rather than pure expiry** — `ETag` (an opaque token representing a specific version of a resource) and `Last-Modified` (a timestamp) let a client that already has a *possibly-stale* cached copy ask the server "has this changed?" cheaply, without re-downloading the whole thing:

```
Client (has a cached copy with ETag "abc123"):

GET /static/app.js HTTP/1.1
If-None-Match: "abc123"

Server, if unchanged:                      Server, if changed:

HTTP/1.1 304 Not Modified                   HTTP/1.1 200 OK
(empty body — client uses its cached copy)   (full new body + new ETag)
```
`304 Not Modified` (chapter 8 §8.3) is the payoff — the server confirms the client's cached copy is still valid without re-sending the actual content at all, saving bandwidth and latency while still guaranteeing correctness (unlike a pure `max-age`-only approach, which trusts the client's cache blindly until it expires, with no way to invalidate it early). `If-Modified-Since` pairs with `Last-Modified` the same way `If-None-Match` pairs with `ETag`.

## Checkpoint

1. `HRNS.Platform.Server`'s `/api/health` endpoint (`../dotnet-mastery/` chapter 10 §10.4) is exactly what a load balancer would poll. If the `Database` check reports `Degraded` (not `Unhealthy`) due to its `failureStatus` configuration, should a load balancer necessarily stop routing traffic to that instance? Revisit the "critical" vs. "important" tag distinction from that chapter with this chapter's load-balancing vocabulary in mind.
2. Explain why `HRNS.Platform.UI`'s built JS/CSS bundle is a much better CDN-caching candidate than `HRNS.Platform.Server`'s `LoadDTOWebResponse` API responses (`../dotnet-mastery/` chapter 5 §5.5) — name the specific `Cache-Control` directive you'd expect each to use.
3. A reverse proxy in front of `HRNS.Platform.Server` terminates TLS (chapter 9 §9.5) and forwards plain HTTP internally. Using this chapter's vocabulary, is that proxy acting as an L4 or L7 load balancer if it's also making routing decisions based on the request's URL path? Why does that distinction matter for whether it *can* even see the URL path to route on (hint: think about what an L4 balancer actually has visibility into).
