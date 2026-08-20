# 12. Calling Model APIs

Goal: strip away the SDK layer once and look at what's actually crossing the wire — makes every SDK you'll ever use afterward easier to debug, and makes streaming and pricing concrete rather than abstract.

## 12.1 The shape every provider's API roughly shares

Despite different exact field names, every major provider's chat API takes broadly the same request shape — the message list from chapter 4 §4.1, plus model selection and sampling parameters from chapter 3 §3.5:

```ts
const response = await fetch("https://api.example-provider.com/v1/chat/completions", {
  method: "POST",
  headers: {
    "Authorization": `Bearer ${apiKey}`, // server-side only — chapter 17 §17.2 covers why this is non-negotiable
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    model: "provider-model-name",
    messages: [
      { role: "system", content: "You are a helpful assistant." },
      { role: "user", content: "Explain RAG in one sentence." },
    ],
    temperature: 0.7,
    max_tokens: 200,
  }),
});

const data = await response.json();
console.log(data.choices[0].message.content);
```

This is deliberately unglamorous, and that's the point: it's the exact same "server-side `fetch`, `Authorization: Bearer`, parse JSON, validate the shape before trusting it" pattern `mastery-bot`'s own `GitHubApiClient` (`src/content/github/GitHubApiClient.ts`) already uses for a completely different external API — the fact that one API returns Markdown file contents and the other returns generated text doesn't change the shape of the trust boundary at all.

## 12.2 Streaming — why chat UIs render token-by-token

Waiting for a complete response before showing anything feels slow, especially for longer answers — so nearly every provider supports **streaming**, where the response arrives as a sequence of small chunks (individual or small groups of tokens, chapter 3 §3.1) as they're generated, rather than all at once at the end. This is `await foreach`-consuming an `IAsyncEnumerable<T>` in HRNS's `AIService` (`../dotnet-mastery/13-ai-ml-integrations.md` §13.2) — the exact mechanism carrying each streamed chunk out over `AIAssistantHub`'s SignalR connection to the browser in real time. In plain HTTP terms, this is typically **Server-Sent Events (SSE)** — the same persistent-connection, server-push mechanism covered generally in `../networking-mastery/11-websockets-sse-realtime.md` — worth rereading that chapter now that you have a concrete reason to care about it. On the client side, streaming is *why* AI chat UIs universally render text appearing progressively rather than popping in all at once — it's not a cosmetic choice, it's directly exposing the streaming response as it arrives.

## 12.3 Rate limits and retries — treat the model API like any other flaky external dependency

Model APIs fail transiently — rate limits (too many requests too fast), timeouts on unusually long generations, occasional 5xx errors — for exactly the same operational reasons any external API does. `mastery-bot`'s `GitHubApiClient` already has a real, working template for this discipline: map specific HTTP statuses to specific typed errors (`ContentProviderRateLimitedError`, `ContentProviderUnavailableError`, `src/content/github/GitHubApiClient.ts`), respect a `retry-after` header when present, and never let a raw provider error leak to the end user (`describeContentError`, `src/telegram/userMessages.ts`). Applying the exact same pattern to a model API call — rather than treating "the AI" as somehow more reliable than any other network dependency — is the correct instinct, not overcaution.

## 12.4 Tokens and pricing — the one thing you actually need to track

Nearly every provider prices per-token, usually with **separate rates for input and output tokens** (output is typically more expensive — it's the part the model actually generates, per the transformer's attention cost, chapter 3 §3.2), and often a different rate per model tier. Every response includes a `usage` field reporting exactly how many input/output tokens that call actually consumed:

```ts
console.log(data.usage); // { prompt_tokens: 42, completion_tokens: 118, total_tokens: 160 }
```

Logging this per call (never the actual prompt/response content unless you have a specific, deliberate reason to — the same "never log full document content/secrets" discipline this project already follows, `src/telegram/webhookHandler.ts`'s error logging) is the practical foundation for cost monitoring — without it, "why did the bill spike" is unanswerable after the fact. This guide deliberately never quotes actual per-token prices — they change often and vary by provider/model/tier; the durable habit is reading `usage` on every real call, not memorizing a number.

## 12.5 Provider differences worth knowing about, without memorizing

The three dominant hosted providers as of this writing — **OpenAI**, **Anthropic**, **Google (Gemini)** — differ in specifics (exact message-format quirks, how system prompts are passed, tool-calling schema details, context window sizes) but share the conceptual model this entire guide has built up: chat messages with roles, streaming, tool calling, token-based pricing. This is precisely the portability problem chapter 11's unified abstractions (`IChatClient`, Vercel AI SDK) solve — write against the abstraction, and the provider-specific request-shape differences become the library's problem, not yours, at the cost of not seeing exactly what's sent unless you go looking (chapter 11 §11.5's tradeoff, made concrete).

## Checkpoint

1. `mastery-bot`'s `GitHubApiClient` validates every response with Zod before trusting it (`src/content/github/GitHubApiClient.ts`). Sketch (in words) what a `parseModelResponse` equivalent would need to check for a model API response, and what should happen if the shape is unexpected.
2. Explain concretely why output tokens are typically priced higher than input tokens, tying it back to chapter 3 §3.2's attention cost and the fact that generation happens one token at a time, each depending on every token before it.
3. If you were adding streaming AI responses to a Telegram-based interface like `mastery-bot` (which, unlike a web chat UI, can't easily "render token-by-token" into a single message that's still being typed), how would you adapt the streaming pattern — what would you actually do with each incoming chunk?
