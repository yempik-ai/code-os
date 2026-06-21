# BOOTSTRAP — Hand-off to Claude Code

> **For the operator (you).** This is the single file you paste into Claude Code to kick off the full second-brain setup. It assumes you have already completed the Day-0 human steps in `DAY-0-CHECKLIST.md` (Parts 1–9 — accounts, toolchain, Obsidian install + plugins, Local REST API key, GitHub repo, sync). If you skipped any of those, go finish them first — Claude Code cannot click GUI toggles, install apps from the Mac App Store, or enter a credit card for you.

## How to use this file

1. Open **Terminal.app**.
2. Run: `cd ~/second-brain`
3. Run: `claude`
4. Wait for the `>` prompt.
5. **Copy the entire block between `===== BEGIN PROMPT =====` and `===== END PROMPT =====` below** and paste it as your first message. Do **not** paste this header or the post-script — those are for you, not the agent.
6. Supervise. Approve tool calls as they come. Read each of the five hook shell scripts before approving them (they run on every tool call — you want to know what they do).

Expected wall-clock: ~1–2 hours of mostly-idle supervision on Day 1. Do other work; check back when Claude Code asks a question.

---

===== BEGIN PROMPT =====

You are about to execute a **long-running, overnight-scale agentic workflow** to set up a production-grade personal knowledge-management system ("second brain") for me, the operator. This is a research engagement — your final outputs will be evaluated for completeness, discipline, and fidelity to the specs. Token budget is effectively unbounded; optimize for meticulous, auditable quality, not speed.

## 0. Read these three files top-to-bottom before doing anything else

1. `handover-birefs/second-brain-spec.md` — **primary**. Vault architecture, frontmatter contract, ingestion/query/lint rules, §10 setup checklist, §8 hook specs, §7 skill specs, §17 seed ADRs.
2. `handover-birefs/KNOWLEDGE_MGMT_SETUP_SPEC.md` — **supporting**. Generic agent-memory / survival-kit discipline for long-running agentic work. §3 defines `agent-memory/` files. §8 defines operating procedures (session-start, blocker protocol, compaction recovery, long-horizon loops).
3. `handover-birefs/DAY-0-CHECKLIST.md` — **status**. What the human has already done before handing off to you. Do not redo Parts 1–9.

Read all three fully. Do not skim. Do not start executing until you have read every section.

## 1. Establish the knowledge layer FIRST (before any vault work)

This is the most important instruction in this prompt. **Do not start on the vault spec's Phase 1 step 3 until the agent-memory layer exists and is being written to.**

Why: this is a multi-day, multi-session, multi-agent task that will cross context-compaction boundaries. Without a durable survival kit, every compaction or fresh session destroys your working state and you will redo work, contradict yourself, or silently drift. With the survival kit, any future Claude session (including ones spawned as subagents, or ones that start after you hit your context limit) can rehydrate from durable files and continue coherently.

Concretely, **as your very first actions after reading the three specs:**

1. Create `agent-memory/` with the files specified in `second-brain-spec.md` §4 and `KNOWLEDGE_MGMT_SETUP_SPEC.md` §3:
   - `PROGRESS.md` — append-only macro log. Every completed step gets a line. Every commit gets a line.
   - `DECISIONS.md` — personal ADRs (start with the seed ADRs from §17 of the vault spec once you reach them).
   - `BLOCKERS.md` — append-only, with resolutions.
   - `OPEN-THREADS.md` — what's unresolved right now.
   - `PREFERENCES.md` — how I want you to talk to me (will grow as preferences emerge; seed with what you can infer from this prompt + the specs).
   - `TIMELINE.md` — macro events for historical context (empty to start).
   - `plans/YYYY-MM-DD-second-brain-setup-plan.md` — the executable plan for this setup, broken into phases and unchecked checkboxes per §10 of the vault spec. This is your task list for the next several sessions.
2. Write `agent-memory/README.md` describing the survival-kit protocol in your own words — so a future fresh session reading only this folder can rehydrate.
3. Commit: `chore(memory): seed agent-memory scaffold`.
4. **Only then** proceed to Phase 1 step 3 of the vault spec.

At every subsequent step, before claiming a step done:
- Append a line to `PROGRESS.md` (format per §4.1 of the vault spec).
- If blocked ≥30 min, write to `BLOCKERS.md` with a `[!blocker]` callout per §1.6, stub, move on.
- Make a commit with the §8.4 message convention.

Session-start protocol (now, and every future session): read `agent-memory/PROGRESS.md` last non-COMMIT line, read `OPEN-THREADS.md`, read `BLOCKERS.md` entries without resolutions, read the active plan in `agent-memory/plans/`, read `wiki/hot.md` if it exists. **Only then start work.**

## 2. Execute the vault spec

After the knowledge layer is seeded and committed, execute `second-brain-spec.md` §10 "Setup checklist" — the 30-step executable plan — **starting from Phase 1 step 3** (folder structure). Phase 1 steps 1–2 (install Obsidian + community plugins) are already done; `DAY-0-CHECKLIST.md` is the evidence.

Use the **Obsidian+ sync path** from §A.5 unless I tell you otherwise.

You are responsible for **writing, not just installing**, every piece of the system from the behavioral specifications in the briefs:

- The five hook shell scripts from §8 (`block_dangerous_writes.sh`, `block_dangerous_bash.sh`, `log_ingestion_to_log_md.sh`, `hot_cache_on_stop.sh`, `session_start_context.sh`, `precompact_save.sh`) and `.claude/settings.json` wiring them.
- The eight vault-local skills from §7.2 (`/ingest`, `/query`, `/lint`, `/today`, `/crystallize`, `/weekly-review`, `/tech-digest`, `/new-domain`).
- All ten templates in `_templates/` (daily, meeting, person, client, project, wiki-concept, wiki-entity, wiki-source, decision, blocker).
- Each domain's `_moc.md` and `_schema.md`.
- `CLAUDE.md`, `AGENTS.md`, `README.md`, `.gitignore`, `.gitattributes`.
- The seven seed ADRs from §17.
- The `.mcp.json` for the Obsidian MCP.

Do **not** ask me for source code. You have the behavioral specs — build from them.

## 3. Interview me briefly before writing `CLAUDE.md`

Before you write `CLAUDE.md`, conduct a short structured interview to capture the identity-level facts that cannot be inferred. Ask about:

- Full name.
- Employer name, role, reporting line, start date.
- Startup codename(s), stage, co-founder(s), sector.
- Freelance sectors / key clients.
- Partner / family / personal anchors relevant to life-admin tracking.
- Primary domains (the top-level buckets `domains/` should contain).

Do not invent any of this. If I decline to answer any item, mark it `[TBD — populate on user upload]` in `CLAUDE.md` and log it to `OPEN-THREADS.md`. The vault spec §3 shows the target `CLAUDE.md` shape.

## 4. Operating discipline — the rules you live by during this setup

- **Commit after every step.** Not every file — every step in the plan. Use the §8.4 message convention: `<type>(<scope>): <subject>`. After each commit, verify the PostToolUse hook appended to `wiki/log.md` (once hooks are installed); before hooks exist, append to `agent-memory/PROGRESS.md` by hand.
- **30-minute stall rule.** If you are stuck on one step for >30 min with no forward progress, stub the work, add a `[!blocker]` callout in context, log to `BLOCKERS.md`, continue to the next step. Perfectionism is the enemy of compounding. Come back to blockers later.
- **Meticulous detail, don't stop.** Within a step, do not ship shallow. §10-bis of the vault spec is the detail-preservation rule — meeting pages 80–150 lines, concept pages 80–150 lines for substantive workstreams, source pages 40–80 lines. If you run out of context mid-step, commit what's there with a `[!blocker]` pointing at the depth gap, log to `BLOCKERS.md`, do not paper over it.
- **Ask me only when a decision is genuinely ambiguous or needs my personal information or a destructive action is required.** Do not ask for approval on routine steps. Examples of when to ask:
  - Identity-interview items (§3 above).
  - Obsidian Local REST API key (Phase 2 step 13 of the vault spec).
  - Any OAuth flow for an MCP (GitHub, Gmail, Calendar if I request them) — walk me through the browser flow; you cannot complete these headlessly.
  - A destructive-looking shell command (`rm -rf`, `git push --force`, `git reset --hard`). Stop. Explain why. Wait for my OK.
- **Never invent facts** about me, my employer, colleagues, clients, or projects. If unknown, say so and log to `OPEN-THREADS.md`. The `## Who / What` and `## Domains` sections of `CLAUDE.md` must carry `[TBD — populate from user upload]` rather than fabrication.
- **Never store secrets in the git-tracked vault.** Credentials, health records, financial account details belong only in `~/.second-brain-local/` (L1). See §6 of the vault spec.
- **Never silently overwrite a contradiction.** Use the `[!contradiction]` callout pattern (§5.6 of the vault spec) and add to `wiki/contradictions.md`.
- **Verification before completion.** Before claiming any step done, run the verification named in the plan (`ls`, `cat`, `bash <hook> <stdin>`, synthetic test commit, etc.). Evidence before assertion.
- **No deferred tools on Day 1.** Do not install any of: `graphify`, `gitnexus`, custom voice-capture pipelines, RAG-Anything, dedicated memory layers (OpenMemory/Supermemory). §15 and §20 of the vault spec explicitly defer these. `graphify` goes in only after the vault has 20+ real sources. `gitnexus` only if a code subdirectory appears later. Push back if any prior session proposed one.

## 5. Checkpoints — when to pause and wait for me

Pause and wait for my explicit OK at these points:

1. **After reading the three briefs** — summarize the full setup plan back to me in ≤10 lines so I can confirm we're aligned. Then wait.
2. **After the identity interview** — echo back what you captured in structured form. Then wait before writing `CLAUDE.md`.
3. **Before I paste the Obsidian Local REST API key** — you will prompt me at Phase 2 step 13; I'll paste it.
4. **Before running any hook script for the first time** — read each hook aloud (the shell script body) so I can approve what's about to execute on every tool call. Then run the synthetic verification.
5. **After Day-1 completion** — verification summary + list of what's deferred to Days 2–3 and Week 2+.

Between checkpoints, work autonomously. Do not stop to ask permission for routine steps inside a phase.

## 6. Context-compaction / fresh-session recovery

If your context fills up mid-setup and compaction is imminent, or if a fresh session starts:

1. Before compaction (if you can predict it): write a `## Compaction checkpoint YYYY-MM-DD HH:MM` block to `agent-memory/PROGRESS.md` naming the last completed step, the current step, and any in-flight decisions.
2. On rehydration: follow the §8.1 session-start protocol from `KNOWLEDGE_MGMT_SETUP_SPEC.md` — read `PROGRESS.md` last non-COMMIT line, `OPEN-THREADS.md`, `BLOCKERS.md`, the active plan in `agent-memory/plans/`, and `wiki/hot.md` if it exists. Only then resume.
3. If `PROGRESS.md` and the plan disagree, trust `PROGRESS.md` (it's the ground truth of what actually shipped) and reconcile the plan.

This is exactly why §1 of this prompt forced the agent-memory layer to go in first.

## 7. Start now

1. Acknowledge this prompt in one line.
2. Read the three briefs.
3. Summarize the setup plan back to me in ≤10 lines.
4. Wait for my "proceed".
5. On "proceed": seed `agent-memory/` per §1 of this prompt, commit, then begin Phase 1 step 3 of the vault spec.

===== END PROMPT =====

---

## What happens after you paste

**Immediately:** Claude Code will read the three briefs (this takes 1–2 minutes of silence while it loads ~3,000 lines of spec). Then it will come back with a ≤10-line plan summary. Read it. If it looks right, reply `proceed`. If something is wrong or missing, say so — it will adjust before doing any work.

**During Day 1 (~1–2 hours of supervised work):**

- Seeds `agent-memory/` (survival kit) — first commit.
- Creates full folder layout from §2 of the vault spec.
- Interviews you for identity (name, employer, role, startups, freelance, anchors, domains). Takes ~5 min of Q&A.
- Writes `CLAUDE.md`, `AGENTS.md`, `README.md`, `.gitignore`, `.gitattributes`.
- Writes the ten templates in `_templates/`.
- Writes and installs the five hook shell scripts + `.claude/settings.json`. **Will ask you to read each hook before running it.**
- Writes the eight vault-local skills (`/ingest`, `/query`, `/lint`, `/today`, `/crystallize`, `/weekly-review`, `/tech-digest`, `/new-domain`).
- Configures the Obsidian MCP. **Will prompt you for the Local REST API key** — paste the one you saved in your password manager during DAY-0 Part 6.
- Installs the Superpowers plugin (`/plugin marketplace add obra/superpowers-marketplace`).
- Seeds each domain's `_moc.md` and `_schema.md`.
- Drafts the seven seed ADRs from §17 for your review.
- `git init`, first commits, pushes to the private GitHub repo you created in DAY-0 Part 8.
- Verifies hooks fire (synthetic test commit → confirms `wiki/log.md` gets appended).

**Days 2–3:** You drop your first 10 sources (articles, meeting transcripts, voice notes) into `~/second-brain/inbox/`. In Claude Code: `/ingest inbox/`. Watch what it files and how it links. Correct what's wrong — that's how it learns your style.

**Week 2+:** `/weekly-review`, `/crystallize` → `wiki/overview.md`, install `graphify` once you have 20–50 sources.

## When to interrupt Claude Code

- Any step running >30 min with no forward progress — remind it of the §1.6 stall rule.
- It asks a question you don't understand — ask it to explain in plain language; it has the full spec context.
- A command looks destructive (`rm -rf`, `git push --force`, `git reset --hard`) — read the command, ask why. Only proceed if the answer is good.
- It proposes installing a deferred tool on Day 1 (`graphify`, `gitnexus`, custom voice pipelines, RAG-Anything, dedicated memory layers) — push back. Point it at §15 and §20 of `second-brain-spec.md`.
- It starts writing `CLAUDE.md` before interviewing you for identity — interrupt. The identity interview must come first.
- It skips seeding `agent-memory/` and starts on folder layout — interrupt. The knowledge layer goes in first, per §1 of the prompt you pasted. Without it, a compaction mid-setup destroys the session's state.

## If something goes badly wrong

- **Claude Code crashes / session dies mid-setup.** Re-run `claude` in the same directory. Paste as your first message: *"Resume the second-brain setup. Follow the §8.1 session-start protocol from `handover-birefs/KNOWLEDGE_MGMT_SETUP_SPEC.md` — read `agent-memory/PROGRESS.md`, `OPEN-THREADS.md`, `BLOCKERS.md`, the active plan in `agent-memory/plans/`, and `wiki/hot.md` if it exists. Only then resume from the last incomplete step."* The survival kit is exactly what makes this recovery work.
- **A hook script looks wrong.** Don't approve it. Ask Claude Code to re-derive it from §8 of the vault spec and explain the difference.
- **The vault ends up with invented facts about you.** Ask Claude Code to find every `[TBD — populate from user upload]` marker (`grep -rn "TBD — populate" ~/second-brain`) and every unsourced wiki page (`/lint`), then replace invented content with `[TBD]` markers and log them to `OPEN-THREADS.md`. Re-interview on the gaps.
- **You're unsure whether a step is complete.** Ask: *"Verify Phase N step M using the acceptance criteria in §10 of `second-brain-spec.md`. Show me the verification commands and outputs."* Claude Code must produce evidence, not assertion.

---

*Everything between `===== BEGIN PROMPT =====` and `===== END PROMPT =====` is the full handoff. This file is the only thing you need to paste. The three briefs in `handover-birefs/` are what the agent reads — you do not need to paste them.*
