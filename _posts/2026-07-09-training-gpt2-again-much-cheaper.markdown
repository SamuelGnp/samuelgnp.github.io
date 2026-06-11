---
title: "Training GPT-2 again, much cheaper: what nanochat changes and why"
layout: post
date: 2026-07-09 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - llms
  - transformers
  - nanochat
  - learning
category: blog
author: samuelgnap
description: Reading Karpathy's nanochat as a diff against GPT-2 — RoPE, sliding-window attention, what got deleted and why, and the data that actually turns an autocomplete into a chatbot.
---

In [the previous post](/how-gpt2-actually-works/) we went through how GPT-2 actually
works, piece by piece. The natural next question: if you wanted to train it again
today, what would you do differently?

It turns out someone has answered that question very thoroughly. Andrej Karpathy
released **nanochat** — a brilliant piece of work: a complete, readable codebase where
you can train your own ChatGPT-style model, end to end, for about $100. Not a toy
either — a model you can actually chat with.

How is that possible? Two reasons. The first is simply that compute got dramatically
cheaper — the GPU boom of the last few years means a dollar buys orders of magnitude
more FLOPs than it did in 2019. The second is everything the field has learned since:
dozens of changes inside and around the architecture, each one buying a little more
model per dollar. That second part is what this post is about.

(To be clear about the ceiling: a few hundred million parameters gets you a model
that performs surprisingly well, but the models that feel genuinely good need billions
of parameters — and that's still out of reach for the retail end of the market, unless
you were lucky with an Nvidia investment ten years ago and don't know what to do with
the money.)

When I read the nanochat code, I kept my own notes of every place where it diverges
from the GPT-2 architecture I described last time — what changed, where in the code,
and why. This post is those notes, organized. We'll go through the architecture first,
and then the part that's discussed far less than it should be: the data.

## 1. What changed inside the architecture

When I first diffed nanochat's model against GPT-2 in my notes, the list of changes
looked long and random. It stops looking random once you sort it by motivation: some
changes handle position better, some delete things that no longer earn their place,
and some are there purely because they get you more model for the same compute.

### Position: from learned embeddings to RoPE

This was the change that surprised me most, because it moves position handling to a
completely different place. GPT-2 added a learned position vector to each token once,
at the very bottom, and then hoped that signal survived twelve layers of processing.
nanochat has no positional embeddings at all. Instead, position enters *inside every
attention layer*, through something called rotary position embeddings — RoPE.

The problem with GPT-2's approach is that absolute position is the wrong question.
Grammar doesn't care that a word is the 1,500th token of the document; it cares that
the adjective sits *right before* this noun. What attention actually needs is the
**distance between tokens**, not their coordinates. On top of that, every absolute
position had to be learned: the vector for position 1,500 is only as good as the
number of times position 1,500 showed up in training.

RoPE's trick is to *rotate* each token's query and key vectors by an angle
proportional to the token's position. This is why you see sines and cosines in the
code — not for speed, but because that's simply what rotation is mathematically. And
here the magic happens: when you take the dot product of two rotated vectors, the
result depends only on the **difference** between their angles. Token 100 attending to
token 97 produces exactly the same positional signal as token 1,000 attending to token
997\. Relative distance falls out of the geometry, with **zero learned parameters**.

The mental image that made it click for me: every pair of channels in the vector is a
clock hand, each spinning at its own speed as you move through the sequence. When two
tokens compare themselves, they compare clock hands — and what they read off is *how
much time passed between them*, not what time it is.

### Deleted: things that no longer earn their place

Quite a few GPT-2 components are simply gone.

**Dropout.** Dropout is regularization — protection against overfitting — and GPT-2
needed it because it went over its small dataset many times. nanochat's data is a
different situation entirely: a much larger, much cleaner corpus, and the model trains
on it for roughly one epoch. Each token is seen about once, so overfitting isn't
something you need to worry about, and dropout goes with it.

**LayerNorm → RMSNorm.** GPT-2 normalized the vectors flowing through the network with
LayerNorm, which does three things: subtracts the mean, divides by the spread, and
applies a learned scale and shift. Over time, people in the community noticed that the
only part actually doing the work is dividing by the spread. RMSNorm keeps just that
one operation, and it's faster.

There are a few other deletions — the bias vectors on the linear layers, the weight
tying between the input and output embeddings — but honestly they don't change the
picture much, so I won't go through them.

One more difference worth mentioning here, even though it's not a deletion: the number
format. GPT-2 trained in 32-bit floats; nanochat computes in bf16, a 16-bit format
that modern GPUs are specifically built to chew through. Half the bits means roughly
half the memory and much faster math, and a couple of small tweaks in the architecture
(extra normalizations, a cap on the output logits) keep training stable at the lower
precision.

### Sliding-window attention

This is the change I find genuinely clever, so I want to spend more time on it.

Recall how attention worked in GPT-2: in every layer, every token computes a score
against *every token before it* and pulls in information from all of them. That full
lookback is what makes transformers powerful, but it's also where the cost lives — the
work grows with how far back each token looks, in every single layer, and it's a big
part of why long contexts are expensive.

Now the observation: when you actually look at what trained models attend to, most
layers spend most of their attention budget on *nearby* tokens. That makes sense —
most of the work of understanding language is local. Resolving syntax, completing a
phrase, matching an adjective to its noun: none of that needs to see 2,000 tokens back.
Genuinely long-range lookups — what was the question at the start of this document? —
matter, but they're the exception, not the rule. So paying for full attention in every
layer means paying full price for something most layers barely use.

nanochat's answer is a pattern it calls `SSSL`. Three out of every four layers are
"short": they only attend over the most recent quarter of the context window. Every
fourth layer — and always the final one — is "long" and sees the full context. You
keep the capability where it's needed and stop paying for it where it isn't.

The natural worry is that the short layers lose access to long-range information. Two
things prevent that. First, the residual stream: once a long layer picks something up
from 2,000 tokens back, that information is *written into the token's vector* and
carried forward through all subsequent layers — short layers can read it without ever
looking that far back themselves. Second, short windows compound across depth. If
layer 1 lets each token see 512 tokens back, then by layer 2 each of those visible
tokens has *itself* already absorbed its own 512-token neighborhood — so the effective
reach keeps growing layer by layer, the same way a CNN's receptive field grows with
depth.

What I like most about this one is that it isn't a small-model trick. The same idea —
most layers local, a few layers global — is how current production models are built
too: Mistral shipped with sliding-window attention, Gemma interleaves local and global
layers, and OpenAI's open-weight models alternate between the two. nanochat is using
the same playbook as the frontier here, just at 1/1000th the size.

Beyond that, there's a longer tail of smaller tricks in the codebase — a different
activation function in the feedforward, learned scalars on the residual stream, a
second embedding table feeding into attention. Each one is worth a percent or so in
the speedrun benchmarks, and they matter in aggregate, but none of them change how you
think about the model, so I'll leave them as a list to explore in the code.

## 2. What changed outside the architecture

The architecture is only half the story — and possibly the smaller half. A few things
around it changed too, and at least two of them matter more than most of the
architecture section.

**The optimizer.** GPT-2 was trained with Adam; nanochat uses **Muon** for all the
matrices in the transformer blocks, with AdamW kept only for the embeddings and the
various scalar knobs — each group with its own learning rate. I won't go deep on how
Muon works (it operates on whole matrices rather than individual weights, keeping their
updates well-conditioned), but it's widely credited as one of the single biggest wins
of the whole speedrun era. The general lesson stuck with me: *how* you descend the
loss landscape matters as much as what model you're descending it with.

**Precision and kernels.** I mentioned bf16 in the architecture section; the other
half of that story is the software that runs it. Attention runs on Flash Attention 3,
an implementation engineered around the GPU's memory hierarchy — it computes exactly
the same attention as GPT-2 did, several times faster, by being careful about when
data moves between fast and slow memory. There's even an experimental fp8 path in the
codebase: every halving of precision is roughly a doubling of speed.

**Hardware.** GPT-2 trained on the GPUs of 2019. nanochat's reference run uses a node
of H100s, each of which does more useful work per second than an entire small cluster
from that era. And the theme running through everything above — bf16, Flash Attention,
the fused kernels — is really one goal: keep those GPUs *busy*. An H100 can do an
enormous amount of math per second, but only if data keeps arriving fast enough to
feed it; a large fraction of modern training engineering is making sure the expensive
part of the machine is never waiting.

**The regime.** And looping back to where the dropout discussion started: the deepest
non-architectural change is that models now train for ~one epoch on enormous, curated
corpora rather than many epochs on small ones. That single shift quietly rewrites the
rulebook — it's why regularization disappeared, why data curation became the
highest-leverage activity, and why the bottleneck moved from "can we fit the data" to
"can we afford the tokens."

## 3. The data: what the model actually learns from

There are really two kinds of data in nanochat, doing two completely different jobs,
and I think the distinction is one of the most useful things to take away from the
whole project.

The first kind is the **pretraining data**, and its job is to teach the model to
predict the next token. GPT-2 trained on WebText — pages collected by following
outbound Reddit links with at least 3 karma, which was a clever quality filter for
2019\. nanochat trains on ClimbMix, a heavily filtered and curated web corpus, and the
difference matters more than it sounds: at a fixed compute budget, the quality of each
token is one of the biggest levers that exists. A cleaner corpus means the model
spends its limited capacity learning language and facts instead of memorizing boilerplate
and spam. (nanochat also trains its own tokenizer on this data — a 32,768-token
vocabulary instead of GPT-2's 50,257 — so the vocabulary is fitted to the corpus it
will actually see.)

But a model that's finished pretraining is still just an autocomplete. Ask it a
question and it's as likely to continue with three more questions as it is to answer —
because that's what text on the internet looks like. Getting from there to something
you can talk to is the second kind of data's job.

## 4. The fine-tuning data: from autocomplete to chatbot

Supervised fine-tuning is, to me, the most underrated step in the whole pipeline.
Mechanically it's the same training loop as pretraining — predict the next token — but
the data is no longer random web text. It's conversations, formatted with special
tokens marking who's speaking, and the model is only trained on the assistant's side
of them. The model isn't learning new facts here; it's learning a *role*.

nanochat's SFT mixture has three ingredients, and you can read each one as buying a
specific capability:

- **SmolTalk** (460K conversations) is the general-purpose one — everyday dialogue,
  questions and answers, multi-turn exchanges. This is what teaches the model the
  assistant format itself: that there is a user and an assistant, that the assistant
  answers rather than continues, how to handle the back-and-forth of a conversation.
- **MMLU auxiliary training data** (100K rows, repeated 3 times) teaches multiple
  choice. This one is easy to dismiss as teaching-to-the-test — and it partly is,
  since the evals are multiple choice — but it's also genuinely useful: it trains the
  model to read a constrained question, weigh fixed options, and commit to exactly one
  answer instead of hedging. That's a different skill from open-ended generation, and
  models don't get it for free.
- **GSM8K** (8K problems, repeated 4 times) teaches grade-school math — and, more
  interestingly, *tool use*: the solutions use a calculator, so the model learns to
  emit special tokens that call out to one rather than trusting its own arithmetic.

What strikes me about this list is how legible it is. The mixture is engineered: you
can see exactly which capability each dataset is buying. And that's also what makes it
inviting to tinker with — this is the obvious place to inject your own data. If you
wanted a model that's disproportionately good at, say, SQL or your own domain, you
could have a stronger model generate thousands of question-answer pairs in that domain
and add them to the mixture. Nothing in the pipeline would change; you'd just be
buying a different capability.

(After SFT there's one more optional stage in nanochat — reinforcement learning on the
math problems, where the model generates its own solutions and the correct ones get
reinforced — but SFT is where the chatbot is actually born.)

## 5. Side question: how is this different from distillation?

That idea at the end of the last section — use a stronger model to write your training
data — has a name, and it's worth untangling, because the terminology is genuinely
confusing. When people say a lab "distilled" a frontier model, they can mean two quite
different things.

**Distillation in the original sense** (the Hinton 2015 version) requires owning the
teacher. The student model isn't trained on the teacher's *answers*; it's trained to
match the teacher's full probability distribution at every token. That's a much richer
signal than SFT: instead of learning "the next token is *Paris*," the student learns
"the teacher puts 81% on *Paris*, 7% on *France*, 2% on *the*..." — thousands of
graded judgments per position instead of one hard label. This is how labs make their
own small models from their big ones, and it's only possible with access to the
teacher's internals.

**Distillation in the colloquial sense** — what's usually meant when people say some
lab distilled GPT-4 or Claude through the API — is mechanically just SFT. You don't
have the teacher's probabilities; all you can do is ask it questions, collect its
answers (including the reasoning, which is where much of the value is), filter them
for quality, and fine-tune your model on the result. The training procedure is
identical to nanochat's SFT stage. The only thing "distillation" adds to the name is
where the data came from: a model wrote it instead of humans.

DeepSeek made this unusually concrete with their R1 release: the small "distilled"
models they published are open about being exactly this — about 800K reasoning
examples generated by the big R1 model, then plain supervised fine-tuning on smaller
base models. No reinforcement learning, no access to logits, nothing exotic. The same
recipe as nanochat's chat_sft, with a better author for the data.

So the answer to my own question: SFT is the *mechanism*, distillation describes the
*data source*. The version that's a genuinely different technique — matching the
teacher's distributions — is reserved for whoever owns the teacher. Everyone else is
doing SFT on synthetic data, which is exactly the door that a codebase like nanochat
leaves open for you.

## Closing

If I had to compress everything above into the three things that actually explain why
training a GPT-2-class model went from millions of dollars to a hundred, it would be
these.

The first is the accumulation of small improvements. Almost nothing in this post was a
breakthrough — RMSNorm saves a couple of operations, Muon descends a bit more
efficiently, a percent here, a percent there. What made the difference is a community
(the speedrun crowd that nanochat grew out of) relentlessly testing these ideas and
keeping only what measurably helped, year after year, while the GPUs underneath them
got faster and cheaper at the same time. The cost collapse is mostly compounding, not
genius.

The second is sliding-window attention, the one change I keep coming back to because
it rethinks the original design rather than just trimming it. GPT-2 treated attention
as one uniform operation — every token looks at everything, in every layer. The newer
view is that attention is doing different jobs at different layers, and most of those
jobs are local; so you let most layers look only nearby and keep a few layers with the
full view, and the residual stream carries the long-range information to everyone
else. It's the same architecture, but it's been *understood* in a way the original
wasn't, and the frontier models are built on the same understanding.

And the third is the data. A curated, filtered corpus instead of raw web text, seen
once instead of looped over — and on top of it, a small, deliberately chosen
fine-tuning mixture where you can point at each dataset and say what capability it
buys. Architecture changes get the attention, but at a fixed budget, what you feed the
model is at least as important as what the model is.

So that's the *what* and the *why*. The obvious next step is to actually do it. In the
next post I want to run the training myself: rent the GPUs, figure out what it
actually takes to go from a cloud node to a finished model, and see where the $100
estimate holds up and where it doesn't. And then the part I'm most curious about —
creating my own data. If the SFT mixture is just capabilities you can buy, I want to
try buying one: generate a dataset in a subdomain with a stronger model, add it to the
mixture, and see if the model actually gets better at it.
