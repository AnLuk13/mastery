# 11. DevOps, CI/CD & Deployment

Goal: how a .NET web app actually gets built, containerized, and deployed — general theory, since (worth saying plainly) **this repo, as checked out, has no committed `Dockerfile` or CI pipeline config** (`.gitlab-ci.yml`/`.github/workflows/`) at the time of writing — deployment is handled outside this repository (GitLab is the remote, per the project's git configuration, but pipeline definitions aren't part of this codebase). What *is* real and worth knowing about first:

## 11.1 What's actually in this repo

- **`.config/dotnet-tools.json`** — a local .NET tool manifest (the `.csproj`/`package.json`-adjacent equivalent of a `devDependencies` pinning file for CLI tools):
  ```json
  { "tools": { "csharpier": { "version": "0.23.0" }, "dotnet-ef": { "version": "10.0.8" } } }
  ```
  `csharpier` is a Prettier-equivalent opinionated code formatter for C#; `dotnet-ef` is the EF Core CLI (chapter 4 §4.8's migration commands). Anyone cloning the repo runs `dotnet tool restore` once to install exactly these pinned tool versions locally — the same idea as `npm install` pulling in versions pinned by a lockfile, scoped to development tools rather than app dependencies.
- **`Data/changelog.db`** — a SQLite file shipped as `Content` in `HRNS.WebApi.csproj`, copied to the output/publish directory — likely an embedded, file-based changelog/version-history store bundled with the deployed app rather than requiring its own database connection.
- **Migrations-on-boot** (chapter 4 §4.8) — `app.ApplyDBContextMigrations()` — is this app's actual "how does the schema get to production" mechanism, in lieu of a separate migration deploy step.

Everything else in this chapter is general .NET deployment knowledge — genuinely useful for understanding how an app like this *would* be built and shipped, and directly transferable if/when a Dockerfile or pipeline does get added to this repo (or to any other .NET project you touch).

## 11.2 Building for deployment: `dotnet publish`

`dotnet build` compiles for local development/testing. `dotnet publish` produces a self-contained, deployable output folder:

```bash
dotnet publish HRNS.WebApi/HRNS.WebApi.csproj -c Release -o ./publish
```

- `-c Release` — the Release build configuration (optimizations on, debug symbols minimal) vs. `Debug` (the default for `dotnet build`/`dotnet run`) — same distinction as a Node app's `NODE_ENV=production` build.
- The output is a folder of compiled DLLs, an entry-point executable, and static content (`appsettings.json`, the `Data/changelog.db` file above, anything else marked `CopyToOutputDirectory`) — everything needed to run the app on a machine with the matching .NET runtime installed, without the SDK or source code.
- **Framework-dependent** (default — needs the .NET runtime pre-installed on the target machine, smaller output) vs. **self-contained** (`--self-contained true -r linux-x64`, bundles the runtime itself, larger output, zero-install-dependency deploy) — the second is closer to what a Docker image typically does implicitly, since the base image already provides the runtime layer.

## 11.3 Docker for .NET — the pattern this repo would use if it had a Dockerfile

.NET's official Docker images split into two roles, and a real Dockerfile for an app like this would use both in a **multi-stage build** — compile in one throwaway image (which has the full SDK), copy just the compiled output into a second, much smaller runtime-only image:

```dockerfile
# Stage 1: build — needs the full SDK (compilers, NuGet, etc.)
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY . .
RUN dotnet restore HRNS.WebApi/HRNS.WebApi.csproj
RUN dotnet publish HRNS.WebApi/HRNS.WebApi.csproj -c Release -o /app/publish

# Stage 2: runtime — only the ASP.NET Core runtime, no SDK/compilers, much smaller image
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "HRNS.WebApi.dll"]
```

Why two stages: the SDK image is large (compilers, MSBuild, NuGet caches); nothing in it is needed to *run* an already-compiled app, only to *build* one. Shipping only the `aspnet` runtime image in stage 2 keeps the final image significantly smaller and reduces its attack surface (fewer tools an attacker could abuse if they got a shell in the container). This is the direct .NET equivalent of a Node multi-stage Dockerfile that `npm ci && npm run build` in one stage and copies only `dist/` + `node_modules --production` into a slim runtime stage.

Configuration inside a container follows the same layered `IConfiguration` model from chapter 2 §2.5 — environment variables are the natural way to inject `ConnectionStrings__PGSQL` or `Authentification__Jwt__jwtSecretKey` (double-underscore `__` is .NET config's env-var equivalent of the `:` section separator, since most shells don't allow `:` in env var names) into a container without baking secrets into the image.

## 11.4 CI/CD pipeline shape (generic, GitLab-flavored since that's this project's remote)

A typical pipeline for a .NET web API, regardless of which CI system runs it, has the same stages:

```yaml
# illustrative — not a file that exists in this repo
stages: [restore, build, test, publish, deploy]

restore:
  script: dotnet restore

build:
  script: dotnet build --no-restore -c Release

test:
  script: dotnet test --no-build -c Release --logger "junit" # HRNS.Tests runs here (chapter 9)

publish:
  script: dotnet publish HRNS.WebApi -c Release -o ./publish
  artifacts: { paths: [publish/] }

deploy:
  script: docker build -t registry.example.com/hrns-platform-server:$CI_COMMIT_SHA . && docker push ...
```

The critical property worth internalizing regardless of which specific tool runs this: **the exact same `dotnet build`/`dotnet test`/`dotnet publish` commands you'd run locally are what CI runs** — there's no CI-specific build magic, which is exactly why "it builds and tests clean locally" is a meaningful, trustworthy signal before you ever push.

## 11.5 Migrations in a real deployment pipeline — the trade-off from chapter 4, revisited

Chapter 4 §4.8 flagged `app.ApplyDBContextMigrations()` (migrate-on-boot) as a real architectural choice worth having an opinion on. Now that you've seen a fuller deployment picture, here's the actual trade-off spelled out:

- **Migrate-on-boot (what this repo does)**: simplest possible deploy — start the new app version, it brings the schema with it. Real risk: if you run *multiple instances* of the app (horizontal scaling, or a rolling deploy with old+new instances briefly both running), every instance attempts the migration on startup — EF Core's migration history table normally makes concurrent attempts safe (only one wins, others see "already applied" and continue), but it's still a mechanism worth understanding rather than assuming is risk-free, especially for a migration that locks a large table for a long time while multiple instances are hammering the same schema mid-rollout.
- **Separate migration step (a common alternative)**: a dedicated CI/CD pipeline stage runs `dotnet ef database update` (or the published app's `--migrate-only` equivalent) *before* any new app instance starts serving traffic, as an explicit, observable, individually-retryable deploy step. More moving parts, but a bad migration fails the deploy cleanly instead of potentially half-succeeding across multiple booting instances.

Neither is objectively "correct" — it's a genuine trade-off between deploy simplicity and deploy safety at scale, and worth being able to articulate both sides (this is precisely the kind of question that comes up in real engineering discussions about a codebase like this one).

## 11.6 Environment-specific configuration in deployment

Recall chapter 2 §2.5's config layering (`appsettings.json` → `appsettings.{Environment}.json` → env vars → ...). In a real deployment pipeline, this typically maps to:

| Environment | `ASPNETCORE_ENVIRONMENT` | Where secrets/config actually come from |
|---|---|---|
| Local dev | `Development` | `appsettings.Development.json` + user secrets (chapter 2 §2.5) |
| CI test run | `Testing` (used explicitly by `TestWebApplicationFactory`, chapter 9 §9.5) | in-memory config collection, no real secrets needed at all |
| Staging/Production | `Production` (or a custom name) | environment variables / a secret manager (cloud provider secret store, Kubernetes Secrets, etc.) — never `appsettings.json` committed to git |

## Checkpoint

1. If you were asked to add a `Dockerfile` to this repo, which `.csproj` would be your publish target, and which other projects would `dotnet publish` need to be able to reach via `<ProjectReference>` to succeed (hint: revisit chapter 2 §2.3's dependency graph)?
2. Run `dotnet tool restore` followed by `dotnet csharpier --check .` (or just `dotnet csharpier .` to auto-format) from the repo root — what, if anything, does it flag?
3. Write out, in your own words, one concrete failure scenario for migrate-on-boot under a rolling deploy with two instances running simultaneously (§11.5) — and one concrete failure scenario for a separate migration step instead. Which would you pick for a schema change that adds a new nullable column vs. one that renames an existing column?
