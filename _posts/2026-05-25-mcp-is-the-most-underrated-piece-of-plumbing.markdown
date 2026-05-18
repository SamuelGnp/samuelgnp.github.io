---
title: "MCP is the most underrated piece of plumbing in the AI stack"
layout: post
date: 2026-05-25 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - llms
  - mcp
  - infrastructure
  - databricks
category: blog
author: samuelgnap
projects: true
description: Protocol matters more than model, and the thing that defines enterprise AI is the boring connectivity layer, not the next benchmark.
---

The week I finished a Databricks MCP server, my agent stopped being a chatbot and started being a thing that could query the warehouse. Same model. Same prompts. Different category of tool. I had spent the previous six months telling colleagues that LLMs would not change my day-to-day until they could see internal data. That week, they could.

The bridge took about seven days to build. It is the most useful thing I have shipped this year.

## MCP in one paragraph

The Model Context Protocol is a wire format for tool use. An MCP server exposes a set of tools, resources, and prompts. An MCP client (Claude Code, GitHub Copilot CLI, Cursor, whatever ships next quarter) speaks the protocol and calls those tools on the user's behalf. The server is just a small process you run locally or in your VPC. The client does not care what language you wrote it in. That is the whole pitch.

If you have integrated anything with an LLM before, you have probably hand-rolled a function-calling layer. MCP is that layer, standardised, with auth and discovery baked in.

## Why I built one for Databricks

Internal data lives in Databricks. Every model I care about, every booking record, every feature table. A chat assistant that cannot see any of that is a smart intern with no laptop access. It can talk about my work. It cannot help me do it.

The options on the table were:

1. Wait for Databricks to ship a first-party integration with my LLM vendor of choice. Open-ended timeline. Probably tied to one client.
2. Build a vendor-specific plugin for each client I care about. Three integrations on day one. Three more whenever a new client shows up.
3. Build an MCP server once. Point every compliant client at it.

I picked option three on a Monday and had it running against production read replicas by the following Friday. The protocol did most of the heavy lifting. I wrote the Databricks-specific glue (a SQL warehouse connection, the catalog walker, a query validator) and let the MCP SDK handle the wire format, the tool registration, the client handshake.

## The pattern that actually works

There is a version of this where you give the agent unrestricted SQL access and let it discover the schema on the fly. That version is a security incident waiting for a calendar invite. The version I shipped looks like this.

**Read by default.** Every tool the server exposes is read-only unless the user explicitly opts in to writes for a specific session. No `INSERT`, no `UPDATE`, no `MERGE`, no `CREATE TABLE` unless I have flipped a config flag for the session. The agent does not get to argue.

**Schema discovery as a first-class tool.** A `list_catalogs`, `list_schemas`, `describe_table` set of calls the agent uses before it writes any SQL. Most failed queries I saw in week one were the model guessing column names. Schema discovery cuts that to near zero.

**Query validation before execution.** Every query goes through a static check (parse, scan for write operations, scan for cross-catalog joins I have not allow-listed) before it ever hits a warehouse. If validation fails, the agent gets a structured error back and can rewrite. If it passes, it runs against a read replica with a row limit.

**An audit trail nobody asked for.** Every call gets logged with the prompt that triggered it, the SQL that ran, the row count returned. Six weeks in, this is the most important feature. When someone asks "what did the agent do on Tuesday", I can answer.

None of this is novel. The point is that it is mundane. The interesting part of a Databricks MCP server is not the protocol. It is the boring set of safety rails that make a non-deterministic caller safe to point at production data.

## Why this beats per-vendor integrations

The same MCP server, unchanged, serves Claude Code on my laptop, GitHub Copilot CLI on the same laptop, and any other client that learns the protocol. When a new agent ships, I do not write a new connector. The new agent already speaks MCP.

This is the part I keep coming back to. The agent layer is moving fast. Models change every quarter. Clients fork and rename themselves. The half-life of a vendor-specific integration is about six months. The protocol has so far been more durable than any specific client or model that speaks it.

The implication is awkward for vendors and useful for everyone else. If your internal tools speak a protocol rather than a vendor's API, you are not picking a winner. You are renting one. When the next model is better, you point your existing MCP server at the next client. The switching cost is a config change.

Protocol is the moat the model is not.

## The bear case

I should be honest about how this could go sideways.

Vendors do not love protocols they do not control. There is an active risk that one or more major LLM vendors push proprietary extensions that fragment the spec until "MCP-compatible" means three different things. We have seen this movie. It rarely ends well for the protocol.

Read-by-default is not a security guarantee. Read access still leaks PII if the warehouse is not partitioned by sensitivity. Row-level security has to be enforced at the database, not at the MCP layer. I have seen demos where the MCP server became the security layer. It should not be. It is plumbing, not a firewall.

Implementations are uneven. The reference servers are good. Some third-party servers are alarming. A bad MCP server in your config is a bad agent on your laptop. The supply-chain story for MCP servers is not solved.

And the protocol itself could lose. Some other standard could win. Eighteen months from now, this post might read like a vote for SOAP in 2002. That is a real risk and I am taking it on purpose, because the alternative (no protocol, bespoke integrations forever) is worse.

## The bet

In eighteen months, every serious enterprise wiring LLMs to internal systems is using a protocol that looks roughly like MCP. It might be called something else by then. It might have a different sponsor. The shape of the thing (a small server in your VPC that exposes tools to whatever client the user has open) is the part that will hold.

The companies that are spending 2026 picking the right client are optimising for the wrong layer. The companies that are spending 2026 building the connectivity (the schema walkers, the query validators, the audit trails, the small servers that wrap their internal systems) are building something that survives the next model release.

The boring layer wins. It usually does.

Without the bridge, the agent is a smart chatbot. With it, the agent is a colleague.
