---
title: "Overbooking is a no-show problem with extra steps"
layout: post
date: 2026-06-25 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - overbooking
  - ml
  - optimization
  - operations
category: blog
author: samuelgnap
description: Predicting no-show rate is the easy half of overbooking; the hard half is an asymmetric loss function in which bumping a passenger costs many times more than an empty seat.
---

Someone in a policy meeting once said it out loud. "Let us just maximise expected revenue." The model would predict no-shows per flight, we would set the overbooking limit at the revenue-maximising point, and we would be done by lunch. The room nodded. I had to be the person who explained why that was wrong, and the explanation took the rest of the afternoon.

The prediction part had been easy. The argument that followed was the actual problem.

## The naive setup is the easy half

The standard pipeline is short. Predict a no-show probability per booking. Aggregate the probabilities to a no-show rate per flight. Compute expected revenue as a function of the overbooking limit. Pick the limit that maximises expected revenue.

The prediction layer is honestly a solved problem for any team with a real booking dataset and a few weeks of engineering. I have built it with XGBoost and Hyperopt for the search over hyperparameters, deployed on a Databricks cluster, with the usual features: lead time, fare class, channel, party size, day of week, prior no-show history on the route. The AUC is high enough to be useful and stable enough to deploy. The hard part of overbooking is not here.

The hard part is the next step. The step where the model output meets a decision and somebody has to write down the cost function.

## Bumping is not the inverse of an empty seat

The mistake hidden inside "maximise expected revenue" is the assumption that the cost of one bumped passenger is roughly the revenue from one extra seat sold. They are not in the same units.

An empty seat costs you the marginal revenue of that seat. That is bounded above by the fare for the cabin. It is also bounded below by zero, because the cost is an opportunity cost on inventory that has perished anyway.

A bumped passenger costs you several things at once. There is the direct rebooking and ground-handling cost. There is statutory compensation, which in Europe is set by the EU261 framework and in the United States by the Department of Transportation rules. There is the cost of accommodating the passenger if the next flight is the following day. There is the lifetime-value hit from a customer who tells everyone they know about the experience. The expected cost of a bumping event is meaningfully larger than the expected revenue of selling one more seat. The exact multiple depends on the carrier, the market, and the regulator. In every business case I have ever built, it is well above one.

That is the loss function nobody writes down. It is also the only number that matters.

## Optimising under asymmetric loss changes the answer

Once you replace expected revenue with expected revenue minus expected bumping cost, the optimisation problem changes shape. The marginal value of selling the next standby seat is still the fare. The marginal cost has a step in it: it is small while you are under the no-show rate, and it explodes the moment the realised no-show rate falls below your overbooking buffer.

The right objective looks roughly like:

```
maximise   E[revenue(limit)]  -  c_bump * P(bumping | limit) * E[bumps | limit, bumping]
```

The interesting term is the bumping cost `c_bump`, calibrated to the real number rather than to the fare. When you re-derive the optimal overbooking limit under this objective, the answer is consistently lower than the naive optimum. Sometimes by a seat, sometimes by more. The model is the same. The data is the same. The cost function has the only opinion that matters.

## Reputational damage is not a one-shot cost

Even this adjusted objective understates the real cost, because it treats bumping as if each event were independent. It is not.

A bumping incident in 2026 does not stay inside the airport. It is filmed, posted, screenshotted, picked up by an aggregator, occasionally turned into a news segment. One event can affect bookings on a route for weeks. The cost function is convex in the number of bumps per period, not linear. Carriers behave more conservatively than the simple expected-value math suggests, and they are right to. The math is missing a term.

I have not seen a clean way to estimate this convex term from first principles. The best teams I know fold it in as a multiplier on `c_bump` and revisit the multiplier after every incident that gets attention. It is not elegant. It is the right thing to do.

## Regulation is in the objective whether you write it down or not

EU261 and the US DoT rules are sometimes treated as external constraints, the kind of thing a compliance team handles separately from the data science work. That framing is convenient and wrong. The compensation schedule those rules define is exactly the floor on `c_bump`. The legal cost is part of the cost function. The reputational cost sits on top of it.

The practical version of this for a modelling team is straightforward. When you sit down to write the loss function for the overbooking optimiser, the regulatory compensation table is one of the inputs. You do not have to wait for legal to hand it to you. It is public. The bigger danger is that the modelling team writes a clean revenue-maximising objective, ships it, and discovers later that the regulator has been part of the objective the entire time, silently, through the costs the airline books when things go wrong.

## What I would tell the next team building this

Spend less time on the prediction model than you think you should. Spend the saved time on the cost function.

A no-show model that is two percentage points worse on AUC but plugged into a properly calibrated asymmetric loss will outperform a state-of-the-prediction model plugged into "maximise expected revenue". I have watched this happen. The expensive sophistication is in the optimisation step, not in the prediction step, and the optimisation step is where most teams stop paying attention because the modelling is over and the spreadsheets begin.

If your overbooking system has a knob anywhere that says "expected revenue", check what it is treating bumping as worth. If the answer is a fare or anything close to a fare, the system is set up to fail in a way that will only become visible on the days you most want it to behave.

The hard part of overbooking is not the prediction. It is the cost function. Bumping a passenger does not cost one seat. It costs trust, and trust does not show up in next quarter's revenue line until it is already gone.
