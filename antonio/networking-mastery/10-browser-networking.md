# 10. Browser Networking

This chapter didn't exist as its own topic in the source roadmap — it got folded into a single mention of "CORS" under web security. It deserves its own chapter here because **the browser enforces network rules no other HTTP client does**, and understanding them precisely (not just "CORS errors are annoying") is directly what makes `HRNS.Platform.UI`'s API layer, and any cross-origin frontend work you do, predictable instead of trial-and-error.

## 10.1 The same-origin policy — the rule underneath everything in this chapter

An **origin** is the exact combination of **scheme + host + port**. Two URLs are the *same* origin only if all three match exactly:

```
https://app.example.com:443/foo   and   https://app.example.com:443/bar     → SAME origin (path doesn't matter)
https://app.example.com           and   http://app.example.com               → DIFFERENT (scheme differs)
https://app.example.com           and   https://api.example.com               → DIFFERENT (host differs)
https://app.example.com:443       and   https://app.example.com:8443           → DIFFERENT (port differs)
```

The browser's **same-origin policy** is a security boundary enforced entirely client-side: by default, JavaScript running on one origin cannot read the response of a network request made to a *different* origin, even though the request itself often still gets sent. This is a browser-only rule — `curl`, a mobile app, a server calling another server (chapter 14) — none of them are bound by it at all. It exists specifically because a browser routinely runs untrusted third-party JavaScript (any ad, any script tag on any page you visit) in a context that also has your logged-in cookies for other sites — same-origin policy is what stops that untrusted script from silently reading your bank's API responses using your own ambient session.

**Concretely, in this workspace**: `HRNS.Platform.UI`'s dev server runs on `http://localhost:3001`; `HRNS.Platform.Server`'s API listens on `https://localhost:5001` (chapter 6 §6.3). Different port *and* different scheme — by the definition above, these are unambiguously different origins, even on the same machine during local development. Every single API call this frontend makes is, by definition, a **cross-origin request** — which is exactly why CORS (§10.2) isn't an edge case for this platform, it's the *normal*, every-request case.

## 10.2 CORS — the mechanism that lets cross-origin requests work at all

**CORS** (Cross-Origin Resource Sharing) is the *server's* opt-in mechanism for relaxing the same-origin policy for specific origins — a set of response headers telling the browser "it's fine for JavaScript running on *this* origin to read *this* response."

**Simple requests** (a narrow category — `GET`/`POST`/`HEAD`, only a small allowed set of headers, `Content-Type` limited to a few simple values) get sent directly; the browser just checks the response's `Access-Control-Allow-Origin` header before deciding whether to expose the response to the calling JavaScript. **Everything outside that narrow category** — which, notably, includes almost every real API call this platform makes, since it sends `Content-Type: application/json` and a custom `Authorization` header (§10.4) — triggers a **preflight** request first:

```
Browser, BEFORE the real request:

OPTIONS /api/Users/LoginAsync HTTP/1.1
Origin: http://localhost:3001
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type, authorization

Server responds:

Access-Control-Allow-Origin: http://localhost:3001  (or *)
Access-Control-Allow-Methods: POST, GET, ...
Access-Control-Allow-Headers: content-type, authorization
```

Only if the server's preflight response actually permits the method and headers the real request needs does the browser then send the **actual** request at all. This is worth being precise about, because it surprises people: **a failed CORS preflight means the real request may never even reach your server's application code** — the browser refuses to send it, full stop. This is why "check the Network tab for an `OPTIONS` request" is the very first diagnostic step for any CORS problem — if you only see the failed `OPTIONS`, the bug is in CORS *configuration*, not in your endpoint's actual logic.

## 10.3 How this platform's CORS is actually configured

```csharp
services.AddCors(options =>
{
    options.AddPolicy(name: _corsConfigurationName, policyOpts =>
    {
        var origins = (webAppBuilder.Configuration["AllowedOrigins"] ?? "").Split(';');
        if (origins.Length > 0 && !origins.Contains("*"))
            policyOpts.WithOrigins(origins);
        else
            policyOpts.AllowAnyOrigin();

        policyOpts.AllowAnyHeader();
        policyOpts.AllowAnyMethod();
        policyOpts.WithExposedHeaders("Content-Disposition");
        policyOpts.WithExposedHeaders("x-file-server-error");
        policyOpts.WithExposedHeaders("x-file-integrity");
    });
});
// ...
app.UseCors(_corsConfigurationName); // must run after UseRouting(), before UseAuthentication() — ../dotnet-mastery/ ch.3 §3.1
```
(`HRNS.WebApi/Program.cs:333-356`, condensed — already introduced in `../dotnet-mastery/` chapter 3 §3.1 purely as a middleware-ordering example; here's the *content* of that policy, in full.)

Two things worth understanding precisely here:

**`WithExposedHeaders(...)` is solving a real, separate CORS restriction most people never hit.** By default, even on a *successful*, CORS-permitted cross-origin response, JavaScript can only read a small allow-list of "safe" response headers (`Content-Type`, `Content-Length`, a few others) — any custom header (`Content-Disposition`, used for file-download filenames; `x-file-server-error`/`x-file-integrity`, custom application headers this platform apparently uses for file-serving diagnostics) is invisible to frontend JavaScript unless the server explicitly exposes it via `Access-Control-Expose-Headers`. This is exactly the mechanism `WithExposedHeaders` configures — without it, `HRNS.Platform.UI` could successfully download a file but never be able to read the filename `Content-Disposition` was trying to tell it.

**`AllowedOrigins` is config-driven, not hardcoded** — the same pattern as everything else in `../dotnet-mastery/` chapter 2 §2.5: a `;`-separated list read from `appsettings.json`/environment, defaulting to `AllowAnyOrigin()` if unset or explicitly `*`. `AllowAnyOrigin()` is a real, deliberate looseness worth having an opinion on: it's a reasonable default for a documented, versioned public API with no cookie-based ambient auth (§10.4 explains exactly why that caveat matters), but would be a real risk for an API that trusted cookies for authentication — worth revisiting once you've read §10.4.

## 10.4 How the frontend actually calls the API — and why cookies never entered this chapter until now

```typescript
// HRNS.Platform.UI/src/services/base/base-api.service.ts
protected apiURL = `${import.meta.env.VITE_API_URL}`;   // base URL, baked in at build time (chapter 15's env-per-build-target pattern)

public async loadItems(...): Promise<LoadResponse<TDTOModel>> {
  const token = getAuthToken();                            // reads the JWT out of client-side storage
  const url = `${this.apiURL}${this.apiPaths.servicePath}${this.apiPaths.loadItemsPath}`;
  response = await axios.post(url, request, {
    headers: { Authorization: this.noAuthToken ? "" : `Bearer ${token}` },  // manually attached, per-call
    signal,                                                  // AbortSignal — request cancellation, §10.6
  });
}
```
(`HRNS.Platform.UI/src/services/base/base-api.service.ts`, condensed.) `axios` is a `fetch`/`XMLHttpRequest` wrapper — under the hood, it's making exactly the same browser-native HTTP requests either of those APIs would, subject to exactly the same CORS/same-origin rules from §10.1–§10.2.

**The single most important, easy-to-miss detail here**: this platform authenticates with a **bearer token attached manually as an `Authorization` header on every call**, not with a cookie. This is a genuinely consequential architectural choice for CORS and browser security together, worth spelling out:

- **Cookie-based auth** requires the browser to *automatically* attach a stored cookie to requests — which only happens cross-origin if the request explicitly opts in (`credentials: "include"` for `fetch`, `withCredentials: true` for axios) *and* the server explicitly responds with `Access-Control-Allow-Credentials: true` *and*, critically, the server can no longer use `Access-Control-Allow-Origin: *` at all once credentials are involved — the spec forbids wildcard-origin-plus-credentials, for good reason: it would mean literally any website could make credentialed requests on your behalf. Cookie-based auth also means every request automatically carries your session, whether your own JavaScript intended to send it or not — the exact ambient-authority property that makes **CSRF** (Cross-Site Request Forgery, `../dotnet-mastery/` chapter 8 §8.7) a real threat: a malicious site can trigger a request to your API and the browser will happily attach your cookie, since it doesn't know the request "came from" an untrusted page.
- **This platform's approach** — `getAuthToken()` reading from client-side storage and the token being attached *explicitly, in application code*, on every call — means there's no ambient authority for a malicious third-party page to piggyback on: it would have to somehow read the token out of this app's storage first (a **XSS** vulnerability, not a CSRF one — a different, though equally serious, threat model, `../dotnet-mastery/` chapter 8 §8.7). This is exactly *why* `Program.cs`'s CORS policy above has no `AllowCredentials()` call at all, and why `axios.post` here never sets `withCredentials: true` — this platform's auth model simply doesn't need the credentialed-CORS machinery, because the token was never going to be sent automatically by the browser in the first place.

## 10.5 Cookies — attributes worth knowing even though this platform doesn't lean on them for auth

Since cookies still appear elsewhere on the web (and might in this platform for other purposes — e.g. a "remember the last selected company account" preference), the attributes worth recognizing on sight: `Secure` (only sent over HTTPS), `HttpOnly` (invisible to JavaScript entirely — `document.cookie` can't read it — a real, effective XSS mitigation *for that specific cookie*), `SameSite` (`Strict`/`Lax`/`None` — controls whether the cookie is sent on cross-site requests at all; `Lax` is most browsers' modern default, and is itself a meaningful CSRF mitigation independent of anything the application does), `Domain`/`Path` (scope the cookie to a specific host/path).

## 10.6 A few more browser-networking mechanics worth knowing

- **`AbortController`/`AbortSignal`** — the standard browser API for cancelling an in-flight `fetch` (or, as here, an axios request that's been wired to accept a `signal`) — e.g. if a user navigates away from a page before a slow request finishes, or triggers a new search before the previous one returned. `loadItems`'s `signal?: AbortSignal` parameter (§10.4) is exactly this.
- **Service workers** — a script the browser runs in the background, separate from any page, capable of intercepting *every* network request the page makes and deciding how to respond (serve from cache, go to network, or some combination) — the foundation of offline-capable PWAs (Progressive Web Apps) and sophisticated client-side caching strategies. Not currently something this platform's frontend uses, but worth recognizing the term and knowing it's a browser-networking-layer interception point, distinct from the HTTP caching covered in chapter 12.

## Checkpoint

1. Open your browser's Network tab against this platform's dev environment, trigger any data-loading action, and find the `OPTIONS` preflight request for it (if one fires — recall §10.2's "simple request" exception). What does `Access-Control-Request-Headers` list, and does it match what `Access-Control-Allow-Headers` in the actual response permits?
2. Explain, precisely, why `Access-Control-Allow-Origin: *` combined with `Access-Control-Allow-Credentials: true` is forbidden by the CORS spec — what attack would it enable if it were allowed?
3. If this platform switched from bearer-token-in-header auth to cookie-based session auth tomorrow, list every change that would ripple through: the CORS policy (`Program.cs`), the frontend's `BaseApiService` (`axios` config), and the threat model (`../dotnet-mastery/` chapter 8 §8.7) it would newly need to defend against that it currently doesn't.
