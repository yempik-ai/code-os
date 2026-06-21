# Handover Briefs — Index

> **What this folder is.** Everything needed to bootstrap a fresh second-brain vault with Claude Code, then operate it. Three audiences: the **operator** (you), the **Claude Code agent** that executes the setup, and any **third-party LLM** you use to export conversations into the vault.

## Read order

If you are about to set up a vault for the first time, read in this order:
1. `DAY-0-CHECKLIST.md` — what you (the human) must do before the agent can start.
2. `BOOTSTRAP.md` — the prompt you paste into Claude Code to kick off setup.
3. `second-brain-tutorial.md` — the field manual for daily operation once setup is done.

`second-brain-spec.md` and `KNOWLEDGE_MGMT_SETUP_SPEC.md` are the authoritative specs the agent reads during setup — you don't need to read them cover-to-cover, but they're the source of truth if the tutorial and the bootstrap disagree.

## Files

### Operator-facing — read these yourself

- **`DAY-0-CHECKLIST.md`** — The ~45–60 min of human steps that Claude Code cannot do (create Anthropic/GitHub/Obsidian accounts, install Homebrew/Node/Python/Claude Code CLI, install Obsidian + community plugins, grant macOS Full Disk Access, create the private GitHub remote, set up sync). Ends by pointing at `BOOTSTRAP.md` for the handoff.

- **`BOOTSTRAP.md`** — The single file you paste into Claude Code to kick off setup. Contains a self-contained prompt (inside `===== BEGIN PROMPT =====` / `===== END PROMPT =====` fences) that frames the work as a long-running overnight agentic workflow, forces the agent-memory survival kit to be built **first** (so compaction mid-setup doesn't destroy state), then executes `second-brain-spec.md` §10. Also documents checkpoints, failure modes, and recovery recipes.

- **`second-brain-tutorial.md`** — Field manual for the eight vault-local slash commands (`/today`, `/ingest`, `/query`, `/crystallize`, `/weekly-review`, `/tech-digest`, `/new-domain`, `/lint`). Per-command: what it does, how it works under the hood, when to use it, when not to, gotchas. Ends with the typical weekly rhythm these commands compose into and a note on the five shell hooks enforcing invariants behind the scenes.

- **`export-for-second-brain-prompt.md`** — A platform-agnostic prompt to paste into **Claude.ai**, **ChatGPT**, **Gemini**, or **NotebookLM** to export a conversation or notebook as a single markdown file shaped for `/ingest`. The complement to `/crystallize`: `/crystallize` captures in-Claude-Code reasoning; this captures reasoning from other platforms. Includes platform-specific notes and failure-mode recovery prompts.

### Agent-facing — what Claude Code reads during setup

- **`second-brain-spec.md`** (~1,660 lines) — **Primary authoritative spec.** Vault architecture, the six non-negotiables, frontmatter contract, target folder layout (§2), `CLAUDE.md` template (§3), `agent-memory/` files (§4), `wiki/` page types + templates (§5), L1/L2 secret split (§6), skills + MCPs to provision (§7), hook specs (§8), capture pipelines (§9), the executable 30-step setup checklist (§10), sync invariants (§11), operating procedures (§12), failure modes (§13), seed ADRs (§17), and deferred-tool policy (§15, §20). If anything in any other file conflicts with this spec, this spec wins.

- **`KNOWLEDGE_MGMT_SETUP_SPEC.md`** (~1,195 lines) — **Supporting spec** on generic agent-memory discipline for long-running agentic work. Stack-agnostic. §3 defines the survival-kit files (`PROGRESS.md`, `DECISIONS.md`, `BLOCKERS.md`, `VERSIONS.md`, `STACK.md`, and the L1 files). §8 defines operating procedures the agent lives by (session-start rehydration, pre-edit checks, TDD, commit cadence, blocker protocol, verification before completion, context-compaction recovery, long-horizon loops). Referenced from `BOOTSTRAP.md` as the basis for the "knowledge layer first" instruction.

- **`second-brain-spec.pdf`** — PDF rendering of `second-brain-spec.md` (same content). The `.md` is authoritative for the agent; the `.pdf` exists for human readability and archival.

## If you're handing this folder to someone else

Copy the entire folder (all seven files). The minimum paths they need are `DAY-0-CHECKLIST.md` → `BOOTSTRAP.md` → paste → supervise. The two specs and the tutorial are consumed on demand by the agent and the operator respectively; they don't need to be read cover-to-cover up front.
