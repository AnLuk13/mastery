# 8. HTTP

Goal: for a full-stack engineer, HTTP should be your single strongest networking subject — you already use it daily; this chapter formalizes what you know and connects it to the transport/security layers underneath, plus HRNS's own, genuinely unconventional take on HTTP routing.

## 8.1 HTTP fundamentals

HTTP (HyperText Transfer Protocol) is a **request/response**, **text-based** (in HTTP/1.1; binary-framed from HTTP/2 onward — §8.4), **stateless** application-layer protocol. "Stateless" means the protocol itself carries no memory of previous requests — each request is handled independently; any notion of "session" (a logged-in user, a shopping cart) is built *on top of* HTTP by the application, typically via cookies (chapter 10) or a bearer token (`../dotnet-mastery/` chapter 8).

```
Request                              Response
─────────────────────────           ─────────────────────────
POST /api/Users/LoginAsync HTTP/1.1   HTTP/1.1 200 OK
Host: hrns.example                    Content-Type: application/json
Content-Type: application/json        Content-Length: 142
Authorization: Bearer ...

{"email": "...", "password": "..."}   {"isSucceeded": true, "items": [...]}
```

Every request has a **method** (§8.2), a **URL/URI** (identifying the resource), **headers** (metadata — content type, auth, caching directives), and an optional **body**. Every response has a **status code** (§8.3), headers, and an optional body.

## 8.2 Methods

| Method | Meaning | Idempotent? |
|---|---|---|
| `GET` | retrieve a resource, no side effects | Yes |
| `POST` | create a resource / trigger an action | No |
| `PUT` | replace a resource entirely | Yes |
| `PATCH` | partially update a resource | Not necessarily |
| `DELETE` | remove a resource | Yes |
| `HEAD` | like `GET` but headers only, no body — cheap existence/metadata check | Yes |
| `OPTIONS` | ask what methods/headers are allowed on this resource | Yes |
| `CONNECT` | establish a tunnel (used by HTTP proxies for HTTPS — chapter 12) | — |
| `TRACE` | diagnostic echo (rarely used in practice; often disabled for security reasons) | Yes |

**Idempotent** means "calling it once has the same effect as calling it many times" — a crucial property for retry logic (`../dotnet-mastery/` chapter 12 §12.6, chapter 14 of this guide): it's safe to blindly retry a `GET` or `DELETE` that timed out (you don't know if the first one succeeded, but retrying causes no *additional* harm), while blindly retrying a non-idempotent `POST` risks a duplicate side effect (charging a card twice, sending an email twice) unless the API was specifically designed to prevent that (an idempotency key is the standard fix, chapter 14).

## 8.3 Status codes

```
1xx  Informational   (rare in day-to-day work)
2xx  Success
3xx  Redirection
4xx  Client error    ("you did something the server won't accept")
5xx  Server error    ("the server failed, not your fault")
```

The ones worth knowing cold: `200` OK, `201` Created, `204` No Content, `301`/`302` (permanent/temporary redirect), `304` Not Modified (chapter 12's HTTP caching), `307`/`308` (redirect that preserves the original method/body, unlike 301/302 which may not), `400` Bad Request, `401` Unauthorized (not authenticated), `403` Forbidden (authenticated, but not allowed), `404` Not Found, `409` Conflict, `429` Too Many Requests (rate limiting), `500` Internal Server Error, `502` Bad Gateway (a proxy/gateway got an invalid response from upstream — chapter 12), `503` Service Unavailable, `504` Gateway Timeout.

**Grounded in this platform**: `HRNSExceptionMiddleware` (`../dotnet-mastery/` chapter 10 §10.5) is the single place that translates an internal business-rule exception (`BadRequestException`, or any `HRNSException` subclass) into the correct status code for the client — e.g. `BadRequestException("Wrong user name or password")` (the login flow, `../dotnet-mastery/` chapter 8 §8.5) becomes an HTTP `400`. This is the concrete mechanism behind "status codes are a *contract*" — the frontend's `BaseApiService` (chapter 10 of this guide) branches on exactly these codes to decide how to react.

## 8.4 HTTP versions — the evolution, and why it happened

```
HTTP/1.0  → one request per connection (connection closed after each response)
HTTP/1.1  → persistent connections (keep-alive) — reuse one TCP connection for many requests
HTTP/2    → multiplexing — many requests IN FLIGHT SIMULTANEOUSLY on one connection
HTTP/3    → runs over QUIC (UDP-based, chapter 5 §5.3) instead of TCP
```

- **HTTP/1.1's persistent connections** were the first big fix for the TCP-handshake-per-request cost (chapter 5 §5.2) — reuse one connection for many sequential requests. Still, requests on one connection are handled one-at-a-time in order (**head-of-line blocking**) — a slow response blocks everything queued behind it on that connection (browsers historically worked around this by opening several parallel connections per host, up to a limit).
- **HTTP/2's multiplexing** solves this properly: many logical request/response **streams** interleaved over a *single* TCP connection, each identified by a stream ID, so a slow response no longer blocks unrelated ones on the same connection. HTTP/2 also introduced **binary framing** (no longer human-readable text on the wire, unlike HTTP/1.1) and **HPACK header compression** (headers repeat enormously across requests to the same host — compressing them is a real, measurable win).
- **HTTP/3 replaces TCP with QUIC** specifically to fix HTTP/2's *remaining* head-of-line blocking problem — HTTP/2 fixed it at the HTTP layer, but a single *lost TCP packet* still blocks every multiplexed stream, because TCP itself doesn't know about HTTP's streams, only about one ordered byte stream. QUIC (UDP-based) implements multiplexing *at the transport layer itself*, so a lost packet only blocks the one stream it belonged to. QUIC also folds the TLS handshake (chapter 9) into its own connection-establishment handshake, saving a further round-trip, and supports **connection migration** (a QUIC connection can survive your device switching from Wi-Fi to cellular mid-request, since it's identified by a connection ID rather than the traditional TCP four-tuple — chapter 6 §6.2).

You don't need to configure HTTP versions yourself in most day-to-day work — Kestrel and modern browsers negotiate the best mutually-supported version automatically — but recognizing *why* each version exists (what specific problem it fixed) is what makes this progression memorable rather than trivia.

## 8.5 HRNS.Platform.Server's unconventional routing — worth revisiting with fresh eyes

You've already read this in `../dotnet-mastery/` chapter 3 §3.2, but it's worth re-reading now with pure networking vocabulary rather than ASP.NET Core vocabulary: **every controller action in this codebase is `[HttpPost]`, even the ones that only read data.**

```csharp
[Route("api/[controller]/[action]")]
```
means the method name is part of the URL (`api/TicketingTicketType/GetTicketingTicketTypeAsync`), and every "query" endpoint accepts its filter/sort/paging parameters in the **body**, not the query string. This is a deliberate divergence from typical REST convention (where `GET` is idempotent, side-effect-free, and cacheable by intermediate caches/CDNs *specifically because* it's `GET` — chapter 12's caching chapter revisits this) — a real, worth-having-an-opinion-on trade-off: HRNS gains the ability to send arbitrarily large, structured filter objects without URL-length limits, at the cost of losing HTTP-level cacheability and the semantic "this is safe to retry blindly" guarantee `GET`/idempotency normally provides for free.

## 8.6 A brief word on how modern APIs are described, beyond raw HTTP verbs

- **REST** — the "everything is a resource, methods are verbs on it" style you already know; not a protocol, a set of conventions layered on HTTP.
- **RPC** (Remote Procedure Call) — "call this named function on a remote server," conceptually closer to what HRNS's `api/[controller]/[action]` convention actually is, despite being built on HTTP the same way REST is.
- **gRPC** — a specific, high-performance RPC framework built explicitly on HTTP/2 (using its multiplexing and binary framing directly) with Protocol Buffers (a compact binary serialization format) instead of JSON — common in internal service-to-service communication (chapter 14) where every byte and every millisecond of serialization overhead compounds at scale; not used in this platform, which speaks JSON-over-HTTP/1.1-or-2 throughout.
- **GraphQL** — a query language layered *on top of* a single HTTP endpoint, letting the client specify exactly which fields it wants; also not used here (this platform's DTO/STO pattern, `../dotnet-mastery/` chapter 5 §5.3, is closer in spirit to REST/RPC's "the server decides the shape").

## Checkpoint

1. Explain why HTTP/2 multiplexing over one TCP connection is still vulnerable to head-of-line blocking at the *TCP* level, and why that's specifically what motivated HTTP/3/QUIC.
2. `SaveTicketingTicketTypeAsync` (`../dotnet-mastery/` chapter 5 §5.5) is a `POST`. Is it idempotent, by this chapter's definition? Does that match how `CommandBaseHandler`'s upsert-by-ID logic (`../dotnet-mastery/` chapter 5 §5.4) actually behaves if the exact same request is sent twice?
3. If HRNS's "everything is POST" endpoints were instead built as conventional REST (`GET /api/tickets?filter=...`), what would you gain, and what would you lose, given the filter objects these queries actually pass (`PropsFilter`, `Sorts[]`, `Page`, `../dotnet-mastery/` chapter 5 §5.3)?
