# 8. Auth & Security

Goal: JWT theory, ASP.NET Core's authentication/authorization split, and a full walkthrough of HRNS's actual login flow — including a couple of real, honest security observations worth learning to spot yourself.

## 8.1 Authentication vs. authorization — two different questions

- **Authentication (AuthN)**: *who* is making this request? Answered once, by validating a credential (a JWT, in this app).
- **Authorization (AuthZ)**: is *this authenticated identity* allowed to do *this specific thing*? Answered per-action, potentially many times per request.

ASP.NET Core's `[Authorize]` attribute (chapter 3 §3.2 — applied at the `BaseController` level, so it's "on" for every endpoint by default) only answers the first question: "is there a valid, signed, non-expired JWT on this request." It says nothing about *what* that user is allowed to do — that's a second, separate layer, and HRNS builds its own for that (§8.4) rather than relying on ASP.NET Core's built-in policy/role system.

## 8.2 JWT — the mechanics

A JSON Web Token is three base64url-encoded segments joined by dots: `header.payload.signature`. The payload is a set of **claims** (arbitrary key/value assertions — "this token was issued for user `abc123`, expires at time `T`"). The signature is a cryptographic guarantee that the payload hasn't been tampered with since the server issued it — computed over `header + payload` using a secret key. Anyone can *decode* (base64-decode) a JWT's payload and read it — a JWT is not encrypted, only signed — so **never put secrets in JWT claims**; put only identity/authorization-relevant, non-sensitive data there.

Compared to session-cookie auth (probably what you're more used to from Express + `express-session`): a JWT is **stateless** — the server verifies the signature and trusts the claims without a database lookup or server-side session store on every request. That's a genuine scalability win (no shared session store needed across multiple server instances) at the cost of **hard revocation**: you can't invalidate a single already-issued JWT before it expires without extra machinery (a blocklist, short expiries + refresh tokens, etc.) — the trade-off HRNS lives with as much as any JWT-based system does.

## 8.3 Wiring JWT auth into the ASP.NET Core pipeline

```csharp
services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidIssuer = jwtConfigSection["issuer"],
        ValidateAudience = false,
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(keySection)),
        RequireExpirationTime = true,
        ValidateLifetime = true,
        ClockSkew = TimeSpan.Zero
    };
});
```
(`HRNS.WebApi/Extensions/ServiceCollectionExtension.cs:24-67`, condensed — `services.ConfigureJwtToken(...)` is called from `Program.cs:331`.) `ClockSkew = TimeSpan.Zero` is a deliberate tightening — the default is a 5-minute grace period on token expiry to tolerate clock drift between servers; HRNS turns that off, meaning "expired" is enforced to the second.

**HRNS goes one step further and swaps out the default token handler:**

```csharp
options.TokenHandlers.Clear();
options.TokenHandlers.Add(new DynamicKeyJwtValidationHandler(jwtConfigSection));
```

`DynamicKeyJwtValidationHandler` implements `ISecurityTokenValidator` itself, with a `GetSecurityKeyForUserId(Guid claimedId)` method whose own comment tells you exactly why it exists:

```csharp
public SecurityKey GetSecurityKeyForUserId(Guid claimedId)
{
    // NB: later this can be replaced by per-user key stored in DB
    var securityKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(Configuration["jwtSecretKey"] ?? ""));
    return securityKey;
}
```
(`HRNS.WebApi/Middleware/DynamicKeyJwtValidationHandler.cs:39-46`.) Today it resolves to the same single shared secret key as the default handler would — but the *shape* of the code (looking up a key by `claimedId` before validating the token's signature against it) is infrastructure for **per-user or per-tenant signing keys** in the future, which would let the app revoke a single user's tokens instantly (rotate their key) without invalidating everyone else's — impossible with one shared secret. Worth remembering as a pattern: sometimes code is more general than its current behavior strictly requires, on purpose, because the extension point is already known to be coming.

## 8.4 HRNS's own authorization layer — `AccessDomainIds` and `CurrentUserAuthorizations`

This is the layer that actually answers "is this user allowed to do *this*," built entirely inside `BaseHandler<TRequest, TResponse>` (chapter 5 §5.4), independent of ASP.NET Core's `[Authorize(Roles = "...")]`/policy system.

Every concrete handler declares, in its constructor, which "access domains" are allowed to invoke it:

```csharp
// LoginUserQueryHandler — reachable by anyone, since nobody's authenticated yet
AccessDomainIds = new Guid[] { Constants.Authorizations.AnonymousUser };

// UpsertTicketingTicketTypeCommandHandler — any authenticated user (fine-grained
// per-contract authorization is then enforced separately, in PreHandle — chapter 5 §5.5)
AccessDomainIds = new Guid[] { Constants.Authorizations.AuthenticatedUser };

// CommandBaseHandler's own default, inherited unless overridden
AccessDomainIds = new Guid[] { Constants.Authorizations.NoAccess };
```

`BaseHandler.Handle()` (chapter 5 §5.4) calls `LoadUserAuthorizations()` then `CheckUserAuthorization()` before ever reaching `DoHandle()`. `LoadUserAuthorizations` is worth reading once for a real bootstrap trick:

```csharp
// If there is only one user, consider it is setup mode thus allow system and providers setup options.
var isSetupMode = _dbContext.Set<UserEntity>().AsNoTracking().Where(e => e.Id != Constants.SystemUserId).Count() == 1;
CurrentUserAuthorizations.IsAdmin = isSetupMode;
```
(`HRNS.Application/CQRS/Base/BaseHandler.cs:262-263`.) On a brand-new deployment with exactly one real user (the one who just registered), that user is automatically treated as admin — solving the classic chicken-and-egg problem of "who grants the first admin their admin rights" without a separate seed script or manual DB edit. Once a second user exists, this heuristic stops applying and normal role-based checks take over.

`CheckUserAuthorization()` then compares the current user's actual authorizations against the handler's declared `AccessDomainIds`, throwing `UnauthorizedAccessException` (converted to an HTTP error response by `HRNSExceptionMiddleware`, chapter 10) if there's no overlap. This two-layer model — ASP.NET Core `[Authorize]` gatekeeping *authentication* at the pipeline level, `AccessDomainIds`/`CheckUserAuthorization` gatekeeping *authorization* at the handler level, with a further, feature-specific layer possible in `PreHandle` (chapter 5 §5.5) — is worth comparing against the built-in ASP.NET Core policy-based authorization system (`[Authorize(Policy = "...")]`) you'll read about in official docs: HRNS's approach is a hand-rolled equivalent, chosen because authorization here needs to reach deep into domain data (which legal entity, which contract, which company account) that a simple attribute on a controller method can't express.

## 8.5 The login flow — `LoginUserQuery`, worth reading start to finish

This single handler (`HRNS.Application/CQRS/Users/LoginUserQuery.cs`, chapter 5 §5.5 already showed the shape of a *thin* handler — this one is the opposite: almost the entire feature is custom `DoHandle` logic) demonstrates several real security patterns at once:

**Account lockout / brute-force throttling:**
```csharp
var lockMinutes = 15; var loginAttemptsMinutes = 1; var maxLoginAttempts = 5;
if (!_encryptor.VerifyPasswordHash(password, userEntity.PasswordHash, userEntity.PasswordSalt))
{
    Thread.Sleep(5000); // slow down brute-force attempts
    // ... increment LoginAttempts; if >= maxLoginAttempts within loginAttemptsMinutes, set LockedUntilUtc
}
```
(`LoginUserQuery.cs:116-155`, condensed.) `Thread.Sleep(5000)` on every failed attempt is a real, deliberate anti-brute-force measure — but it's a **blocking, synchronous sleep** on a request-handling thread, not an `await Task.Delay(...)`. Under real attack traffic (many concurrent failed logins), this ties up thread-pool threads for 5 seconds each rather than yielding them back to the pool — worth noticing as a concrete example of "async all the way" (chapter 1 §1.5, chapter 12) not being followed consistently, and a legitimate thing to flag/fix if you were asked to harden this code.

**Two-factor authentication**: if the user's `TwoFactorAuthMethodId` isn't "none," a random 6-digit code is generated, emailed via a templated notification, and a `TwoFACodeEntity` is stored for later verification (a separate `Confirm2FaLoginQuery` handler checks it) — the JWT is only issued once 2FA passes, not at initial password verification.

**Password verification** — `Encryptor.VerifyPasswordHash` (`HRNS.Application/Implementation/Utils/Encryptor.cs:37-64`):
```csharp
using (var hmac = new HMACSHA512(storedSalt))
{
    var computedHash = hmac.ComputeHash(Encoding.UTF8.GetBytes(password));
    for (int i = 0; i < computedHash.Length; i++)
        if (computedHash[i] != storedHash[i]) return false;
}
return true;
```
Two things worth learning to spot here, in the honest spirit of "read real code critically, not just admiringly": (1) `HMACSHA512` with a random per-user salt is a legitimate way to avoid rainbow-table attacks, but it's a *fast* hash — the code's own comment on the hashing side (`CreatePasswordHash`) says `"// or maybe use Microsoft.AspNetCore.Cryptography.KeyDerivation to make it slower"`, i.e. the author already knew a deliberately-slow key-derivation function (PBKDF2/bcrypt/Argon2 — functions designed to make brute-forcing a stolen hash database expensive) is the more defensible modern choice than a single fast HMAC round; (2) the byte-by-byte comparison loop returns as soon as it finds a mismatch, which is a **non-constant-time comparison** — in principle a timing side-channel (an attacker measuring response-time differences could theoretically infer how many leading bytes of a guessed hash matched). The fix in real .NET code is `CryptographicOperations.FixedTimeEquals(a, b)`, built for exactly this. Neither of these is necessarily an *exploitable* issue in this specific app's threat model — but recognizing them on sight is a genuinely useful security-review skill, and this file is a good place to practice it.

## 8.6 Authenticated-encryption example — AES-GCM for QR codes

Away from passwords entirely, `Encryptor.GenerateLocationQRCode`/`VerifyLocationQRCode` (used for TnA — Time & Attendance — clock-in QR codes, chapter 14) use `AesGcm`, an **authenticated encryption** mode: it doesn't just encrypt, it also produces a tag that proves the ciphertext wasn't tampered with (the modern, correct choice — vs. plain AES-CBC with no integrity check, which is vulnerable to padding-oracle-style tampering). Good example to hold onto: "encrypt" alone is rarely the right primitive; "encrypt + authenticate" (AES-GCM, or ChaCha20-Poly1305) almost always is.

## 8.7 OWASP-for-.NET checklist, mapped to what's actually in this repo

| OWASP concern | How ASP.NET Core / EF Core / this repo handles it |
|---|---|
| SQL injection | EF Core parameterizes every LINQ-translated query automatically — you'd have to go out of your way (raw SQL string concatenation) to reintroduce this |
| XSS | N/A at this layer — this is a JSON API; output-encoding responsibility lives in `HRNS.Platform.UI` |
| CSRF | Lower risk for bearer-token APIs (no ambient cookie sent automatically cross-site) vs. cookie-session auth — still worth confirming the UI never stores the JWT somewhere CSRF-adjacent (e.g. an auto-sent cookie) |
| Secrets management | `appsettings.json` for non-secret config, environment variables / user secrets for real secrets (chapter 2 §2.5) — **never commit a real `jwtSecretKey` or connection string to git** |
| Password storage | HMACSHA512 + per-user salt (§8.5) — functionally sound, arguably below current best practice (a slow KDF would be stronger) |
| Broken auth / brute force | Account lockout + attempt throttling (§8.5) |
| Transport security | `RequireHttpsMetadata = false` in the JWT bearer setup (§8.3) looks alarming out of context — in practice this just means "don't require HTTPS metadata discovery," and real TLS termination happens upstream (load balancer/reverse proxy), a completely standard deployment shape (chapter 11) — but it's exactly the kind of line worth asking about explicitly rather than assuming, on any codebase you're new to |
| Least privilege | The `AccessDomainIds` + `PreHandle` two-layer authorization model (§8.4, chapter 5 §5.5) |

## Checkpoint

1. Read `Confirm2FaLoginQuery.cs` end to end — what does it check, and what does it do differently from `LoginUserQuery` once the code matches?
2. Find one more handler that sets `AccessDomainIds = new Guid[] { Constants.Authorizations.AnonymousUser }` besides login. Why would that specific feature also need to be reachable pre-authentication?
3. If you were asked to fix the non-constant-time comparison in `Encryptor.VerifyPasswordHash` (§8.5), what one-line change would you make, and would it change the method's behavior for any legitimate caller?
