# Changelog

All notable changes to `code-os`, release by release. Format loosely follows [Keep a Changelog](https://keepachangelog.com/); versions follow [SemVer](https://semver.org/) in spirit, where a major bump means the operating doctrine changed and not just the files.

## [Unreleased]

### Added
- `modules/multi-model-orchestration/orchestrating-cli-subagents.md` §6: the wrapper contract for multi-agent fan-out. Thin wrapper agents (cheapest model, lowest effort) run Codex subagents inside workflow pipelines, so the fan-out's heavy work bills the Codex side — assuming the CLI is authenticated against a flat plan rather than an API key. First field result 2026-08-10 (one run): a 12-agent mine-and-map pipeline over two monorepos, ~892k subagent tokens, ~20 minutes, zero failed runs. Includes the `codexAgent()` factory sketch, a success gate requiring a non-empty final message, and two wrapper-specific cautions (strict stage-output formats; empty final messages are silent failures).
- §5 runbook hardening from adversarial review: explicit `--sandbox workspace-write` / `-s read-only` instead of `--full-auto` and config-key-only sandboxing, explicit `-C <repo-root>`, the stdin prompt form for oversized prompt files, an auth/billing caveat, and mechanically accurate stdin-hang and backgrounding explanations.
- Agentic harness: an unattended fresh-session-per-step recipe (`claude -p` with `--max-turns`, `--max-budget-usd`, `dontAsk`), with what each limit really enforces and why `--bare` stays off; `/context all` and `/hooks` checks that the install actually loaded.
- Multi-model orchestration: "Native workers first" (Claude subagents and workflows before foreign CLIs), Claude Code as a worker, Codex structured output and worktree flags, a fifth failure mode (parallel writers in one checkout), and the Claude Code Bash-timeout and auto-mode limits that break wrappers.
- Skills guide: what a skill costs (listing budget, compaction re-attach, `/skill-doctor`), and "Keep only what this project needs": project or local scope, `/mcp` per-project toggle, monthly removal.
- Patterns: #11 hand-authored migrations (expand → migrate → contract) and #12 lifecycles as explicit state machines.
- Second brain: templates for daily, meeting, entity, decision and person pages; a reconstruction test for ingestion depth; "Capture limits" in the export prompt.

### Fixed

Safety first: these defects could run commands without a prompt or put secrets in git.

- Harness safety gate answered `"allow"` to every command it did not block, which skips the permission prompt for all Bash. It now stays silent and lets the normal permission flow decide ([hooks](https://code.claude.com/docs/en/hooks#pretooluse-decision-control)).
- Harness install prompt recommended `acceptEdits`, which also auto-approves `rm`, `mv`, `cp` and `sed`; `bypassPermissions` was described as safe in a worktree. Both corrected ([permission modes](https://code.claude.com/docs/en/permission-modes)).
- Second brain: `inbox/` and Obsidian plugin `data.json` (Local REST API key and TLS private key) were committed; the MCP key was pasted into the chat and a project `.mcp.json`; Obsidian Git auto-commit committed captures before any gate; L1 held a plaintext secrets file. Now gitignored, registered at local scope from a separate terminal, auto-commit off, and credentials go to a password manager.
- Second brain: hooks left `wiki/hot.md`, `wiki/log.md` and `raw/session-learnings/` modified or untracked after every session; the `raw/` guard missed MCP and Bash writes and blocked legitimate new files. Hooks never leave a tracked file modified now, and `raw/` is protected by a permission rule plus a tested move command.
- Hooks: the Stop hook fires every turn and its `systemMessage` never reached Claude (now `additionalContext` with the `stop_hook_active` guard); hook commands used repo-relative paths that break after a `cd` (now `${CLAUDE_PROJECT_DIR}`); PreCompact and SessionStart behaviour described as the docs define it.
- "CLAUDE.md is reloaded every turn" → loaded at session start and re-injected after compaction; GitNexus block no longer duplicated in `CLAUDE.md` and `AGENTS.md` (`@AGENTS.md` import instead); auto-memory 25 KB limit; Claude Code native installer instead of npm.
- Codex runbook: `--full-auto` is gone (codex-cli 0.167.0); progress streams to stderr, not stdout; prompts are read from the file on stdin.
- Production patterns: the approval function took the approver as a parameter and left `EXECUTE` granted to `PUBLIC`; provider I/O was stored verbatim (personal data); Zod 4, AI SDK 6/7 and Inngest examples updated. Spec §17–§18: idempotency keys reserved before the side effect with IETF-draft status codes, migration healthcheck no longer drains old instances mid-deploy, fail-closed defined, erasure as a state machine with the GDPR deadline, unsourced retention figures removed.
- Skills guide: wrong descriptions of the github, security-guidance and chrome-devtools-mcp plugins, an install command for a plugin that is not in the official marketplace, a local-plugin path Claude Code does not scan, plugin auto-update defaults.

### Changed

- **superpowers is optional.** The harness loop is driven by a plain spec with a "done when" check; superpowers skills are named only "if installed", and the second brain no longer installs it on Day 1. The maintainer's note on why is in [`useful-skills.md` §7](modules/skills-guide/useful-skills.md#7-superpowers). The `docs/superpowers/` folder name is unchanged, so existing installs keep working.
- GitNexus and MCP servers are deferred until the repo earns them, at project or local scope, consistently across the harness spec, bootstrap and plugin.
- `modules/skills-guide/useful-skills.md` is now in English, as CONTRIBUTING requires.
- Reference stack refreshed to November 2026 (Node 26 LTS, ESLint 10, TypeScript 7, AI SDK 7, Vitest 5, pnpm 12, Better Auth for new projects, Claude Agent SDK); tool-orchestration compendium moved to AI SDK 7 and the Gemini Interactions API, stale June items cut or labelled.
- Second-brain spec v2.4: frontmatter contract matches the templates, quoted wikilinks in YAML, full-path links, the complete lint list, dated contradiction callouts, every ingestion linked from the daily note, one writer for hub files, trust git over agent logs.

## [2.0.0] - 2026-08-09 · The multi-model release

Six weeks of running v1 daily on production codebases taught us that parts of it were wrong. Not wrong for June 2026, wrong for today. This release is what we would ship knowing what we know now, and the changelog doubles as the list of lessons.

### What we learned (and changed)

**1. One session can orchestrate other vendors' CLI agents, and should.**
v1 was single-agent: Claude Code, alone, doing everything on one plan's token budget, with one vendor's blind spots reviewing that vendor's own output. The new hero module, [`modules/multi-model-orchestration/`](modules/multi-model-orchestration/index.md), documents the pattern we now use daily. Claude Code stays the durable orchestrator; Codex CLI (or any headless CLI, since the contract generalizes to grok, ollama, whatever ships next) is spawned as explorer, implementer, adversarial-reviewer and second-opinion subagents. Bulk work moves to the cheapest capable subscription, reviews come from a model that doesn't share the author's failure modes, and the orchestrator keeps context, taste and the accept-or-reject call. The module ships the full runbook: the three-file run-path convention, the exact `codex exec` invocations for write and read-only modes (verified against codex-cli 0.145.0), and the four failure modes that cost us real hours, namely unclosed stdin, wrong working directory, backgrounded runs, and reading the trail.

**2. Ceremony must be tiered. The full loop was overkill as a default.**
v1 mandated spec, plan and execute for all non-trivial work. That was right for the models of early 2026 and is usage-expensive overkill for today's frontier models. v2 tiers it: trivial work runs direct; work that used to be complex runs inline on one frontier model at high reasoning effort, where the conversation is the plan; the full harness loop is reserved for work that would not survive a single session. The `code-os-core` skill, the harness spec (§4) and the skills guide all carry the new rule. The survival kit and the safety hooks stay Day 1, because memory and safety were never the expensive part.

**3. Prompts are technical debt, including ours.**
v1 shipped five overlapping routing documents (roughly 550 duplicated lines), prescribed agent-generated `CLAUDE.md` files up to 600 lines, and told installing agents their token budget was "effectively unbounded". v2 eats its own dogfood. The routing layer is now `README.md` plus `GETTING_STARTED.md` (`index.md`, `capabilities.md` and `INSTALL.md` are gone, since the plugin's `/code-os:install` does the routing), `CLAUDE.md` is prescribed as minimal and hand-owned (a couple of screens the operator will actually maintain), and the unbounded-budget framing is dead. The big specs remain, clearly labeled as the heavy tier they power.

**4. Model guidance belongs in the docs, dated.**
v1 contained no model-selection guidance at all. v2 adds the cost, intelligence and taste selection table (with the August 2026 roster as a worked example, and the instruction to rank by what *you* actually pay rather than list price), effort ceilings, and the escalate-without-asking rule. Dated content is now labeled as a snapshot, the June 2026 orchestration compendium included.

**5. Cross-model review is a general quality mechanism, not a security niche.**
v1 used a second model only to audit third-party plugins. v2 makes going back and forth between models a first-class part of the workflow (second opinions on designs, adversarial review of diffs), because it is the cheapest quality mechanism we know.

### Added
- `modules/multi-model-orchestration/`, the new hero module: pattern, roles, tiering rule, model table, Codex runbook, failure modes, and the contract for wrapping any other CLI.
- `/code-os:subagents`, a plugin command that installs the conventions (model table, run paths, runbook) into a repo's `CLAUDE.md`.
- `CHANGELOG.md`, this file.

### Changed
- `code-os-core` skill: "work the loop" is now "tier the ceremony", plus the cross-vendor delegation rule with first-person verification.
- `/code-os:install` router: six modules, six commands.
- `/code-os:harness`: the `CLAUDE.md` prescription is now minimal and hand-owned; ceremony tiering is noted up front.
- Harness and second-brain specs and bootstraps: `CLAUDE.md` line-count ceilings replaced with the minimal-hand-owned rule (this also resolves a v1 inconsistency, where the spec said 400 lines and the bootstrap said 600), and the "token budget effectively unbounded" framing removed.
- `modules/skills-guide/`: superpowers and the Explore, Plan, Implement (TDD) workflow re-scoped to the heavy tier; the session-compression plugin note no longer assumes a specific cheap model.
- `modules/tech-references/`: the orchestration compendium is explicitly a dated snapshot with a scope note pointing at the new module for dev-workflow orchestration.
- README (English and Italian): repositioned around one orchestrator and many models; badges and canonical summaries updated; citation metadata (`CITATION.cff`, `codemeta.json`, `AI-CITATION.md`) bumped to 2.0.0.

### Fixed
- `modules/second-brain/`: the hook count said five in eight places while naming six scripts on the same lines. It is six.
- `CONTRIBUTING.md`: the "markdown only, no binaries" rule contradicted the three tracked PNGs in `docs/` that the README itself embeds. The rule now names that exception.
- Grounding claims now say what is true: every reference claim carries a named source or a pinned version. The previous wording promised a fetched URL, and the reference modules cite sources by name rather than by link.

### Removed
- `index.md`, `capabilities.md` and `INSTALL.md`, which were routing duplication. `README.md` positions, `GETTING_STARTED.md` routes, the plugin drives.

## [0.1.0] - 2026-06-21

Initial public release: the agentic-harness, second-brain, skills-guide, tech-references, and ai-tools and personas modules; the code-os plugin (always-on `code-os-core` skill plus five commands); bilingual README; citation metadata.

---

<sub>In produzione, non in slide. · by yempik. · maintained by Simone Bova</sub>
