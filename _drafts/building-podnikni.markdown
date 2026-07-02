---
title: "I built a Stripe Atlas for Slovakia, then parked it"
layout: post
date: 2026-07-02 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - llms
  - product
  - saas
category: blog
---

<!-- DRAFT — review voice, add screenshots where marked, then move to _posts with a real date. -->

Starting a limited company in Slovakia — an s.r.o. — is a bureaucratic obstacle course in a language most foreign founders don't speak. Lawyers charge hundreds of euros to fill in templates. Stripe Atlas solved this for US Delaware companies a decade ago. Nobody had built it for Slovakia.

So I did. Podnikni is a full formation platform: a multi-step wizard collects the founder's details, Stripe takes payment, Claude generates the founding documents (with Opus on the legal text and Sonnet classifying business activities into NACE codes), and a post-formation dashboard tracks compliance deadlines. English-first, bilingual, deployed on Vercel.

<!-- SCREENSHOT: wizard step -->

## What the build actually taught me

**The AI was the easy 20%.** Generating a legally plausible founding deed with Claude took days. The other months went to everything around it: auth, i18n, Stripe webhooks, PDF generation with Slovak diacritics, email sequences, and modelling the actual legal workflow — what happens after payment, who signs what, which registry needs which form. If you're evaluating an "AI product" idea, price the non-AI 80% first.

**Structured legal workflows are a great LLM fit — with guardrails.** Founding documents are templated prose with strict invariants. The model fills a schema, not a blank page; deterministic code owns names, amounts, and dates. This is the same lesson I keep re-learning in my day job: tool calls and structured output for facts, generation for language.

<!-- SCREENSHOT: generated document / dashboard -->

**Distribution is the moat, and I didn't have one.** The product worked. But formation is a once-per-lifetime purchase discovered via Google in Slovak, competing against incumbent law-firm SEO built up over a decade. A solo builder with a full-time job doesn't win that fight on nights and weekends.

So it's parked, deliberately: 246 commits, a working product, and a much sharper filter for what I build next. The engineering lessons went straight into my current projects.

## Stack

SvelteKit 2 / Svelte 5, TailwindCSS v4, Better Auth, Neon Postgres + Drizzle, Paraglide i18n, Stripe, Resend, Claude API, PDFKit, Vercel.
