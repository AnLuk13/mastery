# 9. TLS, HTTPS & Cryptography

Goal: understand what HTTPS actually adds on top of HTTP mechanically (not just "it's encrypted"), the cryptographic primitives that make it possible, and how certificates establish trust — the security layer sitting between chapter 5 (transport) and chapter 8 (HTTP) in the stack picture from the index.

## 9.1 HTTPS = HTTP over TLS — not a separate protocol

A genuinely important reframe if you've never thought about it this way: **HTTPS is not a different application protocol from HTTP.** It's the exact same HTTP (chapter 8) — same methods, same status codes, same headers — carried over a connection that TLS has already encrypted and authenticated, instead of over a bare TCP connection.

```
HTTP
 ↓
TLS      ← this is the only layer HTTPS adds
 ↓
TCP
 ↓
IP
```

**TLS** (Transport Layer Security — the modern name; "SSL" is its deprecated predecessor, still used colloquially) provides three distinct guarantees, worth naming separately because each is solved by a different cryptographic mechanism:

1. **Confidentiality** — nobody eavesdropping on the connection can read the data (encryption).
2. **Integrity** — nobody can tamper with the data in transit without detection (authenticated encryption / MACs).
3. **Authentication** — you can verify you're actually talking to the server you intended to (certificates).

## 9.2 Cryptographic primitives — what each one is actually for

You don't need to implement any of these yourself — .NET's cryptography libraries (already used throughout `HRNS.Platform.Server`, `../dotnet-mastery/` chapter 8) handle the math. What matters is knowing *which primitive solves which problem*, so you can reason about a security design instead of treating it as a black box.

- **Symmetric encryption** (AES, ChaCha20) — one shared secret key encrypts *and* decrypts. Fast, but requires both sides to already have the same key — which raises the obvious question: how do two parties who've never met agree on a shared secret over a network an attacker might be watching?
- **Asymmetric (public-key) cryptography** (RSA, elliptic curves like Ed25519) — a *pair* of mathematically related keys: whatever's encrypted with the public key can only be decrypted with the private key (and vice versa, for signing). Solves the "how do we get a shared secret without ever transmitting it" problem: TLS uses asymmetric crypto briefly, during the handshake, specifically to *establish* a symmetric key — then switches to symmetric encryption for the actual data, because it's dramatically faster. This exact same public/private key pattern is what you already know from SSH (chapter 16) and from `HRNS.Platform.Server`'s own use of asymmetric-adjacent primitives (`../dotnet-mastery/` chapter 8 §8.3's JWT signing key, though that specific case uses a *symmetric* key — worth noting the platform's JWTs are HMAC-signed, not RSA-signed, a real, deliberate simplicity/performance trade-off).
- **Hashing** (SHA-256, SHA-512) — a one-way function: easy to compute in one direction, computationally infeasible to reverse. Used for integrity checks (does this data match what was expected) and as a building block for other primitives — you've already seen this directly: `Encryptor.CreatePasswordHash`'s `HMACSHA512` (`../dotnet-mastery/` chapter 8 §8.5) is a *keyed* hash (HMAC = Hash-based Message Authentication Code — a hash function combined with a secret key, giving you integrity *and* authentication in one primitive).
- **Digital signatures** — proving a piece of data was created by the holder of a specific private key, verifiable by anyone with the corresponding public key, without revealing the private key. This is exactly what a TLS certificate *is* — the certificate authority's digital signature over "this public key belongs to this domain" (§9.4).
- **Key exchange** (Diffie-Hellman, ECDH — Elliptic-Curve Diffie-Hellman) — the actual algorithm two parties use to agree on a shared symmetric key over an untrusted network, without ever transmitting the key itself — remarkably, both sides end up with the same secret by combining their own private value with the other side's *public* value, in a way an eavesdropper watching only the public values can't replicate. This is what happens during the TLS handshake, §9.3.

## 9.3 The TLS handshake

```
Client                                          Server
  │ ── ClientHello (supported versions/ciphers) ──► │
  │                                                 │
  │ ◄── ServerHello + certificate + key exchange ── │
  │                                                 │
  │   (both sides derive the same symmetric key      │
  │    via key exchange, without transmitting it)      │
  │                                                 │
  │ ── Finished ────────────────────────────────► │
  │ ◄──────────────────────────────── Finished ─── │
  │                                                 │
  │        ENCRYPTED APPLICATION DATA (HTTP)          │
```

**TLS 1.2 vs. TLS 1.3**: TLS 1.3 (the current standard, and what QUIC/HTTP-3 requires — chapter 8 §8.4) trimmed this handshake to fewer round-trips, removed several legacy/weaker cipher options entirely (rather than just deprecating them), and made **forward secrecy** the default rather than optional. **Forward secrecy** means the symmetric key for *each* session is derived fresh (via ephemeral Diffie-Hellman) and never stored — so even if a server's long-term private key were ever compromised in the future, *past* recorded traffic still can't be decrypted, because the actual session keys never existed anywhere retrievable. A **cipher suite** is simply the specific combination of algorithms (key exchange + symmetric cipher + hash function) negotiated for one connection — visible in tools like `openssl s_client` (chapter 17).

## 9.4 Certificates & the chain of trust

A TLS certificate is a small, signed document binding a **public key** to an **identity** (typically a domain name), issued and digitally signed (§9.2) by a **Certificate Authority** (CA) — an entity your OS/browser has been told, in advance, to trust. When your browser connects to a server presenting a certificate, it doesn't just trust the certificate blindly — it verifies the **certificate chain**: the server's certificate is signed by an intermediate CA, whose certificate is in turn signed by a **root CA**, whose certificate is pre-installed and trusted by your OS/browser as a starting axiom. If every link in that chain checks out *and* the certificate's stated domain matches the domain you actually connected to (**hostname verification** — a step it's genuinely possible, and dangerous, to skip or misconfigure), the browser proceeds; otherwise, you get the "connection is not private" warning.

```
Root CA (trusted by your OS/browser, pre-installed)
   │ signs
   ▼
Intermediate CA
   │ signs
   ▼
example.com's certificate  ← what the server actually presents
```

## 9.5 TLS termination — a real, deliberate configuration choice in this platform

You've already seen the concrete evidence of this in `../dotnet-mastery/` chapter 8 §8.3: `HRNS.Platform.Server`'s JWT bearer configuration sets `RequireHttpsMetadata = false`. Read plainly, out of context, that looks alarming — but it's a completely standard deployment pattern, and worth understanding precisely *why* it's fine: in a real deployment, TLS is typically **terminated** at a reverse proxy/load balancer (chapter 12) sitting in front of the application — the proxy holds the real certificate, handles the TLS handshake with the actual client, and forwards the *decrypted* HTTP request onward to the app server over a private, trusted internal network (which may not use TLS at all internally, or uses a separate, simpler internal certificate). `ForwardedHeadersOptions`/`X-Forwarded-Proto` (chapter 4 §4.2 of this guide) is exactly how the app recovers "the original request *was* HTTPS from the client's point of view" even though the connection it directly received, from the proxy, wasn't necessarily. This is precisely the kind of line that's worth asking about explicitly on any codebase you're new to, rather than assuming — and now you have the vocabulary to ask the right follow-up question ("where does TLS actually terminate in this deployment") instead of just flagging the line as suspicious.

## Checkpoint

1. Explain, in your own words, why TLS uses *asymmetric* cryptography only briefly during the handshake, then switches to *symmetric* encryption for the actual data — what would go wrong (performance-wise) if it used asymmetric encryption for the whole session?
2. `Encryptor.VerifyPasswordHash` (`../dotnet-mastery/` chapter 8 §8.5) uses `HMACSHA512` — a keyed hash — rather than a plain, unkeyed `SHA512` hash. Using this chapter's vocabulary (§9.2), what extra property does the "keyed" part provide that a plain hash wouldn't?
3. If `RequireHttpsMetadata = false` in a deployment where TLS is *not* actually terminated at a trusted reverse proxy in front of the app (i.e., raw HTTP reaches the app directly from the internet), what would actually be at risk? Contrast that with the legitimate reverse-proxy-termination scenario described in §9.5.
