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

<details>
<summary><strong>Copy prompt - implement this tip for me</strong></summary>

```md
Implement a self-improving learnings system in my OpenClaw workspace.

Do all of the following:

1. Create a `.learnings/` folder if it does not exist.
2. Create `.learnings/ERRORS.md` for failures, breakages, command errors, and exceptions.
3. Create `.learnings/LEARNINGS.md` for corrections, workflow discoveries, preferences, and things the agent should remember.
4. Update `SOUL.md` so it includes these rules:
   - Before starting any task, check `.learnings/` for relevant past errors.
   - When you fail at something or I correct you, log it to `.learnings/` immediately.
5. If there is already an existing memory or lessons system, merge carefully instead of duplicating it.
6. Add a short section explaining how future entries should be written so the files stay useful.

Then show me:
- what files you created or changed
- the exact rules you added
- one example `ERRORS.md` entry
- one example `LEARNINGS.md` entry
```

</details>

For a fuller version with automatic hooks, structured entries, and a promotion workflow, install the [self-improving-agent skill](https://clawhub.ai/pskoett/self-improving-agent).
