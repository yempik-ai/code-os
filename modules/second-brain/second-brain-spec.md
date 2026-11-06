# Second Brain — Grounded Implementation Spec

> [!note] Template spec — examples are illustrative
> The worked examples below (domain names, entities, ADR figures) are neutral
> placeholders. Replace them with your own situation when you build your vault.

**Version** 2.4 · **Date** 2026-11-06 · **Status** Implementation-ready · **Source PDF** `second-brain-spec.pdf` (v2.2, 2026-04-19)

**Purpose.** This spec defines a personal knowledge-management system that survives multi-month growth, context compaction, cross-device sync, and context switches across multiple life domains (e.g. employer / startup / freelance / tech-news) — while giving Claude (via Claude Code; claude.ai only once a remote connector exists, §20.6) enough structure to act as a compounding, maintaining librarian rather than a forgetful search tool.

**Audience.** Two readers:

1. **The operator** (the human whose vault this is).
2. **Any Claude instance** you point at the vault. A Claude Code agent reading this should be able to execute §A "Human prerequisites" → §10 "Setup checklist" top-to-bottom without asking clarifying questions.

This version (2.3) is also written so that a **non-technical operator** (e.g. a CEO handing the spec to their own Claude Code on a fresh Mac) can get to a working system. The human-only prerequisites — the things Claude Code cannot click through from a shell — are collected in the new §A so they stand out.

**Non-negotiable outcome.** After applying this spec: (a) deterministic session re-hydration after compaction via `CLAUDE.md` + hot cache; (b) immutable-raw / LLM-owned-wiki / fixed-schema three-layer architecture per Karpathy; (c) commit-per-ingestion discipline with logged provenance (each commit carries its own `wiki/log.md` line); (d) hook- and permission-enforced safety around vault writes; (e) multi-session work that resumes from durable files (`agent-memory/`, plan files), not from conversation memory; (f) cross-device sync without conflict files; (g) every domain (e.g. employer / startup / freelance / tech-news) has a first-class home with MoC, schema, templates; (h) an append-only operation log that lets any future you reconstruct what happened.

**What changed vs v1.0.** Adds: hooks-as-enforcement, formal file formats for every memory file, survival-kit discipline from long-running agentic work, contradiction-at-ingest pattern, hot-cache for compaction recovery, L1/L2 credential split, setup-as-executable-plan, explicit non-goals, and honest accounting of what breaks at scale. Removes: nothing — v1.0 remains correct; v2.0 is additive.

**What changed in v2.1.** Refactored §9.1 to reflect the Day-1 voice decision (Obsidian plugin + MacWhisper free only) and moved the automated iPhone → `inbox/voice/` pipeline to §20.1 as a deferred custom enhancement to build later with full Apple-ecosystem leverage. Added §20 "Deferred enhancements" as an explicit home for everything that's correct-to-build-later rather than never.

**What changed in v2.2.** Explicit Day-1 install scope in §15: ONLY `uvx mcp-obsidian` and vault-local skill markdown files. (The Superpowers plugin, once in this list, is now optional — §7.2.) Everything else (Graphify, RAG-Anything, CLI-Anything, GitNexus, memory layers, additional MCPs) is deferred or doesn't apply. Fixed §10 Phase 6 step 30 which incorrectly told Claude Code to install Graphify at end of setup (Graphify has zero value on an empty vault). Clarified §2 that the "AGENTS.md GitNexus-style block" is terminology only — GitNexus itself is a code-repo tool and doesn't apply to a markdown vault. §20.2 rewritten with concrete Graphify install and first-run commands for when you do come back to it.

**What changed in v2.3.** Added §A "Human prerequisites" — the non-delegable human steps, collected up front so a non-technical operator hands off to Claude Code cleanly. The companion one-pager `DAY-0-CHECKLIST.md` is the copy-paste-ready artifact for a handoff; §A here is the narrative backup. Rewrote §11 "Sync" to recommend **Obsidian Sync via an Obsidian+ (Obsidian Plus) subscription** as the default for non-technical operators: it collapses the entire cross-device story — desktop, laptop, iPhone, iPad — into toggling one setting inside Obsidian. Syncthing remains the power-user / privacy-max option. Clarified in §A and §10 that these sync steps are human-only (billing + device login) and cannot be done by Claude Code. Claude Code is trusted to write the hook scripts (§8) and vault-local skills (§7.2) from their behavioral specifications — source code is deliberately not pasted into this spec. **Removed MacWhisper as a Day-1 install** — Claude Code now ships with a built-in voice mode that covers desk dictation natively, and external meeting audio is already transcribed by the meeting tool itself (§9.3). A transcription helper gets installed ad-hoc by Claude Code only when a specific recording genuinely needs it. §9.1 and ADR-0008 rewritten accordingly. Marked the whole document as a CEO-readable handoff: prose is self-contained, jargon is expanded on first use.

**What changed in v2.4 (field lessons).** Hooks never leave a tracked file modified (§8). `raw/` and L1 are also protected by permission rules, with their limits stated (§8.1–8.2). Secrets never reach git: `inbox/` is gitignored until ingested, `/ingest` starts with a gate, plugin settings that hold keys are gitignored, and the Obsidian API key never enters the conversation or a tracked file (§3, §10, §14.1). Obsidian Git auto-commit is off. Frontmatter contract, link convention, contradiction callouts and the `/lint` check list are made exact (§3, §5). Every template is specified (§5.7) and every ingested page is linked from the daily note. The reconstruction test replaces line-count targets (§5.8). The Superpowers plugin is optional, not Day 1 (§7.2).

---

## A. Human prerequisites — steps Claude Code cannot do for you

Claude Code can write every file, run every shell command, wire every config, and build every hook and skill in this spec from its behavioral description. It **cannot** click through GUI installers, accept macOS permission prompts, log into your Apple ID, or enter a credit card. Those steps are human-only and must be done **before** Claude Code takes over.

**The copy-paste-ready handoff artefact is `DAY-0-CHECKLIST.md`** (companion to this spec). Hand that checklist to the operator alongside this spec and `KNOWLEDGE_MGMT_SETUP_SPEC.md`; it walks through the ~45 min of human work and ends with the literal prompt to paste into Claude Code. This section is the narrative summary — the checklist is the authoritative list.

Budget: **~45 minutes** of human clicking, then **~2-3 hours** supervising Claude Code on Day 1.

### A.1 Accounts (5 min)

Apple ID (already signed in), free GitHub account, Anthropic account (claude.ai, Pro plan is fine), free Obsidian account.

### A.2 Install the base toolchain (15 min)

Homebrew, Python + uv (`brew install python && curl -LsSf https://astral.sh/uv/install.sh | sh` — needed for `uvx mcp-obsidian` and deferred tools like `graphify`), GitHub CLI (`brew install gh && gh auth login`), Claude Code with the native installer (`curl -fsSL https://claude.ai/install.sh | bash`; Windows PowerShell: `irm https://claude.ai/install.ps1 | iex`; the npm package is an alternative and needs Node.js 22+ — [setup](https://code.claude.com/docs/en/setup)), first run of `claude` for Anthropic OAuth. **Voice capture is already handled** by Claude Code's built-in voice mode — no separate transcription app is needed on Day 1 (see §9.1).

### A.3 Install Obsidian + create the vault (5 min)

Download Obsidian from obsidian.md, create a new vault at `~/second-brain` (exact path matters — the spec references it everywhere).

### A.4 macOS permissions (2 min)

Grant **Full Disk Access** to Terminal.app (or iTerm2) in System Settings → Privacy & Security. Allow Obsidian Files & Folders access when prompted.

### A.5 Install Obsidian community plugins (10 min)

Obsidian won't let an external program install plugins — this is GUI-only. In Obsidian: **Settings → Community plugins → turn on → Browse**, then install *and enable* each of:

**Required for vault operation:** Local REST API, Templater, Dataview, Obsidian Git (manual push/pull only — leave automatic commit and automatic push off, see §11), Periodic Notes, QuickAdd, Omnisearch, Advanced URI, Importer.

**Recommended UX:** Calendar, Excalidraw, Kanban, MarkMind, Iconize (Icon Folder), Advanced Tables.

**Optional:** Style Settings.

Then: **Settings → Local REST API → copy the generated API key to your password manager.** You register it yourself in a terminal at §10 step 13; it never goes into the Claude Code conversation or a tracked file. Leave the default port 27124 HTTPS. Keep Obsidian running in the background.

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

Put this spec, `KNOWLEDGE_MGMT_SETUP_SPEC.md`, and `DAY-0-CHECKLIST.md` into the vault at `~/second-brain/handover-birefs/`. Then `cd ~/second-brain`, run `claude --permission-mode manual`, and paste the handoff prompt from Part 10 of `DAY-0-CHECKLIST.md`. That prompt tells Claude Code to read all three files, execute §10 (starting from step 3 since community plugins are already installed), write hooks and skills from their specifications (not copy them from source), and interview you for identity and domains before committing `CLAUDE.md`. At §10 step 13, Claude Code gives you the `claude mcp add` command; run it in a separate terminal with the Local REST API key from A.5. Never paste the key into the chat. Since Claude Code v2.1.283 an interactive session starts in auto mode, where a classifier approves actions instead of you; `--permission-mode manual` makes every action wait for your approval during setup ([permission modes](https://code.claude.com/docs/en/permission-modes)).

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
5. **Commit-per-ingestion.** Every `/ingest` run produces one git commit with a message following a strict format (§3). The same commit carries the run's `wiki/log.md` line (§8.4). If you find yourself without a commit for >24h of active capture, you're stuck — stub it and move on.
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
│   │   ├── block_dangerous_bash.sh                   # PreToolUse(Bash): block destructive commands
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
├── inbox/                                            # unprocessed capture; Claude drains this; gitignored (§14.1)
│
├── wiki/                                             # LLM TERRITORY — the compiled artifact
│   ├── overview.md                                   # top-of-tree synthesis
│   ├── index.md                                      # LLM-maintained catalog
│   ├── log.md                                        # append-only ingestion log (written before each commit)
│   ├── hot.md                                        # recent-context summary (hook-written; gitignored)
│   ├── tag-dictionary.md                             # the closed tag vocabulary: - `tag` — meaning
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
│   ├── TIMELINE.md                                   # macro events: job changes, big life moments
│   └── plans/YYYY-MM-DD-<slug>.md                    # multi-session plans; the resume point
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
├── finances.md                                       # account numbers, tax codes, bank details
├── health-private.md                                 # anything subject to health confidentiality
└── machine-state.md                                  # ephemeral — per-device session pointers
                                                      # passwords, API keys, tokens: password manager only, never here
```

**Why one vault, not one-per-domain.** Notes compound only if they can cross-link. Your day-job work and your startup's tech stack share 60% of the concept graph. Split them and you lose the compounding. `domains/` provides namespacing without fragmentation.

**Why `domains/` sibling to `wiki/`, not under it.** `wiki/` is Claude's synthesized conceptual model (atemporal, entity- and concept-oriented). `domains/` is time- and event-oriented (meetings, deliverables, accounts). They reference each other constantly — but they're different kinds of artifact.

---

## 3. `CLAUDE.md` — the north-star (template)

Claude Code loads the vault-root `CLAUDE.md` at session start and re-reads it from disk after compaction, so it is in context on every turn and every line costs context on every turn ([memory](https://code.claude.com/docs/en/memory), [context window](https://code.claude.com/docs/en/context-window#what-survives-compaction)). The docs advise under 200 lines per file; `@` imports load with it and count toward that. claude.ai cannot reach the local Obsidian MCP — custom connectors must be remote servers (§20.6) — so on claude.ai these rules are not loaded. Keep it as short as the domain rules allow. Rewrite it when the domains change (new employer, new startup, new service line).

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
       wiki-source | synthesis | raw | decision | blocker | moc | schema]
domain: [<employer> | startup-X | freelance | tech-news | personal | global]
tags: [kebab-case-only, every-tag-in-wiki/tag-dictionary.md]
created: YYYY-MM-DD
updated: YYYY-MM-DD
status: [active | hypothesis | archived | contradicted | superseded]
sources: [list of raw/ paths — required for wiki/* notes]
people: ["[[people/person-a]]", "[[people/person-b]]"]
---
- Quote wikilinks in YAML: Obsidian requires it, and unquoted [[x]] parses
  as a nested list.
- status: superseded adds superseded_by: "[[wiki/concepts/new-page]]".

## Links
Wikilinks use the full vault path without extension: [[wiki/concepts/event-sourcing]],
[[people/ada-lovelace]]. A full path never becomes ambiguous when two files share
a name. /lint resolves a bare name only when exactly one page has it, else exactly
one file; pages win over raw files, because a source page and its raw file may
share a basename. A bare name that matches several pages is a dead link.

## Ingestion rules (how Claude processes `raw/` and `inbox/`)
When a new source lands in raw/ (or inbox/):
0. Gate. Before anything else, decide where the source belongs. Passwords,
   keys, tokens: stop and tell me; they belong in my password manager.
   Health, finances, private relationships: stop, leave the file in inbox/
   (gitignored, so it never reaches git) and tell me; I move it to L1 or
   delete it myself. Sensitive parts of a source I still want archived:
   replace them in the inbox/ copy with [removed for privacy: <kind>]
   before step 8. A silent cut makes an incomplete source look complete.
1. Read it fully, in parts if long. If a tool says its result was saved to
   a file, read that file; never compile from the preview. Identify the
   type: article, transcript, voice note, email, clip, conversation.
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
7. If contradiction with existing wiki content: add a dated [!contradiction]
   callout on the relevant page with both sources. Add to wiki/contradictions.md.
8. Move source from inbox/ to raw/<category>/YYYY-MM/ with the move command
   verified at setup (§10 step 15). Never delete from raw/.
9. Link every page you created or updated from today's daily note
   daily/YYYY/YYYY-MM-DD.md, under ## Captured. Create the note from
   _templates/daily.md if it is missing.
10. Check every page you wrote against the reconstruction test (spec §5.8).
11. Append to wiki/log.md: `## [YYYY-MM-DD HH:MM] ingest | <title> → <pages>`,
    before the commit, so the line lands in the same commit.
12. Commit with message: `ingest(<domain>): <title> — N pages (+M new)`,
    with wiki/log.md staged. Then git status: the tree must be clean.

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
Graph hygiene:
- Orphans: wiki pages with <2 incoming backlinks (meta pages exempt).
- Dead links: [[wikilinks]] that resolve to no file (rule in ## Links),
  including a bare name that matches several pages.
- Missing MoC anchor: a page with a non-global domain that does not link
  [[domains/<domain>/_moc]].
- Under-density: below its type's link floor (concepts 3, entities 2,
  sources 1 into concepts or entities, syntheses 3, questions 1).
- Unindexed: a page in wiki/concepts|entities|sources|syntheses|questions/
  not listed in wiki/index.md.
- Broken round-trip: A links B and B does not link A (hub pages exempt).
- Unowned action items: an open - [ ] item without a [[people/…]] link.
- Unknown tags: a tag missing from wiki/tag-dictionary.md.
Content quality:
- Stale: pages with `updated:` older than 180 days and no source added since.
- Sourceless: wiki/* pages without `sources:` frontmatter (exempt:
  status: hypothesis).
- Contradictions: a [!contradiction] callout without a YYYY-MM-DD date in its
  title, or on a page not linked from wiki/contradictions.md. Debt: open
  callouts older than 30 days.
- Schema drift: frontmatter fields that don't match the contract above.
- Unprocessed: files in inbox/ older than 7 days.

## Commit message convention
`<type>(<scope>): <subject>`
types: ingest | crystallize | lint | refactor | docs | chore | fix | feat | WIP
scope: domain name, or `wiki`, or `meta`, or `hooks`, or `skills`, or `templates`
Examples:
- ingest(<employer>): Q1 board prep transcript — 4 pages (+1 new)
- crystallize(startup-<codename>): pricing strategy discussion — overview.md + 2 entities
- lint(wiki): resolved 12 orphans, 3 dead links

## Operating rules — what NOT to do
- NEVER edit raw/ after capture.
- NEVER silently overwrite a wiki page that contradicts a new source. Use the
  [!contradiction] callout.
- NEVER let secrets or L1 material enter raw/, wiki/ or git. Passwords, API
  keys and tokens live in my password manager, not in the vault and not in
  L1. Health records and financial account details go to
  ~/.second-brain-local/ (§6). If you find any of these in inbox/, leave the
  file there and tell me; I remove it myself.
- NEVER let two agents write the same hub. wiki/index.md, wiki/overview.md,
  wiki/contradictions.md, the _moc.md files and agent-memory/* have one
  writer per session: the main agent. Subagents write leaf pages and report
  back; only the main agent assigns ADR numbers.
- NEVER report work as done from memory. Run git status and read back what
  you wrote: the disk is the truth, not an agent's report.
- NEVER create a wiki page without at least one source (except hypothesis pages,
  which must have status: hypothesis).
- NEVER commit an ingestion without its wiki/log.md line in the same commit.
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
### Pricing-page redesign — which layout ships
- Need conversion numbers from the A/B test (ongoing)
- Decision pending: two-tier or three-tier layout
- Blocked on: design review slot

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
- YYYY-MM — Started first freelance engagement (<project>, <amount>)
- YYYY-MM — Joined <employer> as <role> (<contract type>)
- YYYY-MM — Co-founder conversations with <prospect> begin
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
sources: [raw/articles/2026-03/2026-03-03-karpathy-llm-wiki.md, raw/...]
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
- Link to [[wiki/questions/...]] pages where applicable.

## Related
- [[wiki/concepts/...]] — how this relates
- [[wiki/entities/...]] — who's doing this

## Provenance
- [[wiki/sources/YYYY-MM-DD-<slug>]] — one-line on what this source contributed.
- [[wiki/sources/...]] — ...

> [!contradiction] YYYY-MM-DD — <topic> (only if applicable)
> Source A says X. Source B says Y. Unresolved.
```

`sources:` lists the `raw/` files; the `## Provenance` section links the source pages that summarise them.

### 5.3 Page template — `wiki/sources/<slug>.md`

```markdown
---
type: wiki-source
domain: global
tags: [llm-wiki, knowledge-management, rag]
created: 2026-04-19
updated: 2026-04-19
status: active
sources: [raw/articles/2026-04/2026-04-03-karpathy-llm-wiki.md]
source_url: https://gist.github.com/karpathy/...
author: Andrej Karpathy
published: 2026-04-03
---

# <Source title>

> One-sentence summary of the source.

## What the source argues
2-4 sentences on the source's main claim.

## Key extracted claims
- **Claim 1.** The claim in Claude's words → backlink: [[wiki/concepts/<concept>]]
- **Claim 2.** ...

## Novel to my vault
What this source added that existing wiki pages didn't have.

## Contradicts
- [[wiki/concepts/<other>]] — describe the tension. (If applicable.)

## Related
- [[wiki/concepts/...]]
- [[wiki/entities/...]]
```

A source page carries the full frontmatter contract like any wiki page; its `sources:` is the one raw file it summarises. Name the page and the raw file with the same `YYYY-MM-DD-<slug>` so the pair is obvious; full-path links keep them distinct.

### 5.4 The three meta-files

**`wiki/overview.md`** — Claude's top-of-tree synthesis. What's in the vault right now, at a 30,000-foot view. Rewritten (not appended) by `/crystallize` at major milestones. ≤500 lines.

**`wiki/index.md`** — flat catalog of every wiki page with a one-line description. Format:

```markdown
# Wiki Index

## Concepts
- [[wiki/concepts/llm-wiki-pattern]] — compiling sources into a persistent wiki
- [[wiki/concepts/event-sourcing]] — append-only state as a sequence of events
...

## Entities
- [[wiki/entities/anthropic]] — AI company behind Claude
- [[wiki/entities/<employer>]] — my employer
...

## Sources
- [[wiki/sources/2026-04-03-karpathy-llm-wiki]]
...
```

Claude maintains it on every ingestion. Every page in `wiki/concepts|entities|sources|syntheses|questions/` has one line here (the Unindexed lint check depends on it). When the file passes ~400 lines, split it into `wiki/index-<domain>.md` files that `index.md` lists.

**`wiki/log.md`** — append-only, one entry per ingestion, written by the skill before its commit (§8.4). It has no commit SHA: the entry is part of the commit, and `git log` finds it.


```markdown
## [2026-04-19 11:30] ingest | Karpathy LLM Wiki gist
Source: raw/articles/2026-04/karpathy-llm-wiki.md
Pages created: wiki/concepts/llm-wiki-pattern.md, wiki/sources/2026-04-karpathy-llm-wiki.md
Pages updated: wiki/overview.md, wiki/index.md

## [2026-04-19 14:12] ingest | Board meeting transcript Apr 17
Source: raw/transcripts/2026-04-17-board.md
Pages created: domains/<employer>/meetings/2026/2026-04-17-board.md
Pages updated: domains/<employer>/_moc.md, people/ceo-name.md, domains/<employer>/decisions/...
```

### 5.5 The hot cache — `wiki/hot.md`

Life-changing pattern from `AgriciDaniel/claude-obsidian`. Refreshed by the Stop hook each time Claude finishes a response, read at every session start. ≤200 lines.

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
> [!contradiction] 2026-04-19 — Source A vs Source B
> **[[wiki/sources/2026-03-10-vendor-claim]]** says Claude Sonnet 4 outperforms Opus 4 on task X.
> **[[wiki/sources/2026-04-12-independent-eval]]** shows the opposite on the same benchmark.
> Status: unresolved. Needs investigation.
```

The date in the callout title is required: `/lint` computes contradiction age from it. To resolve, add a dated resolution line under the callout, change it to `> [!contradiction-resolved]`, and remove its line from `wiki/contradictions.md`.

**Don't:** silently overwrite the claim. **Don't:** pick one and bury the other. **Don't:** delete the contradiction or either claim — only resolve it as above.

`wiki/contradictions.md` is a meta-page that Claude maintains listing every unresolved `[!contradiction]` callout across the vault, one line per page. This is your "gaps in the model" view.

### 5.7 Other templates

§10 step 7 writes all ten files in `_templates/`. Concept and source are §5.2–5.3. The rest, as section skeletons; frontmatter is the §3 contract with the type filled in. Section names, once chosen, stay fixed.

- **daily** (`type: daily`, `domain: global`): `# YYYY-MM-DD`; `## Inbox` (quick capture, drained by `/ingest` — §9.6); `## Open threads` (filled by `/today`); `## Captured` (links to every page `/ingest` and `/crystallize` created or updated today); `## Decisions`; `## Action items` (each with `[[people/<owner>]]`); `## Related`.
- **meeting**: `# YYYY-MM-DD — Title`; attendees as `[[people/…]]`; `## Context`; `## Decisions` (verbatim where possible); `## Action items` (owner and date); `## Details` (numbers, names, reasons); `## Open questions`; `## Related`.
- **wiki-entity**: `# Name`; one-line identification; `## Facts` (each with its source page); `## Related`; `## Provenance`.
- **person**: `# Name`; role and context; `## Appearances` (links to pages that mention them); `## Action items`; `## Related`.
- **client** / **project**: `# Name`; one-line scope; `## Status`; `## Key people`; `## Decisions`; `## Open threads`; `## Related`.
- **decision**: `# YYYY-MM-DD — Decision`; `## Context`; `## Decision`; `## Alternatives` (and why rejected); `## Consequences`; `## Supersedes`.
- **blocker**: `# YYYY-MM-DD — What is blocked`; `## What`; `## Tried`; `## Stub`; `## Follow-up` (mirrors §4.3).

### 5.8 The reconstruction test

The most frequent ingestion defect is a page that is too short. For every page you write, ask: can a future reader rebuild the decision, the alternatives and the reasons from this page alone?

For a meeting or transcript the test requires: decisions in the words used, action items with owner and date, every number, name and reason, and the open questions. Padding fails the test as much as cuts: repetition and invented detail are defects too. Length follows the source, not a line-count target. If context runs out mid-source, commit what is there as `WIP(<scope>):` with a `[!blocker]` callout naming the depth gap (§12.8).

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

- Financial account numbers, tax codes, personal bank details
- Health information (medical records, diagnoses, medications)
- Highly private relationship or personal content
- Machine-specific session pointers (`CLAUDE.local.md` additions)

**Not in L1 either:** passwords, API keys and access tokens. They live in a password manager. L1 is plaintext Markdown that Claude and any local process can read.

**The rule:** *Would the LLM making a mistake here be dangerous or embarrassing?* → L1. *Merely inconvenient?* → L2. *A credential?* → password manager.

**Implementation:**

- `~/second-brain/.gitignore` includes a wildcard that blocks `.local/` inside the vault if you want an alternative placement.
- L1 is opened as a **separate Obsidian vault** only when needed — not via MCP by default.
- Claude reads or writes L1 only when you ask in the current message. The `ask` rules in §8.1 make every L1 access wait for your approval: that is the mechanical form of "only when asked".
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
| **filesystem** *(optional)* | Raw FS access. | Capture pipelines outside the vault. Not for `raw/`: its writes bypass §8.2 unless you add its tools to the hook matcher. Claude Code's built-in file tools already cover the vault. | Separate MCP server |
| **github** *(optional)* | Manage the vault repo remotely. | Cross-device commits, backups. | OAuth |
| **fetch** | Pull URLs on demand. | Web-clipping, citation resolution. | Default MCP server |

**Don't install the web-search MCP** if you already use Claude with search enabled on claude.ai — pick one surface.

For **Obsidian-specific writes via MCP**, the `mcp-obsidian-advanced` fork adds NetworkX graph introspection which is worth it once the vault crosses 200+ pages. At that scale, Obsidian's native Dataview becomes slow and Claude's grep-style search hits context limits.

### 7.2 Skills to install

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

Each skill is a ~50-200 line `SKILL.md` with a frontmatter header (name, description, triggers). Put the critical steps first: after compaction an invoked skill body is re-injected only up to 5,000 tokens, and truncation keeps the start of the file ([context window](https://code.claude.com/docs/en/context-window#what-survives-compaction)). Template and authoring rules: follow `obra/superpowers` writing-skills guide.

**Optional add-ons** (install when you hit the need):

- **Superpowers plugin** (`obra/superpowers`) — optional, not Day 1. Adds `/brainstorm`, `/write-plan`, `/execute-plan` and TDD, debugging and verification skills. The vault does not depend on it: plans that must survive sessions live as files in `agent-memory/plans/`. See [useful-skills.md](https://github.com/yempik-ai/code-os/blob/main/modules/skills-guide/useful-skills.md#7-superpowers) for what it is, how to install it, and the maintainer's note on it.
- **Graphify** (`pip install graphifyy`) — when you want a full knowledge graph over `raw/` that you can visualize and query. Run `/graphify ./raw --obsidian --obsidian-dir ~/second-brain/graphify/main --update` weekly.
- **RAG-Anything** (`pip install raganything`) — when you have ≥20 heavy PDFs (research papers, financial reports, technical docs with equations/figures) and Claude's native reading isn't cutting it.
- **CLI-Anything** — if/when you need Claude to drive a specific app without a CLI. Not relevant to core KM.
- **Memory layer** (OpenMemory for full-local, Supermemory for managed) — only after 4-6 weeks of vault use reveals a specific gap the wiki doesn't fill.

---

## 8. Hooks — the unconditional enforcement layer

The lesson from repo-scoped KM: the LLM is the brain, but **only hooks and permission rules are guaranteed to fire**. Anything you want enforced deterministically — vault writes, sensitive-path protection, hot-cache refresh — must live in a hook or a permission rule.

**Rule: a hook never leaves a tracked file modified.** A hook that rewrites tracked files leaves the tree always dirty and can overwrite your edits. `wiki/hot.md` is a per-machine cache: it is gitignored (§14.1). `wiki/log.md` is written by the skill before its commit, never by a hook (§8.4). A hook never writes under `raw/`. A guard hook answers deny or nothing, never `allow`: a PreToolUse `allow` skips the permission prompt, so a guard that allows everything it does not block removes a safety layer ([hooks](https://code.claude.com/docs/en/hooks)).

### 8.1 `.claude/settings.json` (committed)

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "deny": ["Edit(/raw/**)"],
    "ask": ["Read(~/.second-brain-local/**)", "Edit(~/.second-brain-local/**)"]
  },
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
        "matcher": "Write|Edit|NotebookEdit|mcp__obsidian__.*",
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

**Permission semantics** ([permissions](https://code.claude.com/docs/en/permissions#read-and-edit)):

- `Edit` rules cover every built-in tool that edits files, including creating a file. A `/path` pattern in project settings anchors at the directory where the session starts, so `Edit(/raw/**)` holds only when sessions start at the vault root. Start them there. `~/path` is home-relative; `//path` is absolute.
- Read and Edit rules also cover the Bash file commands Claude Code recognizes (such as `cat`, `sed`, `tee`) and redirection targets. They do not cover a script or program that opens files itself. For OS-level enforcement, use the [sandbox](https://code.claude.com/docs/en/sandboxing).
- Whether `mv inbox/x raw/…` counts as an edit of `raw/` is not documented. §10 step 15 tests it and records the move command that works.
- An `ask` rule makes every L1 access wait for your approval.
- `.git/`, `.claude/` and `.mcp.json` are protected paths: writes are never auto-approved — prompted in `default` and `acceptEdits`, sent to the classifier in `auto`, denied in `dontAsk`, allowed in `bypassPermissions` ([permission modes: protected paths](https://code.claude.com/docs/en/permission-modes)).

### 8.2 `.claude/hooks/block_dangerous_writes.sh` (PreToolUse on Write|Edit|NotebookEdit and Obsidian MCP tools)

Reads tool input JSON from stdin. The matcher includes `mcp__obsidian__.*` because `/ingest` writes through the Obsidian MCP, and those calls never match `Write|Edit` ([hooks: MCP tools](https://code.claude.com/docs/en/hooks)). **Denies**:

- Any `mcp__obsidian__*` write whose target path is under `raw/`, new file or existing (enforces §1.1 immutability). The `Edit(/raw/**)` permission rule covers only the built-in file tools; this hook closes the MCP gap.
- Built-in Write/Edit/NotebookEdit calls under `raw/` (a second layer behind the permission rule).
- Writes to `.git/` internals.

New sources enter `raw/` only through the Bash move from `inbox/` that §10 step 15 verified; no file tool or MCP tool ever writes there.

L1 needs no hook: the `ask` rules in §8.1 already make every L1 read and write wait for you. On deny, emits PreToolUse `permissionDecision: deny` with a helpful reason pointing at the relevant non-negotiable. Otherwise it emits nothing and exits 0, which leaves the normal permission flow in charge. It never emits `allow`.

### 8.3 `.claude/hooks/block_dangerous_bash.sh` (PreToolUse on Bash)

Same pattern as the repo-scoped version: blocks `rm -rf /`, `rm -rf ~`, `git push --force` (unless you explicitly want this on a branch), `sudo rm`, etc. For a personal vault the risk surface is smaller than a production repo, but the hook is cheap and the cost of a mistake is the whole vault.

### 8.4 `wiki/log.md` — written before the commit, no hook

Earlier versions used a PostToolUse hook that appended to `wiki/log.md` after each `ingest(` commit. That left the tree dirty after every commit (the entry describing commit N could only land in commit N+1). It is removed.

Now `/ingest`, `/crystallize` and `/lint` append their `wiki/log.md` entry (§5.4 format) **before** they commit, and stage it with the pages, so each commit carries its own entry and `git status` ends clean. Session close does the same for any pending work. `git log --grep='^ingest('` remains the authoritative history; `/weekly-review` compares it with `wiki/log.md` and reports a commit without an entry. Commits made outside Claude Code (a terminal, an Obsidian Git auto-commit) skip the log — one more reason auto-commit stays off (§11).

### 8.5 `.claude/hooks/hot_cache_on_stop.sh` (Stop)

The Stop event fires each time Claude finishes a response, not only at session end ([hooks](https://code.claude.com/docs/en/hooks)). Each time it fires:

1. Compute diff: which `wiki/` files were touched since the last hot-cache refresh (timestamp stored in `wiki/.hot-cache-timestamp`).
2. Read the last 10 entries from `wiki/log.md`.
3. Read unresolved `[!blocker]` and `[!contradiction]` callouts added in this session.
4. Write `wiki/hot.md` with a structured summary:

```markdown
# Hot Cache (refreshed YYYY-MM-DD HH:MM)
## Active domains this session
- <employer>, startup-<codename>
## Pages updated (last 10)
- [[wiki/concepts/llm-wiki-pattern]] — expanded Karpathy section
- ...
## New open threads
- Blocker: API quota exhausted, ingestion queue at 3 items → BLOCKERS.md
- Contradiction: vendor benchmark vs independent eval → wiki/contradictions.md
## Current state
Mid-ingestion: 2 transcripts in inbox/ pending, ingestion paused on rate limit.
```

5. Touch `wiki/.hot-cache-timestamp`.

Both files are gitignored (§14.1): the hook rewrites them every session, so tracking them would leave the tree always dirty.

This is the **single most important hook** in the system. Without it, compaction recovery degrades to re-reading 20+ files blindly on every new session.

### 8.6 `.claude/hooks/session_start_context.sh` (SessionStart)

On session start, reads `wiki/hot.md` + `agent-memory/OPEN-THREADS.md` + `agent-memory/PREFERENCES.md` and injects them as `additionalContext` JSON. That text goes into Claude's context, not onto your screen ([hooks: SessionStart](https://code.claude.com/docs/en/hooks)). With no matcher it fires on every source, including `compact`, so it re-injects after compaction too. To check it worked, see the acceptance test in §10.

This turns "start of session" from "Claude has no memory" into "Claude picks up where it stopped."

### 8.7 `.claude/hooks/precompact_save.sh` (PreCompact)

A command hook cannot prompt Claude. What it can do before the harness compacts the conversation:

1. Print instructions on stdout. For PreCompact, stdout becomes instructions for the compaction summary ([hooks](https://code.claude.com/docs/en/hooks)) — for example: "Keep the current ingestion step, the files in flight, open decisions and what was learned this session."
2. Run the Stop-hook logic pre-emptively so `hot.md` is current.

It writes no tracked file and nothing under `raw/` (§8 rule). Reasoning worth keeping goes through `/crystallize`, which writes to `inbox/` and runs the ingest gate. Prevents valuable mid-session reasoning from being summarized into oblivion. Pattern from `reliabilitywhisperer.substack.com/p/the-andrej-karpathy-llm-wiki-idea`.

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

**Obsidian Web Clipper** browser extension → saves to `inbox/`, never straight to `raw/`: everything passes the `/ingest` gate, and `/ingest` archives it to `raw/clips/YYYY-MM/`. Set the template so it includes `source_url`, `clipped_at`, and a `tags:` field prompt.

### 9.3 Meeting transcripts

Most meeting transcription tools (Otter, Fathom, Granola, Tactiq, Zoom native, Google Meet native) can export `.md` or `.txt`. Pipe via:

- Direct folder-sync to `inbox/` (`/ingest` archives to `raw/transcripts/` after the gate), or
- A cron that pulls from the tool's API daily, or
- Manual drag-and-drop for the ones that matter.

For calls you run yourself: `silverstein/minutes` + a system-audio capture can run entirely local.

### 9.4 File captures (PDFs, DOCX, slides)

Drag to `inbox/`; `/ingest` archives them to `raw/papers/` or `raw/articles/`. Claude's native reading handles most; read PDFs in page ranges. For the hard ones (heavy tables, equations, figures-with-meaning), invoke RAG-Anything — but only when actually needed, not by default.

### 9.5 Email / Slack / WhatsApp

The hard ones. Options:

- **Manual triage** — copy/paste important threads into `inbox/` (archived to `raw/messages/`). Scales poorly.
- **MCP servers for email/Slack** — install the relevant MCP; Claude pulls threads on demand. Better.
- **Periodic export scripts** — weekly cron that dumps Gmail's "knowledge-worthy" label into `inbox/`. Best for high-volume email users.

Start with the first, graduate to the second when it becomes a bottleneck.

### 9.6 Quick capture (inline thoughts, drafts)

Obsidian's daily note has an "Inbox" section at the top. You write stream-of-consciousness there during the day. `/today` at morning and `/ingest inbox/` in the evening drain it.

---

## 10. Setup checklist — executable, in order

Execute in order. Commit after each step with a meaningful message.

> **Prerequisite:** §A "Human prerequisites" must be complete before starting Phase 1. Specifically: Obsidian, Claude Code, and Homebrew are installed; macOS permissions granted; GitHub auth done; sync strategy chosen (usually Obsidian+ per §A.5).

### Phase 1 — Foundation (Day 1, ~2-3 hours)

1. **Install Obsidian.** Create vault at `~/second-brain`. Disable unnecessary default plugins (Templates — we use Templater; Backlinks — we use graph view). Initial commit: `chore: init vault`.
2. **Install community plugins:** Local REST API, Templater, Dataview, Obsidian Git (manual push/pull; automatic commit and push off — §11), Periodic Notes, Advanced URI, Omnisearch, QuickAdd. Configure Local REST API: copy API key to password manager, pick port 27124 HTTPS. Check that `.gitignore` (§14.1) covers `.obsidian/plugins/*/data.json` before this commit: that file holds the API key and the TLS private key. Commit: `chore: install community plugins`.
3. **Create the full folder structure from §2.** Empty folders with `.gitkeep` files. Commit: `chore: scaffold folder structure`.
4. **Write `CLAUDE.md`** from §3 template. Fill in your actual domains (employer, startup codename, freelance, tech-news, personal). Write non-negotiables verbatim. Write frontmatter contract verbatim. Customize ingestion rules. Commit: `docs(meta): write CLAUDE.md`.
5. **Write `AGENTS.md`** as a pointer-to-CLAUDE.md stub (one paragraph + a copy of the non-negotiables). Commit: `docs(meta): write AGENTS.md`.
6. **Write `README.md`** — what this vault is, for you / for future-you / for non-technical humans. Commit: `docs: write README`.
7. **Create the ten `_templates/*.md`** from §5.2, §5.3 and §5.7. The §3 frontmatter contract + section skeletons. Also write `wiki/tag-dictionary.md` with one tag per domain. Commit: `docs(templates): scaffold templates`.
8. **Seed `agent-memory/`** — write all six files with headers per §4, empty bodies. Write your first `PREFERENCES.md` from §4.5 template. Commit: `docs(memory): seed agent-memory`.
9. **Configure `.gitignore`** (§14.1) and `.gitattributes`. Run `git status --ignored` and confirm `inbox/` contents, `wiki/hot.md` and `.obsidian/plugins/*/data.json` are ignored. Commit: `chore: configure git`.
10. **Initialize git remote** (GitHub private repo is the canonical choice; Gitea self-hosted if you want full sovereignty). Push. Commit nothing — push only.

### Phase 2 — Claude Code integration (Day 1-2, ~2 hours)

11. **Install Claude Code** with the native installer: `curl -fsSL https://claude.ai/install.sh | bash` (macOS, Linux, WSL) or `irm https://claude.ai/install.ps1 | iex` (Windows PowerShell). The npm package is an alternative and needs Node.js 22+ ([setup](https://code.claude.com/docs/en/setup)). Verify with `claude --version`. For the setup session, start Claude Code with `claude --permission-mode manual`: since v2.1.283 an interactive session starts in auto mode, where a classifier approves actions instead of you ([permission modes](https://code.claude.com/docs/en/permission-modes)).
12. **No plugins on Day 1.** The Superpowers plugin is optional (§7.2); install it later only if you want it.
13. **Register the Obsidian MCP — the operator runs this, not Claude.** The API key must never enter the conversation or a tracked file. Claude prints the command; the operator runs it in a separate terminal with the key from the password manager:
    ```bash
    read -rs OBSIDIAN_API_KEY   # paste the key; nothing is echoed or saved to shell history
    claude mcp add --scope local \
      --env OBSIDIAN_API_KEY="$OBSIDIAN_API_KEY" --env OBSIDIAN_HOST=127.0.0.1 --env OBSIDIAN_PORT=27124 \
      --transport stdio obsidian -- uvx mcp-obsidian
    unset OBSIDIAN_API_KEY
    ```
    Local scope is stored in `~/.claude.json`, not in the vault; a project-scope `.mcp.json` is meant for version control ([MCP scopes](https://code.claude.com/docs/en/mcp)). `~/.claude.json` holds the key **in plain text**: keep that file private and out of any sync or backup you share. Keep another option between the last `--env` and the server name, or the CLI reads the name as one more `KEY=value`. Preferred where your password manager has a CLI: write a project `.mcp.json` with `"OBSIDIAN_API_KEY": "${OBSIDIAN_API_KEY}"` and export the variable from the password manager before you start `claude`. Claude Code expands `${VAR}` in `.mcp.json`, so no file holds the key. Restart Claude Code. Verify: `/mcp` should list `obsidian` with connected status. Ask Claude "list files in my vault"; it should succeed.
14. **Write the five hook scripts** from their §8 specifications (no source is pasted in this spec): `block_dangerous_writes.sh`, `block_dangerous_bash.sh`, `hot_cache_on_stop.sh`, `session_start_context.sh`, `precompact_save.sh`. Put them in `.claude/hooks/`, `chmod +x`. Show the operator each script before its first run. Write `.claude/settings.json` per §8.1 (hooks and permission rules). Commit: `feat(hooks): install hook pipeline`.
15. **Verify hooks and permissions.** Start a new session at the vault root first (`claude --permission-mode manual`), so the settings load. Then verify and record each result:
    1. Append a test entry to `wiki/log.md`, stage it, and commit `ingest(meta): log test`. `git show --stat HEAD` lists `wiki/log.md`; `git status` is clean.
    2. Writing a new file under `raw/` is denied both with the built-in file tools (permission rule + hook) and with an Obsidian MCP write tool (hook).
    3. Create `inbox/permission-test.md`, then move it to `raw/_archive/permission-test.md` with `mv`. If the move is denied, change the `raw/` rule from `deny` to `ask`, so each archive move asks once. Record the command that works in `CLAUDE.md` (ingestion step 8) and as an ADR. The operator deletes the test file.
    4. Reading a file under `~/.second-brain-local/` asks for approval.
    5. Before reading any file, the new session states the `## Current state` line of `wiki/hot.md` when asked. A correct answer proves the SessionStart hook injected it.

    If any check fails: debug; do not continue until it passes.
16. **Write the eight vault-local skills** (§7.2). Each ~50-200 lines, following the `SKILL.md` frontmatter format. This is the big work of setup — budget 2-3 hours. Commit after each skill: `feat(skills): add /<skillname>`.

### Phase 3 — First ingestions (Day 2-3)

17. **Seed each domain's `_moc.md` and `_schema.md`.** Write 1-paragraph map-of-content per domain, with links to the top 3-5 most important wiki pages or meeting notes that already exist in your head. The schema can be empty at first — you'll grow it.
18. **Ingest your first 10 sources.** Drop things you've been meaning to file into `inbox/`. Run `/ingest` one at a time. Watch what Claude does. Correct the schema when it misfiles something. Commit after each: `ingest(<domain>): <title>`.
19. **Run the first `/lint`.** Expect orphans (new pages with no incoming links yet — that's fine for the first 10 sources). Expect dead links (you referenced pages that don't exist yet — create them or remove the link). Do not `--autofix` yet; walk through manually to learn Claude's style.
20. **Write your first `/today`.** See if it handles the capture → daily note → backlinks flow correctly. Adjust the skill if not.

### Phase 4 — Capture pipeline (Day 3-5)

21. **Pick ONE voice-capture option from §9.1.** Set it up. Test end-to-end: voice → transcript in `inbox/voice/` → `/ingest` → wiki page + daily note.
22. **Install the Obsidian Web Clipper browser extension.** Set its save location to `inbox/`. Clip one article. Verify it lands in `inbox/`, then `/ingest` moves it to `raw/clips/`.
23. **Wire up one meeting transcription source.** Pick your highest-volume meeting tool; set up export to `inbox/`.
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
- Every `ingest(` commit in `git log` includes its own `wiki/log.md` entry (`git show --stat`).
- In a new Claude Code session, before it reads any file, Claude states the `## Current state` line of `wiki/hot.md` when asked. A correct answer proves the SessionStart hook injected it (hook output goes into Claude's context, not onto the screen). `/context all` lists `CLAUDE.md` under Memory files ([memory](https://code.claude.com/docs/en/memory)).
- The permission results of §10 step 15 are recorded, and the working move command is in `CLAUDE.md`.
- `git status` is clean after a session ends: no hook left a tracked file modified.
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
| **GitHub push-pull** | Manual push/pull (`obsidian-git` plugin, auto-commit off). Clean history. Free. Does not cover mobile. | Desktop-heavy workflow where you also want full versioning you can browse in a git UI. Pair with another strategy for mobile. |
| **iCloud Drive** | **Avoid.** Metadata files, conflict files, unpredictable sync timing, breaks `.obsidian/workspace.json`. | Nope. |
| **Dropbox** | **Avoid.** Same problems as iCloud, slightly less bad. | Nope. |
| **OneDrive** | **Avoid.** Worst of both. | Nope. |
| **Obsidian LiveSync (CouchDB plugin)** | Real-time multi-device, end-to-end encrypted, self-hostable. Requires running a CouchDB server yourself. | Power users who want Google-Docs-like real-time sync and are willing to run infra. |

**Recommended stacks:**

- **Non-technical operator (default):** Obsidian+ subscription → Obsidian Sync for every device. Done. Pair with `git push` at session close as a belt-and-braces backup to GitHub. *This is what a CEO-handoff install should choose unless there's a specific reason not to.*
- **Power user / privacy-max:** Syncthing for desktop↔desktop, Möbius Sync (paid iOS app that speaks Syncthing protocol) for mobile, `git push` for versioned backup.
- **Desktop-only:** push to GitHub at session close. Skip mobile until you need it.

**Obsidian Git: never automatic commits.** Claude commits once per ingestion (non-negotiable 5). A timed auto-commit splits units of work, races Claude's commits, skips the `wiki/log.md` entry (§8.4), and commits whatever sits in the tree. Use the plugin for manual push/pull only, or not at all. Let only one computer run git commits.

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
- Web clips → `inbox/` (one click).
- Quick thoughts → append to today's daily note Inbox section (fastest path).
- Important messages/emails → forward/copy to `inbox/`.

### 12.3 End of day (10-15 min)

1. `/ingest inbox/` — Claude drains what you captured today.
2. Scan what Claude wrote. Correct anything wrong. Resolve any `[!contradiction]` or `[!blocker]` callouts that are quick.
3. Close the session: commit what is pending (including `wiki/log.md`), push if a remote exists, check `git status` is clean. The Stop hook keeps `wiki/hot.md` current after every response (gitignored).

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

The harness compacts when context gets long. The PreCompact hook fires and steers the compaction summary; the Stop-hook logic refreshes `hot.md`. Claude Code re-reads the vault-root `CLAUDE.md` from disk ([context window](https://code.claude.com/docs/en/context-window#what-survives-compaction)). When the compacted session continues:

1. The SessionStart hook (source `compact`) re-injects `wiki/hot.md`, `OPEN-THREADS.md` and `PREFERENCES.md`.
2. If mid-ingestion: `git status` shows the partial work; `wiki/log.md`'s last entry shows the last completed source.
3. If mid-plan: the plan file in `agent-memory/plans/` is the resume point.
4. If the files and git disagree, trust git (`git log -10 --oneline`, `git status`) and correct the files.

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
| Credentials accidentally committed to the repo | Password manager for credentials (§6) + `inbox/` gitignored until the `/ingest` gate + `.obsidian/plugins/*/data.json` gitignored + MCP key registered outside the vault (§10 step 13) + secret-scanner pre-commit hook (optional but recommended). If one was pushed: rotate it first; removing it from history is a destructive action the operator decides. |
| The tree is always dirty | A hook writes a tracked file. Fix the hook (§8 rule), gitignore its output, or commit it with the next unit of work. |
| Graph becomes unreadable past ~5k nodes | Per-domain graph filters in Obsidian. Archive old daily notes. Move cold content to a sibling vault. |
| Ingestion eats tokens on recurring topics | Hot cache means recurring topics resolve from 200-token summary, not full reads. Graphify for structural queries without full-text reads. |
| Lint finds hundreds of issues after weeks of neglect | Don't `--autofix` blindly. Walk through manually for the first 2-3 lint passes; once you trust Claude's style, autofix becomes safe. |
| An ingestion commit has no log entry | `/weekly-review` compares `git log --grep='^ingest('` with `wiki/log.md` and lists commits without an entry; add the missing lines in a `docs(wiki):` commit. |
| Vault grows past comfort zone (~20k notes) | Archive strategy: cold content → sibling vault. Per-domain query scoping in skills. |
| I don't trust Claude with some personal content | L1 vault. Separate MCP scope. Obsidian multi-vault support. |
| A raw source proves wrong months later | Move to `raw/_archive/`, update the cited wiki pages with the new source, add an ADR explaining the change. Never delete raw. |
| I get sick / burned out and neglect the vault for a month | It's fine. The wiki doesn't rot in storage — only capture stops. Resume by ingesting whatever accumulated. Write a `PROGRESS.md` entry noting the gap. |

---

## 14. Concrete artefacts — paste-ready templates

### 14.1 `.gitignore`

```gitignore
# Captures stay out of git until /ingest archives them (the gate, §3)
inbox/*
!inbox/.gitkeep

# Obsidian workspace (machine-specific)
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.obsidian/cache
.trash/

# Plugin settings can hold secrets (Local REST API: apiKey and TLS privateKey)
.obsidian/plugins/*/data.json

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

# Hot cache and its timestamp (hook-written every session; per-machine)
wiki/hot.md
wiki/.hot-cache-timestamp

# Secrets (belt-and-braces; should never be here, but just in case)
*.key
*.pem
.env
```

The `.obsidian/plugins/*/data.json` line is not optional: the Local REST API plugin stores `apiKey` and the TLS `privateKey` there ([plugin source](https://github.com/coddingtonbear/obsidian-local-rest-api/blob/main/src/types.ts)). If a key was already pushed, regenerate it in the plugin settings. Obsidian Sync, not git, carries plugin settings between devices.

### 14.2 Example `domains/<domain>/_moc.md`

```markdown
---
type: moc
domain: <employer>
tags: [moc, <employer>]
created: 2026-04-19
updated: 2026-04-19
status: active
---

# <Employer> — Map of Content

## What
Your employer since <start date>. Role: <role>. <Contract type>.
Reports to <manager roles> directly. Responsible for <your remit>.

## Key people
- [[people/<ceo-name>]] — CEO
- [[people/<manager-name>]] — your direct manager
- [[people/<other>]] — ...

## Active workstreams
- [[domains/<employer>/<workstream>/pricing-page]] — pricing-page redesign
- [[domains/<employer>/<workstream>/onboarding]] — onboarding flow
- [[domains/<employer>/<workstream>/...]] — ...

## Recent decisions
- [[domains/<employer>/decisions/2026-03-10-pricing-layout]]
- ...

## Open threads (see also OPEN-THREADS.md § <employer>)
- Pricing-page layout: two-tier or three-tier
- Design review slot for the redesign
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
tags: [schema, <employer>]
created: 2026-04-19
updated: 2026-04-19
status: active
---

# <Employer> — Domain Schema

## Entity types in this domain
- **account** — internal or external project (e.g., <project-name>)
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
Critical steps first: after compaction only the first 5,000 tokens of this
file are re-injected.

In brief:
1. Enumerate unprocessed files (inbox/ or the specified path)
2. For each file, in order:
   a. Gate: secrets → stop and tell me; L1 material → leave it in inbox/
      and tell me; sensitive parts I still want archived → replace with
      [removed for privacy: <kind>] in the inbox/ copy
   b. Read fully, in parts if long. If a tool saved its result to a file,
      read the file, never the preview
   c. Identify type (article, transcript, voice, clip, message, conversation)
   d. Find 3-10 existing wiki pages it touches (use MCP obsidian search +
      reading wiki/index.md)
   e. If type is meeting transcript: also file under domains/<domain>/meetings/
      and extract decisions + action items
   f. If type is voice note: extract TODOs, file to right project
   g. Create or update wiki pages per the templates in _templates/
   h. If new source contradicts existing wiki content: add a dated
      [!contradiction] callout on affected page + entry in wiki/contradictions.md
   i. Move source from inbox/ to raw/<category>/YYYY-MM/ with the move
      command recorded in CLAUDE.md
   j. Link every page created or updated from today's daily note
      (## Captured); create the note from _templates/daily.md if missing
   k. Apply the reconstruction test (spec §5.8) to every page written
   l. Append the wiki/log.md entry for this source (before the commit)
3. After all sources processed, commit with message:
   `ingest(<domain>): <combined summary> — N pages (+M new)`, with
   wiki/log.md staged in the same commit. git status must end clean.

## What NOT to do
- Do not edit raw/ files after moving them (non-negotiable #1)
- Do not silently overwrite contradictory claims (non-negotiable #4)
- Do not create wiki pages without at least one source
- Do not process more than 10 sources in one commit — split them

## Tools needed
- obsidian MCP (read/write vault)
- Bash (for the inbox/ → raw/ move and git commit)

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

- `uvx mcp-obsidian` (via `claude mcp add`, run by the operator — §10 step 13) — the MCP bridge to the vault
- Vault-local skills written as markdown files in `.claude/skills/` — these are just files, no package install

**Do NOT install during setup (defer per §20):**

- Superpowers plugin — optional, not Day 1 (§7.2); the vault's plans live in `agent-memory/plans/`
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

- `<key-workstream>/` — e.g. a pricing-page redesign, its experiments and decisions. This is your load-bearing work.
- `deliverables/<flagship-deliverable>/` — the actual artifact you're shipping
- `proposals/` — the pitches you keep iterating for management

**Domain schema highlights (`_schema.md`):**

- Meetings tagged by attendee group: `#board`, `#manager`, `#peer`, `#team`
- Decisions have a `forum:` field (manager 1:1, team meeting, all-hands, etc.)
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
- **ADR-0006 — L1 separate vault for health and financial details; credentials in a password manager.** Context: vault is git-tracked and cloud-synced; some data is dangerous/embarrassing to leak; L1 is plaintext that Claude and any local process can read, so it is no place for passwords or keys either. Consequences: two-vault workflow; MCP only connects to L2 by default; every L1 access asks for approval (§8.1 `ask` rules).
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

During setup this file is read from `~/second-brain/handover-birefs/second-brain-spec.md` (§A.9). When setup is done, keep that copy as the vault's spec: it is the file `CLAUDE.md` and the skills point to. It's subject to its own rules:

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

Future: a remote MCP server pointed at the vault so claude.ai (not just Claude Code) can read and write notes. Requires hosting an MCP server reachable from the public internet: claude.ai custom connectors cannot use a local stdio server such as `mcp-obsidian` ([custom connectors](https://support.claude.com/en/articles/11175166-getting-started-with-custom-connectors-using-remote-mcp)). Nice-to-have; Claude Code coverage is sufficient for Day 1.

---

## 21. TL;DR — the decision table

| Need | Use | When |
|---|---|---|
| Single SoT for notes | **Obsidian vault** + daily git commits | Day 1 |
| AI reads + writes + maintains the vault | **Claude Code + CLAUDE.md (Karpathy pattern) + MCP obsidian** | Day 1-2 |
| Multi-step plans that survive sessions | **Plan files in `agent-memory/plans/`** (Superpowers plugin optional, §7.2) | Day 1 |
| Structural "what's connected to what" | **Graphify** (`graphify --obsidian --update`) | Week 2+ |
| Query complex PDFs with figures/tables/equations | **RAG-Anything** (surgical) | Only when needed |
| Zero-friction capture (Day 1) | **Claude Code voice mode** (built-in) + **Obsidian voice plugin** + **Web Clipper** | Day 1-3 |
| Always-on iPhone voice capture | **Custom pipeline (§20.1)** — deferred | Week 3+ (when the pain is real) |
| Cross-device sync (non-technical) | **Obsidian+ (Obsidian Sync)** | Day 1 (human step, §A.5) |
| Cross-device sync (power-user) | **Syncthing** (+ Möbius Sync for mobile) | Day 5-7 |
| Enforce discipline (safety, logging) | **Claude Code hooks + permission rules (§8)** | Day 2 |
| Recover from compaction | **Hot cache hook + SessionStart injection (§8.5/8.6)** | Day 2 (baked in) |
| Cross-session personal state beyond vault | **OpenMemory (local) or Supermemory (managed)** | Week 5+, only if gap proven |
| Health / financial | **L1 vault (`~/.second-brain-local/`)** — separate from everything | Day 1 |
| Passwords / API keys / tokens | **Password manager** — never the vault, never L1 | Day 1 |
| Deep investigation of a topic | **A plan file for the wiki page in `agent-memory/plans/`, executed step by step with Graphify + web** | Anytime |

**The single most important thing.** Spend the first week making capture zero-friction. If you can't get a thought into `inbox/` in under 3 seconds without thinking about where it goes, you will stop capturing, and the vault will die. Everything else is optional. Capture is non-negotiable.

---

*End of spec. Revisit §19 when you've used the system for 30 days. The biggest test of this document is whether the next rewrite feels small, pointed, and ADR-driven — vs. a from-scratch rethink. That's how you know it's doing its job.*
