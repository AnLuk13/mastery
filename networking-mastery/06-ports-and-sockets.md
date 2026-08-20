# 6. Ports & Sockets

Goal: the concept that makes "how does one machine run a web server, a database, and a dozen other services simultaneously, all reachable at the same IP address" make sense — genuinely essential vocabulary for reading any connection string, config file, or `netstat`/`ss` output you'll ever encounter.

## 6.1 Ports — disambiguating "which process" on one host

An IP address (chapter 3) identifies a *host*. It says nothing about *which process on that host* a given piece of traffic is meant for — a single server can run a web API, a database, an SSH daemon, and more, all simultaneously, all reachable at the same IP address. **Ports** solve this: a 16-bit number (0–65535), included in every TCP/UDP segment (chapter 5), identifying the specific application-level endpoint.

```
192.168.1.20:443
     │        │
     IP    port
```

- **Well-known ports** (0–1023) are reserved by convention for standard services: `80` (HTTP), `443` (HTTPS), `22` (SSH), `53` (DNS), `5432` (PostgreSQL's conventional default). Binding to one of these on most operating systems traditionally requires elevated privileges — a real, if largely historical, security consideration.
- **Registered ports** (1024–49151) are used by a huge range of specific applications by convention, without requiring central allocation the way well-known ports do.
- **Ephemeral ports** (roughly 49152–65535, though the exact range is OS-configurable) are assigned *automatically* by the OS as the *source* port for an outgoing connection — you never choose these yourself; every time your browser opens a new connection to a server, the OS picks one from this pool. This is the pool that can genuinely run dry under sustained load if connections aren't being reused (chapter 5 §5.2's `TIME_WAIT` discussion) — a real, if uncommon, production incident category (usually diagnosed as "cannot assign requested address" or similar OS-level errors).

## 6.2 Sockets — IP + port, from the application's point of view

A **socket** is the actual programming/OS abstraction an application uses to send and receive network data — conceptually, "one endpoint of a connection," addressed by the combination of an IP address and a port (a **socket address**). A full TCP connection is uniquely identified by *four* values together — source IP, source port, destination IP, destination port — which is exactly what lets a server distinguish thousands of simultaneous connections from different clients all arriving at the *same* destination IP:port.

- A **listening socket** is what a server creates to *wait* for incoming connections on a specific port — Kestrel (the web server underneath ASP.NET Core) creates one of these on app startup.
- A **client socket** is created per outgoing connection — your browser creates one every time it opens a new connection to a server.

## 6.3 Grounded tour: every port this platform actually uses

This is one of the more satisfying exercises in the whole guide — every one of these is a real value from this workspace's own config, and now you have the vocabulary to actually understand what each line means:

```json
// HRNS.WebApi/appsettings.json — Kestrel's OWN listening sockets
"_Kestrel": {
  "Endpoints": {
    "Http":  { "Url": "http://localhost:5000" },
    "Https": { "Url": "https://localhost:5001" }
  }
}
```
This is the platform server's own listening socket configuration — port `5000` for plain HTTP, `5001` for HTTPS (chapter 9). Kestrel binds these ports at startup; if either is already in use by another process, the app fails to start — a very common "why won't my app start" error, and now you know exactly why: two processes cannot both hold a listening socket on the same IP:port simultaneously.

```json
"GeoApi": { "Address": "https://localhost:5005/geo-api" }   // HRNS.Gelocation.Server's listening port
"ClamAv": { "tcp": "tcp://localhost:3310" }                    // ClamAV daemon's listening port (chapter 5 §5.4)
```

`HRNS.Platform.UI`'s dev server listens on its own port too — `npm run dev` starts Vite on `http://localhost:3001` (per the UI project's own tooling config) — a *different* port from the API's `5000`/`5001`, which is precisely why the UI's dev-time API calls are **cross-origin** (different port = different origin, by the same-origin policy's definition — chapter 10 covers exactly why this matters) even though everything's running on the same machine. The Ollama AI server this platform talks to (`../dotnet-mastery/` chapter 13) listens on its own well-known default port too — `11434` — visible in `TestWebApplicationFactory`'s test config (`../dotnet-mastery/` chapter 9 §9.5): `["AIServer:Address"] = "http://localhost:11434"`.

**The pattern worth internalizing**: in local development, *everything* is `localhost` (chapter 3 §3.3 — no real network involved at all), and the *port* is the only thing distinguishing "the API," "the UI dev server," "the geolocation microservice," "ClamAV," and "the AI server" from each other, all coexisting on the same machine. In a real deployment (chapter 15), each of these would more typically get its *own* host/IP address (a separate container, VM, or managed service) and could reuse conventional ports (`443` for HTTPS everywhere) — it's really only the "everything on one dev machine" scenario that forces this many distinct, memorable, non-standard port numbers into existence at once.

## 6.4 Diagnosing with ports

`ss -tulpn` (Linux, modern) or `netstat -ano` (Windows/older) list every listening and established socket on a machine, along with the process holding it — the practical tool for answering "is anything actually listening on port 5001 right now" or "what process has port 5000 bound, since my app just failed to start with an address-in-use error" (chapter 17 covers these properly, hands-on).

## Checkpoint

1. If you started `HRNS.WebApi` locally twice at the same time (two terminal windows), what specific error would you expect, and which concept from this chapter explains it precisely?
2. `HRNS.Platform.UI`'s dev server on port `3001` calling `HRNS.Platform.Server`'s API on port `5001`, both on `localhost` — are these the same "origin" by the browser's definition? (You'll fully answer this in chapter 10; for now, reason from "what does a socket address actually consist of.")
3. Explain, using this chapter's vocabulary, why an ephemeral port being reused too quickly after a connection closes (before its `TIME_WAIT` period elapses, chapter 5 §5.2) could theoretically cause ambiguity — and why the OS's `TIME_WAIT` safety margin exists specifically to prevent that.
