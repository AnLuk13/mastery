# Overview

## What it is

**Mastery Bot** (Telegram: `wiki_master_knowledge_bot`) is a private, four-user Telegram bot that turns a folder of personal Markdown notes into:

- A **browsable knowledge base** — navigate folders, open documents, rendered with proper formatting instead of raw Markdown.
- A **search tool** — keyword search across every document you're allowed to see.
- An **AI chat interface** (`/ask`, or just any plain message) — ask questions grounded in your notes, with live web access for anything the notes don't cover, and memory that carries across a normal back-and-forth conversation.
- A **chat-driven authoring tool** (`/save`) — turn a typed note, a forwarded message, or an uploaded file into a properly organized, committed Markdown file, with one-tap revert if the AI got it wrong.

It is deliberately **not** a general-purpose product — it's scoped tightly to a handful of known users, each with their own private folder, and the AI's authority to act (write files, move files) is bounded and always reversible.

## The three moving pieces

The system is split across three independent things, and understanding that split explains most of the architecture:

```
┌─────────────────┐        ┌──────────────────┐        ┌─────────────────┐
│  mastery         │  read/  │  mastery-bot      │  HTTPS  │   Telegram       │
│  (GitHub repo)   │  write  │  (Next.js app,    │webhook  │   (mobile UI)    │
│                  │◄───────►│   Vercel)         │◄───────►│                  │
│  Markdown files, │  REST   │  Stateless        │         │  Where the       │
│  the actual      │  API    │  serverless       │         │  human is        │
│  knowledge base  │         │  functions        │         │                  │
└─────────────────┘        └──────────────────┘        └─────────────────┘
```

1. **`mastery`** — a separate GitHub repository that holds nothing but the Markdown content itself, organized per-editor: `antonio/`, `andreea/`, `anatol/`, `ana/`, each further organized into topic subfolders (`antonio/networking-mastery/`, `andreea/meetings/`, etc.). This repo is the actual source of truth — if the bot and Vercel disappeared tomorrow, the notes would still exist as ordinary files in ordinary Git history.

2. **`mastery-bot`** — the application itself: a Next.js app deployed to Vercel as serverless functions. It has **no database and no durable storage of its own** (aside from one narrow exception — see [Session Memory](05-session-memory.md)). Every request reads from or writes to the `mastery` repo via GitHub's REST API, and any bot-side "memory" is either encoded in the Telegram conversation itself or, more recently, in a small Redis store.

3. **Telegram** — the UI. No custom frontend was built; Telegram's own message bubbles, inline keyboards, and reply-threading *are* the UI, which is what makes the "stateless serverless" approach viable at all (see [Architecture](02-architecture.md)).

## Why this split

- **Content survives the app.** Notes are just Markdown in Git — readable, diffable, editable outside the bot entirely, and never at risk of being lost to a bad deploy or a Vercel outage.
- **The app can be genuinely stateless.** Vercel functions are ephemeral — nothing persists between invocations by default. Rather than fighting that with a database from day one, the design leaned into it: GitHub *is* the database for content, and Telegram's own message IDs and reply-threading became a database substitute for "where was I in this conversation." (This trade-off was later revisited for conversational memory specifically — see [Session Memory](05-session-memory.md) for why.)
- **Multi-user privacy falls out of the folder structure.** Each editor's content lives under their own top-level folder, and a single `PRIVATE_FOLDERS` mechanism enforces per-folder visibility uniformly across browsing, search, and AI chat (see [Content & Privacy](03-content-and-privacy.md)).

## Full feature list

| Feature | Command / trigger | Covered in |
|---|---|---|
| Browse folders/documents | `/start`, inline navigation buttons | [Content & Privacy](03-content-and-privacy.md) |
| Keyword search | `/search <query>` | [Telegram UX](07-telegram-ux.md) |
| AI chat, grounded in notes + live web | Any plain-text message | [RAG & /ask](04-rag-ask.md) |
| Ambient conversation memory | Automatic, no reply needed | [Session Memory](05-session-memory.md) |
| Save a note / upload a file | `/save <text>`, file upload, reply-to-save, or the 💾 button on an /ask answer | [Save & Authoring](06-save-authoring.md) |
| Auto-organize into topic folders | Automatic (new files), or proposed + confirmed (existing flat files) | [Save & Authoring](06-save-authoring.md) |
| One-tap revert of any write | ↩️ button on save/reorganize results | [Save & Authoring](06-save-authoring.md) |
| Per-folder read privacy | `PRIVATE_FOLDERS` config | [Content & Privacy](03-content-and-privacy.md) |
| Clear chat clutter + memory | `/clear` | [Session Memory](05-session-memory.md) |

## Tech stack, in more detail

See the README's stack table for the one-line version. The sections that follow go deep on each choice — what problem it was solving, what was tried or considered instead, and (where relevant) what broke in production and had to be fixed.
