# 2. .NET Platform & Project Anatomy

Goal: understand what ".NET" actually is, how a solution/project is structured, how `dotnet run` turns source files into a running web server, and how configuration flows in — all mapped onto the real `HRNS.Platform.Server` solution.

## 2.1 What ".NET" means today

Historically confusing; here's the short version relevant to this repo:

- **.NET Framework** (2002–2019, Windows-only, versions up to 4.8) — legacy, not what this repo uses.
- **.NET Core → .NET 5+** (2016–now, cross-platform, open source) — this is "modern .NET." Versioning dropped "Core" from the name at .NET 5; it's just "**.NET 6/7/8/9/10**" now, roughly one major version per year.
- **HRNS.Platform.Server targets `net10.0`** (every `.csproj` has `<TargetFramework>net10.0</TargetFramework>`) — the current version at time of writing. .NET 8 is the most recent LTS (Long-Term Support, 3-year support window); odd-numbered releases like .NET 9 (and even numbers outside the LTS cadence) are STS (18 months). Production teams often pin to the LTS release; this repo is running ahead of that on purpose.

Three pieces work together, roughly analogous to Node.js's own moving parts:

| .NET piece | Rough Node.js analogue |
|---|---|
| **CLR** (Common Language Runtime) — executes compiled IL, does JIT compilation, GC | The V8 engine |
| **.NET SDK** — `dotnet` CLI, compilers, project templates | `node` + `npm` combined |
| **NuGet** — package registry + `PackageReference` in `.csproj` | npm registry + `package.json` |

## 2.2 The `.csproj` file — this repo's `package.json`

Every project (a `.csproj` file) declares its target framework, its NuGet dependencies, and references to sibling projects. Compare `HRNS.Application/HRNS.Application.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="MediatR" Version="12.2.0" />
    <PackageReference Include="FluentValidation" Version="11.9.1" />
    <!-- ... -->
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\HRNS.Common\HRNS.Common.csproj" />
    <ProjectReference Include="..\HRNS.Database\HRNS.Database.csproj" />
    <ProjectReference Include="..\HRNS.Models\HRNS.Models.csproj" />
  </ItemGroup>
</Project>
```

- `<PackageReference>` ≈ a line in `dependencies` of `package.json`. No `node_modules` — NuGet packages are cached globally (`~/.nuget/packages`) and referenced by version, resolved at build/restore time (`dotnet restore`, which `dotnet build` runs implicitly).
- `<ProjectReference>` ≈ a workspace/monorepo-local dependency (like a `"file:../other-package"` or a Lerna/Nx workspace reference) — it's how a multi-project **solution** wires its own internal layers together.
- `Sdk="Microsoft.NET.Sdk"` vs `Sdk="Microsoft.NET.Sdk.Web"` (used only by `HRNS.WebApi.csproj`) — the Web SDK pulls in the ASP.NET Core shared framework (Kestrel, MVC, routing, etc). Every other project in this solution is a plain class library — they know nothing about HTTP.

Two per-project quirks worth noticing because they change how code reads inside that project specifically:
- `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>` appear **only** in `HRNS.WebApi.csproj`. `ImplicitUsings` auto-imports common namespaces (`System`, `System.Linq`, `System.Threading.Tasks`, ...) so files in `HRNS.WebApi` often have few or no `using` statements at the top, unlike `HRNS.Application`/`HRNS.Database`, which spell every `using` out explicitly.
- `<InternalsVisibleTo>` in `HRNS.WebApi.csproj` grants `HRNS.Tests` access to `internal` (non-public) members — the .NET equivalent of exporting a "test-only" symbol.

## 2.3 Solution structure — how HRNS.Platform.Server's projects layer

A **solution** (`.sln`) is just a grouping of projects that build together — roughly a monorepo's root `package.json` workspaces list, minus any shared runtime behavior. The dependency graph between `<ProjectReference>`s **is** the architecture (see chapter 6 for the full picture); for now, just the map:

```
HRNS.WebApi            → depends on → HRNS.Application, HRNS.Database, HRNS.Models, HRNS.Web.CQRS
HRNS.Application        → depends on → HRNS.Common, HRNS.Database, HRNS.Models
HRNS.Database           → depends on → HRNS.Common, HRNS.Models
HRNS.Web.CQRS            (generic web request/response envelope types — depended on by WebApi)
HRNS.Common              (lowest-level shared code — depended on by nearly everything)
HRNS.Models              (DTOs/STOs — the "public API shape" — depended on by nearly everything)
Extensions/*            → four standalone class libraries of extension methods over
                            AutoMapper, EF Core, Newtonsoft.Json and System types
HRNS.Tests              → depends on everything, references nothing depends on it
```

.NET project references are **strictly acyclic** — the compiler physically will not let `HRNS.Database` reference `HRNS.Application` if `HRNS.Application` already references `HRNS.Database`. This is enforced at compile time, unlike a Node monorepo where a circular `import` between two packages will often silently work (or blow up at runtime) rather than fail the build. This is a real, structural guardrail against architecture erosion — internalize it, because it explains *why* things live where they do (chapter 6 goes deeper).

## 2.4 Entry point: `Program.cs`

Modern .NET project templates (`dotnet new webapi`) generate a **top-level statements** `Program.cs` — no class, no `Main` method, the file body itself is the entry point (syntactic sugar the compiler expands for you). HRNS.Platform.Server does **not** use that style — it uses the older, explicit form:

```csharp
namespace HRNS.WebApi
{
    public class Program
    {
        public static void Main(string[] args)
        {
            var webAppBuilder = WebApplication.CreateBuilder(args);
            // ... configure services, build the app, configure the request pipeline
            var app = webAppBuilder.Build();
            // ... app.Use...(), app.Map...()
            app.Run();
        }
    }
}
```
(`HRNS.WebApi/Program.cs:66-152`, condensed — full walkthrough in chapter 3.)

Both styles compile to the same thing; the explicit form is more common in larger/older codebases and is arguably easier to read when the method is doing a lot (as it is here — `ConfigureServices` alone is 230+ lines wiring up dozens of services).

`dotnet run` (or F5 in an IDE) does, in order: `dotnet restore` (resolve NuGet packages) → `dotnet build` (compile every project in dependency order) → execute the compiled entry point of `HRNS.WebApi`, which starts Kestrel (the built-in cross-platform web server — .NET's equivalent of Node's own HTTP server, or what Express sits on top of) listening on the configured port.

## 2.5 Configuration — `appsettings.json` and `IConfiguration`

.NET's built-in configuration system layers multiple **sources** into one `IConfiguration` object, later sources overriding earlier ones — conceptually similar to `dotenv` + a config library like `convict`, but built into the framework:

1. `appsettings.json`
2. `appsettings.{Environment}.json` (e.g. `appsettings.Development.json`) — environment picked from the `ASPNETCORE_ENVIRONMENT` env var, the .NET equivalent of `NODE_ENV`
3. Environment variables
4. User secrets (local dev only, stored outside the repo — see `Microsoft.Extensions.Configuration.UserSecrets` in `HRNS.Application.csproj`, used so nobody accidentally commits a real API key)
5. Command-line arguments

Real (secret values redacted) top-level shape of `HRNS.WebApi/appsettings.json`:

```json
{
  "Logging": { "PathFormat": "./Logs/HRNS.Api-{Date}.log", "LogLevel": { "Default": "Information" } },
  "ConnectionStrings": { /* PostgreSQL connection string */ },
  "Authentification": { "Jwt": { /* issuer, secret key, expiry */ } },
  "Mail": {},
  "FileStorage": {},
  "ReCaptcha": {},
  "ClamAv": {},
  "AIServer": { /* Ollama endpoint + model names, see chapter 13 */ },
  "AIAssistant": {},
  "Firebase": {}
}
```

Config values get consumed in two ways, both present in this repo:

**Manual read**, useful for one-off values or feature flags:
```csharp
var enabled = webAppBuilder.Configuration.GetValue<bool?>("Firebase:Enable") ?? false;
```
(`HRNS.WebApi/Program.cs:228` — exactly this pattern.)

**Strongly-typed binding** ("Options pattern"), preferred for anything with more than one or two fields — a whole JSON section gets deserialized into a C# class:
```csharp
services.ConfigureJwtToken(webAppBuilder.Configuration.GetSection("Authentification:Jwt"));
```
(`HRNS.WebApi/Program.cs:331` — binds the `Authentification:Jwt` JSON section into a `JwtConfiguration` object, consumed later by `JwtGenerator`, see chapter 8.)

## Checkpoint

1. Open `HRNS.Platform.Server.sln` (or just look at the six top-level project folders) and, without opening any code, write down which project you'd expect a brand-new "generate a PDF report" feature to touch, based purely on the dependency map above.
2. Find one more `Configuration.GetValue<...>` or `GetSection(...)` call in `Program.cs` besides the two shown here, and identify what `appsettings.json` key it reads.
3. Run `dotnet build` (or `dotnet build HRNS.Platform.Server.sln`) from the repo root and watch the project-by-project build order in the output — does it match the dependency map in §2.3?
