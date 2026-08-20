# 5. TCP & UDP — the Transport Layer

This is the single most valuable chapter in Part 1 for a working software engineer — nearly everything you build (HTTP, HTTPS, databases, SSH, WebSockets, gRPC) rides on TCP, and understanding it properly is what makes concepts like "connection pooling," "the N+1 socket-exhaustion problem," and "why did this request hang instead of erroring" stop being mysteries.

## 5.1 What the transport layer actually adds

The network layer (chapter 4, IP) gets a packet from one *host* to another. It says nothing about which *process* on that host should receive it, and nothing about whether the packet even arrives intact, in order, or at all — IP is a "best effort" layer by design. The transport layer solves both problems: **ports** (chapter 6) identify which process, and **TCP** (this chapter) adds reliability on top of IP's unreliable delivery.

## 5.2 TCP — connection-oriented, reliable, ordered

TCP (Transmission Control Protocol) turns IP's unreliable packet delivery into something applications can build on without thinking about loss, reordering, or duplication themselves. It does this with real, understandable mechanisms:

- **Reliability**: every TCP segment sent must be acknowledged (**ACK**) by the receiver; unacknowledged segments get retransmitted automatically.
- **Ordered delivery**: every byte in a TCP stream has a **sequence number**; the receiver reassembles data in the correct order even if segments arrive out of order over the network, and holds back out-of-order data until the gap is filled.
- **Flow control**: the receiver advertises a **receive window** — "don't send me more than this much unacknowledged data" — so a fast sender can't overwhelm a slow receiver.
- **Congestion control**: TCP also self-limits based on *network* conditions (not just the receiver's capacity) — it starts sending cautiously and ramps up, backing off sharply if it detects loss (a classic algorithm here is "slow start" + "additive increase, multiplicative decrease"). This is *why* a single TCP connection over a lossy/congested path gets progressively slower rather than just failing outright.

### The three-way handshake

Before any data flows, TCP establishes a connection:

```
Client                              Server
  │                                    │
  │ ── SYN (seq=x) ──────────────────► │   "I want to connect, my starting sequence number is x"
  │                                    │
  │ ◄──── SYN-ACK (seq=y, ack=x+1) ─── │   "OK, acknowledged; my starting sequence number is y"
  │                                    │
  │ ── ACK (ack=y+1) ────────────────► │   "Acknowledged; connection established"
  │                                    │
  │        CONNECTION ESTABLISHED       │
```

This handshake is real, measurable latency cost — it's one full round-trip before the *first* byte of actual application data can even be sent (TLS, chapter 9, adds its own further round-trip(s) on top for HTTPS) — which is exactly why connection *reuse* (keep-alive, connection pooling) is such a consistently recurring performance theme across this whole guide and `../dotnet-mastery/`.

### Connection termination, and the state you'll actually see in tools

```
Client                              Server
  │ ── FIN ──────────────────────────► │   "I'm done sending"
  │ ◄──────────────────────────── ACK ─│
  │ ◄──────────────────────────── FIN ─│   "I'm done too"
  │ ── ACK ──────────────────────────► │
```
(TCP connections are full-duplex and closed independently in each direction — this is why you'll sometimes see "half-closed" connections in diagnostic tools.) A **RST** (reset) is an abrupt, non-graceful termination — sent when, e.g., a packet arrives for a connection that doesn't exist (nothing listening on that port) or an application forcibly aborts a connection rather than closing it cleanly.

**TIME_WAIT** is the one TCP connection-state detail every backend engineer eventually needs to know by name: after closing, the side that sent the *last* ACK holds the connection in `TIME_WAIT` for a couple of minutes (historically ~4 minutes, tunable) before fully releasing it — a deliberate safety margin to make sure any stray, delayed duplicate packets from the old connection don't get misinterpreted as belonging to a *new* connection that happens to reuse the same port pair. This is directly why creating a fresh `HttpClient` for every outgoing request is a real, well-documented .NET performance footgun: each one that gets disposed leaves its socket in `TIME_WAIT`, and under load you can genuinely exhaust the machine's available ephemeral ports (chapter 6 §6.3) — exactly the reasoning behind `HRNS.Platform.Server` using `IHttpClientFactory` everywhere instead (`../dotnet-mastery/` chapter 3 §3.3), which pools and reuses underlying connections specifically to avoid this.

## 5.3 UDP — connectionless, unordered, fast

UDP (User Datagram Protocol) is TCP's opposite: no handshake, no acknowledgments, no retransmission, no ordering guarantee, no congestion control — just "send this datagram, best effort, and move on." What UDP gives up in guarantees, it gains in simplicity and speed: no connection-setup round-trip, no head-of-line blocking (a lost UDP datagram doesn't block delivery of the ones that follow it, unlike a lost TCP segment, which stalls the whole ordered stream until it's retransmitted).

```
TCP                                UDP
Reliable                           Best-effort
Ordered                            No ordering guarantee
Connection-oriented (handshake)    Connectionless
Flow/congestion control            None built in
Higher overhead per byte           Lower overhead
```

UDP is the right choice whenever an application either doesn't need TCP's guarantees (DNS queries, chapter 7 — a lost query just gets retried by the resolver) or needs to build its *own*, different reliability semantics on top rather than inherit TCP's specific trade-offs (video/voice, where a late retransmitted frame is worse than just dropping it and moving on) — or, notably, **QUIC** (the transport underneath HTTP/3, chapter 8), which reimplements TCP-like reliability *on top of* UDP in user-space specifically to escape TCP's head-of-line blocking and to make protocol evolution faster (TCP is baked into OS kernels and evolves glacially; QUIC, being UDP-based, is just application code).

## 5.4 Grounded in this platform: three real examples of "something is a TCP connection"

**Postgres, via Npgsql.** Every `HRNSDbContext` operation ultimately rides on a pooled TCP connection to PostgreSQL (`../dotnet-mastery/` chapter 4 §4.4 covered connection pooling at the EF Core level; this is the transport layer *underneath* that pooling) — this is precisely why `_dbContext.Database.SetCommandTimeout(...)`, called in every `QueryBaseHandler`/`CommandBaseHandler.DoHandle()` (`../dotnet-mastery/` chapter 5 §5.4), matters: it's bounding how long the app will wait on a single TCP-connection-bound database round-trip before giving up, rather than hanging indefinitely if the database is slow or unreachable.

**ClamAV, over a raw TCP socket, no HTTP at all.**
```csharp
var clamAvUriString = config.GetValue<string>("ClamAv:tcp");
var clamAvUri = new Uri(clamAvUriString);
return new ClamAvScannerService(clamAvUri);
```
(`../dotnet-mastery/` chapter 3 §3.3, `HRNS.WebApi/Program.cs:386-398`.) This is a genuinely good example to hold onto: not every network integration in a real app is HTTP — ClamAV speaks its own simple line-based protocol directly over a raw TCP socket, no HTTP framing, no JSON, nothing above the transport layer at all. `ClamAvHealthCheck` (`../dotnet-mastery/` chapter 10 §10.4) exists specifically because this connection, being TCP but not HTTP, isn't something a generic HTTP health probe would even know how to test.

**SignalR — one persistent TCP connection carrying many logical messages.** `AIAssistantHub`/`QrCodeHub` (`../dotnet-mastery/` chapter 10 §10.2, chapter 11 of this guide goes deeper) each hold one long-lived connection open (ideally upgraded to a WebSocket, itself running over one single TCP connection — chapter 11) for the entire duration a user has that page open, rather than opening a new TCP connection per message the way a naive polling implementation would. This is *why* real-time features are architected around persistent connections at all: avoiding the handshake round-trip cost (§5.2) on every single message is the entire point.

## Checkpoint

1. Explain in your own words why UDP is a reasonable choice for a DNS query (chapter 7) but would be a poor choice for downloading a large file.
2. `IHttpClientFactory` pools and reuses `HttpClient` instances/underlying connections (`../dotnet-mastery/` chapter 3 §3.3). Using this chapter's TIME_WAIT explanation, articulate specifically *what* resource gets exhausted if an app instead creates and disposes a fresh `HttpClient` for every single outgoing request under sustained load.
3. A TCP `RST` and a TCP connection that just... never responds (no SYN-ACK, no RST, nothing) look different to the client. Which one indicates "nothing is listening on that port, but the host is reachable," and which one is consistent with "a firewall is silently dropping my packets"? (Chapter 13 will make this distinction concrete — for now, reason from what you know about how TCP is supposed to behave.)
