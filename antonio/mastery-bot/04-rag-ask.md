# RAG & /ask

`/ask` (in practice, just any plain-text message — see below for why) answers questions grounded in the user's own notes, with live web search for anything the notes don't cover, and conversational memory across a normal back-and-forth (see [Session Memory](05-session-memory.md)).

## Why "any message" instead of an `/ask` command

Early on this was a deliberate choice: requiring `/ask <question>` adds friction to what should feel like just chatting. The trade-off is that it makes plain-text messages ambiguous with `/save` requests — resolved by keeping `/save` an **explicit** command (see [Save & Authoring](06-save-authoring.md)) specifically so free-text can safely default to "this is a question," never "this is something to write to disk."

## The pipeline, end to end

```
question ──► embed(transcript + question) ──► retrieveTopK ──► filter by isPathVisible
                                                                        │
                                                                        ▼
                                              build prompt (retrieved notes + reference doc + prior turns)
                                                                        │
                                                                        ▼
                                                              Groq chat completion
                                                                        │
                                                                        ▼
                                          Markdown → Telegram-HTML render → reply, update session
```

(`src/rag/answerQuestion.ts` is the whole pipeline in one function; `src/telegram/handlers/ask.ts` is the Telegram-facing wrapper that loads/saves session state around it.)

## Local embeddings — no embeddings API

Questions and document chunks are embedded with `Xenova/all-MiniLM-L6-v2` (384-dim), run **locally inside the Vercel function** via `@huggingface/transformers`, rather than calling an embeddings API:

```ts
// src/rag/embeddingModel.ts
env.allowRemoteModels = false;
env.localModelPath = path.join(process.cwd(), "public", "models");
...
const output = await extractor(text, { pooling: "mean", normalize: true });
```

The model files are committed under `public/models/` and bundled into the deploy, rather than fetched from the Hugging Face Hub at runtime — a Vercel function has no durable cache between cold starts, so leaving remote-model loading on would mean re-downloading ~23MB on *every* cold start, plus a hard runtime dependency on `huggingface.co` being reachable. Bundling makes embedding fully offline, deterministic, and free (no per-call cost, no rate limit) — consistent with the rest of the app depending on nothing but its own deployed code and the GitHub/Groq/Redis APIs it explicitly calls.

## Chunking

`src/rag/chunk.ts` splits each document by Markdown heading (any level) into sections, then further splits any section over 1200 characters into overlapping ~1200-char windows (150-char overlap, breaking on paragraph boundaries where possible) — long enough to preserve context within a chunk, short enough to keep retrieval precise and prompts small.

## The embeddings index is precomputed, not built live

`scripts/buildEmbeddingsIndex.ts` walks the entire content root, chunks every document, embeds every chunk with the same model the live bot uses, and writes the result to `src/rag/data/embeddingsIndex.json` — which gets **committed and deployed like any other source file**.

This was a deliberate choice, not a shortcut: embedding an entire knowledge base on every chat request would be slow and wasteful when the corpus only changes when notes are edited. The cost is that the index goes stale the moment content changes without a rebuild.

### Keeping it fresh: a scheduled GitHub Action, not a manual step

The first version of this required manually running `npm run build:index`, committing the result, and redeploying — genuinely easy to forget, especially once four people were saving notes through `/save` and none of those writes went anywhere near the `mastery-bot` repo or its deploy pipeline.

`.github/workflows/reindex.yml` (in the `mastery-bot` repo) now automates the whole loop: on a schedule (every 30 minutes, plus a manual "run now" via `workflow_dispatch`), it checks out both repos, reruns `buildEmbeddingsIndex.ts` against the current `mastery` content, and — only if the resulting index actually changed — commits it and triggers a Vercel production deploy via a **deploy hook** (a single-purpose, easily-revocable trigger URL, deliberately not a broad Vercel API token).

This needed exactly one new credential: a **read-only, fine-grained GitHub PAT scoped to just the `mastery` repo** (`Contents: read`), stored as the `CONTENT_REPO_TOKEN` secret on `mastery-bot`, used only to check out the private content repo during the workflow run — the push back to `mastery-bot` itself uses GitHub Actions' own automatic per-run token, which already has write access to the repo the workflow lives in, so no second write-scoped secret was needed for that half.

The remaining trade-off is now a bounded delay (up to ~30 minutes) between saving a note and it becoming answerable via `/ask`, rather than an indefinite one waiting on someone to remember a manual step.

## Retrieval

```ts
// src/rag/retrieve.ts
function cosineSimilarity(a, b) {
  let dot = 0;
  for (let i = 0; i < a.length; i++) dot += a[i] * b[i];
  return dot; // vectors are already L2-normalized at embed time, so dot product == cosine similarity
}
```

A brute-force linear scan over every chunk in the index — the same trade-off already made for plain-text search (`GitHubContentProvider.search()`): simple, fast enough for a personal-scale corpus, and would need real vector-index infrastructure only if the corpus grew far beyond what one person (or four) could realistically write.

Retrieved chunks are filtered through `isPathVisible()` (see [Content & Privacy](03-content-and-privacy.md)) *before* being cited or included in the prompt — a private chunk never reaches another user's context, let alone their answer.

`CITATION_SCORE_THRESHOLD = 0.45` decides which retrieved chunks are cited as sources (vs. just loosely offered to the model as context): calibrated against real questions during testing — unrelated topics scored ~0.1–0.2, genuinely in-corpus topics scored 0.55+, and a borderline case (a React question, sharing vocabulary with AI/app-building chapters without being about them) scored 0.37, high enough to false-positive at an earlier, lower 0.35 threshold.

## Two Groq models, for two different jobs

```
GROQ_MODEL      = "openai/gpt-oss-120b"     — used by /save's structured decisions
GROQ_ASK_MODEL  = "groq/compound-mini"      — used by /ask
```

`groq/compound-mini` is Groq's **agentic** model — it decides on its own when a question needs a live web search and runs it, giving `/ask` real internet access with no separate search API to integrate. This is *not* the model used for `/save`, because live testing established a hard incompatibility: compound models **reject `reasoning_effort` outright** (400 error, "not supported with this model"), while `/save`'s structured decisions need `reasoning_effort` *and* JSON mode — capabilities compound models don't support. Rather than trying to make one model do both jobs, `GroqClient` (`src/rag/groqClient.ts`) takes these as request-time options and the app constructs **two separate client instances**, one per model, wired into `createBot()` as distinct dependencies (`askDeps.groq` vs `saveGroq`) specifically so a future change to one model's parameters can never silently break the other's calls.

```ts
export interface ChatCompletionOptions {
  reasoningEffort?: "low" | "medium" | "high"; // omit entirely for compound models
  jsonMode?: boolean;
}
```

### The empty-completion bug

`openai/gpt-oss-120b` is a *reasoning* model — it spends part of its token budget on a hidden reasoning pass before producing the visible answer. Without capping that (`reasoning_effort: "low"`), the hidden pass alone could exhaust `max_tokens`, leaving the visible `content` field **empty**. `GroqClient` now defensively throws `GroqUnavailableError` if a completion's trimmed content is empty, and `/save`'s calls always pass `reasoningEffort: "low"` — a bug that was invisible until tested against a real model with reasoning enabled, since nothing about the API response *looked* wrong (200 OK, valid JSON, just an empty string).

## Prompt construction

`answerQuestion()`'s prompt has three optional blocks, each framed distinctly so the model doesn't conflate them:

- **Retrieved notes** — always present, framed as ground truth to cite (or ignore, if irrelevant — the system prompt explicitly permits answering from general knowledge when the notes don't apply, rather than forcing a note-grounded answer that doesn't fit).
- **A reference document** — the document the user was last browsing (see [Session Memory](05-session-memory.md)), framed as "may or may not be relevant," never as authoritative in the way retrieved notes are.
- **Prior conversation** — the running session transcript, framed as context for pronoun-heavy follow-ups ("summarize *that*").

The query embedded for retrieval also combines the prior transcript with the new question when one exists — a pronoun-heavy follow-up carries almost no retrievable meaning on its own, so embedding it together with recent turns keeps retrieval on-topic instead of searching on "that" alone.

## Rendering

Model answers are Markdown, and go through the **same** Markdown→Telegram-HTML pipeline used for documents (`renderDocumentMessages`, detailed in [Telegram UX](07-telegram-ux.md)) — bold, bullet lists, and inline code render properly instead of showing up as literal asterisks, and answers that overflow Telegram's 4096-character limit split safely across multiple messages using the same logic a long document uses.
