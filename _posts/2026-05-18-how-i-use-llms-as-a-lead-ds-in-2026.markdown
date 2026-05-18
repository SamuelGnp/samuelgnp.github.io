---
title: "How I use LLMs as a Lead DS in 2026"
layout: post
date: 2026-05-18 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - llms
  - claude-code
  - mcp
  - workflow
category: blog
author: samuelgnap
description: LLMs make me dramatically faster on wide-shallow work and almost useless on deep-narrow work, and the job is figuring out which kind of problem I have in front of me.
---

I use Claude Code daily. It speeds me up by something like 3x on most things. It is also wrong in ways that would have killed projects if I had not caught them.

That is the whole post in three sentences. The rest is what I have learned about which work the LLM is good for, which work it actively hurts, and how the day-to-day shape of a Lead DS job changes once you accept the asymmetry.

I work on dynamic pricing for an airline. The stack is Python, PyMC for the Bayesian models, XGBoost where the data is big and the structure does not matter, Databricks for everything that touches the warehouse, and Claude Code as my editor of choice. None of that is interesting on its own. What is interesting is the pattern of where the LLM helps and where it lies to me.

## Wide and shallow is where it earns its keep

There is a class of work that fills a Lead DS week and that nobody enjoys. Translating a dialect of SQL into another dialect. Writing pytest fixtures for a module that was tested by hand for two years. Sketching out Terraform for a new workspace. Pulling a data dictionary out of fifty tables nobody documented. Drafting docstrings for a library that already works. Scaffolding a new repo with the same five files I always start with.

All of this is wide and shallow. The surface area is large, the depth at any point is small, and the patterns are well represented online. The LLM has seen ten thousand examples of every one of these. It produces something competent in seconds. I read it, correct two things, and ship.

This is where the 3x number lives. It is not a benchmark. It is the felt sense of starting a Monday and finishing on Wednesday what would have taken until Friday. I will not romanticise it. The work was boring before and it is still boring, but it occupies less of the week, which is the only thing that matters.

## Narrow and deep is where it lies

Pricing and revenue management is small enough as a field that the training data is thin. Ask Claude to write a hierarchical Bayesian elasticity model with route-level shrinkage and a force-negative price coefficient, and it produces something confidently wrong. The model runs. The posterior plots look fine. The signs of the slopes are flipped. The non-centred parameterisation is half applied, so the sampler will hit divergences the moment the data has any structure. None of this announces itself. You only catch it if you already know the answer.

I have spent more time debugging plausible-looking models the LLM wrote than I would have spent writing them from scratch. The first few times this happened I blamed myself for not specifying the problem clearly enough. After the fourth or fifth round I stopped blaming the prompt. The problem is structural. You cannot trust the LLM on the parts of the work that actually require you.

The same thing happens with research. Ask for the state of the art on switchback designs under cross-product interference and you get a confident summary that misses the three papers that matter, includes one that was retracted, and cites a fourth that does not exist. The LLM surfaces what is common. Research is the opposite of common. I now treat it as a synonym dictionary that helps me build keywords for a real literature search, and I do the search in Google Scholar like a person.

## The trick is to never let it think for you

When the work is complex, the LLM is a fast typist and nothing more. I write the model spec, the constraints, the priors, the test cases, the expected shape of the posterior. It writes the code. I read every line. The moment I let it choose the prior, it picks a wide normal that breaks the sampler. The moment I let it design the test, it tests that the function returns something instead of that it returns the right thing.

The moment I think for it, though, it is the best collaborator I have ever had. It does not get tired. It does not miss the third edge case because it is hungry. It produces a hundred lines of correct boilerplate while I am thinking about the one line that matters. It is the best collaborator I have ever had, as long as I never let it think for me.

That sentence sounds like an aphorism. It is also a working rule. When I notice myself accepting a suggestion because it looks reasonable rather than because I have verified it, I stop and verify. That habit is most of what separates the days the LLM saves me hours from the days it costs me a morning of debugging.

## The shape of the job has changed

A year ago I was a violinist. I worked on one instrument at a time, deeply. The interesting questions were small enough that I could hold them in my head, and the work was a sustained close reading of one problem.

Now I am closer to a conductor. I have five agents running in parallel. One is writing tests for an old library. One is documenting a repo nobody has touched since 2022. One is investigating an anomaly in last week's bookings. One is drafting an analysis I will read tonight. One is watching a production job and will ping me if something looks wrong. My job is to know the score, set the tempo, and catch the wrong notes when they happen.

This is more leveraged. It is also mentally taxing in a different way. You are never inside the work. You are always above it. The pleasure of getting absorbed in a single problem for four hours is mostly gone, replaced by the pleasure (or the strain, depending on the day) of running a small portfolio of half-finished things. I miss the absorption sometimes. I do not miss the throughput it cost me.

A year ago I was a violinist working on one instrument at a time. Now I am a conductor with five agents in front of me. The job is to know the score and catch the wrong notes.

## What makes conductor mode actually work

Conductor mode is a fantasy without the plumbing. An agent that cannot see your data is a chatbot, and a chatbot is not a colleague.

The piece that closed the gap for me was MCP. I built a small Databricks MCP server in a week. It exposes table discovery, schema inspection, and query execution to any compliant client. Reads are on by default. Writes are gated behind explicit confirmation. There is a full audit trail. It is unglamorous and it is the entire reason the conductor metaphor is not a marketing line.

With it, the agent investigating an anomaly can actually pull the data. The agent drafting an analysis can check whether the join it is about to suggest is sensible. The agent watching production can read the run history without me copying log lines into a chat window. Without that bridge, every "agent" is a chatbot with a long context window.

I will not turn this post into a tutorial on MCP. There is one coming. What matters here is that the part of the stack that made my workflow feel different was not the model, it was the protocol that let the model touch the world I work in.

## What I will not use them for

I do not use LLMs to decide anything that matters. Which model to use. Which metric to optimise. Whether a result is worth shipping. Whether an experiment is correctly specified. Whether a stakeholder is asking the right question or a different one I should answer instead.

These are exactly the things that distinguish a senior data scientist from a junior one, and they are exactly the things the LLM is worst at. The training distribution does not contain the context of my team, the history of decisions on the route, the political reality of the meeting I am presenting to next week, or the specific way my CCO interprets uncertainty. An LLM that confidently picks a metric for me is doing me harm in a way I will not notice until the metric is wrong and the decision is already half made.

So I keep the cognitive parts of the job. The LLM gets the typing and the searching and the boilerplate and a fair amount of the testing. That trade is the best trade I have made in five years.

## The implication

If you are a Lead DS trying to work out where to push and where to hold back, the test I use is simple. Is the work I am about to hand off wide and shallow, with patterns that are well represented in public code? Hand it off. Is it narrow, deep, and specific to your domain in a way the internet has not seen? Do it yourself, and let the LLM type while you think.

Most teams I talk to are getting this backwards. They use the LLM to write the model and write the boilerplate themselves. The model is wrong and the boilerplate takes a week. Switch it around. The week opens up.

I will be wrong about parts of this within a year. The models will get better at the narrow-deep work, or they will not, and either way the answer is to keep testing where the line is. For now, in 2026, the line is where I have drawn it, and the job is mostly about knowing which side of it the next task sits on.
