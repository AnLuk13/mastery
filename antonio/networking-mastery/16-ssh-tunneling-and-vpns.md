# 16. SSH, Tunneling & VPNs

This was the original question that prompted this whole curriculum. Everything in Parts 1–3 was, deliberately, the foundation that makes this chapter make sense on the first read instead of feeling like a list of commands to memorize. There's less direct `HRNS.Platform.Server`/`.UI` *application* code to ground this in than earlier chapters — SSH is dev/ops tooling, not something the app itself uses — but this workspace's own git remotes (GitLab, `../dotnet-mastery/` chapter 0) are, in practice, exactly what most engineers use SSH keys for day to day, and it's called out below.

## 16.1 SSH — what it actually is, in the vocabulary you now have

```
SSH
 │
 ├── an APPLICATION-layer protocol (chapter 1 §1.4)
 ├── runs over TCP (chapter 5) — reliable, ordered delivery matters for a shell session
 ├── conventionally on port 22 (chapter 6 §6.1's well-known ports)
 ├── uses asymmetric + symmetric cryptography together (chapter 9 §9.2) — same pattern as TLS
 ├── authenticates the SERVER to the client (so you know you're not connecting to an impostor)
 ├── authenticates the CLIENT (you) to the server
 └── establishes an ENCRYPTED, bidirectional channel for the actual session
```

Structurally, SSH's connection setup is doing almost exactly what TLS does (chapter 9 §9.3) — negotiate algorithms, exchange keys asymmetrically to derive a shared symmetric session key, then switch to fast symmetric encryption for the actual traffic — SSH just predates TLS as a widely-deployed protocol and evolved its own independent implementation of the same underlying cryptographic ideas.

## 16.2 Two different kinds of keys, and why the distinction matters

**Host keys** — belong to the *server*, and exist to answer "am I actually connecting to the machine I think I am" (the SSH equivalent of chapter 9 §9.4's certificate/hostname verification, though SSH doesn't use a CA hierarchy — it uses **Trust On First Use** instead, below). **User keys** — belong to *you*, and exist to answer "prove you're actually who you claim to be," the authentication half.

**`known_hosts`** (`~/.ssh/known_hosts`) is where your client remembers every server host key it's seen before. The very first time you connect to a new server, SSH shows you the host key's fingerprint and asks you to confirm it looks right — this is **Trust On First Use (TOFU)**: no central authority vouches for the server's identity the way a TLS CA does (chapter 9 §9.4); you're trusting that the *first* connection wasn't already compromised, then pinning that key for every connection after. This is *why* SSH warns loudly, unmissably, if a host key ever changes unexpectedly — that's either a legitimate server reinstall/key rotation, or a genuine man-in-the-middle attempt (chapter 13 §13.2), and SSH has no way to tell the two apart automatically, so it forces you to.

**`authorized_keys`** (on the *server*, per user account) — the list of public keys allowed to authenticate as that user. **Public-key authentication** — the server sends a challenge the client can only correctly respond to if it holds the matching *private* key — is strictly preferable to **password authentication** (no password to brute-force or phish, and the private key never leaves your machine at all during authentication, unlike a password which — depending on the exact protocol variant — could theoretically be exposed to a compromised server). `ssh-keygen` generates a key pair; `ssh-copy-id` (or manually appending) installs your public key into a server's `authorized_keys`.

## 16.3 SSH agents & config — making key-based auth practical

An **SSH agent** (`ssh-agent`, with `ssh-add` to load a key into it) holds your decrypted private key in memory for the duration of a session, so you're not re-typing your key's passphrase on every single connection — the agent handles the cryptographic challenge-response on your behalf, without ever handing the raw private key material to whatever process (or server) asked. **Agent forwarding** lets a remote server you've SSH'd into use *your local* agent to authenticate onward to a *third* server — convenient, and worth knowing the real security trade-off: it means a compromised intermediate server could potentially abuse your forwarded agent to authenticate elsewhere as you, for as long as that forwarded session is open — a real reason to forward an agent deliberately, not by default.

`~/.ssh/config` lets you define per-host shortcuts and defaults instead of typing full options every time:
```
Host geo-staging
    HostName 10.20.30.40
    User deploy
    IdentityFile ~/.ssh/id_ed25519_deploy
    Port 2222
```
`ssh geo-staging` then just works, using exactly these settings.

## 16.4 SCP & SFTP — file transfer over the same authenticated channel

Both reuse SSH's existing authenticated, encrypted connection rather than needing separate credentials/setup — **SCP** (Secure Copy) is simple, one-shot file copying; **SFTP** (SSH File Transfer Protocol — despite the name, not related to FTP's actual protocol, just similar in spirit) is a fuller, interactive file-management protocol (list directories, resume transfers, etc.) over the same SSH transport. Neither opens a separate TCP connection on a separate port the way plain FTP historically did — everything rides over the one already-authenticated SSH session.

## 16.5 SSH tunneling — the part that makes SSH genuinely powerful, not just "remote shell"

This is where the whole rest of this guide pays off directly — SSH tunneling is *just* using an already-established, encrypted TCP connection (§16.1) to carry *other* traffic through it, tunneled inside.

**Local port forwarding** — a port on *your* machine gets forwarded, through the SSH connection, to a destination reachable *from the server's side*:
```bash
ssh -L 5433:internal-db.private:5432 bastion-host
# now connecting to localhost:5433 on YOUR machine actually reaches
# internal-db.private:5432, as seen from bastion-host's network — even
# though your machine has no direct route to that private database at all
```
This is exactly how you'd reach a database sitting in a cloud VPC's private subnet (chapter 15 §15.4 — no direct internet route, by design) from your laptop, without ever exposing that database directly to the internet: the bastion host is the only thing with actual network access to it, and SSH tunnels your traffic through that one narrow, authenticated path.

**Remote port forwarding** — the reverse direction: a port on the *remote* server gets forwarded back to something reachable from *your* machine:
```bash
ssh -R 8080:localhost:3000 remote-host
# now connecting to localhost:8080 on remote-host reaches YOUR machine's
# localhost:3000 — useful for briefly exposing something running locally
# to a remote environment for testing, without a public deployment
```

**Dynamic port forwarding** (`ssh -D 1080 some-host`) turns your SSH client into a **SOCKS proxy** (chapter 12 §12.1) — any application configured to use `localhost:1080` as a SOCKS proxy has *all* of its traffic tunneled through that one SSH connection, not just one fixed destination — effectively an ad-hoc, single-hop VPN (§16.7) built entirely out of SSH, no separate VPN software required.

**ProxyJump / bastion hosts** — a **bastion host** (or "jump box") is a single, deliberately hardened, internet-reachable server that's the *only* entry point into an otherwise-private network — every other server is only reachable *through* it, never directly:
```bash
ssh -J bastion-host internal-server     # -J = ProxyJump: hop through bastion-host to reach internal-server
```
This is chapter 13 §13.1's "minimize attack surface" principle applied directly to remote access: instead of every internal server needing its own public IP and its own exposed SSH port (each one more surface area to secure and monitor), exactly one hardened host is exposed, and everything else's SSH access is reachable only by first authenticating there. `~/.ssh/config`'s `ProxyJump` directive lets you configure this once and then just `ssh internal-server` as if the jump were transparent.

## 16.6 Grounded, honestly: where this touches this workspace at all

There's genuinely no SSH-specific code in `HRNS.Platform.Server`/`.UI` — this entire chapter is infrastructure/dev-tooling knowledge, not application code, and it would be manufacturing a tie-in to pretend otherwise. The one concrete, real touchpoint: this workspace's git remotes point at `gitlab.com/hrns-platform/...` (`../dotnet-mastery/` chapter 0's tech table) — pushing to those over SSH (`git@gitlab.com:...` remote URLs, as opposed to HTTPS remotes) is precisely public-key authentication (§16.2) in its most common day-to-day form for a working engineer, whether or not you've thought of `git push` as "an SSH connection" before. If/when this platform's deployment pipeline (`../dotnet-mastery/` chapter 11, which found no committed CI/CD config) does get built out, SSH — or a cloud-provider-specific equivalent built on the same underlying ideas — is very likely how a deploy process would actually reach a production server to run migrations, restart services, or pull new container images, exactly the bastion-host pattern from §16.5.

## 16.7 VPNs — the same tunneling idea, generalized beyond one application

A **VPN** (Virtual Private Network) generalizes §16.5's SSH tunneling idea into something that carries *all* of a device's (or a whole network's) traffic through an encrypted tunnel, transparently, rather than requiring each individual connection to be deliberately forwarded:

```
Laptop
   │  (all traffic)
   │ encrypted tunnel
   ▼
VPN server / gateway
   │
   ▼
Private network  (or: onward to the internet, appearing to originate from the VPN server)
```

- **Remote-access VPN** — one device connects into a private network remotely (the individual-engineer-working-from-home case) — conceptually the multi-application generalization of what §16.5's dynamic port forwarding does for one SSH client.
- **Site-to-site VPN** — two entire networks (e.g. two office locations, or an on-prem data center and a cloud VPC, chapter 15 §15.4) are bridged together over an encrypted tunnel across the public internet, so machines on either side can reach each other as if on one network — no individual device configuration needed once the tunnel exists.
- **WireGuard** — a modern, deliberately minimal VPN protocol/implementation, widely regarded as simpler to audit and reason about than older alternatives.
- **IPsec** — an older, more complex but extremely widely deployed suite of protocols for authenticated, encrypted IP-layer tunnels — often what "site-to-site VPN" means in an enterprise/cloud-provider context specifically.
- **OpenVPN** — another long-standing, widely deployed VPN implementation, running over either TCP or UDP.

The throughline worth keeping from this whole chapter: **SSH tunneling and VPNs are the same core idea — an authenticated, encrypted tunnel carrying other traffic — at different scopes.** SSH tunnels one (or a few, or effectively-all-via-SOCKS) connections, deliberately, per-command; a VPN tunnels a whole device's or network's traffic, transparently, once established. Once §16.5 has genuinely clicked, a VPN stops looking like separate magic and starts looking like "the same idea, running always-on instead of per-command."

## Checkpoint

1. Explain, precisely, what Trust On First Use actually protects against and what it *doesn't* — if an attacker successfully man-in-the-middled your *very first* connection to a brand-new server, would `known_hosts` catch it?
2. Sketch the exact `ssh -L ...` command you'd use to reach a PostgreSQL instance listening on `5432` inside a private cloud subnet (chapter 15 §15.4), reachable only via a bastion host at `bastion.example.com`, such that connecting to `localhost:5433` on your own laptop reaches it.
3. Using §16.5–§16.7's framing, explain in one paragraph why `ssh -D` (dynamic forwarding/SOCKS) is meaningfully closer to "a lightweight, on-demand VPN" than local or remote port forwarding are — what specifically changes about how many destinations get tunneled?
