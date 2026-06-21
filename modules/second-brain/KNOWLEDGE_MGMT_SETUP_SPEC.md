# Knowledge-Management Setup Spec — for Long-Running Agentic Work

> **Purpose.** This spec defines the knowledge-management system that a long-running coding agent needs in order to do useful work in a repository over hours, days, or sprints without losing progress, context, or discipline. It names every file, directory, hook, MCP, skill, and procedure required, plus the operating rules that make the system survive multi-hour autonomous runs and context compaction.
>
> **Scope.** This spec is **repository-agnostic and stack-agnostic**. It does not assume Python vs TypeScript vs Go, does not assume Postgres vs Neo4j vs SQLite, does not assume any particular LLM provider, ontology, framework, or cloud. Concrete patterns name 2-3 alternatives per slot so the setup-agent picks the one that fits the target repo.
>
> **Audience.** A long-running agent (Claude Opus / Sonnet, Claude Code CLI, or equivalent) with write access to the target repo, Bash, git, and the MCP servers listed below.
>
> **Non-negotiable outcome.** After the agent applies this spec, the target repo can absorb arbitrary multi-hour agentic work with: (a) deterministic session re-hydration after compaction; (b) impact-aware edits via GitNexus; (c) commit-per-subsystem discipline auto-logged to an append-only progress file; (d) safety hooks blocking destructive Bash; (e) spec → plan → execute discipline under the superpowers skills; (f) a per-user Claude auto-memory directory for persistent facts across sessions.

---

## 0. System overview — what the pieces do together

The system has **five cooperating layers**. Each layer has one job. Do not collapse them.

| Layer | Role | Where it lives | Failure mode if missing |
|---|---|---|---|
| **1. Briefing** | Immutable source-of-truth for *why* the project exists. | `docs/briefing-files/` | Agent invents goals. |
| **2. North-star (CLAUDE.md / AGENTS.md)** | In-context operating rules re-loaded on every turn. | Repo root | Agent drifts from non-negotiables. |
| **3. Agent memory (persistent session state)** | What is done / decided / blocked / versioned. | `docs/agent-memory/` | Compaction erases progress; agent redoes work. |
| **4. Spec → Plan (superpowers)** | Design docs per subsystem + ordered implementation plan. | `docs/superpowers/{specs,plans}/` | Agent builds unscoped, unordered mush. |
| **5. Code intelligence (GitNexus) + hooks** | Impact analysis, change detection, safety gates. | `.gitnexus/`, `.claude/{hooks,settings.json,skills}` | Agent breaks callers silently; destructive Bash slips through. |

On top of these, an **MCP plane** (GitNexus, Context7, GitHub, Chrome DevTools, Playwright, Supabase, Sourcegraph) extends the agent with live capabilities, and the **superpowers skill pack** supplies process discipline (brainstorm, TDD, plan, execute, code-review, verification).

A **Claude auto-memory** directory (`~/.claude/projects/<slug>/memory/`) persists user/project/feedback/reference facts across conversations. This is global to the machine, not the repo; the agent writes entries there when user preferences or non-obvious project facts surface.

---

## 1. Target directory layout (what to create)

```
<target-repo>/
  CLAUDE.md                          # north-star briefing (layer 2)
  AGENTS.md                          # GitNexus MUST/NEVER rules (layer 2, auto-managed section)
  README.md
  RUNBOOK.md                         # what runs, what is stubbed, next-N-hour priorities
  .gitignore                         # must include .gitnexus/lbug, .claude/settings.local.json
  .claude/
    settings.json                    # hooks config (committed)
    settings.local.json              # per-machine permissions (gitignored)
    hooks/
      block_dangerous_bash.sh        # PreToolUse Bash safety gate
      log_commit_to_progress.sh      # PostToolUse git-commit auto-logger
      remind_tests_on_stop.sh       # Stop hook reminding to run tests
    skills/
      gitnexus/                      # six GitNexus skills (see §7)
        gitnexus-cli/SKILL.md
        gitnexus-exploring/SKILL.md
        gitnexus-debugging/SKILL.md
        gitnexus-refactoring/SKILL.md
        gitnexus-impact-analysis/SKILL.md
        gitnexus-guide/SKILL.md
  .gitnexus/                         # GitNexus index (generated; lbug file is the DB)
    meta.json
    lbug                             # gitignored (large binary)
  docs/
    briefing-files/                  # immutable inputs
    agent-memory/
      PROGRESS.md                    # append-only, one bullet per milestone/commit
      DECISIONS.md                   # ADR-lite, one per architectural decision
      BLOCKERS.md                    # append-only, one per defer/stub
      VERSIONS.md                    # pinned versions for every dep
      STACK.md                       # one-pager: responsibility → component → why
      SECRETS.md                     # rotation procedure, never the secrets
      PERF.md                        # baseline numbers
      FRONTEND_NOTES.md              # (optional) per-area notes
    superpowers/
      specs/
        YYYY-MM-DD-NN-<subsystem>-design.md
      plans/
        YYYY-MM-DD-<milestone>-plan.md
    KNOWLEDGE_MGMT_SETUP_SPEC.md     # this file (so future agents find it)
```

Notes:
- The **briefing-files** directory is project-specific (memos, PRDs, research). Do not invent contents; copy what the user supplies.
- The **agent-memory** directory is the single durable survival kit across compactions. Every file has a fixed format (§3).
- `docs/superpowers/specs` and `docs/superpowers/plans` are **the output** of the `superpowers:writing-plans` and domain design skills. One spec per subsystem, one plan per milestone.

---

## 2. `CLAUDE.md` — the north-star briefing

`CLAUDE.md` is reloaded into context on every turn. Keep it ≤400 lines. It MUST contain:

1. **Read this first** block listing the authoritative briefing files by path, so the agent can rehydrate them on demand.
2. **Session-local memory** block listing `PROGRESS.md`, `DECISIONS.md`, `BLOCKERS.md`, `VERSIONS.md`, `STACK.md` with one-line purpose each.
3. **Non-negotiables** — 10-15 numbered rules that override defaults. Typical entries: "TDD: failing test → implement → green → refactor", "Commit per subsystem", "If blocked >30 min, stub with `TODO(blocker)`, log to BLOCKERS.md, continue", plus project-specific architectural invariants (e.g. "source X overrides source Y where they disagree", "never mutate the graph directly; open a ticket").
4. **Folder structure** target — even if not all dirs exist yet, pin the canonical shape.
5. **Build order** — numbered list of milestones. This is what the agent resumes from after compaction.
6. **Acceptance criteria** — testable. If a named test command passes (`make test`, `pnpm test`, `cargo test`, `go test ./...`, etc.), the milestone is done. If not, it is not.
7. **Operating rules (quick reference)** — package managers, language versions, commit cadence, logging rules.
8. **Auto-managed GitNexus block** delimited by `<!-- gitnexus:start -->` / `<!-- gitnexus:end -->` containing the MUST-do/NEVER-do rules (see §5). This block is identical in `CLAUDE.md` and `AGENTS.md` so both file-name conventions work.

**Template structure:**

```markdown
# <Project> — <one-line codename/goal>
> Project codename: **<name>**. Goal: <one sentence>. This file is the **north-star briefing** that must survive context compaction. Read top-to-bottom before any non-trivial action.

## Read this first
Primary briefing documents (authoritative):
- `docs/briefing-files/<MEMO_V1>.md` — <what>
- `docs/briefing-files/<MEMO_V2_ADDENDUM>.md` — **<later version> overrides <earlier> where they disagree**
- `docs/briefing-files/<BRIEF>.md` — <scope + acceptance>

Session-local memory:
- `docs/agent-memory/PROGRESS.md` — what's done, what's next (update after every major step)
- `docs/agent-memory/DECISIONS.md` — architectural & stack decisions with rationale
- `docs/agent-memory/BLOCKERS.md` — append-only log of blockers & workarounds
- `docs/agent-memory/VERSIONS.md` — pinned versions for every dep
- `docs/agent-memory/STACK.md` — one-pager of the current stack

## Non-negotiables (do not drift)
1. …

## Folder structure (target) …
## Build order …
## N Acceptance criteria …
## Operating rules (quick reference) …

<!-- gitnexus:start -->
# GitNexus — Code Intelligence
… (see §5)
<!-- gitnexus:end -->
```

---

## 3. `docs/agent-memory/` — the survival kit

Every file here has a **fixed format**. Violating the format breaks the auto-loggers and the rehydration protocol.

### 3.1 `PROGRESS.md` (append-only)

Purpose: the one file an agent reads after rehydrating CLAUDE.md to know where it stopped.

Format:
```
# PROGRESS
Append-only log. One bullet per completed subsystem or milestone. Timestamp in UTC.
Format: `[YYYY-MM-DD HH:MM] <tag> <message>` where tag ∈ {DONE, START, STUB, FIX, REFACTOR, NOTE, COMMIT}.
---
- [2026-04-19 02:10] START Session began. Briefing absorbed.
- [2026-04-19 11:30] DONE Docker Compose stack up: all services healthy.
- [2026-04-19 11:58] COMMIT [commit 7f41754] feat(promotion): promote all graph-bound predicates
```

Rules:
- Never edit or delete existing lines. Only append.
- `COMMIT` lines are appended **automatically** by the `log_commit_to_progress.sh` hook (§6.2). The agent does not write them by hand.
- `DONE` / `STUB` / `FIX` lines are written by the agent at end-of-subsystem.

### 3.2 `DECISIONS.md` — ADR-lite

One entry per architectural decision, separator `---`. Format:
```
## ADR-NNNN — <one-line title>
**Status:** {Proposed, Accepted, Superseded by ADR-NNNN} · **Date:** YYYY-MM-DD
**Context:** <why the decision was needed>
**Decision:** <what was chosen>
**Consequences:** <what this implies, especially tradeoffs>
---
```

Rules:
- Once Accepted, never edit — supersede with a new ADR and update the old one's Status line.
- Reference ADR numbers from code comments sparingly; reference from `BLOCKERS.md` when a blocker forces a decision.

### 3.3 `BLOCKERS.md` — append-only

One entry per deferred/stubbed subsystem. Format:
```
## YYYY-MM-DD — <subsystem>: <one-line what was blocked>
**What:** …
**What I found:** …
**What I tried:** …
**Resolution / stub:** …
**Follow-up (post-<milestone>):** …
---
```

Rules:
- Created only when the agent hits the 30-minute wall on a subsystem (§8).
- Every stub in code carries `TODO(blocker: <YYYY-MM-DD>)` cross-referencing this file.
- When resolved, the entry is amended with a resolution note **below** the original content (append, do not rewrite).

### 3.4 `VERSIONS.md`

Table of pinned versions: Python packages, Node packages, Docker images. Updated whenever a dep lands. The agent consults this before running `uv sync` / `pnpm install`.

### 3.5 `STACK.md`

A single table: **Responsibility → Component → Why**, plus one ASCII data-flow diagram. This is the answer to "what's in this repo" in 60 seconds.

### 3.6 `SECRETS.md`, `PERF.md`, `FRONTEND_NOTES.md`, `COMPLIANCE.md`

Optional per-area notes. Use sparingly — one file per area, not one per feature.

- `SECRETS.md` — inventory + rotation cadence + incident response (never actual secrets).
- `PERF.md` — baseline numbers (p50/p95/p99), load-test command, date, hardware.
- `FRONTEND_NOTES.md` — framework-version-audit log (see §14.11).
- `COMPLIANCE.md` — compliance-framework × status × evidence table (see §18.4). Required as soon as the project touches regulated data.

---

## 4. `docs/superpowers/` — spec → plan → execute

Driven by the **superpowers skill pack** (see §7). The flow is:

1. **Brainstorm** (`superpowers:brainstorming`) — only for creative/open-ended work. Skip if the user hands a clear spec.
2. **Spec** — write one design doc per subsystem into `docs/superpowers/specs/YYYY-MM-DD-NN-<subsystem>-design.md`. Each spec: Problem, Non-goals, Architecture (ASCII diagram), Interfaces, Tests.
3. **Plan** (`superpowers:writing-plans`) — write a single ordered implementation plan into `docs/superpowers/plans/YYYY-MM-DD-<milestone>-plan.md`. Each step: what, 30-90 min budget, tests to go green, commit at end.
4. **Execute** (`superpowers:executing-plans` in a new session, or `superpowers:subagent-driven-development` for parallelizable work).
5. **Verify** (`superpowers:verification-before-completion`) — never claim done without evidence.
6. **Review** (`superpowers:requesting-code-review` / `code-review:code-review`) before merge.

Each spec/plan filename is date-prefixed so the directory reads as a timeline.

---

## 5. `AGENTS.md` and the GitNexus block

`AGENTS.md` contains **only** the auto-managed GitNexus block (with the same `<!-- gitnexus:start -->` / `<!-- gitnexus:end -->` delimiters) so non-Claude agents that honour `AGENTS.md` get the same rules. The identical block is embedded inside `CLAUDE.md`.

Canonical content of the block (substitute `<repo>` and the stats from `.gitnexus/meta.json`):

```markdown
<!-- gitnexus:start -->
# GitNexus — Code Intelligence

This project is indexed by GitNexus as **<repo>** (<N> symbols, <M> relationships, <K> execution flows). Use the GitNexus MCP tools to understand code, assess impact, and navigate safely.

> If any GitNexus tool warns the index is stale, run `npx gitnexus analyze` in terminal first.

## Always Do
- **MUST run impact analysis before editing any symbol.** Run `gitnexus_impact({target: "symbolName", direction: "upstream"})` and report blast radius to the user.
- **MUST run `gitnexus_detect_changes()` before committing** to verify scope.
- **MUST warn the user** if impact analysis returns HIGH or CRITICAL risk.
- Use `gitnexus_query({query: "concept"})` to find execution flows instead of grepping.
- Use `gitnexus_context({name: "symbolName"})` for full symbol context.

## Never Do
- NEVER edit a function/class/method without running `gitnexus_impact` on it first.
- NEVER ignore HIGH or CRITICAL risk warnings.
- NEVER rename symbols with find-and-replace — use `gitnexus_rename`.
- NEVER commit without running `gitnexus_detect_changes()`.

## Resources
| Resource | Use for |
|----------|---------|
| `gitnexus://repo/<repo>/context`   | Codebase overview, check index freshness |
| `gitnexus://repo/<repo>/clusters`  | All functional areas |
| `gitnexus://repo/<repo>/processes` | All execution flows |
| `gitnexus://repo/<repo>/process/{name}` | Step-by-step execution trace |

## CLI
| Task | Read this skill file |
|------|---------------------|
| "How does X work?"           | `.claude/skills/gitnexus/gitnexus-exploring/SKILL.md` |
| "What breaks if I change X?" | `.claude/skills/gitnexus/gitnexus-impact-analysis/SKILL.md` |
| "Why is X failing?"          | `.claude/skills/gitnexus/gitnexus-debugging/SKILL.md` |
| Rename / extract / refactor  | `.claude/skills/gitnexus/gitnexus-refactoring/SKILL.md` |
| Tools, schema reference      | `.claude/skills/gitnexus/gitnexus-guide/SKILL.md` |
| Index / status / wiki CLI    | `.claude/skills/gitnexus/gitnexus-cli/SKILL.md` |
<!-- gitnexus:end -->
```

**Initial indexing:** after first install, run `npx gitnexus analyze` at repo root. This creates `.gitnexus/lbug` (the graph DB) and `.gitnexus/meta.json`. Add `.gitnexus/lbug` to `.gitignore`; commit `.gitnexus/meta.json`.

---

## 6. Hooks — safety, auto-logging, reminders

Hooks run in the Claude Code harness, not inside Claude. They are the only component that can enforce discipline **unconditionally**. Three hooks are required.

### 6.1 `.claude/settings.json`

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": ".claude/hooks/block_dangerous_bash.sh", "timeout": 5, "statusMessage": "Safety gate…" }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": ".claude/hooks/log_commit_to_progress.sh", "timeout": 5, "if": "Bash(git commit *)" }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          { "type": "command", "command": ".claude/hooks/remind_tests_on_stop.sh", "timeout": 5 }
        ]
      }
    ]
  }
}
```

Commit this file. Per-machine permission overrides go to `.claude/settings.local.json` (gitignored).

### 6.2 `.claude/hooks/block_dangerous_bash.sh` (PreToolUse, Bash)

Reads the hook JSON from stdin, extracts `.tool_input.command`, and denies if it matches any of:
`rm -rf /`, `rm -rf ~`, `rm -rf /*`, `rm -fr /`, `git push --force`, `git push -f `, `DROP DATABASE`, `TRUNCATE DATABASE`, `docker compose down -v` / `docker-compose down -v`, `:(){ :|:& };:`, `mkfs.`, `dd if=/dev/zero of=/dev/`, `chmod -R 777 /`.

On deny, emits `{hookSpecificOutput:{hookEventName:"PreToolUse",permissionDecision:"deny",permissionDecisionReason:"..."}}`. On allow, emits the same object with `"allow"`. Implementation: pure Bash + `jq`. Must `chmod +x`.

### 6.3 `.claude/hooks/log_commit_to_progress.sh` (PostToolUse, Bash)

After a successful `git commit`, appends one line to `docs/agent-memory/PROGRESS.md`:
```
- [YYYY-MM-DD HH:MM] COMMIT [commit <short-sha>] <commit subject>
```
Idempotent: skip if `[commit <sha>]` already appears. Silent on non-commit commands. Non-fatal on any error (exit 0).

### 6.4 `.claude/hooks/remind_tests_on_stop.sh` (Stop)

On session stop, if `git status --porcelain` shows uncommitted changes under the code directories, emit a `systemMessage` reminding to run the test suite before stopping. Does not block stop (`continue: true`).

Adapt the path globs and test command per target-repo language:

| Language / stack | Path globs | Test command |
|---|---|---|
| Python | `src/**/*.py`, `tests/**/*.py` | `pytest -q` |
| Node / TypeScript | `src/**/*.{ts,tsx}`, `tests/**/*.{ts,tsx}` | `pnpm test` |
| Go | `**/*.go` | `go test ./...` |
| Rust | `src/**/*.rs`, `tests/**/*.rs` | `cargo test` |
| Java / Kotlin | `src/main/**`, `src/test/**` | `./gradlew test` |

---

## 7. MCP servers & skills to provision

### 7.1 MCP servers (configured at the harness level, not in-repo)

| MCP | Purpose | When to use |
|---|---|---|
| **gitnexus** | Code-graph intelligence (impact, context, rename, detect_changes). | Before every non-trivial edit; before commit. |
| **context7** | Live library documentation. | Anytime a library/framework/SDK/CLI is involved. Prefer over web search for docs. |
| **github** | Read/write GitHub (issues, PRs, checks). Auth via `github__authenticate`. | Opening/commenting PRs and issues. |
| **chrome-devtools** | Chrome automation for perf, a11y, LCP debugging, screenshots. | Frontend verification, performance work. |
| **playwright** | Headless browser automation. | E2E smoke tests, visual checks. |
| **sourcegraph** | Cross-repo code search. | Looking for patterns beyond this repo. |
| **supabase** (optional) | Postgres/Auth/Edge functions. | Only if the target uses Supabase. |
| **prisma** (optional) | Prisma migrate / studio. | Only if the target uses Prisma. |

Document in the target repo's `RUNBOOK.md` which MCPs are required and which are optional.

### 7.2 Skills (superpowers + gitnexus + domain)

**Superpowers process skills (required):**
- `superpowers:using-superpowers` — auto-loaded session start.
- `superpowers:brainstorming` — before any creative/open-ended work.
- `superpowers:writing-plans` — turning a spec into an ordered plan.
- `superpowers:executing-plans` — executing a written plan in a separate session.
- `superpowers:subagent-driven-development` — executing plans via parallel subagents.
- `superpowers:test-driven-development` — TDD discipline.
- `superpowers:systematic-debugging` — any bug or test failure.
- `superpowers:verification-before-completion` — never claim done without evidence.
- `superpowers:requesting-code-review` / `code-review:code-review` — before merging.
- `superpowers:receiving-code-review` — when absorbing feedback.
- `superpowers:finishing-a-development-branch` — deciding merge/PR/cleanup.
- `superpowers:using-git-worktrees` — for isolated multi-branch work.
- `superpowers:dispatching-parallel-agents` — 2+ independent tasks.
- `superpowers:writing-skills` — when extending the skill library.

**GitNexus skills (required, installed under `.claude/skills/gitnexus/`):**
- `gitnexus-exploring` — "how does X work?"
- `gitnexus-impact-analysis` — "what breaks if I change X?"
- `gitnexus-debugging` — "why is X failing?"
- `gitnexus-refactoring` — rename/extract/split/move.
- `gitnexus-pr-review` — pre-merge risk assessment.
- `gitnexus-cli` — index/status/clean/wiki.
- `gitnexus-guide` — tools + schema reference.

**Domain / optional skills:**
- `graphify` — input → knowledge graph (HTML + JSON + audit). Trigger `/graphify`. Useful for turning briefing files into an explorable graph before planning.
- `frontend-design:frontend-design` — when building UI.
- `chrome-devtools-mcp:*` — perf/a11y.
- `claude-api` — when the repo builds on Claude.
- `supabase-postgres-best-practices` — Postgres design.
- `skill-creator:skill-creator` — to add repo-local skills.
- `claude-md-management:claude-md-improver` — periodic CLAUDE.md audit.
- `claude-md-management:revise-claude-md` — pull session learnings back into CLAUDE.md.
- `fewer-permission-prompts` — scan transcripts and allowlist common read-only commands.
- `update-config` — any `settings.json` change.
- `loop` / `schedule` — recurring and scheduled work.
- `ralph-loop:ralph-loop` — long-horizon self-driven iteration.

Skills are triggered by the Skill tool with the exact name. Never invent names.

---

## 8. Operating procedures — the rules the agent follows every hour

### 8.1 Session start (rehydration)

1. Read `CLAUDE.md` top to bottom.
2. Read `docs/agent-memory/PROGRESS.md` — find the last non-COMMIT `DONE`/`STUB` line; that is the current milestone.
3. Read `docs/agent-memory/BLOCKERS.md` — know what is deferred.
4. Read the most recent plan in `docs/superpowers/plans/` — resume from the first non-completed step.
5. If any GitNexus tool reports "stale index", run `npx gitnexus analyze` and re-read `meta.json` stats.
6. Only then do anything else.

### 8.2 Before editing any symbol

- Run `mcp__gitnexus__impact({ target: "<symbol>", direction: "upstream" })`.
- Report blast radius (direct callers, affected processes, risk level) to the user.
- If risk is **HIGH** or **CRITICAL**, warn and request confirmation (or, in auto mode, only proceed on low-risk changes).
- For rename/extract, use `mcp__gitnexus__rename` — never find-and-replace.
- For architecture exploration, prefer `mcp__gitnexus__query` / `mcp__gitnexus__context` over grep.

### 8.3 TDD per subsystem

1. Write failing test (`test_<subsystem>.py` or `<subsystem>.test.ts`).
2. Implement minimum code to green.
3. Refactor under green.
4. Run the full suite (not just the targeted test) as a regression gate.
5. Commit.

### 8.4 Commit cadence

- One commit per subsystem (not per file).
- Meaningful message following repo convention: `<type>(<scope>): <subject>`.
- Before commit: run `mcp__gitnexus__detect_changes()` and confirm scope matches intent.
- The `log_commit_to_progress.sh` hook appends the commit to `PROGRESS.md`. The agent does **not** also write a `COMMIT` line by hand.
- Optional: write a `DONE` line to `PROGRESS.md` when closing a subsystem milestone larger than one commit.

### 8.5 Blocker protocol

If a subsystem eats more than 30 minutes with no forward progress:

1. Stub the offending piece with `TODO(blocker: YYYY-MM-DD)` and the simplest honest fallback.
2. Append a full entry to `BLOCKERS.md` (see format in §3.3).
3. Keep building the rest of the plan.
4. The plan's final step verifies the stub is acceptable for this milestone (often yes) or opens the next milestone to resolve it.

### 8.6 Verification before completion

Before ever saying "done" or "works":

- Run the commands the milestone's acceptance criteria calls for (`make test`, `pytest -q`, `pnpm test`, `pnpm build`, `cargo test`, `go test ./...`, etc.).
- Paste or summarize the output.
- For UI changes: start the dev server, exercise the feature in a browser (Chrome DevTools or Playwright MCP), confirm the golden path and the top two edge cases.
- Only then update `PROGRESS.md` with `DONE`.

### 8.7 Context-compaction recovery

The harness may compact the conversation at any point. The survival protocol:

1. Reload `CLAUDE.md`.
2. Reload `PROGRESS.md` — pick up at the last `DONE` / `STUB`.
3. Reload `BLOCKERS.md`.
4. If mid-plan, reload the current plan file and resume at the first non-completed step.
5. If mid-edit, run `git diff` to see uncommitted work and `mcp__gitnexus__detect_changes()` to see affected scope.

### 8.8 Long-horizon loops

For autonomous runs that must span hours:

- Use `superpowers:executing-plans` in fresh sessions per milestone — each session rehydrates via §8.1.
- Use `loop` skill for dynamic-pacing self-ticks (not <300 s — pick under 270 s to stay in cache, or over 1200 s to amortize a cache miss).
- Use `schedule` skill / CronCreate for cron-driven recurring agents.
- Use `superpowers:using-git-worktrees` for parallel branches without cross-talk.
- Use `superpowers:dispatching-parallel-agents` for 2+ independent tasks.

### 8.9 Claude auto-memory (global, per machine)

Separate from the repo. Claude writes persistent facts into `~/.claude/projects/<slug>/memory/`:

- `MEMORY.md` (index, ≤200 lines, one-line entries).
- Individual memory files with frontmatter (`name`, `description`, `type` ∈ {user, feedback, project, reference}).

Rules (from the harness system prompt, summarized):
- Save user facts, feedback (corrections AND validated choices), project facts with Why/How-to-apply, and pointers to external systems.
- Do NOT save: derivable code patterns, git history, debug recipes, CLAUDE.md content, ephemeral task state.
- Update or remove memories that go stale. Verify before acting on memory claims about specific files/functions.

The setup-agent does not need to write any memory files proactively; it just needs to make sure the directory exists and not confuse memory with repo-local knowledge.

---

## 9. `RUNBOOK.md` — operational single-pager

Required sections:

1. **What runs** — services, ports, environment variables.
2. **How to bring it up** — `make up` / `pnpm dev` / etc.
3. **What is stubbed** — every `TODO(blocker)` cross-referenced to `BLOCKERS.md`.
4. **Known issues** — short list, linked to issues.
5. **Next N hours priorities** — the top 3-5 tasks queued up.

This is what a fresh agent (or a human on-call) reads after `CLAUDE.md` and `PROGRESS.md`.

---

## 10. Setup checklist — the setup-agent's execution plan

Execute in order. Commit after each step with a meaningful message.

1. **Create base layout.** Ensure `docs/{briefing-files,agent-memory,superpowers/{specs,plans}}` and `.claude/{hooks,skills/gitnexus}` exist. Commit.
2. **Seed agent-memory files.** Create `PROGRESS.md`, `DECISIONS.md`, `BLOCKERS.md`, `VERSIONS.md`, `STACK.md` with the headers from §3 but empty bodies. Commit.
3. **Write `CLAUDE.md`.** Use the template in §2. Populate non-negotiables from the target project's brief. Leave the GitNexus block as a placeholder until step 7. Commit.
4. **Write `AGENTS.md`.** Contains only the `<!-- gitnexus:start -->…<!-- gitnexus:end -->` placeholder for now. Commit.
5. **Install hook scripts.** Copy/write the three scripts into `.claude/hooks/`, `chmod +x`. Write `.claude/settings.json` from §6.1. Adapt path globs in `remind_tests_on_stop.sh` to the target's language (Python / Node / Go / etc.). Commit.
6. **Install GitNexus skill files.** Six files under `.claude/skills/gitnexus/*/SKILL.md`. Copy from the official GitNexus skill pack (or a reference repo where they already exist) — they are stack-agnostic. Commit.
7. **Index with GitNexus.** Run `npx gitnexus analyze` at repo root. Read `.gitnexus/meta.json` for the `(nodes, edges, processes)` stats. Fill those numbers into the GitNexus block in both `CLAUDE.md` and `AGENTS.md`. Add `.gitnexus/lbug` to `.gitignore`. Commit.
8. **Provision MCP servers.** Configure at harness level: gitnexus, context7, github, chrome-devtools, playwright, sourcegraph (+ optional supabase/prisma). Verify each responds. List required ones in `RUNBOOK.md`.
9. **Verify hooks fire.** Make a no-op commit; confirm `PROGRESS.md` gained a `COMMIT` line. Attempt a dangerous command like `rm -rf ~/tmp_test_never` (non-existent dir) in Bash — the PreToolUse hook must block it. Revert/cleanup. Commit.
10. **Record versions.** Run the language toolchain's version commands (`python --version`, `node --version`, `uv --version`, `pnpm --version`, Docker image tags from `docker-compose.yml`) and populate `VERSIONS.md`. Commit.
11. **Draft the first plan.** Invoke `superpowers:writing-plans` against the user's current goal. Output `docs/superpowers/plans/YYYY-MM-DD-<milestone>-plan.md`. Commit.
12. **Write `RUNBOOK.md`.** Per §9. Commit.
13. **Session handoff.** Write a `DONE` line to `PROGRESS.md` summarizing the setup. Open the first plan for the next (or same) agent.

Acceptance for the setup itself:

- `git log` shows 10+ meaningful commits from steps 1-13.
- `CLAUDE.md`, `AGENTS.md`, `RUNBOOK.md`, all six `docs/agent-memory/*.md` files, `.claude/settings.json`, three hook scripts, six GitNexus `SKILL.md` files, and `.gitnexus/meta.json` all exist.
- A trivial commit triggers a new line in `PROGRESS.md` (hook verified).
- `mcp__gitnexus__context` returns non-empty stats for the repo.
- `docs/superpowers/plans/` has at least one plan ready to execute.

---

## 11. Failure modes and how the system corrects for them

| Risk | Mitigation |
|---|---|
| Agent forgets project goals after compaction | `CLAUDE.md` reloaded every turn; `PROGRESS.md` + `BLOCKERS.md` rehydrate state. |
| Agent breaks callers silently | `gitnexus_impact` MUST precede edits; `gitnexus_detect_changes` MUST precede commits. |
| Agent runs `rm -rf /` or `git push --force` | PreToolUse hook blocks the known dangerous forms. |
| Agent claims "done" without running tests | `verification-before-completion` skill + Stop hook reminder. |
| Commit history drifts from progress | PostToolUse hook auto-appends every commit to `PROGRESS.md`. |
| Agent stalls indefinitely on a hard subsystem | 30-min blocker rule → stub + `BLOCKERS.md` entry + move on. |
| Subsystems grow into one unmanageable mega-commit | Commit-per-subsystem rule + superpowers plan steps capped at 30-90 min. |
| Library guidance is out of date | Prefer Context7 MCP over web search or training-data recall. |
| Skills invented from memory don't actually exist | Only invoke skills listed in the live skill index; never guess names. |
| Claude auto-memory drifts from reality | Verify file/function existence before acting on a memory; update stale entries on encounter. |
| Docker image or dep pin looks fine but container/process is unhealthy | Read the container's / process's **own** error log before the orchestrator's logs — OS-layer warnings are often red herrings. Record the failure + resolution in `BLOCKERS.md` + `RUNBOOK.md § Known issues`. |

---

## 11.1 Known-good pin discipline

A common trap: the `latest` or `-alpine` tag of an infra image breaks in subtle ways that don't reproduce outside the full stack. **Pin every infra image to a specific known-good tag and commit a short rationale.**

Rules (stack-agnostic):

1. **Never use `latest`** for infra images in `docker-compose.yml` / Helm / anything reproducible. Pin to a specific version.
2. **Prefer release lines over bleeding-edge.** LTS / stable trains (Postgres 16, Node 20, Python 3.12) are almost always the right default.
3. **Be suspicious of `-alpine` variants** for data stores. Alpine's musl libc + non-standard entrypoints are a frequent source of "works everywhere except here" failures.
4. **When an image fails to become healthy**, read its own error log (inside the container) **before** the orchestrator log. OS-layer warnings (e.g. `get_mempolicy: Operation not permitted`) are usually noise; the app log has the real reason.
5. **Mount log directories as host volumes** in dev compose so the next agent can `cat` them without exec'ing into a dead container.
6. **When a breaking version contract change ships** (e.g. a major split-env-var rename), document it in `BLOCKERS.md` the first time it bites, even if already resolved — the next agent upgrading across majors will thank you.

Record the pinned tags + rationale in `VERSIONS.md § Infrastructure images`.

---

## 12. What NOT to install

- Do not add memory files for code patterns, architecture, or git history — those belong in `STACK.md` / `DECISIONS.md` / `git log`, not Claude auto-memory.
- Do not commit `.claude/settings.local.json` (per-machine permissions).
- Do not commit `.gitnexus/lbug` (large binary; regenerate locally).
- Do not add speculative skills "in case we need them later" — skill bloat hurts skill selection accuracy.
- Do not copy briefing-files from another project — briefings are project-specific.
- Do not write `COMMIT` lines by hand in `PROGRESS.md` — the hook does it.

---

## 13. References — what is safe to copy verbatim vs what is project-specific

**Safe to copy verbatim across any repo** (stack-agnostic):

- `.claude/settings.json` (the hook wiring)
- `.claude/hooks/block_dangerous_bash.sh`
- `.claude/hooks/log_commit_to_progress.sh`
- `.claude/hooks/remind_tests_on_stop.sh` (adjust path globs + test command per §6.4)
- `.claude/skills/gitnexus/*/SKILL.md` (six files)
- `docs/agent-memory/*.md` **headers only** — empty bodies
- Structure (not contents) of `CLAUDE.md` §Read-this-first, §Non-negotiables, §Build-order, §Operating-rules, §GitNexus-block
- Structure of `AGENTS.md` (just the GitNexus block)
- This spec itself (`docs/KNOWLEDGE_MGMT_SETUP_SPEC.md`)

**Always project-specific** — must be written fresh against the target repo's actual goals:

- `docs/briefing-files/*`
- `docs/agent-memory/*.md` bodies (PROGRESS, DECISIONS, BLOCKERS, VERSIONS, STACK, SECRETS, PERF, COMPLIANCE)
- `docs/superpowers/specs/*`
- `docs/superpowers/plans/*`
- `RUNBOOK.md` contents
- Non-negotiables numbered list in `CLAUDE.md`
- Architecture folder structure under `CLAUDE.md § Folder structure`

---

## 14. Concrete patterns — stack-agnostic templates

This section distils working templates. Each pattern names 2-3 language/stack alternatives so the setup-agent picks what fits.

### 14.1 `.gitignore` — must-have entries (language-pick per repo)

```
# === Language build / cache (pick what applies) ===
# Python:         __pycache__/ .venv/ .pytest_cache/ .mypy_cache/ .ruff_cache/ .uv/ *.egg-info/ dist/ build/
# Node / TS:      node_modules/ .next/ .turbo/ *.tsbuildinfo .parcel-cache/
# Go:             bin/ vendor/ *.test
# Rust:           target/
# Java / Kotlin:  build/ .gradle/ out/

# === Env (NEVER commit real envs) ===
.env
.env.local
.env.*.local

# === OS / IDE ===
.DS_Store
.vscode/
.idea/
*.swp

# === Local Docker volumes / data ===
data/
volumes/

# === Secrets (safety net; hooks + scanners are primary) ===
*.pem
*.key
secrets/

# === GitNexus index (regenerable; commit only meta.json) ===
.gitnexus/*
!.gitnexus/meta.json

# === Generated code (schemas-as-source-of-truth; ignore generated clients) ===
# Add your codegen output dirs here
```

Critical: `.env` is gitignored but `.env.example` **is committed** with placeholder values. A `scripts/gen-secrets.sh` (or equivalent) detects placeholder sentinels (e.g. `please_change_me_*`, `0000…`, `changeme`) and substitutes strong randoms without touching real values.

### 14.2 `Makefile` — operator entry-point pattern

A single `Makefile` at the repo root with one-line help built via `awk`:

```make
SHELL := /bin/bash
.SHELLFLAGS := -eu -o pipefail -c
.ONESHELL:
.DEFAULT_GOAL := help

.PHONY: help
help: ## Show this help
	@awk 'BEGIN {FS = ":.*?## "}; /^[a-zA-Z_-]+:.*?## /{printf "  \033[36m%-20s\033[0m %s\n", $$1, $$2}' $(MAKEFILE_LIST)
```

Canonical target set (add/drop per language — the same verbs apply whether it's `make`, `just`, `task`, `npm run`, or `nx`):

- **Environment:** `env` (copy `.env.example → .env`), `secrets`, `secrets-merge`.
- **Install:** `install` (+ per-service variants when the repo is a monorepo).
- **Infrastructure:** `up`, `down`, `logs`, `ps` (when Docker Compose is in use).
- **Schema:** `db-migrate`, `db-revision`, `db-rollback` (when there's a persistent store with migrations).
- **Codegen:** `codegen` (when schemas-as-source generate clients/types).
- **Seed / run:** `seed`, `dev`, one target per long-running process.
- **Quality:** `lint`, `fmt`, `typecheck`, `test`, `test-integration`, `test-all`.
- **Cleanup:** `clean`, `reset`.
- **Prod (optional):** `prod-build`, `prod-up`, `prod-logs`.

Every target marked `.PHONY` (for `make`). Every public target carries a `## description` for the help output. The agent always has one entrypoint — `make help` or `just --list` — and never needs to grep the toolchain to know what's runnable.

### 14.3 Local-infra compose + CI pattern

**Dev compose (if infra is containerized):** `docker-compose.yml` at root. Services for infra only (DB, caches, queues, object store, observability backends). Application processes live behind a `profiles: [full]` (or similar) flag so the default `up` brings infra only — the agent runs app processes natively for fast reload.

**Prod compose (optional):** `docker-compose.prod.yml` extends dev with `deploy.resources`, `env_file=.env.prod`, and a TLS reverse proxy (Caddy, Traefik, nginx, or managed load balancer if the deploy target has one).

**CI** (`.github/workflows/ci.yml` for GitHub, `.gitlab-ci.yml`, etc.) — minimum three-stage pattern:

1. **Unit** — lint + typecheck + unit tests. No services. Fast (<5 min). Runs on every push/PR.
2. **Integration** — `services:` block (or equivalent) spins up the same infra images as dev compose, with healthchecks. Runs migrations + integration tests gated by an env flag like `INTEGRATION=1`. Needs the unit stage green.
3. **Build** — compile / bundle / type-emit output as a deployable artefact (Docker image, tarball, wheel). Runs in parallel with integration.

Concurrency: cancel-in-progress per branch/PR (`group: ci-${ref}`, `cancel-in-progress: true`) so stale runs don't block queued ones.

Security invariant: **never interpolate user-controlled inputs** (`github.event.*.body`, commit messages, issue titles, PR titles) into `run:` blocks. That's a command-injection vector. Put a one-line security note at the top of the CI file documenting what is and isn't interpolated.

### 14.4 Bootstrap scripts

Under `scripts/`, all `chmod +x`. A small stable set covers most repos:

- `scripts/gen-secrets.sh` — generates strong randoms (`openssl rand -base64 32`, `-hex 64`, UUIDs). Flags: `--merge-into <.env>` replaces only placeholder sentinels; plain invocation writes `.env.secrets` with `chmod 600`.
- `scripts/bootstrap-<service>.sh` — one per external system that requires first-run admin setup (observability, auth provider, object store bucket, etc.). Idempotent, writes keys to `.env.local`, output `chmod 600`.
- Any other one-shot "wire this external system" helper belongs in `scripts/`, not in application code.

### 14.5 Briefing-files `index.md`

`docs/briefing-files/index.md` is a short pointer file describing every briefing document by filename + purpose. Example:

```markdown
This index tells about which documentation files are needed for what
---
# Memo_v1.docx
Primary architecture memo.
---
# Memo_v2_Addendum.docx
Stack refresh; overrides v1 where they disagree.
---
# CODE_AGENT_BRIEF.md
Overnight-run scope + the acceptance criteria.
```

Purpose: when the agent re-hydrates from `CLAUDE.md`, the briefing-files index tells it **which** briefing file to pull on demand, instead of re-reading all of them blindly.

### 14.6 The per-subsystem spec template (`docs/superpowers/specs/`)

Filename: `YYYY-MM-DD-NN-<subsystem>-design.md` (NN orders the subsystems; the first is always `00-overview-design.md`).

Body:

```markdown
# Spec NN — <Subsystem>
**Status:** {Draft, Accepted, Superseded} · **Date:** YYYY-MM-DD · **Author:** <agent-id>

## Problem
<one paragraph>

## Non-goals
- <explicit non-goals; reduces scope creep>

## Architecture
<ASCII diagram where possible>

## Interfaces
<functions, classes, API endpoints, DB tables — signatures only>

## Tests
<which test files cover this subsystem + which assertions>
```

Spec 00 always carries the system-level ASCII diagram and a non-goals list scoped to the **overnight / initial-scope** window.

### 14.7 The per-sprint plan template (`docs/superpowers/plans/`)

Filename: `YYYY-MM-DD-<milestone>-plan.md`. Body opens with **guiding principles** (DRY / SoT / SRP, fail-closed, seams stay explicit, TDD-where-cheap, observability-by-default, commit-per-subtask, PROGRESS-after-commit, backwards-compatible). Then a numbered list of tasks. Each task section has:

```markdown
### SN.N — <one-line what>
**Problem.** <why this task exists>
**Design.** <what will change + where>
**Tests.** <test files + assertions that prove done>
**Config.** <env vars, flags, migrations>
**Acceptance.** <observable green-condition>
```

Each task sized for 30-90 minutes of focused work. One commit per task. The sprint closes with a test-green run + a `DONE` line in `PROGRESS.md`.

### 14.8 Testing-strategy doc (`specs/NN-testing-design.md`)

Every project needs one. Three tiers:

1. **Unit** — pure, fast, no external systems. Run on every save.
2. **Integration** — tagged with a language-idiomatic marker (`@pytest.mark.integration`, `// +build integration`, `describe.integration(...)`, etc.) and gated by an env flag (`PROJECT_INTEGRATION=1`). Hits real infrastructure spun via local compose or testcontainers.
3. **E2E / acceptance** — one test per explicit acceptance criterion from the project brief. Runs against the full stack (via `make test-integration` or the CI integration job).

Deferred-to-later bucket: performance, security / OWASP, chaos, accessibility. Name them as a non-goal in the testing spec so no reviewer has to re-ask.

### 14.9 `RUNBOOK.md` — canonical sections

Order matters (matches what a sleep-deprived human / fresh agent reads first):

1. **TL;DR** (3-6 bullets).
2. **What runs** — exact commands (`make up && make seed && make api && make web`).
3. **Live-verified golden path** — actual outputs/numbers observed.
4. **Sprint summary table(s)** — per-area outcomes.
5. **What is stubbed** — with why + when to lift.
6. **Known issues** — with workarounds.
7. **Next N hours** — prioritized queue.
8. **Pointers (compaction-survive)** — the file list future agents should read.
9. **Acceptance contract** — the tests that must still be green.

### 14.10 `.env.example` — committed template

Pattern: include **every** env var the code reads, with placeholder values and a `_README:` comment block at the top explaining how to populate real ones. Group by purpose, not by alphabetical order.

Canonical sections (add/drop per repo):

```
# ─── Application ───────────────────────────────────────────
APP_ENV=dev                        # dev | staging | prod
APP_API_KEY=please_change_me_32chars_min
LOG_LEVEL=INFO

# ─── Persistent stores ─────────────────────────────────────
# (one per store the app reads: primary DB, cache, queue, object store, …)
PRIMARY_DB_URL=postgresql://user:please_change_me@localhost:5432/app
CACHE_URL=redis://localhost:6379/0
OBJECT_STORE_ENDPOINT=http://localhost:9000
OBJECT_STORE_ACCESS_KEY=please_change_me
OBJECT_STORE_SECRET_KEY=please_change_me

# ─── External providers (optional, leave blank to no-op) ───
# Any provider API key the app consumes — LLMs, payments,
# maps, email, etc. App should tolerate missing keys gracefully.
PROVIDER_A_API_KEY=
PROVIDER_B_API_KEY=

# ─── Observability ─────────────────────────────────────────
TRACING_ENABLED=false
TRACING_ENDPOINT=
TRACING_API_KEY=
METRICS_PORT=9090

# ─── Feature flags ─────────────────────────────────────────
# Boolean flags default OFF; the app must ship with flags off
# returning a safe no-op path.
FEATURE_X_ENABLED=false

# _README:
# 1. `make env` copies this file to .env.
# 2. `make secrets-merge` replaces placeholder sentinels with strong randoms.
# 3. Fill provider keys from the respective dashboards.
# 4. Every variable here is inventoried in docs/agent-memory/SECRETS.md.
```

The `SECRETS.md` file in `docs/agent-memory/` lists each env var, its purpose, rotation cadence, and generator/source. That inventory is the single source of truth for "what secrets does this project use?".

### 14.11 Framework-notes convention

When the project sits on a framework that recently had a major version bump (Next.js 15→16, Django 4→5, Rails 7→8, Spring Boot 3→4, Go 1.N→1.N+1 with language changes, etc.), add a `docs/agent-memory/<AREA>_NOTES.md` recording:

- Which conventions the project actually uses (audited via typecheck + build commands).
- Which old convention the agent might default to from training data and why it now breaks.
- Audit evidence (command + date + version observed).
- Known gotchas specific to the new version.

Re-audit on each major release. This file is how the agent stops confidently writing last-year's idioms.

### 14.12 Onboarding doc for reference clients

If the project has a reference client, write `docs/<CLIENT>_ONBOARDING.md` as a week-by-week checklist (week 0 pre-kickoff → week N acceptance walkthrough). Include:

- Legal / compliance table (regulations × status × evidence).
- Environment bootstrap commands.
- Client-specific ontology/schema delta (as a new file, never a mutation).
- Hand-off artefact list.
- Known follow-ups not blocking go-live.

---

## 15. Setup-agent prompt template (paste this to the target agent)

When handing this spec to the agent that will install the system into a new repo, use a prompt like:

> You are installing the agentic knowledge-management system into `<target-repo>`. The full spec is at `docs/KNOWLEDGE_MGMT_SETUP_SPEC.md` (this file). Read it top to bottom before writing any code.
>
> Operating rules for this install:
> 1. Execute §10 "Setup checklist" in order. Commit per step.
> 2. Copy verbatim only the files explicitly listed in §13 "Safe to copy verbatim". Everything else (briefings, ADRs, plans, stack, versions, runbook) is project-specific and written fresh against the target repo's actual goals.
> 3. Adapt §13 files where §6.4 calls out language-specific globs and §14 names stack-specific variants.
> 4. Do not invent briefings, ADRs, or plans — create them with headers only, leaving bodies empty for the product team to populate.
> 5. After step 9 (hook verification), confirm both Pre and Post hooks fire before continuing.
> 6. After step 13 (handoff), output a short "what's next" note pointing at the first plan file.
>
> Permission mode: use the most restrictive mode that still lets the install complete without prompt fatigue (often `acceptEdits` is right). Use TDD for any code you write (hook scripts → their smoke tests first).

---

## 16. Ongoing-use rules (after install, for every future session)

1. **Session 1 of every day:** read `CLAUDE.md` → `PROGRESS.md` → `BLOCKERS.md` → `RUNBOOK.md § Next N hours`. Only then start work.
2. **Before any edit:** `gitnexus_impact` on the target symbol; warn on HIGH/CRITICAL.
3. **Before any commit:** `gitnexus_detect_changes`; run the relevant test tier.
4. **After any commit:** confirm the hook appended to `PROGRESS.md`; write a `DONE` line manually when closing a milestone.
5. **On any library question:** Context7 MCP first; web search only if Context7 has no entry.
6. **On any library doc that changed:** update `VERSIONS.md` (and `FRONTEND_NOTES.md` on major framework bumps).
7. **On any >30-min stall:** stub + `TODO(blocker)` + `BLOCKERS.md` entry + continue.
8. **On any architectural decision:** new ADR in `DECISIONS.md` with Context / Decision / Consequences.
9. **Before shipping a milestone:** `superpowers:verification-before-completion` evidence run; `code-review:code-review` on the diff; append a `DONE` line to `PROGRESS.md`; update `RUNBOOK.md`.
10. **Periodic maintenance:** `claude-md-management:claude-md-improver` monthly; `npx gitnexus analyze` after large refactors; `fewer-permission-prompts` after every few sessions to reduce prompt noise.
11. **Per-feature gate (§17.11):** PR is incomplete without spec + plan + test + docs update. Trivial changes flagged with `chore:` prefix; author self-assesses.
12. **Per-dep gate (§17.12):** before pinning a new dep, Context7 MCP + registry check → `VERSIONS.md` update → ADR if major bump.
13. **Per-mutation gate (§17.1-17.2):** any new write path threads `request_id` and emits an `audit_log` row. Reviewers reject silently-writing code.
14. **Per-sprint gate (§18.8):** at sprint close, audit this spec against CLAUDE.md's non-negotiables; reconcile divergences.

---

## 17. Production-readiness invariants (past MVP)

Once the repo moves out of "skeleton that compiles" into "real users hit this", a second tier of non-negotiables activates. These are **additive** to the core rules (TDD, commit-per-subsystem, GitNexus-before-edit) and apply uniformly regardless of domain, stack, or deployment target.

The spec enforces them by making each one a **pattern with a concrete seat** (file, table, middleware, ADR), not a policy statement. Policy statements without a seat rot.

### 17.1 Auditability — the `audit_log` sink

Every mutation to any persistent store (database, cache-that-survives-restart, object store, message queue, task board) writes **one event** to a single append-only audit log. Minimum fields:

| Field | Purpose |
|---|---|
| `id` | stable, time-ordered (UUIDv7, ULID, or Snowflake) |
| `ts` | authoritative wall-clock |
| `actor` | human user id, agent name (`agent:<name>`), or `system` |
| `request_id` | ties to every other layer (§17.2) |
| `op` | verb — `insert` / `update` / `delete` / domain-specific (`promote`, `reject`, `tombstone`, …) |
| `entity_ref` | `<store>:<pk>` |
| `diff` | before/after, field-level; redaction-aware for tombstones (§17.5) |
| `schema_version` | which schema version the mutation was written under |

Rules (store-agnostic):

- **No write path bypasses the audit log.** Code-review gate: any new `session.commit()`, raw SQL `INSERT/UPDATE/DELETE`, graph `MERGE`/`CREATE`, object-store `put`, or queue `publish` that isn't wrapped in an audit-emitter is a review blocker.
- **Retention follows the strictest framework the project answers to** — often 5 years (financial / DORA), 6-7 years (SOX / SOC2), or contractual. Use partitioning + cold-tier archival.
- **Append-only at the storage level**: revoke `UPDATE`/`DELETE` from the application role; only the migration/admin role can evict partitions.
- **Redaction-aware**: when a right-to-erasure request fires (§17.5), the audit row stays, but the `diff` column is crypto-shredded. The fact-of-action is eternal; the content is erasable.

### 17.2 Traceability — one `request_id` end-to-end

A single `request_id` threads through **every layer touched by one request**: inbound protocol header → middleware → logger context → every log line → every external-call trace → every persistent write → every downstream job or agent invocation.

Pattern (works across language stacks):

- Inbound middleware generates a `request_id` (UUIDv7 / ULID) if none supplied, and echoes it in the response header so clients can quote it in bug reports.
- Logger context is bound at request start and cleared at end (contextvars, `AsyncLocal`, `continuation-local-storage`, `context.Context` in Go, etc.).
- External-call facades (LLM, DB, queue, third-party APIs) set the trace/correlation header from the context.
- Persistent-store writes read the `request_id` from context; the audit-log column is never null.
- Agents inherit the `request_id` of the event that triggered them; newly minted cron/scheduled jobs generate a fresh one, recorded as the job's root trace.

Why it matters: without this, post-incident reconstruction requires human eyes across N uncorrelated log streams. With it, `grep <request_id>` in any layer produces the full story.

### 17.3 Fail closed, never silent

A cross-cutting error taxonomy, one place. Typical shape (rename per language convention):

```
<ProjectError>                      # base
├── ValidationError                 # 400
├── AuthError                       # 401
├── PermissionError                 # 403
├── NotFoundError                   # 404
├── ConflictError                   # 409 — idempotency, unique violations
├── RateLimitError                  # 429
├── DomainError                     # 422 — invariant broken
├── IntegrationError                # 502 — upstream failure
└── UnavailableError                # 503 — circuit open, maintenance
```

- Every handler raises one of the above; the error-middleware renders `{ok: false, error: {code, message, request_id, details?}}` (or the equivalent in the target protocol).
- **No catch-all `catch (e) { return { ok: true } }`** — that pattern is a review blocker.
- `200 OK` means the **business operation** succeeded. `{"ok": true}` is redundant (the HTTP code is authoritative) but acceptable for typed clients.
- Domain-invariant violations (business-rule breaks, validator failures, stale-schema writes) → `DomainError`, never log-and-continue.

### 17.4 Idempotency by default on every write

Every side-effectful HTTP endpoint (POST/PUT/PATCH/DELETE) accepts an `X-Idempotency-Key` header. Server behaviour:

1. If key is absent: handle normally.
2. If key is present and new: handle, persist `(route, key, request_id, response_hash, response_body, ts)` into `idempotency_store`.
3. If key is present and seen: **replay** the stored response verbatim. Do not re-run the side effect.
4. Keys expire after 24 h (tunable); collision across different bodies → `ConflictError` 409.

Backing store choice matters at scale:
- **In-memory dict** — fine for single-process MVP, loses state on restart.
- **Postgres table** — correct default up to single-region deployments.
- **Redis with TTL** — cheaper at high QPS, but needs durability story (AOF + replica).
- **Durable workflow engine (Temporal, Inngest)** — right answer when writes trigger multi-step workflows with compensations.

Pick one and name it in `DECISIONS.md`. Don't let two co-exist.

### 17.5 Right-to-erasure as a first-class path

Append-only storage + audit logs are load-bearing for compliance — but privacy regimes (GDPR Art. 17, CCPA, PIPEDA, LGPD) demand a way to erase personal data about a specific subject **without** rewriting history.

Pattern: **tombstone + crypto-shred.**

- Each PII-bearing column is stored encrypted with a **per-subject data-encryption key (DEK)**, wrapped by a **key-encryption key (KEK)** in the managed secret store.
- Erasure = delete the subject's DEK. Ciphertext columns become permanently unreadable; row structure, FKs, and audit-log integrity are preserved.
- A tombstone event in the audit log records `op=tombstone`, `entity_ref=<subject>`, `actor=<requester>`, `request_id=…`.
- Downstream projections / read models replay from source-of-truth storage and naturally exclude tombstoned subjects (filter or rewrite).

Do not implement "just delete the row" — it breaks auditability and any append-only / event-sourced substrate. If the project doesn't carry PII, this section collapses to "we process no personal data" in `COMPLIANCE.md` — still explicit.

### 17.6 Secret hygiene — automatic, not procedural

- **Never** in source: enforced by a secret-scanner pre-commit hook (`detect-secrets`, `gitleaks`, `trufflehog`) + server-side scan in CI.
- **Never** in logs: structured-logger processor strips keys matching `.*_KEY$`, `.*_SECRET$`, `.*_TOKEN$`, `password`, `authorization`.
- **Never** in LLM prompts: prompt assembly goes through a wrapper that redacts matched patterns **before** send; trace backends receive the redacted version.
- **Dev** reads from `.env` populated by the repo's secrets generator (placeholder-only substitution).
- **Prod** reads from a managed store, named in the deployment ADR. Pick one per environment: managed cloud vaults (`GCP Secret Manager`, `AWS Secrets Manager`, `Azure Key Vault`), cloud-neutral (`HashiCorp Vault`, `Doppler`, `Infisical`), or a dedicated provider (`1Password Connect`). **Mixed stores in one environment is a smell** — consolidate.
- **Rotation** is a documented procedure in `docs/agent-memory/SECRETS.md` with cadence + blast radius per secret.

### 17.7 Observability is opt-out, not opt-in

Three signals, mandatory in every service:

1. **Traces** — an OpenTelemetry-backed tracer, or a specialized backend for the workload (e.g. an LLM-observability platform if the project does LLM work). Wraps every outbound external call and every business-meaningful operation. Disabled cleanly when creds are missing (no-op facade) — never crash.
2. **Metrics** — a `/metrics` endpoint (Prometheus format is the common floor; OTLP is an alternative). Minimum set: request rate + duration histograms by route/status, error counts by taxonomy class (§17.3), queue/backlog depth for any async worker, saturation gauges for any resource pool (DB connections, thread pool, rate-limit buckets). Domain-specific metrics on top.
3. **Structured logs** — one JSON line per event, always carrying `request_id`, `actor`, `route`, level. `stdout` only; the runtime/log driver collects.

Dashboards and alerts live in infrastructure-as-code (Grafana JSON, Datadog monitors-as-code, alertmanager.yml, etc.) committed alongside the app — not clickops. `RUNBOOK.md § Known issues` references alert runbooks by link.

### 17.8 Migrations are the only schema-change path

- Every persistent-store schema change lands through the store's idiomatic migration tool (Alembic / Flyway / Liquibase / Prisma Migrate / goose / sqlx-migrate / Cypher constraint files, etc.). **No hand-written DDL in application code.**
- **Healthcheck surfaces the current migration head.** If the running instance's head doesn't match the expected head, the healthcheck returns unhealthy and traffic drains. This single check prevents silent schema drift across replicas.
- Migration PRs are reviewed against a **rollback plan** in the PR description. "Undo by re-running with `-1`" is not a plan.

### 17.9 Schema packaging — core vs extensions

When the project sells a horizontal substrate and vertical clients (or hosts multiple tenants with different needs), keep the **core** and the **extensions** in separate files.

- **Core** (tenant-agnostic) is versioned with semver; breaking changes bump major.
- **Extensions** live in separate files named with the tenant or vertical (`core_v2.yaml` + `core_v2_<vertical>.yaml`, or `schema.prisma` + `schema.prisma.<tenant>` overlays, or core DB migrations + per-tenant extension migrations).
- **Never mutate core for a single tenant.** A missing slot/column belongs either in the tenant extension (tenant-specific) or in a core bump (horizontal feature).
- An impact analyzer diffs core × extension and produces per-tenant migration plans.

The same rule applies to API schemas: core endpoints and types live in one place; tenant-specific extensions live alongside, clearly labelled.

If the project is single-tenant and plans to stay single-tenant, flag this section as N/A in the testing/design spec — don't build packaging for an audience of one.

### 17.10 Deployment target — one, named

The repo commits to **one** deployment target and writes all scripts, Dockerfiles, secrets docs, and IAM policies against it. Mixing (some scripts assume GCP, others AWS, some compose for local-only) is the biggest cause of deploy-day chaos.

Rules:

1. Name the target in `RUNBOOK.md § Deployment target` (e.g. "GCP Cloud Run, EU-only replication", "AWS ECS on Fargate, us-east-1 + eu-west-1", "Fly.io, global"). Include region / residency constraints.
2. All prod Dockerfiles and compose/Helm files encode those constraints (base image, arch, region labels).
3. Minimum-privilege IAM policy snippets live in `infrastructure/<provider>/iam/`.
4. CI deployment job gates on environment labels + required approvals.

If the project legitimately needs multi-cloud portability, pay the complexity cost explicitly in a dedicated ADR — typically that's a Kubernetes + Helm choice plus CRD-driven config. Don't let "multi-cloud" happen by accident; it always happens by accident first, then breaks.

### 17.11 Every feature ships with four artefacts

**Invariant:** a feature PR that's missing any of these is incomplete. Reviewers reject, not "document later".

1. **Spec** — `docs/superpowers/specs/YYYY-MM-DD-NN-<area>-design.md` — Problem, Non-goals, Architecture, Interfaces, Tests.
2. **Plan** — appended as a task section in the current sprint plan.
3. **Test** — at least one test file proving the acceptance condition; tier-tagged (unit / integration / e2e).
4. **Docs update** — one or more of: `RUNBOOK.md` (what runs, what's stubbed), `DECISIONS.md` (if an ADR-worthy choice was made), `BLOCKERS.md` (if any sub-piece was deferred), `VERSIONS.md` (if deps changed).

Trivial changes (typo, dep bump, comment) are exempt — but the "trivial" assessment is on the author, visible in the PR title (`chore:` prefix).

### 17.12 SotA or nothing — verification workflow

Before pinning any new dep:

1. Context7 MCP — `mcp__plugin_context7_context7__resolve-library-id` → `query-docs` to confirm current stable.
2. Registry check — `npm view <pkg> version`, `pip index versions <pkg>`, or DockerHub tag list.
3. Update `VERSIONS.md` with resolved version + date + source.
4. If the bump crosses a major, open a new ADR in `DECISIONS.md`.
5. During a session, if a new major ships upstream: log in `BLOCKERS.md` as "deferred upgrade" with a follow-up ticket — do not yank mid-sprint unless security-critical.

Challenge: "latest stable" is not always right — betas with a specific bug-fix, pre-releases pinned to a CVE patch, or vendor LTS lines (Node 20, Python 3.12) can be the correct choice. Document the why in the ADR.

### 17.13 Impact analysis threshold

- **Symbols with 0-1 callers**: GitNexus impact optional (blast radius is trivial).
- **Symbols with 2+ callers**: `gitnexus_impact` mandatory; log risk level in PR description.
- **HIGH / CRITICAL risk**: open a `BLOCKERS.md` entry pre-emptively (even if no blocker yet) documenting the rollback plan + user confirmation trail (or, in autonomous mode, the self-imposed gating).
- Public API, migrations, cross-service contracts: **always** HIGH by default regardless of caller count.

### 17.14 Commit cadence survives long runs

A 6-hour autonomous subsystem is **6+ commits**, not 1. The hook + `PROGRESS.md` is the trace; `git log --oneline` is the table of contents. If a 6-hour push collapses into one commit, the recovery story breaks: compaction mid-push means losing the in-progress understanding of where things stopped working.

Operational rule: if you find yourself without a commit for >60 minutes, either (a) you're stuck — declare it and stub, or (b) you just finished a subsystem — commit. There is no third case.

---

## 18. MVP → Enterprise scaling path (challenge the assumptions)

The spec above works for both an overnight MVP and a multi-tenant enterprise, but the invariants that are **trivial at MVP scale** become **load-bearing at enterprise scale**. This section names the hand-offs so the agent knows when a pattern needs to be re-evaluated.

### 18.1 Stage map

| Stage | What's true | What's next |
|---|---|---|
| **MVP (this repo now)** | Single-tenant, synthetic data, in-memory caches, sync promotion, Docker Compose, 1 region | First real client: multi-env (dev/staging/prod), durable background jobs, real secret store |
| **Reference client (1-3 tenants)** | Per-tenant project/DB, shared substrate code, observability on by default, compliance evidence | Second vertical: ontology-package split, tenant-aware auth, quota/rate limits |
| **Mid-market (10-30 tenants)** | Tenant isolation enforced at schema + row level, durable workflow engine, CDN on frontend, infra-as-code | Enterprise: SOC2, dedicated single-tenant deployments for the largest, K8s if needed |
| **Enterprise** | Signed BAAs, customer-managed encryption keys, private-link ingress, SSO (OIDC/SAML), region-of-choice | Scale-out substrate: sharded staging, async graph projections, horizontal agents |

### 18.2 Assumptions to challenge at each boundary

**Leaving MVP:**

- "In-memory idempotency store is fine" → wrong as soon as you run ≥2 API replicas. Move to a shared durable store (DB table, Redis with persistence) before horizontal scaling.
- "Synchronous handler is fine" → wrong as soon as any step in the handler crosses ~2 s P95. Introduce a durable queue (Temporal, Inngest, `pg-boss`, SQS, `asynq`, etc.) **before** user-visible degradation, not after.
- "Single-container primary DB is fine" → wrong the first time you need a point-in-time restore. Introduce managed DB (Cloud SQL / RDS / Neon / PlanetScale / equivalent) with backups + PITR before first real data lands.
- "Tracing no-op facade is fine without keys" → wrong as soon as you have real users; you need the traces to debug. Wire up a real backend on day 1 of onboarding.

**Leaving reference-client:**

- "Single compose env is fine" → wrong when tenant 2 onboards with different compliance posture. Split dev/staging/prod environments into distinct cloud projects/accounts.
- "Shared database is fine" → wrong as soon as two tenants want schema forks or isolation guarantees. Move to database-per-tenant or row-level isolation + per-tenant encryption.
- "Rate limiter is nice-to-have" → wrong the first time an agent bug accidentally DoSes the backend. Enforce per-tenant quota.
- "Background work runs in-process" → wrong when work spans minutes or must survive restarts. Move to workers + durable scheduler.

**Leaving mid-market:**

- "BYOK is a differentiator" becomes "BYOK is table stakes" — customer-managed encryption keys (CMEK) on DB, object store, and secret store.
- "Our SSO is OIDC-only" — enterprise SSO means SAML 2.0, SCIM provisioning, just-in-time account creation, and SSO-enforced MFA.
- "Best-effort backups" → RTO ≤ 1h, RPO ≤ 15min, proven by quarterly DR drills.
- "Monolithic API" → split read-mostly path (entity, document, ontology-diff) from write-path (ingest, promotion, tickets) for independent scaling.

### 18.3 Architecture escape hatches to keep open

Build these seams on day 1, even at MVP, because retrofitting them costs 5-20× more. Each is a **pluggable interface with a default implementation** — the seam exists whether or not the plugin does.

1. **Tenant-context seam** — thread a `tenant_id` (can literally be `"default"` at single-tenant) through every query, audit log, and trace. Retrofitting multi-tenancy without this is a multi-month project.
2. **Durable-workflow seam** — side-effectful jobs flow through a `JobRunner` interface. Default impl is synchronous in-process; swapping to Temporal / Inngest / Redis queue / SQS is additive.
3. **Persistent-store seam** — every store (DB, graph, object store, cache, queue) is reached through a thin wrapper module. Domain code never imports the driver directly. Swap `localhost` → managed is a config change.
4. **Secret-store seam** — a `SecretProvider` abstraction with implementations for `env`, whichever cloud vault you're on, and generic `vault`. Config flag picks one. Domain code never reads `os.environ["SECRET"]` / `process.env.SECRET` directly.
5. **External-provider seam** — LLM clients, payment providers, email, maps, etc. all pass through a wrapper that owns retries, redaction, tracing, and the provider swap. Domain code never imports the vendor SDK directly.
6. **Auth seam** — even with a single bearer key at MVP, go through an `AuthContext` / `Principal` object. Swap to OIDC/SAML later is a handler-level change, not a grep-and-replace.
7. **Feature-flag seam** — every risky or in-progress feature reads from a flag evaluator (env-backed default impl; Unleash / LaunchDarkly / GrowthBook later). Retrofitting flags is a full-cycle project.

Document each seam in `DECISIONS.md` as an ADR the first time it's added.

### 18.4 Compliance posture — a living table, not a checklist

A `docs/agent-memory/COMPLIANCE.md` file (required as soon as the project touches regulated data; optional otherwise) tracks:

| Framework | Applies when | Status | Evidence |
|---|---|---|---|
| GDPR (Art. 6, 17, 25, 32) | Any EU personal data | e.g. "Partial: Art.25 data-minimization = yes; Art.17 tombstone + crypto-shred implemented; Art.32 encryption at rest per-column via DEK/KEK" | Paths to code modules, migration files, relevant docs |
| DORA | EU financial services (ICT risk) | e.g. "Partial: 5-year audit retention configured; incident-notification SLA 4h in contract" | — |
| EU AI Act (high-risk systems) | From 2026-08-02 if the system qualifies | e.g. "Covered: human-in-the-loop; provenance on every decision; logging adequate" | — |
| SOC2 (CC6/CC7) | US enterprise customers | e.g. "Not started" | — |
| ISO 27001 | Regulated tenants | e.g. "Not started" | — |
| HIPAA | US healthcare | "N/A" when not applicable | — |
| CCPA / PIPEDA / LGPD | US-CA / CA / BR personal data | — | — |

Update this file whenever a control is added or a tenant asks about a framework. The `Evidence` column **must** point to code, migrations, or commits — never to PR descriptions or Slack threads.

When a framework is N/A, write "N/A — [one-line why]". Explicit > omitted.

### 18.5 Backups, DR, and the "one command away from disaster" check

A useful invariant: `docker compose down -v`, `DROP DATABASE`, and `gcloud projects delete` all have explicit confirmations or are blocked by hooks (already in `block_dangerous_bash.sh`). Past that, the recovery story must be real:

- **Backups** configured in infra-as-code, not by clicking in a console.
- **Restore drill** runs at least quarterly (schedulable via the `schedule` skill). The drill restores into a scratch project and asserts `/health` + key `/entity` queries.
- **RTO / RPO** named in `RUNBOOK.md § Deployment target` with actual numbers, not wishes.
- **Tabletop exercises** for the top-3 failure modes (specific to the project — e.g. external provider down, primary store corruption, credential leak, region outage) documented in `docs/superpowers/specs/NN-incident-response-design.md`.

### 18.6 Scaling knobs checklist

A quick "where would I turn a knob if X happens" table. Maintain one in `RUNBOOK.md` — the generic starter below names the *category* of knob; specialize to the project's actual stores and providers.

| Symptom | First knob | Second knob | Third knob |
|---|---|---|---|
| Hot-path p95 latency rising | Query plan + index (profile first) | Read replica / cache layer | CQRS read model / materialized view |
| External-provider cost overrun | Hit rate of existing cache | Route cheaper model/tier for low-stakes paths | Add a semantic/response cache |
| Async backlog growing | Worker concurrency | Batch size + chunking | Durable queue + backpressure |
| Tenant noisy-neighbour | Per-tenant quota / rate limit | Dedicated connection pool | Dedicated infra per tier |
| Primary-store memory/CPU pressure | Config tuning (heap, workers, pool) | Scale up (vertical) | Shard / archive cold data |
| Agent-loop cost | Tune cadence (avoid cache-miss windows) | Batch via scheduled runs | Convert to cron-only, drop dynamic loop |

### 18.7 Agent-discipline invariants at enterprise scale

Long-running agentic work doesn't disappear when the system goes enterprise — it gets **more important** because manual review can't keep up with fleet-scale changes. The existing invariants tighten:

- **Every agent action that touches production writes to `audit_log` with `actor=agent:<name>`.**
- **Agents never hold a credential that a human couldn't hold** — workload-identity federation (OIDC) replaces long-lived tokens. GitHub Actions → GCP via WIF, not a service-account JSON.
- **Agent decisions are traceable to the plan step that produced them** — the `PROGRESS.md` line + spec filename + plan task id live in the commit trailer.
- **Autonomous mode is scoped, not global** — `--permission-mode bypassPermissions` is confined to worktrees; PR merges require human approval or a signed-off automated reviewer policy.

### 18.8 Update cadence for the spec itself

This file is living documentation. It must be audited at least **once per sprint close** and when any of the following happens:

- A non-negotiable changes in `CLAUDE.md` (this is how §17 got added).
- A new MCP or skill becomes load-bearing.
- A deployment target changes.
- A compliance framework gets added.
- The team discovers a pattern that would have saved hours if documented — pull it in.

Rule of thumb: if you found yourself wishing "the spec should have told me X", edit it. That is the spec earning its keep.

---

*End of spec. Commit this file in the target repo as `docs/KNOWLEDGE_MGMT_SETUP_SPEC.md` so future agents can rediscover the system's own manual. When it drifts, update it — this file itself is subject to the non-negotiables (TDD-where-cheap, commit-per-subsystem, PROGRESS-after-commit, feature-ships-with-four-artefacts per §17.11).*
