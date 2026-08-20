# Networking Mastery — from packets to production, grounded in HRNS.Platform.Server/UI

You asked for a real recap-from-basics networking curriculum — not because you're starting from zero (you're not), but because most full-stack developers have a working, intuition-shaped understanding of the top of the stack (HTTP, DNS-ish, "it just works") and genuinely fuzzy or half-forgotten understanding of everything underneath it. This guide fills that in, bottom to top, then ties every layer back to how `HRNS.Platform.Server`/`HRNS.Platform.UI` actually put it to use.

This curriculum was scoped from a roadmap you got from another source; it's a good, thorough roadmap, and this guide follows its structure closely while reorganizing 43 subtopics into a coherent chapter sequence, adding two chapters that roadmap only mentioned in passing but which are directly relevant to how this platform works (**browser networking mechanics**, and **real service-to-service networking within this platform**), and folding its 🟡 "learn later" tier into one honest appendix instead of pretending every advanced BGP/VLAN topic deserves equal weight for a full-stack engineer.

## How this is organized

Same contract as [`../dotnet-mastery/`](../dotnet-mastery/00-INDEX.md): every chapter is theory → minimal example → **grounded in this workspace's real code** (file paths, real config, real excerpts) → checkpoint. Where a topic is genuinely just general knowledge with nothing platform-specific to show (physical-layer cabling, BGP), the chapter says so plainly rather than manufacturing a tie-in that isn't there.

## The one picture to hold in your head the whole way through

```
APPLICATION    HTTP · DNS · SSH · SMTP · WebSocket
SECURITY       TLS · certificates · encryption
TRANSPORT      TCP · UDP · QUIC
NETWORK        IP · ICMP · routing
LINK           Ethernet · Wi-Fi · ARP · MAC
PHYSICAL       cables · radio · fiber
```

Every chapter in Part 1 builds one layer of this stack, bottom to top. By the end of Part 1 you should be able to answer, layer by layer, "what happens when my browser loads `https://app.hrns.example.com`" — that single question is the thread tying the whole guide together, and chapter 17 makes you actually trace it.

## Suggested path

**Part 1 — The stack, bottom to top**
1. [Fundamentals & Mental Models](01-fundamentals-and-models.md) — what a network even is, client/server, packets/frames, the OSI & TCP/IP models, performance vocabulary
2. [Link Layer & LANs](02-link-layer-and-lan.md) — Ethernet, MAC addresses, switches, ARP
3. [IP Addressing & Subnetting](03-ip-addressing-and-subnetting.md) — IPv4, IPv6, CIDR, private/public, localhost
4. [Routing, NAT & ICMP](04-routing-nat-icmp.md) — how packets cross networks, why your laptop's IP isn't reachable from the internet
5. [TCP & UDP](05-tcp-udp-transport.md) — the transport layer, in depth — this is a 🔴🔴🔴 chapter
6. [Ports & Sockets](06-ports-and-sockets.md) — what's actually listening, and how Kestrel/Postgres/ClamAV each use this
7. [DNS](07-dns.md) — from a domain name to an IP address, record types, and this platform's own host-resolution logic
8. [HTTP](08-http.md) — methods, status codes, HTTP/1.1 → HTTP/3, and HRNS's unconventional routing conventions

**Part 2 — Security & the browser**
9. [TLS, HTTPS & Cryptography](09-tls-https-and-cryptography.md) — the security layer, and how it connects to JWT/password hashing you already know from the .NET guide
10. [Browser Networking](10-browser-networking.md) — fetch/axios, same-origin policy, CORS preflight in full mechanical detail, cookies — grounded directly in `HRNS.Platform.UI`'s `BaseApiService`
11. [WebSockets, SSE & Real-Time](11-websockets-sse-realtime.md) — persistent connections, grounded in HRNS's SignalR hubs

**Part 3 — Infrastructure**
12. [Proxies, Load Balancing & CDNs](12-proxies-load-balancing-cdn.md)
13. [Firewalls & Network Security](13-firewalls-and-network-security.md) — threats and defenses
14. [Service-to-Service Networking & Reliability](14-service-to-service-and-reliability.md) — how `HRNS.Platform.Server` actually calls its sibling microservices (File Server, `HRNS.Gelocation.Server`), timeouts, retries, circuit breakers
15. [Containers & Cloud Networking](15-containers-and-cloud-networking.md) — Docker networking pitfalls, cloud VPC/subnet concepts

**Part 4 — Remote access & practice**
16. [SSH, Tunneling & VPNs](16-ssh-tunneling-and-vpns.md) — the original motivating topic, now with the full foundation underneath it
17. [Diagnostics Toolkit](17-diagnostics-toolkit.md) — `ping`, `traceroute`, `dig`, `curl`, `ss`, `tcpdump`, `openssl` — and the cumulative "trace the whole request" exercise
18. [Further Topics — Appendix](18-further-topics-appendix.md) — BGP, VLANs, wireless internals, email protocol details, Kubernetes CNI internals: the roadmap's 🟡 tier, covered honestly as pointers rather than padded into full chapters

[Glossary](GLOSSARY.md)

## Priority tiers (carried over from the source roadmap, mapped to chapters)

| Tier | Meaning | Chapters |
|---|---|---|
| 🔴 Master | You should be able to explain and debug with these without hesitation | 1, 3, 5, 6, 7, 8, 9, 12, 13, 16, 17 |
| 🟠 Comfortable | Understand conceptually, know when/why to reach for it | 2, 4, 10, 11, 14, 15 |
| 🟡 Later | Know it exists, know roughly what it's for, go deep only if your work demands it | 18 |

## Ground rules

- Read Part 1 in order — it's cumulative on purpose (you can't reason about TCP without ports, can't reason about HTTP without TCP, etc).
- Whenever a chapter says "in this platform," go open the real file — same discipline as `dotnet-mastery`.
- Where a chapter references a concept already covered in depth in `../dotnet-mastery/` (JWT internals, EF Core connection pooling, health checks), it links there rather than re-explaining — read both guides as one connected reference, not two silos.
