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
`codex exec --help` before copying these into automation.

Substitute your own model id for `<model>`. It is whatever your Codex CLI
exposes on your plan, not a universal constant; the value in your
`~/.codex/config.toml` under `model =` is the one you know works.

Prompts always go through the prompt file, never inline in the shell command.
Implementation work (code, tests, migrations) runs in write mode:

```bash
codex exec --sandbox workspace-write -m <model> -c 'model_reasoning_effort="high"' \
  -C <repo-root> \
  -o <scratch>/codex/<slug>.final.md "$(cat <scratch>/codex/<slug>.prompt.md)" \
  < /dev/null > <scratch>/codex/<slug>.log 2>&1
```

(`--full-auto` is an older spelling that still works but no longer appears
in the help output; prefer the explicit `--sandbox` flag.)

Exploration, review, and second opinions pin the sandbox to read-only, so an
adversarial reviewer cannot "helpfully fix" the code it was told to attack.
Pin it explicitly with `-s read-only` — config keys alone do not guarantee
the sandbox policy:

```bash
codex exec -s read-only -m <model> -c 'model_reasoning_effort="high"' \
  -c 'sandbox_permissions=["disk-full-read-access"]' \
  -C <repo-root> \
  -o <scratch>/codex/<slug>.final.md "$(cat <scratch>/codex/<slug>.prompt.md)" \
  < /dev/null > <scratch>/codex/<slug>.log 2>&1
```

Two mechanical notes on these commands:

- `"$(cat …)"` passes the whole prompt as one process argument, which has an
  OS-level size limit. For very large prompt files (a fan-out mapper stage
  inlining a previous stage's output can get there), use the stdin form
  instead: `codex exec … - < <scratch>/codex/<slug>.prompt.md`, dropping the
  `< /dev/null`, since stdin now carries the prompt.
- The cost-offload only holds if your Codex CLI is authenticated against a
  flat subscription. Authenticated with an API key, the same 900k-token
  fan-out is metered spend. Check which auth your `codex` is using before
  pointing a pipeline at it.

The prompt file must be self-contained: repo path, task, constraints,
acceptance criteria, and the instruction to keep the final message to
conclusions, not narration. The subagent has none of your session context
and never will.

## 6. Fan-out: the wrapper contract inside workflows

Everything above is one subagent per shell command, launched by the
orchestrator directly. The pattern also survives multi-agent fan-out — a
workflow that spawns ten or more agents in a pipeline — and that is where
the cost offload gets dramatic, because fan-out is precisely what drains a
flat plan fastest.

The obstacle: workflow and subagent harnesses typically only spawn their own
vendor's models. The escape: a **thin wrapper agent**. Spawn the cheapest
Claude model at the lowest reasoning effort, and make its entire job the
mechanical execution of one Codex run:

1. Write the task (passed verbatim in the wrapper's prompt) to
   `<scratch>/codex/<slug>.prompt.md`.
2. Run the `codex exec` command from §5 — with `-C <repo-root>` set
   explicitly, since the wrapper's own cwd is wherever the harness put it —
   in the foreground, with a generous timeout, never backgrounded, never
   ending its turn before the command returns.
3. Exit 0 **and** `<slug>.final.md` exists and is non-empty: return its
   contents verbatim as the final message. Nothing else. An exit 0 with a
   missing or empty final file is a failure; treat it as step 4.
4. Otherwise: return the last ~50 lines of `<slug>.log`, prefixed
   `CODEX FAILED:`.

The wrapper never invents paths (the workflow script is the single
allocator, slugs unique per run), never reads the log on success, and never
summarizes codex stdout. It is plumbing, not a mind, which is why the
cheapest model at the lowest effort is the correct choice. One transport
caveat: "verbatim" passes through the wrapper's own model response, which
has an output ceiling — instruct subagents to keep final messages to
conclusions, and pass anything bulky by file path, not by relay.

A factory in the workflow script keeps the contract in one place. Sketch,
not runnable code — the real version interpolates the concrete scratch
path, slug, and task text into the wrapper prompt:

```js
function codexAgent({ slug, phase, label, task, effortLevel }) {
  const prompt = buildWrapperPrompt({ slug, task, effortLevel }) // steps 1-4
  return agent(prompt, { label, phase, model: 'sonnet', effort: 'low' })
}
```

Field result (2026-08-10, one run, n=1): a mine-and-map pipeline over two
monorepos — six axes, one codex miner plus one codex mapper per axis,
twelve agents total — completed in ~20 minutes with zero failed runs and
~892k harness-metered subagent tokens. The reading and reasoning ran inside
Codex on a flat plan; the wrappers' own usage was not separately metered,
but twelve low-effort plumbing turns is a small fraction of what twelve
native frontier agents reading two monorepos would have cost the session's
plan.

Two wrapper-specific cautions, learned the same day:

- **Pipeline stages pass data through prompts, not context.** The mapper
  stage receives the miner's output inlined in its prompt file. Keep stage
  outputs in strict, parseable formats or the handoff degrades.
- **An empty final message is a silent failure.** A wrapper that returns
  nothing looks like a completed agent to the workflow. Check the journal
  (or the `final.md` files on disk) before trusting an empty or thin result.

## 7. The four failure modes

Each of these cost us real debugging time. In order of sneakiness:

1. **An open stdin can hang the run.** `codex exec` reads the prompt from
   stdin when no positional prompt is given (or `-` is passed), and appends
   piped stdin to a positional prompt. With a positional prompt and an
   inherited, never-closing stdin — the default inside many harnesses — the
   run can sit waiting while producing nothing, and if the output is piped
   through `tail` or similar the hang is invisible until you inspect the
   process. Close stdin explicitly (`< /dev/null`) on every positional-prompt
   run; when using the stdin form, the prompt file redirect closes it for
   you.
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
   promises to "check on it later" has already returned — and in
   subagent harnesses, a returned agent's background children are dead or
   orphaned, with nobody left to collect the result. Whether the process
   technically survives depends on the harness; the loss of exit-status and
   lifecycle ownership does not. Foreground only, with a generous timeout.
   Work longer than your shell ceiling gets split into shorter runs, not
   backgrounded.
4. **Never read the trail.** Codex stdout is the entire session: reasoning
   summaries, every shell command, every patch, token counters. It is
   worthless as evidence and expensive as context. Take the final message
   from `-o`; open the log only on a non-zero exit, and only the last ~50
   lines. The authoritative record of what a subagent changed is `git diff`,
   not the subagent's narration of it.

## 8. Acceptance discipline

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

## 9. Wrapping any other CLI

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
