---
title: "Being the only person on your team who actually deploys code"
layout: post
date: 2026-07-02 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - career
  - mlops
  - data-science
  - leadership
category: blog
author: samuelgnap
description: Data science teams have a quiet hierarchy in which the one person who can ship to production sets the agenda, and if your team does not have that person, you should become them.
---

The meeting goes around the room. Five data scientists, five slide decks. The XGBoost rework. The new uplift model. A clever hierarchical thing with partial pooling. A reinforcement learning prototype that beat the rules in backtest. By the time it is my turn, I have a quiet observation that I do not say out loud. Of the five models presented, exactly one is running in production. It is mine, and not because it is the best model. It is the only one that anyone has wired into a deployment pipeline.

This is not a story about me being clever. It is a story about a structural pattern that shows up in almost every data science team I have worked with or hired into. There is usually one person who can ship. That person ends up steering the roadmap, often without meaning to, and often without anyone noticing it has happened.

## The notebook graveyard is real

The standard data science career trap is to build interesting things that never reach production. You spend a quarter on a model. The model is good. The notebook is clean. The README has equations. Stakeholders nod in review meetings. Then engineering gets pulled onto a different priority, the platform team is reorganised, the data contract you need is two quarters away, and the model quietly stops being relevant.

I have watched this happen to colleagues whose modelling skills I would put above mine. The bottleneck is almost never the model. It is the gap between a thing that runs in a notebook and a thing that runs at three in the morning when the API the model depends on times out.

The people who close that gap, in most teams, are not the people who got hired for it. They are the ones who decided, on their own time, that a model nobody uses is the same as no model at all.

## The person who ships sets the roadmap

When your work is the only thing actually running in production, your priorities are the team's priorities. This is not political. It is mechanical. The on-call rotation depends on what is in production. The dashboards point at what is in production. The post-mortems are about what is in production. The bonus structure rewards what is in production. Everything in the operating system of the team funnels toward the running code.

So when you propose the next thing to build, the path of least resistance is to extend what already exists. Your model becomes the platform. The hierarchical Bayesian elasticity model someone else built, the one that is technically more correct, sits in a branch. It would require a separate deployment story, separate monitoring, separate rollback logic. Nobody is going to fund that work unless someone fights for it, and the person most able to fight for it is busy keeping the running system up.

I do not love this dynamic. It rewards execution over taste, and taste matters in data science. The fix is not to ship less. The fix is to bring more of the team across the line.

## The skills nobody teaches in a data science programme

The list is unglamorous and it is the entire game. Docker. CI/CD pipelines. The deployment story for whatever cloud you live on, which is AWS or GCP for most people. The Databricks workflow for batch jobs and the serving story for online inference. Writing internal Python packages that other data scientists can actually install with one command. Reading a stack trace from a production failure that has nothing to do with the model. Knowing what a Git rebase will do before you run it.

None of this is in a master's curriculum. Some of it is in MLOps courses online, and those courses are useful, but they teach the pattern, not the muscle memory. The muscle memory comes from breaking production once and fixing it at midnight. There is no substitute.

The thing I tell people who ask how to get there. Pick the smallest model on the team that is currently running in production. Read its repo end to end, including the deployment configuration, the alerting rules, and the rollback procedure. Find one thing in it that is brittle. Fix that one thing. You have just done more for your career than three months of reading papers will do.

## The downside is real and you should know it

Becoming the person who ships has costs that nobody describes when they pitch you on the path.

You become the on-call. When something breaks in production at three in the afternoon on a Friday, the call goes to you, because you are the one who knows where the bodies are. You become the platform owner. When someone wants to deploy a new model, they ask you how. You become the Git champion. People ask you to fix their detached HEAD states. Every infrastructure problem in the building has a tendency to become your problem, because you are the one person who reliably solves them.

This is exhausting. It is also the price of the leverage. I would recommend the path to someone who finds it interesting that the model fails not because the math was wrong but because the parquet file had a schema change three weeks ago that nobody documented.

## The career math

The pure-modelling track caps out at senior IC in most organisations I have seen. There is real demand for deep modelling expertise, and there are companies where it pays well, but in a typical commercial setting, the modelling-only path tops out below the path that combines modelling with shipping.

The ship-it track does not have the same ceiling. It goes through Lead, into staff and principal roles, and often crosses over into engineering management for people who want to manage. The reason is straightforward. A company can hire a contractor for a one-off modelling project. It cannot easily hire someone to own the production system that touches revenue every day. That role compounds with tenure and is hard to replace, which is what compensation tracks.

This is not an argument that the ship-it track matters more than modelling depth. The two are complementary. Teams that do the best work have a mix of both. The career observation is narrower. If you are equally good at the two, the ship-it side will pay better in most places, faster, and for longer.

## For whom this is the right move

If you like building things that work more than you like reading papers, do this. If the satisfaction you get from watching a deployed system handle real traffic exceeds the satisfaction you get from a clean derivation, do this. If you are willing to be the person who carries a pager and answers questions about why the build is failing, do this.

If those things sound miserable, do something else. There is honest work in research, in consulting, in academic-adjacent industry labs. Not every data scientist needs to become a deployment engineer with a Bayesian sideline. The point of this post is not that everyone should ship. The point is that someone on every team has to, and the person who does gets disproportionate leverage over how the team spends its time.

If your team does not have that person, the team is not going to ship. If you want to fix that, you do not have to wait for permission. You can become that person on Monday.
