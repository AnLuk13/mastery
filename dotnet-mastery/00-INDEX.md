# .NET & Web Development Mastery — grounded in HRNS.Platform.Server

This is a self-paced curriculum built for one specific goal: take you from "can read C#, harder to write it, fuzzy on ASP.NET Core/EF Core/DI/CQRS" to genuinely comfortable reading, reasoning about, and extending `HRNS.Platform.Server` — and by extension, most production .NET web backends.

You already know how a web backend *should* behave — you've built them with Nest, Node and Express. That's a real advantage: almost everything here has a direct analogue in the Node/TS world, and each chapter leans on that instead of pretending you're starting from zero. Where .NET does something genuinely different (strong static typing throughout, EF Core's change-tracking, MediatR's mediator pattern), that's called out explicitly.

## How this is organized

Every chapter follows the same shape:
1. **Theory** — the concept explained from first principles, with a Node/TS analogy where one exists.
2. **Minimal runnable example** — small, generic, copy-pasteable, unrelated to HRNS, so the idea is clear in isolation.
3. **In HRNS.Platform.Server** — the same concept, found in the real codebase, with file paths and real (trimmed) code so you can jump in and read it yourself.
4. **Checkpoint** — a few questions or a small exercise to do against the real repo before moving on.

Nothing here is fabricated filler — every "In HRNS.Platform.Server" section quotes code that actually exists in this repo as of 2026-08. File paths are relative to `HRNS.Platform.Server/`.

## Suggested path

**Phase 1 — Language & platform refresh** (you said you need this most)
1. [C# Refresher](01-csharp-refresher.md) — OOP, generics, LINQ, async/await, nullable reference types, records, pattern matching
2. [.NET Platform & Project Anatomy](02-dotnet-platform-and-tooling.md) — SDK/runtime, csproj, NuGet, solution/project layout, how `dotnet run` actually starts a server

**Phase 2 — The web framework**
3. [ASP.NET Core Fundamentals & Dependency Injection](03-aspnetcore-fundamentals-and-di.md) — the middleware pipeline, routing, controllers, the built-in DI container and its lifetimes
4. [Entity Framework Core](04-entity-framework-core.md) — DbContext, migrations, LINQ-to-SQL, relationships, change tracking, query filters

**Phase 3 — Architecture (this is where HRNS gets its own shape)**
5. [CQRS & MediatR](05-cqrs-and-mediatr.md) — the mediator pattern, then HRNS's own generic Query/Command framework layered on top of it — the single most important chapter for reading this codebase
6. [Architecture & Design Patterns](06-architecture-and-design-patterns.md) — layered/clean architecture, SOLID, how HRNS's six projects map to it
7. [Validation & Mapping](07-validation-and-mapping.md) — FluentValidation, AutoMapper, and how HRNS wires both into the CQRS pipeline

**Phase 4 — Cross-cutting concerns**
8. [Auth & Security](08-auth-and-security.md) — JWT deep dive, ASP.NET Core authn/authz, HRNS's login/2FA flow, OWASP for .NET
9. [Testing](09-testing.md) — xUnit, NSubstitute, FluentAssertions, unit vs. integration tests, `HRNS.Tests` walkthrough
10. [Background Jobs, Real-Time & Observability](10-background-jobs-realtime-observability.md) — `IHostedService`, SignalR, Serilog, health checks, resilience
11. [DevOps, CI/CD & Deployment](11-devops-cicd-deployment.md) — Docker for .NET, environment config, migrations in production, CI pipelines

**Phase 5 — Going deeper**
12. [Performance & Scaling](12-performance-and-scaling.md) — async best practices, the N+1 problem, caching, EF Core performance
13. [AI/ML Integrations](13-ai-ml-integrations.md) — bonus chapter: how HRNS uses ML.NET, LangChain+Ollama and pgvector for its AI Assistant — genuinely unusual for an enterprise HR platform
14. [Guided Tour of HRNS.Platform.Server](14-platform-server-guided-tour.md) — folder-by-folder map, plus exercises: trace a request end-to-end using GitNexus/Graphify

[Glossary](GLOSSARY.md) — every acronym and term used above, one line each.

## Ground rules for using this guide

- Don't read it back-to-back like a novel. Read a chapter, then go open the real files it references, in this repo, side by side.
- Chapter 5 (CQRS & MediatR) is the payoff chapter. Everything in 1–4 exists to make chapter 5 make sense the first time you read it, instead of the fifth.
- Use `graphify query "<concept>"` and `gitnexus_context({name: "..."})` (see the project's `CLAUDE.md`) whenever a chapter says "go explore this yourself" — that's not a throwaway line, the tooling is genuinely faster than grep for this codebase.
- This repo runs **.NET 10** (bleeding-edge as of writing) with C# preview-adjacent language features enabled by default (`ImplicitUsings`, `Nullable` in `HRNS.WebApi`). Official docs for .NET 8 LTS concepts still apply almost 1:1.

## Tech stack this guide is calibrated to (confirmed from the real `.csproj` files)

| Concern | Library | Chapter |
|---|---|---|
| Web framework | ASP.NET Core 10 (MVC controllers, not Minimal APIs) | 3 |
| ORM | EF Core 10 + Npgsql (PostgreSQL) + Pgvector | 4 |
| Mediator / CQRS | MediatR 12 | 5 |
| Validation | FluentValidation 11 | 7 |
| Object mapping | AutoMapper 16 | 7 |
| Auth | ASP.NET Core JWT Bearer + custom claims/handlers | 8 |
| Testing | xUnit + FluentAssertions + NSubstitute + EF Core InMemory/SQLite | 9 |
| Logging | Serilog (console, file sinks) | 10 |
| Real-time | SignalR | 10 |
| API docs | Swashbuckle (Swagger/OpenAPI) | 3 |
| Email | MailKit/MimeKit, SendGrid | 10 |
| AI | Microsoft.Extensions.AI, LangChain.Core, OllamaSharp, ML.NET | 13 |
| Misc | ClamAV.Net (malware scan), Tesseract (OCR), Magick.NET/SkiaSharp (images), PdfPig/PdfSharpCore (PDF) | 14 |
