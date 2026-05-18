---
title: "Demand elasticity is not a number"
layout: post
date: 2026-05-08 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - elasticity
  - bayesian
  - pymc
  - hierarchical
category: blog
author: samuelgnap
projects: true
description: Reporting demand elasticity as a point estimate throws away the most useful part of the signal; what you want is a posterior, pooled hierarchically across routes.
---

Someone in a planning meeting asks "what is the elasticity for this route" and waits for a number. I have learned to answer the question by drawing on the whiteboard. I sketch a distribution. I mark the mean. I shade the region that contains most of the mass. I point at the tails and say "if the true value is here, we lose money cutting price; if it is here, we make money". The room either nods or stops the conversation to argue, and either outcome is better than handing over a scalar that nobody can act on responsibly.

Elasticity is not a number. It is a posterior. That is the entire post, but the reasons it matters are worth a few minutes.

## The scalar habit

Most working data scientists learn elasticity from a textbook regression. You fit log demand on log price, possibly with controls, and you read off the coefficient. The textbook reports a point estimate and a standard error. The standard error is acknowledged, perhaps starred, and then in the next chapter it disappears. Industry practice does the same thing on a larger scale. By the time the number reaches a planning meeting it has become a single decimal place on a slide. The interval is gone. The route-level variation is gone. The dependence on the data window is gone.

This is not a complaint about econometrics. It is a complaint about the translation step between an estimate and a decision. The estimate has structure. The decision throws away the structure. That is where the value leaks.

## Why decision-makers actually need the uncertainty

Pricing decisions are loss-asymmetric. The pain of cutting price into inelastic demand is not symmetric with the gain of cutting price into elastic demand. If your point estimate is -1.1 and your interval runs from -0.6 to -1.6, the recommendation depends on where you sit in that interval far more than it depends on the mean. At -0.6 you should hold price. At -1.6 you should cut. The decision flips inside the credible region, and presenting only the midpoint hides the flip.

I have watched commercial teams act on a single elasticity figure for months before noticing that the underlying posterior was so wide it could not distinguish "raise price" from "cut price". The estimate was unbiased. It was also useless, because the variance dominated the decision. Once we started reporting the full posterior, the conversation changed. People stopped asking "what is the elasticity" and started asking "how confident are we, and what should we do under the wide cases". Those are the right questions.

The variance of the estimate matters more than its mean.

## Why a hierarchical model is the right shape

Routes are not independent observations from a single distribution. A leisure route to a beach destination behaves differently from a business route into a financial centre, and a thin overnight route behaves differently from a high-frequency hub-to-hub. Estimating one global elasticity pools too aggressively and hides the structure. Estimating each route separately pools nothing and produces wildly noisy posteriors on the thin routes, which then get reported with false confidence.

The answer is to fit a hierarchical model. Route-level elasticities are drawn from a common distribution whose parameters are themselves estimated from the data. Routes with thousands of observations dominate their own posterior. Routes with very little data borrow strength from the global distribution and end up near the population mean, which is almost always what you want for a thin route. That borrowing is not a hack. It is the formal Bayesian way of saying "I do not know much about this route, so I will pull toward what I know about routes in general".

There is a small art to specifying the model. You want weakly informative priors on the global location and scale, not flat priors. You want a parameterisation that gives the sampler an easy job (more on this below). And in pricing you usually want to constrain the price coefficient to be negative, because nobody wants to read a posterior that gives ten percent probability to upward-sloping demand on a route where the data is thin.

Below is a runnable PyMC v5 snippet that does all of this on synthetic data. Eight routes, varying numbers of observations each, route-level partial pooling, weakly informative priors, and a force-negative price coefficient implemented by sign-flipping a positive prior.

```python
import numpy as np
import pymc as pm
import arviz as az

rng = np.random.default_rng(42)

# Synthetic panel: 8 routes, 60-180 observations each, log price -> log demand.
n_routes = 8
true_alpha = rng.normal(5.0, 0.6, size=n_routes)        # route intercepts
true_beta_pos = rng.gamma(2.0, 0.5, size=n_routes)      # magnitudes of elasticity
true_beta = -true_beta_pos                              # force negative

route_ids, log_price, log_demand = [], [], []
for r in range(n_routes):
    n_obs = int(rng.integers(60, 180))
    lp = rng.normal(4.5, 0.25, size=n_obs)              # log price grid per route
    noise = rng.normal(0.0, 0.20, size=n_obs)
    ld = true_alpha[r] + true_beta[r] * lp + noise
    route_ids.append(np.full(n_obs, r))
    log_price.append(lp)
    log_demand.append(ld)

route_ids = np.concatenate(route_ids)
log_price = np.concatenate(log_price)
log_demand = np.concatenate(log_demand)

coords = {"route": [f"route_{i}" for i in range(n_routes)]}

with pm.Model(coords=coords) as model:
    # Global elasticity magnitude. Force negative by sign-flipping a positive prior.
    mu_beta_pos = pm.HalfNormal("mu_beta_pos", sigma=1.0)
    sigma_beta = pm.HalfNormal("sigma_beta", sigma=0.5)

    # Route-level partial pooling on the positive magnitude.
    beta_pos = pm.Gamma("beta_pos", mu=mu_beta_pos, sigma=sigma_beta, dims="route")
    beta = pm.Deterministic("beta", -beta_pos, dims="route")  # always negative

    # Route intercepts pooled around a global mean.
    mu_alpha = pm.Normal("mu_alpha", mu=5.0, sigma=2.0)
    sigma_alpha = pm.HalfNormal("sigma_alpha", sigma=1.0)
    alpha = pm.Normal("alpha", mu=mu_alpha, sigma=sigma_alpha, dims="route")

    sigma = pm.HalfNormal("sigma", sigma=0.5)
    mu = alpha[route_ids] + beta[route_ids] * log_price
    pm.Normal("log_demand", mu=mu, sigma=sigma, observed=log_demand)

    idata = pm.sample(
        draws=500, tune=500, chains=2, target_accept=0.9,
        random_seed=42, progressbar=False,
    )

print(az.summary(idata, var_names=["mu_beta_pos", "beta"], round_to=3))
```

Running this gives a posterior summary table with one row per route. The means cluster around the simulated truth. The standard deviations are smaller on the routes with more observations and larger on the routes with fewer, which is exactly the shrinkage pattern you want. The global magnitude `mu_beta_pos` sits in the middle, and the route-level posteriors pull toward it for the thin routes and away from it for the thick ones.

If you swap `print(az.summary(...))` for `az.plot_forest(idata, var_names=["beta"], combined=True)` you get a visual of the same point. Each route has its own interval. The intervals on data-poor routes are wide and overlap the global mean. The intervals on data-rich routes are tight and clearly distinct. That is the shrinkage story in one chart, and it is the chart I show in planning meetings instead of handing over a list of numbers.

## What the posterior actually changes about the decision

Three things change once you stop reporting scalar elasticities and start reporting route-level posteriors.

First, when the posterior is tight, act on the mean. The interval has narrowed enough that decisions are stable across it. This is the easy case. Most mature, high-volume routes fall here, and the team can move quickly.

Second, when the posterior is wide, hedge. Wide posteriors do not mean "no signal". They mean "the signal is not strong enough to commit to a directional move". The right action is usually a smaller move and a faster feedback loop, not a confident move informed by a midpoint.

Third, when the route is new or thin, the prior pulls toward the global mean. That is exactly what you want. You did not have evidence to override the population view, and the model is honest about it. A thin-route point estimate fitted in isolation will be wherever the noise happens to land. A thin-route posterior in a hierarchical model is anchored to what is plausible.

These three modes correspond to three different operational tempos on a pricing team. Once you write the modes down, planning conversations get faster, because the question shifts from "what is the elasticity" to "which mode is this route in".

## The Cholesky trick, briefly

The model above pools the intercept and slope separately, treating them as independent. In real pricing data they are not independent. High-baseline-demand routes tend to be less elastic, low-baseline routes more elastic, and the two random effects share information. The correct fix is a multivariate normal on the joint random effect with a full covariance matrix, but the naive parameterisation samples poorly. The standard solution is the Cholesky reparametrisation: sample the lower-triangular factor of the covariance directly with an `LKJCholeskyCov` prior, and then build the random effects from the factor and a standard normal. The sampler runs dramatically faster, and the posterior is the same one you would have got from the naive parameterisation if you had the patience to wait for it.

I have left it out of the snippet to keep the post short. It is the change I would make second, after going hierarchical.

## The implication

Most working pricing teams already have the data and the tools to do this. The blocker is not technical. It is reporting. The slide template asks for a number, the meeting agenda asks for a number, the planning system stores a number, and the modeller obliges. Each step in the chain quietly drops the variance, and by the time the figure reaches a decision-maker, the most informative part of the signal is gone.

The fix is not "make the chart bigger". It is "stop pretending the answer is a scalar". Once you report posteriors and route-level shrinkage, the team learns to ask better questions, and the questions are the part that actually changes outcomes.

If your elasticity workflow ends in a single decimal on a slide, the workflow has a leak. The good news is that you already have everything you need to plug it. Two pages of PyMC, a forest plot, and a willingness to walk into a planning meeting with a distribution instead of a number.
