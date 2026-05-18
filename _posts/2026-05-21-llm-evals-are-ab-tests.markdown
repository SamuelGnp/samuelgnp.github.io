---
title: "LLM evals are A/B tests, and most teams are doing both badly"
layout: post
date: 2026-05-21 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - llms
  - evals
  - ab-testing
  - statistics
category: blog
author: samuelgnap
description: An LLM eval is a randomized comparison of two systems on a held-out dataset, which is an A/B test by another name, and the experimentation field already worked out how to run one properly.
---

I read an eval report last month that said "GPT-X is better, 87% to 84%". No sample size. No confidence interval. No mention of which prompts the two systems disagreed on. The conclusion was that the team should ship GPT-X.

That is an A/B test. It is just a badly-run one.

I want to be careful about the tone here, because the LLM eval community is not stupid. It is young. Most of the people building evals today come from ML research or product engineering, not from the experimentation side of the house. The pricing and ads world has been running randomized comparisons under noise for twenty years, and we made every one of these mistakes ourselves. The point of this post is to compress some of that hard-won knowledge into something an eval team can use this week.

## An LLM eval is an A/B test wearing a different hat

Strip the vocabulary off an LLM eval and you get this: you have two systems, A and B. You have a dataset of inputs. For each input, you generate an output from A and an output from B. You score the outputs, either with a reference answer or with a judge, and you count wins.

That is a randomized comparison of two treatments on a held-out sample. The fact that the inputs are prompts and the outputs are token sequences does not change the statistical structure. It is the same problem the ads people solved with Bernoulli outcomes and the pricing people solved with revenue per booking. The estimator is a sample mean. The thing you actually want is a treatment effect with an uncertainty band around it.

Once you accept that framing, the toolbox is already on the shelf.

## What the experimentation field figured out years ago

The pricing and experimentation world has spent the last two decades collecting tricks for one specific problem: measuring small differences in noisy data without fooling yourself. None of these are exotic. All of them are missing from typical LLM eval reports.

**Sample size calculations before you start.** If your eval set has 200 prompts and your effect size is 3 percentage points, you do not have the statistical power to call it. You can compute this in five lines. Almost nobody does.

**Variance reduction.** CUPED-style adjustments use pre-experiment data to soak up baseline variance and tighten your effect estimate at no extra cost. For LLM evals, the equivalent is using a per-prompt baseline score (from a third reference model, or a previous version) as a covariate. Same prompts, smaller error bars.

**Blocking and stratification.** If your eval set mixes easy prompts and hard prompts, the easy ones dominate the average. Stratify by difficulty and report effects within strata. A 3-point gap on hard prompts is a different fact than a 3-point gap on easy ones.

**Sequential testing with corrections.** If you peek at the result every day during a week-long eval, your reported p-value is wrong. Group-sequential designs and always-valid inference (mSPRT, Bayesian alternatives) fix this. The LLM equivalent is the team that runs the eval on a 500-prompt subset, sees a 2-point gap, expands to the full 5000, and reports the final number without correcting for the early look.

**Pre-registration.** Decide what counts as a win before you run the eval. Otherwise you are doing exploratory analysis and calling it confirmation.

None of this is a CUPED paper away from being applicable to LLM evals. The translation is mechanical.

## A few things really are different about LLMs

I do not want to oversell the analogy. LLM evals have some genuinely new wrinkles that classic web experimentation does not deal with.

**The judge is part of the measurement.** When a judge model decides which of two answers is better, the judge has its own bias and its own variance. A noisy judge inflates the variance of your win rate. A biased judge (preferring verbose answers, preferring its own model family) shifts the estimate. This is closer to the situation in clinical trials, where the rater is a known source of bias and you blind them, randomize their order, and report inter-rater reliability. Most LLM eval reports do none of that.

**Labels are expensive.** In web A/B testing you can usually run for another week to get more samples. In LLM evals, every additional prompt costs human labelling time or judge-model calls. Power calculations matter more, not less, because you cannot just buy your way out of an underpowered design.

**Pairwise vs absolute.** Judge models are noticeably better at saying "B is better than A" than at scoring each on an absolute scale. That makes the natural design within-subject: same prompt, both systems, pairwise judgement. Within-subject designs have lower variance than between-subject for free. The experimentation literature on crossover trials applies almost directly.

**Output is high-dimensional.** A click is a click. An LLM answer has correctness, style, length, helpfulness, safety. You are running a multivariate comparison whether you wrote it down that way or not. Multiple-comparison corrections apply.

These differences are real. They make LLM evals harder than a web A/B test, not easier. They do not mean the experimentation toolbox is wrong. They mean you have to use more of it, not less.

## The vocabulary maps cleanly onto experimentation terms

Here is the dictionary I use when I read an LLM eval report:

- Win rate is a treatment effect on a binary outcome. Report it with a confidence or credible interval.
- The judge model is a measurement instrument. Report its agreement rate with humans, the same way you would report inter-rater reliability.
- Pairwise comparisons on the same prompt are a within-subject design. Use the within-subject estimator, not the between-subject one.
- The eval set is a sample from a population of prompts. The population you actually care about is the one your product will see. If those two distributions differ, you have an external validity problem, and no amount of statistical machinery fixes it.

Once the vocabulary lines up, the existing toolbox lines up too.

## What to stop doing this week

If you run LLM evals, three changes will move you ahead of most teams in your field:

1. Stop reporting bare percentages. Every win rate gets a confidence or credible interval next to it. If the interval crosses zero or crosses your minimum effect of interest, say so out loud.
2. Pre-register the eval set and the success criterion before you look at outputs. If you discover the eval set is wrong, fix it and re-run, but do not silently change the bar after seeing the result.
3. Run a power calculation. If your eval cannot detect a 3-point gap with the data you have, do not claim a 3-point gap is real when you see it.

None of this requires new infrastructure. It requires deciding that "GPT-X 87, GPT-Y 84" is not a result.

## A bet

The LLM community is going to land on this within eighteen months. The teams that run the careful version of the eval today will look prescient. The teams that keep shipping bare-percentage reports will keep changing decisions on noise and wondering why their post-launch metrics never match the eval.

I do not think this is a particularly bold prediction. Every other field that ran into the "are two systems different" problem ended up at the same place. Clinical trials got there in the sixties. Web experimentation got there in the 2010s. Pricing got there over the same window. The LLM field is, on this specific axis, about a decade behind. That is a feature of being a young field with lots of new problems, not a failure of intelligence. It just means the ladder is sitting there, already built, and you can climb it faster than the previous fields did.

The first rung is admitting the thing you are running is an A/B test, and treating it like one.
