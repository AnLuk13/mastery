# 1. Introduction & Mental Models

Goal: before any code or theory, get the vocabulary straight and build one correct mental model for "what is a model, really" — almost every beginner misconception in AI traces back to skipping this step.

## 1.1 AI, ML, deep learning, LLM — one diagram

These four terms get used interchangeably in casual conversation, but they nest inside each other precisely:

```
ARTIFICIAL INTELLIGENCE (AI)
  any technique that makes a computer do something that seems to require intelligence
  (includes: hand-written chess engines, pathfinding, expert systems — no learning required)
  │
  └── MACHINE LEARNING (ML)
        systems that improve at a task by learning patterns from data,
        instead of being explicitly programmed with rules for every case
        │
        └── DEEP LEARNING
              ML using neural networks with many layers ("deep") —
              the specific technique behind nearly everything AI-hyped since ~2012
              │
              └── LARGE LANGUAGE MODELS (LLMs)
                    deep learning models trained on huge amounts of text,
                    specifically built to predict/generate language —
                    GPT, Claude, Gemini, Llama, Mistral, etc.
```

So: every LLM is deep learning, every deep learning system is ML, every ML system is AI — but the reverse isn't true. When people say "AI" today they almost always mean the LLM layer specifically; this guide does too, unless it says otherwise.

## 1.2 The mental model that actually matters: a model is a function

Strip away the hype and a trained model — LLM or otherwise — is a **pure function** you can call, with a specific, if unusual, signature:

```ts
function model(input: Tokens): Tokens
```

That's genuinely most of it. You give it input (for an LLM: a sequence of text, encoded as **tokens** — chapter 3), and it returns output (more tokens). The "intelligence" isn't a separate reasoning module bolted on top — it's entirely encoded in millions or billions of numbers (**weights** / **parameters**) that this function's internals were tuned to, during **training** (chapter 2), before you ever called it.

Two consequences that trip up almost every beginner, both falling directly out of this mental model:

- **A model has no memory between calls, unless you build it yourself.** Each call to `model(input)` is independent — there's no hidden state carried over. "Conversation memory" in a chat app is just the app re-sending the *entire prior conversation* as part of `input` on every single call (chapter 3 §3.4 explains why this is also why context windows matter so much).
- **A model doesn't "know" anything after it was trained — it has no live connection to the internet, your database, or today's date**, unless your application explicitly puts that information into `input` (this is the entire premise of RAG — chapters 5–6 — and of tool calling — chapter 8). If a model tells you today's date, it's either guessing from training patterns, or your application already put the date in the prompt.

## 1.3 Training vs. inference — two completely different phases

- **Training** — the (extremely expensive, done once by a lab like OpenAI/Anthropic/Google/Meta) process of *finding* the weights, by showing the model enormous amounts of data and adjusting the weights to make its predictions better. Chapter 2 covers this properly.
- **Inference** — *using* an already-trained model to produce an output for a specific input. This is what "calling the API" or "running Ollama" means, and it's cheap and fast (relatively) compared to training. Everything you personally do as an application developer is inference — you are essentially never training a model from scratch (chapter 7 covers the much narrower, much cheaper practice of *fine-tuning* an existing one).

This distinction resolves a common confusion: "does the model learn from my conversation?" — no, not in any persistent sense. Your conversation shapes what goes into `input` for the next call (that's why it "remembers" earlier messages), but it doesn't change the model's weights. The weights are frozen at inference time.

## 1.4 Vocabulary you'll see constantly, defined once here

| Term | One-line definition |
|---|---|
| **Model** | The trained function itself — a specific set of weights (e.g. "GPT-4o", "Claude Opus 4.5", "Llama 3.1 70B"). |
| **Parameters / weights** | The numbers the model's function is built from, set during training. "70B" means 70 billion parameters — a rough proxy for model size/capability, not a precise one. |
| **Token** | The unit a model actually reads/writes — not quite a word, not quite a character (chapter 3 §3.1). |
| **Context window** | The maximum number of tokens a model can consider at once (input + output combined) (chapter 3 §3.4). |
| **Prompt** | The input you send the model. **System prompt** = instructions about how to behave; **user prompt** = the actual request (chapter 4). |
| **Inference** | Running a trained model to get an output — this is "using" the model. |
| **Fine-tuning** | Further training an already-trained model on your own, much smaller dataset, to specialize it (chapter 7). |
| **Embedding** | A model's numeric representation of *meaning*, used for search/similarity, not for generating text (chapter 5). |
| **RAG** | Retrieval-Augmented Generation — feeding a model relevant excerpts of your own data as part of the prompt (chapter 6). |
| **Agent** | A model wired into a loop where it can call tools and see the results, iterating toward a goal, rather than answering in one shot (chapter 9). |
| **Hallucination** | A model stating something false with full confidence — not a bug in the traditional sense, but an inherent consequence of how these models generate text (chapter 17). |
| **Open-weight / closed model** | Whether the trained weights are publicly downloadable (Llama, Mistral) or only accessible via a hosted API (GPT, Claude, Gemini) (chapter 15). |

## 1.5 Where this actually shows up for you

You already have two live, real touchpoints with this material, and this guide leans on both throughout:

- **`HRNS.Platform.Server`'s AI Assistant** — a real, production RAG system (Ollama-hosted LLM, pgvector embeddings, hybrid retrieval), already documented in [`../dotnet-mastery/13-ai-ml-integrations.md`](../dotnet-mastery/13-ai-ml-integrations.md). This guide cross-references it rather than repeating it — go reread that chapter once you've finished chapters 5–6 here, it will read completely differently.
- **Claude Code** — the tool you're using to read this sentence — is itself a running example of an **agent harness** (chapter 9) using **tools** (chapter 8) and **MCP** (chapter 10). You've watched it work all session; chapter 9 makes that explicit.

## Checkpoint

1. In your own words: why is "a model is a pure function" a more useful mental model than "a model is like a person you're talking to"? What does it correctly predict that the "person" analogy gets wrong?
2. If an LLM has no memory between calls, explain concretely what a chat app must be doing on message #5 of a conversation for the model to "remember" messages #1–4.
3. Classify each of the following as AI-but-not-ML, ML-but-not-deep-learning, deep-learning-but-not-LLM, or LLM: a chess engine that searches possible moves; a spam filter trained on labeled emails using logistic regression; an image classifier using a convolutional neural network; ChatGPT.
