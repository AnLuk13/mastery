# 14. Guided Tour of HRNS.Platform.Server

This chapter is the reference map plus hands-on exercises — less "new theory," more "here's where everything lives, and here's how to actually navigate a codebase this size using the graph tooling instead of blind grepping."

## 14.1 Full project map

```
HRNS.Platform.Server/
├── HRNS.Common/              lowest layer — InternalRequest<T>, base CQRS message types (chapter 5 §5.3)
├── HRNS.Models/               DTOs/STOs — the "shape" every feature exposes (chapter 5 §5.3, chapter 7)
│   └── <FeatureArea>/           one folder per business domain (Recruiting, Payroll, TnA, Ticketing, ...)
├── HRNS.Database/              EF Core: DbContext, migrations, entities (chapter 4)
│   ├── DbContext/                 HRNSDbContext, connection provider, DI registration
│   ├── Entities/                  one folder per domain; Entities/Entity/ holds the shared base classes
│   ├── Migrations/                the full schema history (chapter 4 §4.8)
│   ├── Seed/                       IEntityTypeConfiguration-based data seeding (chapter 4 §4.2)
│   └── Utils/
├── HRNS.Application/            CQRS handlers + application services (chapter 5, 6, 7, 8, 13)
│   ├── CQRS/Base/                   QueryBase, CommandBase, BaseHandler — read chapter 5 before anything else here
│   ├── CQRS/<FeatureArea>/            one folder per domain, mirrors HRNS.Models/HRNS.Database structure
│   ├── Interfaces/ + Implementation/    dependency-inverted services (chapter 6 §6.2): email, JWT, encryption, AI, OCR, ...
│   └── Common/Exceptions/               HRNSException and friends (chapter 10 §10.5)
├── HRNS.Web.CQRS/                 generic web request/response envelopes controllers speak in (chapter 5 §5.5)
├── HRNS.WebApi/                     presentation + composition root (chapters 2, 3, 10, 11)
│   ├── Program.cs                     THE file to re-read after every chapter — it references almost everything
│   ├── Areas/                          controllers, one folder per feature area; BaseController.cs is the shared base
│   ├── Areas/*Hub*.cs                    SignalR hubs (chapter 10 §10.2)
│   ├── BackgroundServices/                IHostedService jobs (chapter 10 §10.1)
│   ├── HealthChecks/                       IHealthCheck implementations (chapter 10 §10.4)
│   ├── Middleware/                          exception handling, JWT token handler (chapters 8, 10)
│   └── Extensions/                           ServiceCollectionExtension / ApplicationBuilderExtension (chapter 3)
├── Extensions/                       four standalone extension-method libraries (chapter 1 §1.8, chapter 6 §6.2)
│   ├── System.Extensions/               generic .NET type extensions — ObjectExtensions.IsNull(), collection helpers, etc.
│   ├── Microsoft.EntityFrameworkCore.Extensions/    EF Core-specific helpers
│   ├── AutoMapper.Extensions/              AutoMapper-specific helpers
│   └── Newtonsoft.Json.Extensions/          JSON-specific helpers
├── HRNS.Tests/                       xUnit + FluentAssertions + NSubstitute + WebApplicationFactory (chapter 9)
├── docs/user-guides/                   END-USER functional documentation (not this guide — one file per feature, worth
│                                          reading alongside the code for a feature to understand its business purpose)
└── docs/dotnet-mastery/                  you are here
```

## 14.2 Feature areas you haven't seen code from yet — what to expect

Every domain folder under `HRNS.Models/`, `HRNS.Database/Entities/`, `HRNS.Application/CQRS/`, and `HRNS.WebApi/Areas/` follows the same shape you learned in chapters 4, 5 and 7. A non-exhaustive list of the business domains this platform covers, to give you a sense of scale: Employees, Recruiting, Payroll, Time & Attendance (TnA), Absences, Expenses, Documents & Document Templates, Document Signing (DocumentToSign), Policies, Legal Entities, Company Accounts, Shareholders, Power of Attorneys, PnL Reports, Performance Reviews, Employee Surveys, Ticketing (a support-desk subsystem), Mail Dispatching, Alert Line (compliance reporting), AI Assistant/Knowledge (chapter 13), Clarifications, Input Data Gathering. `docs/user-guides/` has one Markdown file per feature written for end users — genuinely useful to skim before diving into a feature's code, since it tells you *what the feature is for* before you spend time on *how it's built*.

## 14.3 Libraries you'll encounter that no chapter covered in depth

These showed up in the `.csproj` tech-stack table (chapter 0) but didn't get a dedicated walkthrough — recognize them by name and general purpose when you hit them, then read the specific usage in context:

| Library | Purpose | Where you'll meet it |
|---|---|---|
| `ClamAV.Net` | Talks to a ClamAV daemon over TCP to scan uploaded files for malware | File upload handlers; `ClamAvScannerService`, `ClamAvHealthCheck` (chapter 10 §10.4) |
| `Tesseract` | OCR (optical character recognition) — extracts text from scanned document images | `OcrService`, used by document processing/knowledge ingestion |
| `Magick.NET` / `SkiaSharp` / `Svg`/`Svg.Skia` | Image manipulation, resizing, format conversion, SVG rendering | Document/avatar/report image processing |
| `PdfPig` / `PdfSharpCore` / `DocumentFormat.OpenXml` | Read PDFs, generate PDFs, read/write Word (.docx) documents | Report generation, document templates, `WordTemplateService` |
| `ClosedXML` | Generate/read Excel (.xlsx) files without Excel installed | Report exports |
| `FuzzySharp` | Fuzzy string matching (edit-distance-based) | Likely deduplication/matching logic (e.g. matching a scanned document to an existing record despite typos) |
| `UAParser` | Parses a `User-Agent` header into browser/OS/device info | Login history (`UserLoginHistoryEntity`, chapter 8 §8.5) — recording *what* device logged in |
| `System.Linq.Dynamic.Core` | Build LINQ `OrderBy`/`Where` clauses from a *string* column name at runtime, not a compile-time lambda | `Sort.OrderBySorts()` (`HRNS.Models/Sort/Sort.cs`) — this is exactly how `QueryBaseHandler.OrderItems()` (chapter 5 §5.4) can sort by a caller-supplied, dynamic column name from an API request body instead of a fixed, hardcoded `.OrderBy(e => e.SomeProperty)` |
| `GoogleAuthenticatorService.Core` | TOTP (Time-based One-Time Password) generation/verification | The "Google/Microsoft Authenticator app" 2FA method mentioned in chapter 8 §8.5's login flow, alongside email-code 2FA |
| `Google.Cloud.RecaptchaEnterprise.V1` | Bot/abuse detection on forms | `RecaptchaHealthCheck` (chapter 10 §10.4); likely gates registration/login |

## 14.4 Practical exercise: trace one feature end-to-end using the graph tools, not grep

Pick any feature you're curious about — something you haven't read code for yet. Do this instead of `grep`-hunting file by file:

1. **`graphify query "<feature name>"`** (or the `graphify` MCP tool) — get the community/cluster this feature belongs to and its cross-file relationships. Read `graphify-out/GRAPH_REPORT.md`'s community list first if you're not sure what to search for.
2. **`gitnexus_query({ query: "<feature name>" })`** — get the ranked, process-grouped execution flow. This should surface the Controller → Command/Query → Handler → Entity chain as one connected result, the same quartet chapter 5 §5.5 walked through by hand for ticketing.
3. **`gitnexus_context({ name: "<the handler class name>" })`** — full 360° view: every caller, every callee, which execution flows it participates in.
4. **Before editing anything in that feature**, `gitnexus_impact({ target: "<symbol>", direction: "upstream" })` — this project's `CLAUDE.md` requires this before any edit, and it's genuinely useful practice regardless: it tells you the blast radius before you touch anything.

If steps 1–3 don't get you to a working mental model of the feature faster than reading files cold would have, that's worth noticing too — the graph is a map, not a replacement for reading the actual code once you know where to look.

## 14.5 Practical exercise: build a new CRUD feature from scratch

Synthesize chapters 4–7 by actually doing this (a scratch branch, not something you need to merge) — the fastest way to know you've really internalized the pattern rather than just recognized it on the page:

1. **Entity** (chapter 4 §4.3): a new `Entity` subclass + `EntityDbConfig<T>` (inherit the base, add your own relationships/constraints) in `HRNS.Database/Entities/<YourFeature>/`.
2. **Migration** (chapter 4 §4.8): `dotnet ef migrations add Add<YourFeature>` from `HRNS.Database/`.
3. **DTO + STO** (chapter 5 §5.3): two plain classes inheriting `DTOModel`/`STOModel` in `HRNS.Models/<YourFeature>/`.
4. **Mapping profile** (chapter 7 §7.2): inherit `EntityMappingProfile`, add `CreateMap<>` calls for anything not covered by the inherited base mappings.
5. **Query + Command** (chapter 5 §5.6): `GetYourFeatureQuery : QueryBase<...>` / `UpsertYourFeatureCommand : CommandBase<...>`, each with its nested `...Handler : QueryBaseHandler<...>`/`CommandBaseHandler<...>` — start with **zero overrides** and see how much already works.
6. **Controller** (chapter 3 §3.2): two-line actions calling `LoadDTOsAsync<...>`/`SaveSTOsAsync<...>`.
7. Now open `/api/docs` (chapter 3 §3.4) and confirm your new endpoints are there, then hit them (Swagger UI's "Try it out," or a REST client) against a real (or InMemory-seeded, chapter 9) database.

If you get stuck at any step, that's the exact chapter to re-read — this exercise is deliberately built to fail loudly at the specific gap in your understanding rather than vaguely.

## 14.6 What's outside this guide's scope

- **`HRNS.Platform.UI`** — the React/TypeScript frontend consuming this API. You already know frontend deeply; if you want, a natural follow-up exercise is picking one feature and reading *both* sides — the UI's API client call, and the controller action it hits — to see the full contract in both directions.
- **`HRNS.Docs`**, **`HRNS.Gelocation.Server`** — sibling repositories in this workspace, not covered here.
- **Kubernetes/cloud infrastructure specifics** — chapter 11 covered general .NET deployment theory; this repo has no committed infrastructure-as-code to study directly (chapter 11 §11.1).

## Checkpoint (final, cumulative)

1. Complete the CRUD exercise in §14.5 end to end, once, even for a throwaway feature.
2. Pick one library from §14.3 you're least familiar with and read one real usage site in this codebase.
3. Using `graphify path "<A>" "<B>"` (or `gitnexus_context` chained by hand), find the connection between two features that seem unrelated at first glance (e.g. "TnA clock-in" and "Payroll") — what's the actual dependency, and does it match what you'd have guessed from the business domain alone?
