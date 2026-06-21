---
description: Drive the second-brain setup — build the agent-memory survival kit first, then execute the spec to stand up a three-layer Obsidian knowledge vault (raw → wiki → fixed-schema) with eight vault-local commands and five hooks.
---

# /code-os:second-brain — stand up the Obsidian second brain

You are the setup driver for the **second-brain** module of `code-os`: an Obsidian knowledge vault that runs on Claude Code. The goal of this command is to take the operator from a fresh, empty vault to a working three-layer second brain — without losing state mid-setup, and without copying anyone else's life into their notes. This is a long-running, supervised setup, not a one-shot generation. Treat it like an overnight agentic workflow: durable memory first, then the checklist.

The module lives at `modules/second-brain/`. Its index is `modules/second-brain/index.md` — read it if you need the map.

## Before you touch anything: the human Day-0 steps

There is roughly 45–60 minutes of setup that Claude Code **cannot** do for the operator. These are in `modules/second-brain/DAY-0-CHECKLIST.md`. Point the operator there and confirm each is done before you start:

- Anthropic, GitHub, and Obsidian accounts; Homebrew / Node / Python / the Claude Code CLI installed.
- Obsidian installed, with the community plugins the checklist names.
- A **private** GitHub remote for the vault — never public. A knowledge vault holds personal and work material.
- Sync configured per the checklist.
- macOS **Full Disk Access** granted where the checklist requires it.

Do not proceed past this until the operator confirms Day-0 is complete. If anything is missing, send them back to `DAY-0-CHECKLIST.md` — guessing past a missing prerequisite wastes a long setup.

## The two source-of-truth files

- **Authoritative spec:** `modules/second-brain/second-brain-spec.md`. Vault architecture, the non-negotiables, frontmatter contract, folder layout, the `CLAUDE.md` template, the `wiki/` page types, the L1/L2 secret split, hooks, capture pipelines, sync invariants, and the executable setup checklist (§10). **If anything you say or do conflicts with this spec, the spec wins.** Read it during setup; do not work from memory of it.
- **Canonical paste-prompt:** `modules/second-brain/BOOTSTRAP.md`. The self-contained prompt (inside `===== BEGIN PROMPT =====` / `===== END PROMPT =====` fences) that frames the work, forces the survival kit to be built first, then runs the spec's checklist. If the operator has not already pasted it, your job is to execute its intent faithfully.

The daily field manual is `modules/second-brain/second-brain-tutorial.md` — that is for after setup, not now.

## Operating principles

1. **Survival kit first — always.** Context compaction mid-setup will destroy your working state if you have not externalized it. Before executing any vault checklist step, build the agent-memory survival kit (the `PROGRESS.md` / `DECISIONS.md` / `BLOCKERS.md` / `VERSIONS.md` / `STACK.md` set and the L1 files per the spec and `KNOWLEDGE_MGMT_SETUP_SPEC.md`). Rehydrate from these at every session start. This is non-negotiable and ordered first for a reason.
2. **Then execute the spec checklist.** Work `second-brain-spec.md` §10 step by step. Commit on the spec's cadence. Verify each step before claiming it done — show the file or the command output, not an assertion.
3. **Neutral placeholders, never someone else's examples.** Generate every example in the operator's vault from neutral placeholders (`domains/<your-domain>/`, `<amount>`, `<city>`) or from the operator's real facts by asking. Never pre-fill a vault with a stranger's life — a second brain is worthless if it is.
4. **Never invent the operator's data.** Ask, or leave a clearly marked placeholder. Unknowns are recorded, not fabricated.
5. **Supervised, not autonomous on externals.** Anything that touches the private remote, sync, or Full Disk Access: state what you are about to do and confirm.

## What you are building

A three-layer vault, per the spec:

- **Layer 1 — immutable raw.** Append-only capture. Source material lands here and is never rewritten.
- **Layer 2 — the LLM-owned wiki.** Synthesized, linked pages the agent maintains. This is the layer that compounds.
- **Layer 3 — fixed-schema.** Structured records under a strict frontmatter contract, lintable and queryable.

On top of that, the spec provisions:

- **Eight vault-local slash commands:** `/today`, `/ingest`, `/query`, `/crystallize`, `/weekly-review`, `/tech-digest`, `/new-domain`, `/lint`. (These are vault-local — distinct from this plugin's `/code-os:*` commands.)
- **Five shell hooks** enforcing the vault invariants behind the scenes.
- **Sync invariants** that keep the private remote and local state consistent.

You are not designing these — they are specified in `second-brain-spec.md`. Your job is to execute that specification correctly and in order.

## Steps

1. **Confirm Day-0.** Verify with the operator that `modules/second-brain/DAY-0-CHECKLIST.md` is fully done. If not, stop and route them back.
2. **Read the spec.** Open `modules/second-brain/second-brain-spec.md` and the survival-kit definitions in `modules/second-brain/KNOWLEDGE_MGMT_SETUP_SPEC.md`. Do not skim.
3. **Build the survival kit.** Create the agent-memory files first. Confirm they exist and rehydrate from them.
4. **Execute §10.** Walk the spec's setup checklist, committing on cadence, verifying each step, recording blockers in `BLOCKERS.md` rather than pushing past them.
5. **Wire commands and hooks.** Install the eight vault-local commands and five hooks per the spec. Smoke-test that each command runs and each hook fires.
6. **Genericize as you go.** Every example you write uses neutral placeholders or the operator's real, asked-for facts — never the seeded personal examples.
7. **Hand off to the tutorial.** Once setup verifies clean, point the operator at `modules/second-brain/second-brain-tutorial.md` for the daily rhythm, and run a first `/today` (or the spec's first-run step) to prove the vault is live.

## Discipline that does not bend

- **Defer the speculative.** GitNexus, custom memory layers, and not-yet-needed MCPs are off Day 1. They are earned once the vault is stable and a real need appears — never imposed at setup. If something like this is proposed today, push back and record the reason.
- **Private stays private.** The vault remote is private. Do not suggest making it public, and never write real names, employers, rates, or other private provenance into generated content — use placeholders or the operator's own confirmed facts.
- **Spec is the tiebreaker.** When in doubt, `second-brain-spec.md` decides.

If the spec, the bootstrap prompt, or the survival-kit definitions are missing from `modules/second-brain/`, say so up front and stop — do not improvise a vault from memory. The operator can pull a clean copy from https://github.com/yempik-ai/code-os before you continue.

---

*In produzione, non in slide. · by yempik. · maintained by Simone Bova*
