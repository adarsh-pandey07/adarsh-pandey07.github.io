---
layout: post
title: "What We Learned Researching Coding Harnesses"
subtitle: "The model is not the whole story."
description: "We read the source code of 12 AI coding agents. The core loop fits in a few dozen lines; everything that separates a good agent from a frustrating one lives around it."
---

**We read the source code of 12 AI coding agents. The core loop that makes them work fits in a few dozen lines. Everything that separates a good agent from a frustrating one lives around that loop.**

I wanted to build my own minimal coding agent. Before writing a line, I wanted to know how the existing ones actually work: not from launch posts, but from their code.

So we cloned eleven open-source agents, read the docs and real session logs of a twelfth (Claude Code), and traced one request through each of them: *"add a failing test for X, then fix it."* This article is what we found.

If you use Claude Code, Codex, Aider or opencode every day, this is a look under the hood. If you want to build your own, it's a map of the decisions you'll face, and which agent made each one best.

## What is a "harness", anyway?

A **harness** is everything around the model. The model only predicts text. The harness turns that into an agent that reads files, runs commands and edits code.

Think of the model as an engine and the harness as the rest of the car: the steering, the brakes, the dashboard and the fuel gauge. Every harness we studied is built from the same parts:

- **The loop.** Send the conversation to the model, run any tools it asks for, add the results, repeat.
- **The prompt.** The standing instructions the model sees on every request.
- **The tools.** Read, write, edit, run shell commands, search.
- **Context management.** What stays in the conversation, and what gets trimmed or summarised when it grows too big.
- **Safety.** Which actions run freely, which ask you first, and which are blocked.
- **Extensions.** Hooks, skills, sub-agents and MCP servers that add behaviour without touching the core.

The diagram below shows how those parts connect.

![Anatomy of a coding harness](/assets/blog/harness-anatomy.png)

The loop sits in the middle and stays simple. Each request is rebuilt from the event log, and every rule (permissions, context trimming, instructions, plugins) hooks in from outside.

## How we did the research

We set one rule: **every claim must point to a file and a function in the source, or to an official doc.** Anything we couldn't see directly is marked as inferred.

| | |
| --- | --- |
| **Agents studied** | Claude Code, Codex CLI, Pi, Oh My Pi, opencode, DeepSeek Harness, Exo, Mini-SWE-Agent, Goose, Aider, OpenHands, Hermes Agent |
| **Evidence** | Shallow clones at pinned commits; docs and session-log structure for closed-source Claude Code (no leaked code) |
| **The trace** | One request followed end to end: entry point, prompt, model call, tool call, result, next turn, stop |
| **Cost pass** | Tool-schema sizes, cache design, public benchmarks and current provider prices |
| **When** | 28 September to 2 October 2026 |

We used AI agents to do the reading in parallel, one per harness. Each wrote a detailed report, and we compared them on eleven dimensions, from the loop to the edit format to the bill.

One honest limit: we read code, we didn't run benchmarks. Prompt sizes and line counts are estimates (within about 30%), and the only same-model comparison we found is small.

## Eight things the code taught us

### 1. The loop is tiny, and almost everyone agrees on it

Every serious harness runs the same loop: call the model, run the tools it asks for, append the results, go again. In 10 of the 12, **a reply with no tool calls means "I'm done."**

How small can it be? Mini-SWE-Agent does it in **37 lines with a single tool, bash**, and its README claims over 74% on SWE-bench Verified. Pi's entire minimal core is about 4,450 lines.

The lesson: the loop is not where the magic is. A capable model plus a simple loop already goes a long way.

### 2. Rules belong around the loop, not inside it

The cleanest designs (Pi and DeepSeek Harness) keep the loop free of policy. Instead they expose a few **hook points**: before a request, before and after a tool call, and before stopping.

Permissions, retries, compaction and project instructions all plug into those hooks. The loop stays small and testable, and every feature can be switched on or off without touching it.

### 3. Keep a log, and build the conversation from it

OpenHands, DeepSeek Harness, Exo, Pi and Claude Code all store every step as an **append-only event log**. The request sent to the model is *derived* from that log, never edited in place.

That one choice pays off five ways: you can resume a session, fork it, replay it, stream it to a UI, and test it. It also keeps the prompt cache intact, which matters for the next point.

### 4. Caching matters more than which harness you pick

This was the biggest surprise. For the same 20-turn task, **turning on prompt caching cuts the bill 3.5 to 4.5 times**. Claude Code showed a 98.5% cache-hit rate across 30 real sessions we looked at.

Once cached, a big system prompt barely matters. Pi's ~1.35k-token starting prompt cost about $0.37 for the task; Claude Code's ~34k cost about $0.58. What really moves cost is how many turns the agent takes.

Caches are fragile, though. We found real cache-busters in shipping code:

- **A date in the system prompt** (opencode), which invalidates the cache every day.
- **A 7.4k-character Slack guide sent on every request**, placed before the history (Exo).
- **Helper tasks quietly using the most expensive model**, instead of a cheap one (Hermes, Pi, Oh My Pi, opencode).

### 5. Editing files is where agents break

Most agent failures happen at the edit step: the model's idea of the file doesn't match the file. The field has converged on a robust recipe:

1. **Exact search-and-replace**, and the match must be unique.
2. **A fallback ladder** if it doesn't match: ignore trailing spaces, then surrounding spaces, then smart quotes and other Unicode punctuation (Codex).
3. **Errors the model can act on**: show the closest real lines, or the line numbers of each ambiguous match.
4. **Stale-read protection**: refuse an edit to a file the model hasn't read, or that changed since (Claude Code, Oh My Pi).

Tool counts vary wildly, from **1** (Mini-SWE-Agent) to **4** (Pi: read, write, edit, bash) to **about 46** (Claude Code). Aider uses **zero**: the model writes edit blocks as plain text and the harness parses them.

### 6. Trim old output cheaply before you summarise

Conversations grow until they overflow the model's context window. Most harnesses summarise the old part when they get close (Codex at 90%, Goose and DeepSeek Harness at 80%).

The cheapest approach in the research we found is two-tier: first **shorten old tool output mechanically**, then summarise with the model only if you're still too big. In one study, shortening old tool results raised the share of fixed tests from 28% to 49% in a small window.

Two harnesses, Exo and Mini-SWE-Agent, never compact at all. Their cost grows quadratically with the number of steps.

### 7. Safety is mostly off by default

We found three schools of safety:

- **Sandbox first:** the OS restricts what commands can touch (Codex, DeepSeek Harness).
- **Rules first:** allow, ask or deny lists per tool (Claude Code, opencode, Goose, Hermes).
- **"Run me in a container":** the harness trusts the box it runs in (Pi, Exo, Mini-SWE-Agent).

The surprise: **five harnesses approve everything by default** (Oh My Pi, OpenHands, Goose, opencode and Exo). Only Codex, DeepSeek Harness and Claude Code ship with guards on.

### 8. Read the code, not the README

Several marketed features turned out different, or missing, in the source:

| What's claimed | What the code shows |
| --- | --- |
| Oh My Pi uses "hash-anchored lines" | One 4-character hash per file, plus plain line numbers |
| Aider fuzzy-matches failed edits | The fuzzy fallback is dead code behind an early `return` |
| Exo forbids editing its own core | It's written in the docs; nothing in the code enforces it |
| Hermes uses a cheap helper model | Helper tasks default to the main, expensive model |
| Exo's prompt is ~2.3k tokens | It's ~4.8k, because the Slack guide is always sent |

None of these are scandals. But if you copy an idea from a README, check the code first.

## What to take away

### If you're choosing an agent

- **On Claude models:** Claude Code had the best measured caching (98.5% hits) and top quality.
- **Best value in a same-model test:** Oh My Pi solved 88% of tasks at $0.52 per solved task (a small, 25-task benchmark).
- **Frugal and hackable:** Pi, with a ~670-token prompt and four tools.
- **Long-lived, repetitive work:** Hermes, which learns reusable skills after tasks (at roughly 30k tokens per review).
- **Whatever you pick:** check its safety defaults. Several approve everything out of the box.

### If you're building your own

We'd build in six steps, each proven by tests against a scripted fake model before touching a real API:

1. **A bare loop** with read, write and bash tools, an append-only log, and cost and turn caps.
2. **A careful edit tool:** search-and-replace, the fallback ladder, and stale-read protection.
3. **Context:** cap tool output, read `AGENTS.md`, set cache breakpoints, prune then summarise.
4. **Parallel tools, permissions and sub-agents.**
5. **Extensions:** hooks compatible with Claude Code's, skills, MCP and a JSON mode for scripts.
6. **Memory, later:** a small index the model writes itself.

The target is about 2,000 lines of TypeScript, half the size of Pi's minimal core.

And the mistakes to avoid, all of which we saw in real code:

- Putting the date or anything else that changes into the system prompt.
- Editing or reordering earlier messages, or switching models mid-session. Both wreck the cache.
- Running tool calls from a reply that was cut off at the output limit.
- Truncating tool output silently, with no way to get the full text.
- Shipping permissive defaults to other people without a sandbox or container.

### One lesson from each agent

| Agent | What to steal |
| --- | --- |
| Claude Code | The transcript as the product: append-only log, layered permissions, hooks everywhere |
| Codex CLI | Sandbox first, approval second; a precise patch format and matching ladder |
| Pi | A ~940-line loop with hooks only; everything else is an extension |
| Oh My Pi | Edits tagged with a file snapshot, so stale edits are caught |
| opencode | Permissions as plain data; detects an agent stuck repeating the same call |
| DeepSeek Harness | Every policy is an event listener; the request is computed from the log |
| Exo | Secrets the model never sees, via a proxy. Also a warning: no caching, no compaction |
| Mini-SWE-Agent | The floor: 37 lines and one tool. Capability comes from the model |
| Goose | A test that checks each request starts with the previous one, so the cache holds |
| Aider | A repo map instead of search; git commits as the undo button |
| OpenHands | Old events hidden rather than deleted, so nothing is lost |
| Hermes Agent | A reviewer that turns finished tasks into reusable skills, with hard caps |

## The short version

A coding agent is a small loop wrapped in a lot of careful engineering. The loop is solved. The edges are not: editing files reliably, keeping the cache warm, trimming context, and deciding what the agent may do without asking.

The best harnesses keep their core boring and push every rule out to hooks, where it can be tested, swapped or turned off. That's the design we're now building toward.

One question we couldn't answer: there is still **no controlled benchmark** that runs these agents on the same model with caching on. Our next experiment is to run Pi, Oh My Pi, Claude Code and our own harness on about 20 SWE-bench tasks and measure real cost and pass rate. If you've run something similar, I'd love to hear about it.

---

*Research done 28 September to 2 October 2026 by reading source code at pinned commits. Figures for prompt sizes and line counts are estimates; claims marked as coming from READMEs were not independently tested.*
