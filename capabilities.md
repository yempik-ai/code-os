# Capabilities catalog

> The "brain" the installer reasons over. For every module: what it is in plain language, **when to propose it** (the trigger — the user goal that should make Claude offer it), and what it sets up or produces. Claude reads this to *proactively* surface the right capability — because the average engineer doesn't know these features exist.

**How Claude should use this file.** When a user describes what they want, match their words to the `Trigger` of each item below. Propose every match in plain language with a one-line *why it fits them*, and let them say no. Default to proposing anything clearly relevant. Never wait for the user to ask for a capability they don't know exists.

**The shape of code-os.** Five self-contained modules under [`modules/`](modules/). Two are **setup** modules (they install things into a repo or a vault, with a human checklist → a paste-in bootstrap prompt → a tutorial). Three are **reference** docs (read, no setup). The plugin's five slash commands route to all of them. Start anywhere — each module stands alone.

---

## 1. Modules (propose by need)

### ⚙️ `agentic-harness/` — the hero

- **What it is.** A stack-agnostic, drop-in safety + memory harness for running Claude Code on a *real* codebase across long, multi-session, multi-day work. It fixes the four ways agents predictably fail without scaffolding: context compaction destroys working state, multi-day tasks drift and contradict themselves, destructive commands (`rm -rf`, `git push --force`, `git reset --hard`) slip through, and incomplete work ships claimed-done without verification.
- **Trigger.** *"I want Claude Code reliable on a real, multi-session codebase"* — anyone running agents on their day-job repo, a multi-day refactor, or anything that spans more than one session. Not for throwaway scripts or one-shot repos.
- **Setup shape.** `DAY-0-CHECKLIST.md` (~10 min human prep: install Claude Code, ensure the target repo has a git history, drop the folder in) → paste `BOOTSTRAP.md` into Claude Code at the repo root → supervise ~1–2 hours → `agentic-harness-tutorial.md` is the daily field manual. Claude reads the authoritative ~1,200-line `KNOWLEDGE_MGMT_SETUP_SPEC.md` during setup.
- **What it produces** (written into the target repo):
  - `docs/agent-memory/` — the append-only survival kit (`PROGRESS.md`, `DECISIONS.md`, `BLOCKERS.md`, `STACK.md`, `VERSIONS.md`, optional domain files like `PERF.md` / `FRONTEND_NOTES.md` / `COMPLIANCE.md`, plus a `README.md`).
  - `docs/superpowers/` — the spec → plan → execute loop (`specs/`, `plans/`, `executed/` + a `README.md`).
  - `.claude/hooks/` — ~5 shell hooks enforcing safety, auto-logging, and reminders (`block_dangerous_bash.sh`, `log_commit_to_progress.sh`, `remind_tests_on_stop.sh`, `session_start_rehydrate.sh`, `precompact_save.sh`), wired through `.claude/settings.json`.
  - `CLAUDE.md` + `AGENTS.md` at repo root — a north-star briefing tailored to the user's stack and invariants, plus the AI-agents policy.
- **Start:** [`modules/agentic-harness/index.md`](modules/agentic-harness/index.md).

### 🧠 `second-brain/` — personal knowledge vault

- **What it is.** A complete blueprint for an Obsidian-based knowledge vault driven by Claude Code. Three-layer architecture (immutable raw capture → an LLM-owned wiki Claude links and synthesises → a fixed schema), vault-local slash commands, enforcement hooks, survival-kit memory, and cross-device sync. Built to grow over months across work / startups / freelance / tech-news without conflicts, drift, or context loss.
- **Trigger.** *"I want a personal Obsidian knowledge vault"* — anyone who wants a durable second brain that ingests sources and stays coherent over time, not a folder of notes that rots.
- **Setup shape.** `DAY-0-CHECKLIST.md` (~45–60 min of human-only steps: accounts, Homebrew/Node/Python/Claude Code, Obsidian + plugins, macOS Full Disk Access, a private GitHub remote, sync) → paste `BOOTSTRAP.md` → supervise. The agent reads the authoritative `second-brain-spec.md` (~1,660 lines) plus the supporting `KNOWLEDGE_MGMT_SETUP_SPEC.md` during setup.
- **What it produces:** the three-layer vault, eight vault-local slash commands (`/today`, `/ingest`, `/query`, `/crystallize`, `/weekly-review`, `/tech-digest`, `/new-domain`, `/lint`), five shell hooks enforcing the vault invariants, the `agent-memory/` survival kit, and cross-device sync. A companion prompt (`export-for-second-brain-prompt.md`) pulls conversations out of Claude.ai / ChatGPT / Gemini / NotebookLM into a file shaped for `/ingest`.
- **Start:** [`modules/second-brain/index.md`](modules/second-brain/index.md).

### 🔌 `skills-guide/` — curated plugins / MCPs / workflows

- **What it is.** A single-page, opinionated guide to the plugins, MCP servers, and workflows worth installing on top of Claude Code — *what / why / when / for whom / how* — plus a procedure for vetting third-party plugins for prompt injection before you trust them.
- **Trigger.** *"Which plugins / MCPs are actually worth installing?"* or *"is this third-party plugin safe?"*
- **What it produces:** nothing to install — it's a reference. The output is a shortlist you install à la carte, and a security-audit checklist you run before adding anything from outside.
- **Read:** [`modules/skills-guide/useful-skills.md`](modules/skills-guide/useful-skills.md). Reference doc; no setup.

### 📚 `tech-references/` — grounded engineering references

- **What it is.** Citation-heavy references for building production AI/agentic web apps: a reference stack layer-by-layer (framework → runtime → AI layer → data → infra → dev/test, each with a rationale and a June-2026 version line), an adversarially-verified tool-orchestration compendium (universal principles → per-provider deep dives → MCP → frameworks → failure modes → evaluation), and vendor-neutral production patterns/anti-patterns. Every public claim is tied to a version source or a fetched URL; peer-reviewed / preprint / vendor-marketing distinctions stay explicit.
- **Trigger.** *"What should I build this with?"* or *"how should agents orchestrate tools?"*
- **What it produces:** a **brief**. You don't read these cover-to-cover — you point Claude at the folder, hand it the copy-paste prompt template, and it returns a grounded `brief-<feature>.md` (Roadmap / Rationale / Stack / Risks / Open decisions) that flags anything outdated instead of trusting the docs blindly. That brief is the spec the `agentic-harness/` then executes against.
- **Read:** [`modules/tech-references/index.md`](modules/tech-references/index.md) for the folder map and the brief workflow.

### 🛠️ `ai-tools/` + `personas/` — tools & people

- **What it is.** A short list of AI-native tools worth a look, and the practitioners and researchers shaping agentic engineering. Counted together as the fifth module.
- **Trigger.** *"What tools should I track?"* or *"who should I follow to keep up with the field?"*
- **What it produces:** nothing to install — pointers worth bookmarking.
- **Read:** [`modules/ai-tools/useful-ai-tools.md`](modules/ai-tools/useful-ai-tools.md) and [`modules/personas/people-to-follow.md`](modules/personas/people-to-follow.md).

---

## 2. Plugin commands → goals

Install the plugin in Claude Code with `/plugin marketplace add yempik-ai/code-os` then `/plugin install code-os@code-os`. It ships an always-on core skill (`code-os-core`) plus five slash commands. Each maps to a goal above.

| Command | Plain-language job | Routes to |
|---|---|---|
| **`/code-os:install`** | Guided router. Asks what you want, then sends you to the right module. Use it when you're not sure where to start. | any module |
| **`/code-os:harness`** | Drives the agentic-harness install into your current repo end-to-end. | `agentic-harness/` |
| **`/code-os:second-brain`** | Drives the second-brain (Obsidian vault) setup. | `second-brain/` |
| **`/code-os:skills`** | Surfaces the curated skills/plugins/MCP guide and the prompt-injection security-audit procedure. | `skills-guide/` |
| **`/code-os:stack`** | Runs the brief-driven tech-references workflow — reference stack + orchestration patterns → a grounded plan. | `tech-references/` |

---

## 3. Mapping cheat-sheet (what the user says → propose)

- *"I want Claude Code to behave reliably on my real codebase / multi-day work"* → `agentic-harness/` (`/code-os:harness`).
- *"I want a personal Obsidian knowledge vault / a second brain"* → `second-brain/` (`/code-os:second-brain`).
- *"Which plugins or MCPs are worth installing?"* / *"is this plugin safe to trust?"* → `skills-guide/` (`/code-os:skills`).
- *"What should I build this with?"* / *"how should agents orchestrate tools?"* → `tech-references/` brief workflow (`/code-os:stack`).
- *"What tools should I track / who should I follow?"* → `ai-tools/` + `personas/`.
- *"I don't know where to start"* → `/code-os:install` (the router asks, then routes).
- *Planning then building* → `/code-os:stack` produces the brief; `/code-os:harness` executes it. One artifact (`brief-<feature>.md`), two repos.

---

## 4. Deferred-tool discipline

**Deferred tools are deferred for a reason.** Both setup modules ship intentionally lean. Code-intelligence tools (GitNexus), custom memory layers, and speculative MCPs are **not** installed on Day 1 — they are earned once the repo or vault is stable, never imposed up front (see §12 of the agentic-harness spec, §15/§20 of the second-brain spec). If a user proposes one of these on Day 1, **push back**: name the cost, point at the deferred-tool policy, and offer the lean path first.

**Grounding discipline.** Anything sourced from `tech-references/` carries its grounding with it. Tie every non-trivial claim to a version source or a fetched URL, keep the peer-reviewed / preprint / vendor-marketing distinction explicit, and re-verify "latest" before pinning a version — the stack drifts weekly.

**Surface capabilities proactively.** The user usually doesn't know these exist. When their goal matches a trigger, propose the capability in plain language with a one-line reason — don't wait to be asked.

---

<sub>In produzione, non in slide. · by yempik. · maintained by Simone Bova</sub>
