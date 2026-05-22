---
title: "I Built an Eval Harness for My RAG Pipeline. Here's What the Numbers Revealed."
date: 2026-05-21
draft: false
---

Most RAG pipelines ship without any systematic way to measure whether they're actually working. You run a few manual queries, the answers look reasonable, and you move on. The problem is that "looks reasonable" doesn't tell you where the system is failing, how often it fails, or whether it would fail on the queries your users actually send.

That was the state of my rag-starter pipeline at the end of Week 4. It could answer questions grounded in a document corpus, and it correctly said "I don't know" when context was insufficient. But I had no numbers. I didn't know if the retrieval was finding the right chunks, whether Claude was staying faithful to the context, or whether the answers were actually addressing the questions asked.

This week I built an automated eval harness to find out. What I found was more specific than I expected, and the specificity is the point.

---

## The Problem With Manual Testing

Manual testing of RAG pipelines has two failure modes. The first is coverage: you test the queries you think will work and miss the ones that won't. The second is memory: you can't compare last week's results to this week's because you didn't record anything. Both problems compound as the system grows.

What you actually need is a fixed dataset of questions with known answers, a repeatable scoring mechanism, and a results table you can track over time.

---

## What I Built

The harness has three components: a dataset of 40 golden question-answer pairs, three automated scorers, and a runner that ties them together.

**The dataset** covers four categories of queries: happy path (questions the corpus should answer easily), edge cases (ambiguous or partial-match queries), adversarial inputs (queries designed to test whether the system hallucinates), and bias-paired cases (the same question phrased with different contextual framing to check for quality variation).

**The three scorers** each target a different failure mode:

- *Faithfulness* measures whether the generated answer stays within the retrieved context. A faithfulness failure means Claude made something up. Scored 1-5 by Claude Haiku acting as a judge, at temperature 0 for deterministic results.
- *Answer relevance* measures whether the answer actually addresses the question asked. A relevance failure means the system retrieved something and Claude generated something, but the output doesn't help the user. Also scored 1-5.
- *Precision@3* measures whether the top-3 retrieved chunks contain enough information to answer the question. This is a retrieval-only metric that tells you if the problem is upstream of generation entirely. Scored 0 or 1.

Using Claude Haiku as judge keeps scoring costs low (fractions of a cent per query) while producing consistent, reasoning-backed scores. Temperature 0 means running the same dataset twice produces the same results.

**The runner** loops through the dataset, calls the live pipeline for each question, scores the output with all three scorers, and writes a structured results file. Running the full eval takes a few minutes and produces a summary table.

---

## What the Numbers Showed

| Category | Avg Faithfulness | Avg Relevance | Precision@3 | Count |
|---|---|---|---|---|
| happy_path | 5.00 | 3.69 | 0.75 | 16 |
| edge_case | 4.80 | 2.90 | 0.30 | 10 |
| adversarial | 5.00 | 5.00 | 0.00 | 4 |
| bias_paired | 5.00 | 3.40 | 0.60 | 10 |
| **OVERALL** | **4.95** | **3.55** | **0.53** | **40** |

The headline number is faithfulness at 4.95 out of 5. Claude is not hallucinating. When the pipeline retrieves chunks, the generated answers stay within that context. That's the grounding constraint working as designed.

The more interesting number is precision@3 at 0.53 overall, and the breakdown by category tells the real story. Happy path queries hit 0.75, which is acceptable for a first implementation. Edge cases drop to 0.30. Adversarial queries hit 0.00.

That adversarial precision@3 of 0.00 initially looked alarming. Looking at the actual outputs though, all four adversarial cases returned "I don't know based on the provided context" with a relevance score of 5.00. The system correctly refused to answer rather than retrieve irrelevant chunks and hallucinate. The grounding constraint handled exactly the scenario it was designed for.

The edge case results are the genuine gap. Precision@3 of 0.30 means the retrieval is finding the wrong chunks for ambiguous queries 70% of the time. When retrieval misses, no generation quality can compensate, which is why edge case relevance scores land at 2.90.

The bias check compared 5 paired question sets where the same factual question was phrased with different contextual framing: US versus EU regulatory context, personal hobby project versus commercial production deployment, and similar variations. Minor relevance variation showed up on 2 of 5 pairs (a difference of 1 point on a 5-point scale). The variation traces to corpus coverage rather than model behavior. The corpus contains less EU regulatory content and less production-deployment context than it does US and personal-project content. No model bias was identified.

---

## What This Means

The results give a precise diagnosis: **generation is solved, retrieval is not.** Faithfulness near-perfect means the grounding architecture is sound. Precision@3 at 0.30 on edge cases means the embedding-based retrieval is failing on non-obvious queries.

That distinction matters because the fix paths are completely different. A generation problem gets addressed through prompt engineering, model selection, or context formatting. A retrieval problem gets addressed through chunking strategy, embedding model choice, query expansion, or reranking. Knowing which problem you have before you start optimizing saves a lot of time chasing the wrong thing.

The retrieval gap will be addressed when this pipeline gets adapted for Project 1 (Docs Copilot) in a few weeks. The eval harness runs against that version too, same metrics and same categories, so the comparison is direct.

---

## What's Next

Next week is observability. The eval harness tells me output quality in aggregate. What it doesn't tell me is what's happening on individual requests in production: which queries are slow, which consume the most tokens, whether latency is consistent or spiky.

I'll be integrating Langfuse, an open-source LLM observability platform, to add structured tracing to every request. The goal is a dashboard showing p50 and p95 latency, token cost per query, and per-request traces that link a user question to the specific chunks retrieved and the Claude call that generated the answer.

---

Eval is the single clearest signal separating production AI engineering from demo AI engineering. Anyone can build a RAG pipeline that produces plausible-looking outputs on curated test queries. Far fewer can tell you, with actual numbers, what the precision@3 is on edge cases and why. The harness is just infrastructure. The specificity it produces is the thing that matters.
