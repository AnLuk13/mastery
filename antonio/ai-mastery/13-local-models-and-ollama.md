# 13. Local Models & Ollama

Goal: running a model on your own hardware, concretely — you already have the tool installed on this exact machine, so this chapter is meant to be actually run alongside reading it, not just read.

## 13.1 What "running a model locally" actually means

Every model discussed so far in the abstract is, physically, a very large file of numbers (the weights, chapter 2 §2.3) plus code that runs the forward pass (chapter 2 §2.4's step 1, minus training) over them. Running a model "locally" means that file lives on and is executed by your own machine's hardware — no network call to a provider, no per-token billing, and (this is the real headline benefit for a platform like HRNS) **no data ever leaves your infrastructure**. This is precisely why HRNS runs an on-prem Ollama instance for its main AI Assistant rather than calling a hosted API (`../dotnet-mastery/13-ai-ml-integrations.md` §13.2) — an enterprise HR/payroll platform sending real employee data to a third-party API is a genuine, serious compliance concern that local hosting sidesteps entirely.

## 13.2 Ollama — the easiest on-ramp, and it's already on this machine

**Ollama** is a self-hosted LLM runtime — it downloads, manages, and serves open-weight models (chapter 15 §15.2) through a simple local API, hiding a lot of the underlying complexity (model format conversion, hardware-specific optimization) behind a `docker`-like command-line experience. You have it installed right now — try this in a terminal:

```bash
ollama pull llama3.2        # downloads a model (one-time; picks a size Ollama judges reasonable)
ollama run llama3.2         # interactive chat, right in your terminal
```

And, the part that matters for actually building something: Ollama exposes an HTTP API on `localhost:11434` that looks almost exactly like chapter 12's provider APIs — meaning code you write against it is structurally identical to code calling a hosted provider, modulo the base URL:

```ts
const response = await fetch("http://localhost:11434/api/chat", {
  method: "POST",
  body: JSON.stringify({
    model: "llama3.2",
    messages: [{ role: "user", content: "Explain embeddings in one sentence." }],
    stream: false,
  }),
});
```

This is exactly the shape `OllamaSharp`'s `OllamaApiClient` (`../dotnet-mastery/13-ai-ml-integrations.md` §13.2) wraps on the .NET side, registered as HRNS's `IChatClient` implementation.

## 13.3 GGUF and quantization — how a huge model fits on ordinary hardware

A model's weights are normally stored and computed at high numeric precision (commonly 16-bit floating point during training) — a 70-billion-parameter model at that precision needs well over 100GB just to hold the weights, far beyond consumer hardware. **Quantization** solves this by storing weights at lower precision (8-bit, 4-bit, sometimes lower) after training, trading a small, usually-acceptable amount of accuracy for a dramatic reduction in memory footprint and faster inference — a 4-bit quantized model can be roughly a quarter the size of its full-precision original. **GGUF** is the file format Ollama (built on `llama.cpp` underneath) uses to package a quantized model's weights plus the metadata needed to run it — when `ollama pull` downloads a model, it's fetching a GGUF file, already quantized to a size Ollama judged reasonable for typical hardware.

This is the direct, mechanical reason "can I run a 70B model on my laptop" has a real, calculable answer rather than being a vague guess: check the quantized (GGUF) file size against your available RAM/VRAM (§13.4), not the "70 billion parameters" headline number at full precision.

## 13.4 A realistic hardware expectation check

Local inference speed and feasibility depend almost entirely on **RAM (or GPU VRAM, which is dramatically faster)** and, to a lesser extent, raw compute. Rough, durable guidance — deliberately not tied to specific model names, which change constantly:

- **Small models** (roughly 1–3B parameters, quantized) — comfortably run on ordinary consumer laptops with no dedicated GPU, at usable speed, though noticeably less capable than larger/hosted models on complex reasoning.
- **Mid-size models** (roughly 7–14B, quantized) — the practical sweet spot for a decent consumer machine (16GB+ RAM, or any recent gaming/workstation GPU) — genuinely useful capability, still local.
- **Large models** (70B+) — need either a serious multi-GPU workstation/server, or heavy quantization plus patience (much slower, CPU-bound inference) — this is realistically the "on-prem enterprise server" tier HRNS's real deployment lives in, not a personal laptop's.

The universal, durable rule of thumb, independent of any specific number: **if the quantized model file doesn't fit comfortably in available RAM/VRAM, it will run — just via slow disk swapping — or fail to load at all**, well before any subtler bottleneck matters.

## 13.5 What local models are, and aren't, good for

Genuinely strong reasons to run locally: **data never leaves your machine/infrastructure** (HRNS's real reason), **no per-token cost at any volume** (real for high-volume, low-complexity tasks), **works offline**, **full control over exactly which model version runs, indefinitely** (a hosted provider can silently swap the model behind an API name — chapter 4 §4.5's "prompts are fragile" risk). Genuine tradeoffs, not just upsides: locally-runnable open-weight models generally lag the very best hosted closed models on the hardest reasoning tasks (chapter 15 covers this landscape properly), you own the operational burden of running and updating the serving infrastructure yourself, and hardware is a real up-front/ongoing cost that a metered API trades for a variable one. Chapter 14 turns this into an explicit decision framework rather than a list of pros and cons.

## Checkpoint

1. Run `ollama pull llama3.2` (or any small model) and `ollama run` it yourself — ask it something, and separately ask it to explain what quantization is. Does its own explanation match §13.3?
2. HRNS validates that configured Ollama models actually exist on the target server *before* the app finishes booting (`../dotnet-mastery/13-ai-ml-integrations.md` §13.2). Explain why this specific check matters more for a self-hosted model than it would for a hosted provider API.
3. A colleague argues "we should always use local models, cloud APIs are a privacy risk." Using §13.5, give one legitimate case where a hosted cloud model would still be the right choice despite that concern.
