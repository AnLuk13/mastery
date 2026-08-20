# 6. Retrieval-Augmented Generation (RAG)

Goal: the architecture that turns "a model that can hallucinate confidently about anything" into "a model that answers *from your actual documents*" — arguably the single most commonly-built real AI product feature, and the direct payoff of chapter 5.

## 6.1 The problem RAG solves

Recall from chapter 1 §1.2: a model has no live connection to your data, and from chapter 3 §3.4: you can't just paste your entire knowledge base into every prompt — it won't fit, and even when it would, it's slow and expensive to try. RAG's answer: **retrieve** only the small number of document excerpts actually relevant to *this specific question*, and **augment** the prompt with them before **generating** a response — so the model answers grounded in real, current, retrieved text instead of guessing from whatever it happened to absorb during training (which may be outdated, wrong for your specific domain, or simply doesn't include your private data at all, since your documents were never part of its training set).

```
User question
     │
     ▼
Embed the question (chapter 5)
     │
     ▼
Vector-search your document store for the closest-matching chunks
     │
     ▼
Build a prompt: [system instructions] + [retrieved chunks as context] + [user question]
     │
     ▼
Send to the chat model → answer, grounded in the retrieved text
```

This is exactly the shape of HRNS's AI Assistant, end to end — `IVectorSearchService` doing the retrieval step, `AIService` doing the generation step (`../dotnet-mastery/13-ai-ml-integrations.md` §13.1).

## 6.2 Chunking — the step everyone underestimates

You can't embed and retrieve "a whole document" usefully — documents are too long, and a single embedding vector for an entire multi-topic document blurs together too many distinct meanings to be a precise match for any specific question. So documents get split into **chunks** first, each chunk embedded and stored separately. Chunking strategy has an outsized, easy-to-underestimate effect on retrieval quality:

- **Chunk size** — too small and a chunk loses necessary surrounding context (a sentence fragment with a pronoun and no antecedent); too large and it dilutes the embedding's precision (back to the "blurs together too many topics" problem) and wastes context-window budget per retrieved chunk.
- **Chunk boundaries** — splitting mid-sentence or mid-code-block produces genuinely worse chunks than splitting at natural boundaries (paragraphs, headings, function boundaries). `KnowledgeChunkEntity.SectionPath` (`../dotnet-mastery/13-ai-ml-integrations.md` §13.3, e.g. `"EMPLOYEES > Creating a New Employee > Required Fields"`) is exactly this discipline — chunks are split along the source document's own heading structure, not blindly by character count, and the section path is preserved as metadata so retrieved chunks stay traceable to where they came from.
- **Overlap** — chunks are often given a small overlapping window with their neighbors, so information sitting right at a chunk boundary doesn't get orphaned into an unhelpful split.

There's no single universally-correct chunk size — it's a real, empirical tuning knob, evaluated against your own eval set (chapter 17 §17.4), not a constant to copy from a tutorial.

## 6.3 Building the augmented prompt

Once you have the top-N relevant chunks, they get inserted into the prompt — typically in the system message or as a clearly-delimited block, with source attribution preserved so the model (and your UI) can cite where an answer came from:

```ts
const systemPrompt = `Answer the user's question using ONLY the context below.
If the answer isn't in the context, say you don't know — do not guess.

Context:
${chunks.map(c => `[Source: ${c.sourceFile} § ${c.sectionPath}]\n${c.content}`).join("\n\n")}`;
```

That "answer ONLY from the context, say you don't know otherwise" instruction is doing real, load-bearing work — without it, a model will happily blend retrieved context with its own (possibly wrong, possibly outdated) parametric knowledge, which defeats much of RAG's purpose. This is also where chapter 4 §4.5's "important instructions belong at the start/end, not buried in the middle" advice matters concretely: put grounding instructions clearly before or after the (often large) context block, not buried inside it.

## 6.4 Hybrid retrieval — why pure vector search alone isn't enough

Pure semantic (vector) search is excellent at "conceptually similar" but genuinely misses exact-term matches — an acronym, a specific field name, an exact error code — that a plain keyword search catches trivially, because an embedding blurs exact wording in favor of meaning (chapter 5 §5.1, by design). The reverse is also true: keyword search misses a paraphrase a vector search catches easily. HRNS's `IVectorSearchService` + `IKeywordSearchService` + `IStructuralSearchService`, merged by `IHybridRetrieverService` and reordered by `IRerankerService` (`../dotnet-mastery/13-ai-ml-integrations.md` §13.4), is exactly this pattern in production: run multiple independent retrieval strategies in parallel, then merge and **rerank** the combined candidate set (a second, often smaller/cheaper model or algorithm specifically trained to judge relevance-ordering) before handing the final top results to the generation step. This is genuinely current, well-regarded RAG architecture, not a toy "just embed everything" implementation — worth recognizing by name if you build or evaluate something like it elsewhere.

## 6.5 RAG vs. fine-tuning — resolving the question you'll be asked constantly

Once people learn a model "doesn't know their data," the instinctive question is usually "should we fine-tune it on our data?" Almost always, RAG is the better first answer:

| | RAG | Fine-tuning (chapter 7) |
|---|---|---|
| Adding new/changed data | Update the document store — no retraining, near-instant | Requires a new fine-tuning run |
| Data freshness | Trivial — retrieval hits current data every call | Frozen at whatever point you fine-tuned |
| Cost to set up | Moderate (embedding pipeline + vector store) | Higher (training infrastructure/cost, and expertise) |
| Grounding/citability | Natural — you know exactly which chunks were retrieved | Opaque — no way to know "why" the model said something |
| What it's actually good for | Injecting *knowledge* the model should reference | Changing *behavior/style/format*, not injecting facts |

The rule of thumb chapter 7 expands on: use RAG to give a model access to facts it doesn't have; reach for fine-tuning only when the problem is genuinely about *how* the model behaves (tone, output format, domain-specific reasoning style) rather than *what* it knows.

## Checkpoint

1. HRNS's `KnowledgeIngestionHostedService` (`../dotnet-mastery/13-ai-ml-integrations.md` checkpoint question 2) presumably re-chunks and re-embeds documents on some trigger. Explain why `ContentHash` (§13.3) existing on `KnowledgeChunkEntity` matters here, in terms of real embedding-call cost.
2. A user asks HRNS's AI Assistant "What's our policy on remote work?" and the answer is subtly wrong — it describes a policy that was true two versions of the document ago. Using this chapter's vocabulary, name the two most likely root causes and how you'd distinguish between them.
3. Design (in words) a chunking strategy for `mastery-bot`'s markdown knowledge base specifically — given that files already use numbered headings (`00-INDEX.md`, `01-...`) and clear `## N.M` section structure like this very file. What natural boundaries would you chunk on, and what metadata (parallel to `SectionPath`) would you want to preserve per chunk?
