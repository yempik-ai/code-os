# Day-0 Checklist — Agentic Harness setup

**Who this is for.** You, the operator. These are the steps a human body has to do before pasting `BOOTSTRAP.md` into Claude Code — installing the CLI, making sure the target repo is in a state Claude can work in, and dropping this folder somewhere inside the repo so Claude Code can read the spec. Everything else — `docs/agent-memory/` survival kit, `docs/superpowers/` scaffold, the ~5 hook shell scripts, `CLAUDE.md`, `AGENTS.md`, the first commits — Claude Code does once you hand off.

**Budget.** ~10 min of clicking + waiting, then ~1–2 hours of supervising Claude Code (mostly idle — do other work).

**What you'll need.**

- A machine with a bash-compatible shell (macOS, Linux, or WSL on Windows).
- A browser (one-time, to sign into Anthropic during the first `claude` run).
- An **Anthropic** account on a paid plan (Pro is fine to start; Max if you expect heavy use).
- A target repo you actually want to run Claude Code on.

**What you're handing to Claude Code.**

- `KNOWLEDGE_MGMT_SETUP_SPEC.md` — the ~1,200-line authoritative spec.
- `BOOTSTRAP.md` — the prompt that drives Day-1 setup.
- `agentic-harness-tutorial.md` — the daily-operation field manual (Claude won't read this on Day 1, but it'll be there when you start operating from Day 2).
- This checklist — so Claude Code knows what you've already done.

---

## Part 1 — Accounts & toolchain (~5 min)

- [ ] Create an **Anthropic** account at [claude.ai](https://claude.ai/) if you don't have one. Pick a paid plan.
- [ ] Open a terminal and install **Claude Code**:
  ```
  npm install -g @anthropic-ai/claude-code
  ```
  Requires Node.js. If you don't have it: macOS `brew install node`, Debian/Ubuntu `sudo apt install nodejs npm`, Windows use the official installer or WSL.
- [ ] Verify: `claude --version` (any version string is fine).
- [ ] Run `claude` once in any directory. A browser opens for Anthropic login. Sign in, then exit (`/exit`) — this caches your credentials so the actual setup session doesn't get interrupted by an auth flow.

## Part 2 — Target repo state (~2 min)

The harness is designed for a real codebase. The target repo needs:

- [ ] **A git history.** At minimum one commit. If this is greenfield:
  ```
  cd <target-repo>
  git init
  git add .
  git commit -m "initial"
  ```
- [ ] **Bash available** — the hooks Claude Code installs are `.sh` scripts. macOS/Linux/WSL all fine. Pure-Windows PowerShell-only environments are not supported.
- [ ] **Stack-agnostic, but not for throwaway code.** Node, Python, Go, Rust, Java, polyglot — all fine. The harness costs ~10 min/day of operating overhead; it earns its keep on multi-session work, not on one-shot scripts you'll throw away in a week.

## Part 3 — Drop this folder into the target repo (~1 min)

Claude Code reads `KNOWLEDGE_MGMT_SETUP_SPEC.md` during setup, so the spec needs to live somewhere inside the target repo.

- [ ] Copy the entire `agentic-harness/` folder into your target repo. A sensible target path is `docs/agentic-harness/`:
  ```
  cp -r /path/to/agentic-harness/ <target-repo>/docs/agentic-harness/
  ```
- [ ] Verify the spec is reachable: `ls <target-repo>/docs/agentic-harness/KNOWLEDGE_MGMT_SETUP_SPEC.md`. Note the exact path — you'll paste it into the bootstrap prompt as `<PATH-TO-SPEC>`.

(You can use any other path you like — `handover/`, `setup/`, repo root, anywhere. Claude Code will find it as long as you tell it where.)

## Part 4 — Hand off to Claude Code

- [ ] In Terminal:
  ```
  cd <target-repo>
  claude
  ```
- [ ] Open `BOOTSTRAP.md` (in this folder, or wherever you copied it). Copy the entire block between `===== BEGIN PROMPT =====` and `===== END PROMPT =====`.
- [ ] Paste it as your first message in the `claude` session. **Replace `<PATH-TO-SPEC>`** with the actual path you noted in Part 3 (e.g. `docs/agentic-harness/KNOWLEDGE_MGMT_SETUP_SPEC.md`).
- [ ] Supervise. Approve tool calls as they come.
- [ ] **Read each of the ~5 hook shell scripts before approving the first tool call that installs them** — hooks fire on every subsequent tool call, so you want to know what they do.

---

## What Claude Code will do on its own

**During Day 1 (~1–2 hours of supervised work):**

- Read `KNOWLEDGE_MGMT_SETUP_SPEC.md` top-to-bottom.
- Summarise the install plan back to you in ≤10 lines and wait for `proceed`.
- Read your repo (git log, root listing, package manifests, existing `README` / `CLAUDE.md` / `docs/`) and summarise what it found.
- Seed `docs/agent-memory/` (the survival kit: `PROGRESS.md`, `DECISIONS.md`, `BLOCKERS.md`, `STACK.md`, `VERSIONS.md`, optional domain files, `README.md`) — first commit.
- Seed `docs/superpowers/` (the spec → plan → execute scaffold) — second commit.
- Write the ~5 hook shell scripts + `.claude/settings.json`. **Will read each hook aloud and pause for your approval** before running any tool call after they're installed.
- Briefly interview you for the facts that can't be inferred from code (one-line product purpose, who the operators are, deployment targets, the one invariant that must never break, current biggest pain point).
- Write `CLAUDE.md` and `AGENTS.md` at the repo root, tailored to your stack and the interview answers.
- Run a synthetic commit to verify the PostToolUse hook appends to `PROGRESS.md`.
- Produce a Day-1 summary: what's installed, what's deferred, what's the next piece of real work.

**Days 2+:** You use the harness. See `agentic-harness-tutorial.md` for the daily-operation field manual — session-start rehydration, the spec → plan → execute loop, blocker protocol, verification gates, compaction recovery.

**Deferred (not Day 1):** GitNexus, custom memory layers, speculative MCPs. The spec's §12 explicitly defers these — they're earned by the repo once stable, not imposed on Day 1. Push back if Claude proposes any of them during setup.

## When to interrupt Claude Code

- It starts writing `CLAUDE.md` before reading your repo or interviewing you — interrupt. Repo read + interview come first.
- It copy-pastes the spec's example shell scripts verbatim without adapting to your stack — interrupt. Hooks must match your shell, branch names, and CI.
- A command looks destructive (`rm -rf`, `git push --force`, `git reset --hard`) — read the command, ask why, only proceed if the answer is good.
- It proposes installing GitNexus, a custom memory layer, or any non-core tool on Day 1 — push back. Point it at §12 of the spec.
- A step runs >30 min with no forward progress — remind it of the §8.5 stall rule (stub, log to `BLOCKERS.md`, move on).

## If something goes badly wrong

- **Claude Code crashes / session dies mid-setup.** Re-run `claude` in the same directory. Paste: *"Resume the harness installation. Follow the §8.1 session-start protocol — read `docs/agent-memory/PROGRESS.md`, `BLOCKERS.md`, `DECISIONS.md`, and the active plan in `docs/superpowers/`. Only then resume from the last incomplete step."* The survival kit is exactly what makes this recovery work.
- **A hook denies a command you actually wanted.** Don't disable the hook. Narrow the deny rule to exclude your case, commit the fix, retry.
- **Hooks don't fire after install.** Check `.claude/settings.json` is valid JSON and references the right hook paths. Common gotcha: absolute vs repo-relative paths.
- **`CLAUDE.md` has invented facts about your repo.** Ask Claude to find every claim not grounded in a file it read during the repo-summary phase, replace with `[TBD]` markers, and log the gaps to `BLOCKERS.md`. Re-interview on the gaps.

---

*Once you've finished Parts 1–4, you're done with Day-0. Hand off to Claude Code via `BOOTSTRAP.md` and supervise. Day 2 onward is `agentic-harness-tutorial.md`.*
