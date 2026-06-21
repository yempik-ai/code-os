# Second Brain — Grounded Implementation Spec

> [!note] Template spec — examples are illustrative
> The worked examples below (domain names, entities, ADR figures) are neutral
> placeholders. Replace them with your own situation when you build your vault.

**Version** 2.3 · **Date** 2026-04-23 · **Status** Implementation-ready · **Source PDF** `second-brain-spec.pdf` (v2.2, 2026-04-19)

**Purpose.** This spec defines a personal knowledge-management system that survives multi-month growth, context compaction, cross-device sync, and context switches across multiple life domains (e.g. employer / startup / freelance / tech-news) — while giving Claude (via Claude Code and claude.ai) enough structure to act as a compounding, maintaining librarian rather than a forgetful search tool.

**Audience.** Two readers:

1. **The operator** (the human whose vault this is).
2. **Any Claude instance** you point at the vault. A Claude Code agent reading this should be able to execute §A "Human prerequisites" → §10 "Setup checklist" top-to-bottom without asking clarifying questions.

This version (2.3) is also written so that a **non-technical operator** (e.g. a CEO handing the spec to their own Claude Code on a fresh Mac) can get to a working system. The human-only prerequisites — the things Claude Code cannot click through from a shell — are collected in the new §A so they stand out.

**Non-negotiable outcome.** After applying this spec: (a) deterministic session re-hydration after compaction via `CLAUDE.md` + hot cache; (b) immutable-raw / LLM-owned-wiki / fixed-schema three-layer architecture per Karpathy; (c) commit-per-ingestion discipline with auto-logged provenance; (d) hook-enforced safety around vault writes; (e) `/brainstorm` → `/write-plan` → `/execute-plan` skill pack for multi-session projects; (f) cross-device sync without conflict files; (g) every domain (e.g. employer / startup / freelance / tech-news) has a first-class home with MoC, schema, templates; (h) an append-only operation log that lets any future you reconstruct what happened.

**What changed vs v1.0.** Adds: hooks-as-enforcement, formal file formats for every memory file, survival-kit discipline from long-running agentic work, contradiction-at-ingest pattern, hot-cache for compaction recovery, L1/L2 credential split, superpowers integration, setup-as-executable-plan, explicit non-goals, and honest accounting of what breaks at scale. Removes: nothing — v1.0 remains correct; v2.0 is additive.

**What changed in v2.1.** Refactored §9.1 to reflect the Day-1 voice decision (Obsidian plugin + MacWhisper free only) and moved the automated iPhone → `inbox/voice/` pipeline to §20.1 as a deferred custom enhancement to build later with full Apple-ecosystem leverage. Added §20 "Deferred enhancements" as an explicit home for everything that's correct-to-build-later rather than never.

**What changed in v2.2.** Explicit Day-1 install scope in §15: ONLY `uvx mcp-obsidian`, Superpowers plugin, and vault-local skill markdown files. Everything else (Graphify, RAG-Anything, CLI-Anything, GitNexus, memory layers, additional MCPs) is deferred or doesn't apply. Fixed §10 Phase 6 step 30 which incorrectly told Claude Code to install Graphify at end of setup (Graphify has zero value on an empty vault). Clarified §2 that the "AGENTS.md GitNexus-style block" is terminology only — GitNexus itself is a code-repo tool and doesn't apply to a markdown vault. §20.2 rewritten with concrete Graphify install and first-run commands for when you do come back to it.

**What changed in v2.3.** Added §A "Human prerequisites" — the non-delegable human steps, collected up front so a non-technical operator hands off to Claude Code cleanly. The companion one-pager `DAY-0-CHECKLIST.md` is the copy-paste-ready artifact for a handoff; §A here is the narrative backup. Rewrote §11 "Sync" to recommend **Obsidian Sync via an Obsidian+ (Obsidian Plus) subscription** as the default for non-technical operators: it collapses the entire cross-device story — desktop, laptop, iPhone, iPad — into toggling one setting inside Obsidian. Syncthing remains the power-user / privacy-max option. Clarified in §A and §10 that these sync steps are human-only (billing + device login) and cannot be done by Claude Code. Claude Code is trusted to write the hook scripts (§8) and vault-local skills (§7.2) from their behavioral specifications — source code is deliberately not pasted into this spec. **Removed MacWhisper as a Day-1 install** — Claude Code now ships with a built-in voice mode that covers desk dictation natively, and external meeting audio is already transcribed by the meeting tool itself (§9.3). A transcription helper gets installed ad-hoc by Claude Code only when a specific recording genuinely needs it. §9.1 and ADR-0008 rewritten accordingly. Marked the whole document as a CEO-readable handoff: prose is self-contained, jargon is expanded on first use.

---

## A. Human prerequisites — steps Claude Code cannot do for you

Claude Code can write every file, run every shell command, wire every config, and build every hook and skill in this spec from its behavioral description. It **cannot** click through GUI installers, accept macOS permission prompts, log into your Apple ID, or enter a credit card. Those steps are human-only and must be done **before** Claude Code takes over.

**The copy-paste-ready handoff artefact is `DAY-0-CHECKLIST.md`** (companion to this spec). Hand that checklist to the operator alongside this spec and `KNOWLEDGE_MGMT_SETUP_SPEC.md`; it walks through the ~45 min of human work and ends with the literal prompt to paste into Claude Code. This section is the narrative summary — the checklist is the authoritative list.

Budget: **~45 minutes** of human clicking, then **~2-3 hours** supervising Claude Code on Day 1.

### A.1 Accounts (5 min)

Apple ID (already signed in), free GitHub account, Anthropic account (claude.ai, Pro plan is fine), free Obsidian account.

### A.2 Install the base toolchain (15 min)

Homebrew, Node.js (`brew install node`), Python + uv (`brew install python && curl -LsSf https://astral.sh/uv/install.sh | sh` — needed for `uvx mcp-obsidian` and deferred tools like `graphify`), GitHub CLI (`brew install gh && gh auth login`), Claude Code (`npm install -g @anthropic-ai/claude-code`), first run of `claude` for Anthropic OAuth. **Voice capture is already handled** by Claude Code's built-in voice mode — no separate transcription app is needed on Day 1 (see §9.1).

### A.3 Install Obsidian + create the vault (5 min)

Download Obsidian from obsidian.md, create a new vault at `~/second-brain` (exact path matters — the spec references it everywhere).

### A.4 macOS permissions (2 min)

Grant **Full Disk Access** to Terminal.app (or iTerm2) in System Settings → Privacy & Security. Allow Obsidian Files & Folders access when prompted.

### A.5 Install Obsidian community plugins (10 min)

Obsidian won't let an external program install plugins — this is GUI-only. In Obsidian: **Settings → Community plugins → turn on → Browse**, then install *and enable* each of:

**Required for vault operation:** Local REST API, Templater, Dataview, Obsidian Git, Periodic Notes, QuickAdd, Omnisearch, Advanced URI, Importer.

**Recommended UX:** Calendar, Excalidraw, Kanban, MarkMind, Iconize (Icon Folder), Advanced Tables.

**Optional:** Style Settings.

Then: **Settings → Local REST API → copy the generated API key to your password manager** (Claude Code will ask for it). Leave the default port 27124 HTTPS. Keep Obsidian running in the background.

Full list with install notes lives in the companion `DAY-0-CHECKLIST.md`.

### A.6 Browser helpers (3 min)

Install the **Obsidian Web Clipper** browser extension for Chrome/Safari/Firefox.

### A.7 GitHub remote (3 min)

Create a **private** empty repo on github.com (no README). Keep the URL handy.

### A.8 Cross-device sync (10 min)

**Recommended: Obsidian Sync via Obsidian+** ([obsidian.md/plus](https://obsidian.md/plus), ~$10/mo). Subscribe, enable Core plugin → Sync, create a new remote vault, save the encryption password in a password manager, install Obsidian mobile on iPhone/iPad, log in. That's the entire cross-device story.

**Power-user alternative: Syncthing + Möbius Sync.** Free, peer-to-peer, fully private, but requires installing Syncthing on every device and a separate paid iOS app because iOS doesn't allow Syncthing natively. Full comparison in §11.

**What Claude Code cannot do here:** enter your credit card, accept the Obsidian+ terms, or type your Apple ID password on your phone. Once Sync is toggled on, Claude Code never has to think about sync again.

### A.9 Hand off to Claude Code

Put this spec, `KNOWLEDGE_MGMT_SETUP_SPEC.md`, and `DAY-0-CHECKLIST.md` into the vault at `~/second-brain/handover-birefs/`. Then `cd ~/second-brain`, run `claude`, and paste the handoff prompt from Part 10 of `DAY-0-CHECKLIST.md`. That prompt tells Claude Code to read all three files, execute §10 (starting from step 3 since community plugins are already installed), write hooks and skills from their specifications (not copy them from source), and interview you for identity and domains before committing `CLAUDE.md`. Paste the Obsidian Local REST API key (from A.5) when Claude Code asks for it.

When A.1–A.8 are done, Claude Code handles everything downstream. You supervise, approve tool calls, and answer identity/domain/ADR questions as they come. **L1 credential-vault setup, voice pipelines, graphify, gitnexus are all deferred** — Claude Code will bring them up when the vault needs them, not on Day 1.

---

## 0. System overview — five cooperating layers

The system has **five layers**. Each layer has one job. Do not collapse them.

| # | Layer | Role | Lives in | Failure mode if missing |
|---|---|---|---|---|
| 1 | **Capture** | Zero-friction ingestion from voice, text, web, files, transcripts. | Capture plugins + `inbox/`, `raw/` | You stop capturing → the whole system dies. |
| 2 | **North-star (`CLAUDE.md`)** | In-context operating rules reloaded every turn. | Vault root | Claude drifts; every session re-explains the domain. |
| 3 | **Storage (Obsidian vault)** | Single source of truth. Raw immutable / wiki LLM-owned / agent memory. | Markdown files | Knowledge fragments across tools; links die. |
| 4 | **Maintenance (skills + hooks)** | The LLM-as-librarian that compiles, cross-references, lints. Hooks enforce the safety floor. | `.claude/skills/`, `.claude/hooks/`, `.claude/settings.json` | Wiki rots within 2 weeks. |
| 5 | **Retrieval & reasoning** | Claude Code + Obsidian MCP + optional Graphify / RAG-Anything / memory layer. | MCP plane + skills | You have a filing cabinet, not a thinking partner. |

**One principle ties them together:** *the vault is a compiled artifact, not a document dump.* Raw stays raw. The wiki is Claude's output. The schema is the contract. When the three are honored, knowledge compounds. When one slips, the whole thing becomes a graveyard of orphans.

---

## 1. The six non-negotiables (the operator + Claude both live by these)

These override everything downstream. If any tactical rule later in this doc conflicts with one of these, the non-negotiable wins.

1. **Raw is immutable.** Files under `raw/` are never edited after capture, ever. Only appended to (new files) or moved to `raw/_archive/` if proven wrong. No exceptions.
2. **The wiki is Claude territory.** Under `wiki/`, you (the human) *read*. Claude writes. You intervene only to correct a factual error or overrule a structural decision. Heavy manual editing in `wiki/` is a smell — it means the schema is wrong; fix the schema.
3. **Every note has provenance.** Every `wiki/` page carries a `sources:` frontmatter list pointing at the `raw/` files that justified it. A claim with no source is a claim to be deleted or reclassified as `status: hypothesis`.
4. **Contradictions are flagged, never overwritten.** When a new source conflicts with existing wiki content, Claude adds a `> [!contradiction]` callout keeping both claims with their sources. You resolve it — or leave it pending. Silent overwrite is a critical discipline failure.
5. **Commit-per-ingestion.** Every `/ingest` run produces one git commit with a message following a strict format (§8.4). The hook auto-logs the commit to `log.md`. If you find yourself without a commit for >24h of active capture, you're stuck — stub it and move on.
6. **30-minute stall rule.** If Claude (or you) is stuck on a single wiki page or ingestion for >30 minutes with no forward progress, stub the problem, add a `> [!blocker]` callout, log it to `agent-memory/BLOCKERS.md`, continue. Perfectionism is the enemy of compounding.

---

## 2. Target directory layout — everything in its place

```
~/second-brain/                                       # the one vault
├── README.md                                         # human-facing: what is this
├── CLAUDE.md                                         # the north-star (§3)
├── AGENTS.md                                         # identical block for non-Claude agents
├── .gitignore                                        # see §14.1
├── .gitattributes                                    # *.md diff=markdown; *.canvas linguist-language
│
├── .claude/                                          # Claude Code runtime
│   ├── settings.json                                 # committed hooks config
│   ├── settings.local.json                           # per-machine overrides — gitignored
│   ├── hooks/
│   │   ├── block_dangerous_writes.sh                 # PreToolUse: protect raw/ and sensitive paths
│   │   ├── log_ingestion_to_log_md.sh                # PostToolUse(Bash): append git commits to log.md
│   │   ├── hot_cache_on_stop.sh                      # Stop: refresh wiki/hot.md
│   │   ├── session_start_context.sh                  # SessionStart: inject hot.md + open threads
│   │   └── precompact_save.sh                        # PreCompact: harvest session learnings
│   └── skills/                                       # vault-local skills (§7)
│       ├── ingest/SKILL.md
│       ├── query/SKILL.md
│       ├── lint/SKILL.md
│       ├── crystallize/SKILL.md
│       ├── today/SKILL.md
│       ├── weekly-review/SKILL.md
│       ├── tech-digest/SKILL.md
│       └── new-domain/SKILL.md
│
├── _templates/                                       # Templater templates (Obsidian)
│   ├── daily.md
│   ├── meeting.md
│   ├── person.md
│   ├── client.md
│   ├── project.md
│   ├── wiki-concept.md
│   ├── wiki-entity.md
│   ├── wiki-source.md
│   ├── decision.md                                   # personal ADR
│   └── blocker.md
│
├── raw/                                              # IMMUTABLE — source layer
│   ├── articles/YYYY-MM/YYYY-MM-DD-<slug>.md
│   ├── papers/YYYY-MM/YYYY-MM-DD-<slug>.pdf
│   ├── clips/YYYY-MM/<slug>.md                       # Obsidian Web Clipper
│   ├── transcripts/YYYY-MM-DD-<slug>.md              # meeting transcripts
│   ├── voice/YYYY-MM-DD-HHMMSS.md                    # Whisper output
│   ├── messages/YYYY-MM/                             # imported threads (email/slack/whatsapp)
│   └── _archive/                                     # things proven wrong/superseded
│
├── inbox/                                            # unprocessed capture; Claude drains this
│
├── wiki/                                             # LLM TERRITORY — the compiled artifact
│   ├── overview.md                                   # top-of-tree synthesis
│   ├── index.md                                      # LLM-maintained catalog
│   ├── log.md                                        # append-only ingestion log (hook-written)
│   ├── hot.md                                        # recent-context summary (hook-written)
│   ├── concepts/                                     # generic concept pages
│   ├── entities/                                     # people, companies, products
│   ├── sources/                                      # one-per-source summary + claims
│   ├── syntheses/                                    # cross-source syntheses
│   ├── questions/                                    # open questions (LLM notes to self)
│   └── contradictions.md                             # index of unresolved [!contradiction]
│
├── domains/                                          # your life's compartments
│   ├── <employer>/
│   │   ├── _moc.md                                   # map of content — the home
│   │   ├── _schema.md                                # domain-specific rules for Claude
│   │   ├── accounts/<account>/
│   │   ├── deliverables/
│   │   ├── meetings/YYYY/YYYY-MM-DD-<slug>.md
│   │   ├── decisions/                                # ADRs for this domain
│   │   └── <key-workstream>/                         # your specific role deliverable
│   ├── startup-<codename>/
│   │   ├── _moc.md
│   │   ├── _schema.md
│   │   ├── strategy/
│   │   ├── product/
│   │   ├── cofounder/                                # shared context with co-founder
│   │   ├── meetings/
│   │   └── decisions/
│   ├── freelance/
│   │   ├── _moc.md
│   │   ├── _schema.md
│   │   ├── leads/<company>/
│   │   ├── key-accounts/<company>/
│   │   ├── stakeholders/                             # cross-account humans
│   │   ├── proposals/
│   │   └── invoices/                                 # billing / revenue tracking
│   ├── tech-news/
│   │   ├── _moc.md
│   │   ├── _schema.md
│   │   ├── weekly/YYYY-WW.md                         # weekly digest
│   │   └── trends/                                   # emerging topic pages
│   └── personal/
│       ├── _moc.md
│       ├── health/
│       ├── learning/
│       └── (private — see L1/L2 split in §6)
│
├── people/                                           # one note per human; linked from everywhere
│   └── <firstname-lastname>.md
│
├── projects/                                         # domain-spanning active work
│   └── <project-slug>/
│       └── <project>.md
│
├── daily/                                            # chronological spine
│   └── YYYY/YYYY-MM-DD.md
│
├── agent-memory/                                     # survival kit (§4)
│   ├── PROGRESS.md                                   # append-only, what's been done
│   ├── DECISIONS.md                                  # life/strategic ADRs
│   ├── BLOCKERS.md                                   # append-only, stalls + resolutions
│   ├── OPEN-THREADS.md                               # unresolved open questions/threads
│   ├── PREFERENCES.md                                # how Claude should talk to me
│   └── TIMELINE.md                                   # macro events: job changes, big life moments
│
├── graphify/                                         # generated — per-corpus knowledge graph
│   └── <corpus>/
│       ├── graph.json
│       ├── report.html
│       └── notes/
│
├── canvases/                                         # Obsidian Canvas files for visual thinking
│   └── <name>.canvas
│
└── attachments/                                      # images, diagrams (Obsidian-linked)

~/.second-brain-local/                                # L1 — NEVER committed, NEVER synced via cloud
├── secrets.md                                        # API keys, access codes, financial
├── health-private.md                                 # anything subject to health confidentiality
└── machine-state.md                                  # ephemeral — per-device session pointers
```

**Why one vault, not one-per-domain.** Notes compound only if they can cross-link. Your day-job work and your startup's tech stack share 60% of the concept graph. Split them and you lose the compounding. `domains/` provides namespacing without fragmentation.

**Why `domains/` sibling to `wiki/`, not under it.** `wiki/` is Claude's synthesized conceptual model (atemporal, entity- and concept-oriented). `domains/` is time- and event-oriented (meetings, deliverables, accounts). They reference each other constantly — but they're different kinds of artifact.

---

## 3. `CLAUDE.md` — the north-star (template)

`CLAUDE.md` is loaded on every turn in Claude Code. In claude.ai via the Obsidian MCP, it's pulled on first reference and should be re-read whenever domain rules might apply. Keep it ≤600 lines. Rewrite it when the domains change (new employer, new startup, new service line).

The example values below are illustrative placeholders — replace them with your own situation.

**Required sections** (fill in from your actual situation; this is the structure, not the content):

```markdown
# <Your name> — Second Brain Operating Schema

> This file is the north-star briefing. Read top to bottom before any non-trivial
> action. Reload when entering a new session. If anything below conflicts with
> something you read later, this file wins.

## Who / What
Who you are, what you do, why this vault exists. Three paragraphs max.
- Primary role: <current job + scope>
- Side ventures: <startups + co-founder>
- Freelance: <status, target sectors, <local tax regime>>
- Partner/personal anchors: <city plans, major life variables>

## Domains (the buckets this vault serves)
1. **<employer>** — employer. Role: <X>. Key people: [[person-a]], [[person-b]].
   Home: [[domains/<employer>/_moc]].
2. **startup-<codename>** — co-founder: [[cofounder-name]]. Home: [[...]].
3. **freelance** — leads/key-accounts/stakeholders under `domains/freelance/`.
4. **tech-news** — weekly digest, emerging trends. Home: [[...]].
5. **personal** — health, learning, relationships. Home: [[...]].

## Read this first (authoritative pointers)
- `agent-memory/PREFERENCES.md` — how I want Claude to talk to me
- `agent-memory/OPEN-THREADS.md` — what's unresolved right now
- `agent-memory/PROGRESS.md` — what's been done in the vault
- `wiki/hot.md` — last session's recent context (ALWAYS read first in a new session)
- `wiki/index.md` — catalog of wiki pages
- `wiki/overview.md` — current state of the compiled understanding
- `domains/<relevant>/_schema.md` — domain-specific rules for whatever domain this session is about

## Non-negotiables (see spec §1 for full rationale)
1. Raw is immutable.
2. Wiki is Claude territory. Contradictions get [!contradiction] callouts, never overwrites.
3. Every wiki page has `sources:` frontmatter.
4. Commit per ingestion with message convention below.
5. 30-minute stall → stub + [!blocker] + BLOCKERS.md entry + continue.
6. Personal health, finances, and credentials live ONLY in ~/.second-brain-local/ (L1).
   Never in the git-tracked vault.

## Frontmatter contract (every note carries this)
---
type: [daily | meeting | person | client | project | wiki-concept | wiki-entity |
       wiki-source | synthesis | raw | decision | blocker]
domain: [<employer> | startup-X | freelance | tech-news | personal | global]
tags: [kebab-case-only]
created: ISO-8601
updated: ISO-8601
status: [active | hypothesis | archived | contradicted | superseded-by]
sources: [list of raw/ paths — required for wiki/* notes]
people: [[[person-a]], [[person-b]]]
---

## Ingestion rules (how Claude processes `raw/` and `inbox/`)
When a new source lands in raw/ (or inbox/):
1. Read it fully. Identify the type: article, transcript, voice note, email,
   clip, conversation.
2. Find 3-10 existing wiki pages it touches. If <3, one of: (a) it's a new
   concept/entity — create one page; (b) it's context-free noise — log to
   log.md and stop.
3. For each touched page: append/update with source-attributed claims. Backlinks
   both directions.
4. For new concepts/entities: create page using the right template, cross-link
   to at least 2 existing wiki pages, add to index.md.
5. If a meeting transcript: file under `domains/<domain>/meetings/YYYY/`,
   extract decisions → `domains/<domain>/decisions/`, action items → the
   attendees' `people/` pages, and today's `daily/` note.
6. If a voice note / inbox entry: extract TODOs, file to the right
   project/domain, update the daily note.
7. If contradiction with existing wiki content: add [!contradiction] callout
   on the relevant page with both sources. Add to wiki/contradictions.md.
8. Move source from inbox/ to raw/<category>/YYYY-MM/. Never delete from raw/.
9. Append to wiki/log.md: `## [YYYY-MM-DD HH:MM] ingest | <title> → <pages>`
10. Commit with message: `ingest(<domain>): <title> — N pages (+M new)`.

## Query rules (how Claude answers questions about the vault)
1. Read wiki/hot.md first (recent context, free).
2. Read wiki/index.md to pick candidate pages.
3. Pull 3-5 relevant wiki pages. Follow [[wikilinks]] one hop.
4. If the question is about a specific domain: also read that domain's _moc.md
   and _schema.md.
5. If the question is temporal ("what did we discuss about X last week"):
   also read the relevant daily/ notes in that window.
6. Answer with citations: "[[wiki/concepts/auth-patterns]] claims X because
   [[raw/articles/2026-03/auth-survey]]."
7. If the answer exposes a gap (an open question, a missing source), append
   to wiki/questions/ or agent-memory/OPEN-THREADS.md.

## Lint rules (what /lint checks)
- Orphans: wiki pages with <2 incoming backlinks.
- Dead links: [[wikilinks]] to nonexistent pages.
- Stale: pages with `updated:` older than 180 days and no source added since.
- Sourceless: wiki/* pages without `sources:` frontmatter.
- Contradiction debt: unresolved [!contradiction] callouts older than 30 days.
- Schema drift: frontmatter fields that don't match the contract above.
- Unprocessed: files in inbox/ older than 7 days.

## Commit message convention
`<type>(<scope>): <subject>`
types: ingest | crystallize | lint | refactor | docs | chore | fix
scope: domain name, or `wiki`, or `meta`
Examples:
- ingest(<employer>): Q1 board prep transcript — 4 pages (+1 new)
- crystallize(startup-<codename>): pricing strategy discussion — overview.md + 2 entities
- lint(wiki): resolved 12 orphans, 3 dead links

## Operating rules — what NOT to do
- NEVER edit raw/ after capture.
- NEVER silently overwrite a wiki page that contradicts a new source. Use the
  [!contradiction] callout.
- NEVER store secrets, API keys, health records, or financial account details
  in the git-tracked vault. Those go to ~/.second-brain-local/ (§6).
- NEVER create a wiki page without at least one source (except hypothesis pages,
  which must have status: hypothesis).
- NEVER skip writing to log.md after an ingestion.
- NEVER invent a skill name Claude doesn't have. Only use skills listed in
  .claude/skills/ or installed via /plugin.
- NEVER run /lint --autofix without committing first.
```

---

## 4. `agent-memory/` — the survival kit

Every file here has a **fixed format**. Format drift breaks the hooks and recovery protocols.

### 4.1 `PROGRESS.md` — append-only, macro-level

**Purpose:** the one file future-you reads to know where the vault is.

```markdown
# PROGRESS

Append-only log of vault milestones (not individual ingestions — those are in
wiki/log.md). Format: `[YYYY-MM-DD] <tag> <message>` where tag ∈
{MILESTONE, START, DONE, STUB, PIVOT, NOTE}.

---
- [2026-04-19] START Vault initialized. Karpathy LLM Wiki pattern. 6 domains.
- [2026-04-26] MILESTONE All four domains have an MoC and 10+ wiki pages each.
- [2026-05-03] NOTE Switched from Obsidian Sync to Syncthing — fewer conflicts.
- [2026-05-10] MILESTONE 100 sources ingested. First lint pass: 17 orphans resolved.
```

Rules: only you write this, one or two lines per week. Detailed operation history lives in `wiki/log.md` (auto-written).

### 4.2 `DECISIONS.md` — personal ADRs

Format (separator `---`):

```markdown
## ADR-NNNN — <one-line title>
**Status:** Accepted · **Date:** YYYY-MM-DD · **Domain:** <domain>
**Context:** Why this decision needed to happen. The forces in play.
**Decision:** What you chose.
**Consequences:** Tradeoffs. What this closes off. What it opens up.
**Supersedes:** ADR-NNNN (if this replaces a prior decision)
---
```

**What belongs here:** choosing Obsidian over Notion; picking Syncthing over Obsidian Sync; deciding which business/operating structure to use; a major focus or positioning choice. These are the load-bearing choices that future-you will want to understand.

**What doesn't:** preferences (those go in `PREFERENCES.md`), tactical choices inside a domain (those go in `domains/<domain>/decisions/`), ephemeral session state.

### 4.3 `BLOCKERS.md` — append-only, with resolutions

Format:

```markdown
## YYYY-MM-DD — <domain>: <one-line what was blocked>
**What:** Concrete description.
**Found:** What investigation revealed.
**Tried:** Approaches attempted.
**Stub:** The workaround in place, with a `TODO(blocker: YYYY-MM-DD)` reference
  in the relevant wiki page.
**Follow-up:** When and how this should be revisited.
---
### Resolution (YYYY-MM-DD)
Appended when the blocker is resolved. What unblocked it. Where the stub was
removed. Never rewrite the original entry — append below.
---
```

Used by both you and Claude. When Claude hits the 30-minute wall (§1.6), it writes an entry here automatically.

### 4.4 `OPEN-THREADS.md` — the "what's unresolved" file

Format: grouped by domain, each thread is a heading with 3-5 bullets.

```markdown
# OPEN THREADS

## <employer>
### AI Adoption Framework — rollout phase 2 criteria
- Need quantitative success metrics for phase 1 (ongoing)
- Decision pending: who gets Claude Code access in phase 2
- Blocked on: CIO sign-off for budget

## startup-<codename>
### Pricing tiers
- Waiting on co-founder's market research (due Apr 22)
- Decision needed: self-serve free tier yes/no

## freelance
### <prospect> contact
- Co-founder reaches out first. Follow-up cadence TBD.
```

Claude reads this at session start. You update it during weekly review. It's the "what's on my plate" view — richer than a todo list, lighter than full project notes.

### 4.5 `PREFERENCES.md` — how Claude should talk to me

This is the file that makes every Claude session feel like one continuous conversation. Format is free-form but stable:

```markdown
# How to work with me

## Language
- Default <your primary language>. Switch to English when the source is English
  or the topic is international tech.

## Tone
- Direct. Skip hedging. Skip apologies. Honest pushback on bad ideas is welcome.
- No emojis in serious work. OK in casual chat.

## Formatting
- Prose over bullets for analysis.
- Bullets OK for checklists, structured options, comparisons.
- No giant headers in short responses.

## Length calibration
- One-paragraph questions get one-paragraph answers.
- Strategic questions get the full treatment.

## Never do
- Never tell me "I'm not a lawyer" when I already know. Just answer.
- Never ask me to reformulate a question — infer.
- Never invent people, companies, or events. If you don't know, say so.
```

### 4.6 `TIMELINE.md` — macro events

Append-only. Single line per event. Scope: life-level things that shape domain context.

```markdown
- 2024-11 — Started first freelance engagement (project A, <amount>)
- 2025-04 — Second freelance engagement closes (project B, <amount>)
- 2025-10 — Signed <role> at <employer> (1yr contract)
- 2026-02 — Co-founder conversations with <prospect> begin
- ...
```

Claude reads this when a question requires historical context ("why did you…?").

---

## 5. `wiki/` — the compiled artifact (Karpathy LLM Wiki, production-grade)

This is the heart of the system. Everything else is plumbing.

### 5.1 Page types and when to use which

| Type | Folder | Purpose | Example |
|---|---|---|---|
| **concept** | `wiki/concepts/` | A generic idea, technique, pattern | `retrieval-augmented-generation.md`, `event-sourcing.md` |
| **entity** | `wiki/entities/` | A specific real-world thing | `anthropic.md`, `<prospect-company>.md`, `claude-code.md` |
| **source** | `wiki/sources/` | One-per-source summary + extracted claims | `2026-03-karpathy-llm-wiki-gist.md` |
| **synthesis** | `wiki/syntheses/` | Cross-source reasoning; an argument | `why-managed-postgres-over-self-hosted.md` |
| **question** | `wiki/questions/` | An open question the vault is investigating | `is-rag-anything-worth-the-complexity.md` |

The `sources/` folder is **new vs v1.0 and critical at scale**. When 200 sources converge on one concept, you need a place where each source is summarized independently — otherwise the concept page becomes an unreadable wall of text. Practice from `vanillaflava/llm-wiki-claude-skills` and `AgriciDaniel/claude-obsidian` confirms this.

### 5.2 Page template — `wiki/concepts/<slug>.md`

```markdown
---
type: wiki-concept
domain: [global | <employer> | ...]
tags: [tag1, tag2]
created: 2026-04-19
updated: 2026-04-19
status: active
sources: [[[sources/2026-03-karpathy-llm-wiki-gist]], [[sources/2026-04-...]]]
---

# <Concept name>

> One-sentence definition. What this thing *is*, not what it does.

## Core claim
2-4 sentences synthesizing the current best understanding. This is Claude's
synthesized position across all cited sources.

## Why it matters
Why this concept is in the vault at all. What it connects to in your life/work.

## Key aspects
- **<Aspect 1>** — short exposition, with [[wikilinks]] to related concepts.
- **<Aspect 2>** — ...

## Open questions
- What's still unresolved about this concept?
- Link to [[questions/...]] pages where applicable.

## Related
- [[concepts/...]] — how this relates
- [[entities/...]] — who's doing this

## Provenance
- [[sources/YYYY-MM-<slug>]] — one-line on what this source contributed.
- [[sources/...]] — ...

> [!contradiction] (only if applicable)
> Source A says X. Source B says Y. Unresolved.
```

### 5.3 Page template — `wiki/sources/<slug>.md`

```markdown
---
type: wiki-source
source_path: raw/articles/2026-03/karpathy-llm-wiki.md
source_url: https://gist.github.com/karpathy/...
author: Andrej Karpathy
published: 2026-04-03
ingested: 2026-04-19
tags: [llm-wiki, knowledge-management, rag]
---

# <Source title>

> One-sentence summary of the source.

## What the source argues
2-4 sentences on the source's main claim.

## Key extracted claims
- **Claim 1.** The claim in Claude's words → backlink: [[concepts/<concept>]]
- **Claim 2.** ...

## Novel to my vault
What this source added that existing wiki pages didn't have.

## Contradicts
- [[concepts/<other>]] — describe the tension. (If applicable.)

## Related
- [[concepts/...]]
- [[entities/...]]
```

### 5.4 The three meta-files

**`wiki/overview.md`** — Claude's top-of-tree synthesis. What's in the vault right now, at a 30,000-foot view. Rewritten (not appended) by `/crystallize` at major milestones. ≤500 lines.

**`wiki/index.md`** — flat catalog of every wiki page with a one-line description. Format:

```markdown
# Wiki Index

## Concepts
- [[concepts/llm-wiki-pattern]] — compiling sources into a persistent wiki
- [[concepts/event-sourcing]] — append-only state as a sequence of events
...

## Entities
- [[entities/anthropic]] — AI company behind Claude
- [[entities/<employer>]] — my employer
...

## Sources (recent 20)
- [[sources/2026-04-karpathy-llm-wiki]]
...
```

Claude maintains it on every ingestion. Source list trims to most recent 20 — full list lives in `wiki/log.md`.

**`wiki/log.md`** — append-only, hook-written, one line per ingestion:

```markdown
## [2026-04-19 11:30] ingest | Karpathy LLM Wiki gist
Source: raw/articles/2026-04/karpathy-llm-wiki.md
Pages created: wiki/concepts/llm-wiki-pattern.md, wiki/sources/2026-04-karpathy-llm-wiki.md
Pages updated: wiki/overview.md, wiki/index.md
Commit: 7f41754

## [2026-04-19 14:12] ingest | Board meeting transcript Apr 17
Source: raw/transcripts/2026-04-17-board.md
Pages created: domains/<employer>/meetings/2026/2026-04-17-board.md
Pages updated: domains/<employer>/_moc.md, people/ceo-name.md, domains/<employer>/decisions/...
Commit: a12c9f0
```

### 5.5 The hot cache — `wiki/hot.md`

Life-changing pattern from `AgriciDaniel/claude-obsidian`. Refreshed at every session end, read at every session start. ≤200 lines.

**Contents** (auto-written by the Stop hook):

- **Active domains this session:** which domains were touched.
- **Pages updated:** last 10 wiki pages edited.
- **Open threads surfaced:** any new questions or blockers raised.
- **Current state:** where you stopped — mid-ingestion, mid-synthesis, mid-review.

The next session reads this **first** before anything else. It kills the "rebuild context from scratch" tax that otherwise destroys compaction recovery.

### 5.6 Contradiction handling

When ingesting a new source, if it conflicts with existing wiki content:

**Do:**

```markdown
> [!contradiction] Source A vs Source B (2026-04-19)
> **[[sources/2026-03-vendor-claim]]** says Claude Sonnet 4 outperforms Opus 4 on task X.
> **[[sources/2026-04-independent-eval]]** shows the opposite on the same benchmark.
> Status: unresolved. Needs investigation.
```

**Don't:** silently overwrite the claim. **Don't:** pick one and bury the other. **Don't:** delete the contradiction — only mark it resolved with a resolution note.

`wiki/contradictions.md` is a meta-page that Claude maintains listing every unresolved `[!contradiction]` callout across the vault. This is your "gaps in the model" view.

---

## 6. L1 / L2 cache split — credentials & privacy

Pattern stolen from `MehmetGoekce/llm-wiki`. A single vault can't safely hold everything — some data must never be synced, committed, or read by Claude without an explicit ask.

**L2 (default)** — the git-tracked, synced vault at `~/second-brain/`. Safe for:

- Work notes, meeting transcripts with colleagues
- Concept/entity synthesis, research
- Freelance leads, proposals (names/contexts OK; contract values OK; account credentials NOT)
- Daily notes, project plans
- Tech news, learning

**L1 (sensitive)** — separate folder at `~/.second-brain-local/`, **not** git-tracked, **not** synced via any cloud provider, **not** in any Obsidian vault you open with MCP by default. Contents:

- API keys, access tokens, passwords (normally would go in a password manager — L1 is for stuff that's context-adjacent, not replacement)
- Financial account numbers, tax codes, personal bank details
- Health information (medical records, diagnoses, medications)
- Highly private relationship or personal content
- Machine-specific session pointers (`CLAUDE.local.md` additions)

**The rule:** *Would the LLM making a mistake here be dangerous or embarrassing?* → L1. *Merely inconvenient?* → L2.

**Implementation:**

- `~/second-brain/.gitignore` includes a wildcard that blocks `.local/` inside the vault if you want an alternative placement.
- L1 is opened as a **separate Obsidian vault** only when needed — not via MCP by default.
- Obsidian supports multiple vaults; switch manually when you need L1.
- Sync L1 via 1Password, Apple Notes with encryption, or a dedicated encrypted volume. Not iCloud Drive. Not Dropbox. Not the same Syncthing pool or Obsidian Sync vault as L2.

---

## 7. Skills & MCPs — the retrieval and maintenance layer

### 7.1 MCP servers (Claude Code harness-level config)

| MCP | Purpose | When to use | Config |
|---|---|---|---|
| **obsidian** | Read/write vault via Local REST API plugin (port 27124 HTTPS). | Everything vault-related when Obsidian can be running. | `uvx mcp-obsidian` with `OBSIDIAN_API_KEY` |
| **obsidian-fallback** | Direct filesystem access when Obsidian is closed. | Background jobs, automation, Obsidian-off sessions. | `npx obsidian-mcp <vault-path>` |
| **context7** | Live library docs. | Any question involving a library, framework, SDK, CLI. | Harness marketplace install |
| **filesystem** | Raw FS access for `raw/` drops. | Capture pipelines. | Default MCP server |
| **github** *(optional)* | Manage the vault repo remotely. | Cross-device commits, backups. | OAuth |
| **fetch** | Pull URLs on demand. | Web-clipping, citation resolution. | Default MCP server |

**Don't install the web-search MCP** if you already use Claude with search enabled on claude.ai — pick one surface.

For **Obsidian-specific writes via MCP**, the `mcp-obsidian-advanced` fork adds NetworkX graph introspection which is worth it once the vault crosses 200+ pages. At that scale, Obsidian's native Dataview becomes slow and Claude's grep-style search hits context limits.

### 7.2 Skills to install

**Superpowers plugin** (`obra/superpowers`) — do this first. Gives you `/brainstorm`, `/write-plan`, `/execute-plan` plus TDD, debugging, verification skills. Install via:

```
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
```

**Vault-local skills** (in `.claude/skills/` of the vault, committed to git):

| Skill | Trigger | Purpose |
|---|---|---|
| **ingest** | `/ingest [path]` | Process files from `inbox/` or `raw/`, following §3 ingestion rules |
| **query** | `/query <question>` | Read hot.md + index.md + relevant pages; answer with citations |
| **lint** | `/lint [--autofix]` | Run all checks in §3 lint rules |
| **crystallize** | `/crystallize` | Distil the current conversation into a wiki page |
| **today** | `/today` | Generate today's daily note with open threads + calendar |
| **weekly-review** | `/weekly-review` | Summarize the week, refresh overview.md, close threads |
| **tech-digest** | `/tech-digest` | Compile the week's tech-news clips into `domains/tech-news/weekly/YYYY-WW.md` |
| **new-domain** | `/new-domain <name>` | Scaffold a new `domains/<name>/` with MoC, schema, templates |

Each skill is a ~50-200 line `SKILL.md` with a frontmatter header (name, description, triggers). Template and authoring rules: follow `obra/superpowers` writing-skills guide.

**Optional add-ons** (install when you hit the need):

- **Graphify** (`pip install graphifyy`) — when you want a full knowledge graph over `raw/` that you can visualize and query. Run `/graphify ./raw --obsidian --obsidian-dir ~/second-brain/graphify/main --update` weekly.
- **RAG-Anything** (`pip install raganything`) — when you have ≥20 heavy PDFs (research papers, financial reports, technical docs with equations/figures) and Claude's native reading isn't cutting it.
- **CLI-Anything** — if/when you need Claude to drive a specific app without a CLI. Not relevant to core KM.
- **Memory layer** (OpenMemory for full-local, Supermemory for managed) — only after 4-6 weeks of vault use reveals a specific gap the wiki doesn't fill.

---

## 8. Hooks — the unconditional enforcement layer

The lesson from repo-scoped KM: the LLM is the brain, but **only hooks are guaranteed to fire**. Anything you want enforced deterministically — vault writes, sensitive-path protection, auto-logging, hot-cache refresh — must live in a hook.

### 8.1 `.claude/settings.json` (committed)

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          { "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/session_start_context.sh",
            "timeout": 10 }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          { "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/block_dangerous_writes.sh",
            "timeout": 5 }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/block_dangerous_bash.sh",
            "timeout": 5 }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/log_ingestion_to_log_md.sh",
            "timeout": 5 }
        ]
      }
    ],
    "PreCompact": [
      {
        "hooks": [
          { "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/precompact_save.sh",
            "timeout": 30 }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          { "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/hot_cache_on_stop.sh",
            "timeout": 15 }
        ]
      }
    ]
  }
}
```

### 8.2 `.claude/hooks/block_dangerous_writes.sh` (PreToolUse on Write|Edit|MultiEdit)

Reads tool input JSON from stdin. **Denies** any write to:

- Paths under `raw/` (enforces §1.1 immutability)
- Paths under `~/.second-brain-local/` from a vault session (L1/L2 discipline)
- `.git/` internals
- `.claude/settings.json` without a `--force` flag in the command description (prevents accidental self-modification)

On deny, emits PreToolUse `permissionDecision: deny` with a helpful reason pointing at the relevant non-negotiable. On allow, emits nothing (exit 0).

### 8.3 `.claude/hooks/block_dangerous_bash.sh` (PreToolUse on Bash)

Same pattern as the repo-scoped version: blocks `rm -rf /`, `rm -rf ~`, `git push --force` (unless you explicitly want this on a branch), `sudo rm`, etc. For a personal vault the risk surface is smaller than a production repo, but the hook is cheap and the cost of a mistake is the whole vault.

### 8.4 `.claude/hooks/log_ingestion_to_log_md.sh` (PostToolUse on Bash)

On a successful `git commit` with a message matching the `ingest(` / `crystallize(` / `lint(` prefix, appends a structured entry to `wiki/log.md`. Extracts commit SHA + subject + body. Idempotent (skip if SHA already present).

This is what makes `log.md` a complete audit trail without manual effort.

### 8.5 `.claude/hooks/hot_cache_on_stop.sh` (Stop)

On session stop:

1. Compute diff: which `wiki/` files were touched since the last hot-cache refresh (timestamp stored in `wiki/.hot-cache-timestamp`).
2. Read the last 10 entries from `wiki/log.md`.
3. Read unresolved `[!blocker]` and `[!contradiction]` callouts added in this session.
4. Write `wiki/hot.md` with a structured summary:

```markdown
# Hot Cache (refreshed YYYY-MM-DD HH:MM)
## Active domains this session
- <employer>, startup-<codename>
## Pages updated (last 10)
- [[concepts/llm-wiki-pattern]] — expanded Karpathy section
- ...
## New open threads
- Blocker: API quota exhausted, ingestion queue at 3 items → BLOCKERS.md
- Contradiction: vendor benchmark vs independent eval → wiki/contradictions.md
## Current state
Mid-ingestion: 2 transcripts in inbox/ pending, ingestion paused on rate limit.
```

5. Touch `wiki/.hot-cache-timestamp`.

This is the **single most important hook** in the system. Without it, compaction recovery degrades to re-reading 20+ files blindly on every new session.

### 8.6 `.claude/hooks/session_start_context.sh` (SessionStart)

On session start, reads `wiki/hot.md` + `agent-memory/OPEN-THREADS.md` + `agent-memory/PREFERENCES.md` and injects them as `additionalContext` JSON so Claude sees them in context before its first token.

This turns "start of session" from "Claude has no memory" into "Claude picks up where it stopped."

### 8.7 `.claude/hooks/precompact_save.sh` (PreCompact)

Before the harness compacts the conversation, this hook:

1. Asks Claude (via a local prompt or a simple template) to emit a brief "what I learned in this session" into `raw/session-learnings/YYYY-MM-DD-HHMM.md`.
2. Triggers the Stop hook logic pre-emptively so `hot.md` is current.

Prevents valuable mid-session reasoning from being summarized into oblivion. Pattern from `reliabilitywhisperer.substack.com/p/the-andrej-karpathy-llm-wiki-idea`.

### 8.8 Per-machine: `.claude/settings.local.json` (gitignored)

For machine-specific tweaks: local model override, local paths, extra MCPs you don't want to share across machines.

---

## 9. Capture pipelines — the layer that makes this work

The rule: **capture must be zero-friction**. If it takes more than 3 seconds to get a thought into `inbox/`, you won't do it, and the vault dies.

### 9.1 Voice capture — Day-1 minimum (defer the rest)

**The Day-1 decision (revised 2026-04-23, ADR-0008):** do NOT install a standalone transcription app at launch. Claude Code's **built-in voice mode** covers desk dictation out of the box — no extra software needed. Two minimal surfaces are enough to start, both available with zero additional install:

1. **Claude Code voice mode** (native in the CLI). Press the mic button / platform shortcut inside a `claude` session and speak. The transcript lands directly in the conversation, which means it can be routed straight into `inbox/` or a specific note via a regular tool call. Use for: thinking out loud at the Mac, dictating into a specific note, quick capture while hands-busy.
2. **Obsidian voice memo plugin** (already installed during community-plugins step, §10 step 2). Backup path if Claude Code isn't open. Desk dictation directly into Obsidian notes.

That's it for Day 1. Everything else waits until the vault is actually running and the daily habit is formed.

**What you're explicitly deferring:** (a) any standalone transcription app for external audio files (meetings, interviews, long-form recordings) — use your meeting tool's native transcript export instead (§9.3), and drop the `.md` into `inbox/`. If a specific recording genuinely needs better transcription than the meeting tool produces, Claude Code can install a transcription helper (MacWhisper, `mlx-whisper`, OpenAI Whisper API) at the moment of need, not upfront. (b) the automated iPhone → iCloud → transcribe → `inbox/voice/` pipeline. It's the highest-value piece (always-on capture away from the laptop) but also the highest-friction to set up (Apple permissions, Automator quirks, dependency management). Building it on Day 1 means debugging it instead of using the vault.

**When to come back:** once you've used the vault for 2-3 weeks and felt the pain of "I had an idea on the walk but forgot to capture it." At that point you'll know exactly what you want the pipeline to do. See §20.1 "Custom voice pipeline (deferred)" for the design.

**Why not MacWhisper Pro as an always-on solution:** MacWhisper has no CLI, no AppleScript, no Shortcuts support — confirmed by the developer as of early 2026. It can't be automated into a folder-watcher pipeline. Great drag-and-drop tool; wrong primitive for always-on capture, and Claude Code voice mode already covers the desk-dictation case it would have solved on Day 1.

Record this in `agent-memory/DECISIONS.md` as ADR-0008.

### 9.2 Web capture

**Obsidian Web Clipper** browser extension → saves to `raw/clips/YYYY-MM/`. Set the template so it includes `source_url`, `clipped_at`, and a `tags:` field prompt.

### 9.3 Meeting transcripts

Most meeting transcription tools (Otter, Fathom, Granola, Tactiq, Zoom native, Google Meet native) can export `.md` or `.txt`. Pipe via:

- Direct folder-sync to `raw/transcripts/`, or
- A cron that pulls from the tool's API daily, or
- Manual drag-and-drop for the ones that matter.

For calls you run yourself: `silverstein/minutes` + a system-audio capture can run entirely local.

### 9.4 File captures (PDFs, DOCX, slides)

Drag to `raw/papers/` or `raw/articles/`. Claude's native reading handles most. For the hard ones (heavy tables, equations, figures-with-meaning), invoke RAG-Anything — but only when actually needed, not by default.

### 9.5 Email / Slack / WhatsApp

The hard ones. Options:

- **Manual triage** — copy/paste important threads into `raw/messages/`. Scales poorly.
- **MCP servers for email/Slack** — install the relevant MCP; Claude pulls threads on demand. Better.
- **Periodic export scripts** — weekly cron that dumps Gmail's "knowledge-worthy" label into `raw/messages/`. Best for high-volume email users.

Start with the first, graduate to the second when it becomes a bottleneck.

### 9.6 Quick capture (inline thoughts, drafts)

Obsidian's daily note has an "Inbox" section at the top. You write stream-of-consciousness there during the day. `/today` at morning and `/ingest inbox/` in the evening drain it.

---

## 10. Setup checklist — executable, in order

Execute in order. Commit after each step with a meaningful message.

> **Prerequisite:** §A "Human prerequisites" must be complete before starting Phase 1. Specifically: Obsidian, Claude Code, and Homebrew are installed; macOS permissions granted; GitHub auth done; sync strategy chosen (usually Obsidian+ per §A.5).

### Phase 1 — Foundation (Day 1, ~2-3 hours)

1. **Install Obsidian.** Create vault at `~/second-brain`. Disable unnecessary default plugins (Templates — we use Templater; Backlinks — we use graph view). Initial commit: `chore: init vault`.
2. **Install community plugins:** Local REST API, Templater, Dataview, Obsidian Git (for auto-commits), Periodic Notes, Advanced URI, Omnisearch, QuickAdd. Configure Local REST API: copy API key to password manager, pick port 27124 HTTPS. Commit: `chore: install community plugins`.
3. **Create the full folder structure from §2.** Empty folders with `.gitkeep` files. Commit: `chore: scaffold folder structure`.
4. **Write `CLAUDE.md`** from §3 template. Fill in your actual domains (employer, startup codename, freelance, tech-news, personal). Write non-negotiables verbatim. Write frontmatter contract verbatim. Customize ingestion rules. Commit: `docs(meta): write CLAUDE.md`.
5. **Write `AGENTS.md`** as a pointer-to-CLAUDE.md stub (one paragraph + a copy of the non-negotiables). Commit: `docs(meta): write AGENTS.md`.
6. **Write `README.md`** — what this vault is, for you / for future-you / for non-technical humans. Commit: `docs: write README`.
7. **Create `_templates/*.md`** from the types in §5. Simple YAML headers + section skeletons. Commit: `docs(templates): scaffold templates`.
8. **Seed `agent-memory/`** — write all six files with headers per §4, empty bodies. Write your first `PREFERENCES.md` from §4.5 template. Commit: `docs(memory): seed agent-memory`.
9. **Configure `.gitignore`** (§14.1) and `.gitattributes`. Commit: `chore: configure git`.
10. **Initialize git remote** (GitHub private repo is the canonical choice; Gitea self-hosted if you want full sovereignty). Push. Commit nothing — push only.

### Phase 2 — Claude Code integration (Day 1-2, ~2 hours)

11. **Install Claude Code** (`npm install -g @anthropic-ai/claude-code` or follow the docs). Verify with `claude --version`.
12. **Install the Superpowers plugin.** From inside a Claude Code session at the vault root:
    ```
    /plugin marketplace add obra/superpowers-marketplace
    /plugin install superpowers@superpowers-marketplace
    /help   # verify /brainstorm /write-plan /execute-plan appear
    ```
13. **Configure the Obsidian MCP.** In your Claude Code MCP config (`~/.claude.json` or per-project `.mcp.json`):
    ```json
    {
      "mcpServers": {
        "obsidian": {
          "command": "uvx",
          "args": ["mcp-obsidian"],
          "env": {
            "OBSIDIAN_API_KEY": "<from Local REST API plugin>",
            "OBSIDIAN_HOST": "127.0.0.1",
            "OBSIDIAN_PORT": "27124"
          }
        }
      }
    }
    ```
    Restart Claude Code. Verify: `/mcp` should list `obsidian` with connected status. Ask Claude "list files in my vault"; it should succeed.
14. **Install hook scripts.** Copy the five `.sh` files from §8 into `.claude/hooks/`, `chmod +x`. Write `.claude/settings.json` per §8.1. Commit: `feat(hooks): install hook pipeline`.
15. **Verify hooks fire.** Make a no-op `git commit --allow-empty -m "chore: hook test"` — the PostToolUse hook shouldn't fire (it matches only `ingest|crystallize|lint` prefixes). Make another: `git commit --allow-empty -m "ingest(meta): hook test"` — the hook **should** fire and append to `wiki/log.md`. If it didn't: debug; do not continue until hooks work.
16. **Write the eight vault-local skills** (§7.2). Each ~50-200 lines, following the `SKILL.md` frontmatter format. This is the big work of setup — budget 2-3 hours. Commit after each skill: `feat(skills): add /<skillname>`.

### Phase 3 — First ingestions (Day 2-3)

17. **Seed each domain's `_moc.md` and `_schema.md`.** Write 1-paragraph map-of-content per domain, with links to the top 3-5 most important wiki pages or meeting notes that already exist in your head. The schema can be empty at first — you'll grow it.
18. **Ingest your first 10 sources.** Drop things you've been meaning to file into `inbox/` or `raw/`. Run `/ingest` one at a time. Watch what Claude does. Correct the schema when it misfiles something. Commit after each: `ingest(<domain>): <title>`.
19. **Run the first `/lint`.** Expect orphans (new pages with no incoming links yet — that's fine for the first 10 sources). Expect dead links (you referenced pages that don't exist yet — create them or remove the link). Do not `--autofix` yet; walk through manually to learn Claude's style.
20. **Write your first `/today`.** See if it handles the capture → daily note → backlinks flow correctly. Adjust the skill if not.

### Phase 4 — Capture pipeline (Day 3-5)

21. **Pick ONE voice-capture option from §9.1.** Set it up. Test end-to-end: voice → transcript in `inbox/voice/` → `/ingest` → wiki page + daily note.
22. **Install the Obsidian Web Clipper browser extension.** Configure template. Clip one article. Verify it lands in `raw/clips/`.
23. **Wire up one meeting transcription source.** Pick your highest-volume meeting tool; set up export to `raw/transcripts/`.
24. **Do a 30-minute "inbox drain" at end of day for 3 days running.** This is the habit formation. If you can do it for a week straight, you've installed the system. If you can't, the capture layer has too much friction — simplify.

### Phase 5 — Cross-device sync (Day 5-7)

> **If you chose Obsidian+ per §A.5, most of this phase is already done.** Verify: edit a file on laptop, see it appear on desktop or iPhone within a minute. Skip to step 27. If you're doing Syncthing or another strategy, proceed with 25-27.

25. **Pick a sync strategy (§11):** Obsidian Sync via Obsidian+ (recommended, non-technical), Syncthing (power-user), Obsidian Git push-pull (manual, clean history), iCloud (**do not** — conflict files). Write the decision to `DECISIONS.md`.
26. **Configure sync on your second device.** Use Obsidian mobile if you want read/some-write access on the go.
27. **Test conflict scenarios.** Edit the same file on two devices offline. Sync both. Verify the resolution is sane.

### Phase 6 — Reinforce (Week 2)

28. **Run `/weekly-review`.** See if it captures the week sensibly. Iterate.
29. **Write your first `wiki/overview.md`** via `/crystallize`. This is the moment the vault feels real.
30. **Do NOT install Graphify yet.** The spec previously suggested this at Phase 6 but it's premature — Graphify needs ~50+ real sources to produce useful output. At 0 sources it generates an empty graph. Skip this step entirely during initial setup. See §20.2 for when to come back to it (~Week 2-3, after 20-50 sources are ingested).

### Acceptance criteria (the install is done when)

- `git log --oneline` shows 20+ commits from setup.
- `/ingest`, `/query`, `/lint`, `/today`, `/crystallize`, `/weekly-review`, `/tech-digest`, `/new-domain` all work.
- All four domains (<employer>, startup-X, freelance, tech-news) have a non-empty `_moc.md` and at least 5 pages.
- `wiki/index.md`, `wiki/overview.md`, `wiki/log.md`, `wiki/hot.md` all exist and are populated.
- A synthetic `git commit -m "ingest(meta): acceptance test"` triggers the PostToolUse hook and appends to `wiki/log.md`.
- Starting a new Claude Code session reads `wiki/hot.md` via SessionStart hook and displays last-session context immediately.
- `/lint` runs clean (no unexpected orphans, no dead links, no sourceless wiki pages).
- Cross-device sync works: edit a file on laptop, appears on phone within 60 seconds (Syncthing) or on next Obsidian Sync cycle.
- `agent-memory/TIMELINE.md` has at least 10 entries (your actual life history relevant to the vault).

---

## 11. Sync — the invariant that breaks most vaults

The #1 cause of vault death after "lost the capture habit" is **sync conflict files** (`file 1.md`, `file 2.md`, `file.md.conflict-2026-04-19`). Pick carefully.

| Strategy | Verdict | When |
|---|---|---|
| **Obsidian Sync (via Obsidian+)** | **Recommended default for non-technical operators.** One subscription, toggled on inside Obsidian. Works seamlessly across macOS, Windows, iOS, iPadOS, Android. End-to-end encrypted. Version history per file. ~$10/mo. Zero maintenance. | The CEO-friendly path. If you're reading this spec as a handoff from someone else, this is almost certainly what you want. |
| **Syncthing** | Technically excellent. Peer-to-peer, handles conflicts well, fully private (no third-party server), free, open-source. Desktop setup is easy; mobile requires a separate paid app (Möbius Sync on iOS). Small setup cost, lifetime $0. | Power users who want no cloud intermediary. Desktop ↔ desktop ↔ NAS. |
| **GitHub push-pull** | Manual (`obsidian-git` plugin auto-commits). Clean history. Free. Does not cover mobile. | Desktop-heavy workflow where you also want full versioning you can browse in a git UI. Pair with another strategy for mobile. |
| **iCloud Drive** | **Avoid.** Metadata files, conflict files, unpredictable sync timing, breaks `.obsidian/workspace.json`. | Nope. |
| **Dropbox** | **Avoid.** Same problems as iCloud, slightly less bad. | Nope. |
| **OneDrive** | **Avoid.** Worst of both. | Nope. |
| **Obsidian LiveSync (CouchDB plugin)** | Real-time multi-device, end-to-end encrypted, self-hostable. Requires running a CouchDB server yourself. | Power users who want Google-Docs-like real-time sync and are willing to run infra. |

**Recommended stacks:**

- **Non-technical operator (default):** Obsidian+ subscription → Obsidian Sync for every device. Done. Pair with `git push` once a week as a belt-and-braces backup to GitHub — the `obsidian-git` plugin can automate this. *This is what a CEO-handoff install should choose unless there's a specific reason not to.*
- **Power user / privacy-max:** Syncthing for desktop↔desktop, Möbius Sync (paid iOS app that speaks Syncthing protocol) for mobile, `git push` for versioned backup.
- **Desktop-only:** `obsidian-git` auto-commits to GitHub. Skip mobile until you need it.

**Note about Obsidian+.** Obsidian+ (marketed as "Obsidian Plus" on the website) is Obsidian's paid subscription that bundles Obsidian Sync + Obsidian Publish + a few smaller goodies. You only need the Sync capability for this setup; Publish is for turning notes into a public website and is unrelated. The Sync component is end-to-end encrypted — Obsidian's servers see ciphertext, not your vault contents.

**L1 vault (`~/.second-brain-local/`)** — do NOT sync via the same channel as L2. Use Obsidian Sync with a **separate, L1-only vault configured inside the same Obsidian+ subscription**, or keep single-device. Do not mix the two vaults in one Sync pool.

---

## 12. Operating procedures — the rules you live by

### 12.1 Morning (5 min)

1. Open Claude Code at vault root. Hot cache loads automatically.
2. `/today` → Claude drafts today's daily note with open threads and (if calendar MCP is wired) today's meetings.
3. Scan `agent-memory/OPEN-THREADS.md` in Obsidian — mentally prioritize 2-3 things.

### 12.2 Mid-day (continuous, zero-friction)

- Voice notes → `inbox/voice/` (automated).
- Web clips → `raw/clips/` (one click).
- Quick thoughts → append to today's daily note Inbox section (fastest path).
- Important messages/emails → forward/copy to `inbox/`.

### 12.3 End of day (10-15 min)

1. `/ingest inbox/` — Claude drains what you captured today.
2. Scan what Claude wrote. Correct anything wrong. Resolve any `[!contradiction]` or `[!blocker]` callouts that are quick.
3. Close the session — Stop hook writes `wiki/hot.md` automatically.

### 12.4 End of week (30 min, Sunday)

1. `/weekly-review` — Claude generates a summary across all daily notes, surfaces patterns, updates `wiki/overview.md`.
2. Update `agent-memory/OPEN-THREADS.md` — close what's resolved, add what's new.
3. `/lint` — fix orphans, dead links, stale pages.
4. `/tech-digest` — generates `domains/tech-news/weekly/YYYY-WW.md` from the week's tech clips.
5. Graphify update: `graphifyy ./raw --update --obsidian`.
6. Append a macro entry to `PROGRESS.md` if warranted ("MILESTONE: passed 500 sources").

### 12.5 End of month

- Review `wiki/contradictions.md` — resolve what you can, flag what you can't.
- Review `agent-memory/BLOCKERS.md` — mark resolved, escalate long-standing ones.
- Archive old daily notes older than 90 days (optional, keeps graph readable).
- Audit `CLAUDE.md` — has the domain landscape changed? Update.

### 12.6 On entering a new domain (e.g., new client, new startup, new phase of life)

1. `/new-domain <slug>` — scaffolds the folder with MoC, schema, templates.
2. Fill in the domain's `_schema.md`: what kinds of entities are unique to this domain, what pages exist, what the naming conventions are.
3. Add to `CLAUDE.md § Domains`.
4. Ingest the first 5 sources to seed it.

### 12.7 On context compaction (mid-session)

The harness compacts when context gets long. The PreCompact hook fires; harvests learnings; the Stop-hook-equivalent refreshes `hot.md`. When the new, compacted session continues:

1. Claude reads `wiki/hot.md` (via SessionStart hook on resume).
2. Reads `agent-memory/OPEN-THREADS.md`.
3. If mid-ingestion: reads `wiki/log.md` last entry to resume exactly where it stopped.
4. If mid-plan (using Superpowers `/execute-plan`): the plan file is the resume point.

### 12.8 On the 30-minute stall (§1.6)

Claude (or you) is spinning on a single ingestion, page, or question with no progress. Protocol:

1. Commit whatever's half-done with a `WIP(<scope>):` prefix.
2. Add a `> [!blocker]` callout on the problematic page.
3. Log to `agent-memory/BLOCKERS.md` using the §4.3 format.
4. Move to the next thing.

Perfectionism is how vaults die. 80% is fine; the wiki is self-healing under `/lint` and future ingestions that revisit the topic.

---

## 13. Failure modes & mitigations

| Risk | Mitigation |
|---|---|
| I stop capturing → vault dies | Zero-friction capture (§9). Daily note Inbox. Voice → auto-transcript. |
| Claude invents facts not in sources | Provenance requirement (§1.3) + `/lint` sourceless check. Rejects wiki pages without `sources:`. |
| Wiki gets overwritten silently on contradiction | Contradiction callout discipline (§1.4) + `wiki/contradictions.md` meta-page. |
| I lose progress after compaction | Hot cache hook (§8.5) + SessionStart injection (§8.6). |
| Mobile edit + desktop edit = conflict file | Syncthing or Obsidian Sync. NOT iCloud/Dropbox. |
| Credentials accidentally committed to the repo | L1/L2 split (§6) + `.gitignore` catch-all + secret-scanner pre-commit hook (optional but recommended). |
| Graph becomes unreadable past ~5k nodes | Per-domain graph filters in Obsidian. Archive old daily notes. Move cold content to a sibling vault. |
| Ingestion eats tokens on recurring topics | Hot cache means recurring topics resolve from 200-token summary, not full reads. Graphify for structural queries without full-text reads. |
| Lint finds hundreds of issues after weeks of neglect | Don't `--autofix` blindly. Walk through manually for the first 2-3 lint passes; once you trust Claude's style, autofix becomes safe. |
| Ingestion hook fails silently and nothing logs | `wiki/log.md` gaps are a diagnostic signal. Weekly review catches: last entry timestamp >48h old = hook broken. |
| Vault grows past comfort zone (~20k notes) | Archive strategy: cold content → sibling vault. Per-domain query scoping in skills. |
| I don't trust Claude with some personal content | L1 vault. Separate MCP scope. Obsidian multi-vault support. |
| A raw source proves wrong months later | Move to `raw/_archive/`, update the cited wiki pages with the new source, add an ADR explaining the change. Never delete raw. |
| I get sick / burned out and neglect the vault for a month | It's fine. The wiki doesn't rot in storage — only capture stops. Resume by ingesting whatever accumulated. Write a `PROGRESS.md` entry noting the gap. |

---

## 14. Concrete artefacts — paste-ready templates

### 14.1 `.gitignore`

```gitignore
# Obsidian workspace (machine-specific)
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.obsidian/cache
.trash/

# Claude Code per-machine
.claude/settings.local.json

# L1 placement alternative (if you ever put L1 inside the vault — don't, but safety net)
.local/
local-*

# OS
.DS_Store
Thumbs.db

# Editor
.vscode/
.idea/

# Graphify index (regenerable; commit only the notes)
graphify/*/graph.json
graphify/*/report.html

# Hot-cache timestamp (ephemeral)
wiki/.hot-cache-timestamp

# Secrets (belt-and-braces; should never be here, but just in case)
*.key
*.pem
.env
```

### 14.2 Example `domains/<domain>/_moc.md`

```markdown
---
type: moc
domain: <employer>
tags: [moc, <employer>]
created: 2026-04-19
updated: 2026-04-19
---

# <Employer> — Map of Content

## What
Your employer since <start date>. Role: <role>. <Contract type>.
Reports to <manager roles> directly. Responsible for <your remit>.

## Key people
- [[people/<ceo-name>]] — CEO
- [[people/<cio-name>]] — CIO, your direct manager
- [[people/<other>]] — ...

## Active workstreams
- [[domains/<employer>/<workstream>/framework]] — Adoption framework rollout
- [[domains/<employer>/<workstream>/core-platform]] — the core platform
- [[domains/<employer>/<workstream>/agents]] — Custom agent work
- [[domains/<employer>/<workstream>/claude-code]] — Claude Code enablement

## Recent decisions
- [[domains/<employer>/decisions/2026-03-adoption-tiers]]
- ...

## Open threads (see also OPEN-THREADS.md § <employer>)
- Phase 2 rollout criteria
- CIO sign-off on phase 2 budget
- <next strategic decision for this domain> by <date>

## Strategic context
- <key strategic variable for this domain> by <date>
  (see [[wiki/syntheses/<relevant-synthesis>]])
- <external option or alternative path being tracked>
- Long-term: <where this domain is heading>
```

### 14.3 Example `domains/<domain>/_schema.md`

```markdown
---
type: schema
domain: <employer>
updated: 2026-04-19
---

# <Employer> — Domain Schema

## Entity types in this domain
- **account** — internal or external project (e.g., Project Atlas)
- **deliverable** — something I ship (tool, framework, training)
- **meeting** — any employer meeting, filed under meetings/YYYY/
- **decision** — architectural or strategic choice (ADR-style)

## Naming conventions
- Meetings: `YYYY-MM-DD-<slug>.md` where slug is lowercase-kebab
- People: linked from `/people/` (global), not domain-local
- Accounts: `/accounts/<account-slug>/` with its own sub-MoC if >5 related notes

## When to create a new entity page
- A project gets >3 meetings → make an account page
- A decision touches 2+ workstreams → make a decision page
- A person is mentioned in 3+ notes → ensure they have a people/ page

## Confidentiality
- No client-identifiable information in filenames if the context is sensitive
- No internal financial details (revenue, margins) — those go in L1 if at all

## Cross-domain relationships
- Employer learnings that generalize → wiki/concepts/
- People also relevant to freelance → links in both /people/ and
  /domains/<employer>/_moc and /domains/freelance/_moc
```

### 14.4 Example `.claude/skills/ingest/SKILL.md`

```markdown
---
name: ingest
description: Use when new files land in inbox/ or raw/ and need to be processed into the wiki
---

# Ingest — process sources into the wiki

## When to use
- User drops files into inbox/ or raw/ and says "process these" / "ingest"
- User runs /ingest [path]
- An automated pipeline deposits a file with a `needs-ingestion` marker

## Protocol
Read CLAUDE.md § "Ingestion rules" (the authoritative checklist).

In brief:
1. Enumerate unprocessed files (inbox/ or the specified path)
2. For each file, in order:
   a. Read fully
   b. Identify type (article, transcript, voice, clip, message)
   c. Find 3-10 existing wiki pages it touches (use MCP obsidian search +
      reading wiki/index.md)
   d. If type is meeting transcript: also file under domains/<domain>/meetings/
      and extract decisions + action items
   e. If type is voice note: extract TODOs, file to right project, update
      today's daily note
   f. Create or update wiki pages per the templates in _templates/
   g. If new source contradicts existing wiki content: add [!contradiction]
      callout on affected page + entry in wiki/contradictions.md
   h. Move source from inbox/ to raw/<category>/YYYY-MM/
   i. Append to wiki/log.md (will also be auto-logged on commit)
3. After all sources processed, commit with message:
   `ingest(<domain>): <combined summary> — N pages (+M new)`

## What NOT to do
- Do not edit raw/ files after moving them (non-negotiable #1)
- Do not silently overwrite contradictory claims (non-negotiable #4)
- Do not create wiki pages without at least one source
- Do not process more than 10 sources in one commit — split them

## Tools needed
- obsidian MCP (read/write vault)
- filesystem MCP (move files between raw/ subdirs)
- Bash (for git commit)

## Output
A commit that added/updated N wiki pages. User can verify by:
- Reading wiki/log.md last entry
- `git show HEAD --stat`
- Opening the affected wiki pages in Obsidian
```

---

## 15. What NOT to install, in order of how commonly people get it wrong

**Explicit scope for Claude Code during initial setup (§10 Phases 1-5):** install ONLY the things listed below. Anything else the user brings up during setup should be politely deferred.

**Install during setup:**

- `uvx mcp-obsidian` (via `claude mcp add obsidian ...`) — the MCP bridge to the vault
- Superpowers plugin (`/plugin install superpowers@superpowers-marketplace`)
- Vault-local skills written as markdown files in `.claude/skills/` — these are just files, no package install

**Do NOT install during setup (defer per §20):**

- Graphify — zero value on an empty vault; wait for Week 2-3 after 20-50 sources ingested
- RAG-Anything — surgical tool only, no value at Day 1
- CLI-Anything — not relevant to personal knowledge management at all
- GitNexus — code-repository tool, this vault is not a code repo; skip entirely
- Supermemory / Mem0 / OpenMemory — vault IS the memory layer for 4-6 weeks
- Any standalone voice transcription app (MacWhisper, `mlx-whisper`, etc.) — Claude Code voice mode covers desk dictation; external audio gets transcribed by the meeting tool itself (§9.1, §9.3). Install a transcription helper ad-hoc only when a specific recording needs it.
- Any local LLM runtime (Ollama, LM Studio) — deferred privacy upgrade
- Additional MCP servers (GitHub, Gmail, Calendar, etc.) — can be added later when a concrete need emerges

**General principles:**

1. **Don't install three Obsidian AI plugins** (Smart Connections + Copilot + CoPilot + …). They're all local-RAG over your vault. Claude Code via MCP already does this. Pick one if you want in-Obsidian chat; don't layer.
2. **Don't add a vector DB on day 1.** Karpathy's point: for <10k pages, structured markdown + graph traversal + BM25 is enough. Vector search earns its keep at 10k+ pages or when specific recall failures emerge. Then: LightRAG (RAG-Anything's backend), Qdrant, or Pinecone.
3. **Don't run Graphify AND RAG-Anything as parallel defaults.** They produce different artifacts. Graphify is for the whole corpus structural view. RAG-Anything is a surgical tool for heavy multimodal PDFs. Default to Graphify; invoke RAG-Anything only when needed.
4. **Don't install a memory layer (Supermemory/Mem0/OpenMemory) before you've used the vault for 4-6 weeks.** The vault IS a memory layer. The memory tool earns its place only if you specifically hit: (a) need cross-client memory (claude.ai + Claude Code + future tools sharing state), (b) need structured user-profile state, (c) need auto-summarization you can't get from Claude + wiki.
5. **Don't install more than one voice-capture option.** Pick one from §9.1 and commit.
6. **Don't build custom MCP servers for things that already have MCPs.** Check the MCP registry first (`tool_search` or Anthropic's registry).
7. **Don't commit `.obsidian/workspace.json`** (it's machine-specific and will cause constant diff noise).
8. **Don't invent skill names.** If you want a skill, write it in `.claude/skills/` or install one via `/plugin`. Never reference a skill Claude doesn't have.
9. **Don't try to make Obsidian the single tool for everything.** Keep a password manager (1Password/Bitwarden). Keep a calendar (Google/Fastmail). Keep task management separate if you need real task management (Things, OmniFocus, Linear for work). The vault is a knowledge base, not a todo app.
10. **Don't backfill "all my notes from the past 5 years" in week 1.** You'll burn out. Backfill opportunistically: when you need a concept and realize you had relevant notes elsewhere, pull them in then.

---

## 16. Per-domain deep config (worked example)

This section walks one illustrative four-domain setup. Replace the example domain
names, roles, and figures with your own.

### 16.1 `domains/<employer>/` — shaped for your role

Key subfolders beyond the defaults:

- `<key-workstream>/` — e.g. an AI-adoption framework, competency tiers, rollout phases. This is your load-bearing work.
- `core-platform/` — the core platform
- `agents/` — custom agent work (Project Atlas, a data-vendor integration, etc.)
- `deliverables/<flagship-deliverable>/` — the actual artifact you're shipping
- `thesis/` — the thesis-for-management pitch (talent amplification → speed → competitive advantage). This is the doc you keep iterating.

**Domain schema highlights (`_schema.md`):**

- Meetings tagged by attendee group: `#board`, `#cio`, `#peer`, `#team`
- Decisions have a `forum:` field (CEO-1:1, CIO-1:1, all-hands, etc.)
- Ship artifacts get a `status:` of draft/review/approved/shipped

**Cross-links to maintain carefully:**

- Every employer meeting should touch `people/` pages for attendees
- The workstream's concepts should live in `wiki/concepts/` when generic (cross-domain), `domains/<employer>/<key-workstream>/` when employer-specific

### 16.2 `domains/startup-<codename>/` — shaped for your co-founded venture

Key subfolders:

- `cofounder/` — shared context. This is the folder you and your co-founder both contribute to (if syncing a shared folder). Contains: shared beliefs, disagreements we want to track, agreed-upon strategy docs.
- `strategy/` — go-to-market, positioning, IP
- `product/` — technical architecture, roadmap, decisions
- `legal/` — equity, IP agreements, co-founder agreement (redacted; full copies in L1)
- `investor/` — pitch materials, investor notes, meeting prep (confidential parts in L1)
- `<prospect-track>/` — a specific named opportunity (investor, partner, anchor customer) tracked separately

**Schema highlights:**

- Every strategic conversation with co-founder gets a `conversation:` entry
- Disagreements get a `type: disagreement` page with both positions
- A named opportunity gets an entity page in `wiki/entities/<prospect-company>.md` + a folder here for the specific negotiation track

### 16.3 `domains/freelance/` — shaped for your independent / contracting work

Key subfolders:

- `leads/<company>/` — one folder per lead
- `key-accounts/<company>/` — converted clients
- `stakeholders/` — people relevant across multiple accounts (recruiters, intermediaries)
- `proposals/` — templates + sent proposals + outcomes
- `invoices/` — billing tracking: revenue per year, any <local tax regime> threshold monitoring, deductibles
- `contracts/` — redacted contracts (full in L1)
- `pricing/` — your rates and rate-justification notes

**Schema highlights:**

- Every lead has a `stage:` (cold / warm / discovery / proposal / closed-won / closed-lost)
- Every month, a `domains/freelance/invoices/YYYY-MM-summary.md` tracks cumulative revenue vs any annual threshold that applies to your <local tax regime>
- Pricing conversations with new prospects link to `wiki/concepts/freelance-pricing-principles.md` (which captures your pricing principles and rate notes)

This is the domain that most benefits from a **CRM-like Dataview dashboard** in `domains/freelance/_dashboard.md` — a live-querying page that surfaces: leads by stage, time-to-response, pipeline value, MRR forecast.

### 16.4 `domains/tech-news/` — shaped for keeping up

Key subfolders:

- `weekly/YYYY-WW.md` — weekly digest (auto-generated via `/tech-digest`)
- `trends/` — emerging topic pages (MCP, agentic coding, local LLMs, etc.)
- `tools/` — tool-specific pages (Claude Code, Cursor, Codex, OpenCode)
- `papers/` — tracked academic work

**Schema highlights:**

- Weekly digest pulls from `raw/clips/` of the past 7 days
- Trend pages are the only pages that can have `status: hypothesis` without a strict source requirement (you're tracking emerging things where sources are thin)
- Tool pages cross-link to your team's or employer's relevant frameworks where the signal lives

---

## 17. Strategic ADRs — decisions you should write early

Most people have a handful of load-bearing strategic decisions that deserve first-class ADRs in `agent-memory/DECISIONS.md`. Writing them now makes every future Claude session instantly oriented to why things are the way they are. The first three below are illustrative placeholders — replace them with your own load-bearing choices.

**Suggested ADRs to write in setup phase:**

- **ADR-0001 — *(illustrative placeholder — replace with your own)* Choose a business/operating structure and a date to revisit it.** Context: the tradeoff between a simpler structure and one with more headroom — thresholds, overhead, and admin burden all weigh in. Consequences: the choice constrains how you track activity against any applicable limits, switching later has a switching cost, and the decision tends to bind for a couple of years. *This is what a load-bearing structural ADR looks like; fill in your actual structure, thresholds, and review date.*
- **ADR-0002 — *(illustrative placeholder — replace with your own)* Decide your focus / positioning for the next planning horizon.** Context: weighing where to concentrate effort across the domains this vault serves — the relative payoff, the opportunity cost of each path, and how reversible the choice is. Consequences: commits attention and resources for the horizon, opens some paths while closing others, and sets a date to re-evaluate. *Replace with the actual positioning or focus decision you face.*
- **ADR-0003 — *(illustrative placeholder — replace with your own)* Cap exposure to any single counterpart to limit concentration risk.** Context: depending too heavily on one client, customer, or channel is fragile. Consequences: stating a generic concentration limit forces you to diversify and cultivate other relationships rather than letting one source dominate. *Set your own threshold and the counterpart type it applies to.*
- **ADR-0004 — Use Obsidian + Claude Code + Karpathy LLM Wiki pattern for knowledge management.** Context: looking at mainstream options (Notion, Roam, LogSeq) and AI-native memory layers; chose local-first + AI-maintained for compounding. Consequences: depends on Claude API availability; lock-in to markdown; requires discipline to not manually edit wiki/.
- **ADR-0005 — Choose Obsidian+ for cross-device sync (revised from Syncthing decision).** Context: simple cross-device setup; one subscription covers desktop↔laptop↔iPhone↔iPad; end-to-end encrypted; zero maintenance. Consequences: ~$10/mo recurring; data at rest on Obsidian's servers (ciphertext only); each device needs one-time login. *If privacy-max is the priority, supersede with an ADR choosing Syncthing + Möbius Sync.*
- **ADR-0006 — L1 separate vault for credentials, health, financial details.** Context: vault is git-tracked and cloud-synced; some data is dangerous/embarrassing to leak. Consequences: two-vault workflow; MCP only connects to L2 by default; manual toggle for L1 access.
- **ADR-0007 — Not opening Claude.ai auto-memory integration yet.** Context: vault already serves as memory layer; wait 4-6 weeks before adding Supermemory/OpenMemory. Consequences: some cross-surface state (claude.ai ↔ Claude Code) doesn't persist; revisit in 6 weeks.
- **ADR-0008 — Day-1 voice capture = Claude Code voice mode + Obsidian voice plugin; no standalone transcription app; custom iPhone pipeline deferred.** Context: Claude Code ships with built-in voice input, which covers desk dictation without any extra install. External meeting audio is already transcribed by the meeting tool itself (§9.3); a standalone transcription app on Day 1 adds install friction without solving a real problem. Building an always-on iPhone → vault pipeline on Day 1 means debugging Apple permissions and Automator quirks instead of forming the vault habit. Consequences: thoughts captured away from the laptop/phone may be lost during the first 2-3 weeks — accept this as the cost of getting the core vault running cleanly. A transcription helper (MacWhisper, mlx-whisper, OpenAI Whisper API) can be installed ad-hoc by Claude Code when a specific recording genuinely needs it. Custom iPhone pipeline designed in spec §20.1, built after 2-3 weeks of real vault use when specific requirements are known. Supersede this ADR with a new one when the pipeline ships.

Each ADR ~200 words. Together they're the map of why.

---

## 18. Scaling — what breaks at what scale, and what to do

| Scale | What's fine | What breaks | Fix |
|---|---|---|---|
| **<100 pages** | Everything. Hot cache optional. Graph view pretty. | Nothing. | Enjoy it. |
| **100-500 pages** | Still fluid. | `/query` starts reading 15+ pages for some questions. | Tighten `index.md`. Start using Graphify for structural questions. |
| **500-2000 pages** | Core flow still works. | Full-vault grep becomes slow. Lint pass takes minutes. | Per-domain skill scoping. Obsidian's native search starts beating grep — use it. Consider Omnisearch plugin. |
| **2000-5000 pages** | Specialized queries fine. | Broad questions get expensive. Graph view gets unusable. | Mandatory per-domain query scoping. Archive daily notes >1yr to sibling vault. Introduce vector search layer (LightRAG) over the wiki. |
| **5000-10000 pages** | With discipline, works. | Claude's context can't hold `overview.md` + relevant chunks. | Hierarchical overview: `wiki/overview.md` + per-domain `_overview.md`. Heavy Graphify reliance. |
| **>10000 pages** | Possible but requires serious infrastructure. | This is where Karpathy's "just markdown" hits real limits. | LightRAG-backed retrieval; dedicated memory layer; consider splitting vault by era or domain. |

**When to look at each upgrade:**

- **Graphify for retrieval:** at 500+ pages
- **Obsidian native search via Omnisearch:** at 1000+ pages
- **LightRAG / vector layer:** at 2000+ pages, OR whenever you feel queries are taking too long
- **Dedicated memory layer:** at any scale, only if you have a specific cross-surface need
- **Vault split:** at 5000+ pages, only if a clean archival line exists (e.g., "everything before Oct 2026 → archive vault")

---

## 19. The meta — this spec itself

This file lives at `~/second-brain/docs/SECOND-BRAIN-SPEC.md` (`docs/` at vault root — create it in setup step 6). It's subject to its own rules:

- Commit it with an `ingest(meta):` prefix when you rewrite it
- It has a source: your repo-scoped KM spec + this conversation
- Update it when you learn something that would have saved hours if documented

**When to rewrite sections:**

- §3 `CLAUDE.md` template: when domains change (new employer, new startup, dropped service line)
- §7 MCPs/skills: when you install/remove any
- §8 hooks: when you add a new discipline
- §16 per-domain: ongoing
- §17 ADRs: never rewrite; add new ones that supersede

**When to rewrite the whole doc:**

- When the model landscape shifts (new cutoff, new context window sizes, new tools)
- When you've done ≥3 sprints against it and learned what's wrong
- When you hit a scale boundary (§18) and the architecture needs to evolve

---

## 20. Deferred enhancements — build these after Day 1

The vault works on Day 1 with the capture surfaces in §9. This section holds designs for upgrades you'll want later, once the habit is formed and the specific pain points are real. Build criterion is the same for everything here: **build it when you've felt the pain of not having it for at least a week**. Not before.

### 20.1 Custom voice pipeline — always-on iPhone capture

**Goal.** Any voice thought captured on iPhone — on a walk, in the car, in bed — lands as a transcribed markdown note in `~/second-brain/inbox/voice/` within a few minutes, with zero manual steps, leveraging the Apple ecosystem end-to-end. Your best strategic thinking happens away from the keyboard; this surface is the one that catches it.

**Why deferred.** Setup cost is real (Apple permissions, Automator quirks, dependency management, inevitable debugging). Building it before the vault habit is formed means debugging the pipeline instead of using the vault. Build when the pain of "I had an idea on the walk and it's gone" is concrete.

**Target architecture (to refine when the time comes):**

```
iPhone Voice Memos app (already installed, zero-friction)
    ↓ iCloud sync (automatic, Apple-native)
Mac folder: ~/Library/Group Containers/group.com.apple.VoiceMemos.shared/Recordings/
    ↓ macOS Folder Action (Automator) triggers on new .m4a
Local transcription (mlx-whisper on Apple Silicon, runs offline, <your languages>)
    ↓ shell script renames, formats YAML frontmatter, handles dedup
~/second-brain/inbox/voice/YYYY-MM-DD-HHMMSS-<slug>.md
    ↓ your next Claude Code session runs /ingest
Wiki pages updated, daily note updated, source moved to raw/voice/
```

**Components to build (Claude Code can write most of this):**

1. **Folder Action shell script.** Triggers on `.m4a` arrival. Calls `mlx-whisper` with a language fallback chain (your primary language → English). Writes to `inbox/voice/` with frontmatter containing recording date, duration, source file path.
2. **Dedup guard.** iCloud sometimes re-fires on the same file. Hash check on filename+mtime before transcribing.
3. **Language detection fallback.** Multilingual flow: try auto-detect → try your primary language → try English. Save which one succeeded in frontmatter.
4. **Post-transcription handoff.** Tag the note so `/ingest` recognizes voice-memo-origin and applies voice-specific processing (looser grammar forgiveness, extract filler words, identify action items flagged with "remember to / ricordati di").

**What you must do yourself (Apple forces GUI for these):**

- Open Voice Memos on Mac once to trigger iCloud sync.
- Grant Automator "Full Disk Access" (System Settings → Privacy & Security → Full Disk Access → `+` → Automator).
- Verify `~/Library/Group Containers/group.com.apple.VoiceMemos.shared/Recordings/` exists with recent files.
- Accept that Folder Actions only fire when the Mac is awake. Memos recorded while the laptop is closed get processed next time you open it.

**Honest limitations to expect:**

- Folder Actions are historically flaky. A macOS update can reset the permission grant. Budget an annual re-verification.
- `mlx-whisper` first run downloads ~1.6GB model to `~/.cache/huggingface/`. Plan for it.
- Multilingual recordings (code-switching mid-sentence, common for bilingual speakers) occasionally confuse Whisper. Not a dealbreaker, but don't expect 100%.
- No retry mechanism in Automator. If transcription fails, the file just sits there. Add a re-run script that scans for `.m4a` without a sibling `.md`.

**Alternative architectures to consider at design time:**

- **Apple Shortcuts on iPhone** — record → upload to iCloud Drive folder → Mac folder watcher. Less reliable than Voice Memos flow (Shortcuts can be flaky), but gives you more control over where the file lands.
- **Apple Watch complication** — tap wrist, record directly to Voice Memos. If you wear a watch, this is the lowest-friction capture possible.
- **Whisper Memos iOS app** — records directly to a Whisper-processed transcript, emails you the result, you forward-to-inbox. Bypasses the Mac entirely. Paid subscription (~$7/mo) but zero Mac setup.
- **OpenAI Whisper API instead of local mlx-whisper** — better accuracy, especially for multilingual. Costs ~$0.006/min, so negligible. Requires an API key (L1 storage). Send audio out = not local-only anymore.

**Build order when you get to it:**

1. One-off shell script that manually transcribes a single `.m4a` → markdown. Verify it works end-to-end.
2. Wrap in a Folder Action, test with a single new recording.
3. Add dedup + language fallback.
4. Add frontmatter + output path logic.
5. Add a weekly cron that re-runs failed transcriptions.
6. Document in `DECISIONS.md` as a new ADR; write a `RUNBOOK.md` section for when it breaks (because it will).

**When to consider this done.** When you can record a 30-second voice memo on a walk, put your phone back in your pocket, come home 2 hours later, and find a correctly-transcribed markdown note waiting in `inbox/voice/` without having touched the Mac.

### 20.2 Graphify integration — build after first batch of real sources

**When to build:** once `raw/` contains ~20-50 ingested sources and you start asking questions like "what concepts have I been thinking about most?" or "which sources keep getting cross-referenced?". Typically Week 2-3.

**Why deferred:** Graphify produces a knowledge graph from the contents of `raw/`. At 0 sources it's empty. At 5 sources it's trivial. Its value compounds — run it when you have real corpus weight.

**Install:**

```bash
pip install graphifyy   # the official package is spelled with two y's
```

**First run:**

```bash
cd ~/second-brain
graphify ./raw --obsidian --obsidian-dir ~/second-brain/graphify/main
```

This produces `graphify/main/graph.json` (the graph), `graphify/main/report.html` (a visualization), and `graphify/main/notes/` (Obsidian-readable notes per cluster that link back to your wiki).

**Weekly refresh (add to `/weekly-review` skill):**

```bash
graphify ./raw --update --obsidian --obsidian-dir ~/second-brain/graphify/main
```

Only re-extracts changed files, merges into existing graph.

**Gitignore confirmation:** `.gitignore` already excludes `graphify/*/graph.json` and `graphify/*/report.html` (regenerable binaries); `graphify/*/notes/` commits normally.

### 20.3 Memory layer (deferred, see §7.2)

Activate after 4-6 weeks, only if a specific cross-surface state gap emerges. OpenMemory local or Supermemory managed.

### 20.4 RAG-Anything (deferred, see §7.2)

Activate when first complex-PDF pile lands that Claude can't reason over directly.

### 20.5 Mobile Obsidian + cross-device sync (deferred, see §11)

*Note: §A.5 and §11 now recommend Obsidian+ as the Day-1 default for non-technical operators. This subsection remains for the power-user path.*

Pick Syncthing + Möbius Sync (free-ish, private) or Obsidian Sync (paid, turnkey — now the recommended default per §A.5) when the desk-only workflow starts to constrain you.

### 20.6 Claude.ai custom connector to the vault (deferred)

Future: a remote MCP server pointed at the vault so claude.ai (not just Claude Code) can read and write notes. Requires hosting an MCP server. Nice-to-have; Claude Code coverage is sufficient for Day 1.

---

## 21. TL;DR — the decision table

| Need | Use | When |
|---|---|---|
| Single SoT for notes | **Obsidian vault** + daily git commits | Day 1 |
| AI reads + writes + maintains the vault | **Claude Code + CLAUDE.md (Karpathy pattern) + MCP obsidian** | Day 1-2 |
| Multi-step plans that survive sessions | **Superpowers plugin** `/brainstorm` → `/write-plan` → `/execute-plan` | Day 2 |
| Structural "what's connected to what" | **Graphify** (`graphify --obsidian --update`) | Week 2+ |
| Query complex PDFs with figures/tables/equations | **RAG-Anything** (surgical) | Only when needed |
| Zero-friction capture (Day 1) | **Claude Code voice mode** (built-in) + **Obsidian voice plugin** + **Web Clipper** | Day 1-3 |
| Always-on iPhone voice capture | **Custom pipeline (§20.1)** — deferred | Week 3+ (when the pain is real) |
| Cross-device sync (non-technical) | **Obsidian+ (Obsidian Sync)** | Day 1 (human step, §A.5) |
| Cross-device sync (power-user) | **Syncthing** (+ Möbius Sync for mobile) | Day 5-7 |
| Enforce discipline (safety, logging) | **Claude Code hooks (§8)** | Day 2 |
| Recover from compaction | **Hot cache hook + SessionStart injection (§8.5/8.6)** | Day 2 (baked in) |
| Cross-session personal state beyond vault | **OpenMemory (local) or Supermemory (managed)** | Week 5+, only if gap proven |
| Credentials / health / financial | **L1 vault (`~/.second-brain-local/`)** — separate from everything | Day 1 |
| Deep investigation of a topic | **`/write-plan` on the wiki page → `/execute-plan` with Graphify + web** | Anytime |

**The single most important thing.** Spend the first week making capture zero-friction. If you can't get a thought into `inbox/` in under 3 seconds without thinking about where it goes, you will stop capturing, and the vault will die. Everything else is optional. Capture is non-negotiable.

---

*End of spec. Revisit §19 when you've used the system for 30 days. The biggest test of this document is whether the next rewrite feels small, pointed, and ADR-driven — vs. a from-scratch rethink. That's how you know it's doing its job.*
