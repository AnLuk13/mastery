# 11. Orchestration Libraries

Goal: a map of the major frameworks you'll hear named constantly, what each is actually for, and — the more durable skill — how to decide whether you need one at all versus calling a provider's API directly.

## 11.1 What "orchestration" actually means here

Everything in chapters 5–9 (embeddings, RAG, tool calling, agent loops) is a *pattern*, not something any provider's raw API hands you fully assembled — you still have to write the retrieval logic, the message-building, the tool-execution loop, the retry handling. **Orchestration libraries** exist to provide reusable building blocks for these patterns, so you're not hand-rolling chapter 9's agent loop (or a RAG pipeline's chunking/embedding/retrieval plumbing) from scratch every time. None of them do anything a model API alone couldn't already do — they're productivity/consistency layers on top, the same relationship an ORM has to raw SQL, or `Microsoft.Extensions.AI`/`IChatClient` (§11.4) has to a raw HTTP call to an LLM provider.

## 11.2 LangChain and LlamaIndex — the general-purpose Python/JS frameworks

**LangChain** — the most widely-known orchestration framework, available in both Python and JavaScript/TypeScript. Provides abstractions for chaining prompts together, RAG pipelines (document loaders, text splitters mapping directly to chapter 6 §6.2's chunking, vector store integrations), agent loops (chapter 9), and a large ecosystem of provider/tool integrations. Its main criticism, worth knowing before you commit to it: the abstractions can add real complexity for simple use cases, and debugging *through* several layers of framework abstraction when something goes subtly wrong is genuinely harder than debugging code you wrote yourself.

**LlamaIndex** — originated specifically as a RAG-focused framework (the name is a direct nod to "index your data for an LLM to query") — historically stronger, more opinionated tooling specifically for the ingestion/indexing/retrieval side (chapter 6) than LangChain's broader, more general-purpose scope, though the two have converged in capability over time.

Both are reasonable choices once you're building something with real RAG/agent complexity and want to avoid reinventing chunking strategies and retrieval plumbing — but for a small, well-defined feature, plenty of production code (chapter 8 §8.4's example included) is entirely reasonable written directly against a provider's API with no framework at all.

## 11.3 Vercel AI SDK — the framework most directly relevant to your stack

The **Vercel AI SDK** (`ai` package) is a TypeScript-first library specifically aimed at building AI features into web apps — streaming chat UI helpers for React/Next.js, a unified interface across multiple model providers (switch from OpenAI to Anthropic to a local Ollama model by changing one provider object, not rewriting your call sites), and built-in tool-calling helpers matching chapter 8's pattern. Given `mastery-bot` is already a Next.js/TypeScript/Vercel project, this is the most natural framework to reach for *first* if you added an AI feature to it directly — it's specifically designed around exactly this stack, handling the streaming-response-to-React-component plumbing (chapter 16 §16.1) that's otherwise genuinely fiddly to get right by hand.

## 11.4 Microsoft.Extensions.AI / Semantic Kernel — the .NET side, and where HRNS already sits

On the .NET side (relevant directly to HRNS), **`Microsoft.Extensions.AI`** is Microsoft's provider-agnostic chat abstraction — `IChatClient`, `ChatMessage`, `ChatRole` — which HRNS's `AIService` is *already built on* (`../dotnet-mastery/13-ai-ml-integrations.md` §13.2), with `OllamaSharp`'s `OllamaApiClient` registered as the concrete `IChatClient` implementation. This is the .NET-ecosystem equivalent of what the Vercel AI SDK's unified-provider interface does for TypeScript: application code (`AIService`) depends on the `IChatClient` abstraction, not on Ollama specifically — chapter 13 §13.6's "why can't we register two `IChatClient`s" DI limitation is a direct, real consequence of this exact abstraction's design (one interface, one default registration).

**Semantic Kernel** is Microsoft's more full-featured orchestration framework (the .NET-world rough analogue of LangChain) — built on top of the same `Microsoft.Extensions.AI` abstractions, adding RAG/agent/planning helpers. HRNS's current implementation uses the lighter-weight `IChatClient` abstraction directly rather than the full Semantic Kernel framework — a reasonable, deliberate choice consistent with §11.1's "framework complexity should match actual need" principle, worth recognizing as *not* an oversight if you go looking for Semantic Kernel usage in HRNS and don't find it.

## 11.5 Deciding whether you need a framework at all

Reach for an orchestration library when at least one of these is true:

- You need **provider portability** — swapping model providers without rewriting call sites (`IChatClient`/Vercel AI SDK's core reason to exist).
- Your **RAG pipeline has real complexity** — multiple document types, hybrid retrieval (chapter 6 §6.4), reranking — enough that hand-rolling it duplicates a lot of well-trodden logic.
- Your **agent loop needs real robustness** — multi-tool, multi-step, with retry/error-recovery logic (chapter 9 §9.3) that's genuinely tedious to get right by hand.

Skip the framework, and call the provider's API directly (chapter 12), when:

- The task is a single, well-defined call — one prompt in, one structured response out (§8.4's `search_knowledge_base` tool, or a single classification/extraction call).
- You'd spend more time learning/fighting the framework's abstractions than the plumbing would have taken to write directly.
- You want full, transparent control over exactly what's sent to the model — a real, common complaint about heavier frameworks is not being sure what prompt they're actually constructing under the hood, which directly fights chapter 4 §4.5's "prompts are fragile, version and inspect them deliberately" discipline.

This mirrors this project's own stated engineering principle almost exactly (`no unnecessary abstractions`, `no premature infrastructure`) — the same judgment call that kept `mastery-bot` out of a full framework for Markdown rendering (chapter 5's hand-written formatter, not a heavyweight templating engine) applies directly here.

## Checkpoint

1. HRNS's `AIService` depends on `IChatClient`, not `OllamaApiClient` directly. Explain concretely what that buys them if they ever wanted to switch from Ollama to a cloud provider — and what §13.6's DI limitation shows about the cost of that abstraction not being unlimited.
2. You're adding a single "summarize this document" button to `mastery-bot`. Would you reach for the Vercel AI SDK, or a direct API call? Justify it using §11.5.
3. Name one concrete risk of a heavy orchestration framework that a hand-written implementation wouldn't have, and one concrete risk of hand-rolling everything that a mature framework would have already solved for you.
