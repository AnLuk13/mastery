# 10. Model Context Protocol (MCP)

Goal: the problem MCP solves is easy to miss until you've felt the pain it fixes — you actually watched it in action this session (the Vercel MCP connector during `mastery-bot`'s deployment), so this chapter names what you already saw.

## 10.1 The problem: every tool integration was a one-off

Chapter 8 showed you how to define tools for a model — a JSON schema plus your own code to execute them. That works, but at scale it doesn't compose: if you want an agent harness to work with GitHub, Vercel, a database, and a filesystem, someone has to hand-write a tool integration for each one, in every harness that wants to use it. **N** harnesses wanting **M** tools/data sources means roughly **N × M** custom integrations, each maintained separately, each with its own auth handling, its own quirks.

**MCP** (Model Context Protocol, introduced by Anthropic and since adopted broadly) standardizes this: instead of every harness inventing its own way to talk to GitHub, an **MCP server** for GitHub exposes its tools/data over a common protocol, and **any** MCP-compatible harness can use it with no GitHub-specific code of its own. This turns the integration problem from N×M into N + M — write one server per tool, one client implementation per harness, and everything interoperates.

## 10.2 Client/server, mapped onto what you actually did this session

MCP is explicitly a client/server protocol, and you drove both roles hands-on during Stage 9:

- **MCP server** — a separate process/service exposing a set of capabilities (tools to call, sometimes data resources to read) over the protocol. The `claude.ai Vercel` MCP server you connected via `/mcp` is exactly this — a server, run by/for Vercel, exposing "deploy a project," "manage environment variables," etc. as MCP tools.
- **MCP client** — the harness that connects to a server and makes its tools available to the model. Claude Code is the client here — once you authenticated the Vercel connector, its tools became available to the agent loop from chapter 9, indistinguishable in kind from Claude Code's own built-in tools (`Bash`, `Read`, etc.) from the model's point of view.
- **Tool discovery** — a client doesn't need to know a server's tools in advance; it asks the server what's available (a live version of chapter 8's static tool list), which is *why* `mcp__claude_ai_Vercel__authenticate`/`mcp__claude_ai_Vercel__complete_authentication` showed up as real, callable tools the moment that server was reachable, with no code change to Claude Code itself.

You also have other MCP servers already available in this environment — `gitnexus`/`graphify`'s tools (`mcp__gitnexus__*`) — same protocol, different server, exposing code-graph queries instead of deployment actions. The consistent naming pattern (`mcp__<server>__<tool>`) you've seen throughout this session is the client's own bookkeeping for "which server does this tool belong to," not a coincidence.

## 10.3 Why this matters for anything you build yourself

If you build an agent-style feature (chapter 9) — for `mastery-bot`, for HRNS, or standalone — and it needs to talk to an external system, you face the same choice MCP resolves generally:

- **Hand-write a tool integration** (chapter 8) — completely reasonable for a small, fixed number of one-off integrations specific to your app; this is what §8.4's `search_knowledge_base`/`get_document` tools are, and there's no need to reach for MCP just to wrap two functions.
- **Expose your own capability as an MCP server** — worth doing once you want *multiple different clients/harnesses* (not just your one app) to be able to use the same capability, or once you're integrating with an ecosystem that already expects MCP. A `mastery-bot`-as-MCP-server is a genuinely plausible extension: expose `search_knowledge_base`/`get_document` (§8.4) as an MCP server, and *any* MCP-compatible agent (not just a bot you build yourself) could query your knowledge base — the same shift from "one-off integration" to "reusable interface" that a public REST API represents for ordinary web services, just standardized specifically for model-facing tools.

## 10.4 MCP resources vs. MCP tools — two different capabilities

MCP distinguishes between **tools** (actions the model can invoke, exactly like chapter 8 — "run this, get a result") and **resources** (data the client can read and hand to the model as context, closer to chapter 6's retrieval step than to chapter 8's tool calling). `ReadMcpResourceTool`/`ListMcpResourcesTool` in this environment are exactly the resource side of this — reading data a server exposes, rather than invoking an action on it. Recognizing which of the two a given MCP capability is (act vs. read) maps directly onto whether you're looking at chapter 8's world or chapter 6's world underneath the shared protocol.

## Checkpoint

1. Before MCP, if you wanted both Claude Code *and* a separate custom agent you built yourself to both be able to deploy to Vercel, what would you have had to build twice? What does MCP let you build once instead?
2. Classify each of the following as more tool-like (action) or resource-like (data to read): `setWebhook` on the Telegram Bot API; `getWebhookInfo`; `ContentProvider.search()`; `ContentProvider.getDocument()`.
3. You're asked whether a small internal company tool should be built as "just a script the AI agent shells out to" (chapter 8-style) or "a proper MCP server." What's the deciding factor, based on §10.3?
