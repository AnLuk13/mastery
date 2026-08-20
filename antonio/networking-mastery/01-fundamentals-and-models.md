# 1. Fundamentals & Mental Models

Goal: the vocabulary and mental models everything else in this guide assumes — what a network actually is, how data is described at each size ("bits" vs "frames" vs "packets" vs "segments"), and the two layering models (OSI, TCP/IP) you'll use to reason about *any* networking problem for the rest of your career.

## 1.1 What is a network?

A computer network is, minimally, two or more devices (**nodes**) capable of exchanging data. Everything else — the internet, your office Wi-Fi, a Docker container talking to another container — is a variation on that one idea at different scales.

- **Host** — any device with a network address (your laptop, a server, a phone).
- **Client** / **server** — roles, not device types. A "server" is just a host running software that listens for and responds to requests; the same physical machine can be a client to one service and a server to another (exactly what happens when `HRNS.Platform.Server` is a *server* to your browser but a *client* to PostgreSQL and to its own sibling microservices — chapter 14).
- **Peer-to-peer** — nodes talk directly to each other with no fixed client/server role (less relevant to this platform, which is entirely client/server, but worth knowing the term exists — e.g. WebRTC).
- **LAN** (Local Area Network) — a network confined to one physical location (an office, a home, a single data-center rack).
- **WAN** (Wide Area Network) — a network spanning larger distances, typically connecting multiple LANs (a company with offices in two cities, connected over the internet).
- **The Internet** — the global network of networks; technically just an enormous WAN built from voluntarily interconnected, independently-operated networks.
- **Intranet** — a private network (often a company's) using internet protocols internally, not necessarily reachable from the public internet.

```
Client                          Server
  │                                │
  │ ── request ──────────────────► │
  │                                │
  │ ◄──────────────────── response │
  │                                │
```
This request/response shape recurs at every layer of this guide — an HTTP request/response, a TCP handshake, a DNS query/answer — same shape, different altitude.

## 1.2 What's actually traveling through the wire

Before any protocol, understand what "data on the network" physically is: a stream of **bits** (0s and 1s), grouped into **bytes** (8 bits). Above the physical layer, that stream gets structured into progressively larger, named units as you go up the stack — this vocabulary matters because networking people (and error messages, and Wireshark) use these terms precisely, not interchangeably:

| Term | Roughly: | Layer |
|---|---|---|
| **Bit** | a single 0 or 1 | Physical |
| **Frame** | a chunk of data with link-layer (Ethernet/Wi-Fi) addressing wrapped around it | Link |
| **Packet** | a frame's payload, with network-layer (IP) addressing wrapped around it | Network |
| **Segment** (TCP) / **Datagram** (UDP) | a packet's payload, with transport-layer (port) info wrapped around it | Transport |

Each layer **encapsulates** the one above it — an HTTP request becomes the payload of a TCP segment, which becomes the payload of an IP packet, which becomes the payload of an Ethernet frame. Every layer only reads its own header and treats everything inside as opaque payload to pass along. This is *the* foundational idea in networking: **each layer solves one problem and trusts the layers around it to solve theirs.**

A quick, necessary aside on how text itself is represented, since it underlies literally everything transmitted: **ASCII** (7-bit, English-only), **Unicode** (a universal character set), **UTF-8** (the dominant *encoding* of Unicode — variable-width bytes, backward-compatible with ASCII, what virtually every HTTP API — including this one — uses for JSON bodies). **Serialization** is turning an in-memory object into bytes for transmission (`JSON.stringify`, or .NET's `System.Text.Json`, chapter 3 of `dotnet-mastery`); **deserialization** is the reverse. **Endianness** (byte order — big-endian vs little-endian) matters for raw binary protocols; virtually never something you'll touch directly working with JSON-over-HTTP, but it's why network protocols specify "network byte order" (big-endian) explicitly when they define binary formats.

## 1.3 Performance vocabulary — precise terms, not interchangeable

- **Bandwidth** — the *maximum* data-transfer capacity of a link (how wide the pipe is).
- **Throughput** — the *actual* data transferred in practice (often less than bandwidth, due to overhead/congestion).
- **Latency** — the time for one bit/packet to travel from sender to receiver — distance and physics matter here, not just pipe width. (A satellite link can have huge bandwidth and still terrible latency.)
- **Jitter** — variation in latency over time — a killer for real-time protocols (voice, video, gaming) even when average latency is fine.
- **Packet loss** — packets that never arrive (dropped by a congested router, a bad link, etc.) — this is exactly what TCP (chapter 5) exists to detect and recover from transparently.
- **Congestion** — too much traffic for a link/router to handle, causing queueing delay and/or loss.

You've already seen these concerns show up concretely, without the vocabulary attached: chapter 12 of `../dotnet-mastery/` (Performance & Scaling) discussed `SetCommandTimeout` on every CQRS handler and the 10–30 second timeouts on each health check (`HRNS.WebApi/Program.cs:367-385`) — those numbers are explicit acknowledgments that **latency is not zero and not guaranteed**, which is the single most important fact in distributed-systems engineering (chapter 14 returns to this in depth).

## 1.4 The OSI model — seven layers, and why you should actually learn them

```
7  Application    ← HTTP, DNS, SSH, SMTP    "what does this data MEAN"
6  Presentation   ← encryption, compression  "how is this data FORMATTED"
5  Session        ← session establishment    "is this an ongoing conversation"
4  Transport      ← TCP, UDP                 "reliable delivery, to which PROCESS"
3  Network        ← IP, ICMP, routing        "how do I reach that HOST"
2  Data Link      ← Ethernet, Wi-Fi, MAC     "how do I reach the next HOP"
1  Physical       ← cables, radio, fiber     "how do bits actually MOVE"
```

You don't need to recite all seven from memory in an interview-trivia sense. What you *do* need is the underlying discipline: **networking problems are divided by layer, and diagnosing a problem means figuring out which layer it's actually in.** In practice, most working engineers collapse OSI's top three layers (Application/Presentation/Session) into one, since in modern protocol design (HTTP, TLS) those distinctions blur — which is exactly what the simpler, more practical **TCP/IP model** does:

```
Application     ← HTTP, DNS, SSH, TLS all live here
Transport       ← TCP, UDP
Internet        ← IP, ICMP, routing
Link            ← Ethernet, Wi-Fi
```

This guide is structured around the TCP/IP model's four layers (plus a dedicated security chapter, since TLS deserves its own treatment even though TCP/IP folds it into "Application").

## 1.5 The debugging mental model — this is the actual payoff

The entire point of learning layers is this: **"it doesn't work" is not a diagnosis.** A layered model turns a vague symptom into a systematic elimination process.

> "The frontend can't reach the API."

```
Is DNS resolving the hostname at all?            ← chapter 7
Is a TCP connection even being established?        ← chapter 5/6
Is TLS negotiating successfully?                     ← chapter 9
Is the HTTP request reaching the app (right route)?   ← chapter 8
Is CORS blocking it at the browser?                     ← chapter 10
Is the app-level response actually an error (401/500)?    ← ../dotnet-mastery/ ch.8/10
```

Each question isolates one layer and rules it in or out — and critically, you can test each layer *independently* of the ones above it (chapter 17's diagnostic toolkit is built entirely around this: `ping` tests network-layer reachability without touching TCP at all; `curl -v` shows you TCP connect, TLS handshake, and HTTP response as separate, inspectable phases of the same request). This is the single most transferable skill in this entire guide — more valuable long-term than memorizing any individual protocol's field names.

## Checkpoint

1. Without looking back, write out the four TCP/IP layers and name one protocol/concept that lives at each.
2. `HRNS.WebApi/Program.cs` sets `HealthCheckOptions` with per-check timeouts ranging from 5 to 30 seconds (chapter 10, `../dotnet-mastery/`). In the vocabulary of §1.3, is a health check that exceeds its timeout necessarily evidence of packet loss? What else could it be evidence of?
3. Pick any bug report you've seen (real or hypothetical) phrased as "X doesn't work" and rewrite it as a layer-by-layer elimination checklist, the way §1.5 did for "the frontend can't reach the API."
