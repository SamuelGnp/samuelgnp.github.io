---
title: "How GPT-2 actually works, piece by piece"
layout: post
date: 2026-07-06 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - llms
  - transformers
  - gpt-2
  - learning
category: blog
author: samuelgnap
description: A walkthrough of the GPT-2 architecture the way it finally clicked for me — embeddings as geometry, attention as communication, the feedforward net as computation, and the glue that makes it trainable.
---

I was curious how GPT actually works — not at the level of "it predicts the next
token," but what the architecture really does, and what makes it so brilliant. So I read
the original paper, and worked through a book alongside it to make things stick. This
post is how I understood the most important parts. It isn't exhaustive — there are more
pieces in the model than I cover here — but these are the ones that matter for
understanding why the thing works at all.

Why bother with a 2019 model? We use LLMs every day now, and I think it's a shame to
use something constantly without understanding what it actually does. Today's models are
far more advanced, and there's genuinely new research out there — text diffusion, for
example — but the core of the architecture has barely moved, and attention is as
relevant as ever. When DeepSeek made headlines with a reasoning model whose base model
was reportedly trained for only around $5–6M, they had optimized an enormous amount —
yet fundamentally it was still this same architecture. You can go read about their
multi-head latent attention, and it's interesting — but it only makes sense if you
understand plain multi-head attention first. The fundamentals are still the fundamentals.

## 1. Embeddings: turning tokens into something math can work with

The first problem a language model has to solve is embarrassingly basic: machines don't
understand words. They don't understand characters either, and they don't even understand
token IDs — to a neural network, token number 4076 carries no more meaning than token
number 9. What machines understand is mathematics. So before anything interesting can
happen, language has to be translated into a mathematical object that can *carry* meaning.

That object is a vector.

Text first gets split into tokens — for intuition you can think of them as words, almost
interchangeably (in practice they're often pieces of words, but that detail doesn't change
the picture). Each token is then mapped to a vector: in GPT-2, a list of 768 numbers.

The point is not the numbers themselves — no single number means anything. The point is
the **geometry**. Picture three dimensions, with arrows pointing in different directions.
Two arrows pointing roughly the same way represent things that are related; arrows
pointing apart represent things that aren't. Now the model does exactly this, except in
768 dimensions instead of 3. You can't visualize it, but the idea is identical: meaning
lives in the directions and distances between vectors. The geometry *is* the machine's
understanding of the token.

One thing worth getting right, because it's a common misconception: these vectors are not
produced by some separate pre-trained algorithm and imported into the model. The embedding
table starts as pure random noise and is learned **together with everything else**, shaped
by nothing but the pressure to predict the next token. The meaningful geometry isn't
given — it *emerges*.

There's a second piece of information the vector alone doesn't capture: *where* the token
sits in the sentence. Partly that's because the same word can mean different things in
different positions. But the deeper reason comes from how attention works (next section):
attention is order-blind. Without position information, the model sees a bag of tokens —
"dog bites man" and "man bites dog" would be literally indistinguishable, not just
ambiguous. So GPT-2 adds a second learned vector, one per position, on top of each token
embedding. That's the only place where the model is told about order.

## 2. The transformer block, bird's eye view

At this point we have vectors that say *what* each token means and *where* it sits. The
rest of GPT-2 is one component, repeated: the transformer block. GPT-2 stacks 12 of them,
all structurally identical, so we only need to understand one.

Inside a block, two things happen. First, **attention**: the tokens talk to each other.
Then, a **feedforward network**: each token processes what it just heard, alone. That's
the whole division of labor — attention is communication, the feedforward net is
computation.

Why stack the same block 12 times? Because one round of communication isn't enough. Each
pass through a block lets the model refine its representations a little further — early
blocks resolve simple, local things; later blocks can build on those results to capture
more abstract relationships. Depth is refinement.

## 3. Zoom in: multi-head attention

So far each token knows its own meaning and its own position — but language doesn't live
in isolated words. We need to know how the tokens **relate to each other**: what context
each one is actually sitting in. "Bank" next to "river" and "bank" next to "loan" should
not stay the same vector for long. Attention is the mechanism that lets each token update
its meaning based on the tokens around it.

Here's how the conversation works. From its embedding, every token produces three new
vectors: a **query** — what am I looking for; a **key** — what do I contain; and a
**value** — what you get from me if you decide I'm relevant. To figure out how much
token A should care about token B, we take the dot product of A's query with B's key:
a high score means "B contains what A is looking for." The scores across all tokens get
turned into weights (a softmax, so they sum to 1), and token A's updated representation
is the weighted average of everyone's *values*. Communication, in one mechanism: each
token asks a question, every other token advertises what it has, and the answers get
blended in proportion to how well they match.

The part that made attention click for me is this: none of those questions and answers
are designed by anyone. Each token's query, key, and value are produced by multiplying
its vector with three weight matrices — and those matrices start essentially *empty*,
just random noise. The architecture only fixes the pattern of interaction: dot products
between queries and keys, weighted sums of values. What a query should ask, what a key
should advertise, what a value should hand over — the model figures all of that out by
itself during training. Nobody told it what "relevance" means; the design just made it
learnable. That's the brilliance of the mechanism.

One constraint sits on top: the **causal mask**. GPT-2 generates text word by word, left
to right — so when a token computes its attention, it is only allowed to look at the
tokens *before* it. All future positions are hidden. (The subtlety: during training the
model predicts every position in the sequence simultaneously, in one pass — the mask is
what stops position 5 from cheating by peeking at position 6.)

**Why multiple heads.** There is no single correct way to look at a sentence. You can read
it for its grammar, for its syntactic structure, for who-refers-to-whom, for topic and
context — these are different *views* of the same text, and they're all useful at once.
That's what the heads are: GPT-2 runs 12 attention operations in parallel in every block,
each with its own learned way of deciding which tokens matter to which. One head might
track grammatical agreement, another might track what a pronoun points back to. The exact
number 12 is an engineering choice, not a magic constant — what matters is the idea of
multiple simultaneous views.

(Don't confuse the two twelves: 12 *heads* run in parallel inside each block — multiple
views at the same time — while 12 *blocks* run in sequence — refinement over multiple
rounds. Width vs. depth.)

## 4. Zoom in: the feedforward network

After attention, each token has gathered information from its context. The feedforward
network is where it actually does something with it.

The structure is almost disappointingly simple: two linear layers with a nonlinearity in
between. Each token's 768-dimensional vector gets expanded to 3072 dimensions, passed
through a GELU (a smooth cousin of "set negatives to zero"), and projected back down to
768\. Crucially, this happens to **every token independently** — no communication here.
Attention was the talking; this is each token thinking alone about what it just heard.

Why is this layer even needed? Look back at what attention does: it computes weighted
*averages*. It's brilliant at moving information between tokens, but averaging is a
fundamentally limited operation — if you stacked attention layers alone, you'd mostly be
blending and re-blending the same vectors. The feedforward network's nonlinearity is what
gives the model real computational power: the ability to *transform* a representation,
not just mix it with others.

And it's not a side dish — roughly two-thirds of GPT-2's parameters live in these layers.
A useful (if simplified) picture from interpretability research: the feedforward layers
are the model's memory. Attention fetches the right context into a token's vector; the
feedforward network matches that vector against the patterns and facts it has stored, and
writes the relevant ones back out. Attention routes, the FFN knows things.

## 5. The glue: skip connections, layernorm, dropout

Three more pieces hold the block together. None of them add intelligence — they exist so
that a 12-layer stack can *train* at all.

**Skip connections.** Around both the attention and the feedforward net, the block adds
its own input back to its output: `x = x + f(x)`. The reason is the vanishing gradient
problem: during training, gradients have to flow backwards through every layer, shrinking
a little at each step — stack enough layers and nothing useful reaches the early ones.
The skip connection is a highway: gradients can flow straight through the additions,
untouched, all the way down. A second way to see it: each block no longer has to
re-encode everything from scratch — it only learns a *correction* to the running
representation, which is a much easier job.

**Layer normalization.** Before the attention and before the feedforward net, each
token's vector is rescaled to a standard size. As values pass through 12 blocks of
matrix multiplications, their scales would otherwise drift — growing or shrinking until
training becomes unstable. LayerNorm resets the scale at every step, keeping the
numbers in a healthy range throughout the network.

**Dropout.** During training, randomly zero out a fraction of activations, forcing the
network not to rely too heavily on any single pathway. It's pure regularization: GPT-2
went over its training data multiple times, so memorizing the dataset instead of learning
the language was a real risk, and dropout was the standard defense.

## 6. The output end

After the final block, there's one last layer normalization — and then we have to leave
the embedding space. Everything so far happened in 768-dimensional vectors, but the
answer we need is a word. The output layer is a single matrix that projects each token's
final vector onto all 50,257 vocabulary entries, producing one score per possible next
token — the **logits**. A softmax turns those scores into a probability distribution,
and that distribution *is* the model's prediction: it's the same translation problem as
the embedding at the input, run in reverse — from representation back to vocabulary.
Fittingly, GPT-2 reuses the embedding table itself as this output matrix (known as
*weight tying*): one shared map between words and vectors, used in both directions.

## Closing

What stays with me after all of this is how simple the architecture is —
retrospectively simple, the way elegant ideas always are. Vectors whose geometry
carries meaning. A mechanism that lets tokens talk to each other. A network that lets
each one think. Normalize, add, repeat twelve times. That this is enough — that
mathematical representations can capture language this well, and that the same recipe
scales into the systems we now use every day — is genuinely incredible.

It also makes you wonder what these models are really learning from. Today they train
on video, on synthetic data generated by other models, on reinforcement learning where
a model figures out answers on its own. That works beautifully when answers can be
checked — math, code. It gets harder where they can't: you can't verify a joke, only
have people rank them. But step back far enough and there's a common thread: language
is an expression of thinking that happened to get surfaced. The text was never the
point — it's the trace of a reasoning process. So the thing you'd ultimately want to
train on isn't the words; it's the thinking itself. I honestly hope we never fully get
there, because a closed model trained on how people think is a much scarier thing than
one trained on what they wrote.

One more thought, as a bridge to the next post. Training GPT-2 in 2019 reportedly cost
on the order of $30–40k of compute. Knowing what we know now — about the architecture,
the optimizers, the data — how cheaply could you train it today? And could you push it
further: generate your own training data with current models, tilt it to learn more in
some areas than others? That's the experiment I want to look at next.
