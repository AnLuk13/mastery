# Glossary

Every acronym and term used across this guide, one line each. Chapter references point to where the term is explained in depth.

**Agent** — a model wired into a loop where it can call tools and see results, iterating toward a goal rather than answering in one shot (ch. 9).
**Alignment** — the research field studying how to make a model's behavior reliably match intended goals/values (ch. 18).
**Attention (self-attention)** — the transformer mechanism letting every token weigh the relevance of every other token, regardless of distance (ch. 3).
**Chain-of-thought (CoT)** — prompting a model to reason step-by-step before giving a final answer, improving multi-step accuracy (ch. 4).
**Chunk / chunking** — splitting a document into smaller pieces before embedding, since a whole document is too large/blurry to embed usefully (ch. 6).
**Closed model** — a model whose weights are never released; only accessible via a hosted API (ch. 15).
**Context window** — the maximum number of tokens (input + output combined) a model can process in one call (ch. 3).
**Cosine similarity** — the angle-based similarity metric most commonly used to compare embedding vectors (ch. 5).
**Deep learning** — machine learning using many-layered neural networks; the technique behind nearly all recent AI progress (ch. 1).
**Distillation** — training a smaller/cheaper model to imitate a larger one on a narrow task, via fine-tuning on the larger model's outputs (ch. 7).
**Embedding** — a fixed-length numeric vector representing a piece of text's meaning, used for similarity/search, not generation (ch. 5).
**Fine-tuning** — further training an already-trained model on your own smaller dataset to specialize its behavior (ch. 7).
**Function/tool calling** — the mechanism letting a model request that your code execute a specific function with specific arguments (ch. 8).
**GGUF** — the file format Ollama/llama.cpp use to package a quantized model's weights for local inference (ch. 13).
**Gradient descent** — the training algorithm that repeatedly nudges a model's weights in the direction that reduces error (ch. 2).
**Hallucination** — a model stating something false with full confidence, an inherent consequence of how text generation works, not a simple bug (ch. 17).
**HNSW** — Hierarchical Navigable Small World; the approximate-nearest-neighbor graph index algorithm most vector databases use (ch. 5).
**Hybrid retrieval** — combining vector (semantic) search and keyword search, then reranking, since neither alone is sufficient (ch. 6).
**Inference** — running an already-trained model to produce an output for a given input (ch. 1).
**JSON Schema** — the structured format used to describe a tool's expected arguments to a model (ch. 8).
**LangChain / LlamaIndex** — general-purpose orchestration frameworks providing reusable RAG/agent building blocks (ch. 11).
**LLM** — Large Language Model; a deep learning model trained on huge amounts of text to predict/generate language (ch. 1).
**LoRA / PEFT** — Low-Rank Adaptation / Parameter-Efficient Fine-Tuning; training only a small add-on to a frozen base model, making fine-tuning cheap (ch. 7).
**MCP** — Model Context Protocol; a standard letting any compatible harness use any compatible server's tools/data with no custom integration (ch. 10).
**Mixture-of-Experts (MoE)** — an architecture routing each token to only a subset of a model's sub-networks, saving compute per token (ch. 18).
**Neural network** — a stack of layers, each a matrix multiply plus a non-linearity, whose weights are found via training (ch. 2).
**Ollama** — a self-hosted LLM runtime for downloading, managing, and serving open-weight models locally (ch. 13).
**Open-weight model** — a model whose trained weights are published for anyone to download and run themselves (ch. 15).
**Overfitting** — a model performing well on training data but poorly on new data, having memorized rather than generalized (ch. 2).
**Parameters / weights** — the numbers a model's function is built from, set during training; a rough proxy for model size (ch. 1, 2).
**Pretraining** — the first, massive-scale training stage where a model learns to predict the next token from huge amounts of text (ch. 3).
**Prompt** — the input sent to a model; **system prompt** sets behavior, **user prompt** is the actual request (ch. 4).
**Prompt injection** — untrusted content embedding instructions the model follows instead of treating as inert data, structurally identical to path traversal (ch. 17).
**Quantization** — storing a model's weights at lower numeric precision after training, shrinking size/cost with a small accuracy trade (ch. 13).
**RAG** — Retrieval-Augmented Generation; retrieving relevant document excerpts and inserting them into the prompt before generation (ch. 6).
**Reranker** — a model/algorithm that reorders a combined set of retrieval candidates by relevance before generation (ch. 6).
**RLHF** — Reinforcement Learning from Human Feedback; the training stage that shapes a raw model into a helpful, well-behaved assistant (ch. 3).
**Sampling (temperature, top-p)** — the inference-time controls determining how deterministic vs. varied a model's token choices are (ch. 3).
**Semantic Kernel / Microsoft.Extensions.AI** — the .NET-ecosystem provider-agnostic chat abstraction and orchestration framework (ch. 11).
**Streaming** — sending a model's response as it's generated, in small chunks, rather than waiting for the complete output (ch. 12).
**Supervised / unsupervised / reinforcement learning** — the three broad flavors of how a model learns from data or feedback (ch. 2).
**Token** — the actual unit a model reads/writes; roughly ¾ of a word or ~4 characters in English (ch. 3).
**Tokenizer** — the component that splits raw text into tokens before a model processes it (ch. 3).
**Transformer** — the neural network architecture (built on self-attention) underlying nearly all modern LLMs (ch. 3).
**Vector database** — a data store optimized for similarity search over embedding vectors (Postgres+pgvector, Pinecone, Chroma, etc.) (ch. 5).
