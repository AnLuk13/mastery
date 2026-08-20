# 7. DNS

Goal: DNS is the internet's phone book, and it's absolutely essential for a web developer — from here on, "what happens when you load a URL" (the exercise chapter 17 makes you trace end-to-end) starts with this chapter.

## 7.1 Why DNS exists

Computers address each other with IP addresses (chapter 3). Humans want to type `example.com`, not `93.184.216.34`. **DNS** (Domain Name System) is the distributed, hierarchical system that translates the former into the latter — and it does far more than that once you look closely (mail routing, service discovery, ownership verification), all through the same core mechanism: a **record** stored under a **domain name**, of a specific **type**.

## 7.2 The resolution hierarchy

DNS is deliberately distributed — no single server holds the entire internet's mappings. Resolving `www.example.com` walks a hierarchy:

```
Your device asks a recursive resolver (often your ISP's, or a public one like 1.1.1.1/8.8.8.8)
        │
        ▼
Recursive resolver asks a ROOT server: "who handles .com?"
        │
        ▼
Root server: "ask this .com TLD server"
        │
        ▼
Recursive resolver asks the TLD server: "who's authoritative for example.com?"
        │
        ▼
TLD server: "ask this authoritative nameserver"
        │
        ▼
Recursive resolver asks the AUTHORITATIVE nameserver: "what's the A record for www.example.com?"
        │
        ▼
Authoritative server: "93.184.216.34"
        │
        ▼
Recursive resolver returns the answer to your device (and CACHES it)
```

- **Recursive resolver** — does the actual multi-step legwork on your behalf; what your device is configured to ask (often set via DHCP, or manually — `resolv.conf` on Linux is where this lives).
- **Root servers** — the top of the hierarchy; there are only a handful of root server *addresses* globally (though each is served from many physical/anycast locations) — they don't know individual domains, only which TLD servers to ask next.
- **TLD** (Top-Level Domain) servers — handle one suffix (`.com`, `.org`, `.io`, ...) — know which authoritative server to ask for any specific domain under that TLD.
- **Authoritative nameserver** — the actual source of truth for one specific domain's records — usually run by whoever manages that domain's DNS (a registrar, or a dedicated DNS provider).

## 7.3 Record types you'll actually encounter

| Record | Maps a name to... | Example use |
|---|---|---|
| `A` | an IPv4 address | `example.com → 93.184.216.34` |
| `AAAA` | an IPv6 address | `example.com → 2606:2800:220:1::` |
| `CNAME` | another domain name (an alias) | `www.example.com → example.com` |
| `MX` | the mail server(s) handling email for this domain, with priority | routes inbound email — relevant if this platform sends via a custom domain (§7.5) |
| `TXT` | arbitrary text | domain ownership verification, and critically, **SPF**/**DMARC** policy strings for email (§7.5) |
| `NS` | which nameservers are authoritative for this domain | delegation — "ask *these* servers about this domain" |
| `SOA` | administrative info about a zone (primary nameserver, refresh timers, etc.) | mostly relevant to whoever administers the DNS zone, not to application code |
| `PTR` | the reverse mapping — IP address → domain name | **reverse DNS** (§7.4) |
| `SRV` | a specific host+port for a service (with priority/weight) | service discovery — more common internally (e.g. some Kubernetes setups) than on the public internet |

Every record carries a **TTL** (Time To Live, in seconds) — how long a resolver is allowed to cache it before asking again. This is a real, deliberate trade-off every domain owner makes: a low TTL means changes propagate fast but every resolver has to re-query more often (more load, slightly higher average latency); a high TTL means better caching/performance but slower propagation if you ever need to change a record (e.g. during an emergency failover). "**DNS propagation**" delay — the reason a DNS change doesn't take effect everywhere instantly — is entirely a consequence of caches around the world honoring the *old* record's TTL until it expires.

## 7.4 Reverse DNS & `/etc/hosts`

**Reverse DNS** (via `PTR` records) answers the opposite question — given an IP address, what domain name is associated with it. Used for things like mail server reputation checks (many mail servers reject email from IPs with no valid reverse DNS, as a spam signal) and for making log files human-readable.

**`/etc/hosts`** (or Windows' `C:\Windows\System32\drivers\etc\hosts`) is a local, manual override that's checked *before* any DNS query is made at all — a simple text file mapping hostnames to IPs directly on your own machine. This is exactly the mechanism that makes `localhost → 127.0.0.1` work (chapter 3 §3.3) — it's an `/etc/hosts` entry (or an OS-level equivalent), not a real DNS lookup at all. Developers commonly add temporary entries here to point a real domain name at a local or staging IP for testing, bypassing DNS entirely for that one hostname.

## 7.5 Where DNS shows up in this platform, beyond "resolving a hostname"

**Multi-tenant host resolution — DNS gets you to the server, then the app decides what you meant.** DNS resolves a domain to an IP and delivers the request to *a* server — it says nothing about *which tenant/site* that request is for once it arrives. `HRNS.Platform.Server` is explicitly multi-tenant at the HTTP layer (`ProviderSiteEntity`, looked up by host, `../dotnet-mastery/` chapter 5 §5.5's login walkthrough) — this is exactly the job `BaseController.ExtractHostFromRequest`'s fallback chain does (`../dotnet-mastery/` chapter 3 §3.2): DNS's job ends the moment the TCP connection reaches Kestrel; from there, the **`Host`** header (and `Origin`/`Referer`/`X-Forwarded-Host` as fallbacks) is what the *application* uses to figure out which provider site the request was actually meant for. Worth being precise about the boundary: DNS is a network-layer-adjacent concern that happens *before* any packet reaches your server; the `Host` header is an *application-layer* (HTTP) concept that happens to carry similar-looking information for a completely different purpose.

**Email deliverability — `MX`/`SPF`/`DKIM`/`DMARC`, relevant because this platform sends real email.** `HRNS.Application.csproj` includes both `SendGrid` and `MailKit`/`MimeKit` (`../dotnet-mastery/` chapter 0's tech-stack table), and `appsettings.json` has a `Mail` config section — meaning whichever domain this platform sends email *from* needs correctly configured DNS records for that mail to reliably land in inboxes rather than spam: an `MX` record (where *inbound* mail for that domain goes — less relevant if this platform only sends, never receives), an `SPF` `TXT` record (which servers are authorized to send email claiming to be from this domain), `DKIM` (a cryptographic signature, verified via a `TXT` record published under a special subdomain, proving the email wasn't tampered with in transit), and `DMARC` (a policy telling receiving mail servers what to do if SPF/DKIM checks fail). None of this is application code — it's DNS zone configuration living entirely outside this repo — but it's worth knowing *why* it exists: a perfectly correct `EmailService.SendEmailAsync(...)` call (`../dotnet-mastery/` chapter 8 §8.5) can still land in spam if the sending domain's DNS isn't configured to youch for it.

**Every outbound integration this platform has resolves a hostname first.** `AIServer:Address` (Ollama), the ClamAV/GeoApi addresses (chapter 6 §6.3, `localhost` in dev — a real hostname in a real deployment), SendGrid's API endpoint, Google's reCAPTCHA/Firebase endpoints — every single one of these begins with a DNS lookup before any TCP connection (chapter 5) is even attempted. This is why "is DNS resolving?" is one of the very first questions in the §1.5 layered-debugging checklist — it's genuinely the *first* thing that has to succeed, before anything else in the chain can even be attempted.

## Checkpoint

1. Walk through §7.2's resolution hierarchy from memory for a domain of your choosing — which server actually holds the authoritative answer, and which ones were just pointing you toward it?
2. If `HRNS.Platform.Server` needs to send transactional emails from `notifications@hrns-platform.example`, which DNS record types (§7.5) would you expect to need configuring on that domain, and what does each one actually prove to a receiving mail server?
3. Explain the difference between DNS resolving `hrns-platform.example` to an IP address, and `BaseController.ExtractHostFromRequest` reading the `Host`/`Origin` header once the request arrives — which layer is each operating at, and why does this platform need *both* mechanisms rather than just one?
