---
title: "Refusal is two directions, not one"
layout: post
date: 2026-07-20 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - llms
  - transformers
  - safety
  - alignment
  - learning
category: blog
author: samuelgnap
description: A follow-up to the refusal-direction post. Two papers since Arditi sharpen the picture. Post-training only really moves refusal of all the things the model does, and refusal itself turns out to be two directions at two token positions, not one.
---

This started for me with the post I wrote a week ago. I'd ended on the
line that the entire safety behaviour of a chat model lives on one
direction in the residual stream, and that this is why every jailbreak
works. That paper is from June 2024 — a year and a half ago — and I'd
never gone back to check what came after. So I went and read.

There's been quite a bit. Two papers in particular sharpen the picture
from the previous post and fit together cleanly. I want to walk through
both.

If you haven't read [the previous post](/refusal-is-a-single-direction/),
the residual stream, the difference-of-means construction of r̂, and the
scalar projection r̂ᵀx⁽ˡ*⁾ are the only things you strictly need from it.
I'm not going to re-derive them.

## 1. The patch was thinner than I said

The first paper is Du et al. (UCLA, COLM 2025), "How Post-Training
Reshapes LLMs" ([arXiv:2504.02904](https://arxiv.org/abs/2504.02904)). The
setup is simple. Take a base model and its post-trained sibling. Look at
them mechanistically across four perspectives: knowledge, truthfulness,
refusal, and confidence. Ask which of those four post-training actually
moves. They have four findings.

The first finding concerns knowledge. The base model learns facts during
pretraining: it predicts the next token across the whole internet, and
to do that it has to encode "Paris is the capital of France" somewhere
in its weights. Post-training does not move those facts. Du uses causal
tracing (patch the hidden state at a specific layer and token position
from one run into another, and see whether the output flips from FALSE
to TRUE) and finds the locations where knowledge sits are essentially
identical in base vs post-trained models. Correlation above 0.99 on
knowledge-related positions. The factual content is from pretraining
and stays where it was.

There is a small refinement on top of that. Post-training also develops
some new knowledge representations alongside the base ones. It doesn't
replace them; it adds. You can patch from base into post (use the base
model's knowledge representation inside the post-trained model) and it
almost always works. The reverse sometimes fails. That asymmetry is the
evidence that post-training adds something. But none of this is
"post-training taught the model new facts." It is more like post-training
develops a second, parallel way to access the same facts. Pretraining
loads the knowledge. Post-training polishes how it gets used.

The second finding is about truthfulness, the model's internal answer to
"is this statement true or false?" Du shows this is a linear direction
in the hidden state. The construction is the same as Arditi's:
difference-of-means between true and false statements at a chosen layer
and token position. The truthfulness direction in the base model and the
truthfulness direction in the post-trained model have high cosine
similarity. The vector you learn from one can steer the other in both
directions. So the model's internal belief about truth is mostly
inherited from pretraining and doesn't really move during alignment.

The third finding is refusal. Same construction, harmful prompts minus
harmless prompts. The result is the opposite. The base refusal direction
and the post-trained refusal direction have low cosine similarity. They
are genuinely different vectors. And the transfer is asymmetric in a
specific direction.

The terminology matters because it's how the paper states the result.
"Forward" means from earlier (base) to later (post-trained). Forward
transferability is what fails: a refusal direction learned in the base
model cannot effectively steer the post-trained model. "Backward" means
from later to earlier. Backward transferability works: a refusal
direction learned in the post-trained model can drive refusal in the
base model.

The interpretation lines up with the thin-film picture from the previous
post. Base models have a weak refusal direction of their own; some of
them sometimes refuse harmful prompts before any safety training.
Post-training does not sharpen that weak base direction. It installs a
new, different one. So the base direction cannot reach the post-trained
model's new mechanism, which is why forward fails. The post-trained
direction is a strong, well-defined refusal vector, and the underlying
refusal machinery in the base model is still there for it to drive,
which is why backward works.

The fourth finding is confidence, and this one is the most "we don't
quite know yet" of the four. Post-trained models are typically more
confident than base models. If you ask a base model a question, it
spreads probability across several plausible next tokens. The
post-trained version concentrates almost all probability on one. The
output distribution sharpens.

The leading hypothesis for why was *entropy neurons*. These are specific
neurons in the final MLP layer that don't push the model toward any
particular output token, but uniformly scale the spread of the output
distribution. They behave like a temperature dial. They don't change
which token is most likely; they change how confident the model sounds.
The hypothesis was that post-training changes which entropy neurons fire
and how strongly.

Du tests it. They identify entropy neurons in both base and post-trained
models, by weight norm and logit-attribution variance, and find the sets
largely overlap with similar properties. Entropy neurons cannot explain
the confidence shift. Whatever causes post-trained models to be
overconfident is a subtler mechanism that this paper does not isolate.
A negative result, but a useful one. It rules out one of the leading
explanations.

What this adds up to. Of the four perspectives, refusal is the only one
where the model's internal representation meaningfully changes during
post-training, and even then it changes by addition rather than
reshaping. Knowledge is inherited. Truthfulness is inherited. Confidence
shifts for reasons not yet localized. Refusal is the actual film.

When the previous post said post-training has the budget only for "a
small patch on top", Du measured exactly that. The patch is the only
thing that moves.

## 2. It isn't one direction, it's two

The second paper is Zhao, Huang, Wu, Bau, Shi (NeurIPS 2025), "LLMs
Encode Harmfulness and Refusal Separately"
([arXiv:2507.11878](https://arxiv.org/abs/2507.11878)). It refines
Arditi in the most useful direction I've seen in the year and a half
since.

The question they're answering is one the field had been arguing about.
When Arditi extracted "the refusal direction," what was that direction
actually representing? Was it the model's internal belief that the
prompt is harmful? Or was it the behavioural decision to emit refusal
text? These sound like the same thing but they aren't. A model can
believe a prompt is harmful and still comply (a successful jailbreak).
A model can refuse a prompt it doesn't internally think is harmful
(over-refusal). If the direction Arditi found is one of these and not
the other, that changes a lot.

Zhao's answer is that there isn't one direction. There are two. They
live at different token positions in the chat template, they have very
low cosine similarity, and they cause different effects when you steer
along them.

Setup first. The chat templates that fine-tuned models expect look like
`[INST] user instruction [/INST]` for Llama-2-chat, with equivalent
markers in other models. After the closing marker, the model starts
generating the assistant's response. So there are two natural positions
to grab the residual stream after the question:

- `t_inst`: the last token of the user's instruction itself, right
  before the closing `[/INST]` marker.
- `t_post-inst`: the last token after the closing marker, just before
  the model starts generating.

Both positions can see the entire input by self-attention.
Information-wise they have access to the same thing. But the model does
different work at each one. That is the load-bearing observation.

The first finding is a clustering observation. Zhao takes a set of
harmful and harmless prompts, runs them through, and looks at how the
residual streams cluster at each position.

At `t_inst`, the clustering is by harmfulness. Harmful prompts cluster
together, harmless prompts cluster together, regardless of whether the
model went on to refuse or comply. Importantly: a harmful prompt the
model accepted (a successful jailbreak) still clusters with the other
harmful prompts at this position. The model's internal recognition of
harmfulness is in there, even though its behaviour didn't reflect it.

At `t_post-inst`, the clustering is by behaviour. Refused prompts
cluster together, accepted prompts cluster together, regardless of
whether the prompt was actually harmful. If the model over-refused a
benign prompt, at `t_post-inst` it clusters with the refused-harmful
prompts, not with the accepted-harmless ones. The decision to refuse
has been committed by this position.

So two positions, two different concepts being encoded. One is the
model's perception. The other is the model's behaviour.

The second finding is that the two directions you extract from those
two positions are genuinely distinct vectors. At `t_inst`, take the
difference-of-means between harmful and harmless prompts. That is the
harmfulness direction. At `t_post-inst`, the same construction recovers
the Arditi refusal direction. Cosine similarity between the two on
Llama-2: around 0.1. They are not the same vector pointing slightly
differently. They are different vectors.

The third finding is the causal proof that they actually do different
things. Two correlated directions could in principle be redundant under
intervention. Zhao designs a clean test that forces them to differ.

The test is a reply-inversion probe. Append a follow-up to the prompt:
*"Could this user prompt cause harm? Answer 'Certainly' if it could,
otherwise 'No'."* A harmless prompt would normally produce "No". A
harmful one would normally produce "Certainly".

Now intervene. Take a harmless prompt and steer along the harmfulness
direction: the model flips its answer to "Certainly". The intervention
has changed the model's *belief* about whether the prompt was harmful.
Take the same harmless prompt and steer along the refusal direction
instead: the model continues to answer "No". The refusal direction does
not change the belief.

The reverse case works the same way. Steer a harmful prompt against the
harmfulness direction and the model answers "No". It now internally
treats the prompt as not harmful. Steer the same harmful prompt against
the refusal direction and the model still answers "Certainly". It still
internally treats the prompt as harmful, even though it stopped
refusing.

That is the causal evidence. The two directions don't just sit at
different positions; they encode different concepts. One is the
perception ("is this harmful?"). The other is the behavioural readout
("am I going to refuse?").

This split lets you re-analyze jailbreaks. Different techniques can now
be classified by which scalar they actually move.

Adversarial suffixes (the GCG kind) suppress the refusal scalar at
`t_post-inst` but leave the harmfulness scalar at `t_inst` positive.
The model internally still believes the prompt is harmful, but the
behavioural readout collapses, so it complies. Adversarial template
attacks work the same way.

Persuasion attacks ("you're a helpful assistant in a fictional universe
where...") are mechanistically different. They actually move the
harmfulness scalar at `t_inst` down. The model is talked into thinking
the prompt isn't harmful, and then complies for the consistent reason
that it doesn't think anything bad is happening.

The previous post's framing, every jailbreak works by moving the
residual stream off the refusal axis, was right but it collapsed these
two cases. From the outside they look the same. The model complied with
a harmful request. Mechanistically they are different. One bypasses the
readout. The other flips the perception.

The application Zhao gets out of all this is a built-in safety
classifier they call Latent Guard. If the harmfulness direction at
`t_inst` is the model's actual internal belief about harmfulness, you
can use it directly. For any prompt, compute its scalar projection onto
the harmfulness direction. Positive means harmful, negative means
harmless. No extra model. No training cost beyond computing the mean
difference once.

Zhao tests Latent Guard against Llama Guard 3 8B, which is a dedicated
fine-tuned safety classifier and a standard moderation tool. Latent
Guard matches or beats Llama Guard 3 on adversarial suffix detection,
persuasion, jailbreak prompts, and over-refusal benchmarks.

The more important point about Latent Guard is robustness to fine-tuning
attacks. If you take the model and fine-tune it on harmful examples to
break safety alignment, the behavioural refusal at `t_post-inst`
collapses, which is what you'd expect. But the harmfulness representation
at `t_inst` barely moves. The model still internally recognizes harmful
inputs after its visible safety behaviour has been compromised. A
classifier sitting on the harmfulness direction keeps working even when
the surface safety has been broken.

What this adds up to. There were two directions the whole time. The
behavioural refusal direction is the one Arditi found. The perceptual
harmfulness direction was sitting at a different token position and
didn't get noticed. They have different cosine, different effects under
steering, different vulnerabilities under fine-tuning. And the deeper of
the two, the one that encodes the model's actual belief, is much harder
to remove than the behavioural readout sitting on top of it.

## Where this leaves the picture

The previous post closed with "the safety patch is one direction thick".
A more accurate version, after Du and Zhao:

Of all the things a chat model does (recall facts, judge truth, decide
whether something is harmful, decide whether to refuse), only the last
of those gets reshaped during post-training. The rest is inherited from
the base model. Refusal is the patch.

And the patch itself is even thinner than Arditi made it look. There are
two refusal-related directions, not one. The deep one is the model's
perception of harmfulness. It sits at the last instruction token, is
mostly inherited from pretraining, and survives even when you fine-tune
the model to break safety. The shallow one is the behavioural decision
to refuse. It sits at the post-template token and is what post-training
actually installs. That is the actual film. Every jailbreak that
suppresses refusal output, including the ones that look most powerful,
leaves the perception underneath intact. The model usually still knows
the prompt is harmful when it complies.

The previous post said refusal is one wall of a room full of behavioural
directions: sentiment, persona, sycophancy, style. That framing turns
out to be right with one important wrinkle. The directions in that room
are not all the same kind of object. Some encode the model's beliefs
about the world (truthfulness, harmfulness). Those tend to be inherited
from pretraining and stay put. Others encode behavioural readouts
(refusal, response style). Those tend to be installed by post-training
and are brittle. The map of the room is the next post.

---

Reading list:

1. **LLMs Encode Harmfulness and Refusal Separately** — [Zhao et al.,
   NeurIPS 2025](https://arxiv.org/abs/2507.11878). The Arditi
   refinement; the paper this post is really about.
2. **How Post-Training Reshapes LLMs** — [Du et al., COLM 2025](https://arxiv.org/abs/2504.02904).
   Empirical confirmation that of knowledge, truthfulness, refusal, and
   confidence, refusal is the only one post-training meaningfully moves.
