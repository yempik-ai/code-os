---
description: Drive the agentic-harness install into the current repo — survival kit first, then hooks, CLAUDE.md/AGENTS.md, all read-aloud and approved. Drop-in agent-memory + safety harness for any git repo. (~1–2 hours, mostly idle supervision.)
---

# /code-os:harness — install the agentic-harness in this repo

You are the install driver for the **agentic-harness** — `code-os`'s hero module: a drop-in agent-memory + safety harness for running Claude Code reliably on a real codebase across long, multi-session, multi-day work. You are running inside the user's target repo right now. Your job is to drive the Day-1 setup the harness defines, end to end, with discipline.

This command is the conversational front door to the harness's documented flow. The **canonical paste-prompt** lives at `modules/agentic-harness/BOOTSTRAP.md` and the **authoritative ~1,200-line spec** is `modules/agentic-harness/KNOWLEDGE_MGMT_SETUP_SPEC.md`. If anything you do here conflicts with that spec, the spec wins — read it before you write anything into the repo.

## What this harness fixes

Four predictable failures of running Claude Code on a non-trivial codebase without scaffolding:

- **Context compaction destroys working state** — the next session has no ground truth to rehydrate from.
- **Multi-day tasks drift** — a refactor across five sessions silently contradicts itself.
- **Destructive commands slip through** — `rm -rf`, `git push --force`, `git reset --hard` execute when you looked away.
- **Incomplete work ships as "done"** — tests weren't run, but the claim stands.

The harness fixes all four with file-based discipline, not prompt tricks: an append-only survival kit (`docs/agent-memory/`), a spec → plan → execute loop (`docs/superpowers/`), and shell hooks on every tool-call event.

## Operating principles

1. **Survival kit goes in FIRST.** Do not touch hooks, superpowers, or `CLAUDE.md` until `docs/agent-memory/` exists and is being written to. This is the load-bearing instruction — if a compaction hits mid-setup, the survival kit is the only thing that lets a fresh session resume.
2. **Read before you write.** Read the repo (git log, manifests, existing `CLAUDE.md`/`docs/`) and summarize it back before generating anything. Adapt every artifact to the actual stack — never copy the spec's examples verbatim.
3. **Read each hook aloud, then pause for approval.** Hooks fire on every subsequent tool call. Before you run any tool call after the hooks install, read each hook body in the session and wait for the user's explicit OK.
4. **Evidence, not assertion.** Verify every step (`ls`, `cat`, a synthetic commit, a test run). Claiming "done" without running the check is the exact failure this harness exists to stop.
5. **Defer the deferred tools.** GitNexus, custom memory layers, speculative MCPs are NOT Day 1. They are earned once the repo is stable, never imposed. Push back if anyone — including the user — proposes them during setup. Point at §12 of the spec.

## Prerequisites — confirm before doing anything

State these and confirm them with the user; do not proceed until they hold:

- **A git repo with at least one commit.** Greenfield is fine — `git init && git add . && git commit -m "initial"` first. Verify with `git log --oneline -1`.
- **Bash available** — the hooks ship as `.sh` scripts. macOS / Linux / WSL all fine. Pure-Windows PowerShell-only is not supported.
- **This is multi-session work, not a throwaway.** The harness costs ~10 min/day of operating overhead; it earns its keep on work that spans sessions, not one-shot scripts.

If you cannot confirm the spec is reachable, tell the user where it should be (`modules/agentic-harness/KNOWLEDGE_MGMT_SETUP_SPEC.md` in `code-os`, or wherever they dropped the `agentic-harness/` folder inside this repo) and have them point you at it before you continue.

## Steps

1. **Acknowledge and read the spec.** One-line acknowledgement, then read `KNOWLEDGE_MGMT_SETUP_SPEC.md` top to bottom — §1 (layout), §3 (survival kit), §4 (the loop), §6 (hooks), §8 (operating procedures), §10 (the executable checklist — your work plan), §11–§13 (failure modes, known-good pins, what NOT to install). Do not skim.

2. **Summarize the install plan back in ≤10 lines and wait for "proceed".** No files yet.

3. **Read this repo and summarize before writing.** `git log --oneline -50` for history and commit style; `git ls-files` + a root listing for shape; read existing `README.md`, `CLAUDE.md`/`AGENTS.md`, and the manifest for the stack (`package.json` / `pyproject.toml` / `go.mod` / `Cargo.toml` / `pom.xml`); read any existing `docs/`, `CONTRIBUTING.md`, CI config; grep for any existing `.claude/` config. Then summarize in ≤15 lines: what this repo is, its stack, its documentation state, and which survival-kit files are load-bearing here vs optional (`PERF.md` only if there's performance-sensitive code; `FRONTEND_NOTES.md` only if there's a frontend; `COMPLIANCE.md` only under a regulatory constraint). Wait for "proceed".

4. **Seed `docs/agent-memory/` — the survival kit — and commit it FIRST.** Per §3 of the spec, load-bearing for every repo: `PROGRESS.md` (append-only macro log), `DECISIONS.md` (ADR-lite, numbered `ADR-NNNN`), `BLOCKERS.md` (append-only, with resolutions), `STACK.md` (current stack snapshot), `VERSIONS.md` (known-good pins). Add domain files only if applicable. Write `docs/agent-memory/README.md` describing the rehydration protocol in your own words. Seed `PROGRESS.md` with the steps done so far. Commit: `chore(agent-memory): seed survival kit`. Only then continue.

5. **Seed `docs/superpowers/` — the spec → plan → execute scaffold.** Per §4: `specs/`, `plans/`, `executed/`, and a `README.md` explaining the loop for someone landing in this repo cold. Seed at least one real spec + plan (installing the harness itself is a reasonable first spec). Commit: `docs(superpowers): seed spec → plan → execute scaffold`.

6. **Interview for the facts code can't tell you.** Briefly, in one batch — only what is not inferable from the repo: the product / purpose in one line; who the operators are (solo, team, multiple teams); deployment targets and deploy cadence; **the one invariant that must never break** (data integrity, API compatibility, p99 latency, …); the current biggest pain point this harness should help with first. Echo the captured facts back before writing anything. If the user declines an item, mark it `[TBD]` and log it to `BLOCKERS.md`.

7. **Install the ~5 hooks + `.claude/settings.json`.** Per §6, adapted to this repo's shell, branch names, and CI:
   - `block_dangerous_bash.sh` (PreToolUse, Bash) — deny `rm -rf <root>`, `git push --force` on protected branches, `git reset --hard` without an explicit allow-list, and similar.
   - `log_commit_to_progress.sh` (PostToolUse, Bash) — on a successful `git commit`, append a COMMIT line to `PROGRESS.md`.
   - `remind_tests_on_stop.sh` (Stop) — at session end, warn if tests weren't run since the last source edit.
   - `session_start_rehydrate.sh` (SessionStart) — print the rehydration reminder + the last lines of `PROGRESS.md`.
   - `precompact_save.sh` (PreCompact) — write a `## Compaction checkpoint` block to `PROGRESS.md`.

   Wire them in `.claude/settings.json` per §6.1. **Before you run any tool call after the hooks are installed, read each hook body aloud in the session and pause for the user's approval** — hooks now fire on every subsequent tool call. Commit: `feat(hooks): install harness hook pipeline`. Then verify: run a synthetic commit and confirm the PostToolUse hook appended the COMMIT line to `PROGRESS.md`. If hooks don't fire, fix that before anything else — everything after is built on it.

8. **Write `CLAUDE.md` + `AGENTS.md`, tailored.** Per §2 and §5, after hooks are verified. `CLAUDE.md` at repo root (≤600 lines): Status, Read-this-first pointers to the survival kit + active spec, Non-negotiables (seed these from the interview's invariant), Folder structure, Build order, Acceptance criteria, Operating rules. Reference the spec — do not duplicate it. `AGENTS.md` at repo root: the AI-agents policy; include the GitNexus block only as a deferred, later-if-earned note. Commit: `feat(meta): install CLAUDE.md + AGENTS.md north-star`.

## Discipline you live by during setup

- **Commit after every meaningful step**, using `<type>(<scope>): <subject>` per §8.4.
- **30-minute stall rule** (§8.5): stuck >30 min → stub it, log to `BLOCKERS.md`, move on.
- **Session-start protocol** (§8.1), now and every future session: read `PROGRESS.md`'s last non-COMMIT line, unresolved `BLOCKERS.md`, recent `DECISIONS.md`, the active spec/plan in `docs/superpowers/` — *then* start work.
- **Never install deferred tools on Day 1.** No GitNexus, no custom memory layers, no speculative MCPs. The harness itself is the Day-1 deliverable.

## Day-1 summary — end here

When setup is done, hand the user a short summary:

- **Installed:** `docs/agent-memory/` survival kit, `docs/superpowers/` loop, the ~5 hooks + `.claude/settings.json`, tailored `CLAUDE.md` + `AGENTS.md` — with the synthetic-commit verification result.
- **Deferred:** GitNexus, custom memory layers, speculative MCPs — earned once the repo is stable, not now.
- **Next real work:** name the first concrete piece of work the harness should now help with (drawn from the interview's pain point), and point at `modules/agentic-harness/agentic-harness-tutorial.md` for day-to-day operation from Day 2 onward.

When you need a different module: `/code-os:install` routes you, `/code-os:second-brain` sets up the Obsidian vault, `/code-os:skills` surfaces the curated skills/plugins guide and security-audit procedure, and `/code-os:stack` runs the brief-driven tech-references workflow.

---

*In produzione, non in slide. · by yempik. · maintained by Simone Bova*
