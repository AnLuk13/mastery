# 4. Prompting & Prompt Engineering

Goal: prompting is the one skill you'll use in literally every remaining chapter — worth being deliberate about before treating it as an afterthought.

## 4.1 The three roles, and why they're not just labels

Every modern chat-style API (chapter 12) structures input as a list of messages, each tagged with a **role**:

```ts
const messages = [
  { role: "system", content: "You are a helpful assistant for an HR platform. Answer only from the provided context." },
  { role: "user", content: "How many vacation days do new employees get?" },
  { role: "assistant", content: "Based on the policy document, new employees accrue..." }, // a prior turn, if any
  { role: "user", content: "What about part-time staff?" },
];
```

- **system** — instructions about *how the model should behave* for the whole conversation: persona, constraints, output format, what it should refuse to do. Sent once (or updated rarely), not part of the visible "conversation."
- **user** — the actual human request.
- **assistant** — the model's own prior replies, included when you resend conversation history (chapter 3 §3.4) so the model has continuity.

This distinction is why "the system prompt" is worth caring about deliberately: it's the single highest-leverage lever you control, because — as training deliberately reinforces — models generally give system-role instructions more weight than the same instruction phrased as a user message. `AIService.AskJsonStructuredAsync` in HRNS (`../dotnet-mastery/13-ai-ml-integrations.md` §13.2) is exactly this pattern: a fixed `systemPrompt` establishing behavior/format, paired with a variable `userPrompt` for the actual question.

## 4.2 Zero-shot, few-shot, and why examples beat descriptions

- **Zero-shot** — asking for a task with no examples, just an instruction ("Classify this ticket's urgency as low/medium/high").
- **Few-shot** — including a handful of worked examples *in the prompt itself* before the real request:

```
Classify the urgency of each support ticket.

Ticket: "My password reset email never arrived."
Urgency: medium

Ticket: "The entire payroll system is down and payday is tomorrow."
Urgency: high

Ticket: "Can you update my mailing address?"
Urgency: low

Ticket: "Two employees report they can't access the system and it's a company-wide outage."
Urgency:
```

Few-shot prompting reliably improves consistency for classification/formatting/style-matching tasks, because you're showing the model the *exact* pattern instead of describing it in words the model has to interpret — a form of in-context learning (the model adapts its behavior using nothing but the prompt, no weight changes, no training). When a zero-shot instruction gives inconsistent results, adding 2–3 good examples is usually the fastest fix, cheaper and faster to iterate on than fine-tuning (chapter 7).

## 4.3 Chain-of-thought — letting the model "think" before answering

Asking a model to reason step-by-step before giving a final answer ("think through this step by step, then give your answer") measurably improves accuracy on multi-step reasoning tasks (math, logic, multi-condition decisions) — because generation is sequential (chapter 3 §3.1: each token is predicted given everything before it, including tokens the model *itself* just generated), so forcing intermediate reasoning tokens onto the page gives the model's own later tokens more relevant context to condition on, rather than jumping straight to an answer with no scratch space. Some current models do a version of this internally by default; explicitly asking for it remains a reliable, portable technique across providers.

The direct product implication: if you want a final answer *only* (no visible reasoning), ask the model to reason first, then extract just the final field from structured output (§4.4) — rather than suppressing reasoning entirely, which tends to quietly hurt accuracy on anything non-trivial.

## 4.4 Getting structured output instead of free text

Real applications almost never want "a paragraph back" — they want a specific field, a JSON object, a chosen category. Two levels of reliability, roughly in order of preference:

1. **Native structured output / JSON mode** — most current APIs let you pass a JSON schema and the provider *constrains generation* so the output is guaranteed valid JSON matching that shape (this overlaps heavily with tool/function calling, chapter 8 — many providers implement structured output as a degenerate case of "call this one function").
2. **Prompted JSON** (older, less reliable, still common) — instruct the model to "respond with only valid JSON matching this shape: {...}" and parse the response yourself, defensively (the model can still occasionally wrap it in prose or produce near-valid JSON). `AIService`'s `AskJsonStructuredAsync` name (`../dotnet-mastery/13-ai-ml-integrations.md` §13.2) is exactly this second pattern.

Whichever level you use, **never trust the output blindly** — parse it, validate its shape against a real schema (this project already does exactly that for GitHub API responses with Zod — `src/content/github/GitHubApiClient.ts` — the same discipline applies here: an LLM response is just another untrusted external input), and handle the case where it doesn't match.

## 4.5 Why prompts are fragile, and how to manage that

A prompt that works great today can behave differently after: the model provider updates the underlying model version (even a "minor" version bump), you change unrelated wording elsewhere in a long system prompt, or you switch providers entirely (prompts are not portable across model families — each has its own trained "habits" about how it interprets instructions). Practical mitigations, all pointing the same direction as ordinary software engineering:

- **Version and diff your prompts** like code — a prompt change is a behavior change, review it like one.
- **Keep an eval set** (chapter 17 §17.4) — a small fixed set of representative inputs with expected-good-output characteristics, so you can tell whether a prompt edit actually helped before shipping it.
- **Prefer explicit, structured instructions over vague ones** — "respond in under 50 words, as a single paragraph, no markdown" beats "be concise."
- **Put the most important instructions at the start or end of a long prompt** — models are measurably less reliable at attending to instructions buried in the middle of very long context (a real, documented effect, not folklore) — relevant directly to how you structure RAG context (chapter 6 §6.3).

## Checkpoint

1. Rewrite this vague instruction as a well-structured prompt with a clear system/user split and an explicit output format: "make the AI summarize support tickets."
2. Design 2–3 few-shot examples for a task of your choosing (something plausible for `mastery-bot` or HRNS), and explain why you chose those specific examples rather than others.
3. Your app parses an LLM's JSON response and it's occasionally malformed (trailing comma, wrapped in a markdown code fence). List two independent layers of defense you'd add, one at the prompting level and one at the code level — and explain why relying on just one is risky.
