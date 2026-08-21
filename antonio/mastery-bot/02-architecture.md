# Architecture

## The stateless-serverless constraint

Vercel functions are ephemeral: nothing in memory survives between invocations, and even a "warm" instance can be recycled at any time. Rather than reaching for a database to work around that, the design treated statelessness as a constraint to design *with*:

- **Content** lives in GitHub (see [Content & Privacy](03-content-and-privacy.md)) — every read re-fetches from the API (or, for embeddings, from a file committed into the deploy itself).
- **Navigation state** ("what folder am I in") is encoded directly into each inline button's `callback_data` — any invocation can decode any button tap on its own, with no lookup (see [Telegram UX](07-telegram-ux.md)).
- **Conversation state** was, for most of this project's life, encoded into the bot's own messages via Telegram's reply-threading, recovered from `message.reply_to_message.text` on the next turn. This was eventually replaced by a small Redis store for a specific reason — see [Session Memory](05-session-memory.md) for the full story of why "no state at all" stopped being the right trade-off.

The one deliberate exception is that Redis-backed session memory. Everything else in the app remains genuinely stateless: kill the process, redeploy, or route the next request to a different data-center, and nothing breaks, because nothing was being remembered outside of GitHub, Redis (for that one thing), or the Telegram conversation itself.

## Layering

The codebase is organized so that **business logic never touches grammY, GitHub, or `process.env` directly** — every layer depends on an interface, not a concrete implementation:

```
route handlers (app/api/telegram/*)
        │
        ▼
   botInstance.ts   ← the ONLY place that reads process.env and constructs real clients
        │
        ▼
      bot.ts        ← wires handlers to grammY events; the only place that imports grammY types
        │
        ▼
  adapter.ts (adaptContext) ← translates a real grammY Context into a plain BotContext
        │
        ▼
     handlers/*     ← business logic; programs against BotContext + ContentProvider + ContentWriterLike
```

`BotContext` (`src/telegram/types.ts`) is a small, plain interface — `sendMessage`, `updateMessage`, `answerCallbackQuery`, `userId`, `messageText`, `replyToMessageText`, `callbackMessageText`, and so on. No handler ever imports anything from `grammy`. This buys two things:

1. **Tests never touch the network.** A `createFakeBotContext()` helper (`src/telegram/testHelpers.ts`) builds a plain object satisfying the interface, records what was sent, and lets a test assert on exact message text and keyboard contents — no HTTP, no grammY internals, no flakiness.
2. **grammY could be swapped out** without touching a single handler, if it ever needed to be.

The same pattern applies to content access (`ContentProvider` / `ContentWriterLike` interfaces — see [Content & Privacy](03-content-and-privacy.md)) and to the Groq client (`Pick<GroqClient, "createChatCompletion">` rather than the concrete class, so a test fake never needs to be a real `GroqClient` instance).

## The two webhook routes

```
POST /api/telegram/webhook   ← Telegram calls this on every update (message, callback, etc.)
PUT  /api/telegram/setup     ← operator-only: (re)configures Telegram's webhook URL + registers the command menu
```

`TELEGRAM_WEBHOOK_SECRET` (sent by Telegram on every request, verified by the webhook route) is deliberately a *separate* secret from `TELEGRAM_SETUP_SECRET` (used by the operator to call `/setup`) — one is inbound-from-Telegram, the other is outbound-from-you; conflating them would mean a leaked webhook secret also grants the ability to redirect the bot's webhook elsewhere.

That verification (`secureCompare`, `src/lib/secureCompare.ts`) is constant-time on purpose — comparing an attacker-controlled header value against a real secret with `===` or `string.includes` leaks the secret one byte at a time through response-time differences. It SHA-256-hashes both sides first, then compares the fixed-length digests with Node's `crypto.timingSafeEqual` — hashing first sidesteps `timingSafeEqual`'s own requirement that both buffers be equal length (unequal-length secrets would otherwise throw, or — if length were checked first as a shortcut — leak the correct length through timing anyway).

`getInitializedBot()` / `getBotApi()` (`botInstance.ts`) hold a module-scope singleton `Bot` instance, reused across warm invocations purely as a cold-start optimization — never a correctness dependency, since a fresh cold start rebuilds the exact same thing.

## Configuration as a single validated surface

Every environment variable the app depends on is declared once, in `src/lib/env.ts`, as a Zod schema — comma-separated lists (`ALLOWED_TELEGRAM_USER_IDS`, `EDITORS`, `PRIVATE_FOLDERS`) get parsed into typed arrays, optional-but-cross-validated fields (e.g. `GITHUB_TOKEN` becomes required the moment `EDITORS` is non-empty; `KV_REST_API_URL`/`KV_REST_API_TOKEN` must be set together or not at all) are checked with `superRefine`, and `getEnv()` lazily parses `process.env` once per warm instance and throws immediately on anything invalid — a misconfigured deploy fails loudly at first request rather than failing confusingly three layers deep.

```ts
// src/lib/env.ts (excerpt)
const envSchema = baseSchema.superRefine((value, ctx) => {
  if (value.EDITORS.length > 0) {
    // /save always writes to GitHub directly, regardless of CONTENT_PROVIDER
    // — so these become required the moment any editor is configured.
    for (const key of ["GITHUB_OWNER", "GITHUB_REPOSITORY"] as const) {
      if (!value[key]) ctx.addIssue({ code: "custom", path: [key], message: "is required when EDITORS is set" });
    }
  }
});
```

## Testing philosophy

The project leans hard on **dependency injection + fakes over mocks**. Concretely:

- Handlers take small interfaces (`ContentProvider`, `ContentWriterLike`, `SessionStore`, `Pick<GroqClient, "createChatCompletion">`) rather than concrete classes, so a test can hand-write a fake in a few lines instead of mocking a library.
- `createFakeContentProvider`, `createFakeContentWriter`, `createFakeSessionStore`, `createFakeBotContext` (all in `src/telegram/testHelpers.ts`) are the shared vocabulary every handler test is built from — they *record* calls (`writes`, `deletes`, `reverts`, `sendMessageCalls`) so tests assert on exactly what happened, not on internal implementation details.
- For anything touching the real GitHub API shape, `src/content/github/mockGitHubApi.ts` provides an in-memory fixture (`dir()`, `file()`, `createMockGitHubFetch()`) that speaks the actual Contents API response shape, so `GitHubContentProvider`/`GitHubContentWriter` are tested against realistic responses without a real network call.
- **The "offline diagnostic" technique**: for verifying real end-to-end behavior against production dependencies (a real Groq model's exact parameter acceptance, a real Redis instance, real GitHub auth) without spamming the real Telegram chat, the actual `createBot()` is constructed with real env vars loaded from `.env.local`, then `bot.api.config.use(async (_prev, method, payload) => {...})` intercepts every outgoing Telegram API call — so a synthetic update can be run through the *real* handler chain, hitting *real* Groq/GitHub/Redis, while every Telegram-bound side effect is captured and inspected instead of actually sent.

This combination is why the project could iterate quickly on genuinely risky changes (rewriting how `/save` decides where content goes, adding a multi-step reorganize flow that moves and deletes files) with high confidence — the test suite exercises real decision logic against realistic fakes, and the offline-diagnostic technique catches the class of bug that only shows up against a real external API.
