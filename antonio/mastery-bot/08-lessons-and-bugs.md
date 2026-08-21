# Lessons & Bugs

A log of the sharper bugs and reversals from building this — the ones worth remembering specifically *because* they weren't obvious in advance. Several are covered in depth in their own topic file; this page is the index plus the ones that don't fit anywhere else.

## Covered in depth elsewhere

| Lesson | Where |
|---|---|
| `<tg-spoiler>` blurs text but doesn't collapse layout — `<blockquote expandable>` actually does | [Session Memory](05-session-memory.md) |
| Reply-threading solved "context carries over when you reply," not "just keep chatting" — a real pivot to Redis-backed ambient memory | [Session Memory](05-session-memory.md) |
| A document split across multiple Telegram messages broke reply-based context recovery; solved by not depending on replies at all | [Session Memory](05-session-memory.md) |
| `revert()` never handled undoing a *delete* correctly — treated "currently missing" as "already reverted" | [Content & Privacy](03-content-and-privacy.md), below |
| A cold-start editor with no existing files got a new note saved flat instead of into a topic folder | [Save & Authoring](06-save-authoring.md) |
| A plain-text "edit" request went through `/ask` (which never writes) and looked saved when it wasn't | [Save & Authoring](06-save-authoring.md) |
| `groq/compound-mini` rejects `reasoning_effort`; `openai/gpt-oss-120b` needs it or risks an empty completion | [RAG & /ask](04-rag-ask.md) |

## Vercel's `KV_REST_API_*` naming, not `UPSTASH_REDIS_REST_*`

Vercel's "Upstash for Redis" marketplace integration injects env vars named `KV_REST_API_URL` / `KV_REST_API_TOKEN` (a legacy of Vercel's own now-retired first-party KV product), which is *not* what Upstash's own SDK documentation leads you to expect from a manual setup (`UPSTASH_REDIS_REST_URL` / `UPSTASH_REDIS_REST_TOKEN`). The fix was simple once discovered — match the schema to what the actual project's env vars were — but it's a reminder that platform-integration env var names are worth confirming against the real dashboard, not assumed from a library's generic docs. See [Session Memory](05-session-memory.md).

## The `revert()`-and-delete gap, in more detail

`GitHubContentWriter.revert()` predates `delete()` by a long way — it was written when the only thing to revert was ever a *write* (a create or an update), so "the path doesn't currently exist" could only mean "already gone, nothing to do," and the code returned early in that case. Adding `delete()` (for the reorganize feature) quietly broke that assumption: reverting a delete means the path is *expected* to be currently missing, and the early-return turned "undo this delete" into a silent no-op.

The fix separates two independently-tracked facts — *did the path exist before the commit being reverted* (`priorContent`) and *does it exist right now* (`currentSha`) — and only short-circuits when both agree there's truly nothing to do:

```ts
if (priorContent === undefined) {
  if (currentSha !== undefined) await this.client.deleteFile(...); // else: already gone
} else {
  // existed before — restore it, whether that's an update (currentSha present)
  // or a recreate (currentSha undefined — GitHub rejects a create that passes a sha)
  await this.client.createOrUpdateFile(githubPath, priorContent, message, this.branch, currentSha);
}
```

This is a good example of a latent bug that unit tests didn't catch earlier for a mundane reason: nothing in the test suite had ever exercised "revert something that was deleted," because nothing in the app could delete anything until the reorganize feature needed it. The bug was only found because a *new* test for that exact scenario was written alongside the new capability — a reminder that a green test suite only proves what it actually tests.

## `ALLOWED_TELEGRAM_USER_IDS` initially had one ID

When the bot grew from a single-user tool into a four-person one, `ALLOWED_TELEGRAM_USER_IDS` in production still only listed the original user — a second user genuinely couldn't get past the authorization gate at all. This was caught via the "offline diagnostic" technique (see [Architecture](02-architecture.md)): constructing the real bot against real production env vars and running a synthetic update through it showed the access-denied response immediately, without needing the second user to hit it live first.

## Zod's `discriminatedUnion` requires a unique discriminator *value*, not just a unique shape

An early draft of `/save`'s decision schema tried two `write` branches distinguished only by an `isNewFile` boolean — both still keyed `action: "write"`. Zod's `discriminatedUnion` rejects this at schema-construction time ("duplicate discriminator value") because it indexes branches strictly by the literal value of the discriminator field, not by trying each branch's full shape. The fix collapsed the two into one `write` branch with `content` made optional, moving the "content is required exactly when `isNewFile` is true" correlation check into `decideSave()` itself, after parsing — a schema can enforce shape, but a cross-field business rule like that has to live in code.

## The ONNX runtime native binary vanished from the Vercel deploy

`@huggingface/transformers` pulls in `onnxruntime-node`, whose actual inference engine is a native binary (`libonnxruntime.so.1`) loaded via `dlopen` at the OS level — invisible to Vercel's static file tracer, which only follows `require`/`import` graphs. The result: the function deployed successfully, imported fine, and then threw `libonnxruntime.so.1: cannot open shared object file` the moment it actually tried to embed something, in production only (nothing about this could reproduce locally, where the file is just present on disk). The fix is an explicit `outputFileTracingIncludes` entry in `next.config.ts` forcing the binary to be bundled for both webhook routes — `/api/telegram/setup` needed it too, not just `/api/telegram/webhook`, because constructing the bot for *either* route transitively imports the embedding model even though `/setup` never actually calls it at runtime.

## `llama-3.3-70b-versatile` quietly stopped existing on Groq

A model referenced by name early in the project was later removed from Groq's catalog entirely — the first sign was a webhook call returning an error where none had before, with no code change on this side. Groq model names aren't a stable, versioned contract the way a pinned package version is; the practical lesson was to treat the configured model name as something to periodically re-verify against Groq's live catalog, not a fact to set once and forget. `GROQ_MODEL`/`GROQ_ASK_MODEL` being separate, overridable env vars (rather than hardcoded constants) means a future catalog change doesn't require a code deploy to route around — just an env var update.
