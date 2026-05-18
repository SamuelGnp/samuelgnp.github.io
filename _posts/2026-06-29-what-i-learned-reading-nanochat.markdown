---
title: "What I learned reading Karpathy's nanochat as a pricing scientist"
layout: post
date: 2026-06-29 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - llms
  - learning
  - nanochat
  - transformers
category: blog
author: samuelgnap
description: Coming to LLM internals from an econometrics background, certain things are obvious and others are surprising, and the boundary between the two is the most useful map I have drawn this year.
---

The moment it clicked for me was halfway through the attention block. I had been reading Karpathy's nanochat for a few evenings, treating it as a slow vocabulary exercise, and then I noticed that the dot product of queries and keys followed by a softmax is a kernel. It is a similarity score, weighted, normalised, and used to take a convex combination of values. I have spent years writing Gaussian processes and kernel ridge regressions. The thing that the deep learning literature calls "the attention mechanism" is a kernel smoother with learned features. A few pages later I hit the byte-pair encoder and realised it is a greedy compression algorithm with a frequency table, which is something I would have called Huffman-adjacent if I had seen it in a different course.

That is the post. Two of the load-bearing ideas in a modern LLM are things a stats person already knows, dressed in different vocabulary. The rest of what I learned reading nanochat is a map of which intuitions transfer from econometrics and which do not.

I came to nanochat with the wrong prior. My background is Bayesian inference, hierarchical models, demand estimation, the usual airline pricing stack. I assumed transformer internals would be a wall of unfamiliar math. They are not. The math is mostly things I have seen before. What is unfamiliar is the scale, the engineering, and the data discipline, and that is a different problem from the one I went in expecting to solve.

## What transferred from the stats side

Three things were obvious on first read, and would be obvious to anyone who has spent time with MCMC and regularised regression.

Loss curves look like trace plots. The shape of a training loss curve, the way it descends, plateaus, and occasionally jumps, is the same shape I have stared at on a thousand PyMC runs. Convergence intuitions transfer directly. A loss that is still descending at the end of training is the same warning sign as a chain that has not mixed. A loss that drops sharply and then flatlines is the same pattern as an MCMC chain finding a mode. The diagnostic instincts I built watching NUTS samplers map cleanly onto reading training curves.

Regularisation is the same machinery with different labels. Weight decay is ridge regression. Dropout is bagging at the activation level. Early stopping is a posterior approximation in disguise. Label smoothing is a Dirichlet prior on the target distribution. Every regularisation trick in the deep learning toolkit has a Bayesian or frequentist twin in the statistics literature. The vocabulary is different. The ideas are the same.

Learning rate schedules are step size adaptation. Anyone who has tuned a Metropolis sampler knows that step size determines whether you explore the space or get stuck. Cosine decay and warmup are the same problem as a tuned proposal distribution. The deep learning community has converged on a small set of schedules empirically. The Bayesian computation community got to the same place from the other direction.

If you read nanochat as a stats person, you keep noticing this. The architecture is unfamiliar. The numerical optimisation problem it solves is not.

## What did not transfer

Three things genuinely surprised me, and I think they are where the field actually lives.

The first is how few tricks deep learning relies on. A transformer is a small number of operations repeated many times. Embedding, attention, feed-forward, layer norm, residual connection, repeat. That is most of it. I expected something baroque, a tower of clever ideas accreted over a decade. What I found was the opposite: a clean architecture that has barely changed since 2017, scaled up. The art is not in the model. The art is in the data and the compute.

The second is how much performance depends on data quality versus architecture. You can change the architecture and shift performance by a couple of percent. You can clean the data and shift it by an order of magnitude. The papers that get attention are usually architecture papers. The work that actually moves the needle is the data curation that nobody publishes because it does not look like science. I think this is the single most underrated point in the public conversation about LLMs. The model is the part that is legible. The data is the part that matters.

The third is how much of the work is infrastructure. The tokenizer is a separate project, written in Rust in nanochat's case, with its own build system and benchmarks. The eval harness is its own codebase. The inference server is a FastAPI app with its own deployment story. The training loop has a custom data loader, a distributed setup, and a checkpointing system. The model itself, the PyTorch file that defines the architecture, is maybe ten percent of the total surface area. The other ninety percent is engineering. If you came in expecting LLMs to be a research problem, the ratio is humbling.

## What pricing people get wrong about LLMs

We assume the math is the hard part. We assume that because, in pricing, the math is the hard part. We spend our careers thinking about identification, endogeneity, hierarchical pooling, posterior inference. The technical depth is what separates a good pricing scientist from a mediocre one. So when a pricing person looks at LLMs, they expect to find a similar mountain of mathematical depth at the centre.

It is not there. The mountain is somewhere else. The mountain is data curation, eval design, and infrastructure. A pricing scientist who wants to be useful in an LLM context needs to update on this. The skills that make you good at pricing are not useless, but they are not the differentiator. The differentiator is being willing to spend three weeks cleaning a corpus and another two weeks writing a benchmark, when neither of those activities looks like real work to someone trained on regression.

## What LLM people get wrong about pricing

The mirror image holds. LLM people look at pricing and see supervised learning. There is a feature matrix, a target, a regression problem. So they reach for the supervised learning toolkit, fit a model, and report cross-validated metrics.

That framing misses what pricing actually is. Pricing is closer to reinforcement learning than to supervised learning. The data is endogenous: your past pricing decisions shape the distribution of bookings you observe, which shapes the model you fit, which shapes the next pricing decision. The eval is online, not offline: a cross-validation score on historical data tells you almost nothing about what happens when you deploy the new policy. The reward signal is delayed and noisy, and competitors are reacting in the same loop.

Pricing is closer to RL than to supervised learning. We just call it regression and pretend.

## The skill underneath both worlds

Reading nanochat made me more confident, not less, that the fundamental skill is the same across both worlds. It is not about which loss function you minimise or which architecture you use. It is about instrumenting reality well enough that you can tell whether your system is doing what you think it is doing. In LLMs, that means an eval harness that actually catches regressions. In pricing, that means an experimentation system that survives interference and seasonality. The shape of the engineering is different. The shape of the discipline is the same.

If I were starting over and wanted to be a useful person in both worlds, I would not spend more time on math. I would spend more time on building the instruments that tell me whether my models are wrong. Almost everything else is a special case of that.
