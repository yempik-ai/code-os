# Agentic Harness — Index

> **What this folder is.** A drop-in knowledge-management + safety harness for running Claude Code on a real codebase across long, multi-session, multi-day tasks. Written for engineers who are new to agentic engineering and want Claude Code to behave reliably under real workloads — not for toy projects or one-shot scripts.

## The problem this solves

Running Claude Code on a non-trivial codebase fails in four predictable ways without scaffolding:

- **Context compaction destroys working state.** Claude's context fills up mid-task and compacts; the next session has no ground truth to rehydrate from.
- **Multi-day tasks drift.** A three-day refactor across five sessions silently contradicts itself because no durable record survives.
- **Destructive commands slip through.** `rm -rf`, `git push --force`, `git reset --hard` sometimes execute when you looked away.
- **Incomplete work ships as "done."** Claude claims a task is finished, but tests weren't run. Without verification gates, the claim stands.

The harness fixes all four with concrete, file-based discipline: an append-only survival kit (`docs/agent-memory/`), a disciplined spec → plan → execute loop (`docs/superpowers/`), and shell hooks on every tool-call event.

## Read order

If you are about to install the harness in a target repo for the first time, read in this order:

1. `DAY-0-CHECKLIST.md` — the ~10 min of human steps Claude Code cannot do (install Claude Code CLI, ensure target repo has a git history, drop this folder into the target repo). Ends by pointing at `BOOTSTRAP.md`.
2. `BOOTSTRAP.md` — the prompt you paste into Claude Code in your target repo to install the harness. ~1–2 hours of mostly-idle supervised setup.
3. `agentic-harness-tutorial.md` — the daily-operation field manual. Read before you start working day-to-day. ~15 min read.

`KNOWLEDGE_MGMT_SETUP_SPEC.md` is the authoritative ~1,200-line spec. You don't need to read it cover-to-cover; Claude Code reads it during setup. Reference it when the tutorial and your installed harness disagree on something.

## Files in this folder

- **`DAY-0-CHECKLIST.md`** — Operator-facing. What you (the human) must do before pasting the bootstrap prompt: install Claude Code, prepare the target repo, drop this folder into it. The handoff to `BOOTSTRAP.md` is the last step.

- **`BOOTSTRAP.md`** — Operator-facing. Installation prompt. Self-contained, inside `===== BEGIN PROMPT =====` / `===== END PROMPT =====` fences. Frames the work as a long-running agentic task, forces the `docs/agent-memory/` survival kit to go in first (so any mid-setup compaction doesn't destroy state), reads your repo before writing anything, interviews you for context, then executes the spec's §10 setup checklist. Includes checkpoints, failure modes, and recovery recipes.

- **`agentic-harness-tutorial.md`** — Operator-facing. Daily-operation field manual. Covers: the survival-kit files and what each does, the spec → plan → execute loop, the session-start rehydration protocol, commit cadence, blocker protocol, verification before completion, context-compaction recovery, long-horizon loop discipline, what each hook does, the typical day, common failure modes, and what the harness is *not*.

- **`KNOWLEDGE_MGMT_SETUP_SPEC.md`** — Agent-facing. The authoritative ~1,200-line spec Claude Code reads during setup. Defines the target directory layout (§1), the survival-kit files file-by-file (§3), the spec → plan → execute loop (§4), the hook scripts and their wiring (§6), the operating procedures (§8), the executable setup checklist (§10), and the deferred-tool policy (§12). If anything in any other file conflicts with this spec, this spec wins.

- **`index.md`** — This file.

## Prerequisites

- **Claude Code installed** (`npm install -g @anthropic-ai/claude-code`).
- **A git repo with at least one commit.** Greenfield is fine — just `git init && git add . && git commit -m "initial"` first.
- **Shell with bash** (hooks are written as `.sh` scripts).
- No other prerequisites. Stack-agnostic — Node, Python, Go, Rust, Java, polyglot all work.

See `DAY-0-CHECKLIST.md` for the full pre-paste sequence.

## Using this as a drop-in

To apply the harness to a new repo:

1. Copy this entire folder (`index.md`, `DAY-0-CHECKLIST.md`, `BOOTSTRAP.md`, `agentic-harness-tutorial.md`, `KNOWLEDGE_MGMT_SETUP_SPEC.md`) into the target repo, for example at `docs/agentic-harness/`.
2. `cd` into the target repo.
3. `claude`.
4. Paste the prompt block from `BOOTSTRAP.md` (replacing `<PATH-TO-SPEC>` with the path where you placed `KNOWLEDGE_MGMT_SETUP_SPEC.md`).
5. Supervise. ~1–2 hours of setup. See `agentic-harness-tutorial.md` for operation from Day 2 onward.

## Scope — what the harness installs

During Day-1 setup, Claude Code writes into the target repo:

- `docs/agent-memory/` — survival-kit files (`PROGRESS.md`, `DECISIONS.md`, `BLOCKERS.md`, `STACK.md`, `VERSIONS.md`, optional domain files like `PERF.md` / `FRONTEND_NOTES.md` / `COMPLIANCE.md`, plus a `README.md`).
- `docs/superpowers/` — `specs/`, `plans/`, `executed/` + a `README.md` explaining the loop.
- `.claude/hooks/` — ~5 shell scripts enforcing safety, auto-logging, and reminders (`block_dangerous_bash.sh`, `log_commit_to_progress.sh`, `remind_tests_on_stop.sh`, `session_start_rehydrate.sh`, `precompact_save.sh`).
- `.claude/settings.json` — wires the hooks to their trigger events.
- `CLAUDE.md` — north-star briefing at repo root, tailored to your stack and invariants.
- `AGENTS.md` — AI-agents policy at repo root.

Nothing else. No code-intelligence tools (GitNexus), no memory layers, no speculative MCPs — those are all explicitly deferred per §12 of the spec, and are earned by the repo once stable, not imposed on Day 1.

## Who this is for

- **Primary:** engineers new to agentic engineering who want Claude Code to work reliably on their day-job repo, not just toy scripts.
- **Secondary:** teams standardising how Claude Code operates across multiple repos — the harness is stack-agnostic and produces the same discipline regardless of language.

Not for: quick scripts, one-shot repos, experiments you'll throw away in a week. The harness costs ~10 min/day of overhead (session-start + session-end discipline); it earns its keep only on work that spans multiple sessions.
