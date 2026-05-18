---
title: "Bayesian A/B testing is mostly a communication tool"
layout: post
date: 2026-06-15 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - ab-testing
  - bayesian
  - statistics
category: blog
author: samuelgnap
description: The mathematical advantage of Bayesian over frequentist A/B testing is overstated; the real win is that posteriors give non-technical stakeholders a quantity they actually act on.
---

A few years ago I sat in a readout with a commercial stakeholder, a head of pricing who had to decide whether to roll out a fare change across a family of routes. The analyst presented the result with the standard frequentist apparatus. Treatment beat control. P less than 0.05. Confidence interval did not cross zero. The stakeholder listened, nodded, and asked the question I have now heard in some form at every company I have worked at: "but is it good?"

I watched the analyst try to translate. He explained what the p-value meant. He explained what the confidence interval was. He explained what "significant" did not mean. The stakeholder kept nodding, which is what stakeholders do when they have stopped following. Twenty minutes later he asked the question again, slightly differently. The meeting ended without a decision.

The next test, the same team reported it as a posterior. "There is a 73% probability the new price beats the old one, and the expected uplift is between zero and four percent." The same stakeholder said "fine, let us ship it", and the meeting was over in ten minutes.

That moment is what convinced me that the Bayesian-versus-frequentist debate is, for almost every working data scientist, the wrong debate to be having.

## The technical comparison, briefly and honestly

There is a real technical conversation underneath all of this. Bayesian A/B tests let you bring in prior information, peek at the data without inflating false-positive rates in the same way, and stop early when a decision is clear. Frequentist tests are immune to prior misspecification by construction, because there are no priors. Sample size calculations on the frequentist side are simpler and more standardised. Multiple comparisons are handled with mature, well-understood corrections. Sequential testing exists on both sides now, with mSPRT and group sequential designs on the frequentist side, and posterior-based stopping on the Bayesian side.

Neither framework is mathematically dominant. A reasonable practitioner can run either and arrive at decisions of comparable quality. People who tell you otherwise are usually selling a tool, a course, or a personal brand.

## Where frequentist tests fail commercially

The failure mode of frequentist testing in industry is not statistical. It is linguistic. The p-value answers a question the stakeholder is not asking. "What is the probability of observing data this extreme or more, under the null hypothesis of no effect" is not the question. The question is "should I do this?" And the path from the first to the second is paved with caveats most decision-makers do not retain past the next coffee.

The other recurring failure is that confidence intervals get confused with credible intervals on a daily basis, including by data scientists who should know better. I have read internal reports that called a 95% confidence interval "the range in which the true effect lies with 95% probability". That is wrong, by the strict frequentist definition. It is also, almost word for word, the right description of a 95% credible interval. The frequentist quantity is harder to explain because the thing people want to say about it is not actually true of it.

## The real Bayesian win

P(treatment beats control) is a quantity humans already know how to use. We make decisions under probability all day. We do not make decisions under "the long-run frequency of rejecting a true null at the alpha level". A 73% posterior probability of a positive effect, paired with an expected value and a credible interval, is something a head of pricing can act on without a statistics tutor in the room.

That is not a small win. That is the entire game. Most of the value an experimentation team produces is not the test itself. It is the speed and quality of decisions that come out of the test. A framework that makes the right decision faster, with fewer follow-up meetings, is a better framework regardless of how much sharper its theoretical properties are.

The Bayesian framing also lets you communicate a thing the frequentist framing actively obscures: how much you do not know. A wide posterior is honest. A confidence interval that does not cross zero is presented as a binary, which it never really is.

## The honest counterpoint

I do not want to oversell this. There are real costs.

Priors are a vector for organisational politics. "Why did you choose that prior" is a question that can be asked in good faith, and it can also be asked as a stalling tactic by a stakeholder who does not want the result the test produced. I have watched both versions of this conversation happen. The defence is good documentation and weakly informative priors that survive sensitivity checks, but it is real work.

The infrastructure burden is also real. Frequentist sample size calculations are a closed-form line in a textbook. Bayesian designs often require simulation. Sequential Bayesian stopping rules need to be agreed in advance with the same discipline as frequentist alpha spending, otherwise you get garbage. None of this is hard, but it is work, and most teams underestimate it.

There are also legitimate cases where the frequentist apparatus is the right tool. Regulated environments where the false-positive rate must be controlled at a known level. Settings where the prior literature is genuinely empty and any prior is informative in a way you cannot defend. Multi-arm tests where the frequentist machinery happens to be more developed in your stack.

## Practical advice from someone who has run both in production

If your team is set up for frequentist tests, keep running them. Switching frameworks for the sake of the framework is a year of organisational effort that yields nothing your stakeholders will notice.

Whatever framework you use under the hood, report posteriors. Even if your test is frequentist, you can compute and report P(treatment beats control) with a sensible prior at the end. The communication advantage does not require you to rebuild the platform. It requires you to translate the output.

Stop reporting bare p-values to non-technical audiences. Translate them. Pair every test result with a quantity the reader can act on, which usually means a probability, an expected effect, and an honest interval.

## The closing read

The Bayesian-versus-frequentist debate, as it plays out on the internet, is mostly aesthetics. People line up by tribe and argue about coverage properties and prior elicitation as if those were the things their stakeholders cared about. The decision-makers reading your report do not care which framework you used. They care whether they can act on the answer.

If you take only one thing from this post, take this. Run whichever test your team can run well. Communicate the result as a probability of a decision being correct. Watch how much faster the meetings end.

P-values do not answer the question stakeholders are asking. The Bayesian-frequentist debate is aesthetics. The decision is the only thing that matters.
