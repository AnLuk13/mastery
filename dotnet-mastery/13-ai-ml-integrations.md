# 13. AI/ML Integrations (bonus)

An enterprise HR/payroll platform with a built-in AI assistant backed by a real RAG (Retrieval-Augmented Generation) pipeline, hybrid vector+keyword search, and its own ML.NET model is genuinely unusual — worth a dedicated chapter both because it's interesting and because it's a great forcing function to consolidate everything from chapters 3–7 (DI, EF Core, generic services) against an unfamiliar domain.

## 13.1 The shape of the feature

Confirmed from `Program.cs` and the entities behind them:

```csharp
services.AddChatClient(new OllamaApiClient(ollamaHttpClient, ollamaModel));
services.AddScoped<IAIService, AIService>();
services.AddScoped<IAIAssistantService, AIAssistantService>();
services.AddScoped<IVectorSearchService, VectorSearchService>();
services.AddScoped<IKeywordSearchService, KeywordSearchService>();
services.AddScoped<IStructuralSearchService, StructuralSearchService>();
services.AddScoped<IHybridRetrieverService, HybridRetrieverService>();
services.AddScoped<IRerankerService, RerankerService>();
services.AddScoped<IConversationStore, ConversationStore>();
services.AddScoped<IFeedbackService, FeedbackService>();
services.AddHostedService<KeywordIndexHostedService>();
services.AddHostedService<KnowledgeIngestionHostedService>();
```
(`HRNS.WebApi/Program.cs:295-305, 216`.) Reading the interface names alone (before opening a single implementation file) already tells you the architecture — this is a **hybrid retrieval RAG pipeline**: multiple independent retrieval strategies (vector similarity, keyword match, structural/metadata match) each propose candidates, a reranker combines/reorders them, and the result grounds an LLM's answer instead of the LLM hallucinating from parametric memory alone. This is worth doing as a general reading skill, not just for this feature: a well-named set of DI registrations is often a legitimate architecture diagram, no source-reading required (chapter 3 §3.3's registration-list pattern, paying off directly here).

## 13.2 `Microsoft.Extensions.AI` — the provider-agnostic chat abstraction

`Microsoft.Extensions.AI` is Microsoft's attempt at a standard, provider-agnostic interface for talking to any LLM — `IChatClient`, `ChatMessage`, `ChatRole` — so application code doesn't couple directly to a specific vendor SDK. This app's actual implementation is `OllamaSharp`'s `OllamaApiClient`, registered *as* an `IChatClient`:

```csharp
public class AIService : IAIService
{
    private readonly IChatClient _chatClient;

    private async Task<string> AskJsonStructuredAsync(string systemPrompt, string userPrompt)
    {
        var chatHistory = new List<ChatMessage>
        {
            new ChatMessage(ChatRole.System, systemPrompt),
            new ChatMessage(ChatRole.User, userPrompt)
        };

        var aiResponse = new StringBuilder();
        await foreach (var update in _chatClient.GetStreamingResponseAsync(chatHistory))
            aiResponse.Append(update.Text);

        return aiResponse.ToString().Trim();
    }
}
```
(`HRNS.Application/Implementation/AIService/AIService.cs:22-78`, condensed.) `await foreach` is C#'s **async streaming** construct — consuming an `IAsyncEnumerable<T>` element-by-element as they arrive, rather than waiting for the whole response then processing it at once (the same reason `ChatGPT`-style UIs render token-by-token instead of waiting for the full answer). `AIAssistantHub`'s SignalR connection (chapter 10 §10.2) is exactly what carries these streaming chunks out to the browser in real time as they're generated.

**Ollama** itself is a self-hosted LLM runtime (runs open-weight models like Llama/Mistral locally or on your own infrastructure, rather than calling a hosted API like OpenAI's) — consistent with this being an on-prem-friendly enterprise HR platform where sending employee data to a third-party AI API might be a real compliance concern. Startup validates the configured chat/embedding models actually exist on the target Ollama server *before* the app finishes booting (`Program.cs:256-293`, already excerpted in chapter 2 §2.5) — a "fail fast and loud at startup" choice, rather than discovering a missing model on the first real user request.

## 13.3 Embeddings & pgvector — how "semantic search" actually works

An **embedding** is a fixed-length vector of floating-point numbers (768 dimensions here, using `nomic-embed-text` per this entity's own doc comment) that represents the *meaning* of a piece of text — texts with similar meaning produce vectors that are close together in that 768-dimensional space, regardless of exact wording. "Semantic search" = embed the search query the same way, then find the stored chunks whose embeddings are geometrically closest to it (commonly via cosine similarity — the angle between two vectors, not their raw distance).

`KnowledgeChunkEntity` (`HRNS.Database/Entities/AIAssistantKnowledge/KnowledgeChunkEntity.cs`) is the concrete storage for this — genuinely worth reading in full, it's a small, clean example of the whole RAG-storage concept in one class:

```csharp
public class KnowledgeChunkEntity : Entity
{
    public string SourceFile { get; set; }        // e.g. 'absences.md'
    public string SectionPath { get; set; }         // e.g. 'EMPLOYEES > Creating a New Employee > Required Fields'
    public string Content { get; set; }               // the actual chunked text
    public string Summary { get; set; }                 // LLM-generated 1-sentence summary
    public Vector Embedding { get; set; }                 // 768-dim vector of Content — pgvector's Vector type
    public Vector SummaryEmbedding { get; set; }             // separate embedding of just the Summary
    public string[] Keywords { get; set; }                    // ML.NET TF-IDF extracted keywords — see §13.5
    public string ContentHash { get; set; }                     // for detecting unchanged content on re-ingestion
}
```

And the schema configuration behind it — a genuinely advanced piece of EF Core Fluent API, worth slowing down on:

```csharp
builder.Property(e => e.Embedding).HasColumnType("vector(768)"); // Postgres column type, via the pgvector extension

builder.HasIndex(e => e.Embedding)
    .HasMethod("hnsw")                   // Hierarchical Navigable Small World — an approximate nearest-neighbor index
    .HasOperators("vector_cosine_ops")     // index for cosine-distance queries specifically
    .HasStorageParameter("m", 16)
    .HasStorageParameter("ef_construction", 64);

builder.HasIndex(e => e.Keywords).HasMethod("gin"); // GIN index — Postgres's standard index type for array/full-text columns
```
(`HRNS.Database/Entities/AIAssistantKnowledge/KnowledgeChunkEntityDbConfig.cs:29-50`, condensed.) `HNSW` is the current state-of-the-art approximate-nearest-neighbor index algorithm — it trades a small amount of search accuracy for dramatically faster lookups than an exhaustive "compare against every row" scan, which matters enormously once you have more than a few thousand chunks. This entire chunk-of-EF-Core-config is only possible because `HRNS.Database.csproj` references `Pgvector`/`Pgvector.EntityFrameworkCore` (chapter 2's tech-stack table) and `HRNSDbContext.OnModelCreating` calls `modelBuilder.HasPostgresExtension("vector")` (chapter 4 §4.2) to enable the underlying Postgres extension in the first place.

## 13.4 Hybrid retrieval — why vector search alone isn't enough

`IVectorSearchService`, `IKeywordSearchService`, `IStructuralSearchService`, and `IHybridRetrieverService` + `IRerankerService` together implement a well-known, deliberate RAG pattern: pure semantic (vector) search is excellent at "conceptually similar" but can miss exact terminology matches (an acronym, a specific field name) that a plain keyword search catches trivially — and vice versa, keyword search misses paraphrases a vector search catches easily. A **hybrid retriever** runs both (plus, here, a "structural" search — almost certainly matching on `SourceFile`/`SectionPath`/`PageRoute` metadata rather than content at all, worth confirming by reading `StructuralSearchService` yourself), then a **reranker** merges and reorders the combined candidate set before handing the top results to the LLM as context. This is genuinely current, production-grade RAG architecture — not a toy "just embed everything and cosine-similarity it" implementation, and a good thing to recognize by name if you encounter it (or need to build something like it) elsewhere.

## 13.5 ML.NET — classic ML alongside the LLM pieces

`Microsoft.ML`/`Microsoft.ML.LightGbm` (chapter 2's stack table) are ML.NET, Microsoft's classic (non-LLM) machine learning framework — training/running traditional models (gradient-boosted trees via LightGBM, TF-IDF text vectorization, etc.) directly in .NET, no Python required. `ISmartKeywordExtractorService`/`SmartKeywordExtractorService` (registered alongside the AI services in `Program.cs:224`) is very likely what populates `KnowledgeChunkEntity.Keywords` via TF-IDF (Term Frequency–Inverse Document Frequency — a classic, cheap, non-LLM statistic for "which words in this document are distinctively important," older and much faster than embedding-based extraction) — worth opening yourself to confirm and see ML.NET's pipeline-building API (`MLContext`, `Transforms`, `Fit`/`Transform`) in a real, working example rather than a tutorial.

## 13.6 The two-Ollama-instance workaround — a real DI limitation, worked around

`AIInputGathering` (a related but separate AI feature) deliberately talks to a **different**, cloud-hosted Ollama instance than the rest of the platform, per this comment in `Program.cs`:

```csharp
// ── AI Input Gathering: Ollama Cloud ──
// Uses a separate cloud-based Ollama instance (with API key)
// instead of the local Ollama used by the rest of the platform.
// The AIInputGatheringAIService creates its own OllamaApiClient internally
// from AIInputGathering:CloudAddress + ApiKey + Model config, so no
// IChatClient conflict with the local instance.
services.AddScoped<IAIInputGatheringAIService, AIInputGatheringAIService>();
```
(`Program.cs:307-313`.) The comment explains a real, structural DI limitation worth understanding rather than papering over: `services.AddChatClient(...)` registers exactly *one* `IChatClient` implementation against the container (chapter 3 §3.3's registration model is one-interface-to-one-implementation by default) — you can't register two different `IChatClient`s and have DI automatically know which consumer wants which. The workaround here is `AIInputGatheringAIService` **not** taking `IChatClient` as a constructor dependency at all — it builds its own `OllamaApiClient` internally, pointed at the cloud instance's config, sidestepping the DI container entirely for this one dependency. The more modern, "correct" fix for this exact scenario is .NET 8+'s **keyed DI services** (`AddKeyedScoped<IChatClient>("cloud", ...)`, resolved via `[FromKeyedServices("cloud")]`) — worth knowing that this capability now exists in the platform, even though this specific code predates using it here or made a deliberate choice not to.

## Checkpoint

1. Read `StructuralSearchService` (or `KeywordSearchService`) yourself and confirm/correct the guess in §13.4 about what it actually matches on.
2. `KnowledgeIngestionHostedService` and `KeywordIndexHostedService` are both `IHostedService`s (chapter 10 §10.1). What do you expect each to do at startup, based purely on its name and the entity fields it would need to populate (`Embedding`, `SummaryEmbedding`, `Keywords`, `ContentHash`)?
3. Explain, in one paragraph, why `ContentHash` exists on `KnowledgeChunkEntity` — what problem does re-ingesting the same source document without it cause, given that generating an embedding is a real (cost/latency) LLM call?
