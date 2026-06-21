# Day-0 Checklist — Second Brain setup

**Who this is for.** You, the operator. These are the steps a human body has to do — logging into accounts, clicking through installers, granting permissions, entering a credit card, clicking a GUI toggle inside an app Claude Code can't see. Everything else — folder structure, hook scripts, vault-local skills, CLAUDE.md, ADRs, templates, git init, graphify later — Claude Code does once you hand off.

**Budget.** ~45-60 min of clicking + waiting, then ~1-2 hours of supervising Claude Code (mostly idle — do other work).

**What you'll need.**

- A Mac (Apple Silicon preferred).
- A browser.
- A credit card for Obsidian+ (~$10/mo) if you take the recommended sync path.
- Your Apple ID password.

**What you're handing to Claude Code.**

- `second-brain-spec.md` — the primary vault spec (architecture + 30-step setup checklist).
- `KNOWLEDGE_MGMT_SETUP_SPEC.md` — the generic agent-memory discipline reference.
- This checklist — so Claude Code knows what you've already done.

---

## Part 1 — Accounts (5 min)

- [ ] You're signed into your Mac with your Apple ID.
- [ ] Create a free **GitHub** account at [github.com](https://github.com/) if you don't have one.
- [ ] Create an **Anthropic** account at [claude.ai](https://claude.ai/). Pick a plan — Pro is fine to start; Max if you expect heavy use.
- [ ] Create a free **Obsidian** account at [obsidian.md](https://obsidian.md/) (needed for Part 5 sync).

## Part 2 — Install the base toolchain (15 min)

Open **Terminal.app** (Applications → Utilities → Terminal) and run these in order.

- [ ] **Homebrew** — paste the one-liner from [brew.sh](https://brew.sh/). Wait ~5 min.
- [ ] **Node.js** — `brew install node` (Claude Code is an npm package).
- [ ] **Python + uv** — `brew install python && curl -LsSf https://astral.sh/uv/install.sh | sh` (needed for `uvx mcp-obsidian` and for optional tools like `graphify` later).
- [ ] **GitHub CLI** — `brew install gh && gh auth login` → GitHub.com → HTTPS → Yes, authenticate via browser.
- [ ] **Claude Code** — `npm install -g @anthropic-ai/claude-code`, then run `claude` once. A browser opens for Anthropic login. Sign in. Verify: `claude --version`.

## Part 3 — Install Obsidian + create the vault (5 min)

- [ ] **Download Obsidian** from [obsidian.md/download](https://obsidian.md/download) → drag to Applications → open it.
- [ ] **Create a new vault at `~/second-brain`.** In the Obsidian opening dialog, pick "Create new vault", name it `second-brain`, set location to your home folder. **This exact path matters** — the spec references it everywhere.
- [ ] If Obsidian asks for Files & Folders access, click **Allow**.

## Part 4 — macOS permissions (2 min)

- [ ] **System Settings → Privacy & Security → Full Disk Access** → `+` → add **Terminal.app** (or iTerm2 if that's what you use). Lets Claude Code work without a permission prompt on every single file.

## Part 5 — Install Obsidian community plugins (10 min)

This is the painful-but-unavoidable GUI part — Obsidian won't let an external program install plugins for you. Open Obsidian, go to **Settings → Community plugins → "Turn on community plugins"** (first-time only), then **Browse** and install each of the following. After each install, also click **Enable**.

**Required (Claude Code integration + vault operation):**

- [ ] **Local REST API** — *this is the one Claude Code talks to the vault through.* Must be installed, enabled, and configured (see Part 6).
- [ ] **Templater** — templating engine the vault's `_templates/` rely on.
- [ ] **Dataview** — query engine for dashboards (e.g. freelance CRM view).
- [ ] **Obsidian Git** — auto-commits, belt-and-braces backup layer.
- [ ] **Periodic Notes** — daily/weekly/monthly note scaffolding.
- [ ] **QuickAdd** — fast capture shortcuts into `inbox/`.
- [ ] **Omnisearch** — full-text search that beats grep at scale.
- [ ] **Advanced URI** — deep-linking so Claude Code can open specific notes.
- [ ] **Importer** — one-off imports from Notion/Roam/Evernote if you're migrating.

**Recommended (visual thinking + UX):**

- [ ] **Calendar** — sidebar calendar, pairs with Periodic Notes.
- [ ] **Excalidraw** — hand-drawn diagrams inline in notes.
- [ ] **Kanban** — board views for any project.
- [ ] **MarkMind** — mind-mapping plugin.
- [ ] **Iconize** (a.k.a. Icon Folder) — folder/note icons for visual wayfinding.
- [ ] **Advanced Tables** — table editor (much better than default).

**Optional styling:**

- [ ] **Style Settings** — lets themes expose config knobs; install if you want to customize appearance later.

*(If any of these aren't in your "Browse" list, search by the exact name above. Obsidian sometimes renames plugins between versions.)*

## Part 6 — Configure Local REST API (2 min)

- [ ] In Obsidian: **Settings → Local REST API → copy the generated API key** into your password manager. Claude Code will ask for this.
- [ ] Leave the default port (**27124 HTTPS**).
- [ ] Keep Obsidian running in the background — Claude Code talks to it via this plugin.

## Part 7 — Install browser helpers (3 min)

- [ ] **Obsidian Web Clipper** — install the browser extension for Chrome/Safari/Firefox from the respective extension store. One-click web-page capture into the vault.

> **Voice capture: nothing to install.** Claude Code ships with built-in voice mode — press the mic button / platform shortcut inside a `claude` session and dictate. No separate app needed. External meeting audio is handled by whatever tool recorded the meeting (Otter, Fathom, Granola, Zoom native — they export transcripts; drop the `.md` into `inbox/`).

## Part 8 — GitHub remote (3 min)

- [ ] Go to [github.com/new](https://github.com/new). Create a **private** repo called `second-brain` (or any name). **Do NOT initialize with a README** — leave it empty.
- [ ] Keep the repo URL handy. Claude Code will use it when it initializes git.

## Part 9 — Sync — subscribe once, flip one switch (10 min)

- [ ] Subscribe to **Obsidian+** at [obsidian.md/plus](https://obsidian.md/plus) (~$10/mo). Bundles Obsidian Sync.
- [ ] In Obsidian: **Settings → Core plugins → Sync → enable**.
- [ ] **Settings → Sync → Choose remote vault → Create new vault**. Set an encryption password and **save it in a password manager** (you cannot recover the vault without it).
- [ ] *(Mobile, recommended)* Install **Obsidian** on your iPhone/iPad from the App Store → log in with the Obsidian account → open the synced vault. It'll download in the background.

*(Alternative free path: Syncthing + Möbius Sync. Tell Claude Code you want this instead and skip Obsidian+. You'll install Syncthing yourself, pair devices by QR code, and buy Möbius Sync (~$25 one-time) on iOS. More setup; zero ongoing cost; no third-party server.)*

## Part 10 — Hand off to Claude Code

- [ ] Put all three files (`second-brain-spec.md`, `KNOWLEDGE_MGMT_SETUP_SPEC.md`, this checklist) into the vault at `~/second-brain/handover-birefs/`.
- [ ] In Terminal: `cd ~/second-brain`
- [ ] Run: `claude`
- [ ] Paste this prompt **exactly**:

```
Read these three files top-to-bottom before doing anything else:

1. handover-birefs/second-brain-spec.md   (primary — vault architecture)
2. handover-birefs/KNOWLEDGE_MGMT_SETUP_SPEC.md   (supporting — agent-memory discipline)
3. handover-birefs/DAY-0-CHECKLIST.md   (what I've already done)

Then execute §10 "Setup checklist" of the vault spec, starting from Phase 1
step 3 (folder structure) — Phase 1 steps 1-2 (install Obsidian + community
plugins) are already done. Use the Obsidian+ sync path from §A.5 unless I
tell you otherwise.

You are responsible for writing — not just installing — every piece of the
system: the five hook shell scripts from §8, all eight vault-local skills
from §7.2, the domain schemas, the templates, CLAUDE.md, AGENTS.md,
README.md, .gitignore, .gitattributes, the seed ADRs from §17. Work from
the behavioral specifications. Do NOT ask me for source code; you built
these for another operator and you can build them again.

I will paste the Obsidian Local REST API key when you ask for it (Phase 2
step 13 of the spec). Everything else is yours.

Deferred tools (graphify, gitnexus, custom voice pipelines, RAG-Anything,
memory layers) are not Day-1 work. Install graphify per §20.2 only after
the vault has 20+ real sources. Install gitnexus only if we later add a
code subdirectory. Do not propose any deferred tool on Day 1.

Before writing CLAUDE.md, interview me briefly for: full name, employer
name + role, startup codename(s) + stage, freelance sectors, partner /
personal anchors. Do not invent any of these.

Rules:
- Commit after every step with the §8.4 message convention.
- On a 30-minute stall, follow §1.6: stub, log to BLOCKERS.md, continue.
- Stop and ask me only when a decision is genuinely ambiguous or needs my
  personal info. Don't ask for approval on routine steps.
- For OAuth-style MCP auth (GitHub, Gmail, Calendar — if I ask for any),
  walk me through the browser flow; you can't complete it headlessly.

Start now. Begin by summarizing the setup plan back to me in 10 lines so
I know we're aligned, then start Phase 1 step 3.
```

- [ ] Supervise. Approve tool calls as they come. Read the five hook scripts before approving them (Phase 2 step 14) — they run on every tool call, so you want to see what they do.

---

## What Claude Code will do on its own

**During Day 1:**

- Create the full folder layout from §2 of the spec (`mkdir -p` + `.gitkeep`).
- Interview you for identity, domains, role, strategic context.
- Write `CLAUDE.md`, `AGENTS.md`, `README.md`, `.gitignore`, `.gitattributes`.
- Write all 10 files in `_templates/` (daily, meeting, person, client, project, wiki-concept, wiki-entity, wiki-source, decision, blocker).
- Seed `agent-memory/` — all six survival-kit files.
- Write and install the five hook shell scripts (`block_dangerous_writes.sh`, `block_dangerous_bash.sh`, `log_ingestion_to_log_md.sh`, `hot_cache_on_stop.sh`, `session_start_context.sh`, `precompact_save.sh`) — plus `.claude/settings.json` to wire them.
- Write the eight vault-local skills (`/ingest`, `/query`, `/lint`, `/today`, `/crystallize`, `/weekly-review`, `/tech-digest`, `/new-domain`).
- Configure the Obsidian MCP (writes `.mcp.json`; uses the API key you pasted).
- Install the Superpowers plugin (`/plugin marketplace add obra/superpowers-marketplace` then `/plugin install`).
- Seed each domain's `_moc.md` and `_schema.md`.
- Draft the seven seed ADRs from §17 for your review.
- `git init`, first commits, push to your GitHub remote.
- Verify hooks fire correctly (synthetic test commit, confirm `wiki/log.md` gets appended).

**Days 2-3:**

- Ingest your first 10 sources (you drop them into `inbox/`, then run `/ingest inbox/`).
- First `/lint` — walk through results with you.
- Wire web clipper → `raw/clips/`; hook up one meeting transcription source → `raw/transcripts/`.

**Week 2+:**

- First `/weekly-review`.
- First `/crystallize` → `wiki/overview.md`.
- Install `graphify` (`pip install graphifyy` + first run) once you have 20-50 sources.

**Later, only when the pain is real:**

- Custom iPhone voice-capture pipeline (§20.1).
- RAG-Anything for heavy multimodal PDFs.
- Dedicated memory layer (OpenMemory / Supermemory).
- GitNexus — only if you add a code subdirectory.
- L1 credential vault — not a Day-1 concern; Claude Code will set this up when you first have something sensitive enough to warrant it.

## When to interrupt Claude Code

- Any step running >30 min with no forward progress — remind it of the §1.6 stall rule.
- It asks a question you don't understand — ask it to explain in plain language.
- A command looks destructive (`rm -rf`, `git push --force`, `git reset --hard`) — read the command, ask why, proceed only if the answer is good.
- It proposes installing a "deferred" tool on Day 1 — push back; point it at §15 and §20.

## First real use, once setup is done

Drop an article, meeting transcript, or voice dictation into `~/second-brain/inbox/`. In Claude Code, run `/ingest inbox/`. Watch what it files, where, and how it links things. Correct what's wrong — that's how the system learns your style. Do this daily for a week and you'll have a working second brain.

---

*If anything in this checklist is unclear, ask Claude Code directly after you've started it. It has the full context of both specs and can explain anything.*
