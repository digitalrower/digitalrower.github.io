---
title: "What I Learned Reading a Production SDK Cover to Cover"
date: 2026-06-10
draft: false
author: "Jack Monte"
description: "I made my RAG project production-ready by reading the Anthropic Python SDK end to end and stealing four patterns from it. Here is each one, why it mattered, and the lesson about reading source instead of docs."
summary: "I made my RAG project production-ready by reading the Anthropic Python SDK end to end and stealing four patterns from it. Here is what each one was and why it mattered."
tags: ["Python", "RAG", "Anthropic", "Pydantic", "Engineering", "LLM"]
categories: ["Engineering"]
cover:
  image: "images/w7e-sdk-patterns-cover.png"
  alt: "Four patterns read out of a production SDK and applied to a RAG project"
  caption: "Reading the Anthropic Python SDK to level up rag-starter"
  relative: false
images: ["images/w7e-sdk-patterns-cover.png"]
ShowToc: true
TocOpen: false
---

Most advice on writing production-grade Python tells you the principles: validate at boundaries, type your interfaces, handle errors as data. The principles are easy to nod along to and hard to apply, because a principle does not tell you *where* in your own messy code it belongs. What does tell you is reading a real production codebase that already got it right, and asking what it does that yours does not.

So I spent a couple days reading the [Anthropic Python SDK](https://github.com/anthropics/anthropic-sdk-python) cover to cover, the actual source, not the docs, and pulled four concrete patterns out of it into my own RAG project, `rag-starter`. This is what they were.

![A map of the four patterns and where each one landed in the rag-starter codebase](/images/p1-patterns-map.svg)

## Why read source instead of docs

Docs tell you how to *use* a library. Source tells you how its authors *think*, and those are different lessons. The SDK's docs would never tell me "here is how to structure error classes" because that is internal to the library. But the structure is right there in `_exceptions.py`, and it is a better answer than anything I would have invented. Reading source is how you absorb judgment, not just API surface.

A warning that paid off immediately: I verified every line reference against the cloned source at a pinned version rather than trusting my memory or a summary. Line numbers drift between versions, and more than once my recollection of "how the SDK does X" was simply wrong. Grounding claims in the actual file caught those before they became mistakes in my own code.

## Pattern 1: a client factory, and a lesson in not building what already exists

My code constructed the API client inline in four different places, each picking up default settings, none of them tunable. The obvious fix is a single factory function. The interesting part was what I learned *not* to build.

I had assumed making the client production-ready meant writing retry logic: exponential backoff, jitter, parsing the `Retry-After` header, deciding which status codes are worth retrying. Then I read `_base_client.py` and found all of it already implemented, carefully, by people who do this for a living.

![What I almost built versus what I actually wrote: the SDK already provides backoff](/images/p1-retry-reframe.svg)

The pattern was not "build a retry loop." It was "configure and centralize the one the SDK already gives you." My factory ended up being about ten lines: set an explicit timeout and retry count, return a configured client. The hard part was done. The skill was recognizing that, which only came from reading the source instead of reaching for a tutorial on retry loops.

This is the most transferable lesson of the four. A lot of "make it production-ready" work is not adding machinery. It is finding the machinery that already exists and stopping yourself from reinventing it worse.

## Pattern 2: typed boundaries, so wrong shapes fail loudly

Data moved between the layers of my pipeline as plain dictionaries: a chunk was `{"text": ..., "source": ...}`, a score was `{"score": ..., "reasoning": ...}`, an eval item was another bag of strings. This works right up until it doesn't. A typo in a key, a missing field, a drifted shape, all of it flows silently through several layers and fails somewhere far from where the data actually went wrong, if it fails at all.

The SDK's response types are all Pydantic models. That is why a call returns an object you can write `message.usage.input_tokens` against instead of digging through a dict. I did the same thing to my own boundaries.

![Before and after: untyped dicts with subscript access become validated Pydantic models with attribute access](/images/p1-before-after.svg)

One detail mattered more than I expected. The SDK sets its models to *allow* unknown fields, deliberately, so that a server adding a new field never breaks an older client. That is correct for a library that has to stay forward-compatible with an API it does not control. My situation is the opposite: my boundaries are internal, and an unexpected field almost always means a bug, not a feature. So I set mine to *forbid* extra fields. Reading the SDK taught me the pattern and, just as usefully, taught me where my needs differed from theirs.

This is the part that earns its keep later. The retrieval layer now returns a typed `Chunk` no matter what storage sits behind it, so when I swap the vector store down the line, that change stays behind a stable contract instead of rippling through every file that touched a chunk dict.

It also caught a real bug the first time I ran it. The strict models rejected my eval dataset because it contained fields my model did not declare, a genuine, previously silent mismatch between the data and the code's assumptions about it. The strictness surfaced it immediately and forced me to decide about each field on purpose.

## Pattern 3: errors as typed values, not strings

My answer-generation function used to catch an API error and return the error message *as the answer*. Read that again, because I did not notice how bad it was until I wrote it down: a failure was being handed back to every caller dressed up as a successful result. Downstream, my eval harness would then dutifully score the error string as if it were a real answer.

The SDK's exception hierarchy is a small tree: one base error, then categorical subclasses, each carrying structured context rather than just a message. I built the same shape for my project: a `RAGError` base with categories for retrieval, generation, and scoring failures. Now a failure raises instead of masquerading, and the type tells the caller what kind of failure it was.

The payoff showed up in the eval runner. Wrapping each item in a handler for the error base means one bad item gets recorded as failed and the run continues, instead of one transient hiccup killing a forty-item evaluation. Reliability failures became visible and isolatable instead of fatal or, worse, silent.

## Pattern 4: the one that found a hidden bug

The fourth pattern, moving my eval judges to structured outputs, deserves its own post, because applying it surfaced a reliability problem that had been hiding in the project the whole time, and then a second bug hiding behind that one. That story is [coming next](/posts/a-green-eval-was-lying/). For here, the short version: the typed parse boundary from Pattern 3 made a class of intermittent judge failures visible, and constrained decoding fixed them.

## What actually transferred

The specific patterns matter, but the meta-lesson is the one I will keep using: when you want to level up the quality of your own code, find a production codebase in your domain and read it like a book. Not the docs. The source. You absorb a hundred small decisions about structure, naming, error handling, and typing that no tutorial bundles together, because in a tutorial they would be noise and in real code they are the whole point.

Four patterns, a couple days of reading, and a project that fails loudly, validates at its edges, and stopped reinventing a retry loop that already existed. The full write-up of each pattern, with the eval numbers before and after, lives in `patterns-applied.md` in the repo.
