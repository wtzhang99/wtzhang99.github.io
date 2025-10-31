---
layout: post
title: "Reproducing <em>Self-Adapting Language Models</em>"
date: 2025-09-30
author: Wentao Zhang (w564zhan@uwaterloo.ca)
tags: [NLP, LLM, Research]
---
<span class="post-author"> Original Paper:<a href="https://arxiv.org/pdf/2506.10943">https://arxiv.org/pdf/2506.10943</a> by Adam Zweiger, Jyothish Pari, Han Guo, Ekin Akyürek, Yoon Kim, Pulkit Agrawal </span>

<span class="post-author"> Our Reproduction Code: <a href="https://github.com/yuntian-group/self_edit_adpt_lm_reproduce">https://github.com/yuntian-group/self_edit_adpt_lm_reproduce</a> </span>

## TLDR

We reproduced the paper's knowledge integration experiments and extended them to additional models. Our findings are:

* The results are **reproducible**.
* SEAL consists of two steps:
  (A) **Self-editing** — the model generates edits based on new external information and is fine-tuned on them.
  (B) **Reinforcement learning (RL)** — the model is trained to become a better self-editor by optimizing its edit quality through RL.
* For **instruction-tuned models**, almost all improvements came from Step A. Step B added little benefit despite being far more expensive.
* Even with a 3× speedup from our Ray-based reimplementation, Step B remains slow because each reward computation requires fine-tuning.

In short, **self-editing alone provides most of the benefit**. Simply prompting a strong model to generate edits is far more cost-effective than training it via RL to become a better self-editor.

---

## SEAL in Two Steps

### The Goal
The goal of SEAL (Self-Adapting Language Models) is to enable models to **internalize new knowledge into their parameters.**

---

### Step A — Self-editing

To incorporate new knowledge, the model generates **self-edits**—training examples that can be used for fine-tuning.

For example, suppose the news reads:

> Apple launches iPhone 17.

If we ask an outdated model *“What is the latest iPhone?”*, it might still answer *“iPhone 16.”*
With self-editing, the model itself generates a new training pair:

```
Q: What is the latest iPhone?
A: iPhone 17
```

Fine-tuning on such edits updates the model so it answers correctly in the future.

---

### Step B — Training the model to be a better self-editor

Step A assumes the model can already generate useful edits—but SEAL makes an important assumption: **in practice, models are initially not strong enough at generating good self-edits on their own.** We will revisit this assumption later. 

To address this, SEAL introduces Step B: training the model to improve its ability to generate effective edits through RL.

For intuition, suppose we prompt the model to sample self-edits from the same news article. We might sample two different versions using two different random seeds:

- **Version 1:**
  ```
  Q: What is the news about?
  A: Apple.
  ```

- **Version 2:**
  ```
  Q: What is the latest iPhone?
  A: iPhone 17.
  ```

If we fine-tune on Version 1, the model still cannot answer *“What is the latest iPhone?”* correctly.
If we fine-tune on Version 2, it now answers correctly.

SEAL uses the outcome after fine-tuning as the reward. Version 1 would receive **reward = 0** because the fine-tuned model couldn't answer correctly and Version 2 **reward = 1**.
The training loop then nudges the model toward producing more Version 2–style edits — effectively making it a **better self-editor**.

---

**How this RL works in practice ?**

Although described as reinforcement learning, the procedure reduces to a form of supervised training on successful self-edits. The loop is essentially:

1. Generate multiple candidate self-edits.
2. fine-tune the model separately on each and evaluate performance.
3. Keep the edits that improved performance, discard the rest.
4. Fine-tune the model on edits from step 3.

In practice, this process looks like: *sample → fine-tune (incorporate new knowledge) → evaluate → filter → meta-fine-tune (learn to teach itself better).*

---

**Key distinction:**

- **Step A:** *directly injects new knowledge into model parameters through self-edits.*
- **Step B:** *optimizes the model's ability to generate better self-edits that can more effectively teach itself.*

---

## Our Faster Reimplementation in Ray

While the original paper provided code, we found training to be very slow: even with only 50 examples, one RL round took **3 hours** on two L40 GPUs. Running 10 RL rounds required roughly **30 hours per experiment**.

For each text input, we need to rollout different generations, and we need to fine-tune the model to compute the reward (in the actual implementation, 3 times of finetuning per rollout to compute average reward).

To accelerate training and make experiments feasible, we reimplement the training framework using **Ray**, which enables parallel SFT training and flexible worker scheduling. The below figure illustrates the pipeline how the pipeline runs:

<div class="responsive-image-container">
    <img src="implementation_updated.png" alt="SEAL Implementation Pipeline" class="responsive-image"/>
</div>

By fully using GPUs, we achieved a 3x speed up, reducing each training run from 30 hours to 10 hours.

---

## Are the results in the original paper reproducible?

We'd say, yes! At least in the knowledge incorporation setting.

<div class="responsive-table-container">
<table class="responsive-table">
<thead>
<tr>
<th>Model</th>
<th>Adapting using Vanilla Model</th>
<th>Adapting using Trained Model</th>
<th>Improvement</th>
</tr>
</thead>
<tbody>
<tr>
<td data-label="Model">Qwen 2.5 3B (Reported by Author)</td>
<td data-label="Adapting using Vanilla Model">31.93%</td>
<td data-label="Adapting using Trained Model">36.98%</td>
<td data-label="Improvement">5.05%</td>
</tr>
<tr>
<td data-label="Model">Qwen 2.5 3B</td>
<td data-label="Adapting using Vanilla Model">31.31%</td>
<td data-label="Adapting using Trained Model">38.91%</td>
<td data-label="Improvement">7.60%</td>
</tr>
</tbody>
</table>
</div>

Note: The original paper didn't do checkpoint selection. Here we selected the best checkpoint across RL rounds. One caveat is we used test set for checkpoint selection due to the lack of a pre-defined validation set. Ideally, we should reserve part of training set for validation.

---

## Is training the model to be better self-editors really necessary?

Our surprising finding is that as models as models become stronger (e.g., instruction-tuned rather than base models), the improvement from RL (training the model to be a better self-editor) becomes smaller.

<div class="responsive-image-container">
    <img src="result-instruct-vs-base.png" alt="Model Performance Comparison" class="responsive-image"/>
</div>

<!-- From the above bar plot we can see that for instruction-tuned models, the improvement brought by RL training 
the model to be better self-editors of themselves becomes moderate to small, compared to base models. -->

In the above plot, compare the red bars (without RL) to the yellow bars (with RL), they are not much different especially for instruction-tuned models. 

The following table presents the same results in numbers (with a column: Is-Instruction-Tuned). From this table you can see the improvement from RL is smaller for instruction-tuned models.

<div class="responsive-table-container">
<table class="responsive-table">
<thead>
<tr>
<th>Model</th>
<th>Adapting using Vanilla Model</th>
<th>Adapting using Trained Model</th>
<th>Is-Instruction-Tuned</th>
<th>Improvement</th>
</tr>
</thead>
<tbody>
<tr>
<td data-label="Model">Qwen 2.5 3B-Base (Original Paper)</td>
<td data-label="Adapting using Vanilla Model">31.93%</td>
<td data-label="Adapting using Trained Model">36.98%</td>
<td data-label="Is-Instruction-Tuned">No</td>
<td data-label="Improvement">5.05%</td>
</tr>
<tr>
<td data-label="Model">Qwen 2.5 3B-Instruct (Ours)</td>
<td data-label="Adapting using Vanilla Model">54.11%</td>
<td data-label="Adapting using Trained Model">56.47%</td>
<td data-label="Is-Instruction-Tuned">Yes</td>
<td data-label="Improvement">2.36%</td>
</tr>
<tr>
<td data-label="Model">Qwen 3 0.6B-Instruct (Ours)</td>
<td data-label="Adapting using Vanilla Model">40.55%</td>
<td data-label="Adapting using Trained Model">43.53%</td>
<td data-label="Is-Instruction-Tuned">Yes</td>
<td data-label="Improvement">2.98%</td>
</tr>
<tr>
<td data-label="Model">Qwen 3 4B-Instruct (Ours)</td>
<td data-label="Adapting using Vanilla Model">64.78%</td>
<td data-label="Adapting using Trained Model">67.15%</td>
<td data-label="Is-Instruction-Tuned">Yes</td>
<td data-label="Improvement">2.37%</td>
</tr>
<tr>
<td data-label="Model">Llama 3.2-1B-Instruct (Ours)</td>
<td data-label="Adapting using Vanilla Model">37.98%</td>
<td data-label="Adapting using Trained Model">38.91%</td>
<td data-label="Is-Instruction-Tuned">Yes</td>
<td data-label="Improvement">0.93%</td>
</tr>
</tbody>
</table>
</div>

---

## Our Recommendation

From the above results, most improvements come from self-editing the model rather than from the expensive RL training. Our first recommendation is to **prompt the model to generate self-edits directly, instead of training it to become a better self-editor.**

<div class="responsive-image-container">
    <img src="result-instruct-vs-base.png" alt="Model Performance Comparison" class="responsive-image"/>
</div>

We also ran an additional experiment showing that using an external editor can achieve comparable or even better results at a fraction of the cost.
For example, the figure above shows the results of using GPT-5 to generate edits with **low reasoning effort**, which still perform very well overall. Moreover, using GPT-5 is significantly cheaper than the RL training process.

Training one round of RL required two L40S GPUs for one hour, costing $0.88 (the lowest available rate). Ten rounds cost $8.80.
In contrast, directly calling the GPT-5 API used 44,496 input tokens and 143,673 output tokens, costing only **$0.746** with batch requests.

This demonstrates that **using strong LLM APIs can be a highly effective and cost-efficient alternative** to RL training for most use cases.




<!--
## Why Instructional fine-tuned Models Peform Better ?
From the tables and figures above, we can find that instructional-fine-tuned model can perform self-edit much better then non instructional-fine-tuned model, like Qwen2.5-3B vs Qwen2.5-3B-Instruct. The reason could be instructional fine-tuned models are already good enough for generating self-edits, at least for simple dataset like SQuAD, the gain of performing RL is small. For base model, it is more like improve models' specific instruction following capability, and lead to better performance, but it can still not make the model comparable with instructional fine-tuned one.
The base models' performance seems not decaying or improving significantly, which means RL training neither helping the knowledge incorporation, nor hurting it.


Another observation is the proposed approach can not bring improvement to the instructional fine-tuned model, this may requires further exploration. One reason can be we perform too small sampling due to hardware constraint (But its hard to scale up, because we need to perform fine-tuning). The algorithm itself may also be not good enough for training the model.-->


