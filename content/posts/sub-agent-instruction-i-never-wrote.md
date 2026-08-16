---
title: "My Sub-Agent Followed an Instruction I Never Wrote"
date: 2026-08-15T18:00:00-07:00
draft: false
author: "Jack Monte"
description: "Sub-agents are sold on context isolation, and that part holds up. Less obvious is that three separate inputs cross into a sub-agent's context, and one of them is composed by a model on your behalf and never appears in your transcript. Here is which input won, and what delegation actually costs in tokens."
summary: "Three inputs cross into a sub-agent's context window. You write two of them. The third is generated on your behalf, never shown to you, and it is the one that overrode a strict output format. Includes the token cost of delegating, measured."
tags: ["LLM", "Agents", "Claude Code", "Context Engineering", "Python"]
categories: ["Engineering"]
cover:
  image: "images/sub-agent-instruction-i-never-wrote-cover.png"
  alt: "What reaches a sub-agent: the agent body and project rules you wrote, plus a task prompt generated on your behalf"
  caption: "Delegating a task to a sub-agent"
  relative: false
images: ["images/sub-agent-instruction-i-never-wrote-cover.png"]
ShowToc: true
TocOpen: false
---

I gave a read-only auditor four Python scripts and a strict output format: five fields per finding, an ordering rule, and an instruction to say so plainly when nothing turns up.

It found ten real defects. It used none of the formatting I specified.

Nothing errored. The findings were good, several of them bugs that had sat unnoticed for weeks. The shape was just not the one in the definition.

The obvious read is that the model ignored its instructions. That read was wrong, and what was actually going on is worth knowing before you delegate anything that matters.

## Terms

- **Sub-agent.** A delegated task that runs in its own context window, with its own system prompt and its own tool allowlist, returning a single result to the session that spawned it.
- **Agent definition.** The file you write. Frontmatter for name, description, model, and tools, plus a body that becomes the sub-agent's system prompt.
- **Tool allowlist.** The `tools` field. What the sub-agent may call. Anything absent from it is not discouraged, it is unavailable.
- **Project instruction file.** Repo-level rules loaded into context automatically. Custom sub-agents receive these.
- **Task prompt.** The instruction the orchestrating model composes and hands to the sub-agent at delegation time. Not written by you, and not visible in your transcript. This one is the whole point.

## The setup

The auditor's job was narrow: read a small internal toolkit of standard-library Python scripts and flag correctness defects, meaning logic that does not do what its name or docstring claims. Read-only by design, because an auditor that can edit quietly becomes a refactorer.

```markdown
---
name: scripts-auditor
description: Audits Python scripts for correctness defects, logic that does
  not do what its name or docstring claims.
model: opus
tools: Read, Grep, Glob
---

You are a correctness auditor for Python code.

## Output format

For each finding:

**File:** path
**Function:** name
**Evidence:** the specific lines, quoted
**Why it is wrong:** what input produces the wrong result, and what the
correct result would be
**Confidence:** certain, or dependent on an assumption about intent
```

What came back was a structured review panel with severity dots and category tags. Good content, wrong shape, no Confidence field anywhere.

## What actually crossed the boundary

**Short version: the format was overridden by an instruction I never wrote and never saw.**

Finding that out cost one invocation. Ask the agent, not the transcript:

> Report, without auditing anything: which skills you loaded during your last run if any, whether your system prompt specified an output format, and why your report did not use that format.

Three answers came back:

- **Skills loaded:** none.
- **Format specified:** yes, and it recited all five fields back accurately.
- **Why the deviation:** the task prompt had told it to use a `ReportFindings` tool. That tool was not in its allowlist. Rather than flag the mismatch, it silently substituted a different delivery method.

That instruction appears nowhere in the definition and nowhere in the one sentence I typed to start the run.

Here is why. When you delegate through a chat interface, the sub-agent never receives your words. The orchestrating model writes a task prompt on your behalf, and that prompt lands in the sub-agent's context alongside everything you did write. It does not land in your transcript.

Self-report is weak evidence on its own, so it matters that the main session backs this up independently: two `Invalid tool parameters` errors and a note about dropping an optional field, both appearing after the agent block had already closed. The parent was fumbling a call to a tool the sub-agent never had.

![What reaches a sub-agent: the agent definition body and the project instruction file, both written by you, plus a model-generated task prompt that never appears in your transcript. Your conversation does not cross. Only the finished report returns, at 21k tokens.](/images/sub-agent-boundary.svg)

**Not everything is invisible, and that is the part worth sitting with.** The project instruction file crosses too. I confirmed it with a second run, planting an instruction in the agent's body that directly contradicted a project rule. The project rule won.

So the boundary is not a wall. It is a filter with several inputs passing through it, and the one that beat my output format was the only input I could not see.

## The isolation is real, and it is not free

**Delegation moves context cost from process to product. It does not remove it.**

- **Definition cost: 67 tokens.** Appending an entire new section to the body left that number unchanged. Only the description loads, for routing. The body goes to the sub-agent and never to you.
- **Main context: 31.1k to 52k tokens.** The message portion went from 82 to 21k.
- **What stayed out:** four file reads and roughly two and a half minutes of intermediate reasoning.
- **What came in:** 21,000 tokens of finished report.

So the isolation claim holds, but a verbose sub-agent is still verbose. You skip paying for its work and you still pay for its conclusions.

**One caveat on the findings themselves.** The same audit against the same directory returned ten defects on the first pass and nine on the second, with different items in each. A tooling update landed between the runs, so I cannot attribute that delta cleanly. Either way, a single audit pass is a sample, not a coverage claim, and not something to hand a client as though it were exhaustive.

## Worth checking on your own setup

- **Ask a sub-agent what it was told**, not only what it found. One invocation, and the answer is often not the one you assume.
- **Watch for capability mismatches.** An agent told to use a tool it does not have will usually route around the gap instead of announcing it.
- **Measure the return, not the run.** The cost that matters is the size of what comes back, and the isolation story trains you to watch what stays out.

## The takeaway

The risk everyone designs around is the context a sub-agent lacks. That one is real and well understood, and it is why definitions get written carefully in the first place.

The risk that actually bit me runs the other direction: the context a sub-agent gains on my behalf. Written by a model, never surfaced to me, and treated as authoritative by something that then quietly works around whichever parts of it turn out to be impossible.

So the rule I am carrying forward is a small one. When a delegated result looks fine but off, do not start by asking the agent what it found. Ask what it was told.
