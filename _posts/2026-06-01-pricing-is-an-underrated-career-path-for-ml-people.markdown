---
title: "Pricing is an underrated career path for ML people"
layout: post
date: 2026-06-01 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - career
  - pricing
  - ml
category: blog
author: samuelgnap
description: Ambitious ML talent should consider pricing roles over FAANG because the problems are richer, the field is less crowded, and ownership arrives faster.
---

Picture two engineers with the same CV. One joins a senior IC track on a FAANG ad ranking team. The other becomes a Lead DS on an airline pricing team. Five years later both have done good work. The first has shipped one slice of a giant ranker, knows the model architecture in detail, and is one of forty people on the team. The second has owned the demand model, the optimisation layer, the experimentation framework, and the production handover, and is one of six. The skill ceilings are similar. The two careers are not.

I write this as the second engineer, after watching enough strong ML people sleepwalk into the first path because nobody told them the second one was on the menu.

## The path most ambitious DS people walk into without thinking

The implicit ranking goes FAANG, then a research lab, then a startup. Pricing sits somewhere below all of these in the unwritten hierarchy, alongside "operations data science" and other rooms with bad lighting. It is read as a back-office function that runs regressions to set numbers. The ML talent pool sorts itself accordingly. The strongest applicants apply to the obvious places, the obvious places hire the strongest applicants, and the cycle confirms itself.

There is nothing wrong with FAANG. The compensation is real, the engineering bar is real, the people are excellent. I am not running them down. I am pointing out that the default path is the default for reasons of brand and pay, not for reasons of problem quality. If you have not actually compared the problems, you are buying a brand on the assumption that it correlates with the work. Sometimes it does. Often it does not.

## What pricing actually does

Pricing teams make decisions, in production, that move money. The job is to figure out what to charge for a finite resource that customers value differently depending on time, context, and competition. That is a sequence of modelling and optimisation problems where the data is endogenous, the loss function is asymmetric, and the environment fights back.

Concretely, a working pricing team runs:

- Demand models, often hierarchical, often Bayesian, because the same data point speaks to twenty correlated decisions.
- Real-time inference. Prices update on the order of minutes, not days. There is no notebook-to-prod gap. There is a model and a serving path.
- Reinforcement learning, used because rules-based systems get out-explored by anything that systematically tries new prices. Production RL is rare in industry. Pricing is one of the few places it actually pays for itself.
- Causal inference at scale. A/B tests, switchbacks, synthetic control, geo splits. Not because the team is showing off methodology, but because unit-level randomisation does not work when units interfere.
- A loss function that includes regulation, brand, and competitor reaction. The math is not the hard part. The hard part is what you optimise.

This is the kind of stack the ML community talks about when it talks about how it wants the work to look. In pricing, it is just the work.

## What you get that you do not get at FAANG

The headline difference is ownership. On a pricing team of the size most companies run, one person can own the model that sets prices for a meaningful portion of revenue. Not "contribute to". Own. The decision to ship is yours. The board presentation is yours. The post-mortem is yours.

The second difference is competition for headcount. Every ambitious DS goes to FAANG. Almost none come to pricing. That is the opportunity. The applicant pool is thinner. The bar to get hired is not lower in absolute terms, but the variance in candidate strength is higher, and a strong candidate stands out more. Once you are in, the people around you are senior, the team is small, and the work is leveraged. There is nowhere to hide and nowhere to be diluted.

The third difference is asymmetric responsibility. In a small team you are accountable for things that on a big team would be three layers of management away. You will present to commercial leadership. You will own a system that, if it breaks, costs real money in real time. You will be the person who has to explain a counterintuitive recommendation to people who do not have a stats background, and convince them anyway. Asymmetric responsibility is the best teacher there is.

## What you give up

Brand recognition. A FAANG name on the CV opens doors that an airline name does not. This matters in some contexts. It matters less than people assume in others. Hiring managers at quant funds and AI-native companies care more about what you have shipped than where you shipped it. Recruiters at less specialised firms care more about the logo.

Compensation, at junior and mid levels. The gap closes at senior and Lead levels because pricing is leverage-heavy and the small team structure pays well at the top. If you are early career and optimising for total comp over the next three years, FAANG wins. If you are looking at the next ten, the comparison is closer than the salary tables suggest.

Perks. Free food, on-site gym, conference budgets large enough to attend ICML without filing an expense report. These are real. They are also not the reason most senior people stay at their jobs.

## When pricing is the wrong move

If you want to publish papers, do not do this. The work is proprietary and the incentive to write it up externally is low. There are exceptions, but the median pricing scientist publishes nothing.

If you need a famous logo to clear a credibility bar (a visa application, a credential-sensitive cofounder relationship, a particular kind of recruiter pipeline), the FAANG name does the work that pricing experience will not do for years. Pick the brand. Be honest with yourself about why.

If you dislike commercial context, this is not for you. Pricing sits next to revenue, and revenue sits next to executives. You will be asked to explain modelling decisions to people who measure success in pounds and dollars. If that prospect bores you, pick a job where the customer is another engineer.

## The hiring read

Pricing teams hire less aggressively than FAANG and have lower turnover. The implication for candidates is structural. The bar to get in is high, because postings are rare and the team has time to be picky. Once in, the path is longer and more stable. The role compounds. A pricing scientist with five years on one team has owned more systems end to end than a FAANG IC with the same tenure on the same level, simply because the surface area per person is bigger.

From the hiring side, the candidates who stand out are the ones who have shipped something, anywhere, that is still running. Not the ones with the longest list of architectures. ML at FAANG selects for breadth of model familiarity. Pricing selects for depth of decision ownership. Different jobs.

If the standard track is the right one for you, take it. If you have been told it is the only ambitious one, that is not true. The room with bad lighting has the harder problems and fewer people in it.
