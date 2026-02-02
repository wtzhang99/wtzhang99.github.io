---
layout: post
title: "Program-as-Weights: What If Your Code <em>Was</em> the Neural Network?"
date: 2026-01-30
author: Wentao Zhang (w564zhan@uwaterloo.ca)
tags: [NLP, LLM, Research, Neural Programming]
---
<!-- <span class="post-author"> Original Paper: <a href="#">[Link to paper when available]</a> </span>

<span class="post-author"> Code & Demo: <a href="#">programasweights.com</a> | <a href="#">pip install programasweights</a> </span> -->

## TLDR

* **Program-as-Weights (PAW):** *Compile* natural language function descriptions into neural network weights—for tasks that are impossible to write as clean code.
* **Architecture:** A large "compiler" model reads your spec and produces a tiny weight blob (~7MB). A fixed small "interpreter" model runs it locally—no internet, no API calls.
* **Training:** Neural programs are a hybrid of discrete tokens and continuous vectors. A three-stage pipeline—RL for discrete programs, SFT for continuous components, then joint refinement—is key to making this work.
* **Result:** A 0.6B interpreter running compiled programs matches an 8B LLM prompted directly—13× smaller at inference time.
* **Vision:** Neural binaries can be version-controlled and shared like code, enabling a new software ecosystem.

---

## The Problem: Code Can't Handle Everything

Programming has always been about writing explicit rules. And for most tasks, this works beautifully—sorting lists, computing gradients, rendering UI. But there's a surprisingly large class of functions that resist clean implementation:

- **Extracting the final answer from a chain-of-thought response** (Is it in `\boxed{}`, after "Answer:", or just the last number?)
- **Parsing options like "(A) cat (B) dog"** (What if there's a nested "(A)" inside an option?)
- **Deciding if a review is positive** (What about sarcasm? Mixed feelings?)

These are what researchers call **fuzzy functions**—tasks humans find intuitive but that can't be fully captured by symbolic rules. Every regex you write has edge cases. Every heuristic breaks on some input.

---

### The Current Workaround: Just Call an LLM

```python
def extract_answer(text):
    return gpt("Extract the final answer from: " + text)
```

It works. But it's **expensive**, **fragile** (API outages, silent model updates), **non-reproducible**, and **not self-contained**. What if we could have the *intelligence* of a large model, but packaged as a local, deterministic, shareable artifact?

---

## The Idea: Compile Intent into Weights

**Program-as-Weights (PAW)** introduces a compiler-interpreter architecture for fuzzy functions:

<!-- IMAGE PLACEHOLDER: paw-overview.png
     Suggested: A diagram showing the two-step flow:
     1. User writes "Extract the final answer" → Compiler (4B) → Neural Program blob
     2. Neural Program + Input "...answer is \boxed{42}" → Interpreter (0.6B) → "42"
     Similar to Figure 1 in the paper -->
<div class="responsive-image-container">
    <img src="paw-fig1.png" alt="PAW Overview: Compile then Interpret" class="responsive-image"/>
</div>

1. **Neural Compiler** (large model): Takes a natural language specification and produces a compact "neural program"
2. **Neural Interpreter** (small model, frozen): Executes the program locally

```python
import programasweights as paw
f = paw.compile("Extract the final answer from a reasoning trace")
f.dump("extract_answer.weights")  # version-controllable artifact

f = paw.function("extract_answer.weights")
print(f("... the answer is \\boxed{42}"))  # → "42"
```

The compiled `.weights` file runs locally, deterministically, with no API calls—like shipping a binary.

---

## Neural Program Representation

A neural program isn't just a prompt. It's a **hybrid of discrete and continuous components**:

<div class="responsive-image-row">
    <div class="responsive-image-container">
        <img src="neural-program-discrete.png" alt="Discrete Component: Pseudo-program" class="responsive-image"/>
    </div>
    <div class="responsive-image-container">
        <img src="neural-program-continuous.png" alt="Continuous Component: KV Cache Vectors" class="responsive-image"/>
    </div>
</div>


**Discrete component:** A variable-length sequence of tokens acting as a "pseudo-program." The compiler learns to generate task descriptions and input-output examples that help the interpreter understand the function.

**Continuous component:** A fixed number of continuous vectors extracted from the compiler's hidden states. These are injected as a KV cache prefix into the interpreter's attention mechanism, providing fine-grained control that's hard to express in text.

At execution time, the interpreter sees:
- The continuous vectors as prefix key-value states
- The discrete pseudo-program prepended to the actual input
- Together, these condition the interpreter to implement the fuzzy function

This hybrid design is crucial. Experiments show that **neither component alone works well**—discrete tokens give the interpreter too little control (causing formatting errors), while continuous-only representations lack the structure that explicit examples provide. The combination is synergistic, roughly doubling accuracy compared to either alone.

---

## Training: A Three-Stage Approach

Training the compiler is challenging because we don't have ground-truth neural programs—discovering effective programs is part of the learning problem. The authors use a staged approach.

### The Core Challenge

The goal is to maximize the likelihood of correct outputs given a specification and input. But since we never observe "correct" programs directly, program generation becomes a **latent decision process**. The compiler must discover what programs work by trial and error, guided by whether the interpreter produces correct outputs.

<div class="responsive-image-row">
    <div class="responsive-image-container">
        <img src="rl-train-1.png" alt="Discrete Component: Pseudo-program" class="responsive-image"/>
    </div>
    <div class="responsive-image-container">
        <img src="sft-train-2.png" alt="Continuous Component: KV Cache Vectors" class="responsive-image"/>
    </div>
</div>


---

### Stage 1: RL for Discrete Pseudo-Programs

First, train the compiler to generate effective discrete pseudo-programs only (no continuous component yet).



The compiler learns to act as a **prompt engineer** for the interpreter. Using policy gradient with Group Relative Policy Optimization (GRPO), it samples multiple candidate programs, evaluates which ones lead to correct outputs, and updates toward better programs.

**What the compiler discovers:** Regardless of initialization, the compiler converges to a consistent strategy—paraphrase the task specification, then generate as many input-output examples as fit in the token budget. This emergent behavior is remarkably robust.

**The limitation:** Discrete programs alone provide insufficient control when the interpreter is small. The interpreter still makes formatting errors and misses edge cases. This motivates adding continuous components.

---

### Stage 2: SFT for Continuous Programs

Next, train the compiler to generate continuous vectors while keeping discrete programs fixed.

The key insight is that continuous vectors can capture nuances that are hard to express in text. To avoid the instability of online RL, this stage proceeds offline: sample and store discrete programs from Stage 1, then train the continuous component via supervised fine-tuning to maximize interpreter accuracy.

**The drift problem:** Updating the compiler risks changing what discrete programs it would generate, creating a mismatch with the stored programs. The solution is to add a regularization term that encourages the compiler to stay close to its original discrete-generation policy.

**Impact:** This stage provides the largest accuracy gain—continuous representations give substantially finer control than discrete prompts alone, more than doubling performance.

---

### Stage 3: Joint Refinement

Despite regularization, some mismatch between discrete and continuous program generation may remain. A final joint training stage optimizes both components together.

At this point, both components are well-initialized, so only a few additional gradient steps are needed to align them. This final refinement provides modest but consistent improvements.

---

## Why This Works: Shifting Complexity to Compile Time

Fuzzy functions *do* require substantial knowledge—linguistic patterns, formatting conventions, edge case handling. The question is **where that knowledge lives**.

<div class="responsive-table-container">
<table class="responsive-table">
<thead>
<tr>
<th>Approach</th>
<th>Where Knowledge Lives</th>
<th>Tradeoff</th>
</tr>
</thead>
<tbody>
<tr>
<td data-label="Approach">Hand-written code</td>
<td data-label="Where Knowledge Lives">In the programmer's head</td>
<td data-label="Tradeoff">Brittle, tedious, incomplete</td>
</tr>
<tr>
<td data-label="Approach">LLM API calls</td>
<td data-label="Where Knowledge Lives">In a remote 100B+ model</td>
<td data-label="Tradeoff">Expensive, fragile, non-reproducible</td>
</tr>
<tr>
<td data-label="Approach">Fine-tune small models</td>
<td data-label="Where Knowledge Lives">In collected training data</td>
<td data-label="Tradeoff">Data collection + training per task</td>
</tr>
<tr>
<td data-label="Approach"><strong>PAW</strong></td>
<td data-label="Where Knowledge Lives">In a compiled weight blob</td>
<td data-label="Tradeoff">Local, cheap, no data collection</td>
</tr>
</tbody>
</table>
</div>

PAW **amortizes** the complexity: a powerful compiler does the hard work once, producing a compact artifact that runs cheaply forever.

---

## Results: Small Model, Big Capability

<!-- IMAGE PLACEHOLDER: results-comparison.png
     Suggested: A bar chart comparing PAW (small interpreter) vs local LLMs of various sizes,
     showing PAW matching much larger models on both text and image tasks -->
<div class="responsive-image-container">
    <img src="result.png" alt="PAW vs Larger Models" class="responsive-image medium"/>
</div>

The core finding: **a tiny 0.6B interpreter running PAW programs matches or exceeds the performance of models 13× its size.**

On text tasks (80K+ diverse fuzzy functions), PAW with a 0.6B interpreter matches an 8B LLM prompted directly. On image tasks (formula-to-LaTeX, molecule-to-SMILES, visual QA), the same small interpreter outperforms comparably-sized vision-language models—the compiler effectively "teaches" a text-only model to handle visual inputs.

This efficiency gap is the key advantage: **you get large-model intelligence at small-model inference cost.** The expensive reasoning happens once at compile time; execution is cheap forever after. PAW also significantly outperforms program synthesis approaches that generate Python code—symbolic heuristics simply can't handle the linguistic variability that neural programs capture naturally.

---

## The Bigger Picture

Program-as-Weights represents a conceptual shift: **programming fuzzy logic isn't about enumerating rules, but specifying intent**.

The compiler acts as a teacher that distills task-specific intelligence into a form a small model can execute. This cleanly separates:

- **What intelligence costs to create** (expensive, one-time compilation)
- **What intelligence costs to run** (cheap, repeated inference)

As foundation models improve, compilers produce better programs without changing the interpreter. As edge devices get faster, the same programs run more efficiently. The architecture is future-proof in both directions.

---

## Final Thoughts

We're used to thinking of neural networks as opaque, monolithic systems. PAW shows they can be something else: **modular, shareable, composable artifacts** that behave like functions.

Write what you want in English, compile it into weights, run it anywhere.

This isn't about replacing symbolic programming—it's about extending what "programming" can mean. For fuzzy functions that resist clean implementation, we now have a principled alternative to "just call GPT."


