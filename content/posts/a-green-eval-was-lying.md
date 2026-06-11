---
title: "A Green Eval Was Lying to Me"
date: 2026-06-10
draft: false
author: "Jack Monte"
description: "Adding strict typing to my eval harness exposed an intermittent judge failure. Fixing that exposed a second bug a passing, zero-error evaluation had been hiding the whole time. A green eval is not a correct eval."
summary: "Adding strict typing to my eval harness exposed an intermittent judge failure, and fixing that exposed a second bug a passing eval had been hiding. The lesson: a green eval is not a correct eval."
tags: ["Evals", "LLM", "RAG", "Anthropic", "Testing", "Python"]
categories: ["Engineering"]
cover:
  image: "images/w7e-green-eval-cover.png"
  alt: "The same eval scored two ways: a buggy run averaging 3.40 and the corrected run averaging 4.95"
  caption: "When a passing eval measures the wrong thing"
  relative: false
images: ["images/w7e-green-eval-cover.png"]
ShowToc: true
TocOpen: false
---

Here is a thing that is easy to believe and dangerous to believe: if your evaluation runs clean, every score in range, zero errors, then your evaluation is correct.

It is not. An eval can run perfectly and measure the wrong thing. I know because mine did, and the only reason I caught it was that I read the judge's reasoning instead of trusting its scores. This is the story of two bugs, found one behind the other.

## Background: how the harness works

`rag-starter` is a retrieval-augmented generation pipeline, and I grade its answers with an LLM-as-judge harness. Three judges score every answer: faithfulness (is the answer grounded in the retrieved context?), relevance (does it address the question?), and precision (did retrieval surface the right chunks?). Each judge is a Claude Haiku call at temperature zero, asked to return a JSON object with a score and a reasoning string.

The harness had been passing for weeks. Clean runs, sensible numbers. I had no reason to think anything was wrong with it.

## Bug one: the judge that wouldn't always answer in JSON

I had just tightened the harness as part of a larger refactor, replacing a hand-rolled JSON parse with strict schema validation. Where the old code did a loose `json.loads` and hoped, the new code validated the judge's output against a typed model and raised if it did not conform.

The next full run, 13 of 40 items failed.

The strictness did not *create* the failures. It *revealed* them. Here is what the judge had been doing, intermittently, the whole time:

![The judge sometimes returns a prose preamble before its JSON, which a strict parser rejects at the first character](/images/p2-failure-mode.svg)

Asked for JSON, the model would sometimes preface it with prose: "I need to evaluate whether the retrieved chunks contain enough information..." and then, eventually, the JSON. The old loose parser had been silently mangling or dropping these. The new strict parser rejected them outright, with an error pointing at character one: it expected the start of a JSON object and found the letter "I".

This is the first lesson, and it is a quiet one: **strictness is not what breaks your pipeline. It is what shows you what was already broken.** The loose parser had been hiding an intermittent reliability problem. Temperature zero is not fully deterministic, so the same prompt yielded clean JSON on some runs and a chatty preamble on others. My earlier "passing" runs had simply drawn lucky.

The fix was structured outputs. Instead of asking the model nicely for JSON and parsing whatever came back, I used the API's constrained decoding: the response is forced to conform to a schema at the token level, so schema-valid JSON is the only output the model can physically produce. No preamble is possible.

Parse failures after the change: zero of forty. Reliability problem solved.

## The trap: a re-baseline that looked like a finding

With the judge now reliable, I re-ran the full eval to establish a clean baseline. Faithfulness came back at 3.40, well below the project's earlier reading of nearly 5.

I had a tidy story ready for that drop. "The new constrained judge is scoring more decisively," I told myself. "The old free-text judge was inflating scores; this is the more honest number." It is a plausible story. It is the kind of story that sounds like insight. And I very nearly wrote it down as the new baseline and moved on.

Then I did the one thing that saved me: I opened the results file and read what the judge actually *said* about a few of the low-scored items.

## Bug two: the eval was measuring the wrong answer

The reasoning did not match the scores. The judge was carefully evaluating text that was not the answer my system had generated. It was reasoning about the *expected* answer, the golden reference, instead of the model's actual output.

I went to the faithfulness scorer and found it. Earlier in the refactor, I had rewrapped some long prompt strings to satisfy a line-length linter. In the rewrap, one function ended up passing `expected_answer` to the judge where it should have passed `generated_answer`. So the faithfulness judge had been asking "is the golden answer grounded in the retrieved chunks?" instead of "is the model's answer grounded?" The golden answers are terse reference text; many of them are not verbatim grounded in the retrieved chunks, so the judge correctly scored *them* low. The 3.40 was real, precise, and completely meaningless.

The fix was one word: `generated_answer`. Faithfulness returned to 4.95.

Here is what that one word did to the distribution:

![The same forty items scored two ways: the buggy run averaging 3.40 with seventeen items dragged to 1 or 2, and the corrected run averaging 4.95 with thirty-eight of forty scoring 5](/images/p2-two-distributions.svg)

Same eval. Same pipeline. Same forty items. One variable swapped. Both runs passed with zero errors and every score in range. The only thing that distinguished the correct run from the wrong one was reading the reasoning.

## The whole arc

![The investigation, step by step: a typed boundary surfaces the preamble failures, structured outputs fixes reliability, the re-baseline reads a misleadingly low number, reading the reasoning reveals the wrong variable, and a one-word fix restores the real baseline](/images/p2-investigation.svg)

Notice the shape of it. The first bug was found *because* I added strictness. The second bug was found *despite* the eval being green, and only because I refused to accept a plausible number at face value. Strictness is a tool you can add to your harness. Skepticism is a habit you have to bring yourself.

## What I take from this

A few things I now believe more firmly than I did:

A passing eval is necessary, not sufficient. Zero errors and in-range scores tell you the machinery ran. They tell you nothing about whether the machinery measured what you intended.

Read the reasoning, not just the score. Every LLM-as-judge setup should make the judge explain itself, and you should actually read those explanations on a sample of items, especially when a number surprises you. The explanation is where a wrong measurement gives itself away. A bare score cannot.

Be most suspicious when the number tells a good story. "The honest new judge scores lower" was a *satisfying* explanation, and that is exactly why it was dangerous. A surprising result that flatters your narrative deserves more scrutiny than a boring one, not less.

The same tightening that exposes a reliability bug can introduce a new one. Adding strict typing surfaced the preamble failures and, in the same refactor, the line-length cleanup introduced the variable swap. Improvements are also changes, and changes carry their own bugs.

The corrected baseline, the real one, is faithfulness 4.95, relevance 3.45, precision 0.45 across forty items with zero errors. It is close to where the project started, which is the honest outcome: the structured-output migration bought reliability, not a score change, and the scary-looking drop was a bug I almost shipped as a finding.

The full engineering write-up, including the structured-outputs migration and the honesty log for the scorer bug, is in `patterns-applied.md` in the repo.
