# Session Memory

This feature has a real evolution story — it was built once, hit a genuine design ceiling, and was rebuilt on different foundations after a direct conversation about the trade-off. Both versions are worth understanding, because the second design only makes sense in light of what the first one couldn't do.

## Version 1: reply-threading (stateless)

The original design had **zero server-side memory**, in keeping with the app's stateless philosophy (see [Architecture](02-architecture.md)). Every `/ask` answer embedded its own running Q/A transcript inside itself, hidden inside a collapsed Telegram `<blockquote expandable>` block:

```
[visible answer text]

▸ ⎯⎯⎯ ask-context (tap to expand, do not edit) ⎯⎯⎯
  Q: what are embeddings?
  A: they map text to vectors.
  ===
  Q: and what about vector search?
  A: ...
```

Replying to that message let the next turn recover the whole transcript from `message.reply_to_message.text` — and because every answer embedded the *full* transcript so far (not just its own turn), replying to *any* earlier answer in a chain recovered everything up to that point, with no extra tracking. This was genuinely clever and fully stateless — but it had a hard requirement: **you had to reply**, every time, or the conversation started fresh.

Two real bugs surfaced from actually using it:

- **The visual-footprint bug**: the transcript block was first hidden with `<tg-spoiler>` — which, it turned out, only *blurs* text in place; the message still occupies its full vertical space either way. Switching to `<blockquote expandable>` (verified live via the Telegram API response showing `entities: [{type: "expandable_blockquote"}]`) gave a genuine collapse/expand UI element instead.
- **The silent-context-loss bug**: a very long answer plus its context block could exceed Telegram's 4096-char limit, in which case the block was (correctly) dropped rather than sent broken — but replying to that now-marker-less message then found *no* context at all. Fixed with a fallback: when a reply carries no recognized marker, fall back to using its raw visible text as context anyway — a reply always means "this is relevant," whether or not the bot's own bookkeeping survived.
- **The document-splitting gap**: a long document opened from the knowledge base gets split across several Telegram messages (see [Telegram UX](07-telegram-ux.md)); replying to a *non-last* fragment only carried that fragment's text, not the whole document.

## The pivot

The document-splitting bug was the trigger for a direct question: *does this really solve the underlying problem, or does the user still have to remember to reply every time?* The honest answer was the latter — reply-threading only ever fixed "context carries over **when you reply**," never "I can just keep chatting normally." That's a materially different feature, and building it required abandoning the zero-server-state constraint for this one piece.

## Version 2: ambient memory via Redis

```ts
// src/session/types.ts
export interface Session {
  transcript: string;
  documentPath?: string;
}

export interface SessionStore {
  get(userId: number): Promise<Session>;
  set(userId: number, session: Session): Promise<void>;
  clear(userId: number): Promise<void>;
}
```

Every plain-text message now loads the caller's session (keyed by Telegram user ID — a private bot's DMs, where `chat.id` and the user's own ID coincide) at the start of `ask.ts`, uses it as context automatically, and saves the updated transcript afterward — **no reply required**. `/clear` deletes the Redis key outright, rather than the old behavior of merely sweeping recent messages from the chat.

A second, related field rides along in the same session: `documentPath`, the last document the user opened via browsing. `document.ts`'s callback handler records it on every successful open; `ask.ts` re-fetches that document **fresh from GitHub** (never cached) on the next question, so "summarize this" or "add an agenda to the meeting" works with no reply needed either — and this is what actually superseded the document-splitting bug, more cleanly than patching the old reply-marker system further would have: since context no longer depends on which message fragment you reply to, only a tiny path is ever carried, never the document content itself, so there's nothing left to split.

This also let a chunk of message-visible complexity be **deleted**: the `<blockquote expandable>` context block, and all its marker-parsing functions, are gone from every `/ask` answer — answers are just the answer now, with no hidden bookkeeping riding along.

### Storage: Upstash Redis, with a graceful no-op fallback

```ts
// src/telegram/botInstance.ts
function createSessionStore(env): SessionStore {
  if (!env.KV_REST_API_URL || !env.KV_REST_API_TOKEN) {
    return new NullSessionStore(); // ambient memory simply off; everything else still works
  }
  return new RedisSessionStore(new Redis({ url: env.KV_REST_API_URL, token: env.KV_REST_API_TOKEN }));
}
```

Both env vars are optional and cross-validated (must be set together or not at all — `src/lib/env.ts`). This means the feature could be deployed *before* Redis was provisioned, with `NullSessionStore` keeping the bot fully functional (just without ambient memory) in the meantime — and flips on automatically the moment the real credentials are added, no code change required.

One naming gotcha worth recording: Vercel's **"Upstash for Redis" marketplace integration** injects `KV_REST_API_URL` / `KV_REST_API_TOKEN` (plus `KV_URL`, `REDIS_URL`, `KV_REST_API_READ_ONLY_TOKEN`) — **not** `UPSTASH_REDIS_REST_URL` / `UPSTASH_REDIS_REST_TOKEN`, which is what Upstash's own SDK docs lead you to expect when set up manually. The code matches whatever the actual integration injects, confirmed by inspecting the real Vercel project's env vars rather than assuming from documentation.

### `RedisSessionStore` / `NullSessionStore`

```ts
// src/session/RedisSessionStore.ts
export class RedisSessionStore implements SessionStore {
  async get(userId) {
    return (await this.redis.get<Session>(`mastery-bot:session:${userId}`)) ?? EMPTY_SESSION;
  }
  async set(userId, session) { await this.redis.set(`mastery-bot:session:${userId}`, session); }
  async clear(userId) { await this.redis.del(`mastery-bot:session:${userId}`); }
}
```

`RedisLike` (a `Pick<Redis, "get" | "set" | "del">`) keeps the same test-fake-friendly pattern used everywhere else in the app — tests construct `RedisSessionStore` against a plain in-memory `Map`-backed fake, never a real Redis connection.

No TTL is set on session keys — a deliberate choice given the tiny data volume (four users, short transcripts) and the explicit design goal of "persists until `/clear`," not "persists for some arbitrary window." For a much larger user base this would need revisiting.

## What each version got right

The reply-threading version wasn't a wrong turn — it correctly identified that most "memory" needs in a chat bot really are about *this specific thread*, and its zero-infrastructure approach was the right starting point for validating the feature before committing to real state. The pivot happened because a specific, concrete complaint ("I wouldn't like always to reply") revealed the actual requirement was ambient, not threaded — a distinction that's easy to miss until a real user hits it in practice.
