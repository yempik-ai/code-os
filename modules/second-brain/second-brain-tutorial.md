# Second Brain — Command Tutorial

> **For the operator.** Once Claude Code has finished the setup (see `BOOTSTRAP.md`), your vault ships with **eight custom slash commands** that do all the real work. This file is the field manual: what each command does, how it works under the hood, when to reach for it, and the rhythm they compose into over a week.

## How to invoke a command

Inside a `claude` session run from anywhere under `~/second-brain/`, type the command as the first token of your message:

```
/ingest inbox/
/query what did I decide about pricing last month?
/today
```

Commands take free-text arguments after the slash name. If you forget the shape of an argument, just type the slash name alone — the command will ask.

All commands share a handful of invariants from `CLAUDE.md`:
- They read `wiki/hot.md` and the relevant domain's `_moc.md` / `_schema.md` before touching anything.
- They respect the six non-negotiables (raw is immutable, provenance on every claim, contradictions flagged not overwritten, commit-per-ingestion, 30-minute stall rule, L1/L2 secret split).
- They commit after meaningful work with the `<type>(<scope>): <subject>` convention.
- They obey graph hygiene (bidirectional round-trip, MoC anchoring, link-density floors, tag dictionary).

If a command ever does something you don't expect, the fix is almost always in the schema — either a `domains/<domain>/_schema.md` file or `CLAUDE.md` itself. Don't fight the command; fix the schema.

---

## Daily cadence

### `/today` — seed the day's capture surface

**What it does.** Generates today's daily note at `daily/YYYY/YYYY-MM-DD.md` from the daily template, pre-filled with (a) open threads pulled from `agent-memory/OPEN-THREADS.md`, and (b) today's meetings if a calendar MCP is wired.

**How it works.**
1. Reads the daily template from `_templates/daily.md`.
2. Reads `agent-memory/OPEN-THREADS.md` and copies unresolved threads relevant to today.
3. If a calendar MCP (Google Calendar, etc.) is configured, fetches today's events and stubs meeting notes in `domains/<domain>/meetings/YYYY/` with attendees linked via `[[people/<name>]]`.
4. Writes the daily note. Does not commit — daily notes accumulate and commit at end-of-day or on next ingestion.

**When to use.** First thing in the morning. One command; ~30 seconds of Claude work. Gives you a single page to capture into all day.

**When not to use.** Don't run it twice in one day — it will warn but won't re-create. If you want to regenerate from a fresh open-threads snapshot, delete the file manually first.

### `/ingest [path]` — process sources into the wiki

**What it does.** The workhorse. Takes files sitting in `inbox/` (or a specific `raw/` path), reads them fully, finds the wiki pages they touch, updates those pages with source-attributed claims, creates new concept/entity/source pages when warranted, handles contradictions, files the raw source under `raw/<category>/YYYY-MM/`, and commits.

**How it works** (follows `CLAUDE.md` §"Ingestion rules" exactly):
1. Reads the source end-to-end. Identifies type: article, transcript, voice note, email, clip, conversation.
2. Finds 3–10 existing wiki pages the source touches. If <3 → treats as a new concept/entity (creates one page) or context-free noise (logs to `wiki/log.md` and stops).
3. For each touched page: appends or updates with claims, each carrying a source citation.
4. For new concepts/entities: creates the page using the right template, cross-links to ≥2 existing wiki pages, adds the page to `wiki/index.md`.
5. Meeting transcript specifically: files under `domains/<domain>/meetings/YYYY/`, extracts decisions → `domains/<domain>/decisions/`, extracts action items → attendees' `people/` pages + today's daily note.
6. Voice / inbox entry: extracts TODOs, files to the right project/domain, updates the daily note.
7. Contradiction with existing wiki: adds `> [!contradiction]` callout keeping both claims + sources. Adds to `wiki/contradictions.md`. Never silently overwrites.
8. Moves the raw source from `inbox/` to `raw/<category>/YYYY-MM/`. Never deletes.
9. Appends to `wiki/log.md`: `## [YYYY-MM-DD HH:MM] ingest | <title> → <pages>`.
10. Commits with: `ingest(<domain>): <title> — N pages (+M new)`.

**Detail-preservation contract** (§10-bis of the spec): a 400-line transcript compiles to an 80–150 line meeting page (20–30% retention), not a 40-line skim. If the command runs out of context mid-ingest, it commits what's there with a `[!blocker]` callout + `BLOCKERS.md` entry pointing at the depth gap — it will never ship shallow silently.

**When to use.**
- Any source lands in `inbox/` (article clipped via Obsidian Web Clipper, transcript exported from Otter/Fathom/Granola, voice note dictated into Claude, email forwarded, PDF dropped in).
- You have a specific raw file that wasn't processed and needs to be: `/ingest raw/articles/2026-04/foo.md`.
- Daily, ideally — the longer `inbox/` sits, the heavier each ingestion becomes.

**When not to use.**
- A source already in `raw/` that was ingested before. Check `wiki/log.md` first.
- A source that is genuinely secret (health, credentials, financial). That goes to `~/.second-brain-local/` (L1), not through `/ingest`.

**Gotchas.**
- If `/ingest` asks you to disambiguate which domain a source belongs to, answer precisely — the answer gets cached as a new entry in that domain's `_schema.md` so the next similar source auto-routes.
- If it flags a contradiction, read the callout. The resolution is a separate `/crystallize` pass, not an overwrite.
- `inbox/` files older than 7 days are a `/lint` failure. Clear inbox weekly.

### `/query <question>` — ask the vault

**What it does.** Answers a question about the vault using only vault content, with citations. Not a chat with Claude's general knowledge — a retrieval-then-synthesis over your own notes.

**How it works** (follows `CLAUDE.md` §"Query rules"):
1. Reads `wiki/hot.md` (recent context, free).
2. Reads `wiki/index.md` to pick candidate pages.
3. Pulls 3–5 relevant wiki pages. Follows `[[wikilinks]]` one hop.
4. Domain-specific question: also reads that domain's `_moc.md` + `_schema.md`.
5. Temporal question ("what did we discuss about X last week"): also reads relevant `daily/` notes in the time window.
6. Answers with citations in the form `[[wiki/concepts/auth-patterns]] claims X because [[raw/articles/2026-03/auth-survey]]`.
7. If the answer exposes a gap (open question, missing source), appends to `wiki/questions/` or `agent-memory/OPEN-THREADS.md`.

**When to use.**
- "What did I decide about X?" — decision lookup.
- "Who is Y and what have we discussed?" — person lookup.
- "What did I say about Z in last week's meetings?" — temporal cross-cut.
- Before starting a new piece of work, to surface everything the vault already knows about the topic so you don't redo thinking.

**When not to use.**
- Questions Claude's general knowledge answers better ("what is OAuth?"). Use a normal chat.
- Questions the vault literally cannot know (live market data, real-time anything). The answer will be "no sources" — that's the point.

**Gotchas.**
- Every answer cites sources. If you see a claim without a `[[source]]` citation, that's a bug — push back. An uncited claim in a `/query` answer means either the vault has a sourceless page (a `/lint` failure) or the command hallucinated. Either way, investigate.
- If `/query` returns "no sources found", the gap is real — it will have been logged to `wiki/questions/` or `OPEN-THREADS.md` for you.

### `/crystallize` — preserve a good conversation

**What it does.** Takes the current conversation (or a specified recent thread) and distils the insight into wiki pages — a new concept, entity, synthesis, or decision. The key distinction: `/ingest` files an **external** source; `/crystallize` captures **Claude-and-you reasoning** that happened live in the session.

**How it works.**
1. Reads the active conversation buffer and identifies the core insight.
2. Decides page type: concept, entity, synthesis (connecting 2+ existing ideas), or decision (if a choice was made).
3. Creates or updates the relevant wiki page(s).
4. Sources point at the conversation log itself — a dated transcript file is saved in `raw/conversations/YYYY-MM/` so provenance still holds.
5. Cross-links into the existing graph (respects the link-density floor from `CLAUDE.md` §"Graph hygiene").
6. Updates `wiki/index.md` if new pages were created.
7. Commits with: `crystallize(<scope>): <title> — <N pages>`.

**When to use.**
- A productive Claude session produced a framing, decision, or synthesis worth keeping.
- You worked through a hard problem verbally with Claude and now want the answer durable.
- An open thread got resolved mid-session — `/crystallize` captures the resolution and closes the thread in `OPEN-THREADS.md`.

**When not to use.**
- For routine external sources. That's `/ingest`.
- Mid-conversation when the insight isn't settled yet — wait until you've actually converged. Premature crystallization creates brittle pages that need rewriting.

**Gotchas.**
- Crystallized pages start with provenance pointing at a conversation log, not an external source. That's fine — provenance is satisfied. But a page whose only source is a conversation log is a hypothesis until external evidence corroborates it. Mark `status: hypothesis` if uncertain.

---

## Weekly cadence

### `/weekly-review` — the Sunday regroup

**What it does.** The ~30-minute Sunday ritual. Summarises the week across daily notes, refreshes `wiki/overview.md` deltas, closes resolved entries in `agent-memory/OPEN-THREADS.md`, runs `/lint`, surfaces blockers, and writes a weekly summary note.

**How it works.**
1. Reads every `daily/YYYY/YYYY-MM-DD.md` from the past 7 days.
2. Aggregates: what was decided, what shipped, who appeared (people touched), what projects advanced, what got stuck.
3. Writes `weekly/YYYY-WW.md` with the summary.
4. Updates `wiki/overview.md` with deltas — new concepts surfaced, entities added, syntheses earned.
5. Walks `agent-memory/OPEN-THREADS.md`: closes threads that got resolved, promotes stale threads that need attention.
6. Runs `/lint` (report-only, no autofix) and includes the lint report in the weekly note.
7. Surfaces blockers from `agent-memory/BLOCKERS.md` that are still unresolved after ≥7 days — asks whether to escalate or deprioritise.
8. If you run the tech-news domain, chains into `/tech-digest` (below).
9. Commits.

**When to use.** Sundays, or whatever cadence day you pick. Budget 30 min — 15 min of Claude work, 15 min of you reading and correcting.

**Gotchas.**
- Skipping weekly reviews is the fastest way to let the vault rot. After two skipped weeks, `OPEN-THREADS.md` and `wiki/overview.md` drift from reality and future `/query` answers degrade.
- Don't bury disagreements with the weekly summary. If Claude summarises something wrong, correct it in the weekly note — the correction becomes a source for the next week.

### `/tech-digest` — weekly tech-news compiler

**What it does.** Compiles the week's tech-news clips into a weekly digest at `domains/tech-news/weekly/YYYY-WW.md`. Promotes recurring themes to `domains/tech-news/trends/` when they cross a **3-citation threshold** (same theme appearing across 3+ sources).

**How it works.**
1. Reads all sources ingested into the `tech-news` domain during the past 7 days.
2. Clusters them by theme (model releases, infra shifts, regulatory moves, etc.).
3. Writes the weekly digest with per-theme summaries, each carrying citations.
4. Counts citations per theme across all weekly digests to date. Themes at 3+ citations get a `domains/tech-news/trends/<slug>.md` page (or an update if the trend page exists).
5. Updates `domains/tech-news/_moc.md` with links to the new weekly + any new trends.
6. Commits.

**When to use.** During `/weekly-review` (it will chain automatically). Rarely on its own unless you're catching up after a skipped week.

**When not to use.** If you don't run a tech-news domain, this command is a no-op. Remove it from your workflow in `PREFERENCES.md`.

---

## Occasional / structural

### `/new-domain <slug>` — scaffold a new life compartment

**What it does.** Adds a new top-level domain to the vault. Creates `domains/<slug>/` with `_moc.md`, `_schema.md`, and initial subfolders (meetings, decisions, projects, people-of-interest as appropriate). Adds the domain to `CLAUDE.md` § Domains. Seeds `agent-memory/OPEN-THREADS.md` with a "just-added" thread so you can't forget the domain exists.

**How it works.**
1. Validates the slug is kebab-case and not already present.
2. Creates the folder tree.
3. Writes `_moc.md` from the MoC template — empty shell pointing back to `CLAUDE.md`.
4. Writes `_schema.md` from the schema template — empty shell waiting for rules as they emerge from use.
5. Edits `CLAUDE.md` § Domains to include the new entry.
6. Appends a thread to `OPEN-THREADS.md` reminding you to populate identity, key people, and initial sources.
7. Commits: `feat(domains): add <slug> domain scaffold`.

**When to use.** A new life compartment emerges — new employer, new startup codename, new freelance sector, new life phase (e.g., a move to a new city, starting a degree). **Not** for narrow projects inside an existing domain — those go under `domains/<existing>/projects/`.

**When not to use.** Don't create a domain you're not committed to filling. Empty domains are lint failures and noise in every future `/query`. If you're unsure, stage it as a project under an existing domain first; promote later if it earns its own tree.

### `/lint` — vault health check

**What it does.** Runs the full set of graph-hygiene and content-quality checks defined in `CLAUDE.md` § Lint rules. Produces a report of failures grouped by severity. Does **not** modify files in report-only mode.

**Checks (the full set):**

*Graph hygiene failures (non-negotiable):*
- **Orphans** — wiki pages with <2 incoming backlinks (meta pages exempt).
- **Dead links** — `[[wikilinks]]` pointing at nonexistent pages.
- **Missing MoC anchor** — non-meta wiki page with non-`global` domain and no link to `domains/<domain>/_moc.md`.
- **Under-density** — page below the link-density floor for its type (concepts ≥3, entities ≥2, sources ≥1+raw, syntheses ≥3, questions ≥1).
- **Unindexed** — new `wiki/*` page not listed in `wiki/index.md`.
- **Broken round-trip** — `[[B]]` cited in A but A not mentioned in B's Related section.
- **Unowned action items** — `- [ ]` checkboxes without a `[[people/<owner>]]` link.
- **Unknown tags** — tags used in frontmatter not in `wiki/tag-dictionary.md`.

*Content-quality failures:*
- **Stale** — pages with `updated:` older than 180 days and no source added since.
- **Sourceless** — `wiki/*` pages without `sources:` frontmatter (exempt: `status: hypothesis`).
- **Contradiction debt** — unresolved `[!contradiction]` callouts older than 30 days.
- **Schema drift** — frontmatter fields that don't match the contract in `CLAUDE.md`.
- **Unprocessed** — files in `inbox/` older than 7 days.

**How it works.** Pure read + report. Walks the vault, checks each invariant, prints a grouped report. Zero side effects in report-only mode.

### `/lint --autofix` — apply safe fixes

**What it does.** Same checks as `/lint`, but applies the subset of fixes that are **unambiguous and reversible**: adds missing index entries, fixes round-trip (adds the reciprocal link in Related), inserts missing MoC anchors, demotes uncited pages to `status: hypothesis`. Does **not** auto-fix dead links, contradictions, unowned action items, or sourceless claims — those need human judgment.

**Safety rule (from `CLAUDE.md`):** Never run `/lint --autofix` without committing first. If the autofix is wrong, a `git reset` gets you back. Without a prior commit, your in-flight work is at risk.

**When to use.**
- Report-only `/lint`: weekly (chained from `/weekly-review`) and any time something feels off.
- `--autofix`: monthly, or after a heavy ingestion burst, or when a report-only run shows >20 fixable findings. Always commit first.

**Gotchas.**
- Graph-hygiene failures compound. 5 orphans this week becomes 30 next month if you let the floor drop. Fix when small.
- Contradiction debt is the most costly failure to leave unresolved — every `/query` that touches a contradicting page has to reason about both claims. Resolve contradictions during `/weekly-review` or with a dedicated `/crystallize` pass.

---

## Typical week (the rhythm these compose into)

- **Monday–Friday morning:** `/today` (~30 sec). Read the pre-filled open threads and calendar. Capture throughout the day into the daily note, `inbox/`, or via Obsidian Web Clipper.
- **Monday–Friday evening (or as sources land):** `/ingest inbox/` (~2–10 min depending on volume). Drains the inbox, updates the wiki, commits.
- **Any time:** `/query` when you need to look something up before starting a new piece of work. `/crystallize` after a conversation with Claude that produced a durable insight.
- **Sunday, 30 min:** `/weekly-review`. Chains into `/tech-digest` (if tech-news is a domain) and `/lint` (report-only).
- **First of the month (~15 min):** `git commit` everything pending, then `/lint --autofix`. Review and commit the fixes.
- **Whenever life changes shape:** `/new-domain <slug>` for a genuinely new compartment.

**Skipped-week recovery.** If you skip a week, the catch-up order is: `/ingest` (drain inbox) → `/weekly-review` (will summarise two weeks) → `/lint`. Do **not** skip straight to `/lint --autofix` on a two-week-stale vault — the autofix set assumes recent, coherent state.

---

## Behind every command: the hooks

Five shell hooks (specified in `second-brain-spec.md` §8, written by Claude Code during setup) enforce the invariants these commands rely on. You don't invoke hooks directly — they fire on every tool call — but knowing they exist helps debug unexpected command behavior:

- `block_dangerous_writes.sh` (PreToolUse, Write|Edit|MultiEdit) — denies writes to `raw/` (immutability) and to L1 paths from L2.
- `block_dangerous_bash.sh` (PreToolUse, Bash) — denies `rm -rf ~`, `git push --force`, `git reset --hard` and similar.
- `log_ingestion_to_log_md.sh` (PostToolUse, Bash) — auto-appends every `ingest` commit to `wiki/log.md`.
- `hot_cache_on_stop.sh` (Stop) — rebuilds `wiki/hot.md` with recent context at session end.
- `session_start_context.sh` (SessionStart) — prints the session-start protocol reminder.
- `precompact_save.sh` (PreCompact) — writes a compaction checkpoint to `agent-memory/PROGRESS.md`.

If a command produces unexpected output, check whether a hook denied something — look for `wiki/log.md` and `agent-memory/BLOCKERS.md` for clues.

---

## What these commands are not

- They are **not** a replacement for thinking. They compile capture into a graph; the thinking still happens between your ears and in conversation.
- They are **not** generic across vaults. Each command reads `CLAUDE.md` + the relevant domain schema. Copy-pasting these commands into a different vault without the schema and the hooks produces garbage.
- They are **not** infallible. Every command's output is reviewable and reversible — every command commits, every commit is a rollback point. If a command did the wrong thing, `git revert` and fix the schema that let it.

The commands exist so you can spend your attention on the content, not the filing system. When filing friction creeps back in, the schema is wrong — fix it, and the commands will respect the fix on the next invocation.

---

*For the behavioral specs these commands implement, see §7.2 of `second-brain-spec.md`. For the agent-memory discipline they rely on, see §3 of `KNOWLEDGE_MGMT_SETUP_SPEC.md`. For the invariants they enforce, see `CLAUDE.md` (the vault's north-star briefing, written during setup).*
