# Getting started

`code-os` is six self-contained modules. You don't install all of them. You pick the one that solves the problem in front of you and follow its steps.

Two modules are **setups** (they write files into a repo or a vault and need supervised time), one is a **light setup** (a few conventions in your `CLAUDE.md`), and three are **read-only references** that you open when you need them.

---

## Not sure where to start? Install the plugin

In Claude Code:

```text
/plugin marketplace add yempik-ai/code-os
/plugin install code-os@code-os
```

Then run `/code-os:install`, the guided router. It asks what you want and points you at the right place. The plugin ships six commands: the router plus five that go straight to a module if you already know your destination.

| Command | What it does |
|:--|:--|
| `/code-os:install` | Guided router. Asks what you want, routes you to the right module. |
| `/code-os:subagents` | Sets up CLI-subagent orchestration: model table, run paths, runbook. |
| `/code-os:harness` | Drives the agentic-harness install into the current repo. |
| `/code-os:second-brain` | Drives the second-brain (Obsidian vault) setup. |
| `/code-os:skills` | Surfaces the curated skills, plugins and MCP guide, plus the security-audit procedure. |
| `/code-os:stack` | Runs the brief-driven tech-references workflow: reference stack and orchestration patterns into a plan. |

Skip the rest of this page unless you'd rather drive by hand.

---

## Multi-model orchestration: start here

*"I want more throughput and second opinions without burning my plan."*

Claude Code can run any other agentic CLI through its shell, the same way it runs `git` or `pnpm`. That turns one session into an orchestrator: it keeps the context, the memory and the final say, while other vendors' CLI agents (Codex today, any headless CLI tomorrow) act as explorers, implementers and adversarial reviewers on their own subscriptions. The module also carries the ceremony-tiering rule that decides when work runs inline on one frontier model and when it earns the full harness loop.

**Time budget:** about 15 minutes. It's conventions in your `CLAUDE.md`, not infrastructure.

1. Read [`modules/multi-model-orchestration/orchestrating-cli-subagents.md`](./modules/multi-model-orchestration/orchestrating-cli-subagents.md). The whole module is one document.
2. Check you have at least one other agentic CLI installed (`which codex`).
3. Copy the model table, the run-path convention and the runbook commands into your `CLAUDE.md`, adapted to the plans you actually pay for. Or let `/code-os:subagents` drive it.

**Prerequisites:** Claude Code, plus at least one other agentic CLI on a plan you already own.

## Agentic harness: memory and safety

*"I want Claude Code to behave reliably on my day-job repo across multi-day work."*

Installs survival-kit memory (`docs/agent-memory/`), a spec-to-plan-to-execute scaffold (`docs/superpowers/`) for the work that earns it, about 5 enforcement hooks, and a minimal `CLAUDE.md` you own by hand. Stack-agnostic: Node, Python, Go, Rust, Java, polyglot all work.

**Time budget:** about 10 minutes of your own prep, plus 1 to 2 hours of mostly-idle supervised setup.

1. Read [`modules/agentic-harness/index.md`](./modules/agentic-harness/index.md) for the read order and what gets installed.
2. Do the human steps in `modules/agentic-harness/DAY-0-CHECKLIST.md` (install Claude Code, make sure the target repo has at least one commit, drop the folder in).
3. Paste `modules/agentic-harness/BOOTSTRAP.md` into Claude Code at your repo root and supervise, or let `/code-os:harness` drive it.
4. From Day 2, operate it with `modules/agentic-harness/agentic-harness-tutorial.md`.

**Prerequisites:** Claude Code installed (`npm install -g @anthropic-ai/claude-code`), a git repo with at least one commit, a bash shell.

## Second brain: Obsidian knowledge vault

*"I want a durable second brain that ingests sources and stays coherent over time."*

An Obsidian vault driven by Claude Code: three-layer architecture (immutable raw, then a Claude-owned wiki, then a fixed schema), vault-local slash commands, enforcement hooks, cross-device sync. You ingest sources; Claude links and synthesises.

**Time budget:** about 45 to 60 minutes of your own prep, plus 2 hours of supervised setup.

1. Read [`modules/second-brain/index.md`](./modules/second-brain/index.md) for the read order.
2. Do the human steps in `modules/second-brain/DAY-0-CHECKLIST.md` (accounts, Homebrew, Node, Python, Claude Code, Obsidian and community plugins, macOS Full Disk Access, a private GitHub remote, sync).
3. Paste `modules/second-brain/BOOTSTRAP.md` into Claude Code and supervise, or let `/code-os:second-brain` drive it.
4. Operate it day to day with `modules/second-brain/second-brain-tutorial.md` and its eight slash commands (`/today`, `/ingest`, `/query`, `/crystallize`, `/weekly-review`, `/tech-digest`, `/new-domain`, `/lint`).

---

## The reference modules (read, no setup)

- **Skills and plugins:** [`modules/skills-guide/useful-skills.md`](./modules/skills-guide/useful-skills.md). A guide with a point of view to the plugins, MCP servers and workflows worth installing on top of Claude Code (what, why, when, for whom, how), plus a procedure to vet third-party plugins for prompt injection. Install à la carte. Plugin shortcut: `/code-os:skills`.
- **Tech references:** [`modules/tech-references/index.md`](./modules/tech-references/index.md). A generalized reference stack, an adversarially-verified tool-orchestration compendium, vendor-neutral production patterns, and a brief-driven workflow that turns them into a plan. Every public claim carries a named source or a pinned version, and the snapshots are dated, so re-verify before you pin anything. Plugin shortcut: `/code-os:stack`.
- **AI tools and people:** [`modules/ai-tools/useful-ai-tools.md`](./modules/ai-tools/useful-ai-tools.md) and [`modules/personas/people-to-follow.md`](./modules/personas/people-to-follow.md). The AI-native tools worth a look and the practitioners shaping agentic engineering.

---

## A note on deferred tools

The setup modules deliberately keep code-intelligence tools, custom memory layers and speculative MCPs off Day 1. They're earned once the repo or vault is stable, not imposed up front. Every tool you add on Day 1 is one you have to debug before you know whether you needed it, so if anything (or anyone) proposes them early, push back.

## Keep it healthy

The harness and the second brain are operated daily through their tutorials, not set and forget. Budget about **10 minutes a day** of discipline:

- **Start of session:** let the harness rehydrate from `docs/agent-memory/` before you start working.
- **End of session:** log progress, decisions and blockers; don't let work ship "done" without verification.
- **Second brain:** ingest as you go; run `/weekly-review` and `/lint` to keep the vault from drifting.
- **Subagents:** re-verify every subagent claim in first person. The acceptance discipline is what makes delegation safe.

That overhead is what makes the system compound instead of resetting every session. It earns its keep on work that spans multiple sessions, not on throwaway scripts.

---

<sub>In produzione, non in slide. · by yempik. · maintained by Simone Bova</sub>
