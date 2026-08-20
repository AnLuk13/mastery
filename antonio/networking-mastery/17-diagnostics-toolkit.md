# 17. Diagnostics Toolkit

This is where every preceding chapter's theory becomes something you can actually run. Each tool below is mapped explicitly to the layer(s) it tests, following the §1.5 layered-debugging model this entire guide has been building toward — and the chapter closes with the single most valuable exercise in this curriculum: tracing one real request through every layer, end to end.

## 17.1 Layer-1/3: is the network even reachable?

```bash
ping example.com              # ICMP echo request/reply (chapter 4 §4.3) — network-layer reachability + rough latency
ping -c 4 192.168.1.1           # -c limits to 4 pings instead of running forever
traceroute example.com            # (tracert on Windows) — enumerates every router hop via TTL-exceeded (chapter 4 §4.1/§4.3)
```
A successful `ping` proves layers 1–3 are working and gives you a latency figure — nothing more. It says *nothing* about whether any application is actually listening at that address (chapter 4 §4.3's exact caveat) — a common, avoidable misdiagnosis.

## 17.2 Layer application: is DNS resolving correctly?

```bash
dig example.com                 # detailed DNS query — shows the actual record(s), TTL, which server answered
dig example.com MX                # query a specific record type (chapter 7 §7.3)
dig +trace example.com               # walk the FULL resolution hierarchy yourself (chapter 7 §7.2), root → TLD → authoritative
nslookup example.com                   # older, simpler, still common (especially on Windows)
```
`dig +trace` is genuinely worth running once deliberately — watching your own machine perform chapter 7 §7.2's hierarchy live, one hop of the DNS hierarchy at a time, is what makes that diagram stop being abstract.

## 17.3 Making the actual request

```bash
curl -v https://example.com                        # -v (verbose) shows TCP connect, TLS handshake, request AND response headers
curl -X POST https://api.example.com/thing \        # method, headers, body — exactly what your app's HttpClient does
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"key":"value"}'
curl -I https://example.com                            # HEAD request — headers only (chapter 8 §8.2)
wget https://example.com/file.zip                          # simpler than curl for straightforward downloads
```
`curl -v` is arguably the single most useful command in this entire chapter for a working engineer — its output shows you, in order, exactly the layered handshake this whole guide has walked through: `* Connected to ... port 443` (TCP, chapter 5), `* TLS handshake, ...` (chapter 9), then the actual HTTP request/response headers (chapter 8) — one command, every layer from chapter 5 up, visible and inspectable.

## 17.4 What's actually listening, locally

```bash
ss -tulpn              # modern Linux: TCP+UDP listening sockets, with process names (chapter 6 §6.4)
netstat -ano            # Windows / older systems — same idea
lsof -i :5001              # "what process is using port 5001" — genuinely the fastest answer to "why won't my app start" (chapter 6 §6.3)
```
This is the direct, practical follow-up to chapter 6 §6.4's "is Kestrel's port already taken" scenario — `ss -tulpn` or `lsof -i :5001` answers it in one command.

## 17.5 Looking at actual packets — where networking really starts "clicking"

```bash
tcpdump -i any port 443           # capture packets on any interface, filtered to port 443
tcpdump -i lo0 port 5001              # capture ONLY loopback traffic (chapter 3 §3.3) to your local dev server
```
**Wireshark** is `tcpdump`'s graphical counterpart — same underlying packet capture, with an interactive UI for drilling into individual packets layer by layer (literally labeled Ethernet/IP/TCP/TLS/HTTP in its display, chapter 1 §1.2's encapsulation made visible). Genuinely worth doing once, deliberately: capture traffic while making one real HTTPS request, and identify the TCP handshake (chapter 5 §5.2's SYN/SYN-ACK/ACK), the TLS handshake (chapter 9 §9.3), and the actual HTTP request/response — as literal, separate, inspectable groups of packets, not just a diagram in this guide.

## 17.6 Testing raw TCP/UDP connectivity — below HTTP entirely

```bash
nc -zv example.com 443          # netcat: "zero-I/O, verbose" — just test whether a TCP connection to this port succeeds
nc -u -zv example.com 53           # same, for UDP (less reliable as a test, since UDP has no handshake to confirm against)
nc -l 9000                            # listen on a port — useful for quickly standing up a throwaway TCP endpoint to test against
```
This is exactly the right tool for testing something like `HRNS.Platform.Server`'s raw-TCP ClamAV connection (chapter 5 §5.4) — `nc -zv localhost 3310` answers "is anything listening there at all" without needing to speak ClamAV's actual protocol, isolating the transport-layer question from the application-layer one.

## 17.7 Inspecting TLS directly

```bash
openssl s_client -connect example.com:443                 # manually perform (and inspect) a TLS handshake (chapter 9 §9.3)
openssl s_client -connect example.com:443 -servername example.com   # SNI — needed when multiple certs share one IP
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates -subject -issuer
                                                              # extract just the certificate's validity dates and chain info (chapter 9 §9.4)
```
This is how you'd manually verify a certificate's expiry date or issuing CA without relying on a browser's UI — genuinely useful when diagnosing "why is this specific client rejecting our certificate" issues.

## 17.8 SSH toolset, gathered in one place (chapter 16, for reference)

```bash
ssh user@host                    # connect
ssh -L 5433:db.internal:5432 bastion   # local port forwarding (chapter 16 §16.5)
scp file.txt user@host:/path/          # secure copy
sftp user@host                           # interactive file transfer
ssh-keygen -t ed25519                       # generate a modern key pair
ssh-add ~/.ssh/id_ed25519                     # load a key into your agent
```

## 17.9 The cumulative exercise — trace the whole thing, yourself

Chapter 1 §1.5 promised this: by the end of this guide, you should be able to explain every arrow in this chain, using the actual tools above to verify each step rather than reciting it from memory.

```
You open your browser, type https://app.hrns.example.com
        │
        ▼
DNS lookup                          ← dig app.hrns.example.com (ch. 7)
        │
        ▼
IP address returned
        │
        ▼
Routing (many hops)                 ← traceroute app.hrns.example.com (ch. 4)
        │
        ▼
TCP connection (SYN/SYN-ACK/ACK)     ← curl -v (shows "Connected to...") (ch. 5)
        │
        ▼
TLS handshake                        ← curl -v (shows the TLS negotiation), or openssl s_client (ch. 9)
        │
        ▼
HTTPS request sent                    ← the actual HTTP request line + headers, visible in curl -v (ch. 8)
        │
        ▼
Reverse proxy / load balancer           ← terminates TLS, adds X-Forwarded-* headers, routes to an instance (ch. 4 §4.2, ch. 12)
        │
        ▼
HRNS.Platform.Server (Kestrel)            ← BaseController resolves host/origin, [Authorize] checks the JWT (../dotnet-mastery/ ch.3, ch.8)
        │
        ▼
MediatR → CQRS handler                      ← the actual business logic (../dotnet-mastery/ ch.5)
        │
        ▼
Database (PostgreSQL, over its own TCP)       ← Npgsql, pooled connection (ch. 5 §5.4, ../dotnet-mastery/ ch.4)
        │        (possibly ALSO: a call to HRNS.Gelocation.Server, File Server, etc. — ch. 14)
        ▼
HTTP response                                   ← status code, headers, JSON body (ch. 8)
        │
        ▼
Browser receives it — CORS check FIRST             ← was this origin allowed to read the response? (ch. 10)
        │
        ▼
Your JavaScript processes the response
```

Do this for real, once, against this platform's own local dev setup: start `HRNS.Platform.Server`, open the Network tab in your browser, trigger a login, and — using `curl -v` against the same endpoint separately, `ss -tulpn` to confirm what's listening, and the browser's own Network tab timing breakdown (which typically shows DNS/connect/TLS/request/response as separate phases, directly mirroring this chain) — identify each step actually happening. If you can point to real, concrete evidence for every arrow above, you've genuinely completed this guide's core goal.

## Checkpoint

1. Run `curl -v https://example.com` (or any HTTPS site) right now and identify, in the output, exactly where the TCP connection is established and exactly where the TLS handshake happens — are they distinguishable as separate phases in the output?
2. If `ping app.hrns.example.com` succeeds but `curl https://app.hrns.example.com/api/health` hangs indefinitely (no response, no error), which tool from this chapter would you reach for *next*, and what would you be trying to rule in or out?
3. Complete §17.9's exercise against this platform's local dev environment and write down, for your own reference, which specific tool/browser-devtools-feature gave you visibility into each arrow in the chain.
