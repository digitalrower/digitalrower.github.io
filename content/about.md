---
title: "About"
summary: "AI engineer building production LLM and RAG systems for B2B SaaS and logistics."
showtoc: false
ShowReadingTime: false
---

I'm Jack Monte, an AI engineer building production LLM systems for B2B SaaS and 3PL/logistics companies.

My focus areas are retrieval-augmented generation (RAG), document and invoice automation, and AI agents. The common thread across all of them is a bias toward systems that are measured, not just demoed. Every project I ship carries an auditable evaluation dataset and published benchmarks, because in production the difference between "looks impressive" and "works reliably" is the whole job.

## How I work

**Evals first.** Before a pipeline changes, it gets a baseline. Quality metrics (faithfulness, relevance, retrieval precision) are tracked separately from reliability metrics (error rate), so a regression in one never hides behind the other.

**Observability built in.** Every query is traced end to end with Langfuse, with scores attached to traces. When something degrades, the answer to "what changed" is in the data, not in guesswork.

**Production patterns from day one.** Typed data boundaries, centralized API clients with configured retries, and explicit error hierarchies. The goal is code a client's team can read, extend, and trust after the engagement ends.

## Stack

Python, FastAPI, ChromaDB and pgvector, Anthropic API, Langfuse. Development with Claude Code and full CI (ruff, mypy) on every project.

## Proof

My open-source work and benchmark write-ups live on [GitHub](https://github.com/digitalrower). The [Projects](/projects/) page covers each build in detail.

## Contact

The fastest way to reach me is [LinkedIn](https://www.linkedin.com/in/jack-monte/). <!-- TODO: add your business email here when ready, e.g. "or email me at hello@jackmonte.com" -->
