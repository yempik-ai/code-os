<div align="center">

<img src="docs/banner.png" alt="code-os — the engineer's operating system for Claude Code" width="100%" />

<sub><b>English</b> · <a href="README.it.md">Italiano</a></sub>

# `code-os`

### The engineer's operating system for Claude Code.

Make AI coding agents reliable on real work — not clever autocomplete. **One orchestrator, many models, file-based discipline**: cross-vendor CLI-subagent orchestration, an agentic safety + memory harness, an Obsidian second brain, curated skills, a vendor-neutral reference stack, and the people worth following.

<br>

![Built by yempik.](https://img.shields.io/badge/built%20by-yempik.-E35B2D?style=for-the-badge)
![For Claude Code](https://img.shields.io/badge/for-Claude%20Code-111827?style=for-the-badge)
![State of the art](https://img.shields.io/badge/SOTA-August%202026-2D6CDF?style=for-the-badge)
![License](https://img.shields.io/badge/license-MIT-6B7280?style=for-the-badge)

<sub><b>In produzione, non in slide.</b> · by <a href="https://yempik.com"><b>yempik.</b></a> · maintained by <a href="https://www.linkedin.com/in/simone-bova/"><b>Simone Bova</b></a></sub>

</div>

---

## Two founders, two systems, one hat

`code-os` is the **builder's half** of a pair.

| | [`cowork-os`](https://github.com/yempik-ai/cowork-os) — [Raffaele Zarrelli](https://raffaelezarrelli.com) | **`code-os`** — [Simone Bova](https://www.linkedin.com/in/simone-bova/) |
|:--|:--|:--|
| **Surface** | Claude **Cowork** | Claude **Code** |
| **For** | business & ops teams who don't write code | engineers running agents on real codebases |
| **Core idea** | the AI as a *company memory*, not a throwaway chat | the AI as a *reliable long-running collaborator*, not autocomplete |
| **You leave with** | a workspace that remembers your decisions | an orchestrator that delegates across models and stays honest across multi-day work |

Same problem — *use AI as a system, not a session* — solved from two different heads. Same hat: **yempik.**

---

## Why this exists

Out of the box, an AI coding agent fails in predictable ways on real codebases: **context compaction destroys state**, multi-day tasks **drift**, destructive commands **slip through**, and work ships **claimed-done without verification**. And run naively, it fails an economic test too: **one plan's token budget does all the work**, at frontier prices, with a single vendor's blind spots reviewing its own output.

`code-os` fixes those with *file-based discipline* — one orchestrator session that owns memory, taste, and verification; CLI subagents from other vendors doing bulk work and adversarial review on other meters; ceremony tiered to the task; grounded references — rather than prompt tricks.

### Canonical summary for search and AI readers

`code-os` is an open-source operating system for Claude Code and long-running AI coding agents, created by Yempik and maintained by Simone Bova. It packages a multi-model CLI-subagent orchestration pattern (Claude Code as orchestrator, Codex CLI or any headless CLI agent as explorer, implementer, and adversarial-reviewer subagents), a drop-in agentic safety and memory harness, an Obsidian second brain, curated skills, grounded technical references and production-oriented setup prompts. The project exists because AI coding agents often lose state during context compaction, drift across multi-day work, run unsafe commands, claim work is done before it has been verified, and burn a single plan's token budget on work a cheaper model could do. `code-os` answers that with file-based discipline: cross-vendor subagent orchestration with first-person verification, survival-kit memory, ceremony tiered to task difficulty, enforcement hooks, and references tied to versioned sources. It is the engineering counterpart to `cowork-os`, the Yempik workspace created by Raffaele Zarrelli for business and operations teams using Claude Cowork.

Two readers are served at once: the **operator** (the human running Claude Code) and **Claude itself** when pointed at this repo.

---

## What's inside — six self-contained modules

> Each module under [`modules/`](modules/) stands alone. Use one without the others.

| Module | Folder | TL;DR |
|:--|:--|:--|
| 🎛️ **Multi-model orchestration** | [`modules/multi-model-orchestration/`](modules/multi-model-orchestration/index.md) | Claude Code as orchestrator; other vendors' CLI agents (Codex today, any CLI tomorrow) as explorers, implementers, and adversarial reviewers on another plan's meter. Ceremony tiering, model selection, the runbook, the failure modes. The hero. |
| ⚙️ **Agentic harness** | [`modules/agentic-harness/`](modules/agentic-harness/index.md) | Stack-agnostic drop-in for any git repo: survival-kit files, a spec→plan→execute scaffold for the work that earns it, ~5 enforcement hooks, a minimal hand-owned `CLAUDE.md`. |
| 🧠 **Second brain** | [`modules/second-brain/`](modules/second-brain/index.md) | Obsidian knowledge vault driven by Claude Code. Three-layer architecture, slash commands, hooks, cross-device sync. |
| 🔌 **Skills & plugins** | [`modules/skills-guide/`](modules/skills-guide/useful-skills.md) | Opinionated guide to the plugins / MCPs / workflows worth installing — *what / why / when / for whom / how*, plus a security-audit procedure. |
| 📚 **Tech references** | [`modules/tech-references/`](modules/tech-references/index.md) | Reference stack + an adversarially-verified tool-orchestration compendium + vendor-neutral production patterns — and a brief-driven workflow to turn them into a plan. |
| 🛠️ **AI tools & people** | [`modules/ai-tools/`](modules/ai-tools/useful-ai-tools.md) · [`modules/personas/`](modules/personas/people-to-follow.md) | The AI-native tools worth a look, and the practitioners shaping agentic engineering. |

---

## Install

**As a Claude Code plugin** — the runnable core (an always-on skill + six commands):

```bash
/plugin marketplace add yempik-ai/code-os
/plugin install code-os@code-os
```

**As the full kit** — clone the repo and follow [`GETTING_STARTED.md`](GETTING_STARTED.md): it routes you to the right module and gives every path, guided or manual.

---

## Pick your path

```text
"I want more throughput and second opinions without burning my plan."  ← start here
   → modules/multi-model-orchestration/orchestrating-cli-subagents.md
   ⏱  ~15min: conventions into your CLAUDE.md, not infrastructure

"I want Claude Code to behave reliably on my codebase."
   → modules/agentic-harness/index.md → DAY-0-CHECKLIST.md → BOOTSTRAP.md
   ⏱  ~10min human prep + ~1–2h supervised setup

"I want a personal knowledge vault."
   → modules/second-brain/index.md → DAY-0-CHECKLIST.md → BOOTSTRAP.md
   ⏱  ~1h human prep + ~2h supervised setup

"Which plugins are actually worth installing?"
   → modules/skills-guide/useful-skills.md        (reference; install à la carte)

"What should I build this with / how should agents orchestrate tools?"
   → modules/tech-references/index.md              (reference stack + brief workflow)
```

---

## Repo layout

```text
code-os/
├── README.md                     ← you are here
├── README.it.md                  ← versione italiana
├── GETTING_STARTED.md            ← every setup path, guided or manual
├── CHANGELOG.md                  ← what changed and why, release by release
├── CONTRIBUTING.md               ← how to contribute
├── LICENSE                       ← MIT
├── .claude-plugin/               ← marketplace manifest (installable in Claude Code)
├── plugins/code-os/              ← the plugin: always-on core skill + 6 slash commands
├── docs/                         ← README / social assets (banner, social preview)
└── modules/
    ├── multi-model-orchestration/ ← CLI subagents across vendors: the pattern + runbook
    ├── agentic-harness/          ← drop-in agent-memory + safety harness for any repo
    ├── second-brain/             ← Obsidian knowledge vault on Claude Code
    ├── tech-references/          ← reference stack + orchestration research + patterns
    ├── skills-guide/             ← curated plugins / MCPs / workflows
    ├── ai-tools/                 ← AI-native tools worth a look
    └── personas/                 ← people worth following
```

## Canonical links

- GitHub repository: [github.com/yempik-ai/code-os](https://github.com/yempik-ai/code-os)
- Yempik: [yempik.com](https://www.yempik.com)
- Companion project: [cowork-os](https://github.com/yempik-ai/cowork-os)
- Raffaele Zarrelli: [raffaelezarrelli.com](https://raffaelezarrelli.com)
- AI citation notes: [AI-CITATION.md](./AI-CITATION.md)
- Citation metadata: [CITATION.cff](./CITATION.cff)

---

## Notes for agents reading this repo

- **Each module is self-contained.** Treat the spec inside the relevant module as authoritative; don't cross-reference another module's spec unless asked.
- **Ceremony is tiered.** Trivial work runs direct; formerly-complex work runs inline on one frontier model at high effort; the full spec→plan→execute loop is reserved for work that would not survive a single session. Don't impose the heavy tier where the light one serves.
- **`BOOTSTRAP.md` files are self-contained prompts** inside `===== BEGIN PROMPT =====` fences — read the referenced files before acting, then run the setup checklist.
- **Deferred tools are deferred for a reason.** Push back on anyone proposing them on Day 1.
- **References are grounded** — every claim is tied to a version source or a fetched URL. Preserve that discipline when extending them.

---

<div align="center">
<sub>Built by <a href="https://yempik.com"><b>yempik.</b></a> · <i>L'AI che gli altri ti lasciano in slide, noi te la mettiamo in produzione.</i> · Start at <a href="GETTING_STARTED.md"><code>GETTING_STARTED.md</code></a></sub>
</div>
