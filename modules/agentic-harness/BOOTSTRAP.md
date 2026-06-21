# BOOTSTRAP — Install the agentic harness in this repo

> **For the engineer.** You want Claude Code to work reliably on your codebase across long sessions, context compactions, multi-day tasks, and handoffs between fresh sessions. This file is the single prompt you paste into Claude Code — pointed at `KNOWLEDGE_MGMT_SETUP_SPEC.md` — to install the full agent-memory discipline as a drop-in.

## What this harness is

Running Claude Code on a non-trivial codebase fails in predictable ways without scaffolding:

- **Context compaction destroys working state.** Mid-task, Claude's context hits its limit and compacts. Decisions made, blockers discovered, and subsystems half-finished get summarised away — and the next session has no ground truth to rehydrate from.
- **Long-horizon tasks drift.** A three-day refactor spread across five sessions silently contradicts itself because each session starts with no durable record of what the earlier ones decided.
- **Destructive commands slip through.** `rm -rf`, `git push --force`, `git reset --hard` sometimes execute when you looked away. Without PreToolUse hooks, nothing stops them.
- **Incomplete work ships as "done."** Claude claims a task is finished, but the tests weren't run, the build wasn't verified, or the subsystem was only half-implemented. Without verification gates, the claim stands unchallenged.

The harness fixes all four with concrete, file-based discipline — not prompt tricks. It installs:

- **`docs/agent-memory/`** — a **survival kit** of append-only markdown files (`PROGRESS.md`, `DECISIONS.md`, `BLOCKERS.md`, `VERSIONS.md`, `STACK.md`, plus per-domain files). Any session — fresh, compacted, subagent — can rehydrate from these in <60 seconds.
- **`docs/superpowers/`** — the **spec → plan → execute** loop. Specs define what a subsystem must do; plans break it into TDD-sized steps; execution proceeds one step at a time with commits per step.
- **`.claude/hooks/`** — **shell hooks** on PreToolUse / PostToolUse / Stop / SessionStart / PreCompact events that enforce safety, auto-log commits, remind about tests, and save compaction checkpoints.
- **`CLAUDE.md`** and **`AGENTS.md`** — the north-star briefing the agent reloads every session, plus the AI-agents policy for GitNexus or other code-intelligence tools if you add them.

Claude Code writes all of it during Day 1, tailored to your repo's actual stack.

## Prerequisites

- **Claude Code installed** — `npm install -g @anthropic-ai/claude-code`. Verify: `claude --version`.
- **A repo with a git history** — the harness assumes git is already initialised and you have at least one commit. If this is a greenfield repo: `git init && git add . && git commit -m "initial"` first.
- **You've read this file** — you're about to let Claude Code write shell scripts and config into your repo. Know what you're approving.

No other prerequisites. The harness is stack-agnostic — Node, Python, Go, Rust, Java, polyglot — all fine.

## How to use this file

1. Drop the entire `agentic-harness/` folder into your target repo — `docs/agentic-harness/` is a sensible location. The path just has to live somewhere inside the repo so Claude Code can read `KNOWLEDGE_MGMT_SETUP_SPEC.md`. (See `DAY-0-CHECKLIST.md` for the full pre-paste sequence.)
2. `cd` into your repo root.
3. Run: `claude`
4. **Copy the block between `===== BEGIN PROMPT =====` and `===== END PROMPT =====`** below. Paste it as your first message. Replace `<PATH-TO-SPEC>` with the path to `KNOWLEDGE_MGMT_SETUP_SPEC.md` inside your repo (e.g. `docs/agentic-harness/KNOWLEDGE_MGMT_SETUP_SPEC.md`).
5. Supervise. Approve tool calls as they come. **Read each hook script before approving the first tool call that installs it** — hooks run on every subsequent tool call, so you want to know what they do.

Expected wall-clock: ~1–2 hours of mostly-idle supervision. Do other work; check back when Claude Code asks a question.

---

===== BEGIN PROMPT =====

You are about to install a **long-running-agentic-session harness** in this repo. This is a research engagement — your final outputs will be evaluated for completeness, discipline, and fidelity to the spec. Token budget is effectively unbounded; optimise for meticulous, auditable quality, not speed.

## 0. Read the spec fully before doing anything else

Read `<PATH-TO-SPEC>` (the `KNOWLEDGE_MGMT_SETUP_SPEC.md` file) top-to-bottom. All ~1,200 lines. Do not skim. Pay particular attention to:

- §1 (target directory layout)
- §3 (the `docs/agent-memory/` survival kit — file by file)
- §4 (the `docs/superpowers/` spec → plan → execute loop)
- §6 (hooks — the unconditional enforcement layer)
- §8 (operating procedures — session-start, blocker protocol, verification, compaction recovery, long-horizon loops)
- §10 (the executable setup checklist — this is your work plan)
- §11–§13 (failure modes, known-good pin discipline, what NOT to install)

Do not start executing until you have read every section.

## 1. Understand this repo before writing anything

Before creating any files, build a picture of the existing codebase:

1. `git log --oneline -50` — recent history, commit style.
2. `git ls-files | head -100` + root directory listing — folder shape.
3. Read existing `README.md`, `CLAUDE.md` / `AGENTS.md` if present, `package.json` / `pyproject.toml` / `go.mod` / `Cargo.toml` / `pom.xml` — stack and dependency shape.
4. Read any existing `docs/`, `specs/`, `RUNBOOK.md`, `CONTRIBUTING.md` — pre-existing documentation conventions.
5. Grep for existing `.claude/` config, if any, and existing hooks.
6. Identify: primary language(s), framework(s), test runner(s), build tool(s), lockfile, CI config (`.github/workflows/`, `.gitlab-ci.yml`, etc.).

Then pause and summarise to me, in ≤15 lines: what this repo is, its stack, its current state of documentation, and which parts of the spec's survival kit are load-bearing for it vs optional (e.g. `PERF.md` only if there's performance-sensitive code; `FRONTEND_NOTES.md` only if there's a frontend; `COMPLIANCE.md` only if there's a regulatory constraint). Wait for my "proceed".

Do not copy-paste the spec's examples verbatim into the repo. Adapt them to the actual stack, language, and conventions you just read.

## 2. Establish the knowledge layer FIRST (before any other work)

This is the most important instruction in this prompt. **Do not proceed to hooks or superpowers until `docs/agent-memory/` exists and is being written to.**

Why: the harness is only useful if it survives compaction and fresh sessions. Without a durable survival kit, every compaction destroys working state and you redo work or silently drift. With the survival kit, any future session (including subagents, or sessions after context limits) can rehydrate and continue coherently.

Concretely, **as your first actions after the §1 repo summary is approved:**

1. Create `docs/agent-memory/` with the core files from §3 of the spec. Load-bearing for every repo:
   - `PROGRESS.md` — append-only macro log.
   - `DECISIONS.md` — ADR-lite, numbered `ADR-NNNN`.
   - `BLOCKERS.md` — append-only, with resolutions.
   - `STACK.md` — the current stack snapshot (versions, deps, runtime requirements).
   - `VERSIONS.md` — known-good pins (§11.1 of the spec).
2. Add domain files only if applicable (`PERF.md`, `FRONTEND_NOTES.md`, `COMPLIANCE.md`, `SECRETS.md` — the latter is a pointer file, never actual secrets).
3. Write `docs/agent-memory/README.md` describing the survival-kit protocol in your own words — so a future fresh session reading only this folder can rehydrate.
4. Seed `PROGRESS.md` with entries for the steps you've completed so far ("Read spec", "Summarised repo", "Seeded agent-memory/").
5. Commit: `chore(agent-memory): seed survival kit`.
6. **Only then** proceed to §3 of this prompt.

At every subsequent step, before claiming a step done:
- Append to `PROGRESS.md` per §3.1 of the spec.
- If blocked ≥30 min, write to `BLOCKERS.md` per §8.5, stub, move on.
- Commit per §8.4.

Session-start protocol (now, and every future session): read `docs/agent-memory/PROGRESS.md` last non-COMMIT line, `BLOCKERS.md` unresolved entries, `DECISIONS.md` recent entries, the active spec/plan in `docs/superpowers/`. **Only then start work.** See §8.1 of the spec for the full checklist.

## 3. Install the spec → plan → execute scaffold

After agent-memory is seeded and committed, create `docs/superpowers/` per §4 of the spec:

- `docs/superpowers/specs/` — per-subsystem specs (use the §14.6 template).
- `docs/superpowers/plans/` — TDD-sized plans per spec.
- `docs/superpowers/executed/` — archive of completed plans with commit refs.
- `docs/superpowers/README.md` — the spec → plan → execute loop explained for operators who land in this repo cold.

Seed at least one real spec + plan for the first meaningful piece of harness work you'll take on in this repo (the harness itself is a reasonable first spec — "Install the agent-memory harness"). Commit: `docs(superpowers): seed spec → plan → execute scaffold`.

## 4. Install hooks

Write the hooks from §6 of the spec, adapted to this repo's shell conventions. At minimum:

- `.claude/hooks/block_dangerous_bash.sh` (PreToolUse, Bash) — deny `rm -rf <root>`, `git push --force` on protected branches, `git reset --hard` without explicit allow-list, and similar.
- `.claude/hooks/log_commit_to_progress.sh` (PostToolUse, Bash) — when a `git commit` succeeds, append a COMMIT line to `docs/agent-memory/PROGRESS.md`.
- `.claude/hooks/remind_tests_on_stop.sh` (Stop) — at session end, check whether tests were run since the last source-file edit; emit a reminder if not.
- `.claude/hooks/session_start_rehydrate.sh` (SessionStart) — print the rehydration protocol reminder + last 5 lines of `PROGRESS.md`.
- `.claude/hooks/precompact_save.sh` (PreCompact) — write a `## Compaction checkpoint YYYY-MM-DD HH:MM` block to `PROGRESS.md`.

Wire them via `.claude/settings.json` per §6.1 of the spec. **Before running any tool call after the hooks are installed, read each hook's body aloud in the session so I can approve what's about to execute on every subsequent tool call.** Then commit: `feat(hooks): install harness hook pipeline`.

Verify hooks fire: run a synthetic commit and confirm `PROGRESS.md` got the COMMIT line appended by the PostToolUse hook.

## 5. Write `CLAUDE.md` and `AGENTS.md`

After hooks are verified, write:

- **`CLAUDE.md`** at repo root — the north-star briefing (≤600 lines, per §2 of the spec). Sections: Status, Read this first (pointers to survival-kit files + active spec), Non-negotiables, Folder structure, Build order, Acceptance criteria, Operating rules (quick reference). Do **not** duplicate the full spec — reference it.
- **`AGENTS.md`** at repo root — the AI-agents policy (§5 of the spec). Include the `GitNexus — Code Intelligence` block if you add GitNexus later (deferred; don't install on Day 1).

Before writing either, conduct a short interview for the facts that cannot be inferred from code:
- What is this codebase's product / purpose in one line?
- Who are the operators (solo engineer, team, multiple teams)?
- What are the deployment targets (local, staging, prod — and the deploy cadence)?
- What's the one invariant that must never break (data integrity, API compatibility, p99 latency, etc.)?
- What's the current biggest pain point this harness should first help with?

Use those answers to shape `CLAUDE.md`'s Status and Non-negotiables sections. If I decline to answer any item, mark it `[TBD]` and log to `BLOCKERS.md`.

Commit: `feat(meta): install CLAUDE.md + AGENTS.md north-star`.

## 6. Operating discipline — the rules you live by during this setup

- **Commit after every meaningful step**, not every file. Use the §8.4 convention: `<type>(<scope>): <subject>`.
- **30-minute stall rule** (§8.5). Stuck >30 min → stub, log to `BLOCKERS.md`, move on.
- **Verification before completion** (§8.6). Before claiming any step done, run the verification named in the spec (`ls`, `cat`, synthetic commit, test-run for the affected subsystem). Evidence, not assertion.
- **Meticulous detail, don't ship shallow.** Hooks must actually deny dangerous commands — test them. `PROGRESS.md` entries must actually name what shipped. Survival-kit files must be complete enough that a fresh session can rehydrate without asking me anything.
- **Ask me only when:** a decision is genuinely ambiguous, a destructive action is required, or personal/contextual information is needed (stack details I didn't mention, operator identity, invariants).
- **Never install deferred tools on Day 1.** Per §12 of the spec: no GitNexus unless the repo is already stable and ≥5k LoC of real code; no custom memory layers; no speculative MCPs. The harness itself is the Day-1 deliverable.
- **Never skip the verification of the hooks.** The first hook-installation commit is the load-bearing one. If hooks don't fire correctly, everything after is built on sand.

## 7. Checkpoints — when to pause and wait for me

Pause and wait for my explicit OK at these points:
1. **After reading the spec** — summarise the installation plan back in ≤10 lines. Wait for "proceed".
2. **After the §1 repo summary** — show me what you learned and which survival-kit files you'll instantiate vs defer.
3. **After the §5 interview** — echo back the captured facts before writing `CLAUDE.md` / `AGENTS.md`.
4. **Before running any tool call after hooks are installed** — read each hook body aloud. Wait for approval.
5. **After Day-1 completion** — verification summary + list of what's deferred to later.

Between checkpoints, work autonomously. Do not stop for routine approvals inside a phase.

## 8. Context-compaction / fresh-session recovery

If your context fills up mid-setup or a fresh session starts:
1. Before compaction (if predictable): write a `## Compaction checkpoint YYYY-MM-DD HH:MM` block to `PROGRESS.md` naming last completed step, current step, in-flight decisions. The PreCompact hook handles this once installed.
2. On rehydration: §8.1 of the spec — read `PROGRESS.md` last non-COMMIT line, `BLOCKERS.md` unresolved, `DECISIONS.md` recent, active spec/plan in `docs/superpowers/`. Only then resume.
3. If `PROGRESS.md` and the active plan disagree, trust `PROGRESS.md` (ground truth of what shipped) and reconcile the plan.

This is exactly why §2 of this prompt forced agent-memory to go in first.

## 9. Start now

1. Acknowledge this prompt in one line.
2. Read the spec.
3. Summarise the installation plan in ≤10 lines.
4. Wait for my "proceed".
5. On "proceed": begin with §1 (read the repo) and work through §2–§5 in order, pausing at the checkpoints in §7.

===== END PROMPT =====

---

## What happens after you paste

**Immediately:** Claude Code reads the spec (~1,200 lines, 1–2 minutes of silence). Then replies with a ≤10-line installation plan. Read it. If it looks right, reply `proceed`. If something is wrong or off-base for your repo, say so — it will adjust before writing any files.

**During Day 1 (~1–2 hours of supervised work):**

- Reads your repo and summarises what it found (stack, existing docs, current pain points).
- Interviews you briefly for repo context (product purpose, operators, deployment targets, invariants, pain points).
- Seeds `docs/agent-memory/` (survival kit) — first commit.
- Seeds `docs/superpowers/` (spec → plan → execute scaffold) — second commit.
- Writes the ~5 hook shell scripts + `.claude/settings.json`. **Will ask you to read each hook before running any tool call after they're installed.**
- Writes `CLAUDE.md` + `AGENTS.md` tailored to your repo.
- Runs a synthetic commit to verify the PostToolUse hook appends to `PROGRESS.md`.
- Produces a final Day-1 summary: what's installed, what's deferred, what's the next piece of real work.

**Days 2+:** You use the harness. See `agentic-harness-tutorial.md` in this folder for the daily-operation field manual.

## When to interrupt Claude Code

- It starts writing `CLAUDE.md` before interviewing you or reading the repo — interrupt. The §1 repo read and the §5 interview come first.
- It copy-pastes the spec's example shell scripts verbatim without adapting to your stack — interrupt. Hooks must be written for your shell, your branch names, your CI.
- A command looks destructive (`rm -rf`, `git push --force`, `git reset --hard`) — read, ask why, only proceed if the answer is good.
- It proposes installing GitNexus, a custom memory layer, or any non-core tool on Day 1 — push back. Point it at §12 of the spec.
- A step runs >30 min with no forward progress — remind it of the §8.5 stall rule.

## If something goes badly wrong

- **Claude Code crashes mid-setup.** Re-run `claude` in the same directory. Paste: *"Resume the harness installation. Follow the §8.1 session-start protocol — read `docs/agent-memory/PROGRESS.md`, `BLOCKERS.md`, `DECISIONS.md`, and the active plan in `docs/superpowers/`. Only then resume from the last incomplete step."* The survival kit is exactly what makes this recovery work.
- **A hook script denies a command you actually wanted.** Don't disable the hook. Narrow the deny rule to exclude your case, commit the fix, retry.
- **Hooks don't fire after install.** Check `.claude/settings.json` is valid JSON and references the right hook paths. Common gotcha: absolute vs repo-relative paths.
- **`CLAUDE.md` has invented facts about the repo.** Ask: *"Find every `[TBD]` marker and every claim in `CLAUDE.md` not grounded in a file you read during §1. Replace invented content with `[TBD]` markers and log the gaps to `BLOCKERS.md`. Re-interview me on the gaps."*

---

*Everything between `===== BEGIN PROMPT =====` and `===== END PROMPT =====` is the full handoff. The spec at `<PATH-TO-SPEC>` is what the agent reads — you do not need to paste it. See `agentic-harness-tutorial.md` for day-to-day operation after setup, and `index.md` for an overview of the files in this folder.*
