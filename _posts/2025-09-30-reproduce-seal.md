---
layout: post
title: "Reproducing <em>Self-Adapting Language Models</em>: A Two-Step View and Observations from a Small-Scale Reproduction"
date: 2025-09-30
author: Wentao Zhang (w564zhan@uwaterloo.ca)
tags: [NLP, LLM, Research]
---
<span class="post-author"> Original Paper:<a href="https://arxiv.org/pdf/2506.10943">https://arxiv.org/pdf/2506.10943</a> by Adam Zweiger, Jyothish Pari, Han Guo, Ekin Akyürek, Yoon Kim, Pulkit Agrawal </span>

<span class="post-author"> Our Reproduction Code: <a href="https://github.com/yuntian-group/self_edit_adpt_lm_reproduce">https://github.com/yuntian-group/self_edit_adpt_lm_reproduce</a> </span>

## TLDR

We reproduced the paper's knowledge integration experiments and extended them to additional models. Our findings are:

- The results are **reproducible**. We observed improvements slightly above the reported numbers.
- SEAL has two steps:
  (A) Self-editing — based on new external information, the model generates edits and finetunes on them.
  (B) Training the model to be a better teacher of itself — the model is trained using RL to generate better edits that can more effectively teach itself.
- On **instruction-tuned models**, Step A alone yielded most of the observed improvements. Step B added modest additional gains but is computationally expensive.
- Step B is slow in practice, even after a 3x speedup via our Ray-based re-implementation, because each reward computation requires finetuning.

Overall, our reproduction confirms that both Step A and Step B can work. However, our results suggest that strong instruction-tuned models may already be effective at generating self-edits to teach themselves, reducing the need for further RL-style training to improve their self-teaching ability.


## SEAL in Two Steps

### The Goal
Language models are trained on static data, so when the world changes they may fall behind. The goal of SEAL (*Self-Adapting Language Models*) is to give models a way to **incorporate new knowledge into their parameters** so they can answer correctly without needing external prompts or retrieval.

---

### Step A — Self-editing
When new knowledge appears (e.g., a news article), the model generates **self-edits**---training examples that can be used for finetuning.  

For instance, suppose the news reads:

> Apple Launches iPhone 17

If we then ask an old model *"What is the latest iPhone?"*, it might still answer *"iPhone 16"*.  
With self-editing, the model itself generates a new training pair:

```
Q: What is the latest iPhone?
A: iPhone 17
```


Finetuning on such edits updates the model so that it answers correctly going forward.

---

### Step B — Training the model to be a better teacher of itself
Step A assumes the model can already generate useful edits — but SEAL makes an important implicit assumption: **in practice, models are not strong enough at generating good self-edits on their own.** We will revisit this assumption later. For now, let's assume the model is not good at generating data to teach itself.

To address this, SEAL introduces Step B: training the model to improve its ability to generate effective edits. This is done through RL.

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

If we finetune on Version 1, the model still cannot answer *“What is the latest iPhone?”* correctly.
If we finetune on Version 2, it now answers correctly.

In SEAL, Version 1 would receive **reward = 0** and Version 2 **reward = 1**.
The training loop then nudges the model toward producing more Version 2–style edits — effectively making it a **better teacher of itself**.

---

**How this "RL" works in practice.**

Although described as reinforcement learning, the procedure reduces to a form of supervised training on successful self-edits. The loop is essentially:
1. Generate multiple candidate self-edits.
2. Finetune the model separately on each and evaluate performance.
3. Keep the edits that improved performance, discard the rest.
4. Train the model again on the higher-reward edits.

So, in practice, RL here functions like sample -> finetune (train to incorporate new knowledge) -> evaluate -> filter -> meta-finetune (train to be better at teaching).

---

**Key distinction.**
- Step A: *directly injects new knowledge into model parameters through self-edits.*
- Step B: *optimizes the model's ability to generate better self-edits that can more effectively teach itself.*



## Why did we reimplement it?
While the paper provided code, we found that training under the framework is very slow: even when only using 50 examples in squad for training, it took **3 hours** to run one round of RL (on 2 L40s), or **30 hours per training run** when we perform 10 rounds of RL. For each text input, we need to rollout different generations, and we need to finetune the model to compute the reward (in the actual implementation, 3 times of finetuning per rollout to compute average reward).

To accelerate training and make experiments feasible, we rewrote the training framework using Ray, which enables parallel SFT training and flexible worker scheduling. The below figure illustrates the pipeline how the pipeline runs:

<div class="responsive-image-container">
    <img src="/blog/2025/09/30/reproduce-seal/implementation_updated.png" alt="SEAL Implementation Pipeline" class="responsive-image"/>
</div>

By fully using GPUs, we achieved a 3x speed up, reducing each training run from 30 hours to 10 hours.


## Are the results in the original paper reproducible?
We'd say, yes! At least on the main knowledge incorporation setting, our reproduction observes even slightly better improvements from training the model to be a better teacher of itself, as shown below.

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

Note: We reported the best performance by taking the best performance over all RL rounds (i.e., using the test set for best checkpoint selection). A separate validation set should ideally be used for best checkpoint selection.


## Is training the model to be better teachers of themselves really necessary?
One surprising finding is that as models get stronger—especially instruction-tuned models—the improvement from RL (training the model to be a better teacher of itself) becomes moderate to small compared to base models.

<div class="responsive-image-container">
    <img src="/blog/2025/09/30/reproduce-seal/result-with-gpt5.png" alt="Model Performance Comparison" class="responsive-image"/>
</div>

From the above bar plot we can see that for instruction-tuned models, the improvement brought by RL training the model to be better teachers of themselves becomes moderate to small, compared to base models.

We also provide the result self-edit generated by GPT-5 with **low** reasoning effort. The result is quite impressive for all models, which means it is a good choice to directly use GPT-5. 


The following table presents the same results in numbers (with an added boolean column right before the improvement column: Instruction-Tuned).

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
<td data-label="Model">Qwen 2.5 3B (Reported by Author)</td>
<td data-label="Adapting using Vanilla Model">31.93%</td>
<td data-label="Adapting using Trained Model">36.98%</td>
<td data-label="Is-Instruction-Tuned">No</td>
<td data-label="Improvement">5.05%</td>
</tr>
<tr>
<td data-label="Model">Qwen 2.5 3B</td>
<td data-label="Adapting using Vanilla Model">31.31%</td>
<td data-label="Adapting using Trained Model">38.91%</td>
<td data-label="Is-Instruction-Tuned">No</td>
<td data-label="Improvement">7.60%</td>
</tr>
<tr>
<td data-label="Model">Qwen 2.5 3B Instruct</td>
<td data-label="Adapting using Vanilla Model">54.11%</td>
<td data-label="Adapting using Trained Model">56.47%</td>
<td data-label="Is-Instruction-Tuned">Yes</td>
<td data-label="Improvement">2.36%</td>
</tr>
<tr>
<td data-label="Model">Qwen 3 0.6B (Instruct)</td>
<td data-label="Adapting using Vanilla Model">40.55%</td>
<td data-label="Adapting using Trained Model">43.53%</td>
<td data-label="Is-Instruction-Tuned">Yes</td>
<td data-label="Improvement">2.98%</td>
</tr>
<tr>
<td data-label="Model">Qwen 3 4B (Instruct)</td>
<td data-label="Adapting using Vanilla Model">64.78%</td>
<td data-label="Adapting using Trained Model">67.15%</td>
<td data-label="Is-Instruction-Tuned">Yes</td>
<td data-label="Improvement">2.37%</td>
</tr>
<tr>
<td data-label="Model">Llama 3.2-1B-Instruct</td>
<td data-label="Adapting using Vanilla Model">37.98%</td>
<td data-label="Adapting using Trained Model">38.91%</td>
<td data-label="Is-Instruction-Tuned">Yes</td>
<td data-label="Improvement">0.93%</td>
</tr>
</tbody>
</table>
</div>


## Which One to Choose In Term of Cost ? 

To train 1 round of RL-based learning, we used 2 L40S for 1 hour, which may cost $0.88 (almost lowest price we can find), 10 rounds may take around $8.8 . But if direct calling GPT-5 API, we have 44496 input tokens, and 143673 output tokens. This convert to $1.492, or $0.746 when using batch request. It seems like calling GPT-5 is saving time and money compare with the self editing proposed. More efficient algorithm need to be designed to achieve better performance. 


## Why Instructional Finetuned Models Peform Better ?
From the tables and figures above, we can find that instructional-finetuned model can perform self-edit much better then non instructional-finetuned model, like Qwen2.5-3B vs Qwen2.5-3B-Instruct. The reason could be instructional finetuned models are already good enough for generating self-edits, at least for simple dataset like SQuAD, the gain of performing RL is small. For base model, it is more like improve models' specific instruction following capability, and lead to better performance, but it can still not make the model comparable with instructional fine-tuned one.
The base models' performance seems not decaying or improving significantly, which means RL training neither helping the knowledge incorporation, nor hurting it.


Another observation is the proposed approach can not bring improvement to the instructional fine-tuned model, this may requires further exploration. One reason can be we perform too small sampling due to hardware constraint (But its hard to scale up, because we need to perform fine-tuning). The algorithm itself may also be not good enough for training the model.


