# 18. Further Topics — Appendix

Goal: know these exist, know roughly what each is for, go deeper only if your work actually demands it — the same honest "🟡 later tier" as `../networking-mastery/18-further-topics-appendix.md`, rather than padding them into full chapters they don't need.

## Multimodal models

Models that accept and/or produce more than text — **vision** (understanding images: describing a screenshot, reading a scanned document), **audio** (speech-to-text, text-to-speech, or reasoning directly over audio), and **image/video generation**. Mechanically, these extend chapter 3's transformer architecture with additional encoders/decoders for the relevant media type, but the core "tokens in, tokens out, attention over everything" story (chapter 3 §3.2) still holds — an image gets converted into a sequence of tokens the same attention mechanism operates over, conceptually. Relevant the moment a feature needs to reason about something that isn't naturally text (chapter 15 §15.3 already flagged this as a model-selection criterion).

## RLHF and alignment, in more depth

Chapter 3 §3.3 covered RLHF at the level needed to understand *why* a released model behaves like an assistant rather than a raw text-completion engine. The deeper field here — **alignment research** — studies how to make a model's behavior reliably match intended goals/values, including harder variants of RLHF (like **DPO**, Direct Preference Optimization, which achieves a similar effect without needing a separate trained reward model) and the broader, still-open problem of ensuring a highly capable model behaves well in situations its training didn't explicitly anticipate. Worth knowing this is an active research field, not a solved problem — relevant context for why current models still occasionally fail in surprising ways despite extensive RLHF.

## Mixture-of-Experts (MoE) architectures

An architectural variant where a model isn't one dense network every token passes through in full, but many smaller "expert" sub-networks, with a learned **router** sending each token to only a handful of relevant experts rather than all of them. This lets a model have a very large total parameter count (more learned capacity) while only "activating" a fraction of those parameters per token (keeping inference cost closer to a much smaller dense model) — several current frontier and open-weight models use this. Relevant if you ever see a model's parameter count reported as e.g. "total" vs. "active" — that distinction is exactly this.

## Quantization, beyond chapter 13's practical level

Chapter 13 §13.3 covered quantization at the "why a huge model fits on your laptop" level. Deeper territory: different quantization *schemes* (GPTQ, AWQ, and GGUF's own internal variants) trade off compression ratio, accuracy retention, and inference speed differently, and **quantization-aware training** (baking quantization into the training process itself, rather than applying it after the fact to an already-trained model) can retain more quality than naive post-training quantization. Only worth this level of depth if you're actually optimizing a specific local deployment's cost/quality tradeoff yourself, rather than using whatever quantization Ollama picked by default.

## Speech and real-time voice

Real-time voice AI (a model listening and responding in spoken conversation, with low enough latency to feel natural) combines speech-to-text, an LLM, and text-to-speech — either as three separate pipeline stages, or increasingly as models trained to handle audio more directly end-to-end, reducing the latency and quality loss that comes from chaining three separate systems. Relevant specifically for voice-interface features — not needed for anything text-based like `mastery-bot`'s current Telegram interface.

## Vector database internals beyond HNSW

Chapter 5 §5.3 covered HNSW as "the algorithm your pgvector index actually uses" at the level needed to use it correctly. Beyond that: alternative index algorithms (IVF-based approaches trade different accuracy/speed/build-time characteristics), **hybrid search fusion algorithms** (precisely how a hybrid retriever like HRNS's, chapter 6 §6.4, mathematically combines vector and keyword scores into one ranking — commonly **Reciprocal Rank Fusion**, worth knowing by name if you ever need to implement hybrid merging yourself rather than relying on a library/database that does it for you), and vector quantization *of the embeddings themselves* (a separate concept from chapter 13's model-weight quantization, applied to shrink a large vector index's memory footprint).

## Where to actually go deeper

Unlike the earlier chapters, this guide won't build out full grounded chapters for any of the above unless a real task in front of you demands it — that's the whole point of an honest "later" tier. When one of these becomes actually relevant to something you're building, that's the right moment to go find current, authoritative documentation on that specific topic — by then you'll have the vocabulary and mental models (chapters 1–17) to actually make sense of it quickly.
