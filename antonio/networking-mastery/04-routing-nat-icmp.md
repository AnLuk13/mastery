# 4. Routing, NAT & ICMP

Goal: how a packet actually gets from your machine to a server that isn't on your local network — routing — plus the mechanism (NAT) that lets your private-IP laptop talk to the public internet at all, and the diagnostic protocol (ICMP) you already use constantly without necessarily knowing its name.

## 4.1 Routing — how a packet crosses networks

Chapter 2 covered how a frame moves *within* one LAN (switches, MAC addresses). Routing is the layer-3 equivalent for moving a packet *between* different networks — the normal case for literally any internet-facing traffic, since your device's network is essentially never the same network the server lives on.

A **router** is a device connected to two or more networks that forwards packets between them, using a **routing table** — a list of "to reach network X, send via next-hop Y." Your own device has a simplified version of this too: a **default gateway** (usually your home router's LAN address, e.g. `192.168.1.1`) is where *anything not on your local subnet* gets sent, on the assumption the gateway knows better (or at least knows the next hop toward knowing better).

```
Your computer  (192.168.1.10)
      │
      ▼
Default gateway  (192.168.1.1)   ← your router
      │
      ▼
Your ISP's router
      │
      ▼
   ...more routers...
      │
      ▼
Destination server
```

Each router along the way makes an independent, local decision: "given this packet's destination IP, which of my next hops gets it closer?" — nobody has a full end-to-end map; the internet works via many routers each knowing a little, cooperatively. Each hop is exactly that — a **hop** — and every IP packet carries a **TTL** (Time To Live) field, decremented by one at every hop; if it hits zero, the packet is dropped and the router that dropped it sends back an ICMP message (§4.3) — this is precisely how `traceroute` (chapter 17) maps out the path to a destination, one hop at a time, by deliberately sending packets with TTL=1, then 2, then 3...

**Static routes** are manually configured; **dynamic routing protocols** (BGP, OSPF — chapter 18) let routers automatically discover and share reachability information — this is genuinely how the internet's backbone stays up to date as networks change, but it's not something you'll configure as an application developer; your involvement with routing in practice is almost entirely at the "default gateway" / cloud "route table" level (chapter 15).

## 4.2 NAT — why your laptop's private IP works on the public internet at all

Chapter 3 established that private IP ranges (`192.168.x.x`, `10.x.x.x`, `172.16-31.x.x`) aren't globally routable. **NAT** (Network Address Translation) is the mechanism that lets a device with a private IP still reach the public internet: your router rewrites the *source* address of every outgoing packet to its own public IP (and remembers the mapping, so return traffic gets routed back to the right internal device).

```
Laptop                    Router                       Internet
192.168.1.10   ──────►   translates source to    ──────►   sees only the
                          router's public IP                  router's public IP
```

Precise vocabulary: **SNAT** (Source NAT) rewrites the source address — what your home router does for outbound traffic; **DNAT** (Destination NAT) rewrites the destination address — used to route inbound traffic to an internal service (e.g. "requests to my public IP on port 443 get forwarded to this internal server" — a reverse proxy, chapter 12, does something conceptually similar at the application layer). **PAT** (Port Address Translation, sometimes just called "NAT overload") is the common refinement that lets *many* internal devices share *one* public IP simultaneously, by also rewriting the *source port* of each connection so return traffic can be disambiguated — this is what your home router actually does for every device on your Wi-Fi at once.

**A genuinely important, easy-to-miss consequence of NAT (and of any reverse proxy sitting in front of an app, chapter 12)**: by the time a request reaches your application, the "source IP" visible to the OS/framework is the *proxy's* or *NAT gateway's* address, not the real original client's. This is exactly why `X-Forwarded-For`/`X-Forwarded-Host`/`X-Forwarded-Proto` headers exist — a proxy that performs this translation is expected to record the *original* client info in these headers before forwarding the request onward, so the application can recover it. `HRNS.Platform.Server` explicitly trusts these headers:

```csharp
app.UseForwardedHeaders(new ForwardedHeadersOptions
{
    ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto | ForwardedHeaders.XForwardedHost
});
```
(`HRNS.WebApi/Program.cs:135-140` — already introduced in `../dotnet-mastery/` chapter 3 §3.1 as a middleware-ordering example; here's *why* it exists.) Without this, every `UserLoginHistoryEntity.ClientIP`/`ExternalIP` value recorded during login (`../dotnet-mastery/` chapter 8 §8.5) would record the reverse proxy's own IP for every single user — useless for the audit trail it's meant to produce. `BaseController.ExtractHostFromRequest`'s Origin → Referer → X-Forwarded-Host → Host fallback chain (`../dotnet-mastery/` chapter 3 §3.2) is solving the exact same class of problem for the *host* rather than the client IP.

## 4.3 ICMP — the protocol behind `ping` and `traceroute`

**ICMP** (Internet Control Message Protocol) isn't for carrying application data at all — it's the network layer's own diagnostic/control channel, used by routers and hosts to report problems and answer connectivity probes.

- **Echo request / echo reply** — the mechanism behind `ping`: send an ICMP echo request, wait for an echo reply, measure the round-trip time. A successful ping tells you network-layer reachability is fine — but nothing about whether any *application* is listening on any port at that address (a very common source of confusion: "I can ping the server but the website's down" is completely consistent, because a working ping only proves layers 1–3 are fine).
- **TTL exceeded** — sent back by a router when it drops a packet whose TTL hit zero (§4.1) — this is the entire mechanism `traceroute` exploits to enumerate the path hop-by-hop.
- **Destination unreachable** — sent when a router/host can't deliver a packet at all (network unreachable, host unreachable, port unreachable, etc.) — a substantively different signal from "no reply" (which just means silence/timeout, possibly due to a firewall dropping packets rather than rejecting them — chapter 13 covers this distinction, since it's a deliberate security choice).

```bash
ping example.com        # is the network layer working, roughly how far away is it (latency)?
traceroute example.com  # what path (which routers) does traffic take to get there?
```

## Checkpoint

1. If a `ping` to a server succeeds but an HTTP request to that same server times out, use the §1.5 layered-elimination approach: what layers has the successful ping already ruled out, and what layers are still unverified?
2. Explain, in your own words, why `X-Forwarded-For` is fundamentally a *trust* problem — what would go wrong if `HRNS.Platform.Server` blindly trusted this header from *any* client, rather than only from a reverse proxy it controls sitting directly in front of it?
3. Your home router is doing NAT/PAT for every device on your network simultaneously. If two different laptops on your home Wi-Fi both open a connection to the same website at the same time, what has to differ between the two connections (from the outside world's point of view) for return traffic to reach the correct laptop? (Hint: revisit §4.2's PAT definition.)
