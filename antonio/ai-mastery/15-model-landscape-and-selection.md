# 15. The Model Landscape & How to Choose

Goal: a durable framework for navigating a fast-moving space, deliberately without a table of "current best models" that would be stale within weeks of being written.

## 15.1 Why this chapter avoids naming a "best" model

Model rankings change every few months as labs release new versions — any specific claim like "Model X is currently the strongest at coding" would be actively wrong by the time you're reading this months or years later. What doesn't go stale: **how to evaluate a model for your task yourself**, which is the actual skill this chapter teaches. Treat any specific model name below as an *example of a category*, not a recommendation frozen in time.

## 15.2 Open-weight vs. closed models

- **Closed models** — weights are never released; you access the model only through the provider's hosted API (chapter 12). Examples of *labs* that primarily operate this way: OpenAI's GPT family, Anthropic's Claude family, Google's Gemini family. You cannot run these locally (chapter 13) under any circumstances — there's no file to download.
- **Open-weight models** — the trained weights are published for anyone to download and run themselves (chapter 13's Ollama pulls exactly these). Examples of *families* that publish open weights: Meta's Llama family, Mistral's models, and a large, fast-moving ecosystem of others. "Open-weight" is a more precise term than "open-source" here — the weights are downloadable and runnable, but the training data and exact training code usually aren't, so it's not "open source" in the same full sense as, say, this project's own codebase.

The practical consequence, tying directly back to chapter 14: **the cloud-vs-local decision and the open-vs-closed decision are almost the same decision** — closed models are only ever available via cloud API; open-weight models are usable either locally (chapter 13) *or* via a cloud API that happens to host them (several providers serve open-weight models over a hosted API too, for teams that want the model but not the operational burden of running it).

## 15.3 What "context window," "multimodal," and "reasoning model" mean when comparing models

Beyond raw capability, models differ on real, comparable dimensions worth checking for your specific task, without needing exact current numbers:

- **Context window size** (chapter 3 §3.4) — varies significantly by model; check the actual current number for your candidate model against your actual expected input size (a full RAG context block, chapter 6, plus conversation history, chapter 3 §3.4) before assuming it fits.
- **Multimodal input/output** — whether a model accepts (or produces) images, audio, or other non-text media alongside text, briefly covered further in chapter 18. Relevant if your feature needs to reason about a screenshot, a scanned document, or generate an image — irrelevant, and not worth paying for, if your task is pure text.
- **"Reasoning" models** — a newer category of model specifically trained/prompted to spend more inference-time computation deliberately "thinking" before answering (a more built-in, automatic version of chapter 4 §4.3's chain-of-thought), generally at higher cost and latency per call, in exchange for measurably better performance on hard multi-step problems. Not the right default for simple, well-defined tasks (classification, extraction) where the extra cost buys you nothing.

## 15.4 A practical evaluation checklist for picking a model for a real task

1. **Does the task require local/private execution** (chapter 14 §14.2 step 1)? If yes, your candidate list is already narrowed to open-weight models you can self-host (chapter 13), full stop.
2. **What's the actual complexity of the task?** Simple classification/extraction/formatting almost never needs the largest/most expensive/"reasoning" model available — chapter 2 §2.6's instinct (match tool to task complexity) applies within the LLM category too, not just LLM-vs-classic-ML.
3. **What's your actual context requirement?** Compute it concretely (system prompt + RAG context + conversation history, in tokens, chapter 3 §3.1's ~4-chars-per-token estimate) and check candidate models against it, not the other way around.
4. **Run your own eval** (chapter 17 §17.4) **against 2–3 realistic candidates on your actual task**, rather than trusting a general-purpose public leaderboard — leaderboards measure broad, generic capability; your task's specific failure modes may not correlate with where a model ranks generally.
5. **Only then, compare cost/latency among your remaining viable candidates.**

The consistent theme, worth naming explicitly: **start from the task's actual requirements and narrow down**, rather than starting from "which model is best" (an unanswerable, constantly-shifting question) and hoping it fits.

## Checkpoint

1. You're picking a model for `mastery-bot`'s hypothetical `search_knowledge_base` tool (chapter 8 §8.4) — a narrow task: given a query, decide which retrieved chunks are actually relevant. Walk through §15.4's checklist for this specific task.
2. Explain why "multimodal" capability would be entirely irrelevant to that specific `mastery-bot` feature, but potentially very relevant to a hypothetical "let users photograph a printed HR form and extract its fields" feature.
3. A colleague picks a model purely because it's #1 on a popular public benchmark leaderboard, for a narrow internal classification task. Using this chapter, explain what's potentially wrong with that reasoning.
