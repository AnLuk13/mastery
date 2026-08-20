# 16. Building AI Features Into Apps

Goal: pull every earlier chapter together into the actual product-engineering questions — where inference runs, how streaming reaches a UI, and a concrete worked sketch of adding an AI feature to `mastery-bot`.

## 16.1 Where should inference actually run?

Three real options, each with a genuine tradeoff, not a default "obviously correct" answer:

- **Server-side (your backend)** — the model API call happens in your own server/serverless function, never exposing the API key to the client (chapter 17 §17.2 — this is close to non-negotiable for any provider requiring a secret key). This is exactly `mastery-bot`'s existing architecture for GitHub calls (`src/content/github/GitHubApiClient.ts` runs server-side, never in the browser or in Telegram) — an AI feature added to it would follow the identical shape: a Route Handler or server action makes the model call, the client only ever sees the result.
- **Edge functions** — a server-side call, but running on infrastructure geographically closer to the user for lower latency (Vercel Edge Functions, Cloudflare Workers). Genuinely valuable for latency-sensitive, high-traffic global apps; adds real constraints (not every Node.js API is available in an Edge runtime — this project's own webhook route explicitly opts into `runtime = "nodejs"` rather than Edge, `src/app/api/telegram/webhook/route.ts`, precisely because it needs full Node APIs like `node:crypto`).
- **On-device (client-side)** — the model runs directly in the browser or mobile app, no server round-trip at all. Currently realistic only for small, specifically-optimized models (on-device inference is an active, fast-moving area — WebGPU-accelerated in-browser models, mobile-optimized quantized models) — not a drop-in replacement for a frontier hosted model, but genuinely compelling for privacy-sensitive or offline-required features at small scale, echoing chapter 13's local-model tradeoffs one layer further down the stack (your user's device instead of your own server).

For nearly everything you'll build early on: **server-side is the correct default**, for the same reason `mastery-bot` never lets Telegram or a browser talk to GitHub directly — secrets stay server-side, and you keep full control over validation, rate limiting, and cost (chapter 12 §12.3).

## 16.2 Streaming into a real UI

Chapter 12 §12.2 covered *why* providers stream; here's *how* it actually reaches a rendered UI, using the Vercel AI SDK's pattern (chapter 11 §11.3) as the concrete, stack-relevant example, since it's specifically designed around Next.js:

```tsx
// Server: a Route Handler that streams a model's response back to the client
import { streamText } from "ai";

export async function POST(request: Request) {
  const { messages } = await request.json();
  const result = streamText({ model: /* provider model */, messages });
  return result.toTextStreamResponse();
}
```

```tsx
// Client: a React hook that consumes the stream and re-renders as chunks arrive
"use client";
import { useChat } from "ai/react";

export function ChatPanel() {
  const { messages, input, handleInputChange, handleSubmit } = useChat();
  return (
    <div>
      {messages.map(m => <p key={m.id}>{m.content}</p>)}
      <form onSubmit={handleSubmit}>
        <input value={input} onChange={handleInputChange} />
      </form>
    </div>
  );
}
```

`useChat` is doing, under the hood, exactly what chapter 12 §12.2 described conceptually — consuming an SSE-style stream and updating component state as each chunk arrives, which React then re-renders — the library exists specifically so you don't hand-write that plumbing yourself.

## 16.3 A worked sketch: adding AI-powered search to `mastery-bot`

Pulling together chapters 5, 6, 8, and 16.1 into one concrete plan, respecting the project's existing architecture and stated principles rather than bolting something on carelessly:

1. **Keep `ContentProvider.search()`'s existing contract** (`src/content/types.ts`) — a new capability shouldn't force every caller to change; add semantic search as an *additional* method or an option on the existing one, not a parallel, disconected system.
2. **Precompute embeddings, don't compute them per-request** — at build/deploy time (or lazily, cached per warm instance — never relied on for correctness, matching this project's existing caching stance, `README.md`'s caching notes), embed every document once (chapter 5 §5.1) and store the vectors alongside the content — for a personal knowledge base's realistic scale (chapter 5 §5.4), a JSON file checked into the content repo, or computed on demand and cached, is entirely sufficient; no dedicated vector database needed at this scale.
3. **Retrieval, not generation, for the search feature itself** — `/search` returning ranked *documents* (chapter 5 §5.2's cosine similarity) doesn't need a chat model call at all — it's chapter 5 alone, no chapter 6 needed, keeping cost and latency low for something used on every keystroke-equivalent search.
4. **A genuinely new feature — `/ask`** — *this* is where chapter 6's full RAG pattern earns its place: a natural-language question, embedded, matched against the same document vectors, top chunks assembled into a grounded prompt (chapter 6 §6.3), sent to a chat model, streamed back — reusing `renderDocumentMessages`'s existing Markdown-to-Telegram-HTML formatter (`src/telegram/formatting/index.ts`) to render the answer exactly like any other bot message, no new formatting code needed.
5. **Same security posture as everything else in this project** — the model never receives anything path-related to execute (chapter 8 §8.4's point about tool arguments staying validated) — it only ever receives already-retrieved *text content*, and its output goes through the exact same HTML-escaping pipeline (`src/telegram/formatting/html.ts`) as any Markdown document already does, so a model output containing `<`/`>`/`&` can't produce malformed Telegram markup any more than untrusted Markdown content already can't.

## Checkpoint

1. Why does step 3 (`/search`) deliberately avoid calling a chat model at all, when step 4 (`/ask`) does? What would calling a chat model for every keystroke-style search cost, using chapter 12 §12.4's per-token pricing model?
2. `mastery-bot`'s webhook route already awaits full processing before responding (`src/telegram/webhookHandler.ts`, chapter 9 of this project's own build — see Stage 7's design notes). Explain why an `/ask` feature calling a model would make that "await everything before responding" choice *more* consequential, given chapter 3's inference isn't instant, and connect it to the Telegram callback-timing bug this project actually hit and fixed.
3. Sketch, in one sentence each, what changes and what stays identical in `mastery-bot`'s architecture (Telegram layer → application layer → `ContentProvider` abstraction → concrete provider, `README.md`) if `/ask` is added.
