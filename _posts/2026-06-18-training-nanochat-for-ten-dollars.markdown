---
title: "Training nanochat for ten dollars, and the wall I hit"
layout: post
date: 2026-06-18 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - llms
  - nanochat
  - transformers
  - rl
  - learning
category: blog
author: samuelgnap
description: I ran the nanochat training pipeline end-to-end on a single H100 — four stages, two and a half hours, about ten dollars of compute. Three stages worked. The fourth hit a precision wall I didn't expect. This is the honest writeup of what each stage does, what the model can do, and the gotcha at the end.
---

A few weeks after reading nanochat I wanted to actually run it. The point of reading Karpathy's code was to understand the shape of a modern training pipeline. The plan was the speedrun: rent one GPU H100 on Vast.ai, follow the script, get to a model you can talk to. There are four stages in total. 1. train a BPE tokenizer, 2. pretrain a small transformer on web text, 3. supervised-fine-tune it into a chat model, and ;lastly 4. reinforce-learn it on a verifiable task. I takes about 3 hours and costs around 10 dollars, which is quite okay for what you get. 

Three of those four stages worked without any issues but the the RL stage crashed at step two with a CUDA assertion, every time. The rest of this post is what each stage actually does, what the model can do at each one, and what the wall was at the end.

## The pipeline

nanochat trains a chat model in four stages. Each one writes its output to a known directory under `~/.cache/nanochat/` and the next stage finds it there by convention.  That was the first useful thing I learned, because the answer to "where does the SFT script find the base model?" is "it doesn't, it finds it by filename layout," which is exactly how Karpathy keeps the code small.

The four stages:

1. **Tokenizer training.** A couple of GB of web text. Train a byte-pair encoder. Save it.
2. **Base pretrain.** Train a 286 M parameter transformer on ~1.3 B tokens of FineWeb-Edu. Save the weights.
3. **SFT.** Take the base model, fine-tune it on chat-formatted instruction/response data. Save the weights.
4. **RL.** Take the SFT model, do GRPO-flavour reinforcement learning against GSM8K, where reward = "did you get the final number right." Save the weights.

The configuration I ran is the smallest of the standard speedrun sizes — **depth 12** (12 transformer layers, 768-dim residual stream, 286 M parameters, trained for 1.32 B tokens). I chose it to be small enough to debug the full pipeline in 2-3 hours on a single H100, big enough that the model isn't completely incoherent at the end.

Total clock for stages 1-3 was 2h27m at $3/hour, so about **$7.35**. The RL attempts added another couple of dollars in failed runs.

## Stage 1: Tokenizer training

This was the fastest stage by far. About 1 minute 13 seconds to train, on 2 billion characters of text. The output is a BPE tokenizer with a 32,768-token vocabulary and 9 special tokens (BOS, EOS, `<|user_start|>`, etc.).

The interesting numbers come from comparing it with GPT-2 and GPT-4's tokenizers on different text types. Karpathy's eval feeds a fixed number of bytes through each and reports the bytes-per-token ratio (higher = better compression = fewer tokens you spend on the same content).

Mine vs. GPT-2:

| Text type | GPT-2 ratio | Ours ratio | Delta |
|---|---|---|---|
| News | 4.50 | 4.49 | -0.2% |
| Code | 2.19 | 3.18 | **+31.2%** |
| Math | 1.96 | 2.01 | +2.7% |
| Science | 4.28 | 4.50 | +5.0% |
| Korean | 1.20 | 1.21 | +0.5% |

Where I beat GPT-2 by a wide margin is on code: 31% better compression. This is because GPT-2's tokenizer was trained on 2018-vintage web text, which underweighted code; modern pretraining corpora like FineWeb are code-heavy by virtue of containing more StackOverflow, GitHub, and tutorial content. The byte-pair encoder soaks up code-specific merges (`def `, `import `, `self.`, etc.) and you save real tokens on every Python file.

On Korean text my tokenizer needs almost twice as many tokens — GPT-4's tokenizer has explicit CJK coverage; mine doesn't, because FineWeb-Edu is overwhelmingly English. That's a real cost if you ever want to do Korean inference with this model: you'd retrain the tokenizer on a multilingual corpus first.


## Stage 2: Base pretrain

This is the stage that takes most of the budget. Wall clock 46 minutes, ~$2.30, at 37% model FLOPs utilization on the H100.

Configuration:

- 12 layers, 6 heads, head_dim 128, hidden 768
- 286 M parameters
- 1.32 B training tokens (token:parameter ratio of 12:1)
- Sequence length 2,048
- Window pattern `SSSL` (mostly sliding-window attention with one full-window layer)
- fp8 tensorwise precision

To set the scale: this is roughly **GPT-2 small's architecture** with a few modern touches. Same 12 layers, same 768-dim residual stream. Half as many attention heads (6 instead of 12) but each one is 128-dim wide instead of 64-dim — the Llama-era choice of fewer, wider heads. Double the context window (2,048 vs. 1,024). About 2× the parameter count, mostly from a larger vocabulary and longer position-embedding tables. The training data: **1.3 billion tokens**. GPT-2 saw ~40 B; Llama-3-8B saw 15 T. So our model has seen ~30× less data than GPT-2 and ~11,000× less than a modern frontier model. 

The 12:1 ratio is well past Chinchilla's 20:1 sweet spot on the under-train side. Karpathy explicitly chose under-training because the downstream usefulness of a small model is more sensitive to *base capability per FLOP* (which Chinchilla measures) than to *downstream eval quality* (which under-training improves at fixed FLOP). It's a choice I had to read about, because at first glance "train less than Chinchilla says" looks wrong.

The base model's final validation bits-per-byte was **0.85**. For comparison, GPT-2 (1.5 B parameters, larger by 5×) gets around 0.92 on the same eval; a smaller model on its own tokenizer scoring better makes the bpb comparison less informative than I expected. The right yardstick is CORE, which the eval script gives us at **0.137** — well below GPT-2-large (around 0.27) but in the same ballpark as a scaled-down GPT-2-medium.

What is the base model actually doing? Here are some of the unconditioned samples (model generates with only `<|bos|>` as input):

> The capital of France is Paris, and the capital of the United Kingdom is London. The capital of the...

OK, knows facts.

> If yesterday was Friday, then tomorrow will be Saturday. If yesterday was Saturday, then tomorrow will be Sunday. If yesterday was...

Reasonable temporal logic.

> The opposite of hot is cold. The opposite of hot is cold. The opposite of hot is hot.

And there's the classic small-LM failure: collapse into repetition, then drift into contradicting itself. This sample tells you exactly what a 286 M dense model with 1.3 B tokens of training is — a coherent next-token predictor that hasn't seen enough text to have stable long-range concept tracking. Every modern instruction-following property is going to come from the next stage.

## Stage 3: Supervised fine-tuning

This is the stage I was most curious about, because the difference between "GPT-2-medium" and "ChatGPT" is approximately one round of SFT plus a learned chat template.

SFT trained for 971 steps, 15 minutes wall clock, ~$0.75. The data is a mixture nanochat builds out of `HuggingFaceTB/smoltalk` (general chat), variants of OpenMathInstruct (reasoning), and a small `identity_conversations.jsonl` that teaches the model to say it's nanochat made by Karpathy, not GPT.

Format is the standard chat template:

```
<|user_start|>What is 2 + 2?<|user_end|>
<|assistant_start|>4.<|assistant_end|>
```

What the post-SFT model can do, by benchmark:

| Eval | Score | What this means |
|---|---|---|
| MMLU | 0.31 | Above random (0.25) by a hair. Real knowledge, but sparse. |
| ARC-Easy | 0.37 | A bit above random; barely better than the base. |
| ARC-Challenge | 0.32 | Almost random. |
| GSM8K | 0.04 | Near zero. The model can talk about math; it cannot do math. |
| HumanEval | 0.10 | Writes code-shaped tokens; rarely works. |
| SpellingBee | 0.99 | Can spell. (Not hard.) |
| ChatCORE | 0.24 | Overall "is this a useful chat model" composite. |

Reading these honestly: the 286 M model after SFT is a *legible chat-style responder*. It writes in the right register, follows the chat template, identifies itself correctly, and produces grammatical English. What it doesn't have is meaningful knowledge or reasoning. MMLU at 0.31 is "knows some general facts above noise"; GSM8K at 0.04 is "cannot reliably do grade-school math."

This is fine. It's a d12 dress rehearsal. The whole point of running it is to know what $7 buys you, so that when you spend $30 on a d20 you know what you're paying for.

## Stage 4: RL, and the wall

This is where the post stops being a clean walkthrough.

The RL stage in nanochat is reinforcement learning against **GSM8K**, the dataset of grade-school math word problems. The reward is automatically computable: extract the final number from the model's response, compare to ground truth, reward = 1 if match else 0. No reward model, no human feedback, no judge LLM — just the math itself.

Karpathy's algorithm is GRPO-flavoured but the implementation is closer to vanilla REINFORCE with mean-subtracted advantage. From his own docstring: *"I put GRPO in quotes because we actually end up with something a lot simpler"*. Each step:

1. Pick 16 problems from GSM8K train.
2. Sample 16 completions per problem from the current SFT model.
3. Score each completion (0 or 1).
4. Advantage = reward − mean(reward across the 16 samples).
5. Loss = − Σ logp · advantage on the assistant tokens.
6. Backprop, step.

At my SFT model's Pass@1 of ~1.75% at temperature 1.0, this means roughly 250 of 256 sequences per step have reward 0 and the 5-or-so right ones carry the entire gradient signal. That's a real concern about *learning rate*, but not in itself about *correctness*.

The crash was correctness:

```
RuntimeError: probability tensor contains either `inf`, `nan` or element < 0
  File "engine.py", line 240, in generate
      sampled_tokens = next_ids[:, 0].tolist()
```

Step 0 ran cleanly. Step 1 took one optimizer step. Step 2 tried to sample from the model and `torch.multinomial` rejected the logits as containing NaN. I tried lowering the learning rate 5×. Same crash, same step. The crash position is invariant to LR, which rules out "instability you can tune around."

What I believe is happening: my SFT checkpoint was saved in **fp8 tensorwise** precision. RL loads it and runs it in **bf16**. The cast for forward inference is fine — that's why step 0's eval ran. But after one bf16 gradient update, the weights drift into a regime where the next forward produces something just-barely-NaN deep in the attention path. Karpathy's RL script has no `clip_grad_norm_` and no `torch.isnan(loss)` guard. There is no safety net.

There's also a secondary culprit: torch 2.11. The project's venv shipped a CPU-only torch wheel, so I reinstalled torch with the cu128 wheel — and got 2.11 instead of the 2.9 nanochat is developed against. Torch 2.11 added a hard CUDA assert in `multinomial` for any inf/nan/negative input. Older torch would warn and continue; a small model might recover; you'd never see this crash.

So the wall is a stack of three small things: an fp8/bf16 precision interaction in the checkpoint, missing safety nets in the RL script, and a stricter torch than the one Karpathy wrote against. None of them are intrinsic — adding `clip_grad_norm_(model.parameters(), 1.0)` before `optimizer.step()` and `if not torch.isfinite(loss).all(): continue` before `.backward()` probably fixes it. But I didn't fix it, because the expected gain from RL on a model with Pass@1 = 1.75% is one to two points of pass rate, and that's not a result worth another $5 of compute.

## Talking to the model

Numbers in a benchmark table are abstract. Four prompts on the chat CLI are not. The model I just trained, asked four things — three at temperature 0.6, the creative one at 0.8.

**1. A fact it knows.**

> User: What is the capital of France?
> Assistant: The capital of France is Paris.

Clean. This is the easy regime — Paris is in FineWeb millions of times, the SFT chat template formats the response cleanly, the model genuinely knows this kind of fact.

**2. Two-digit arithmetic.**

> User: What is 17 plus 24?
> Assistant: To find the value of 17 + 24, we need to isolate 17 on one side of the equation. We can do this by subtracting 24 from both sides of the equation, but this is not necessary in this case. We can still solve for 17 by dividing both sides of the equation by 17. This results in 17 + 24 = 3(17). Now, we need to get rid of the 24 that is being multiplied by 17. To do this, we can subtract 24 from both sides of the equation, which gives us 17 + 24 - 24 = 3(17). This simplifies to 3(17) = 3(17), or 17 * 3 = 3(17)...

The answer is 41. The model never reaches it. What it produces is *math-shaped tokens* — the register, the vocabulary, the algebraic notation, even the impulse to "isolate" and "simplify" — with no arithmetic underneath. This is exactly what GSM8K = 0.04 means in practice. The SFT mix has math-tutorial text in it; the model copied the *form* of how that text reads and nothing of the substance. A frontier model would just say 41. Ours has read enough math text to imitate it and not enough to do it.

**3. A short coding task.**

> User: Write a short Python function that returns the square of a number.
> Assistant:
> ```python
> def square(n):
>     return n ** 2
> ```
> This function uses Python's exponentiation operator (`**`), which raises the number to the power of the base. [...]
> You can use this function like this:
> ```python
> print(square(16))  # Output: 16
> ```

The function is correct. The explanation is correct. The example's comment is wrong (`square(16)` is 256, not 16) — the model wrote "Output: 16" because the number 16 was nearby in the buffer and the comment slot wanted *some* number. This pattern (correct code, wrong comment) is a useful tell: the model has memorised the function shape but doesn't simulate execution. A modern coding model would actually compute 16² in its head before writing the comment.

**4. A creative prompt at temperature 0.8.**

> User: Tell me a short story about a cat who learns to fly.
> Assistant: It was a chilly winter morning, but the sun was shining bright, and the misty mornings seemed to whisper secrets to the wind. A young cat named Luna, named whiskers, had just returned from a long flight. She had always been a curious feline, and her mother, a skilled musician, had been studying her for years. [...] His eyes sparkled like diamonds in the mist...

Sentences are vivid. Coherence falls apart by sentence two: "Luna, named whiskers" gives the cat two names; the mother is suddenly a musician for no reason; a few paragraphs later "we met at a tavern" puts a narrator in scene who wasn't there. This is the classic small-LM creative-writing pattern. Each next-token prediction is locally plausible because attention sees the recent few hundred tokens fine. Tracking *which entity is which* across a paragraph requires more attention heads or more layers, so the model can dedicate some of them to entity identity. With 12 layers and 6 heads we don't have the capacity to spare.

**5. Bonus — the identity test.**

> User: Who are you?
> Assistant: I am an AI assistant specialized in editing and improving written content. I specialize in refining written texts by offering suggestions for grammar, style, clarity, and overall structure...

The `identity_conversations.jsonl` in the SFT mix was supposed to teach the model to say "I am nanochat, made by Karpathy." It didn't take. The smoltalk-style "I'm a helpful editing assistant" persona dominated because that persona appears thousands of times in the general SFT mix and the identity dataset is small. SFT data ratios are not a knob you can wave at — if you want a specific persona to win, it needs to be loud enough.

## Why each failure is the shape it is

A 286 M parameter, 12-layer, 6-head, 2,048-context model trained on 1.3 B tokens of web text has very specific failure modes — and the five prompts above hit them cleanly.

- **Facts you can state in one sentence** — fine. The information is in the weights; SFT learned the chat template. 
- **Anything that requires a multi-step computation under the surface** (math, simulation, mentally running code) — broken. The model produces the *register* of the task without performing the task. Closing this gap is a slow S-curve in parameters (~10× bigger) and training tokens (~100× more).
- **Anything that requires tracking entities across more than ~50 tokens degrades**. The model writes fluent sentences indefinitely; it can't keep them consistent. More layers and more attention heads help, because the model needs spare capacity to dedicate to entity bookkeeping. Six heads doesn't leave much.
- **Identity, persona, refusal style** — driven by SFT data ratios. Whatever appears most in the mix wins, regardless of intent.

None of these are bugs. They're what a $7 base plus $1 SFT model can do, by construction. The value of running the dress rehearsal is exactly *seeing the shape of the failure modes per stage*, so that when you spend $30 on the next size up you know what you're paying for.

## What you get for ten dollars

A 286 M parameter chat model that:

- Writes coherent, register-appropriate English.
- Follows the chat template, identifies itself, refuses some obviously harmful prompts.
- Knows the kind of general facts any modern model trained on FineWeb knows ("the capital of France is Paris").
- Cannot reliably do math past single-digit arithmetic.
- Cannot write working code past trivial syntactic templates.
- Has no real long-range reasoning.

This is roughly GPT-2-medium-after-an-instruction-tune, with a better tokenizer and a slightly more modern stack underneath. For ten dollars and three hours, it's a very good deal. For a model you'd actually deploy, it's a starting point you'd train a lot more.