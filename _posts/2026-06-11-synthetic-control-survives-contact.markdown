---
title: "Synthetic control is the only causal tool that survives contact with a pricing team"
layout: post
date: 2026-04-21 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - causal-inference
  - pricing
  - synthetic-control
category: blog
author: samuelgnap
description: Every causal inference method dies on commercial data except synthetic control, which has the right data shape for the questions a pricing team actually gets asked.
---

I was staring at a request that read something like "what would revenue have done on this route if we had not changed the fare structure last month". One treated route, a panel of controls, a few years of daily history. That is the data shape, and it is the only data shape I get for this kind of question.

Causal inference textbooks present a menu. RCTs at the top, then difference-in-differences, then instrumental variables, then regression discontinuity, then synthetic control somewhere near the bottom. The implication is that you read the question and pick the matching tool. After five years of doing this for a living, I have come to the opposite view. The data picks the tool. Stop pretending it goes the other way.

## RCTs do not survive interference

I wrote a whole post about this last week, so I will keep it short. Unit-level randomisation breaks in pricing because demand interferes between products, search results reshuffle when you move one price, and competitors react inside your treatment window. You can run the test and report the number. You cannot claim it is the causal effect on total revenue, because the assumption the design rests on is not true in your market. More units cannot recover a quantity the design is no longer estimating.

## Difference-in-differences asks for something the world will not give you

Difference-in-differences is the next thing most people reach for. It is also the most common quiet failure I see, because it looks like it is working when it is not.

Parallel trends is a fairy tale in commercial settings. Demand has seasonality. Competitors run campaigns. Holiday calendars shift year on year. A flu outbreak in February dents one market and not another. You can plot the pre-period trends, run a pre-trends test, and convince yourself the assumption holds. Then the moment you treat one unit, the trend stops being parallel for reasons that have nothing to do with your intervention.

The honest read is that the pre-period test is necessary and nowhere near sufficient. You are betting that the counterfactual control trajectory would have stayed parallel forever, conditional on a finite window of pre-period observation. Most teams making that bet have not priced it.

## Instrumental variables require instruments

Textbook IV examples in pricing all involve cost shocks, weather, fuel prices, or regulation. They are clean in the textbook because the textbook gets to choose its example. In practice, for a specific commercial question, one of two things is true. Either the candidate instrument does not satisfy exclusion (the same shock that moves your price also moves the demand you want to measure), or you do not have the data on it at a useful resolution. I have spent enough time looking for instruments to say with confidence that they are not lying around. When someone in a meeting suggests an IV approach, the usual answer is that the proposed instrument is a treatment in disguise.

## Synthetic control fits the shape you actually have

Synthetic control builds a counterfactual for one treated unit by taking a weighted combination of donor units, with weights chosen so the combination tracks the treated unit closely over a pre-treatment window. After treatment, the synthetic version keeps going as if nothing had happened. The gap between the real treated unit and its synthetic twin is the estimated effect.

Three things make it land for pricing.

The data shape is right. One treated unit, many candidate controls, panel observations. That is what a pricing team has after almost any intervention. One route gets a new fare ladder. One product family gets repriced. One channel changes its margin. The world hands you one treated thing and a hundred candidate cousins.

The weights are estimated, not assumed. DiD assumes equal weights and parallel paths. Synthetic control lets the data choose which donors look like the treated unit and how much each one contributes. If the closest match to your treated route is 60% one comparator and 40% another, that is what you get.

The pre-treatment fit is testable. You can look at it. You can quantify it. You can decide whether to trust the method on this case before you let it tell you what the effect was.

That last point is the one I think is most under-discussed, and it is the one I want to spend the rest of the post on.

## Pre-treatment fit is the entire test, and most people skim past it

The standard synthetic control output is a chart. The treated unit and its synthetic twin overlaid for the pre-period, then a divergence after treatment. People look at the divergence and quote it as the effect.

The chart that matters is the one before the dashed line. If the synthetic twin tracks the treated unit cleanly through the pre-period, including the bumps and the seasonality and the recovery from last year's shock, the method has found a credible counterfactual. If the pre-period fit is mediocre, the post-period gap is uninterpretable. You are not measuring a treatment effect. You are measuring a treatment effect plus an unknown amount of "the model never fit this unit in the first place".

I treat the pre-treatment root mean squared prediction error as the gate. If it is small relative to the post-treatment gap, the result is real. If it is comparable to the post-treatment gap, the result is noise. I will not show a synthetic control estimate to a stakeholder without showing the pre-period fit alongside it, because the pre-period fit is the credibility, and the headline number on its own is misleading.

The corollary is the failure mode. When you do not have good donor units, synthetic control will still return weights, still draw a chart, and still hand you a post-treatment estimate. It will simply not have fit the pre-period well. The method does not refuse to answer. It answers badly, and pretends not to. The discipline is on the analyst.

## A stylised example

To make this concrete, imagine the setup the way I run it. One product family received a price change at a known date. A donor pool of twenty other product families did not. The outcome is daily bookings. The pre-period is two years of daily data, the post-period twelve weeks.

You set up the optimisation to choose non-negative weights summing to one across the donor pool, fitting on the pre-period bookings series with controls for day of week, lead time mix, and a seasonal index. The optimiser returns weights, most of which are zero. Three or four donor families carry the synthetic twin. The pre-period fit looks clean, with the synthetic series tracking the real one through the visible spikes and dips.

After the dashed line, the real treated series and the synthetic one diverge. The cumulative gap is the estimated effect. Two diagnostics check it. A placebo test pretends each donor family was treated in turn, fits synthetic controls on the remaining donors, and looks at the distribution of placebo gaps. If the real gap sits in the tail of that distribution, the effect is unlikely under the null. A leave-one-out test removes each donor in turn, refits, and shows how much the estimate moves. If the estimate is stable, you are not leaning on a single donor.

Public datasets in this shape exist. The California tobacco control example from Abadie and coauthors is the one most people learn on. For pricing specifically, the Dominick's Finer Foods scanner data has the right panel structure and is openly available.

## What this changes about how I work

When a stakeholder asks the kind of question that has one treated thing and a panel of comparators, my first move is synthetic control. Not DiD, not an attempt to find an instrument, not a creative RCT design. Synthetic control is the only causal tool that has the right shape for the questions you actually get asked.

The rest of the methods are not useless. They are useful when the data shape fits them, which in commercial pricing is not most of the time. If you find yourself running DiD on a single treated unit because that is what you learned in the textbook, stop and look at the data. The method should follow the panel, not the other way round.
