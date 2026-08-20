# 14. Cloud vs. Local Models

Goal: turn chapters 12–13 into an actual decision framework — the comparison chapter, because "just use whichever" is not a real answer and "always use X" from either direction is almost always wrong.

## 14.1 The four axes that actually matter

Every real cloud-vs-local decision reduces to weighing four things against your specific situation — not a universal winner on any of them:

| Axis | Cloud API | Local (Ollama/self-hosted) |
|---|---|---|
| **Cost shape** | Metered per-token (chapter 12 §12.4) — scales with usage, zero fixed cost, but genuinely unbounded at high volume | Fixed hardware/hosting cost, ~zero marginal cost per call — better at high, steady volume; wasteful at low/spiky volume |
| **Latency** | Network round-trip + provider queueing, generally optimized infrastructure | No network hop; can be faster *or* slower depending entirely on your own hardware vs. the provider's |
| **Privacy/data residency** | Your data leaves your infrastructure — a real compliance question for regulated/sensitive data (HRNS's exact reasoning, chapter 13 §13.1) | Data never leaves your infrastructure — the deciding factor for a genuine subset of real use cases |
| **Capability** | Access to the current frontier of model quality — closed labs' best models are, as of this writing, still generally ahead of what's realistically self-hostable | Real, usable capability at small-to-mid scale (chapter 13 §13.4), but a genuine, currently-persistent gap vs. the best hosted models on the hardest tasks |

## 14.2 A decision framework, not a religious stance

Work through these questions **in order** — the first one that gives a clear answer usually settles it:

1. **Is there a hard privacy/compliance requirement that data cannot leave your infrastructure?** If yes, local is not optional — this alone overrides every other axis (HRNS's actual situation).
2. **Does the task genuinely need frontier-level reasoning capability** (complex multi-step reasoning, nuanced judgment, cutting-edge coding ability)? If yes, and §14.2.1 didn't already force local, a hosted API is very likely the right call — this is exactly why HRNS's *separate* `AIInputGathering` feature deliberately uses a cloud Ollama instance instead of the local one (`../dotnet-mastery/13-ai-ml-integrations.md` §13.6) — different feature, different requirements, different answer, in the *same platform*, which is itself the strongest evidence there's no single universal right answer.
3. **Is volume high and predictable, and is the task well within a smaller/local model's real capability** (classification, extraction, simple formatting — chapter 2 §2.6's "narrow, well-defined task" territory)? Local starts winning on cost at real scale.
4. **Is volume low, spiky, or unpredictable, and does the task benefit from frontier capability?** Metered cloud pricing usually wins — you're not paying for idle hardware.
5. **None of the above forced a decision — genuinely either works.** Default to whichever is operationally simpler for your team *right now* (usually: cloud API, since it requires no infrastructure to stand up) — and treat the choice as reversible, not permanent, since chapter 11's provider-agnostic abstractions (`IChatClient`, Vercel AI SDK) exist specifically to make switching later cheap.

## 14.3 It's rarely all-or-nothing — hybrid is normal and often correct

HRNS's own architecture already demonstrates this isn't a single platform-wide decision: the main AI Assistant runs against a **local** Ollama instance (compliance-driven), while `AIInputGathering` deliberately calls a **cloud-hosted** Ollama instance instead (`../dotnet-mastery/13-ai-ml-integrations.md` §13.6) — two features in the same codebase, two different, individually-justified choices, both correct for their own requirements. A common, entirely reasonable pattern worth internalizing: route by task — cheap/fast/local for high-volume narrow tasks (classification, keyword extraction — echoing chapter 2 §2.6's classic-ML-vs-LLM instinct one level up: cloud-vs-local, not just LLM-vs-classic-ML), and cloud/frontier for the harder, lower-volume, quality-critical tasks — rather than treating "which model do we use" as one company-wide setting.

## Checkpoint

1. Walk through §14.2's decision framework for a hypothetical `mastery-bot` feature that classifies whether a search query is likely a filename, a topic, or a question (a narrow, high-volume, low-stakes task). Where does the framework land, and why?
2. Now walk through it for a hypothetical feature that drafts a nuanced, context-aware summary comparing two long HRNS policy documents. Where does it land, and why is the answer different from question 1?
3. Explain, using §14.3, why "the whole company should standardize on one model provider" is a more defensible policy for *procurement/contracting* reasons than it is for *technical* reasons — what technical cost does that standardization actually impose, if any, given chapter 11's abstraction layers?
