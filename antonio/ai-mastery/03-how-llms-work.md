# 3. How LLMs Actually Work

Goal: tokens, attention, and the training pipeline — the mechanics behind the "model is a function" mental model from chapter 1, concrete enough that context windows, pricing, and hallucination all stop feeling mysterious.

## 3.1 Tokens — the actual unit a model reads and writes

A model never sees raw characters or whole words — it sees **tokens**, produced by a **tokenizer** that splits text into a few thousand to ~100k possible chunks, each mapped to an integer ID. A token is often a whole common word (`" the"`), sometimes a word-piece (`"token"` + `"ization"`), sometimes a single character (for rare words or code symbols). Rough rule of thumb for English: **~4 characters per token**, or ~¾ of a word per token — useful for quickly estimating cost/context usage without running an exact tokenizer.

```ts
// Conceptually — real tokenizers are trained (via an algorithm like BPE), not this naive:
tokenize("mastery-bot handles GitHub webhooks")
// → ["mastery", "-", "bot", " handles", " Git", "Hub", " web", "hooks"]  (illustrative, not exact)
```

Why this matters practically: pricing is per-token (chapter 12), context windows are measured in tokens not characters (§3.4), and *code and non-English text often tokenize less efficiently than English prose* — a real cost/context consideration if `mastery-bot` ever fed source code into a model.

## 3.2 The transformer, conceptually — attention over everything at once

Before 2017, the dominant architecture for text (RNNs) processed tokens strictly one at a time, in order, which made it hard to relate a word to another word far earlier in a long passage. The **transformer** architecture (the *T* in GPT) changed this with a mechanism called **self-attention**: for every token, the model computes how much it should "attend to" (weight the influence of) every *other* token in the input, simultaneously, regardless of distance.

Concretely, each token gets converted into three vectors — a **query** (what am I looking for), a **key** (what do I offer), and a **value** (what information do I actually contribute) — and attention is computed as: how well does this token's query match every other token's key, and use that match strength to weight a blend of everyone's values. Stack many attention layers, each refining the representation using the *previous* layer's output, and the model builds up increasingly rich, context-aware representations of every token — this is *why* the model can figure out that "it" refers to "the file" three sentences earlier, or that a variable name used at the top of a code block is relevant to a bug described at the bottom.

You do not need to derive the attention math to work with LLMs day-to-day — but "attention lets every token directly relate to every other token, weighted by learned relevance" is the one sentence worth actually retaining, because it explains *why* context window size (§3.4) matters so much: attention's cost grows with the square of the input length, which is the direct, mechanical reason very long contexts are expensive and were historically capped much lower than they are today.

## 3.3 The training pipeline: pretraining → fine-tuning → RLHF

A model like GPT-4o or Claude doesn't go from random weights to "helpful assistant" in one step — it's a pipeline, each stage a different flavor of learning from chapter 2:

1. **Pretraining** (supervised, at a scale nothing else in ML approaches) — train on a vast corpus of internet text/code/books with one deceptively simple objective: predict the next token, given everything before it. This alone produces a model that's very good at *continuing text plausibly* — but a raw pretrained model isn't an assistant, it's a very sophisticated autocomplete, just as likely to continue your question with more questions as to answer it.
2. **Instruction fine-tuning / SFT** (supervised) — further train on examples specifically formatted as `(instruction, ideal response)` pairs, teaching the model the assistant *behavior* — answering questions, following instructions, using the system/user/assistant format (chapter 4 §4.1) — rather than just continuing text.
3. **RLHF** (reinforcement learning, chapter 2 §2.1) — humans rank multiple model outputs for the same prompt; a separate "reward model" learns to predict those rankings; the main model is then further tuned via RL to produce outputs the reward model scores highly. This stage is largely responsible for a model *refusing* harmful requests, *hedging* on uncertain claims, and generally "sounding like" a helpful assistant rather than a raw text-completion engine.

The practical upshot: everything you interact with via an API or Ollama has already been through all three stages. **Fine-tuning** as *you* would do it (chapter 7) is a much smaller, much later, optional fourth step on top of an already fully-trained model — a completely different scale of effort than any of the above.

## 3.4 The context window — why it's the resource you'll manage most

The **context window** is the maximum number of tokens a model can process in a single call — input (system prompt + conversation history + any retrieved documents, chapter 6) plus output, combined. Exceeding it isn't a soft degradation — the call fails outright, or older content gets silently truncated depending on how your client library handles it.

This single limit is *why*:
- **"Conversation memory" gets expensive and eventually breaks** — since each call resends the full history (chapter 1 §1.2), a long-running chat's token cost (and latency) grows every turn, until it eventually can't fit at all without summarizing or truncating older messages.
- **RAG exists at all** (chapter 6) — you can't just paste your entire knowledge base into every prompt; you retrieve only the most relevant excerpts to fit the budget.
- **Chunking strategy matters** (chapter 6 §6.2) — how you split documents directly determines how much of the context window each retrieved piece consumes.

Context window sizes have grown enormously and keep growing — this guide deliberately doesn't quote current numbers, since they're stale within months; the durable skill is knowing *why* the limit exists and *which of your design decisions* (history length, RAG chunk count, system prompt size) are competing for it.

## 3.5 Inference-time controls: temperature, top-p, and why the same prompt gives different answers

A model doesn't deterministically output "the one correct next token" — internally, it computes a **probability distribution** over every possible next token, then *samples* from it. Two parameters you'll set on nearly every API call control that sampling:

- **Temperature** (typically 0–2) — how "sharp" vs. "flat" that distribution is before sampling. Temperature near 0 makes the model almost always pick the single highest-probability token (deterministic, focused, safest for factual/structured tasks like generating JSON or code). Higher temperature flattens the distribution, giving lower-probability tokens a real chance of being picked (more varied, more "creative," also more prone to going off the rails).
- **Top-p (nucleus sampling)** — instead of considering every possible token, only sample from the smallest set of tokens whose cumulative probability reaches `p` (e.g. 0.9), discarding the unlikely long tail entirely. Often used *together* with temperature.

This is the direct, mechanical explanation for "why did I get a different answer asking the same question twice" — and a genuinely useful lever: set temperature low for anything where you need consistency (classification, extraction, code generation for `mastery-bot`-style features) and reserve higher temperature for genuinely open-ended, creative tasks.

## Checkpoint

1. Estimate, using the ~4-characters-per-token rule of thumb, roughly how many tokens this chapter itself consumes — then explain why that number matters if you were sending this whole chapter as context to a model.
2. Explain, in your own words, why a *raw pretrained* model (before instruction fine-tuning/RLHF) would be a poor choice to power a customer support chatbot, even though it technically "knows" a huge amount from pretraining.
3. If you were building a feature that extracts structured JSON fields from an HR document (a real HRNS-adjacent task), what temperature would you choose, and why? What about a feature that drafts a friendly, varied welcome email for new employees?
