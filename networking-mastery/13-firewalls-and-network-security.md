# 13. Firewalls & Network Security

Goal: firewall mechanics, the common network-level attack categories, and — in the spirit of the honest security reviews in `../dotnet-mastery/` chapters 8 and 12 — a clear-eyed look at which of these threats this platform actually defends against today, and which are open questions worth having an opinion on.

## 13.1 Firewalls — filtering traffic by rule

A **firewall** decides whether to allow or block traffic based on rules — typically source/destination IP, port, and protocol. Two fundamentally different designs:

- **Stateless (packet-filtering) firewall** — evaluates each packet independently against its rule list, with no memory of prior packets. Simple, fast, but can't distinguish "a reply to a connection I initiated" from "an unsolicited new connection" without you writing rules for both directions explicitly.
- **Stateful firewall** — tracks active connections (chapter 5 §5.2's connection concept) and automatically allows return traffic belonging to a connection that was legitimately initiated from the trusted side, without needing an explicit rule for the reply — the overwhelmingly more common design today, since it maps much more naturally onto "allow outbound, and allow only the replies to what we ourselves initiated" as a default posture.

**Inbound vs. outbound rules** — a firewall can restrict traffic in either direction independently; a very common, sound default posture is "allow all outbound, restrict inbound to only what's explicitly needed" (chapter 15's cloud security groups follow exactly this default). `ufw`, `iptables`, `nftables` are the standard Linux tools for configuring this at the OS level (chapter 17 touches these); cloud environments typically express the same concept as **security groups** (instance-level, usually stateful) and **network ACLs** (subnet-level, often stateless) — chapter 15 covers both properly.

```
Internet
    │
    ▼
Firewall  (allow 443 inbound, deny everything else inbound; allow all outbound)
    │
    ▼
Server
```

**A crucial distinction for diagnosing connectivity, tying directly back to chapter 4 §4.3's ICMP coverage**: a firewall can either *reject* a disallowed packet (send back an ICMP "destination unreachable" or a TCP `RST`, chapter 5 §5.2 — the client finds out immediately, if unsurprisingly, that it's blocked) or *drop* it silently (no response at all — the client just times out, with no way to distinguish "blocked by a firewall" from "the network is broken" or "nothing's listening"). Silent dropping is a deliberate security choice in many environments — it denies an attacker doing reconnaissance (port scanning, below) the useful information a `RST` would leak ("this port is closed" vs. "this port doesn't even exist on this network"), at the cost of making legitimate debugging slower.

## 13.2 Common threats — and how this platform actually stands against each

**Eavesdropping / packet sniffing** — passively reading traffic on a network you have access to (a shared Wi-Fi network, a compromised router, a malicious actor with access to network infrastructure). **Defense**: TLS (chapter 9) — this is precisely the "confidentiality" guarantee from §9.1, and every HTTPS connection this platform makes or receives is defended against this by construction, *assuming* TLS is actually enforced everywhere (worth actually verifying rather than assuming, on any real deployment).

**Man-in-the-middle (MITM)** — an attacker actively intercepts and potentially modifies traffic between two parties who believe they're talking directly to each other. **Defense**: TLS's *authentication* guarantee (§9.1, §9.4) — a valid certificate chain proves you're actually talking to the server you intended, which is exactly why certificate/hostname validation failures should never be silently ignored or bypassed in application code (a classic, real vulnerability pattern: code that disables certificate validation "just to get past an error" in development, then ships that way).

**Spoofing** (IP spoofing, DNS spoofing, ARP spoofing) — forging the apparent source of traffic. **IP spoofing**: forging a packet's claimed source IP — TCP's handshake (chapter 5 §5.2) makes this hard to exploit for anything requiring a full bidirectional connection, since the attacker can't see the SYN-ACK sent to the address they spoofed, but it's still a real vector for certain reflection/amplification attacks. **DNS spoofing**: providing a forged DNS response, redirecting a victim to a malicious server for what looks like a legitimate domain (`DNSSEC` is the — sparsely deployed — cryptographic defense; TLS certificate validation, §9.4, is what actually saves you in practice even if DNS itself was spoofed, since the attacker's server wouldn't have a valid certificate for your intended domain). **ARP spoofing**: on a local network (chapter 2 §2.3), falsely claiming to own another host's IP address, redirecting local traffic through the attacker's machine — a LAN-local attack, not something remote over the internet.

**Replay attacks** — capturing a legitimate request/message and resending it later to fraudulently repeat its effect. **Defense**: this is precisely why JWTs carry an expiry (`../dotnet-mastery/` chapter 8 §8.3's `Expires`/`ValidateLifetime`) — a captured, expired token becomes useless; combined with TLS (preventing the capture in the first place under normal circumstances), and idempotency keys (chapter 8 §8.2, chapter 14) for the specific case of a legitimate client accidentally retrying its own request.

**Session hijacking** — stealing or guessing an already-authenticated session's identifier to impersonate a logged-in user. **Defense**: this is exactly why `HttpOnly` cookies (chapter 10 §10.5) matter when cookies are in play (preventing JavaScript-based theft), and part of why this platform's bearer-token-in-`Authorization`-header approach (chapter 10 §10.4) needs XSS defense specifically (since the token *is* readable by the page's own JavaScript, by design, for the API calls to work) — a real trade-off between the CSRF exposure cookies carry and the XSS exposure token-in-storage carries, already flagged as worth thinking through in chapter 10's checkpoint.

**Port scanning** — systematically probing which ports are open on a target, typically as reconnaissance before a more targeted attack. **Defense**: minimizing the actual attack surface (only exposing the ports genuinely needed — chapter 6 §6.1's well-known ports plus whatever this app specifically needs), and the silent-drop firewall posture from §13.1.

**Denial of Service (DoS) / Distributed DoS (DDoS)** — overwhelming a target with traffic/requests until it can't serve legitimate users. **Honest gap worth naming, in the same spirit as `../dotnet-mastery/` chapter 12 §12.5's caching-gap discussion**: `HRNS.Platform.Server` has account-lockout throttling *specific to login attempts* (`../dotnet-mastery/` chapter 8 §8.5) and reCAPTCHA integration (`ReCaptcha` config section, `RecaptchaHealthCheck`, `../dotnet-mastery/` chapter 0's tech table) — a real, if narrow, defense against *credential-stuffing/brute-force* traffic specifically — but there's no general-purpose, application-wide rate limiting visible in `Program.cs`'s middleware pipeline (`../dotnet-mastery/` chapter 3 §3.1) beyond that. In most real deployments this is an *intentional* division of labor rather than an oversight: broad DDoS mitigation is usually handled at the infrastructure layer (a CDN/reverse proxy/cloud provider's DDoS protection, chapter 12 and 15) rather than in application code, since the application layer is often the *last* place you'd want to be absorbing an actual volumetric attack — but it's worth being able to name precisely where that responsibility sits in a given deployment rather than assuming either "the app handles it" or "someone else handles it" without checking.

**Brute force / credential theft** — already covered in depth (`../dotnet-mastery/` chapter 8 §8.5's account lockout, and the honest password-hashing critique there).

## Checkpoint

1. Using §13.1's stateful-firewall description, explain why a legitimate response to a database query `HRNS.Platform.Server` initiated doesn't need its own explicit inbound firewall rule, even though it's technically inbound traffic.
2. Pick one threat from §13.2 this platform defends against only partially or not at all (per the honest gaps named), and sketch — in one paragraph, no code needed — what a defense-in-depth addition would look like and at which layer (application, reverse proxy, cloud infrastructure) it would most naturally live.
3. A firewall configured to silently drop disallowed packets makes port scanning slower/less informative for an attacker, at the cost of making legitimate connectivity debugging harder (§13.1). Given chapter 17's diagnostic toolkit is built around expecting *some* response (even a `RST` or ICMP unreachable) to reason about failures, what would you actually observe running `curl`/`ping` against a silently-dropping firewall, and how would that differ from the "definitely broken, nothing listening" case?
