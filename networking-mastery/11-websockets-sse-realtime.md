# 11. WebSockets, SSE & Real-Time

Goal: how a persistent, bidirectional connection actually works over what started as a one-shot request/response protocol — and how this platform's two SignalR hubs (introduced in `../dotnet-mastery/` chapter 10 §10.2) actually get their real-time behavior on the wire.

## 11.1 The problem: HTTP is fundamentally request/response

Plain HTTP (chapter 8) has no built-in way for a server to push data to a client unprompted — the client always initiates. For a long time, the only way to fake "real-time" was **polling**: the client repeatedly asks "anything new?" on a timer. **Long polling** improved on naive polling by having the server *hold the request open* until it actually has something to say (or a timeout expires), then the client immediately reopens a new request — better latency, still fundamentally one HTTP request per update, with all of chapter 5 §5.2's connection-establishment overhead recurring constantly.

## 11.2 WebSockets — a real persistent, bidirectional connection

A WebSocket connection starts as a normal HTTP request, then **upgrades** — the same underlying TCP connection (chapter 5) is repurposed for a completely different framing protocol, no longer HTTP at all:

```
Client                                             Server
  │ ── GET /hub/ai-assistant HTTP/1.1 ────────────► │
  │    Upgrade: websocket                             │
  │    Connection: Upgrade                              │
  │    Sec-WebSocket-Key: <random>                        │
  │                                                     │
  │ ◄─ HTTP/1.1 101 Switching Protocols ──────────────  │
  │    Upgrade: websocket                                │
  │    Sec-WebSocket-Accept: <computed from the key>       │
  │                                                          │
  │   ══════ now a WebSocket connection, NOT HTTP ══════      │
  │   ◄────────────── either side can send a frame ────────►  │
```

The `101 Switching Protocols` status code (chapter 8 §8.3) is the tell — after that response, the *same TCP connection* stops speaking HTTP entirely and starts speaking the WebSocket framing protocol instead: a lightweight, message-oriented framing (not a raw byte stream the way TCP itself is — each WebSocket **frame** carries an explicit message boundary, so the receiver doesn't need application-level delimiters the way raw TCP consumers often do). Either side can send a frame at any time, with no request/response pairing required — genuine full-duplex communication over one long-lived TCP connection.

`ws://` is a WebSocket connection over plain (unencrypted) TCP; `wss://` is over TLS (chapter 9) — the direct WebSocket analogue of `http://` vs `https://`, and just as non-optional for anything carrying real user data.

## 11.3 Server-Sent Events (SSE) — the simpler, one-directional alternative

**SSE** is a much lighter-weight mechanism: a single, long-lived, *plain* HTTP response (`Content-Type: text/event-stream`) that the server keeps writing new events into over time, and the browser's built-in `EventSource` API consumes incrementally — no protocol upgrade, no special framing, just a response that never quite finishes. The trade-off is right there in the name: SSE is **server → client only**; there's no built-in mechanism for the client to send anything back over that same connection (a client that also needs to send data uses a normal, separate HTTP request for that). SSE also gets automatic reconnection *for free*, built into the browser's `EventSource` — a real advantage WebSockets don't provide natively (reconnection logic for a dropped WebSocket has to be hand-rolled by the application).

| | WebSocket | SSE | Long polling |
|---|---|---|---|
| Direction | bidirectional | server → client only | client-initiated, server responds when ready |
| Protocol | upgrades away from HTTP entirely | stays plain HTTP | plain HTTP, repeated |
| Built-in reconnect | no (hand-rolled) | yes (`EventSource`) | trivially — it's just "make another request" |
| Typical use | chat, collaborative editing, gaming, this platform's AI-assistant token streaming | live dashboards, notification feeds, server-push-only cases | fallback when neither of the above is available |

## 11.4 How this platform's SignalR hubs actually use all three

`../dotnet-mastery/` chapter 10 §10.2 already introduced `QrCodeHub` and `AIAssistantHub` as SignalR endpoints, and chapter 13 covered `AIAssistantHub` streaming response tokens as they're generated. What wasn't explained there, and is worth knowing now: **SignalR doesn't force a single transport — it negotiates the best one available and falls back automatically.** Concretely, in priority order: **WebSocket** (§11.2, used whenever available — the browser, the network path, and the server all have to support it), then **Server-Sent Events** (§11.3, if WebSocket negotiation fails — e.g. a restrictive corporate proxy that blocks the `Upgrade` header but allows plain long-lived HTTP responses through), then **long polling** (§11.1, the universal fallback that works essentially anywhere plain HTTP works, at the cost of the highest latency/overhead of the three).

This is a genuinely elegant, real-world application of everything in this chapter: SignalR's `HubConnectionBuilder` (the client-side `@microsoft/signalr` library `HRNS.Platform.UI` uses, per its own `CLAUDE.md`) handles this negotiation transparently, so `AIAssistantHub`'s response-streaming code (`../dotnet-mastery/` chapter 13 §13.2's `await foreach`) doesn't need to know or care which of the three transports is actually carrying its data on any given user's network — the exact same "each layer solves one problem and trusts the layers around it" principle from chapter 1 §1.2, now visible in a real feature you've already read the application-layer code for.

`app.MapHub<QrCodeHub>("/hub/qrcode")`/`app.MapHub<AIAssistantHub>("/hub/ai-assistant")` (`../dotnet-mastery/` chapter 3 §3.1) is the ASP.NET Core side of this negotiation — it's what actually handles the `Upgrade` handshake (or serves the SSE/long-polling fallback response) at those specific routes.

## Checkpoint

1. If a corporate network's proxy strips the `Upgrade`/`Connection: Upgrade` headers from outgoing requests (a real, if increasingly rare, restrictive-network scenario), what would you expect to observe in the browser's Network tab when connecting to `AIAssistantHub`, given §11.4's fallback order?
2. Explain why SSE, despite being "just HTTP," still requires the connection to stay open indefinitely rather than closing after the response body is sent — what would break if a proxy or load balancer in front of the server enforced a strict per-request timeout (chapter 12 revisits load balancer configuration)?
3. `QrCodeHub` pushes a clock-in confirmation the instant a QR code is scanned (`../dotnet-mastery/` chapter 10 §10.2). Could this feature have been implemented with plain long polling instead of SignalR/WebSockets? What would the user-visible trade-off have been?
