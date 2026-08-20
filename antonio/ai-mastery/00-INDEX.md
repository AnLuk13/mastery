# AI, LLMs & Applied ML Mastery — from zero to "I can build this into a product"

This is a self-paced curriculum for one specific goal: take you from "I don't really know what an LLM *is*" to genuinely understanding how modern AI systems work under the hood, and — more importantly — how to actually put that knowledge to use building AI features into real web and mobile applications.

You're not starting from zero on software engineering — you know TypeScript, C#, databases, HTTP APIs (see `../dotnet-mastery/` and `../networking-mastery/`). You *are* starting from zero on AI/ML specifically, so this guide never assumes prior ML background. Every new concept is explained from first principles and, wherever one exists, tied back to something you already know from ordinary programming.

## How this is organized

Every chapter follows roughly the same shape:
1. **Theory** — the concept explained from first principles, with a plain-programming analogy where one helps.
2. **Minimal runnable example** — small, concrete, usually TypeScript (your primary stack) or plain pseudocode.
3. **Grounded example** — a real tie-in wherever an honest one exists: your `HRNS.Platform.Server` AI Assistant (already documented in [`../dotnet-mastery/13-ai-ml-integrations.md`](../dotnet-mastery/13-ai-ml-integrations.md) — cross-linked here, not repeated), the `mastery-bot` project you just built this session (used throughout as a sandbox for "how would I add this feature here"), or Ollama, which is already installed on this machine.
4. **Checkpoint** — a few questions or a small exercise before moving on.

Nothing here is fabricated filler. Where a chapter references your HRNS codebase, it either points at facts already confirmed in `13-ai-ml-integrations.md`, or poses it as something to go verify yourself — never an invented file/line citation you can't check.

## The one picture to hold in your head the whole way through

```
YOUR APP  ──prompt──▶  MODEL  ──tokens──▶  YOUR APP
   │                     ▲
   │                     │ (optional, this is most of the curriculum)
   └──context──▶  RETRIEVAL (embeddings + vector search over YOUR data)
                         ▲
                         │
                    TOOLS / FUNCTIONS  (the model asks your app to do things)
```

A "chatbot" is the simplest possible version of this picture (just the top row). Everything from chapter 5 onward is about the bottom two rows — grounding the model in real data, and letting it act — because that's what turns "a chat window" into an actual product feature.

## Suggested path

**Part 1 — Foundations (read this even if you're impatient to build something)**
1. [Introduction & Mental Models](01-introduction-and-mental-models.md) — AI vs. ML vs. deep learning vs. LLM, why this exploded now, vocabulary you'll see everywhere, how to think about a model as a black-box function
2. [Machine Learning Fundamentals](02-machine-learning-fundamentals.md) — training vs. inference, supervised/unsupervised/reinforcement learning, what a neural network actually is, overfitting — just enough to make chapter 3 click
3. [How LLMs Actually Work](03-how-llms-work.md) — tokens, the transformer/attention mechanism (conceptually), context windows, pretraining → fine-tuning → RLHF, sampling parameters (temperature, top-p)
4. [Prompting & Prompt Engineering](04-prompting-and-prompt-engineering.md) — system/user/assistant roles, zero/few-shot, chain-of-thought, structured output, why prompts are fragile

**Part 2 — Grounding models in real data**
5. [Embeddings & Vector Search](05-embeddings-and-vector-search.md) — what an embedding actually is, similarity metrics, vector databases, cross-linked to HRNS's real pgvector setup
6. [Retrieval-Augmented Generation (RAG)](06-retrieval-augmented-generation.md) — the architecture that makes a model answer from *your* documents instead of guessing; grounded in HRNS's hybrid retriever and a worked "add RAG to `mastery-bot`" example
7. [Fine-Tuning & Customization](07-fine-tuning-and-customization.md) — fine-tuning vs. RAG vs. prompting, LoRA/PEFT conceptually, when fine-tuning is actually worth it (rarer than you'd think)

**Part 3 — Letting models act: tools, agents, harnesses**
8. [Function/Tool Calling](08-function-calling-and-tool-use.md) — how a model asks your code to do something, JSON schema tool definitions, a worked TypeScript example
9. [Agents & Harnesses](09-agents-and-harnesses.md) — the plan → act → observe loop, ReAct, grounded in Claude Code itself (the tool you're using right now) as a real agent harness
10. [Model Context Protocol (MCP)](10-model-context-protocol-mcp.md) — the standard for connecting models to tools/data, grounded in the Vercel/GitNexus/Graphify MCP tools already in this environment
11. [Orchestration Libraries](11-orchestration-libraries.md) — LangChain, LlamaIndex, Vercel AI SDK, Microsoft.Extensions.AI/Semantic Kernel — when a framework earns its keep vs. calling the API directly

**Part 4 — Models & infrastructure**
12. [Calling Model APIs](12-calling-model-apis.md) — OpenAI/Anthropic/Google request shapes, streaming, tokens & pricing, a real TypeScript `fetch` example
13. [Local Models & Ollama](13-local-models-and-ollama.md) — running models on your own machine (you already have Ollama installed), GGUF & quantization, hardware reality-check
14. [Cloud vs. Local Models](14-cloud-vs-local-models.md) — cost, latency, privacy, and capability tradeoffs, and a decision framework instead of a religious opinion
15. [The Model Landscape & How to Choose](15-model-landscape-and-selection.md) — model families, open vs. closed weights, how to evaluate a model for a task without memorizing numbers that go stale in a month

**Part 5 — Building it, and not shooting yourself in the foot**
16. [Building AI Features Into Apps](16-building-ai-features-in-apps.md) — streaming UI patterns, where inference should run (server vs. edge vs. on-device), a worked "add AI search to `mastery-bot`" sketch
17. [Safety, Security & Evaluation](17-safety-security-and-evaluation.md) — hallucination, prompt injection (this will feel very familiar after `mastery-bot`'s path-traversal work), data leakage, how to actually test an AI feature
18. [Further Topics — Appendix](18-further-topics-appendix.md) — multimodal models, speech, RLHF in more depth, mixture-of-experts, quantization internals — the "know it exists, go deeper only if you need to" tier

[Glossary](GLOSSARY.md) — every acronym and term used above, one line each.

## Ground rules for using this guide

- Read a chapter, then go try the example yourself — in a scratch script, in Ollama, or against a real API key — before moving to the next one. This material does not stick from reading alone.
- Chapters 5–6 (embeddings, RAG) are the payoff chapters for "grounding a model in real data," the single most common thing you'll actually want to build. Chapters 1–4 exist to make them click the first time, not the third.
- Chapter 9 (agents/harnesses) is a good one to reread *after* you've used Claude Code a lot more — you'll recognize the pattern immediately once you've seen it from the inside.
- This field moves fast. Version numbers, pricing, and "best" models are deliberately avoided or hedged throughout — the durable content is *how to evaluate*, not a snapshot of today's leaderboard.
