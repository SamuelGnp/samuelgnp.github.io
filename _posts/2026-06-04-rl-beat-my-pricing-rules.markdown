---
title: "Reinforcement learning beat my pricing rules, but not for the reasons I expected"
layout: post
date: 2026-06-04 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - reinforcement-learning
  - pricing
  - ab-testing
category: blog
author: samuelgnap
description: The RL win in our pricing test came from systematic exploration, not from clever sequential policies, and a well-tuned contextual bandit would have captured most of it.
---

The first time the team sat down to inspect the winning policy, we were ready to be impressed. We had spent months on a Q-learning agent for perishable inventory pricing. We had A/B tested it against a mature rules-based engine that had years of business logic baked into it, on a meaningful slice of traffic, over a meaningful window. The RL system beat the rules system by a meaningful margin. We opened the policy, expected to see something subtle, and instead found something that looked stupid.

It was sampling the same neglected segments over and over. Not in some clever exploration schedule. Just visiting parts of the state space that the rules system had decided, years ago, were not worth visiting.

## The expected story was about sequential cleverness

Going in, I had a story I wanted to be true. Deep RL would discover non-obvious price trajectories. It would learn that the price set today affects the demand observed tomorrow in some way no human had written down. The output would be a time-varying policy that no analyst would have produced by hand, the kind of thing you put in a deck with a smug arrow that says "human intuition stops here".

That is the story the literature trains you to want. Atari, AlphaGo, autonomous driving demos. RL wins because it sees sequential structure humans cannot see. If you have spent any time around pricing teams, you have heard versions of this pitch. Booking curves are sequential. Customers arrive over time. The state evolves. RL is the natural shape for the problem.

I believed the pitch. I had run the math. The Bellman equation is real and the optimal policy under the right state representation is in principle better than any myopic rule.

## The actual story was about exploration

What the policy actually did, once we looked at it, was different. It was not learning a clever long-horizon trade-off. It was not exploiting some non-obvious cross-product effect. The action distribution it produced was, segment by segment, not far from what a competent analyst would have written down if you gave that analyst permission to be wrong sometimes.

The win was upstream of the policy. The win was that the agent had visited prices the rules system never tried.

The rules system was good. It was not a strawman. It was the product of years of iteration by people who understood the business, and it produced sensible prices for the segments it had data on. The catch was that the segments it had data on were, by construction, the segments earlier versions of itself had been willing to price. The rules had hardened over time into something close to a fixed point. Every price it set was a price that confirmed the prior. Every segment it priced confidently was a segment it had been pricing confidently for a long time.

The RL agent had no such inheritance. It explored because exploration was in the objective. It tried prices that the rules system, anchored on years of cached intuition about "the right price for this product", would never have tried. Some of those prices were terrible. Some of them revealed a willingness to pay nobody had measured because nobody had bothered to check.

The policy was not that clever. It just looked at the parts of the state space everyone else had stopped looking at.

## Most of the lift would have shown up in a bandit

Once you see that, the question changes. If the value came from exploration discipline, how much of it required Q-learning at all?

Honestly, not much. A well-tuned contextual bandit on the same feature set, with the same willingness to sample under-explored arms, would have captured a large fraction of the same gain. You give up the sequential value function and you keep the part that actually mattered, which was forcing the system to price segments it had previously refused to price. The infrastructure for a contextual bandit is dramatically simpler. The off-policy evaluation story is cleaner. The failure modes are more interpretable, because there is no value function to misread.

I am not saying we should have built a bandit and skipped the Q-learning. The Q-learning project taught us things about state representation and reward shaping that we would not have learned from a bandit. But if the question is "where did the lift come from in production", the answer is not "from the temporal-difference updates". The answer is "from the exploration policy".

This is uncomfortable because it is a less heroic story. It is harder to sell internally. "We built an RL system and it found non-obvious sequential strategies" is a better board slide than "we built a system whose exploration schedule was different from the one we had before".

## When the full RL machinery actually earns its keep

I do not want to over-correct into "RL is never worth it". There are problems where the sequential structure dominates, and you cannot get away with a bandit. The honest list is short and it is worth being specific about it.

Long horizons with real state evolution. If the price set today meaningfully shifts the demand distribution one week out, not in some statistical-correlation sense but in a "your action changes the world" sense, you need a method that propagates value through time. Perishable inventory pricing has a version of this, but the version is weaker than the framing usually implies. Most of the within-day price effect is myopic. The cross-day effect is real but smaller than people expect.

Large state spaces where you cannot precompute a policy table. If the relevant state has continuous dimensions and a meaningful cardinality of categorical features, you need function approximation, and once you have function approximation you might as well do value iteration with it. Contextual bandits can struggle here, depending on the parameterisation, though good linear bandits with feature interactions go further than people credit.

Genuine sequential dependence in the reward. Not statistical dependence. Causal dependence, where the action at time t changes the reward distribution at time t plus k by mechanism, not by correlation. Pricing has some of this. Less than the textbook framing suggests.

If you do not have at least one of these, the marginal gain from full RL over a well-instrumented bandit is small. The marginal cost in complexity is large. The trade-off is rarely worth it on the first version of a pricing system, and often not worth it on the second version either.

## What I take away from this

I went into the project expecting RL to win because it was RL. It won because it explored. Knowing the difference matters, because it changes what you build next.

If you are about to start an RL pricing project, the question worth asking before you write any code is: how confident are you that your current system is sampling the right segments? If the answer is "not very", you do not need RL to fix that. You need to instrument exploration into whatever system you already have, and you need to be honest with yourself about how anchored your current pricing decisions are on assumptions that nobody has stress-tested in a long time.

The version of this post I would have written a year ago would have been a defence of full RL on first principles. I no longer think that is the right read. Most of the win was exploration discipline. The Q-learning was the vehicle. The vehicle was not the point.

RL did not beat the rules. Exploration did.
