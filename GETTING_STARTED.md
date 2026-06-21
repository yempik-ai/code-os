# Getting started

`code-os` is five self-contained modules. You don't install all of them — you pick the one that solves the problem in front of you and follow its steps. This page gives you two ways to do that.

- **Path A — guided.** Let `code-os` ask what you want and route you. Use this if you're not sure which module you need.
- **Path B — manual.** You already know which module you want. Go straight to it.

Two of the five modules are **setups** (they write files into a repo or a vault and need supervised time). The other three are **read-only references** — open them, no install. We cover both below.

If you only read one other file first, make it [`index.md`](./index.md) — the full repo map.

---

## Path A — guided setup (recommended if unsure)

You tell `code-os` what you're trying to do; it asks a few questions and routes you to the right module's setup. Two ways to start it:

**Option 1 — paste the installer.** Open [`INSTALL.md`](./INSTALL.md), paste its contents into Claude Code at the root of the repo you're working in, and answer the questions. It's a router, not a monolith: it figures out whether you want the harness, the second brain, the skills guide, or the reference stack, and hands you off to the right module.

**Option 2 — install the plugin.** In Claude Code:

```text
/plugin marketplace add yempik-ai/code-os
/plugin install code-os@code-os
```

Then run `/code-os:install` — the guided router. It asks what you want and points you at the right place. The plugin ships five commands in total: the router plus four that go straight to a module if you already know your destination.

| Command | What it does |
|:--|:--|
| `/code-os:install` | Guided router — ask what you want, route to the right module. |
| `/code-os:harness` | Drive the agentic-harness install into the current repo. |
| `/code-os:second-brain` | Drive the second-brain (Obsidian vault) setup. |
| `/code-os:skills` | Surface the curated skills/plugins/MCP guide and the security-audit procedure. |
| `/code-os:stack` | Run the brief-driven tech-references workflow — reference stack and orchestration patterns into a plan. |

That's it for Path A. Skip the rest of this page unless you'd rather drive the setup by hand.

---

## Path B — manual setup (per module)

Pick the module. Each setup module follows the same shape: a human pre-step checklist (`DAY-0-CHECKLIST.md`), then a paste-into-Claude-Code prompt (`BOOTSTRAP.md`), then supervised time while Claude builds it. The authoritative spec is read by Claude during setup — you don't read it cover-to-cover.

### Agentic harness — the hero

Make Claude Code reliable on a real codebase across long, multi-session work. It installs survival-kit memory (`docs/agent-memory/`), a spec → plan → execute loop (`docs/superpowers/`), ~5 enforcement hooks, and a tailored `CLAUDE.md` into your target repo. Stack-agnostic — Node, Python, Go, Rust, Java, polyglot all work.

**Time budget:** ~10 min human prep + ~1–2h mostly-idle supervised setup.

1. Read [`modules/agentic-harness/index.md`](./modules/agentic-harness/index.md) for the read order and what gets installed.
2. Do the human steps in `modules/agentic-harness/DAY-0-CHECKLIST.md` (install Claude Code, make sure the target repo has at least one commit, drop the folder in).
3. Paste `modules/agentic-harness/BOOTSTRAP.md` into Claude Code at your repo root and supervise.
4. From Day 2, operate it with `modules/agentic-harness/agentic-harness-tutorial.md`.

**Prerequisites:** Claude Code installed (`npm install -g @anthropic-ai/claude-code`), a git repo with at least one commit, a bash shell.

### Second brain — Obsidian knowledge vault

An Obsidian vault driven by Claude Code: three-layer architecture (immutable raw → Claude-owned wiki → fixed schema), vault-local slash commands, enforcement hooks, cross-device sync. You ingest sources; Claude links and synthesises.

**Time budget:** ~45–60 min human prep + ~2h supervised setup.

1. Read [`modules/second-brain/index.md`](./modules/second-brain/index.md) for the read order.
2. Do the human steps in `modules/second-brain/DAY-0-CHECKLIST.md` (accounts, Homebrew/Node/Python/Claude Code, Obsidian + community plugins, macOS Full Disk Access, a private GitHub remote, sync).
3. Paste `modules/second-brain/BOOTSTRAP.md` into Claude Code and supervise.
4. Operate it day-to-day with `modules/second-brain/second-brain-tutorial.md` and its eight slash commands (`/today`, `/ingest`, `/query`, `/crystallize`, `/weekly-review`, `/tech-digest`, `/new-domain`, `/lint`).

---

## The reference modules (read, no setup)

Three modules are documents, not installs. Open them when you need them.

- **Skills & plugins** — [`modules/skills-guide/useful-skills.md`](./modules/skills-guide/useful-skills.md). An opinionated guide to the plugins, MCP servers, and workflows worth installing on top of Claude Code — what / why / when / for whom / how — plus a procedure to vet third-party plugins for prompt injection. Install à la carte. (Plugin shortcut: `/code-os:skills`.)
- **Tech references** — [`modules/tech-references/index.md`](./modules/tech-references/index.md). A generalized reference stack, an adversarially-verified tool-orchestration compendium (SOTA, June 2026), vendor-neutral production patterns, and a brief-driven workflow that turns them into a plan. Every public claim is tied to a version source or a fetched URL; peer-reviewed, preprint, and vendor-marketing sources are kept distinct. (Plugin shortcut: `/code-os:stack`.)
- **AI tools & people** — [`modules/ai-tools/useful-ai-tools.md`](./modules/ai-tools/useful-ai-tools.md) and [`modules/personas/people-to-follow.md`](./modules/personas/people-to-follow.md). The AI-native tools worth a look and the practitioners shaping agentic engineering.

---

## A note on deferred tools

The setup modules deliberately defer code-intelligence tools (GitNexus), custom memory layers, and speculative MCPs off Day 1. They're earned once the repo or vault is stable — not imposed up front. If anything (or anyone) proposes them on Day 1, push back. The discipline is the point.

---

## Keep it healthy

The harness and the second brain are operated daily through their tutorials, not set-and-forget. Budget about **10 minutes a day** of discipline:

- **Start of session** — let the harness rehydrate from `docs/agent-memory/` before you start working.
- **End of session** — log progress, decisions, and blockers; don't let work ship "done" without verification.
- **Second brain** — ingest as you go; run `/weekly-review` and `/lint` to keep the vault from drifting.

That overhead is what makes the system compound instead of resetting every session. It earns its keep on work that spans multiple sessions — not on throwaway scripts.

---

<sub>In produzione, non in slide. · by yempik. · maintained by Simone Bova</sub>
