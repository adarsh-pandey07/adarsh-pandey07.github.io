---
layout: post
title: "I read the source of 12 coding agents. Don't trust the README"
subtitle: "Dead code, permissive defaults, and a date that breaks the cache: what the docs don't tell you about Claude Code, Codex, Aider and nine others."
description: "What I found reading the source of 12 AI coding agents: a fuzzy-match fallback that never runs, five agents that approve every command by default, and why the prompt cache decides the bill."
---

Line 183 of Aider's `editblock_coder.py` is a bare `return`:

```python
    return
    # Try fuzzy matching
    res = replace_closest_edit_distance(whole_lines, part, part_lines, replace_lines)
    if res:
        return res
```

Everything after it is the fuzzy-match fallback, the code meant to rescue an edit when the model's copy of a file doesn't quite match the real one. It can never run. ([source, at the commit I read](https://github.com/Aider-AI/aider/blob/5dc9490bb35f9729ef2c95d00a19ccd30c26339c/aider/coders/editblock_coder.py#L183))

It turned up while I was reading the source of 12 AI coding agents, and it set the tone for the rest of the study. I did the reading with AI agents, one per codebase, and I'll explain how that worked further down. The core loop everyone writes about turned out to be the same in all twelve. Where they differ is in things you only see in the code: defaults, dead branches, and what sits in front of the prompt cache.

## Why I did this

I wanted to build my own minimal coding agent. Before writing any of it, I wanted to know how the existing ones actually work. Launch posts and READMEs describe what a tool is meant to do. I wanted what it does.

So I picked a single request, *"add a failing test for X, then fix it"*, and followed it through each agent: entry point, prompt assembly, model call, tool call, result, next turn, stop.

The twelve: Claude Code, Codex CLI, Pi, Oh My Pi, opencode, DeepSeek Harness, Exo, Mini-SWE-Agent, Goose, Aider, OpenHands and Hermes Agent. Eleven are open source and I read them at pinned commits. Claude Code is closed, so for that one I used the official docs and the structure of my own session logs. I didn't use the leaked source.

## How I did it, with AI help

Reading twelve codebases by hand would have taken me months, so I wrote a research brief and handed the reading to AI agents, one per harness, running in parallel. Each one traced the same request and wrote a report on the same eleven topics, from the loop to the edit format to safety.

The brief had one rule I wouldn't bend: every claim must cite a file and a function, or an official doc URL. Anything the agent didn't see directly had to be marked as inferred. I also made the agents stop after the first report (Pi, the smallest) so I could check its depth and format before the other eleven ran.

That rule is what made the reports usable. When a report said something surprising, I could open the file and the function and look. Most of what follows comes with a link you can check the same way.

The research ran from 28 September to 2 October 2026.

## The loop really is tiny

This part isn't news. Thorsten Ball showed in 2025 that [a working coding agent fits in a few hundred lines](https://ampcode.com/notes/how-to-build-an-agent), and an April 2026 paper, [Inside the Scaffold](https://arxiv.org/abs/2604.03515), maps the architecture of 13 open-source agents from their source.

What I saw matches. Every agent runs the same loop: send the conversation to the model, run whatever tools it asks for, append the results, repeat. In 10 of the 12, a reply with no tool calls means the agent is done.

Mini-SWE-Agent does it in 37 lines with a single tool, bash, and its README claims over 74% on SWE-bench Verified. Pi's whole minimal core is about 4,450 lines.

![The parts of a coding harness and how they connect](/assets/blog/harness-anatomy.png)

So I stopped looking at the loop and looked at everything around it.

## The docs and the code disagree

These are the cases where what a project says about itself and what its code does came apart:

- Oh My Pi advertises "hash-anchored lines". The code keeps one 4-character hash per file, plus ordinary line numbers (`crates/pi-edit/src/store.rs`, `file_hash()`).
- Aider's fuzzy fallback for failed edits is the dead code above.
- Exo's docs say the agent may not edit its own trusted core. Nothing in the code enforces that, and the repo is mounted read-write.
- Hermes is described as using a cheaper helper model for side tasks. In the code, helper tasks default to the main model. A comment in `agent/auxiliary_client.py` says it outright: "every settings surface defines 'auto' as the main model".
- Exo's prompt is described as about 2.3k tokens. It's about 4.8k, because a 7.4k-character Slack guide is sent with every request.
- Goose's built-in tools look like MCP servers. They are in-process objects that speak MCP types, with no server behind them.
- Pi 1.0's release notes call its new "codemode" faster. There is no benchmark or timing for it in the repo.

They're all small. But they changed how I read projects: before I copy an idea now, I find the function that implements it.

## Five of the twelve approve everything by default

Before running a command the model wrote, an agent can do one of three things: run it inside an operating-system sandbox that limits what it can touch, ask you first, or just run it. Most of them can do the first two. What I didn't expect was how many ship with both switched off.

![What each of the 12 agents does with a shell command, out of the box](/assets/blog/safety-defaults.png)

Five agents approve every action by default: Oh My Pi (approval mode `yolo`), OpenHands (`NeverConfirm`), Goose (`Auto`), opencode (`"*": "allow"`) and Exo (`AlwaysAllow`). OpenHands and Exo are built to run inside Docker, so the container is their safety net. Goose and opencode have proper permission systems; they're just off until you turn them on. Pi has no permission checks in its core at all, and its docs tell you to use a container.

Only Claude Code, Codex and DeepSeek Harness put commands in a sandbox by default. If you try one of the five on your laptop with its defaults, whatever shell command the model writes runs without asking you.

## The cache decides the bill

I almost left cost out. It wasn't in my first brief, and I added a separate cost pass after the main reading. It turned into the most useful part of the study.

Every turn, an agent sends the whole conversation to the model again. Prompt caching lets the provider skip re-processing the part it has already seen, and charges about a tenth of the normal price for it. I worked out what a 20-turn task would cost on each agent, with caching on and off:

![Estimated cost of the same 20-turn task for each agent, with and without prompt caching](/assets/blog/cache-cost.png)

Caching cuts the bill 3.5 to 4.5 times. The most expensive agent with caching, Claude Code at $0.58, is still cheaper than the cheapest one without it, Mini-SWE-Agent at $1.29. Once the prompt is cached, its size barely matters: Claude Code's starting prompt is about 25 times bigger than Pi's, and the task costs only about 1.6 times as much. What moves the bill is how many turns an agent takes.

The real numbers back this up. In 30 of my own Claude Code sessions, 1.76 billion tokens were read from the cache against 27.4 million written to it, a 98.5% hit rate.

The catch is that the cache only matches from the very first byte. The provider reuses the longest stretch at the start of a request that's identical to one it has already seen. Everything after the first difference is processed, and billed, at full price.

![How a prompt cache matches a request from the first byte, and how a date at the front of the system prompt makes the whole request miss](/assets/blog/cache-prefix.png)

That makes anything that changes near the front of a request expensive. opencode puts the date in its system prompt ([`session/system.ts`, line 83](https://github.com/anomalyco/opencode/blob/ad6c72c7068812d43b31f3cfb9e413356a19d850/packages/opencode/src/session/system.ts#L83)):

```ts
`  Today's date: ${new Date().toDateString()}`,
```

The date sits inside the first block of every request. When a session runs past midnight, that block changes, so nothing after it matches and the whole conversation is billed at full price again. Exo does a bigger version of the same thing: it sends a 7.4k-character Slack guide and other blocks that change ahead of the conversation history, and it sets no cache markers at all. OpenHands' condenser rewrites the start of the history roughly every 40 events. And Hermes, Pi, Oh My Pi and opencode run their helper tasks, such as titles and summaries, on the expensive main model by default.

The best fix I found is in Goose: a test that checks each request is a byte-for-byte prefix of the next one. If someone adds a timestamp to the prompt, that test fails in CI.

## Editing files is where agents break

Most failures happen at the edit step, when the model's idea of a file doesn't match the file. The agents that handle this well have converged on the same recipe:

1. Exact search-and-replace, and the match has to be unique.
2. If it doesn't match, a fallback ladder: ignore trailing whitespace, then surrounding whitespace, then smart quotes and other Unicode punctuation. Codex does this.
3. Errors the model can act on: show the closest real lines, or the line numbers of every ambiguous match.
4. Refuse to edit a file the model hasn't read, or one that changed since it read it. Claude Code and Oh My Pi do this.

Tool counts vary wildly: one tool in Mini-SWE-Agent, four in Pi (read, write, edit, bash), about 46 in Claude Code. Aider has none; the model writes edit blocks as plain text and Aider parses them, which is exactly where that dead fallback would have helped.

## Smaller things I didn't expect

No agent has a built-in test runner. "Verify your work" means the model running tests through bash. A few nudge it along: Hermes adds a reminder if code was edited and nothing was checked, Codex's `Stop` hooks can block the agent from finishing, and Aider lints after every edit.

Plan mode is a permission preset, not a separate system. In opencode, the `plan` agent is the `build` agent with editing denied.

Codex adopted Claude Code's hook event names as they are, and DeepSeek Harness and Oh My Pi ship bridges for them. Nobody announced a standard, but hooks are converging on one.

And Claude Code's own tools page lists 46 tools under a heading that says 59.

## Where my own research went wrong

Three things went sideways, and they're worth knowing if you try this yourself.

The list grew. I started with six core agents and a short optional list. By the end there were twelve, including four that weren't on either list.

The cost pass ran out of budget. The agents researching token costs hit 98% of my session limit and stopped before writing anything. I resumed it a few days later and it finished on 2 October. Research about token cost ran out of tokens.

My own report went stale within days. Pi released 1.0 on 1 October, in the middle of the study. It added built-in MCP support, so the line in my report saying Pi has no MCP was wrong. I re-read Pi at v1.0.0 and fixed it, and afterwards I pinned every repo at the exact commit I'd read, so each claim points at code that won't move.

## What this doesn't tell you

I read code; I didn't run benchmarks. Prompt sizes and line counts are estimates, within about 30%, and the cost chart is a worked example from those estimates and current prices, not a measurement. Claude Code's internals are inferred from its docs and my logs. Benchmark numbers quoted from READMEs, like Mini-SWE-Agent's 74%, I didn't test.

The question I couldn't answer is the one that matters most for choosing an agent: how they compare on the same model, with caching on, on the same tasks. I couldn't find a controlled benchmark like that. My next experiment is to run Pi, Oh My Pi, Claude Code and my own harness on about 20 SWE-bench tasks, and record the real cost and pass rate. If you've logged cache-hit rates for opencode or Codex on real work, I'd like to compare numbers.

## Appendix: one idea worth taking from each

- Claude Code: the transcript as the product. Append-only log, layered permissions, hooks everywhere.
- Codex CLI: sandbox first, approval second; a precise patch format and its matching ladder.
- Pi: a roughly 940-line loop with hooks only. Everything else is an extension.
- Oh My Pi: edits tagged with a file snapshot, so stale edits get caught.
- opencode: permissions as plain data, and detection of an agent stuck repeating the same call.
- DeepSeek Harness: every policy is an event listener, and the request is computed from the log.
- Exo: secrets the model never sees, via a proxy. Also a warning: no caching, no compaction.
- Mini-SWE-Agent: the floor. 37 lines and one tool; the capability comes from the model.
- Goose: the test that each request starts with the previous one, so the cache holds.
- Aider: a repo map instead of search, and git commits as the undo button.
- OpenHands: old events hidden from the model, never deleted.
- Hermes Agent: a reviewer that turns finished tasks into reusable skills, with hard caps.
