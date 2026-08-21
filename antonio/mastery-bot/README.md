# Mastery Bot — Build Documentation

A retrospective on **Mastery Bot**: a private Telegram bot that turns a personal Markdown knowledge base into something you can browse, search, ask questions about (with live web access), and write to — from your phone, in a chat.

This isn't user documentation for the bot — it's *engineering* documentation for the person who built it: what got built, in what order, with which tools, and — more importantly — **why**. Every non-obvious decision here was made under a real constraint (cost, latency, a Telegram API limit, a Vercel platform quirk, or a bug found in production use), not picked arbitrarily. Where a decision was reversed later, both the original reasoning and the reason it changed are kept, because that's usually the more useful lesson.

## Reading order

1. **[Overview](01-overview.md)** — what the system is, the three moving pieces (content repo / app / Telegram), the full tech stack at a glance.
2. **[Architecture](02-architecture.md)** — the stateless-serverless design, how the codebase is layered, and the testing philosophy that made rapid iteration safe.
3. **[Content & Privacy](03-content-and-privacy.md)** — how documents are read and written on GitHub, and how per-folder privacy is enforced.
4. **[RAG & /ask](04-rag-ask.md)** — the retrieval-augmented chat pipeline: local embeddings, chunking, retrieval, and the Groq models behind it (including live web search).
5. **[Session Memory](05-session-memory.md)** — how conversational memory evolved from a stateless reply-threading trick into real ambient memory backed by Redis.
6. **[Save & Authoring](06-save-authoring.md)** — the AI-driven `/save` flow: deciding where content belongs, merging, one-tap revert, and the ask-first "reorganize" feature.
7. **[Telegram UX](07-telegram-ux.md)** — the grammY layer, stateless callback-data encoding, and the Markdown-to-Telegram-HTML rendering pipeline.
8. **[Lessons & Bugs](08-lessons-and-bugs.md)** — a log of the sharper bugs and reversals, and what each one taught.

## Stack at a glance

| Concern | Choice | Why (short version) |
|---|---|---|
| App framework | Next.js 16 (App Router) on Vercel | Serverless functions, zero infra to manage, generous free tier for a 4-user private bot |
| Language | TypeScript, strict mode | Correctness at the boundaries (paths, callback data, API responses) matters more than usual here |
| Telegram framework | [grammY](https://grammy.dev) | Modern, fully typed, small API surface |
| Validation | Zod | Every external input — env vars, API responses, callback data, model output — is untrusted until parsed |
| Content source of truth | A separate GitHub repo (`mastery`), read/written via the REST API | Durable storage the serverless app itself doesn't have |
| Embeddings | `@huggingface/transformers` running `Xenova/all-MiniLM-L6-v2` locally | Free, offline, deterministic — no embeddings API, no per-call cost |
| LLM | [Groq](https://groq.com) (two models: `openai/gpt-oss-120b`, `groq/compound-mini`) | Fast + cheap inference; the compound model adds live web search with no separate search API |
| Ambient memory | Upstash Redis (via Vercel's marketplace integration) | The only piece of real server-side state in an otherwise stateless app |
| Testing | Vitest, dependency injection everywhere | Every handler takes plain interfaces, so tests use fakes, not mocks or network calls |
| Reindexing | A scheduled GitHub Action (`.github/workflows/reindex.yml`) | Keeps `/ask`'s embeddings index in sync with the content repo automatically, instead of a manual rebuild-and-redeploy step |

## A note on scope

This documentation covers the *whole* build, end to end — from the original directory-browsing foundation through RAG chat, live web search, chat-driven authoring, per-folder privacy, ambient session memory, the reorganize-proposal feature, and automated reindexing. It reflects the codebase as it stands after that work, not a snapshot from partway through.
