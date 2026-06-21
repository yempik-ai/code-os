# code-os — Index

> **What this repo is.** The engineer's operating system for Claude Code: five independent, drop-in modules that make AI coding agents reliable on real work. Each module under `modules/` is self-contained — use one without the others.
>
> **The pair.** `code-os` (this repo, [Simone Bova](https://www.linkedin.com/in/simone-bova/)) is the builder's counterpart to **[`cowork-os`](https://github.com/yempik-ai/cowork-os)** (Raffaele Zarrelli) — Claude Code for engineers vs. Claude Cowork for business teams. Two systems, one hat: **[yempik.](https://yempik.com)**
>
> **Audiences.** Two readers: the **operator** (the human running Claude Code) and **Claude itself** when pointed at this repo. Each module has its own internal `index.md` for deeper navigation; this file is the root map.
>
> **Install.** `code-os` runs as a Claude Code plugin (`/plugin marketplace add yempik-ai/code-os`) for the runnable core — an always-on skill + five commands. Or clone the full kit and paste [`INSTALL.md`](INSTALL.md), which asks your goal and routes you. Setup paths: [`GETTING_STARTED.md`](GETTING_STARTED.md); catalog of what gets set up: [`capabilities.md`](capabilities.md).

---

## Repo layout at a glance

```text
code-os/
├── README.md                    ← GitHub landing page (the front door)
├── index.md                     ← you are here (the full repo map)
├── INSTALL.md                   ← guided router — paste into Claude Code, it routes you
├── GETTING_STARTED.md           ← the two setup paths (guided / per-module)
├── capabilities.md              ← catalog of what each module sets up (the installer's brain)
├── CONTRIBUTING.md              ← contribution rules
├── LICENSE                      ← MIT
│
├── .claude-plugin/              ← marketplace manifest
├── plugins/code-os/             ← installable plugin: core skill + 5 slash commands
├── docs/                        ← README / social assets
│
└── modules/
    ├── agentic-harness/         ← drop-in agent-memory + safety harness for any repo
    │   ├── index.md
    │   ├── DAY-0-CHECKLIST.md
    │   ├── BOOTSTRAP.md
    │   ├── KNOWLEDGE_MGMT_SETUP_SPEC.md
    │   └── agentic-harness-tutorial.md
    │
    ├── second-brain/            ← Obsidian knowledge vault on Claude Code
    │   ├── index.md
    │   ├── DAY-0-CHECKLIST.md
    │   ├── BOOTSTRAP.md
    │   ├── second-brain-spec.md
    │   ├── KNOWLEDGE_MGMT_SETUP_SPEC.md
    │   ├── second-brain-tutorial.md
    │   └── export-for-second-brain-prompt.md
    │
    ├── tech-references/         ← grounded engineering references
    │   ├── index.md              ← folder hub + the brief-driven workflow
    │   ├── reference-stack.md    ← the reference stack, layer by layer (generalized)
    │   ├── tool-orchestration-best-practices-and-resources.md
    │   └── patterns/             ← vendor-neutral production patterns + anti-patterns
    │
    ├── skills-guide/            ← curated plugins / MCPs / workflows
    │   └── useful-skills.md
    │
    ├── ai-tools/                ← AI-native tools worth a look
    │   └── useful-ai-tools.md
    │
    └── personas/                ← people worth following in agentic engineering
        └── people-to-follow.md
```

---

## The five modules

### `modules/agentic-harness/` — the hero

- **What.** Stack-agnostic drop-in that installs survival-kit files (`docs/agent-memory/`), a spec → plan → execute loop (`docs/superpowers/`), ~5 shell hooks (PreToolUse / PostToolUse / Stop / SessionStart / PreCompact), and a tailored `CLAUDE.md` into a target git repo.
- **Why.** Claude Code on real codebases fails in four predictable ways without scaffolding: context compaction destroys state, multi-day tasks drift, destructive commands slip through, work ships claimed-done without verification. The harness fixes all four with file-based discipline.
- **How.** ~10 min of human prep (`DAY-0-CHECKLIST.md`), then paste `BOOTSTRAP.md` into Claude Code at your repo root and supervise ~1–2 hours. Start: [`modules/agentic-harness/index.md`](modules/agentic-harness/index.md).

### `modules/second-brain/` — personal knowledge vault

- **What.** A complete blueprint for an Obsidian-based knowledge vault driven by Claude Code: three-layer architecture (immutable raw → LLM-owned wiki → fixed schema), vault-local slash commands, enforcement hooks, survival-kit memory, cross-device sync.
- **Why.** A vault that grows over months across work / startups / freelance / tech-news without conflicts, drift, or context loss. The wiki is owned by Claude — you ingest sources, it links and synthesises.
- **How.** ~45–60 min of human-only steps (`DAY-0-CHECKLIST.md`), then paste `BOOTSTRAP.md` and supervise. Start: [`modules/second-brain/index.md`](modules/second-brain/index.md).

### `modules/tech-references/` — grounded engineering references

- **What.** Citation-heavy references: a reference stack (generalized), an adversarially-verified tool-orchestration compendium (June 2026), and vendor-neutral production patterns/anti-patterns — plus a brief-driven workflow.
- **Why.** To plan agentic-coding work on *grounded* best practices rather than folklore. Every public claim is tied to a version source or fetched URL.
- **How.** Start at [`modules/tech-references/index.md`](modules/tech-references/index.md) for the folder map and the copy-paste brief workflow.

### `modules/skills-guide/` — curated plugins / MCPs / workflows

- **What.** Single-page opinionated guide to the plugins, MCP servers, and workflows worth installing on top of Claude Code, plus how to vet third-party plugins for prompt injection.
- **How.** Read [`modules/skills-guide/useful-skills.md`](modules/skills-guide/useful-skills.md). Reference doc; no setup.

### `modules/ai-tools/` + `modules/personas/` — tools & people

- **What.** A short list of AI-native tools worth a look, and the practitioners/researchers shaping agentic engineering.
- **How.** Read [`modules/ai-tools/useful-ai-tools.md`](modules/ai-tools/useful-ai-tools.md) and [`modules/personas/people-to-follow.md`](modules/personas/people-to-follow.md).

---

## How to use this repo

1. **"I want Claude Code to behave reliably on my codebase."** → [`modules/agentic-harness/index.md`](modules/agentic-harness/index.md) → `DAY-0-CHECKLIST.md` → `BOOTSTRAP.md`. ~10 min human + ~1–2h supervised.
2. **"I want a personal knowledge vault."** → [`modules/second-brain/index.md`](modules/second-brain/index.md) → `DAY-0-CHECKLIST.md` → `BOOTSTRAP.md`. ~1h human + ~2h supervised.
3. **"Which plugins are worth installing?"** → [`modules/skills-guide/useful-skills.md`](modules/skills-guide/useful-skills.md). Reference doc.
4. **"What should I build this with, and how should agents orchestrate tools?"** → [`modules/tech-references/index.md`](modules/tech-references/index.md). Reference docs; no setup.
5. **"Who should I follow to track the field?"** → [`modules/personas/people-to-follow.md`](modules/personas/people-to-follow.md).

The agentic-harness and second-brain modules are independent. They share the same underlying agent-memory discipline (the `KNOWLEDGE_MGMT_SETUP_SPEC.md` in each is similar but tailored to its target — codebase vs. vault).

## Notes for Claude / coding agents reading this repo

- **Each module is self-contained.** When asked to operate from one, treat its spec as authoritative. Do not cross-reference another module's spec unless explicitly asked.
- **`BOOTSTRAP.md` files contain self-contained prompts** inside `===== BEGIN PROMPT =====` / `===== END PROMPT =====` fences. The prompt is the brief — read the referenced files top-to-bottom first, establish the knowledge layer first, then run the setup checklist.
- **Deferred tools are deferred for a reason.** Both harnesses defer code-intelligence tools, memory layers, and speculative MCPs off Day 1. Push back if anyone proposes them on Day 1.
- **`modules/skills-guide/useful-skills.md` is the curated install list.** Recommend from there rather than improvising.
- **`modules/tech-references/` holds grounded references, not setups.** Every public claim is tied to a version source or fetched URL — preserve that discipline, and keep peer-reviewed / preprint / vendor-marketing distinctions explicit.
