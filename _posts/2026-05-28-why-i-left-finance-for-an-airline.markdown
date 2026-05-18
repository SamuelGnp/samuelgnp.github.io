---
title: "Why I left finance for an airline"
layout: post
date: 2026-04-10 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - career
  - pricing
  - finance
category: blog
author: samuelgnap
description: Airline pricing is closer to fixed-income market making than people realise, and it is a better place to learn the craft because the problem shape is the same but the clock is slower and the ownership is higher.
---

A former colleague from EIB messaged me last month asking, half seriously, whether he should jump to an airline. He had been pricing corporate bonds for eight years and was bored. He wanted to know if airline pricing was a real quant job or a glorified Excel model.

I told him the same thing I am about to write here. An airline gives you less data than a bank. That makes the modelling harder, not easier. Five years in and the trade still looks the same to me: brand for craft.

## The path that got me here

I started at the European Investment Bank, working on corporate bond pricing and an XVA library in Python. XVA is the family of adjustments banks make to derivative prices for counterparty risk, funding, and capital. The work was technical, the books were thick, and the data was clean. It was a good first job and I learned how to read a fixed-income market.

In the middle of that I did a master's thesis at Imperial College that I still think about regularly. We built a multinomial logit choice model with Bayesian updating and a deep learning component, written from scratch in Python and C++, on hundreds of thousands of flights. The project was supervised by professors at Imperial and at Erasmus in Rotterdam. It was the first time I had to glue together econometrics, optimisation, and a real engineering pipeline in one project, and it was the first time I noticed that airline data was a problem worth a serious career.

After Imperial I joined an airline pricing team. I have been doing some version of dynamic pricing ever since.

## The two problems are closer than they look

Fixed-income market making and airline pricing share more structure than most people on either side recognise.

Both have thin liquidity for any individual product. A specific corporate bond trades a few times a week. A specific seat on a specific flight on a specific day sells, in the aggregate, a few times a day. You never have the dense, well-behaved data that powers a typical FAANG recommender. You always have a panel with holes in it.

Both have asymmetric information. The market maker knows their inventory and their flow. The customer knows their reservation price. Neither side knows the other's. The pricing problem is, in part, a problem of inferring private information from public actions, and that inference is what the model is for.

Both have mean reversion punctuated by jumps. Bond spreads drift, then a credit event repricings them. Fares drift, then a competitor changes a schedule or a sporting event hits the calendar and the curve jumps. The same statistical instincts apply. So do the same failure modes when you model the drift and forget the jumps.

Both operate inside regulatory constraints that shape the objective function. In finance it is capital rules, conduct rules, best-execution requirements. In airlines it is consumer-protection rules, transparency requirements, slot constraints, EU261. In both cases the modelling team that ignores the constraints ships a model that the business cannot deploy.

If you took the problem statement on a whiteboard and stripped the labels, a fixed-income quant would recognise the shape of airline pricing inside a few minutes. The state variables are different. The mathematics is not.

## What is different, and what is better

The first thing that is different is information. Banks have Bloomberg, a consolidated tape, broker quotes, and a regulator that publishes trade data. An airline has its own bookings and a partial, lagged view of competitors through shopping data and screen scrapes. The information set is smaller and noisier.

People assume that is bad for modelling. It is not. It is good for modelling. When the data is rich and clean, the modeller is in competition with hundreds of other people in the same building and thousands across the street. The marginal contribution of any given model is small. When the data is thin and structural assumptions matter, the modeller is the work. The choice of likelihood, the prior, the pooling strategy, the way you treat censored bookings, the way you handle the no-show problem: these become the product, not the wrapper around the product.

The second thing that is different is the distance from your code to a decision. In a bank you might own a piece of a pricing library used by a desk that sits two floors away. The model goes through a model risk function, a validation team, a deployment process, and a desk head who decides what to do with the number. In an airline you build the model, you ship the model, you stand in front of commercial leadership when the model is wrong, and you change the model the next week. Three layers collapse into one. The compression is exhausting on a bad week and clarifying on every other week.

The third thing that is different is the shape of the day. Bank quant work is, in my experience, more about producing artefacts (a calibration, a report, a library update) and less about owning a live system. Airline pricing has a clock. Inventory moves all day. The model is making decisions while you sleep. You feel responsible for a thing that is alive, not for a deliverable that is finished.

## What is different and worse

I will not pretend this is a free lunch.

Compensation is lower. Not catastrophically lower at the senior end, but consistently lower, and the gap is larger at the junior end. If the priority is total comp over a five-year window, an airline is not the optimisation.

Brand recognition is lower. There are still recruiters who do not understand what a revenue management team does, and there are still hiring managers at other firms who read "airline" on a CV and assume it means scheduling software. The signal does land eventually, but it lands more slowly than "Goldman Sachs" or "DeepMind" lands.

The tooling is, on average, less mature than at a top bank. Banks have been spending money on infrastructure for forty years. Airlines have been spending money on infrastructure for fifteen. You will, at some point, write code that fills a gap that does not need to be filled at a bank because someone else filled it in 1998.

None of these are reasons not to do it. They are reasons to be honest with yourself about what you are choosing.

## The structural read

The thing I want to be careful about is that this is not a story about good people versus bad people. The bank colleagues I worked with were sharp, generous, and serious about the craft. The constraint is not the people. The constraint is the shape of the work.

A bank is a regulated entity carrying a balance sheet that, if it moves the wrong way, ends careers and occasionally ends institutions. The shape of the work has to be: many layers, slow change, validated steps. That is correct for what banks are. It is also a slower way to learn the craft, because the loop between "I had an idea" and "the idea is in production and I can see whether it worked" is long.

An airline is not carrying that balance sheet. It is selling perishable inventory under competition. The shape of the work can be tighter. You can move faster, fail faster, and learn faster. For a quant in the first ten years of their career, that loop is the thing that compounds.

## Who this is for

If you want to publish papers, an airline is the wrong move. If you want the largest possible total comp number, an airline is the wrong move. If you want a famous logo on the CV to clear a credibility bar later, do the bank first and the airline second.

If you want to own a model end to end, ship it to production, stand in front of leadership when it breaks, and rebuild it on Monday morning, the airline is the right move and most quants do not know it yet. The problem shape is the one you already understand. The clock is slower in the way that lets you actually learn. The freedom is higher in the way that matters.

That is the trade, and five years in I would make it again.
