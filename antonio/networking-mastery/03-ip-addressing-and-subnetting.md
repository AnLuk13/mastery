# 3. IP Addressing & Subnetting

Goal: read and reason about IP addresses and CIDR notation fluently — one of the highest-leverage topics in this whole guide, since it underlies routing (ch. 4), NAT (ch. 4), Docker networking (ch. 15), and cloud VPC configuration (ch. 15) alike.

## 3.1 IPv4 addresses

An IPv4 address is a 32-bit number, almost always written in **dotted-decimal notation**: four 8-bit numbers (0–255 each), separated by dots — `192.168.1.10`. Every IP address splits conceptually into two parts: a **network portion** (which network is this host on) and a **host portion** (which specific host, within that network) — but *where* that split happens isn't fixed; it's defined by the **subnet mask**.

```
192.168.1.10
└──┬──┘ └┬┘
network  host    (if the mask says the first 24 bits are "network")
```

## 3.2 Subnet masks & CIDR — the part worth truly internalizing

A subnet mask is another 32-bit number, made of a run of 1-bits (network portion) followed by a run of 0-bits (host portion) — e.g. `255.255.255.0` is 24 ones followed by 8 zeros. **CIDR notation** (Classless Inter-Domain Routing) writes the same thing far more concisely: `/24` means "the first 24 bits are the network portion."

```
192.168.1.10/24
```
means: the network is `192.168.1.0` through `192.168.1.255` (256 addresses, 24 network bits + 8 host bits = 2^8 = 256), and this specific host is `.10` within it.

**Worked comparison — this is the exercise that makes CIDR click:**

```
192.168.1.0/24   → 256 addresses total (192.168.1.0 – 192.168.1.255)
                     usable hosts: 254 (network address + broadcast address are reserved)

192.168.1.0/25   → 128 addresses (192.168.1.0 – 192.168.1.127)
                     a SECOND /25 network, 192.168.1.128/25, covers .128–.255

192.168.1.0/16   → 65,536 addresses (192.168.0.0 – 192.168.255.255)  ← note: /16 means the
                     FIRST TWO octets are fixed, not the first one
```

**The rule to internalize**: *fewer* host bits (a *larger* `/number`, e.g. `/28`) means a *smaller* network, with fewer usable addresses. `/24` → 254 usable hosts. `/25` → 126. `/30` → 2 (just enough for a point-to-point link, e.g. between two routers). This inverse relationship trips up almost everyone the first time — sit with it until "a bigger CIDR number means a smaller network" feels obviously true rather than backwards.

Every subnet reserves exactly two addresses it can't hand out to a host: the **network address** itself (all host bits zero — identifies the subnet as a whole, e.g. `192.168.1.0`) and the **broadcast address** (all host bits one, e.g. `192.168.1.255` for a `/24` — a packet sent here reaches every host on that subnet).

## 3.3 Private vs. public addresses

Three IPv4 ranges are reserved by standard (RFC 1918) as **private** — not globally routable on the public internet, free for anyone to reuse inside their own network:

```
10.0.0.0/8         (10.0.0.0 – 10.255.255.255)      — huge range, common in large corporate/cloud networks
172.16.0.0/12      (172.16.0.0 – 172.31.255.255)     — Docker's default bridge network lives here
192.168.0.0/16     (192.168.0.0 – 192.168.255.255)    — the classic home-router range
```

Every other IPv4 address is (in principle) **public** — globally unique, routable across the internet. This is *why* NAT exists (chapter 4): your laptop's private `192.168.1.x` address means nothing to the rest of the internet, so your router translates it to its one public address before anything leaves your home network.

**Loopback / localhost**: `127.0.0.1` (and the entire `127.0.0.0/8` range, though `127.0.0.1` is the one you'll ever actually use) always refers to *this same machine* — traffic to it never touches a physical network interface at all. `localhost` is simply the DNS name that resolves to `127.0.0.1` (chapter 7) by convention/OS configuration. Every `dotnet run` you've done locally, hitting `https://localhost:5001` or similar, has been exercising exactly this loopback path — no NIC, no switch, no ARP, none of chapter 2 involved at all, which is part of why local development can mask real-network problems (a service that only ever gets tested via `localhost` has never actually proven it works across a real network boundary).

## 3.4 IPv6 — the essentials

IPv4's 32-bit address space (≈4.3 billion addresses) has been effectively exhausted for years; IPv6 is the (slow, ongoing) replacement, with a vastly larger 128-bit address space. Notation is hexadecimal, colon-separated, in eight 16-bit groups:

```
2001:0db8:0000:0000:0000:ff00:0042:8329
```
almost always abbreviated by (a) stripping leading zeros in each group and (b) collapsing exactly one run of consecutive all-zero groups with `::`:
```
2001:db8::ff00:42:8329
```
`::1` is IPv6's loopback address — the direct equivalent of `127.0.0.1`. A **link-local** address (prefix `fe80::/10`) is auto-assigned to every interface and only valid on its immediate local network segment (never routed anywhere) — every IPv6-capable interface has one whether or not it has a routable global address too. A **global unicast address** is IPv6's equivalent of a public IPv4 address — routable across the internet.

You don't need to become an IPv6 specialist to be effective here — `HRNS.Platform.Server`/`.UI`'s actual deployment and local-dev configuration (connection strings, `AllowedOrigins`, the `HostDomain`/`HostPort` settings from `../dotnet-mastery/` chapter 2 §2.5) is entirely IPv4/hostname-based, consistent with the vast majority of business web apps today — but you should be able to recognize IPv6 addresses on sight, understand `::1` the instant you see it in a log or error message, and know that "the network stack tried IPv6 first and it failed" is a real, if uncommon, class of connectivity bug (this is literally what "happy eyeballs" — the dual-stack connection algorithm modern OSes use — exists to work around).

## 3.5 Grounded example: where a hardcoded address/port shows up in this platform

`GeoServerHealthCheck.cs` (`HRNS.Platform.Server/HRNS.WebApi/HealthChecks/GeoServerHealthCheck.cs:38-39`) has a real, concrete fallback worth reading with this chapter's vocabulary:

```csharp
var geoApiBase = _configuration.GetSection("GeoApi:Address").Value
    ?? "https://localhost:5005/geo-api";
```

Two things this teaches at a glance: (1) `localhost` here means "the geolocation microservice, when run locally during development, listens on the *same machine* as the platform server" — chapter 14 covers why sibling microservices in this workspace (`HRNS.Gelocation.Server`) are configured this way for local dev, and how that changes in a real deployment (they'd get real hostnames/addresses, resolved via DNS — chapter 7 — not `localhost`); (2) `:5005` is a **port** — a concept this chapter deliberately hasn't covered yet, because it belongs to the *transport* layer, not the network layer — chapter 6 picks it up properly, once TCP/UDP (chapter 5) has given you the vocabulary to understand what a port actually is.

## Checkpoint

1. Given `10.20.30.0/22`, work out: how many total addresses does this subnet contain, what's the network address, and what's the broadcast address? (Hint: `/22` leaves 10 host bits — `2^10 = 1024` addresses.)
2. Is `172.20.5.5` a private or public address? Which of the three RFC 1918 ranges (if any) does it fall in?
3. `127.0.0.1` and a service's real LAN/public IP address behave identically from the *application's* point of view (both are just "an IP address to connect to") but very differently from the *network's* point of view. Name one concrete way local-only testing (against `localhost`) could pass while the same code fails once deployed to talk to a real remote address.
