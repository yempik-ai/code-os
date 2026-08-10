# Orchestrating CLI subagents

> One Claude Code session as the brain. Other CLI agents as hands. This
> document is the field-tested version of that pattern as of August 2026:
> what to delegate, to which model, with which exact commands, and the four
> failure modes that cost us real hours so they don't have to cost you any.

---

## 1. The pattern

Claude Code can run any other agentic CLI through its shell, the same way it
runs `git` or `pnpm`. That turns a single session into an orchestrator:

- **The orchestrator** (your Claude Code session) holds the durable context:
  repo state, memory files, the conversation with you, and the taste to judge
  output. It never outsources the accept/reject decision.
- **Subagents** are other vendors' CLI agents (OpenAI Codex CLI today; the
  contract works for grok, ollama, or any headless CLI) launched as
  foreground shell commands with a prompt in and a final message out.

Three properties make this worth the plumbing:

1. **Cost offload.** A Codex subagent bills your OpenAI plan, not your Claude
   plan. Bulk work moves to whichever meter is cheapest for you, and the
   orchestrator's context window stays clean because the subagent's trail
   never enters it.
2. **Uncorrelated second opinions.** A different vendor's model has different
   blind spots. A Claude plan reviewed by a GPT-family model (or vice versa)
   catches failure modes that same-model review reliably misses.
3. **Parallel hands.** Explorers can sweep a codebase while the orchestrator
   keeps working. The orchestrator reads conclusions, not file dumps.

## 2. The four subagent roles

| Role | What it does | Mode | Typical model bar |
|---|---|---|---|
| **Explorer** | Sweeps code or docs, returns a map or an answer. | read-only | cheap and fast |
| **Implementer** | Executes a clearly specified change: code, tests, migrations. | write | strong on instruction-following; taste can be average if the spec carries the taste |
| **Reviewer, adversarial** | Tries to refute the work: find the bug, break the migration, reject the claim. | read-only | as smart as you can afford |
| **Second opinion** | Independent take on a plan or design before you commit to it. | read-only | different vendor from the author, on purpose |

The reviewer and second-opinion roles are the underrated ones. Going back and
forth between models on a design, with each prompted to attack the other's
proposal, is the cheapest quality mechanism we know. It works on any budget
because the loser of the argument is usually obvious from the transcript.

## 3. Tier the ceremony, then pick the model

### Ceremony tiers

The original code-os harness prescribed spec, plan, TDD, review for all
non-trivial work. That was correct for the models of early 2026 and is
usage-expensive overkill for the frontier models of today. The current rule:

- **Trivial** (one symbol, a known file): the orchestrator does it directly.
  No subagent, no ceremony.
- **Formerly complex** (a feature, a refactor, a bug with an unclear cause):
  one frontier model, inline, at high reasoning effort. No spec document, no
  plan file. The conversation is the plan.
- **Formerly impossible** (multi-day, multi-system, or work you would not
  have trusted any early-2026 model to survive): the full spec, plan,
  TDD-sized steps, verification loop from
  [`../agentic-harness/`](../agentic-harness/index.md). This tier still earns
  every line of its ceremony.

### Model selection

Rank the models you can actually reach on three axes and write the table
down in your own `CLAUDE.md`. **Cost is what you pay, not list price**: a
model included in a flat plan you already own is nearly free regardless of
its API price. Worked example, August 2026:

| model | cost (as paid) | intelligence | taste |
|---|---|---|---|
| gpt-5.6 (Codex CLI, flat plan) | 9 | 8 | 5 |
| sonnet-5 | 5 | 5 | 7 |
| opus-5 | 4 | 7 | 8 |
| fable-5 | 2 | 9 | 9 |

Rules of application, in order:

1. For anything that ships: intelligence beats taste beats cost. Cost breaks
   ties only.
2. Anything user-facing (UI, copy, API surface) needs taste at 7 or above.
3. Bulk and mechanical work (clear-spec implementation, migrations, data
   analysis) goes to the cheapest high-intelligence model you have. In the
   example table that is gpt-5.6, because it is effectively free.
4. Escalate without asking. If a cheaper model's output misses the bar, rerun
   with a smarter one. Judge the output, not the price tag: escalating costs
   less than shipping mediocre work.
5. Cap reasoning effort at **high** on frontier models. Above-high effort and
   multi-agent fan-out modes only pay for themselves on top-tier flat plans;
   on anything smaller they drain the week's budget on one afternoon.

## 4. The run-path convention

Every subagent run gets exactly three files, allocated by the orchestrator
before launch, under a scratch directory outside the repo:

```text
<scratch>/codex/<slug>.prompt.md   # written before launch: the full task
<scratch>/codex/<slug>.final.md    # the subagent's final message: the only thing anyone reads
<scratch>/codex/<slug>.log         # stdout+stderr trail: opened only on failure
```

`<slug>` is a short kebab-case task name (`rls-migration`, `auth-refactor`),
unique per run; re-runs get `-2`, `-3`, never a reused slug. The orchestrator
is the single allocator of these paths. Two rules follow from pain:

- **Concurrent runs sharing an output path clobber each other** and the
  loser's final message is gone. One allocator, unique slugs, always.
- **The result is recoverable from disk.** If the launch wrapper dies or
  summarizes badly, `<slug>.final.md` is still there. State on disk beats
  state in context; the same principle the agentic-harness is built on.

## 5. Codex runbook

Verified against **codex-cli 0.145.0**. Flags drift between releases: check
`codex exec --help` before copying these into automation, and note that
`--full-auto` is accepted but not listed in the help output.

Substitute your own model id for `<model>`. It is whatever your Codex CLI
exposes on your plan, not a universal constant; the value in your
`~/.codex/config.toml` under `model =` is the one you know works.

Prompts always go through the prompt file, never inline in the shell command.
Implementation work (code, tests, migrations) runs in write mode:

```bash
codex exec --full-auto -m <model> -c 'model_reasoning_effort="high"' \
  -o <scratch>/codex/<slug>.final.md "$(cat <scratch>/codex/<slug>.prompt.md)" \
  < /dev/null > <scratch>/codex/<slug>.log 2>&1
```

Exploration, review, and second opinions run read-only (same model, writes
impossible, so an adversarial reviewer cannot "helpfully fix" the code it
was told to attack):

```bash
codex exec -m <model> -c 'model_reasoning_effort="high"' \
  -c 'sandbox_permissions=["disk-full-read-access"]' \
  -o <scratch>/codex/<slug>.final.md "$(cat <scratch>/codex/<slug>.prompt.md)" \
  < /dev/null > <scratch>/codex/<slug>.log 2>&1
```

The prompt file must be self-contained: repo path, task, constraints,
acceptance criteria, and the instruction to keep the final message to
conclusions, not narration. The subagent has none of your session context
and never will.

## 6. The four failure modes

Each of these cost us real debugging time. In order of sneakiness:

1. **Unclosed stdin hangs forever.** `codex exec` without `< /dev/null`
   waits for more input and produces nothing. If the output is piped through
   `tail` or similar, the hang is invisible until you inspect the process.
   Close stdin explicitly, every run.
2. **The launch cwd is the sandbox's writable workspace, and it must be a git
   repo.** Launch from the repo root, never a subdirectory, or writes outside
   that cwd are rejected. Codex refuses to start outside a git repo at all.
   Both have first-class escapes once you know they exist: `-C <dir>` sets the
   working root explicitly, `--add-dir <dir>` grants an extra writable
   directory, and `--skip-git-repo-check` allows a non-repo directory. Reach
   for the flags rather than the folklore, and keep the blast radius
   deliberate: every extra writable directory is one more place a subagent
   can go wrong.
3. **Never background the run.** A wrapper that backgrounds the shell and
   promises to "check on it later" has already returned; the shell dies with
   it and the output is discarded. Foreground only, with a generous timeout.
   Work longer than your shell ceiling gets split into shorter runs, not
   backgrounded.
4. **Never read the trail.** Codex stdout is the entire session: reasoning
   summaries, every shell command, every patch, token counters. It is
   worthless as evidence and expensive as context. Take the final message
   from `-o`; open the log only on a non-zero exit, and only the last ~50
   lines. The authoritative record of what a subagent changed is `git diff`,
   not the subagent's narration of it.

## 7. Acceptance discipline

The orchestrator accepts nothing on faith. Before a subagent's work counts:

- **Read the diff.** `git diff` in first person, hunk by hunk.
- **Re-run the checks in first person.** Typecheck, tests, build. A
  subagent's green output is a claim; your green output is a fact. This is
  the same no-done-without-verification gate the
  [`code-os-core`](../../plugins/code-os/skills/code-os-core/SKILL.md) skill
  enforces, applied across vendors.
- **For migrations that touch live definitions**: compare the generated body
  against the live one before applying, every time.

Rejected work goes back with the reviewer's findings pasted into a fresh
prompt file under a new slug. Two rejections means the task was underspecified:
fix the prompt, or move the task up a ceremony tier.

## 8. Wrapping any other CLI

Codex is the worked example, not the point. The contract any headless CLI
must meet to slot in as a subagent:

1. Accepts a prompt (from a file via substitution, or stdin).
2. Can write its final answer to a known file, or at worst prints it last so
   you can capture it.
3. Runs non-interactively to completion in the foreground.
4. Has some sandbox or permission story you have read before granting write
   access to a repo.

grok CLI and ollama-served local models both meet the contract; so will the
next vendor's CLI. Keep the three-path convention and the acceptance
discipline identical across all of them, and swapping vendors becomes a
one-line change in the launch command. That is the point: **own the harness,
swap the model.**

---

## Relation to the rest of code-os

- [`../agentic-harness/`](../agentic-harness/index.md) gives the orchestrator
  its memory and safety rails. Subagents do not get the harness; they get a
  prompt file and a sandbox. Only the orchestrator carries state.
- [`../tech-references/`](../tech-references/index.md) covers orchestration
  *inside* the products you build. This module is about orchestrating your
  own development workflow. Same principles, different layer.
- The tiering rule in §3 supersedes the always-on spec, plan, execute loop
  that earlier versions of code-os prescribed for all non-trivial work. See
  [`CHANGELOG.md`](../../CHANGELOG.md) for what changed and why.

---

*In produzione, non in slide. · by yempik. · maintained by Simone Bova*
