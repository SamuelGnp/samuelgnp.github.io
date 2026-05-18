---
title: "Stop building RAG. You probably need search."
layout: post
date: 2026-06-22 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - llms
  - rag
  - search
  - infrastructure
category: blog
author: samuelgnap
description: Most projects sold as RAG are search projects in disguise, and the perceived win comes from finally building a decent index over internal data.
---

A while back I built a RAG system over a few years of my own Telegram chats. LangChain, ChromaDB, OpenAI embeddings, the full stack. The thing worked well enough that I caught myself bragging about it. Then I sat down to look at the failure cases and the wins side by side, and noticed something uncomfortable. Every time the system was useful, the retrieval had done the work. Every time it failed, the retrieval had failed first. The LLM was reading what I had already found and saying it back to me in cleaner sentences.

That is the experience that made me stop trusting most RAG projects. Not because the architecture is wrong. Because the architecture is doing more bookkeeping than thinking.

## The pitch sells you the generator. The retriever does the work.

The standard RAG pitch goes like this. Embed your documents. Stick the vectors in a database. At query time, embed the question, pull the top-k matching chunks, drop them into the prompt, let the LLM answer. The diagram has an arrow pointing at the LLM and that is where the eye lands. The implicit story is that the model is doing the heavy lifting and the vector database is a clever new way to feed it context.

It is the wrong way round. The retriever decides what the model sees. If the right passage is in the top-k, the answer is going to be roughly correct regardless of which frontier model you use. If the right passage is not in the top-k, no amount of prompt engineering recovers it. The model cannot reason about evidence it never received. So the system performance is bounded above by the retrieval quality, and below by the worst output the model produces given perfect retrieval. The interesting work lives in the bound, not in the model.

Run the obvious ablation on your own system. Take the retrieved chunks, drop the LLM, render them as quoted passages with a one-line header. Compare user satisfaction. On every internal "ask the docs" tool I have looked at, the quoted-passages baseline keeps something like 70 to 80 percent of the perceived utility, and people complain less about hallucinations. The LLM is doing reformatting. Reformatting is nice. It is not the product.

## A lot of what gets shipped as RAG is search with a coat of paint.

The clearest case is the internal knowledge base bot. Someone wires up Confluence or Notion or a SharePoint dump, embeds every page, stands up a chat interface. Two years ago this would have been called search. The team would have invested in a good index, decent tokenisation, faceted filters, and a results page that highlighted matching snippets. The whole project would have shipped in a quarter and people would have used it.

The same project today gets called RAG, includes a vector database, ships in three quarters, costs more to run, and produces answers that are harder to verify. The retrieval problem is identical. Documents in, ranked chunks out. The thing that changed is the surface. A chat box implies you can ask anything in natural language. A search box admits its limits. Users forgive a bad result from a search box. They feel lied to by a confident wrong answer from a chat box.

I am not saying every chat-over-docs tool is worse than a search box. I am saying the question to ask before building one is, would search alone have been enough. For internal documentation, customer-facing FAQs, code search, and most knowledge-base lookups, the answer is yes. Search alone would have been enough. The reason these projects feel transformative is that the organisation finally invested in an index over data that was previously unindexed. The transformation came from the index. The LLM rode along.

## When the LLM actually earns its place.

There is a real case for putting a model in the loop. It is narrower than the pitch suggests. Three patterns where I have seen it pay.

Synthesis across chunks. The answer is not in any single passage. It is in the combination of three. The model reads all three and produces a coherent answer that no individual chunk contains. This is the case that motivated RAG in the first place and it is genuinely useful. It is also rarer than people think. Most user questions have an answer in one chunk if the retriever is any good.

Reformatting under a constraint. A passage from a technical document is correct but unreadable for the target audience. The model rewrites it in the voice of customer support, or strips the jargon for a sales context, or formats it as a checklist. This is style transfer over retrieved content. It is real work the model does well.

Question rewriting for retrieval. The user types something the index cannot match. The model rewrites the query into a form the index can match, sometimes multiple variants, and the system runs each. This one is worth highlighting because it is RAG using an LLM to improve the retrieval, not to replace it. That is the right way round.

If your use case does not look like one of these, the LLM is probably a tax you are paying on a search problem.

## What I tell people who want to build RAG.

Invest in retrieval first. Get the boring stuff right before you get clever. Decide whether you actually need a vector index, or whether BM25 plus some metadata filters does the job. For a lot of internal data, lexical search with good preprocessing beats embeddings on accuracy and is two orders of magnitude cheaper. Hybrid retrieval helps. Pure dense retrieval over messy documents is rarely the right starting point.

Measure retrieval directly. Build a small labelled set of question to expected passage pairs and report recall at k. If recall at five is under 70 percent, the system is broken regardless of which model is doing the generation. Fix that before tuning prompts.

Add the LLM at the end if it adds value. Ship the quoted-passages baseline first. If users keep asking for synthesis or reformatting, add the model. Many "RAG" projects that fail were really retrieval projects that did not invest in retrieval, then blamed the model.

The interesting work in 2026 is not RAG. It is making search work on internal data. The LLM is a bonus and sometimes it is not even that.

The Telegram project I started with still runs on my laptop. I still find it useful. I also rebuilt the retrieval recently with BM25 over the same corpus, kept the embedding side as a fallback, and removed the LLM from the default path. It is faster, cheaper, and the answers are easier to verify. If you replace the LLM with quoted passages, you keep most of the utility. That is the test I run on every RAG project I see now, and most of them do not survive it.
