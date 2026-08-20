# 5. Embeddings & Vector Search

Goal: the mechanism underneath "semantic search," and the foundation everything in chapter 6 (RAG) is built on. This is the single most reusable, product-relevant concept in this whole guide.

## 5.1 What an embedding actually is

An **embedding** is a fixed-length list of floating-point numbers (a **vector**) that represents the *meaning* of a piece of text (or an image, or audio — chapter 18), produced by a separate, much smaller model trained specifically for this (an **embedding model** — not the same model that generates chat responses, though some providers offer both under one API). Two texts with similar meaning produce vectors that end up *close together* in that numeric space, regardless of exact wording — "vacation policy" and "time off rules" land near each other; "vacation policy" and "database migration" land far apart.

Nothing here is symbolic or rule-based — the model was trained (chapter 2 §2.1, closer to unsupervised) on massive amounts of text where similar contexts get pulled together geometrically, purely through the same gradient-descent training loop from chapter 2 §2.4, just with a different objective than next-token prediction. Your HRNS `KnowledgeChunkEntity.Embedding` (`../dotnet-mastery/13-ai-ml-integrations.md` §13.3) is a concrete, real 768-dimensional vector produced exactly this way, using `nomic-embed-text`.

```ts
// Conceptual shape — a real embedding call, illustrative response:
const vector = await embed("How many vacation days do new employees get?");
// vector: [0.0123, -0.441, 0.892, ...]  (768, 1536, or however many dimensions the model produces)
```

## 5.2 Measuring "closeness": similarity metrics

Given two embedding vectors, you need a number expressing how similar they are. The overwhelmingly common choice is **cosine similarity** — the cosine of the angle between the two vectors, ranging from -1 (opposite meaning) to 1 (identical meaning), deliberately ignoring vector *length* and only caring about *direction* (why: embedding models tend to encode meaning primarily in direction, with magnitude often reflecting less-relevant properties like text length). Your HRNS index config explicitly targets this — `HasOperators("vector_cosine_ops")` (`../dotnet-mastery/13-ai-ml-integrations.md` §13.3) tells Postgres/pgvector to build its index specifically for cosine-distance queries. Euclidean (straight-line) distance is the other common option, used less often for text embeddings specifically.

```ts
function cosineSimilarity(a: number[], b: number[]): number {
  const dot = a.reduce((sum, v, i) => sum + v * b[i], 0);
  const magA = Math.sqrt(a.reduce((s, v) => s + v * v, 0));
  const magB = Math.sqrt(b.reduce((s, v) => s + v * v, 0));
  return dot / (magA * magB);
}
```

"Semantic search" is now a precise, three-step recipe: embed the search query the same way you embedded your documents, compute similarity between the query vector and every stored document vector, and return the top-N highest-scoring matches.

## 5.3 Why you can't just brute-force it at scale — and what an index buys you

Computing cosine similarity against *every single stored vector* for every query (an exhaustive scan) is correct but doesn't scale — fine for hundreds of vectors, painful for millions. A **vector index** trades a small amount of accuracy for dramatically faster approximate lookups. **HNSW** (Hierarchical Navigable Small World) — the algorithm your HRNS pgvector index actually uses (`HasMethod("hnsw")`, `../dotnet-mastery/13-ai-ml-integrations.md` §13.3) — is the current standard approach: it builds a multi-layer graph structure over the vectors at index-build time, letting queries navigate toward the nearest neighbors in roughly logarithmic time instead of scanning everything linearly. You don't need to implement HNSW yourself — every real vector database/extension provides it — but recognizing the name, and knowing it's an *approximate* nearest-neighbor search (a small, tunable, usually-acceptable accuracy trade for enormous speed) is worth having.

## 5.4 Where the vectors actually live: vector databases

You need somewhere to store embeddings alongside enough metadata to retrieve the original content, queryable by similarity. Broad options, roughly by where you're likely to reach for each:

- **Postgres + pgvector** — an extension adding a native `vector` column type and index support to an ordinary Postgres database. This is HRNS's real choice (`../dotnet-mastery/13-ai-ml-integrations.md` §13.3) — a strong default whenever you already run Postgres, since you get vector search alongside your normal relational data with no new infrastructure to operate.
- **Dedicated vector databases** (Pinecone, Weaviate, Qdrant, Milvus) — purpose-built, often with more advanced filtering/scaling features out of the box, at the cost of one more system to run and pay for. Worth it once vector search is a large, independently-scaling part of your workload.
- **Embedded/local options** (Chroma, sqlite-vec, or `mastery-bot`'s own filesystem) — fine for prototypes or small, single-instance workloads; not what you'd reach for behind a production multi-instance web app.

For a hypothetical "semantic search" feature added to `mastery-bot` (which currently does plain substring matching in `ContentProvider.search()` — `src/content/LocalFilesystemContentProvider.ts`/`GitHubContentProvider.ts`), the honest answer is: you'd need *somewhere* to persist embeddings, which immediately means introducing real storage — exactly the kind of "don't add a database prematurely" tradeoff the project's own CLAUDE.md instructions call out explicitly. Small-scale, this is solvable without a dedicated vector database at all: precompute embeddings for every markdown file at build/deploy time, store them as a JSON file alongside the content (or fetched once and cached in memory per warm instance — never relied on for correctness, same principle as this project's existing caching stance), and brute-force cosine similarity in code (§5.3) — genuinely fine up to a few thousand documents, which comfortably covers a personal knowledge base.

## 5.5 Embeddings vs. chat models — different tools, don't conflate them

A beginner mistake: assuming "the AI" is one undifferentiated thing. In practice you're normally combining *at least* two distinctly different models — a **chat/completion model** (generates text, chapter 3) and a separate, much smaller, much cheaper **embedding model** (produces vectors, this chapter) — often from the same provider but always a distinct API call and a distinct model name. HRNS's `AddChatClient(...)` registration (`../dotnet-mastery/13-ai-ml-integrations.md` §13.1) is the chat model; the embedding model that produces `nomic-embed-text` vectors (§13.3) is configured and called separately. Embedding models are not used to "chat" — asking one to answer a question does nothing useful; its only job is turning text into a vector.

## Checkpoint

1. Explain why `KnowledgeChunkEntity` stores *two* separate embeddings (`Embedding` for `Content`, `SummaryEmbedding` for `Summary`) rather than just one — what retrieval scenario does each serve?
2. Two documents about "employee onboarding" use almost entirely different vocabulary from each other. Explain, using §5.1's training intuition, why their embeddings could still end up close together.
3. Sketch (in words, not code) how you'd add a `/search --semantic` mode to `mastery-bot`'s existing `/search` command, reusing as much of the existing `ContentProvider` abstraction as possible — where would embeddings get computed, and where would they be stored, given the project's "no premature database" stance?
