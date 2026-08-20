# 2. Machine Learning Fundamentals

Goal: enough classic ML to make chapter 3's "how LLMs work" click — not a full ML course. You need to understand *what training is actually doing* before "predicting the next token" means anything.

## 2.1 The three flavors of learning

- **Supervised learning** — you have labeled examples (`input → correct output` pairs) and the model learns to reproduce that mapping. Spam filter: emails labeled "spam"/"not spam". This is how LLM *pretraining* is framed too (chapter 3 §3.3) — "given this text so far, predict the next token" — the "label" is just the actual next word in real text, which is free and infinite (no human has to hand-label it).
- **Unsupervised learning** — no labels at all; the model finds structure in raw data on its own (clustering similar customers together, for example). Embeddings (chapter 5) are learned in a way that's closer to this — the model isn't told "these two sentences mean the same thing," it learns which texts tend to appear in similar contexts.
- **Reinforcement learning (RL)** — the model takes actions and receives a reward signal, and learns to maximize reward over time, without being told the "correct" action directly. This is precisely how **RLHF** (Reinforcement Learning from Human Feedback, chapter 3 §3.3) turns a raw text-predictor into a helpful assistant: humans rank model outputs, and the model is tuned to produce outputs that rank higher.

Almost everything in this guide is a supervised-learning or RL story, occasionally unsupervised for embeddings — worth being able to name which one you're looking at.

## 2.2 Features, and why deep learning stopped needing you to pick them

**Classical ML** requires a human to decide which measurable properties of the input (**features**) the model gets to see — e.g. "email length," "number of exclamation marks," "sender domain" for a spam filter. `ISmartKeywordExtractorService`/ML.NET's TF-IDF in your HRNS AI Assistant (`../dotnet-mastery/13-ai-ml-integrations.md` §13.5) is exactly this style: a human-designed statistic (word frequency) fed into a classic model (LightGBM), not a neural network learning from raw text.

**Deep learning** largely removed this step: instead of hand-picking features, you feed the model something close to raw data (raw pixels, raw text) and — critically — stack many layers of a neural network so that *early* layers learn simple features (edges, in an image; common letter pairs, in text) and *later* layers combine those into increasingly abstract ones (faces; grammatical structure; meaning), entirely from training, with no human specifying any of it. This is the "deep" in deep learning, and it's *why* LLMs became viable — nobody could hand-design a feature set for "meaning."

## 2.3 A neural network, minimally

A neural network is a stack of layers, where each layer is nothing more exotic than: multiply the input by a matrix of weights, add a bias, then apply a simple non-linear function (like ReLU: `max(0, x)`). Written as pseudocode for one layer:

```ts
function layer(input: number[], weights: number[][], bias: number[]): number[] {
  const linear = matrixMultiply(weights, input).map((v, i) => v + bias[i]);
  return linear.map(v => Math.max(0, v)); // ReLU non-linearity
}
```

Stack a few dozen (or, for modern LLMs, over a hundred) of these, chained so each layer's output feeds the next layer's input, and you have a neural network. The non-linearity between layers is not decorative — stacking purely linear layers collapses mathematically into a single linear layer no matter how many you stack, so without it, "depth" would buy you nothing. **Every one of those weight matrices and biases is a parameter** — this is literally what "a 70-billion-parameter model" is counting.

## 2.4 Training: how the weights get found

At the start, weights are random — the network outputs garbage. Training repeats this loop, once per batch of examples, potentially trillions of times for a modern LLM's pretraining:

1. **Forward pass** — run the current weights on a batch of inputs, get outputs.
2. **Loss** — a single number measuring how wrong those outputs were vs. the correct answer (for next-token prediction: essentially "how surprised was the model by the actual next token").
3. **Backward pass (backpropagation)** — compute, for every single weight in the entire network, "if I nudged this weight slightly, would the loss go up or down, and by how much" (the *gradient*).
4. **Update** — nudge every weight slightly in the direction that reduces loss (this is **gradient descent**; the step size is the **learning rate**).

Repeat, over and over, and the loss trends downward — the model gets statistically better at the task. This loop is *why* training is so expensive: step 3 requires computing a gradient for every parameter, for every batch, for potentially trillions of batches, which is why it needs massive parallel hardware (GPUs/TPUs) and why it's done once, by a well-funded lab, rather than by you.

## 2.5 Overfitting — the single most important failure mode to recognize

A model can get *very* good at its training data while getting *worse* at data it hasn't seen — it's memorized specific examples instead of learning the general pattern. This is **overfitting**, and it's why ML practice always splits data into a **training set** (what the model learns from) and a held-out **validation/test set** (used only to check whether it actually generalizes). If a spam filter is 99.9% accurate on training emails but 60% accurate on new ones, it overfit.

This concept resurfaces directly in chapter 7 (fine-tuning): fine-tuning on too small or too narrow a dataset is a fast way to make a model worse at everything except the exact examples you fine-tuned it on.

## 2.6 Where classic ML still beats deep learning

LLMs get the attention, but plenty of production ML — including in your own HRNS platform — is deliberately *not* an LLM: `Microsoft.ML`/LightGBM (`../dotnet-mastery/13-ai-ml-integrations.md` §13.5) is classic gradient-boosted-tree ML, chosen for TF-IDF keyword extraction rather than an LLM call, almost certainly because it's dramatically cheaper, faster, and more deterministic for a narrow, well-defined task like "extract distinctive keywords from this document." A durable rule of thumb: reach for an LLM when the task is genuinely open-ended (understanding free-form meaning, generating novel text); reach for classic ML when the task is narrow, well-labeled, and needs to be fast/cheap/deterministic. Using an LLM for everything is a very common and very avoidable beginner mistake once you're building real products.

## Checkpoint

1. Classify HRNS's TF-IDF keyword extraction and its Ollama-based chat assistant as supervised, unsupervised, or RL-flavored — and justify each.
2. Explain, without using the word "overfitting," what it would mean for a resume-screening model to have "memorized" its training data rather than learned to screen resumes well.
3. Give one hypothetical feature you'd design by hand for a classic ML model that predicts whether an HR support ticket needs escalation — and explain why a deep learning model wouldn't need you to specify it.
