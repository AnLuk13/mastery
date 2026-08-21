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

## Command menu registration

`PUT /api/telegram/setup` calls `setMyCommands()` to register the `/` autocomplete menu (`BOT_COMMANDS` in `src/telegram/commands.ts`) — a step that has to be **re-run** any time a new top-level command is added, since Telegram doesn't infer the menu from what the bot happens to handle. This was the source of a real, if minor, bug: `/clear` and later `/save` were fully functional but invisible in Telegram's `/` menu for a while after being added, until `/setup` was called again with the updated list.
