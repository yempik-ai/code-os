# Harness Tutorial — operating a repo with the agent-memory discipline

> **For the engineer.** Once `BOOTSTRAP.md` has run and Claude Code has installed the harness in your repo, this file is the field manual. Covers the daily rhythm, the survival-kit protocol, the spec → plan → execute loop, what happens at each hook event, and the failure modes worth knowing.

## The core idea

Agentic work on a codebase has two failure modes the harness directly addresses:

1. **Memory loss across session boundaries.** Claude Code's context compacts or ends; the next session starts cold. Without durable state, it redoes work, contradicts earlier decisions, or silently forgets blockers.
2. **Claims without evidence.** Claude says "done" when tests weren't run or the build wasn't verified. Without gates, the claim becomes the ground truth and you find out later.

The harness fixes both with file-based state (`docs/agent-memory/`), a disciplined loop (`docs/superpowers/`), and unconditional hooks (`.claude/hooks/`). The files are the ground truth; Claude's session memory is scratch space.

## The survival-kit files — what each does

`docs/agent-memory/` is the entire "what does a fresh session need to know" answer. Every file is append-only markdown the agent writes and reads. You read them too.

- **`PROGRESS.md`** — append-only macro log. One line per completed step, one line per commit. The last non-COMMIT line is the current frontier. Format per §3.1 of the spec. Read this first at every session start — it tells you (and the agent) where you left off.
- **`DECISIONS.md`** — ADR-lite, numbered `ADR-NNNN — <title>`. Records architectural choices with context, options considered, decision, consequences. Reference ADRs by number from `PROGRESS.md`. Read the recent ones at session start if you're touching the relevant subsystem.
- **`BLOCKERS.md`** — append-only, with resolutions. Every `[!blocker]` stub gets an entry here. When resolved, append a `### Resolution (YYYY-MM-DD)` block under the entry — never delete. Read unresolved entries at session start; they're the landmines.
- **`STACK.md`** — current stack snapshot (languages, frameworks, runtime versions, critical deps). Updated when deps change. Answers "what does this codebase run on?" in 30 seconds.
- **`VERSIONS.md`** — known-good pins (§11.1 of the spec). Lists versions that have been battle-tested vs the bleeding-edge. Guards against "let's bump the dep" drift.
- **`SECRETS.md`** — pointer file only. Names *which* secrets the app needs and *where they live* (1Password vault, `.env`, cloud secret manager). **Never contains actual secret values.** If you find a real secret here, remove it immediately.
- **`PERF.md`** — present only if the repo has perf-sensitive code. Known hot paths, measured baselines, perf budgets.
- **`FRONTEND_NOTES.md`** — present only if there's a frontend. UI conventions, component patterns, accessibility rules.
- **`COMPLIANCE.md`** — present only if there's a regulatory constraint (GDPR, HIPAA, SOC2, PCI-DSS). What's in scope, what the audit hooks look like.
- **`README.md`** — meta: describes the survival-kit protocol itself. A fresh session reading only `docs/agent-memory/README.md` can bootstrap.

**The contract:** if a fact is load-bearing across sessions, it lives in one of these files. If it only matters this session, it stays in Claude's context. The rule is "what would a new hire need to know by Monday morning?" — that goes in agent-memory.

## The spec → plan → execute loop

`docs/superpowers/` is how non-trivial work moves from "idea" to "shipped" without drift.

- **`specs/NN-<subsystem>.md`** — the spec for a subsystem. What it must do. What it must not do. Architecture. Test strategy. Template in §14.6 of the spec.
- **`plans/NN-<subsystem>.md`** — the plan derived from the spec. Broken into TDD-sized steps, each a checkbox. Each step small enough to finish and commit in <60 minutes. A step is "write test X, watch it fail" or "implement Y until test X passes".
- **`executed/NN-<subsystem>.md`** — archived plans once complete. Contains the original plan + commit refs for each step + a closing retrospective note.

**How you use it:**

1. New subsystem → write a spec with Claude's help.
2. Spec approved → derive a plan. The plan is your task list for the next N sessions.
3. Execute one step at a time. TDD (§8.3): red → green → refactor → commit.
4. When the last step ships, move the plan to `executed/` with closing notes.

**Why this works:** the spec is a contract with future-you (and future-Claude). The plan is a contract with the current subsystem. The archive is the audit trail. A fresh session reading the spec + the plan knows exactly what's in-progress and what ships next — no ambiguity.

## Session-start protocol (every session, no exceptions)

Before doing any real work, both you and Claude Code must rehydrate. §8.1 of the spec spells it out; the short version:

1. Read `docs/agent-memory/PROGRESS.md` — last non-COMMIT line is the frontier.
2. Read `docs/agent-memory/BLOCKERS.md` — unresolved entries are live landmines.
3. Read `docs/agent-memory/DECISIONS.md` — any ADRs in the last 30 days relevant to the current subsystem.
4. Read the active plan in `docs/superpowers/plans/` — find the first unchecked box; that's your next step.
5. Only then start work.

The `session_start_rehydrate.sh` hook prints the last 5 lines of `PROGRESS.md` automatically at session start, so you don't forget step 1.

Skipping rehydration is the #1 way to waste a session. You'll redo work, contradict decisions, or re-hit a blocker that was already logged.

## Before editing any symbol (§8.2)

Before Claude edits a function, class, module, or public API:

1. `git grep -n` (or `rg`) for every call site.
2. If GitNexus is installed (not Day 1), use it for impact analysis — see the §5 AGENTS.md block.
3. If the change crosses a subsystem boundary or public interface, write the change as a spec + plan first, don't hack it live.

This is the gate that stops "quick fix" from silently breaking three other callers.

## Commit cadence (§8.4)

- **One commit per plan step**, not per file. Commits bracket logical pieces of work.
- Convention: `<type>(<scope>): <subject>` where type ∈ `feat | fix | refactor | test | docs | chore | perf | WIP`.
- `WIP` is allowed but must be rebased or fixed-up before merging to main.
- After every commit, the PostToolUse hook appends a `COMMIT` line to `PROGRESS.md`. Verify this once per session.

## Blocker protocol (§8.5)

You or Claude hit a wall — a flaky dep, a missing credential, an architectural question without an answer.

1. **30-minute stall rule.** If no forward progress in 30 min, stop. Perfectionism is the enemy of compounding.
2. **Stub in code** — add a `// TODO: blocked — see BLOCKERS.md YYYY-MM-DD` comment where the work stopped.
3. **Log to `BLOCKERS.md`** with format from §3.3 of the spec: date, subsystem, one-line what was blocked, full context, what was tried.
4. **Move on** — pick the next unblocked step in the plan. Come back when the blocker can be resolved.
5. **On resolution** — append a `### Resolution (YYYY-MM-DD)` block under the original entry. Never delete the entry.

## Verification before completion (§8.6)

Before claiming a step or a subsystem done:

1. Run the tests relevant to the change — output, not claim.
2. Run the build — output, not claim.
3. If the change is user-visible, exercise it in the app (not just unit tests).
4. Only then tick the checkbox in the plan.

The `remind_tests_on_stop.sh` hook fires at session end and checks whether tests were run since the last source-file edit. If not, it nags you.

## Context-compaction recovery (§8.7)

Claude's context is about to fill up, or compaction just happened:

- **If you can predict compaction:** ask Claude to write a `## Compaction checkpoint YYYY-MM-DD HH:MM` block to `PROGRESS.md` naming the last completed step, current step, in-flight decisions. The `precompact_save.sh` hook does this automatically when the harness fires a PreCompact event.
- **On rehydration:** Claude runs the §8.1 session-start protocol. If `PROGRESS.md` and the active plan disagree on what's done, trust `PROGRESS.md` — it's the ground truth of what shipped.
- **If mid-step work got summarised away:** the checkpoint line tells you which step was in flight. Claude re-reads the plan's description of that step and resumes. If it's unclear, Claude logs a blocker and asks you.

## Long-horizon loops (§8.8)

Multi-day work (a big refactor, a migration, a new subsystem that spans a week):

- Spec + plan are non-negotiable. Don't work multi-day off ad-hoc instructions.
- Every session starts with the §8.1 rehydration.
- Every session ends with a `PROGRESS.md` entry summarising what shipped this session, what's next.
- If the session didn't commit anything in >24h of active work, that's a stall — stub, blocker, move on.
- For overnight / unattended runs: set a narrow scope ("complete steps 4–7 of plan NN") rather than open-ended ("finish the refactor"). Claude can't course-correct scope without you.

## The hooks — what fires when

You installed ~5 shell hooks during setup. They fire automatically; you rarely invoke them directly.

- **`block_dangerous_bash.sh`** (PreToolUse, Bash) — denies `rm -rf <root>`, `git push --force <protected-branch>`, `git reset --hard <HEAD~>`, and similar. When denied, Claude surfaces the command; you decide.
- **`log_commit_to_progress.sh`** (PostToolUse, Bash) — when `git commit` succeeds, appends a `COMMIT <hash> <subject>` line to `PROGRESS.md`.
- **`remind_tests_on_stop.sh`** (Stop) — at session end, checks if tests ran since the last source-file edit. Nags if not.
- **`session_start_rehydrate.sh`** (SessionStart) — prints the §8.1 protocol reminder + last 5 lines of `PROGRESS.md`.
- **`precompact_save.sh`** (PreCompact) — writes a checkpoint block to `PROGRESS.md` naming last-completed + current-in-flight work.

If a command misbehaves, check if a hook intercepted it. Hook logs live wherever the hook writes — usually `.claude/hooks.log` by convention.

**Updating a hook:** edit the script, read it aloud in the Claude session so you know what's running on the next tool call, commit with `feat(hooks): <change>`.

**Never disable a hook to unblock a command.** Narrow the deny rule to exclude your case; commit the narrower rule. A disabled hook means the next person (or next session) doesn't get the safety either.

## The typical day

- **Session start** (~2 min): Claude reads `PROGRESS.md`, `BLOCKERS.md`, recent `DECISIONS.md`, active plan. Prints summary. You skim.
- **During work:** TDD loop on the active plan. One step at a time. Commits between steps. Blockers stubbed and logged as they appear.
- **Session end** (~2 min): last commit + `PROGRESS.md` entry describing the session's outcome. `remind_tests_on_stop.sh` nags if tests didn't run.
- **End of week:** sweep `BLOCKERS.md` — resolve what's become resolvable, escalate what's stale >7 days. Move finished plans to `executed/`.
- **On a new subsystem:** spec first → plan derived from spec → execute. No exceptions for "simple" subsystems; "simple" is where drift hides.

## Common failure modes

- **`PROGRESS.md` is empty or sparse.** Claude stopped writing to it. Fix: at next session start, ask Claude to reconstruct the last ~5 entries from `git log --oneline -20`. Then enforce the commit → `PROGRESS.md` loop.
- **Plan and `PROGRESS.md` disagree.** `PROGRESS.md` is ground truth. Ask Claude to reconcile the plan to match what actually shipped.
- **`BLOCKERS.md` has 15 unresolved entries.** You've accumulated debt. Dedicate a session to triage: resolve what's easy, escalate or deprioritise the rest, delete entries where the subsystem no longer exists.
- **Hook denied a command you needed.** Narrow the rule, commit, retry. Don't disable.
- **Claude claims a step is done without tests run.** The verification-before-completion rule was skipped. Back up — re-run tests, ensure evidence is in `PROGRESS.md`, then re-tick.
- **Fresh session contradicts last session's decision.** Check `DECISIONS.md` — if the ADR is there and the session ignored it, remind the session of the §8.1 protocol (read `DECISIONS.md` at start). If the ADR isn't there, the earlier session failed to record it — add the ADR retroactively from `git log`.
- **Multi-day task drifted.** Symptoms: the plan's first 3 steps shipped, step 4 has been "in progress" across three sessions without a commit. Back up to the spec; is step 4 actually a single step, or was the plan wrong? Split and re-plan.

## What this harness is not

- **Not a replacement for thinking.** It's scaffolding. The actual engineering decisions are yours.
- **Not free.** It costs you ~5 minutes at session start and ~5 minutes at session end. Skip those 10 minutes and you lose the ground truth that makes the rest work.
- **Not a universal fit.** Tiny repos (<500 LoC, one engineer, one session of work ever) don't need this. The harness earns its keep on multi-session, multi-subsystem, multi-engineer work — or on a single engineer doing week-long tasks where their own memory isn't enough.
- **Not static.** When a convention emerges in your work that the harness doesn't encode, add it. Update `CLAUDE.md`, add an ADR to `DECISIONS.md`, extend a hook. The harness compounds when you treat it as a living artifact.

---

*For the full authoritative spec, see `KNOWLEDGE_MGMT_SETUP_SPEC.md`. For the initial setup prompt, see `BOOTSTRAP.md`. For an overview of this deliverable, see `index.md`.*
