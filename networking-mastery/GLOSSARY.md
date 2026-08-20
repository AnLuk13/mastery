# Glossary

Every acronym and term used across this guide, one line each. Chapter references point to where the term is explained in depth.

**Anycast** — one IP address announced from many physical locations; routing naturally sends each user to the nearest one (ch. 12, 18).
**ARP** — Address Resolution Protocol; resolves an IP address to a MAC address on a local network (ch. 2).
**Backoff (exponential)** — waiting progressively longer between retry attempts to avoid overwhelming a struggling dependency (ch. 14).
**Bandwidth vs. throughput** — maximum capacity of a link vs. what's actually achieved in practice (ch. 1).
**Bastion host** — a single, hardened, internet-reachable server that's the only entry point into an otherwise-private network (ch. 16).
**BGP** — Border Gateway Protocol; how independent networks announce reachability to each other across the internet (ch. 18).
**Broadcast domain** — the set of devices that receive a broadcast frame; one LAN segment, by default (ch. 2).
**CDN** — Content Delivery Network; geographically distributed caching reverse proxies (ch. 12).
**Certificate / Certificate Authority (CA)** — a signed binding of a public key to an identity, vouched for by a trusted issuer (ch. 9).
**Cipher suite** — the specific combination of key-exchange, encryption, and hashing algorithms negotiated for one TLS connection (ch. 9).
**Circuit breaker** — stops attempting calls to a failing dependency for a cooldown period instead of repeatedly waiting out timeouts (ch. 14).
**CIDR** — Classless Inter-Domain Routing; the `/24`-style notation for IP address ranges and subnet masks (ch. 3).
**Connection pooling** — reusing established TCP connections instead of opening a new one per request/query (ch. 5, 6, 14).
**CORS** — Cross-Origin Resource Sharing; the server opt-in mechanism relaxing the browser's same-origin policy (ch. 10).
**Correlation ID** — an identifier threaded through a request (and its downstream calls) to reconstruct which log lines belong together (ch. 14).
**DNAT / SNAT / PAT** — Destination/Source Network Address Translation; PAT lets many devices share one public IP via port rewriting (ch. 4, 15).
**DNS** — Domain Name System; resolves domain names to IP addresses via a hierarchical, cached, distributed lookup (ch. 7).
**Encapsulation** — each network layer wraps the layer above it as opaque payload, only reading its own header (ch. 1).
**Ephemeral port** — a port automatically assigned by the OS as the source port for an outgoing connection (ch. 6).
**Ethernet frame** — the link-layer envelope wrapping a packet with source/destination MAC addressing (ch. 2).
**Forward proxy vs. reverse proxy** — represents the client vs. represents the server (ch. 12).
**Forward secrecy** — each TLS session's key is derived fresh and never stored, so a future key compromise can't decrypt past traffic (ch. 9).
**Head-of-line blocking** — one slow/lost item blocking everything queued behind it on the same channel; motivated HTTP/2 and then HTTP/3/QUIC (ch. 8).
**Host key vs. user key** — SSH's server-identity key vs. your own client authentication key (ch. 16).
**Idempotent** — calling an operation once has the same effect as calling it many times; safe to retry blindly (ch. 8, 14).
**Jitter** — variation in latency over time (ch. 1).
**Latency** — the time for one bit/packet to travel from sender to receiver (ch. 1).
**Load balancer (L4 vs L7)** — distributes traffic across backend instances, at the transport layer or the HTTP layer respectively (ch. 12).
**MAC address** — a link-layer hardware address identifying a network interface (ch. 2).
**MITM** — Man-in-the-middle; an attacker intercepts traffic between two parties who believe they're talking directly (ch. 13).
**NAT** — Network Address Translation; lets private IP addresses reach the public internet (ch. 4).
**NDP** — Neighbor Discovery Protocol; IPv6's equivalent of ARP (ch. 2).
**Origin (same-origin policy)** — the exact combination of scheme + host + port; the browser's cross-origin security boundary (ch. 10).
**Port** — a 16-bit number identifying which process on a host a piece of traffic is meant for (ch. 6).
**Private vs. public IP address** — RFC 1918 ranges not globally routable vs. globally unique, internet-routable addresses (ch. 3).
**Proxy (forward/reverse/transparent/SOCKS)** — see Forward proxy vs. reverse proxy; SOCKS relays arbitrary traffic below the HTTP layer (ch. 12, 16).
**QUIC** — the UDP-based transport protocol underneath HTTP/3, avoiding TCP's head-of-line blocking (ch. 5, 8).
**Reverse DNS (PTR)** — resolving an IP address back to a domain name (ch. 7).
**RST (TCP)** — an abrupt, non-graceful TCP connection termination (ch. 5).
**Route table / default gateway** — where a device sends traffic destined for networks it isn't directly connected to (ch. 4).
**Service discovery** — how a caller finds out where a callee currently is, given that could change (ch. 14, 15).
**Session cookie attributes (Secure/HttpOnly/SameSite)** — controls limiting when and how a cookie is sent/read (ch. 10).
**SFTP / SCP** — file transfer protocols riding over an existing authenticated SSH connection (ch. 16).
**SOA / NS / TXT / MX / A / AAAA / CNAME / SRV** — DNS record types (ch. 7).
**Socket** — an application's endpoint for sending/receiving network data, addressed by IP + port (ch. 6).
**SPF / DKIM / DMARC** — DNS-based email authenticity mechanisms protecting against spoofed sending domains (ch. 7).
**SSH** — Secure Shell; an authenticated, encrypted remote-access protocol, also used for tunneling (ch. 16).
**SSH tunneling (local/remote/dynamic forwarding)** — carrying other TCP traffic through an established SSH connection (ch. 16).
**Subnet mask** — defines which bits of an IP address are the network portion vs. the host portion (ch. 3).
**Switch** — a link-layer device forwarding Ethernet frames only to the port with the matching destination MAC (ch. 2).
**TCP handshake (SYN/SYN-ACK/ACK)** — the three-step process establishing a reliable TCP connection (ch. 5).
**TCP vs. UDP** — reliable, ordered, connection-oriented vs. best-effort, connectionless, low-overhead (ch. 5).
**TIME_WAIT** — a TCP connection-closing state held briefly to avoid confusing delayed duplicate packets with a new connection (ch. 5).
**TLS handshake** — the negotiation establishing an encrypted, authenticated connection before HTTP data flows (ch. 9).
**Trust On First Use (TOFU)** — SSH's model of pinning a server's host key on first connection rather than using a CA hierarchy (ch. 16).
**TTL (Time To Live)** — a packet field decremented per router hop, used by `traceroute`; also DNS's record-caching duration — two different meanings, same term (ch. 4, 7).
**VPN** — Virtual Private Network; an always-on encrypted tunnel carrying a device's or network's traffic transparently (ch. 16).
**Well-known port** — a reserved, conventional port (0–1023) for a standard service, e.g. 443 for HTTPS (ch. 6).
**X-Forwarded-For / X-Forwarded-Proto / X-Forwarded-Host** — headers a reverse proxy adds to preserve the original client's info after NAT/proxying (ch. 4, 9).
