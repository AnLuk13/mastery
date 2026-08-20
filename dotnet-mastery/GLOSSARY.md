# Glossary

Every acronym and term used across this guide, one line each. Chapter references point to where the term is explained in depth.

**ACL** — Access Control List; per-record permission checks (ch. 5, 6, 8).
**AutoMapper** — object-to-object mapping library (ch. 7).
**Bearer token** — an auth token (here, a JWT) sent in the `Authorization: Bearer <token>` HTTP header (ch. 8).
**BackgroundService** — .NET base class for long-running background work outside the request/response cycle (ch. 10).
**CLR** — Common Language Runtime; .NET's execution engine (ch. 2).
**Cascade / Restrict (DeleteBehavior)** — what happens to a dependent row when its referenced row is deleted (ch. 4).
**Change tracking** — EF Core's mechanism for detecting which loaded entities were modified, to generate `UPDATE` SQL (ch. 4).
**Claims / ClaimsPrincipal** — key/value assertions about an authenticated identity, carried in a JWT (ch. 8).
**Cosine similarity** — a measure of how similar two vectors' directions are, used in semantic/vector search (ch. 13).
**CQRS** — Command Query Responsibility Segregation; separating reads (Queries) from writes (Commands) (ch. 5).
**Csproj** — the XML project file (`.csproj`) declaring a .NET project's dependencies and settings (ch. 2).
**DbContext** — EF Core's Unit-of-Work object; one instance per request tracks and commits changes (ch. 4).
**DbSet\<T\>** — a queryable, table-like view of one entity type on a `DbContext` (ch. 4).
**Deferred execution** — LINQ queries build a pipeline lazily; nothing runs until enumerated (ch. 1, 4).
**Dependency Injection (DI)** — a container that constructs and hands out objects based on registered interface→implementation mappings (ch. 3).
**DTO** — Data Transfer Object; the read-side shape returned by a Query (ch. 5).
**Embedding** — a numeric vector representing the meaning of a piece of text, used for semantic search (ch. 13).
**Entity** — a class mapped to a database table by EF Core (ch. 4).
**Extension method** — a static method that appears to extend a type you don't own, called with instance-method syntax (ch. 1).
**Fluent API** — EF Core's code-based (as opposed to attribute-based) way of configuring entity schema/relationships (ch. 4).
**FluentValidation** — declarative input-validation library (ch. 7).
**GIN index** — a Postgres index type suited to arrays/full-text search (ch. 13).
**HMAC** — Hash-based Message Authentication Code; used here for password hashing (ch. 8).
**HNSW** — Hierarchical Navigable Small World; an approximate-nearest-neighbor index algorithm used for fast vector search (ch. 13).
**HTTPS / TLS termination** — where encrypted transport is decrypted, often at a reverse proxy/load balancer rather than the app itself (ch. 8, 11).
**IHostedService** — the interface behind background/long-running services in ASP.NET Core (ch. 10).
**IQueryable\<T\>** — a LINQ query that (for EF Core) translates to SQL rather than running in memory (ch. 1, 4).
**JWT** — JSON Web Token; a signed, self-contained token proving an identity's claims (ch. 8).
**LINQ** — Language Integrated Query; C#'s built-in, statically-typed query syntax (ch. 1).
**Mediator pattern** — decouples a message sender from its handler via a central dispatcher (ch. 5).
**MediatR** — the .NET library implementing the mediator pattern used for CQRS in this repo (ch. 5).
**Middleware** — a pipeline stage that can inspect/modify an HTTP request and response (ch. 3).
**Migration** — a versioned, incremental description of a database schema change (ch. 4).
**N+1 problem** — issuing one query per item in a loop instead of one batched query (ch. 4, 12).
**NuGet** — .NET's package registry/manager (ch. 2).
**Onion / Clean Architecture** — architectural styles where dependencies point inward toward a framework-agnostic Domain (ch. 6).
**OWASP** — Open Web Application Security Project; the de facto standard checklist for common web vulnerabilities (ch. 8).
**pgvector** — a PostgreSQL extension adding a native vector column type and similarity search (ch. 13).
**Query filter (global)** — an automatic `WHERE` clause EF Core applies to every query against an entity, used here for soft delete (ch. 4).
**RAG** — Retrieval-Augmented Generation; grounding an LLM's answer in retrieved documents instead of parametric memory alone (ch. 13).
**Record (C#)** — a reference type with built-in value equality and concise declaration syntax (ch. 1).
**SaveChanges / SaveChangesAsync** — the EF Core call that commits all tracked changes as SQL (ch. 4).
**Scoped / Transient / Singleton** — the three DI lifetimes controlling how often a new instance is created (ch. 3).
**Serilog** — a structured logging library used as the concrete provider behind `ILogger` (ch. 10).
**SignalR** — ASP.NET Core's real-time (WebSocket-based) communication library (ch. 10).
**Soft delete** — flagging a row as deleted (`IsDeleted = true`) instead of physically removing it (ch. 4, 5).
**SOLID** — five object-oriented design principles: Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion (ch. 6).
**Split query (`AsSplitQuery`)** — issuing one SQL query per `Include()` instead of one large join, to avoid row multiplication (ch. 4, 12).
**STO** — Storage/State Transfer Object; the write-side shape received by a Command (ch. 5).
**Structured logging** — logging named properties as separate, queryable fields rather than one flattened string (ch. 10).
**TF-IDF** — Term Frequency–Inverse Document Frequency; a classic statistic for identifying important words in a document (ch. 13).
**Template Method (pattern)** — a base class defines an algorithm's skeleton, leaving individual steps for subclasses to override (ch. 6).
**Thread pool** — the shared set of worker threads ASP.NET Core uses to handle concurrent requests (ch. 12).
**Two-factor authentication (2FA)** — a second proof of identity beyond a password, e.g. an emailed code or an authenticator app (ch. 8).
**Unit of Work** — a pattern grouping multiple changes into one atomic commit; `DbContext` already implements this (ch. 4, 6).
**WebApplicationFactory** — a test host that boots the real ASP.NET Core app in-process for integration testing (ch. 9).
**xUnit** — the .NET test framework used in `HRNS.Tests` (ch. 9).
