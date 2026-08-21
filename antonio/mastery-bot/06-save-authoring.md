# Save & Authoring

`/save` turns a typed note, a forwarded message, or an uploaded `.txt`/`.md` file into a committed Markdown file in the caller's own folder — with an AI deciding *where* it belongs, not just writing it verbatim.

## Design principles, set early and still true

- **Explicit trigger, not free text.** Unlike `/ask` (any plain message), `/save` requires the literal command — specifically so a plain message can safely default to "this is a question" without ever risking an unintended write. The one exception is replying to a *clarify* prompt (see below), which is unambiguous by construction.
- **Commit immediately, not confirm-first — with two exceptions.** Every write commits directly to GitHub as soon as the AI decides where it belongs, backed by an easy, one-tap **Revert** button rather than a "are you sure?" step beforehand. This was a deliberate, explicit choice: friction on every save was judged worse than the (cheap, reversible) cost of an occasional bad guess. **Reorganize** (below) and a **multi-file delete** are the deliberate exceptions — one touches a file *other* than the one just asked about, the other removes several files in one request — both bigger blast radius than any single write, so both ask first, on a genuinely fresh request. Neither exception applies once the user has already answered one clarifying round — see "Two rounds, then decisive" below.
- **Per-editor folder confinement.** Every path a model proposes is re-validated to start with `findEditorFolder(userId, editors)` before anything is written — see [Content & Privacy](03-content-and-privacy.md).

## The decision engine: `decideSave()`

```ts
// src/authoring/decideSave.ts — the model returns one of:
{"action":"clarify","questions":["..."]}
{"action":"write","path":"...","isNewFile":true,"content":"...","commitMessage":"..."}
{"action":"write","path":"...","isNewFile":false,"commitMessage":"..."}
{"action":"reorganize","moveFrom":"...","moveTo":"...","newPath":"...","content":"...","commitMessage":"..."}
{"action":"delete","paths":["..."],"commitMessage":"..."}
```

Given the request text and a listing of the editor's existing document paths, the model decides:

1. **Merge into an existing file** (`isNewFile: false`) — if the request fits an existing file's specific topic. Content isn't included here; a separate `composeUpdate()` call handles the actual merge once the *current* file content is fetched (see below).
2. **Create a new file** (`isNewFile: true`) — with a topic subfolder required: `<editorFolder>/<topic>/<file>.md`, never flat directly under the editor's folder. If existing documents establish a topic structure already, the model is instructed to follow it rather than invent a near-duplicate; if there are none yet, it's instructed to choose a sensible topic itself.
3. **Propose a reorganize** — when a genuinely new note's topic overlaps with an existing *flat* file (see below).
4. **Delete** one or more existing documents — a specific file, or every document under a folder the request named (see "Deleting content" below).
5. **Ask a clarifying question** — when it genuinely can't tell, capped at one retry.

### Two rounds, then decisive

`decideSystemPrompt()` tells the model that on a second attempt (`clarifyRound > 0` — the user has already answered one clarifying question) it *must* return `write`, `reorganize`, or `delete` — never `clarify` again, so a request can never loop forever on ambiguity. What changed since that cap was first added: **`reorganize` and a multi-file `delete` also execute immediately on that second attempt**, rather than still stopping to propose+confirm. The reasoning: normally proposing first is what makes those two actions safe despite their bigger blast radius, but once the user has already answered a clarifying question, that reply *is* their go-ahead — asking `Yes/No` on top of it would just be a second confirmation step right when the save should happen. This was a direct fix for a reported bug: a reorganize proposal appearing *after* the user had already answered a clarifying question read as "the bot replied with a restructuring plan instead of just saving."

### Never trust a model-generated path

Every path the model returns — for `write`, all three `reorganize` fields, and every entry in a `delete` decision's `paths` — is re-validated with the same rigor as any other untrusted path in the app (see [Content & Privacy](03-content-and-privacy.md)): normalized, checked to still start with the editor's folder *after* normalization (defeats a traversal like `"antonio/../other/x.md"` that would pass a naive string-prefix check), and checked for a `.md` extension. A validation failure throws `GroqUnavailableError` — a hard failure surfaced as a generic "couldn't process that save" message, never a write to a path that wasn't independently verified safe.

## `composeUpdate()`: merging, not overwriting

```ts
const COMPOSE_SYSTEM_PROMPT = `You are updating an existing file... Merge the new material in naturally
(a new section, or extending an existing one) without discarding anything already there unless the
new material clearly supersedes it. Preserve the existing heading style.`;
```

A separate model call from `decideSave()` — by the time this runs, the file's *actual current content* has been fetched (existing content is never assumed; a `ContentNotFoundError` during that fetch is treated as "starts empty," any other error propagates), so the merge is grounded in what's really there, not in what the model guessed might be there.

## The topic-subfolder bug, found in real use

The first real editor to use `/save` (with an empty folder — nothing yet to pattern-match against) got a file written flat, `andreea/meeting.md`, instead of into a topic subfolder — the prompt *mentioned* topic subfolders, but with zero existing entries to imitate, the model took the easier flat path. The fix was two-layered, following the project's general habit of never relying on a prompt alone for something code can guarantee:

1. **Prompt**: made explicit that a new file *always* needs a topic subfolder, even from a cold start — "a knowledge base organized by subject is the whole point."
2. **Code**: `decideSave()` now rejects a `write` decision with `isNewFile: true` whose path has no subfolder segment beyond the editor's own folder, regardless of what the model claims — the same "never trust the model" posture as path-traversal validation.

## Revert

Every successful write's result carries a `beforeCommitSha` (see [Content & Privacy](03-content-and-privacy.md)), encoded directly into a **Revert** button's `callback_data` (`v:<path>%<sha>`) — no server-side record of "what was this write" is needed; the button itself carries everything `GitHubContentWriter.revert()` needs to undo exactly that one change.

## The 💾 "Save this" button — closing a real UX gap

A user asked the bot (in plain chat, no `/save`) to add an agenda to a meeting note it had just created. The bot answered fluently — a plausible-looking "edited" version of the note — but because it was a plain message, it went through `/ask`, which never writes anything, by design. The user reasonably assumed it had been saved; it hadn't, and nothing indicated that.

Rather than blurring the `/ask` vs `/save` line (which would undermine the very reason `/save` requires an explicit trigger), the fix adds a **💾 Save this** button to every `/ask` answer, shown only to registered editors: one tap runs the *exact same* decide-and-write flow, sourced from that answer's own text. This required a small new piece of plumbing — `BotContext.callbackMessageText`, the text of the message a tapped button is attached to (distinct from `replyToMessageText`, since a button tap isn't a reply) — and a no-payload callback (`SAVE_ANSWER_CALLBACK_DATA`) whose handler re-validates the caller's editor status server-side regardless of how the tap arrived.

## The reorganize-proposal flow

The topic-subfolder fix only governs *new* files — it says nothing about a file that was already saved flat before the fix existed (like the original `meeting.md`). Grouping that old file together with a newly-related note means **moving and deleting content the user didn't just ask about** — a meaningfully bigger blast radius than "write the one file I asked for," so this is the one flow in `/save` that asks first.

```
You: /save add a note about tomorrow's sales call

Bot: 📁 This looks related to your existing
     andreea/meeting.md — want me to group
     them under andreea/meetings/?

     [✅ Yes, reorganize]  [❌ No, keep separate]
```

**Nothing executes on a first-attempt proposal** — `decideSave()`'s `reorganize` action is a proposal; `performSave()` sends the confirmation message and returns (unless this is a second attempt after a clarifying round — see "Two rounds, then decisive" above — in which case it's applied directly instead). The pending proposal (moveFrom/moveTo/newPath/content/commitMessage) rides along as plain JSON inside the confirmation message's own text, behind a marker string, the same no-server-state trick the old clarify flow used — except recovered via `callbackMessageText` (a button tap) rather than `replyToMessageText` (a reply):

```ts
// src/telegram/userMessages.ts
export function formatReorganizePrompt(proposal: ReorganizeProposal): string {
  return `📁 This looks related to your existing ${proposal.moveFrom}...\n\n${REORGANIZE_MARKER}\n${JSON.stringify(proposal)}`;
}
```

On **Yes**, `createReorganizeConfirmHandler` re-validates every one of the three paths against the *caller's own* folder (never trusting the echoed JSON alone — it round-tripped through a Telegram message and is treated as untrusted input, same as `callback_data`), then runs three operations in sequence: write the new note, copy the existing file's current content to its new location, delete the old path. Each operation independently returns its own `beforeCommitSha`, so the result carries **three separate Revert buttons** — undoing all three restores exactly the pre-reorganize state, reusing the *same* generic single-path `revert()` mechanism as an ordinary save, unchanged. On **No**, only the new note is written, at the topic-folder path it was always going to use; the old flat file is left untouched.

This flow is also what surfaced the `revert()`-never-handled-a-delete bug described in [Content & Privacy](03-content-and-privacy.md) and [Lessons & Bugs](08-lessons-and-bugs.md) — a latent gap in existing code that a genuinely new code path (deleting something and expecting to be able to undo it) was needed to expose.

## Deleting content

There's no `/delete` command — like everything else in `/save`, deletion is triggered by natural language through the same command: `/save delete andreea/meeting.md`, or `/save delete everything in the old-notes folder`. `decideSave()` recognizes delete intent the same way it recognizes save intent, resolving a folder reference into the actual list of existing document paths under it (from the same `existingEntries` listing used for topic-matching) rather than the model inventing paths — every returned path is still independently re-validated to be an exact existing document within the editor's folder, same as everywhere else.

**Blast radius decides whether it asks first**, mirroring reorganize's reasoning exactly:

- **A single file deletes immediately** — same immediate-commit-plus-Revert pattern as any other save action. One file disappearing is no bigger a risk than any other single write.
- **Two or more files at once ask for confirmation first**, on a fresh (first-attempt) request:

```
You: /save delete everything in the old-notes folder

Bot: 🗑️ Delete 3 files?

     • andreea/old-notes/one.md
     • andreea/old-notes/two.md
     • andreea/old-notes/three.md

     [🗑️ Yes, delete]  [❌ No, keep them]
```

- On a **second attempt** (after the user already answered a clarifying question), a multi-file delete executes directly instead — same "Two rounds, then decisive" reasoning as reorganize.

The proposal itself rides in the confirmation message's own text behind a `⎯⎯⎯ delete-proposal (do not edit) ⎯⎯⎯` marker (`formatDeleteConfirmPrompt`/`extractDeleteProposal` in `userMessages.ts`) — the same no-server-state trick as reorganize, just with its own marker string and its own confirm/decline callback sentinels (`DELETE_CONFIRM_CALLBACK_DATA`/`DELETE_DECLINE_CALLBACK_DATA`, distinct from reorganize's `y`/`n`) so a tap can tell which kind of proposal it's confirming.

Executing a delete (`executeDelete` in `save.ts`) removes each path as its own commit, in sequence, and the result carries **one independent Revert button per file** — undoing any subset restores just those files, reusing the exact same generic `revert()` mechanism as every other write in the app (see [Content & Privacy](03-content-and-privacy.md) for why reverting a delete specifically needed a bug fix first).
