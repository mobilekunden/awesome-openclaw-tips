<div align="center">

<img src="./assets/cover.jpg" alt="OpenClaw Tips cover" width="100%" />

<br />
<br />

# Awesome OpenClaw Tips

<strong>The practical playbook for turning OpenClaw from a fun chatbot into a reliable operating system for recurring work.</strong>

<br />

![Tips](https://img.shields.io/badge/tips-18-8b5cf6?style=flat-square)
![Topics](https://img.shields.io/badge/topics-memory%2C%20reliability%2C%20cost%2C%20automation-0f172a?style=flat-square)
![Status](https://img.shields.io/badge/status-tested%20and%20curated-16a34a?style=flat-square)

</div>

OpenClaw gets valuable when it stops acting like a chat window and starts acting like infrastructure.

Most setups break for predictable reasons: cost drift, context bloat, weak memory, unsafe defaults, too many agents, too much trust in chat history, and not enough visibility into what the system is actually doing.

This repo collects the best practical patterns, prompts, and guardrails for fixing those problems.

## Table of Contents

- [Memory](#memory)
  - [MEM-01: Make your agent learn from its mistakes](#mem-01-make-your-agent-learn-from-its-mistakes)
  - [MEM-02: Flush important state before compaction eats it](#mem-02-flush-important-state-before-compaction-eats-it)
  - [MEM-03: Use SQLite memory search before you pay for embeddings](#mem-03-use-sqlite-memory-search-before-you-pay-for-embeddings)
  - [MEM-04: Treat chat history as cache, not the source of truth](#mem-04-treat-chat-history-as-cache-not-the-source-of-truth)
  - [MEM-05: Split conversations into threads so context stops bleeding across topics](#mem-05-split-conversations-into-threads-so-context-stops-bleeding-across-topics)
  - [MEM-06: Make the workspace folder the source of truth and put it under git](#mem-06-make-the-workspace-folder-the-source-of-truth-and-put-it-under-git)
  - [MEM-07: Let the agent learn implicit triggers, not just explicit "remember this"](#mem-07-let-the-agent-learn-implicit-triggers-not-just-explicit-remember-this)
  - [MEM-08: Periodically self-clean memory instead of letting it rot forever](#mem-08-periodically-self-clean-memory-instead-of-letting-it-rot-forever)
- [Reliability](#reliability)
  - [REL-01: Don't put all your fallbacks on the same provider](#rel-01-dont-put-all-your-fallbacks-on-the-same-provider)
  - [REL-02: Your agent says "done" when it isn't](#rel-02-your-agent-says-done-when-it-isnt)
  - [REL-03: Rotate heartbeat checks instead of using one dumb alive ping](#rel-03-rotate-heartbeat-checks-instead-of-using-one-dumb-alive-ping)
- [Cost](#cost)
  - [COST-01: Your heartbeat model is costing you more than you think](#cost-01-your-heartbeat-model-is-costing-you-more-than-you-think)
  - [COST-02: Use cache-ttl pruning or idle sessions will re-cache junk history](#cost-02-use-cache-ttl-pruning-or-idle-sessions-will-re-cache-junk-history)
  - [COST-03: Local models are often a false economy](#cost-03-local-models-are-often-a-false-economy)
  - [COST-04: Use local models only for repetitive mechanical work](#cost-04-use-local-models-only-for-repetitive-mechanical-work)
- [Architecture](#architecture)
  - [ARCH-01: Stop using one generic agent for everything](#arch-01-stop-using-one-generic-agent-for-everything)
  - [ARCH-02: Keep your orchestrator as a manager, not the doer](#arch-02-keep-your-orchestrator-as-a-manager-not-the-doer)
  - [ARCH-03: Give different models different prompt files](#arch-03-give-different-models-different-prompt-files)
- [Automation](#automation)
  - [AUTO-01: Standing orders define what, cron defines when](#auto-01-standing-orders-define-what-cron-defines-when)
- [Operations](#operations)
  - [OPS-01: Set concurrency limits or one stuck task will drain your quota](#ops-01-set-concurrency-limits-or-one-stuck-task-will-drain-your-quota)
  - [OPS-02: Use a task manager so agent work stops being a black box](#ops-02-use-a-task-manager-so-agent-work-stops-being-a-black-box)
- [Who this is for](#who-this-is-for)
- [Contributing](#contributing)
- [Final thought](#final-thought)

## Memory

### MEM-01: Make your agent learn from its mistakes

By default, every session starts clean. Your agent has no memory of the last time it tried something and failed, or the last time you corrected it. Two weeks later, it is still repeating day-three mistakes.

The minimum fix is a `.learnings/` folder in your workspace with two files:

- `.learnings/ERRORS.md` - command failures, breakages, exceptions
- `.learnings/LEARNINGS.md` - corrections, knowledge gaps, workflow discoveries

Add this to `SOUL.md`:

```md
Before starting any task, check .learnings/ for relevant past errors.
When you fail at something or I correct you, log it to .learnings/ immediately.
```

Over time, repeated lessons can be promoted into `SOUL.md` itself so they apply automatically in future sessions.

For a fuller version with automatic hooks, structured entries, and a promotion workflow, install the [self-improving-agent skill](https://clawhub.ai/pskoett/self-improving-agent).

### MEM-02: Flush important state before compaction eats it

Long OpenClaw sessions do not grow forever. Once context gets full, Pi compacts it. That usually means a summary survives, plus recent turns. Anything important that only lives in chat is now at risk.

Treat compaction as a deadline. Important state should already be on disk before it happens.

Turn on the built-in pre-compaction memory flush for embedded Pi sessions:

```json
"agents": {
  "defaults": {
    "compaction": {
      "memoryFlush": {
        "enabled": true,
        "softThresholdTokens": 4000
      }
    }
  }
}
```

This watches context usage, triggers a silent flush before hard compaction, and writes durable state to the workspace. It runs once per compaction cycle and uses `NO_REPLY`, so it does not spam the user.

This pairs naturally with `MEM-01`. Chat is working memory. Files are memory.

### MEM-03: Use SQLite memory search before you pay for embeddings

Most personal OpenClaw setups do not need a vector database just to find old notes. A local SQLite index with FTS5 is often enough.

This repo includes a minimal version in `tools/memory-db/`.

Build the database:

```bash
node tools/memory-db/rebuild-db.js
```

Search it:

```bash
node tools/memory-db/relevant-memory.js "query about previous work"
```

The scripts scan markdown files, build a SQLite database, and use full-text search to surface likely matches quickly. No API costs, no embedding pipeline, no external service.

### MEM-04: Treat chat history as cache, not the source of truth

If you want real autonomy, stop treating the transcript like durable state. Chat is useful while a session is alive, but compaction, resets, and long-running drift make it a bad source of truth.

Keep state and artifacts on disk, then make the agent reconstruct what to do next from files after any compaction or restart. If the system cannot recover from the workspace alone, it is more fragile than it looks.

### MEM-05: Split conversations into threads so context stops bleeding across topics

One long OpenClaw conversation turns into a junk drawer. Coding, research, admin, and random questions all get mixed together, and every new turn drags that baggage forward.

The fix is simple: split work by thread or channel. Keep one thread for coding, another for research, another for admin, another for personal operations. Focused threads give the agent focused context.

### MEM-06: Make the workspace folder the source of truth and put it under git

`AGENTS.md`, `SOUL.md`, `USER.md`, `HEARTBEAT.md`, `MEMORY.md`, and related files are operating state, not loose notes.

Treat the workspace like the real brain of the system. Put it under git, ideally in a private repo, so changes to prompts, memory files, and operating rules are versioned and recoverable.

That gives you two things chat history never does: backup and diff.

### MEM-07: Let the agent learn implicit triggers, not just explicit "remember this"

The best memory systems do not wait for exact phrases like "remember this" or "note that." They also catch decisions, corrections, recurring workflow rules, preferences, and lessons from failures.

If memory only updates when the human says the magic words, a lot of the useful stuff never gets written down.

### MEM-08: Periodically self-clean memory instead of letting it rot forever

Memory quality depends on what gets removed as much as what gets added.

Over time, memory files collect duplicates, stale facts, old preferences, and contradictions. Review them periodically, merge duplicates, resolve contradictions, remove stale items, and archive outdated details instead of leaving them active forever.

Good memory is not just persistent. It is maintained.

## Reliability

### REL-01: Don't put all your fallbacks on the same provider

When one provider rate-limits or goes down, every model behind that provider can fail together. If your entire fallback chain lives inside one vendor, that is not redundancy.

Bad:

```json
"primary": "openai/gpt-5",
"fallbacks": [
  "openai/gpt-5-mini",
  "openai/gpt-5-nano"
]
```

Better:

```json
"primary": "openai/gpt-5",
"fallbacks": [
  "kimi-coding/k2p5",
  "synthetic/hf:zai-org/GLM-4.7",
  "openrouter/google/gemini-3-flash-preview"
]
```

The goal is simple: one provider failure should not take down your whole system.

### REL-02: Your agent says "done" when it isn't

One of the most common OpenClaw failures is fake completion. The agent acknowledges the task, says it is done, and never verifies whether anything actually happened.

Add this to `AGENTS.md`:

```md
Every task follows Execute-Verify-Report. No exceptions.

- Execute: do the actual work, not just acknowledge the instruction
- Verify: confirm the result happened (file exists, message delivered, data saved)
- Report: say what was done and what was verified

"I'll do that" is not execution.
"Done" without verification is not acceptable.
If execution fails, retry once with an adjusted approach.
If it fails again, report the failure with a diagnosis. Never fail silently.
3 attempts max, then escalate to the user.
```

This does not change what your agent can do. It changes what counts as finished.

### REL-03: Rotate heartbeat checks instead of using one dumb alive ping

A basic heartbeat is almost useless once OpenClaw is doing real work. It can return `HEARTBEAT_OK` while email sync is broken, calendar access expired, git is dirty, or a task queue has been stalled for hours.

What works better is a rotating heartbeat. Each run performs the single most overdue real check and updates a state file:

```md
# HEARTBEAT.md

Read `heartbeat-state.json`. Run whichever check is most overdue.

- Email: every 30 min (9 AM - 9 PM)
- Calendar: every 2 hours (8 AM - 10 PM)
- Tasks: every 30 min
- Git: every 24 hours
- System: every 24 hours (3 AM)

Process:
1. Load timestamps from heartbeat-state.json
2. Calculate which check is most overdue
3. Run only that check
4. Update timestamp
5. Report only if something is actionable
6. Otherwise return HEARTBEAT_OK
```

Heartbeat should mean the system is healthy, not just that the agent replied.

## Cost

### COST-01: Your heartbeat model is costing you more than you think

Heartbeats run constantly. Each run is small, but the cost compounds because they never stop.

Set a cheap model specifically for heartbeats:

```json
"agents": {
  "defaults": {
    "heartbeat": {
      "model": "openai/gpt-5-nano"
    }
  }
}
```

Heartbeats do not need intelligence. They need to be fast, reliable, and cheap.

### COST-02: Use cache-ttl pruning or idle sessions will re-cache junk history

Prompt caching only saves money if old context stops coming back after the session sits idle. Without pruning, an old session can wake up and re-cache oversized history that no longer matters.

Turn on cache-ttl pruning:

```yaml
agents:
  defaults:
    contextPruning:
      mode: "cache-ttl"
      ttl: "1h"
```

This trims stale tool-result context after the cache TTL window passes, which helps keep idle sessions from becoming expensive again.

### COST-03: Local models are often a false economy

Local models get pitched as the answer to OpenClaw costs. Most of the time, they are not.

Serious local setups require expensive hardware, and cheaper setups usually pay you back in latency, queueing, or degraded output. Free hosted options often hide the same problem behind long queues.

Local can make sense if you already need the hardware for broader reasons. For most people, a few cents saved on API calls is not worth slowing the whole workflow down.

### COST-04: Use local models only for repetitive mechanical work

Local models start making economic sense once the task is boring enough: email sorting, calendar parsing, simple classification, repetitive extraction.

Save paid models for work that benefits from quality and reasoning. Use local for cheap, mechanical chores. The rule is not "local saves money." The rule is "local is for repetitive work that tolerates weaker output."

## Architecture

### ARCH-01: Stop using one generic agent for everything

Better systems usually come from not treating OpenClaw like one giant do-everything assistant. A monitor, a researcher, and a communicator should not have the same prompt, the same model, or the same job.

Start with a small split:

```json
{
  "agents": {
    "list": [
      {
        "id": "monitor",
        "model": {
          "primary": "openai/gpt-5-nano"
        }
      },
      {
        "id": "researcher",
        "model": {
          "primary": "kimi-coding/k2p5"
        }
      },
      {
        "id": "communicator",
        "model": {
          "primary": "anthropic/claude-opus-4-6"
        }
      }
    ]
  }
}
```

The monitor stays cheap and boring. The researcher reads and compares. The communicator writes things a human might actually send.

Start with three roles, not twelve.

### ARCH-02: Keep your orchestrator as a manager, not the doer

The main agent works best as a manager. Its job is to plan, delegate, keep context straight, and report back.

Once it starts trying to do every coding task, research job, and background chore itself, everything gets slower and more fragile.

If the main agent is busy doing the work itself, it is not really orchestrating.

### ARCH-03: Give different models different prompt files

Model switching is not just about price or speed. Different models respond differently to the same instructions.

Keep separate prompt versions for the main models you rely on, then tune each one for how that model actually behaves. One shared prompt file is often leaving quality on the table.

## Automation

### AUTO-01: Standing orders define what, cron defines when

Standing orders and cron jobs solve different problems. Standing orders define what the agent is allowed and expected to do. Cron defines when it runs.

Keep scheduled prompts small and clean. Point the cron at the standing order instead of pasting the whole workflow into every job.

```bash
openclaw cron create \
  --name daily-inbox-triage \
  --cron "0 8 * * 1-5" \
  --tz America/New_York \
  --timeout-seconds 300 \
  --announce \
  --channel bluebubbles \
  --to "+1XXXXXXXXXX" \
  --message "Execute daily inbox triage per standing orders. Check mail for new alerts. Parse, categorize, and persist each item. Report summary to owner. Escalate unknowns."
```

This gives you one place to update logic later instead of editing every schedule.

## Operations

### OPS-01: Set concurrency limits or one stuck task will drain your quota

OpenClaw has no concurrency cap by default. If a task gets stuck in a retry loop, or several subagents launch at once, they can all run in parallel with nothing stopping them.

Two lines in your config fix that:

```json
"agents": {
  "defaults": {
    "maxConcurrent": 4,
    "subagents": {
      "maxConcurrent": 8
    }
  }
}
```

That keeps failures bounded instead of expensive.

### OPS-02: Use a task manager so agent work stops being a black box

One of the most frustrating parts of running OpenClaw is not knowing what it is doing right now, what is stuck, and what needs your attention.

The exact tool does not matter much: Todoist, Linear, GitHub Projects, Notion, anything with an API can work. What matters is the state model: queue, active, blocked, done.

Once the agent writes work into a system you already check, you get operational visibility without digging through logs or transcripts.

## Who this is for

- People running OpenClaw daily, not just testing it once
- Builders who want durable memory and lower operating cost
- Anyone whose current setup feels smart in demos and flaky in production
- Teams or solo operators trying to make agent workflows predictable

## Contributing

PRs are welcome if they add patterns that are practical, tested, and specific.

Good additions usually include:

- a real failure mode
- the guardrail or pattern that fixed it
- a concrete config, prompt, or operational example
- a short explanation of why it works

## Final thought

The best OpenClaw setups are usually not the most complicated ones.

They are the ones with boring memory, boring verification, boring cost controls, and boring operational limits. That is what makes them trustworthy.
