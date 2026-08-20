# 9. Agents & Harnesses

Goal: generalize chapter 8's single tool call into a loop, and recognize that loop in the tool you've been using this entire session — this chapter is best read *after* you've spent real time watching Claude Code work, not before.

## 9.1 From one tool call to a loop

Chapter 8's weather example made exactly one round trip: model asks for a tool call, your code runs it, model gives a final answer. Most real tasks need more than one step, and — critically — the *right* next step often depends on what the *previous* step's result actually was, which you can't know in advance. An **agent** is exactly this generalized into a loop:

```
1. Give the model a goal + a set of available tools
2. Model decides: either give a final answer, OR call a tool
3. If it called a tool: execute it, feed the result back as a new message
4. Repeat from step 2 — the model sees everything so far, decides the NEXT step
5. Stop when the model gives a final answer (or a step/turn limit is hit)
```

This is often called the **plan → act → observe** loop, or by the name of a specific, influential prompting pattern for it: **ReAct** (Reason + Act) — at each step, the model is prompted to articulate its reasoning about what to do next *before* deciding on an action, which measurably improves an agent's ability to recover from a tool call that didn't give the expected result (chapter 4 §4.3's chain-of-thought, applied specifically to tool-using loops).

## 9.2 You've been watching this loop all session — literally

Claude Code — running right now, having just built and deployed `mastery-bot` with you across this entire conversation — is a real, production agent harness, and every mechanism in this chapter maps onto something you directly observed:

- **Tools** (chapter 8) = `Read`, `Write`, `Edit`, `Bash`, `Grep`, and every other capability offered each turn — each one is a function with a JSON-schema-described signature, exactly like §8.2.
- **The loop** (§9.1) = every multi-step task this session (e.g. Stage 9's deployment: run a command, read the output, decide the next command based on what actually happened, repeat) was this exact plan → act → observe cycle, not a single scripted sequence decided in advance.
- **A "harness"** is the surrounding application/infrastructure that runs this loop, presents tools to the model, executes what it asks for, and manages context — Claude Code itself is the harness; the underlying model is a separate, swappable component the harness calls into, exactly like chapter 8's `callModel()`.
- **Context management** — as the loop runs long, conversation history grows toward the context window limit (chapter 3 §3.4); this session's own "summarized" system reminders you've seen are the harness actively managing that limit, not something the model does to itself.
- **Human-in-the-loop checkpoints** — this project's own CLAUDE.md-driven instructions ("STOP after Stage N, wait for confirmation") are a deliberate, human-imposed control on an otherwise autonomous agent loop — a real, common production pattern for agents empowered to take consequential actions (chapter 17 §17.2 covers why this matters for exactly this reason).

## 9.3 Why agents fail differently than single-shot prompts do

A single bad response from chapter 4's simple prompting is a one-time annoyance. An agent's mistakes can **compound**, because each step's output becomes part of the input to every subsequent step:

- **Error propagation** — a wrong assumption in step 2 can steer every later step further off course, since the model is reasoning from its own (now-wrong) prior conclusions, not re-deriving from scratch.
- **Tool misuse / hallucinated arguments** — the model can call a real tool with plausible-looking but wrong arguments (an invented file path, a malformed query) — exactly why `executeTool` in chapter 8 §8.4 still validates everything through `ContentProvider`'s real path-safety logic rather than trusting the model's arguments outright.
- **Runaway loops** — without a step limit or clear stopping condition, an agent can loop indefinitely, retrying a failing action slightly differently each time. Production harnesses always cap this (a maximum number of turns/tool calls), the same defensive instinct as a `while` loop needing a guaranteed exit condition in ordinary code.
- **Cost/latency compounding** — each loop iteration is a full model call (chapter 12); a 10-step agent task costs roughly 10x a single prompt, non-negotiably, since each step needs the model's actual reasoning about what happened in the previous one.

## 9.4 Building a minimal agent loop yourself

Stripped to its essentials, on top of chapter 8's single-call pattern:

```ts
async function runAgent(goal: string, tools: Tool[], maxSteps = 10) {
  const messages: Message[] = [{ role: "user", content: goal }];

  for (let step = 0; step < maxSteps; step++) {
    const response = await callModel({ messages, tools });

    if (!response.toolCall) {
      return response.text; // model decided it's done — final answer
    }

    const result = await executeTool(response.toolCall.name, response.toolCall.arguments);
    messages.push(
      { role: "assistant", content: null, toolCall: response.toolCall },
      { role: "tool", toolCallId: response.toolCall.id, content: JSON.stringify(result) },
    );
  }

  throw new Error("Agent exceeded max steps without finishing");
}
```

This ~15-line loop is, mechanically, not far from what's actually running underneath much more sophisticated harnesses (including Claude Code) — the sophistication lives in tool design, context/memory management, error recovery, and safety controls layered around this loop, not in the loop's basic shape.

## Checkpoint

1. During this session, Stage 7's webhook implementation required multiple tool calls (reading files, running `npm test`, checking output, fixing a type error, re-running tests). Describe that sequence explicitly as plan → act → observe steps.
2. Explain concretely why `maxSteps` (§9.4) is a necessary safeguard, using an example of what could make an agent loop indefinitely without one.
3. This project's mega-spec explicitly required "STOP after Stage N, wait for confirmation" before git push/deploy actions. Using §9.2 and §9.3, explain why that's a reasonable safety control specifically for an *agent* (as opposed to being unnecessary for a single-shot prompt).
