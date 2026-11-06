---
description: Set up CLI-subagent orchestration in this repo — Claude Code as the orchestrator, other vendors' CLI agents (Codex today, any headless CLI tomorrow) as explorers, implementers, and adversarial reviewers on another plan's meter.
---

# /code-os:subagents

The user wants their Claude Code session to delegate work to other CLI agents. Your job is to install the conventions, not to over-engineer: this is a few lines in `CLAUDE.md` and a discipline, not an infrastructure project.

Source of truth: [`modules/multi-model-orchestration/orchestrating-cli-subagents.md`](../../../modules/multi-model-orchestration/orchestrating-cli-subagents.md). Read it before acting.

## What you do

1. **Check what's reachable.** `which codex` (and any other agentic CLI the user names). If nothing is installed, say what to install and stop; don't scaffold conventions for tools that aren't there.
2. **Build the model table with the user.** Ask which flat plans they actually pay for, then write the cost / intelligence / taste table into their `CLAUDE.md` with the rules of application from the module (intelligence beats taste beats cost for anything that ships; taste at 7+ for anything user-facing; bulk work to the cheapest high-intelligence model; escalate without asking; effort capped at high).
3. **Write the run-path convention into `CLAUDE.md`.** The three-file triple (`<slug>.prompt.md`, `<slug>.final.md`, `<slug>.log`) under a scratch directory, orchestrator as single allocator, unique slugs.
4. **Copy the runbook commands** for each reachable CLI (write mode and read-only mode for Codex, from the module) including every failure-mode guard: prompt on stdin (or stdin closed), launch from repo root, foreground only with an explicit Bash timeout, trail unread, one worktree per parallel writer.
5. **State the acceptance discipline once, plainly:** diff read in first person, checks re-run in first person, subagent output is a claim until verified.

## Boundaries

- Keep the `CLAUDE.md` addition short. If it grows past roughly a screen, you are doing it wrong; the module holds the long form, `CLAUDE.md` holds the contract.
- Do not install wrappers, scripts, or new tools for this. It is shell commands and discipline.
- The orchestrator keeps the accept/reject call. Never set up a flow where subagent output lands unreviewed.
- Need the memory harness first? That's [`/code-os:harness`](harness.md). Subagents without the harness work fine for single sessions; across multi-day work, install the harness too.

---

<sub>In produzione, non in slide. · by <b>yempik.</b> · maintained by <b>Simone Bova</b></sub>
