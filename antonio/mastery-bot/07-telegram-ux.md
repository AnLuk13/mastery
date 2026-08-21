# Telegram UX

## Why grammY

[grammY](https://grammy.dev) was chosen over older alternatives (e.g. `node-telegram-bot-api`) for being fully TypeScript-typed and having a small, modern API surface — `bot.command()`, `bot.on("message:text", ...)`, `bot.on("callback_query:data", ...)`, `ctx.reply()`. Its footprint in the codebase is deliberately tiny, though — see the adapter pattern below.

## The `BotContext` adapter

grammY's own `Context` type is never imported outside `bot.ts` and `adapter.ts`. Every handler programs against `BotContext` (`src/telegram/types.ts`), a small plain interface:

```ts
export interface BotContext {
  readonly userId: number | undefined;
  readonly messageText: string | undefined;
  readonly replyToMessageText: string | undefined;
  readonly callbackMessageText: string | undefined; // text of the message a tapped button is attached to
  sendMessage(text: string, keyboard?: InlineKeyboard, parseMode?: ParseMode): Promise<void>;
  updateMessage(text: string, keyboard?: InlineKeyboard, parseMode?: ParseMode): Promise<void>;
  answerCallbackQuery(text?: string, showAlert?: boolean): Promise<void>;
  deleteMessages(fromMessageId: number, count: number): Promise<void>;
  sendTyping(): Promise<void>;
  // ...
}
```

`adaptContext()` (`adapter.ts`) is the one place that translates a real grammY `Context` into this shape — e.g. `updateMessage` tries `ctx.editMessageText()` when reached via a callback (falling back to a fresh reply if the message is too old to edit, or unchanged), while a plain command/message always sends fresh. This single adapter function is also where every best-effort behavior lives: `sendTyping()` swallows its own errors (never blocks the real reply on a failed typing indicator), `deleteMessages()` swallows per-message failures (a message already gone, or older than Telegram's 48h bot-delete window, shouldn't block the rest of a cleanup sweep).

The payoff (detailed further in [Architecture](02-architecture.md)) is that every handler is testable with a plain fake object — no grammY internals, no network, and a test can assert on exact recorded message text and keyboard contents.

### Which Telegram Bot API methods actually get called

grammY hides the raw HTTP, but every `BotContext` method still resolves to one specific [Telegram Bot API](https://core.telegram.org/bots/api) method underneath:

| `BotContext` method | Telegram Bot API method | Notes |
|---|---|---|
| `sendMessage` | `sendMessage` | Always used for a fresh message |
| `updateMessage` | `editMessageText`, falling back to `sendMessage` | Falls back if the message is too old to edit, or unchanged ("message is not modified") |
| `answerCallbackQuery` | `answerCallbackQuery` | With `show_alert` for the errors/toasts that need to interrupt (search-help, invalid-navigation), without it for the routine acknowledgment every callback handler does first |
| `sendTyping` | `sendChatAction` (`action: "typing"`) | Best-effort — failures are swallowed, never blocks the real reply |
| `deleteMessages` | `deleteMessage`, once per message ID | `/clear`'s sweep and document-overflow cleanup both loop this; each individual failure (already gone, too old) is swallowed rather than aborting the batch |
| `downloadDocument` | `getFile`, then a plain HTTPS `GET` | `getFile` (Bot API) resolves a `file_id` to a `file_path`; the actual bytes come from a *separate*, non-API endpoint: `https://api.telegram.org/file/bot<token>/<file_path>` |

Two more Bot API methods are called outside `BotContext`, directly against `bot.api`: **`getMe`** once per cold start (`bot.init()`, required before `handleUpdate` will work), and the four methods behind `/api/telegram/setup` below.

## Stateless navigation via `callback_data`

Every inline button encodes everything its handler needs directly into `callback_data` — no server-side lookup table, because any Vercel invocation might handle the tap, with nothing remembered from when the button was created.

```ts
// src/telegram/callbackData.ts — prefix scheme
"d:<path>[%<firstMsgId>+<count>]"   directory navigation (optional cleanup hint)
"f:<path>"                          open a document
"l:<remaining>-<limit>-<remaining>-<limit>"   Groq rate-limit snapshot (toast only)
"v:<path>%<beforeCommitSha>"        revert a specific write
"a"                                  save the tapped /ask answer
"y" / "n"                            confirm / decline a reorganize proposal
"s" / "x"                            search-help / path-too-long sentinels
```

Telegram caps `callback_data` at **64 bytes UTF-8** — `isCallbackDataTooLarge()` is checked on every encode, and anything that would exceed it degrades gracefully rather than silently breaking: a too-long document path falls back to a dedicated "too long" sentinel button instead of a truncated (and potentially unsafe) path; a too-long cleanup hint is simply dropped, leaving navigation working but without automatic cleanup of stale messages.

`decodeCallbackData()` never trusts its input — an unrecognized prefix, a malformed cleanup hint, or a path that fails `normalizeRelativePath()` (traversal attempts, absolute paths, null bytes) all decode to `{ type: "invalid" }` rather than throwing or guessing.

### The cleanup-hint trick

Opening a document that overflows one Telegram message edits the current message for the first chunk, then sends the rest as new messages — leaving extra messages behind that a later "Back"/"Home" tap should ideally clean up. Rather than tracking that server-side, the *count and starting message ID* of those extra messages ride along in the Back/Home button's own `callback_data` (`%<firstMessageId>+<count>`) — a later, entirely unrelated invocation can delete that whole run before showing the menu again, with zero state kept anywhere between the two taps.

## Markdown → Telegram-HTML rendering

Telegram's `parse_mode: "HTML"` only supports a narrow subset of tags (`<b>`, `<i>`, `<code>`, `<pre>`, `<a>`, `<blockquote>`, `<blockquote expandable>`). Rather than showing raw Markdown, `src/telegram/formatting/` parses real Markdown (via `markdown-it`) into a small `Block` union (`heading`, `paragraph`, `list`, `blockquote`, `code`, `divider`) and renders each into that supported subset — headings become `<b>...</b>`, ordered/unordered lists (including nested ones) render as indented bullet/number text, fenced code blocks become `<pre><code class="language-x">`.

Raw HTML embedded in source Markdown is **never** passed through as trusted markup — `html_block` tokens render as plain escaped text instead, so a note (or a model-generated answer) can't accidentally inject Telegram markup it didn't intend.

```ts
// src/telegram/formatting/blocks.ts
case "html_block":
  blocks.push({ kind: "paragraph", html: escapeHtml(token.content.trim()) });
```

### Message splitting

`assembleMessages()` joins rendered blocks into as few Telegram messages as fit under the 4096-char limit, preferring to break **between** blocks (paragraph/list/code-block boundaries) and only splitting *inside* a single block when that block alone is too long — a code block splits at line boundaries (never mid-line if avoidable, re-opening `<pre><code>` per resulting chunk since tags can't span independent messages), and a single line still too long is hard-sliced by Unicode code point, always *after* HTML-escaping is accounted for so a slice point can never land inside a produced entity like `&amp;`.

This exact pipeline is shared between opening a document from the knowledge base and rendering an `/ask` answer (see [RAG & /ask](04-rag-ask.md)) — one implementation, two call sites.

## Root-navigation UX

Early on, opening the bot always showed the top-level `mastery`/`antonio` folder structure, requiring an extra tap through a folder that (for a given user) usually had exactly one thing in it. `renderDirectory()`'s root-collapse logic now walks straight through any level that has exactly one visible entry, up to a bounded depth (`MAX_ROOT_COLLAPSE_DEPTH`), landing the user directly on their real content instead of a chain of single-item menus.

## `/api/telegram/setup`: one route, four HTTP methods, four Bot API calls

This is the operator-only route (never called by Telegram or by end users) for standing the bot up and inspecting its webhook state. Every method requires an `X-Setup-Secret` header matching `TELEGRAM_SETUP_SECRET` — deliberately a *different* secret from `TELEGRAM_WEBHOOK_SECRET` (see [Architecture](02-architecture.md)) — and maps 1:1 to a Bot API call via `src/telegram/setupHandler.ts`:

| HTTP method | Bot API method | Purpose |
|---|---|---|
| `POST` (body: `{ "url": "https://..." }`) | `setWebhook` | Points Telegram at `/api/telegram/webhook`, with `TELEGRAM_WEBHOOK_SECRET` registered as the secret token Telegram must echo back on every update |
| `DELETE` | `deleteWebhook` | Stops Telegram from delivering updates at all — used to take the bot offline deliberately |
| `PUT` | `setMyCommands` | Registers the `/` autocomplete menu (`BOT_COMMANDS` in `src/telegram/commands.ts`) |
| `GET` | `getWebhookInfo` | Reports the currently registered webhook URL, pending update count, and the last delivery error Telegram recorded — the same data `curl`ed directly against Telegram's API during the diagnostic in [Lessons & Bugs](08-lessons-and-bugs.md) |

`PUT` (command registration) has to be **re-run** any time a new top-level command is added, since Telegram doesn't infer the menu from what the bot happens to handle. This was the source of a real, if minor, bug: `/clear` and later `/save` were fully functional but invisible in Telegram's `/` menu for a while after being added, until `/setup` was called again with the updated list.
