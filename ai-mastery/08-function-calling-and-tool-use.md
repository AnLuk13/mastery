# 8. Function/Tool Calling

Goal: the mechanism that lets a model *do things* instead of just describing them — the foundation chapter 9's agents and chapter 10's MCP are both built directly on top of.

## 8.1 The problem: a model can't actually do anything on its own

A model (chapter 1 §1.2) is a pure function from tokens to tokens — it cannot query a database, call an API, run code, or check today's date. **Function calling** (also called **tool calling** or **tool use**) is the mechanism that bridges this gap: you describe, in your API request, a set of functions the model is *allowed to ask for*, and instead of (or alongside) a normal text reply, the model can respond with a structured request to call one of them, with specific arguments it decided on. Your application code — not the model — actually executes the function, then sends the result back to the model as part of the next message, so it can use that result to continue.

```
You: "What's the weather in the office city, and should I bring an umbrella?"
      │
      ▼
Model: (can't know current weather — no live data, chapter 1 §1.2)
      → responds with: call get_weather(city: "Chisinau")
      │
      ▼
Your code: actually calls a weather API, gets { tempC: 14, condition: "rain" }
      │
      ▼
Your code: sends that result back to the model as a new message
      │
      ▼
Model: "It's 14°C and raining in Chisinau — bring an umbrella."
```

The model never executes anything itself — it only ever *decides what should be called, with what arguments*, and your code stays fully in control of whether/how to actually run it (a real security boundary, chapter 17 §17.2).

## 8.2 Defining a tool: JSON schema

You describe each available function to the model using **JSON Schema** — the same schema format you'd use to validate any structured data (this project already uses Zod, which compiles to/validates against this exact concept, for GitHub API responses — `src/content/github/GitHubApiClient.ts`):

```ts
const tools = [
  {
    name: "get_weather",
    description: "Get the current weather for a given city",
    parameters: {
      type: "object",
      properties: {
        city: { type: "string", description: "City name, e.g. 'Chisinau'" },
      },
      required: ["city"],
    },
  },
];
```

The `description` fields aren't decoration — the model reads them (in natural language, alongside the schema) to decide *whether* and *how* to call each tool, so vague descriptions produce unreliable tool-selection, exactly the same "garbage in, garbage out" sensitivity prompting has (chapter 4).

## 8.3 A minimal worked example

```ts
const response = await callModel({
  messages: [{ role: "user", content: "What's the weather in Chisinau?" }],
  tools,
});

if (response.toolCall) {
  const { name, arguments: args } = response.toolCall;
  if (name === "get_weather") {
    const result = await fetchWeather(args.city); // your real implementation
    const followUp = await callModel({
      messages: [
        { role: "user", content: "What's the weather in Chisinau?" },
        { role: "assistant", content: null, toolCall: response.toolCall },
        { role: "tool", toolCallId: response.toolCall.id, content: JSON.stringify(result) },
      ],
      tools,
    });
    console.log(followUp.text); // the model's final answer, using the real result
  }
}
```

The exact message shape (`role: "tool"`, `toolCallId`, etc.) varies slightly by provider (chapter 12), but the pattern — call the model, check for a tool call, execute it yourself, send the result back as a new message, call the model again — is universal, and is exactly the loop chapter 9 generalizes into "agents."

## 8.4 A concrete, honest example: giving a model access to `mastery-bot`'s content

If you wanted a model to answer "what does the TCP chapter say about the three-way handshake?" *using* `mastery-bot`'s real content rather than guessing, tool calling is the natural mechanism — and it composes cleanly with the project's existing `ContentProvider` abstraction (`src/content/types.ts`) without needing to change it at all:

```ts
const tools = [
  {
    name: "search_knowledge_base",
    description: "Search the mastery knowledge base by filename, path, or content",
    parameters: {
      type: "object",
      properties: { query: { type: "string" } },
      required: ["query"],
    },
  },
  {
    name: "get_document",
    description: "Retrieve the full text of a specific document by its path",
    parameters: {
      type: "object",
      properties: { path: { type: "string" } },
      required: ["path"],
    },
  },
];

// Executing a call is then just delegating straight to the existing provider:
async function executeTool(name: string, args: Record<string, unknown>, provider: ContentProvider) {
  if (name === "search_knowledge_base") return provider.search(args.query as string);
  if (name === "get_document") return provider.getDocument(args.path as string);
}
```

Note what didn't need to change: `ContentProvider`'s path-safety guarantees (`normalizeRelativePath`, `src/content/paths.ts`) apply exactly as-is here — the model can only ever *request* a path as a string argument, and `executeTool` still routes it through the same validated `getDocument`, which still rejects traversal attempts. This is worth internalizing as a general principle, not a coincidence: tool arguments are untrusted input from the model's perspective, exactly like a Telegram callback path was untrusted input in `mastery-bot`'s design (chapter 17 §17.2 makes this explicit).

## Checkpoint

1. Why does the model never execute the function itself — what would go wrong (functionally and from a security standpoint) if an LLM provider's infrastructure ran your `get_weather`/`get_document` code directly, rather than handing the request back to your application?
2. Extend §8.4's tool list with a `list_folder` tool matching `ContentProvider.listDirectory`. Write its JSON schema.
3. A tool's `description` is vague ("gets stuff"). Predict, using chapter 4's prompting intuition, what kind of failure you'd expect to see in practice, and rewrite it well.
