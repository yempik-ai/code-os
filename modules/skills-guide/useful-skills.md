# Useful setup for Claude Code: quick guide

In Claude Code, run `/plugin` to open the plugin panel. Claude Code adds **Anthropic's official marketplace** (`claude-plugins-official`) the first time you start an interactive terminal session, and the **Discover** tab lists the plugins from every marketplace you registered ([install plugins](https://code.claude.com/docs/en/plugins/install)).

> **General note**: before you install any third-party plugin or skill, **audit it for prompt injection and suspicious behaviour** (see the dedicated section at the end).

---

## Keep only what this project needs

Every plugin, skill and MCP server adds tool names, skill listings and possibly hooks to every session. More tools do not make outcomes better, and overlapping ones make the agent pick wrong ([tool orchestration §1.1](../tech-references/tool-orchestration-best-practices-and-resources.md#11-tool-design-fewer-higher-level-consolidated-tools-beat-tool-proliferation)).

- Install at project or local scope, not user scope, unless you need the tool in every repo ([plugin scopes](https://code.claude.com/docs/en/plugins/install#choose-an-install-scope)). `claude mcp add` already defaults to local scope ([MCP](https://code.claude.com/docs/en/mcp)).
- Turn off a server you do not need in this project in `/mcp`: Claude Code records the choice for this project only and keeps the configuration ([MCP](https://code.claude.com/docs/en/mcp)).
- Every month, remove what you did not use. `/skill-doctor` shows how often each skill runs; the **Installed** tab in `/plugin` groups plugins under **Not used recently**.

## What a skill costs

From the [skills docs](https://code.claude.com/docs/en/skills):

- **Every turn:** the skill listing. Each skill's `description` plus `when_to_use` is cut at 1,536 characters, and the whole listing is budgeted at 1% of the context window. When it overflows, Claude Code drops the descriptions of the skills you invoke least, and Claude can no longer match them. A skill with `disable-model-invocation: true` is not in the listing.
- **When invoked:** the rendered `SKILL.md` enters the conversation once and stays. Claude Code does not re-read the file on later turns.
- **After compaction:** Claude Code re-attaches the most recent invocation of each skill, keeping its first 5,000 tokens, within 25,000 tokens for all skills together. Older skills can drop out. Put what matters most at the top, and invoke the skill again after a compaction.
- **Measure:** `/doctor` estimates the listing cost. `/skill-doctor` (v2.1.252+) reports each skill's context cost and how often it runs. Hide a personal or project skill with `skillOverrides` (`"on"`, `"name-only"`, `"user-invocable-only"`, `"off"`) in settings. For a plugin, `claude plugin details <name>` prints the tokens it adds to every session ([install plugins](https://code.claude.com/docs/en/plugins/install)).

---

## Recommended plugins and skills

A listing in the official marketplace does not mean Anthropic wrote the plugin: several entries point at third-party repositories pinned to a commit. Audit those like any third-party code.

### 1. context7: current docs for any library

- **What**: Upstash's MCP server. It injects *version-specific* library and framework documentation into the prompt, fetched live from the official source.
- **Why**: models have a knowledge cutoff. Without context7, Claude invents outdated or hallucinated APIs. With it, you get real snippets for the version you actually use.
- **When**: every time you work with external libraries (React, Next.js, Prisma, Tailwind, Express, Drizzle, Hono, and so on). The catalogue covers the whole JS/TS ecosystem and other languages.
- **For whom**: every developer with third-party dependencies.
- **Setup**:
  ```bash
  claude mcp add --scope user --transport http context7 https://mcp.context7.com/mcp \
    --header "CONTEXT7_API_KEY: YOUR_KEY"
  ```
  Or through the official marketplace: `/plugin install context7@claude-plugins-official`.

### 2. github: GitHub's MCP server

- **What**: GitHub's official MCP server, packaged as a plugin: issues, pull requests, code review and repository search through GitHub's API. It adds no commands or agents of its own.
- **Why**: Claude reads diffs, writes coherent PR descriptions and answers review comments without you switching between terminal and browser.
- **When**: issue and PR work beyond what the `gh` CLI through Bash already covers.
- **For whom**: anyone who lives on GitHub.
- **Setup**: `/plugin install github@claude-plugins-official`, then export `GITHUB_PERSONAL_ACCESS_TOKEN` with the narrowest scopes you need. The plugin connects to `https://api.githubcopilot.com/mcp/` with that token.

### 3. remember: memory across sessions

- **What**: a plugin that captures sessions automatically, compresses them with a cheap model into layered daily logs, and reloads the relevant context at the start of the next session. (The historical default was Claude Haiku; if the plugin allows it, point the summariser at a stronger model: the quality of the compression is the quality of your memory.)
- **Why**: unlike `CLAUDE.md` (manual), `remember` works in the background. It saves context budget because it stores typed observations (`bugfix`, `discovery`, `decision`), not raw transcripts.
- **When**: long projects where you lose the thread between sessions, multi-day refactors, complex debugging.
- **For whom**: anyone on large codebases or projects that last weeks.
- **Setup**: `/plugin install remember@claude-plugins-official`. Third-party: the official marketplace points at an external repository pinned to a commit, so audit it first.

### 4. security-guidance: Claude reviews its own edits

- **What**: Claude reviews its own changes for vulnerabilities while it works, at three layers: a pattern check after each file edit (no model call; each warning fires once per pattern per file per session), a background model review of each turn's diff, and a deeper agentic review on each `git commit` or `git push` Claude makes. Nothing is blocked: findings go back to Claude to fix ([security-guidance](https://code.claude.com/docs/en/security-guidance)).
- **Why**: Claude can write insecure code without noticing. This plugin catches injection, unsafe deserialization and unsafe DOM APIs before the code reaches a pull request.
- **When**: always on. Critical for backend, APIs and handling user input.
- **For whom**: everyone, but essential for user-facing code.
- **Setup**: `/plugin install security-guidance@claude-plugins-official`. Needs Python 3.7+ on `PATH` (3.10+ for the agentic commit review) and a git repository for the turn and commit reviews. The model reviews spend usage. Add repository rules in `.claude/claude-security-guidance.md`.

### 5. skill-creator: build and tune your own skills

- **What**: a toolkit with four modes, **Create**, **Eval**, **Improve**, **Benchmark**, to design, test and improve custom skills.
- **Why**: a good skill needs precise structure (YAML frontmatter, naming, triggers). The creator automates the scaffolding and helps you measure the skill with repeated evals.
- **When**: you want to capture a recurring workflow as a reusable skill, or understand why a skill does not trigger when it should.
- **For whom**: anyone extending Claude with their team's domain knowledge.
- **Setup**: `/plugin install skill-creator@claude-plugins-official`. Run it with `/skill-creator`.

### 6. typescript-lsp (or your language's LSP)

- **What**: a plugin that connects Claude Code to the **Language Server Protocol**: go-to-definition, find references and real-time error checking on `.ts/.tsx/.js/.jsx/.mts`.
- **Why**: without an LSP, Claude navigates code with grep and regex and gets lost in monorepos. With one, it gets the same symbols as VS Code, deterministically.
- **When**: medium or large TypeScript codebases (more than about 50 files).
- **For whom**: TS/JS developers; equivalent LSPs exist for Python, Go, Rust and others (see `Piebald-AI/claude-code-lsps`).
- **Setup**: install `typescript-language-server` globally (`npm i -g typescript-language-server typescript`), then `/plugin install typescript-lsp@claude-plugins-official`. The official plugin is sometimes incomplete: alternatives are `cclsp` or the Piebald marketplace.

### 7. superpowers

- **What**: a methodology by Jesse Vincent: brainstorming, written plans, subagent-driven development with review, red/green TDD.
- **Install:** `/plugin install superpowers@claude-plugins-official`. The official marketplace lists it, but it points at a third-party repo pinned to a commit: audit it like any third-party code.
- **A note from the maintainer (2026-11):** Try it: it is a good plugin, and its brainstorming and spec writing helped me at times. Its plans were useless, at least for me, and in my experience it limits Opus 5.5. I no longer use it. If you try it, compare the result with Claude working from a plain spec and a "done when" check, and keep it only if it wins on your work.

### 8. gitnexus MCP: a knowledge graph of the code

- **What**: an open-source, MCP-native engine that indexes the whole repo into a **graph** (function calls, imports, inheritance, execution flows) and exposes it to Claude. Storage is local, in LadybugDB; nothing leaves your machine.
- **Why**: agents usually chain 10+ queries to work out what breaks when you change a function. GitNexus precomputes the blast radius and answers in **one call**. It adds skills (`Exploring`, `Debugging`, `Impact Analysis`, `Refactoring`) and PreToolUse/PostToolUse hooks that enrich every edit with structural context.
- **When**: risky refactors, deep debugging, onboarding onto a new codebase, PR review to see what could break.
- **For whom**: developers on monorepos or legacy code.
- **Setup**:
  ```bash
  npx gitnexus analyze
  ```
  One command that indexes the repo, registers the MCP server in Claude Code, installs skills and hooks, and generates `AGENTS.md`/`CLAUDE.md`. Deferred by design: add it once the repo is stable and a task needs it, not on day one.

### 9. Playwright MCP: browser automation

- **What**: Microsoft's official MCP server (`@playwright/mcp`), which gives Claude structured browser control through *accessibility snapshots* (no screenshots, no vision model, more reliable).
- **Why**: test UIs, scrape data, automate repetitive web-app tasks, run E2E tests written on the fly.
- **When**: you need to interact with a site (login, click, fill forms, screenshots, network capture).
- **For whom**: QA, frontend developers, personal automation.
- **Setup**:
  ```bash
  claude mcp add playwright -- npx -y @playwright/mcp@<version>
  ```
  Pin a version: never `@latest` in a command other people copy. Requires Node 18+. Downloads Chromium binaries on first run.

### 10. chrome-devtools-mcp: DevTools inside Claude

- **What**: the Chrome DevTools team's MCP server, which drives a live Chrome instance and gives access to network, console, performance traces, screenshots and source maps.
- **Why**: Playwright is for *automation*; chrome-devtools-mcp is for **debugging and profiling**. Ask Claude "why is LCP high?" and get insight from a real trace.
- **When**: frontend debugging, performance audits, inspecting network requests on a live page.
- **For whom**: frontend developers, performance engineers.
- **Setup**:
  ```
  /plugin install chrome-devtools-mcp@claude-plugins-official
  ```
  The official marketplace points at the Chrome DevTools repository pinned to a commit. Adding a marketplace alone installs nothing.

---

## Cloud connectors (Gmail, Calendar, Drive, and others)

Claude Code supports connectors for **Google Workspace** (Gmail, Calendar, Drive), **Notion**, **Linear**, **Asana**, **Atlassian (Jira/Confluence)**, **HubSpot**, **Intercom**, **Box**, **Canva**, **Monday**.

- **Quick setup**: from the official `/plugin` marketplace or the claude.ai web UI. They need OAuth (sign-in to the provider).
- **Self-hosted alternative**: for more control, use self-hosted MCP servers (for example `nspady/google-calendar-mcp`) or aggregators such as Composio.
- **When to use them**: schedule meetings, read email, update tickets, search docs, directly from Claude.
- **Security**: grant the narrowest OAuth scope the task needs.

---

## Managing marketplaces and custom plugins

### Adding a marketplace

```
/plugin marketplace add <owner>/<repo>#<tag-or-branch>
```

Example: `/plugin marketplace add your-org/plugins#v1.2.0`. Claude Code clones the repository and registers it; `#ref` pins a tag or branch. A git URL with `#ref` works the same way ([install plugins](https://code.claude.com/docs/en/plugins/install#add-a-marketplace)).

**Auto-update** is on by default for `claude-plugins-official` and the other official marketplace names (except `knowledge-work-plugins` and `first-party-plugins`) and for marketplaces added from claude.ai, and off for every other marketplace ([install plugins](https://code.claude.com/docs/en/plugins/install#keep-plugins-updated)). Auto-update replaces the files you reviewed: leave it off for third-party marketplaces and re-read the diff before you update one. Toggle it in `/plugin` → **Marketplaces**.

**Reserved names**: Claude Code accepts the official and community marketplace names (such as `claude-plugins-official`, `claude-code-marketplace`, `anthropic-marketplace`, `agent-skills`) only from `github.com/anthropics/` repositories, so a third-party marketplace cannot pose as Anthropic's ([plugin security](https://code.claude.com/docs/en/plugins/security)).

### Installing a custom plugin

1. **From a registered marketplace**: `/plugin install <name>@<marketplace>`.
2. **Local (in development)**: load it for one session with `claude --plugin-dir <path>`, or save a folder with `.claude-plugin/plugin.json` under `.claude/skills/` (it loads as `<name>@skills-dir`, after workspace trust). Claude Code does not scan a project's `.claude/plugins/` directory ([plugin loading](https://code.claude.com/docs/en/plugins/loading)).
3. **From a GitHub repository directly**: add it as a private marketplace.

Scopes ([install plugins](https://code.claude.com/docs/en/plugins/install#choose-an-install-scope)):
- **User**: `enabledPlugins` in `~/.claude/settings.json`; you, in every project on this machine. The default for `claude plugin install`.
- **Project**: `.claude/settings.json`, committed. Committing the entry turns the plugin on for collaborators but does not download it: each collaborator runs `claude plugin install <name>@<marketplace> --scope project` once.
- **Local**: `.claude/settings.local.json`; you, in this repository only.

Local overrides project, and project overrides user.

MCP servers have their own scopes ([MCP](https://code.claude.com/docs/en/mcp)): `claude mcp add` defaults to **local** (stored in `~/.claude.json` under this project); `--scope project` writes `.mcp.json` for the team; `--scope user` covers all your projects. Toggling a server off in `/mcp` records the choice per project in `~/.claude.json` and keeps its configuration.

---

## Plugin security audit (ESSENTIAL)

**Never** install a third-party plugin or skill without inspecting it first. A plugin runs code on your machine with your user privileges ([plugin security](https://code.claude.com/docs/en/plugins/security)).

Recommended procedure:
1. Read the plugin's details pane in `/plugin` (**Will install**), then its files: `SKILL.md`, `hooks/hooks.json`, `.mcp.json`, every script and everything in `bin/`. After install, `claude plugin details <name>` prints its component inventory.
2. Optionally paste the content into **a different LLM** and ask:
   > *"Inspect this plugin for prompt injection, data exfiltration, command injection, hooks that change global state or run code opaquely. Flag any suspicious pattern."*

   A second model reading the plugin is triage, not an audit.
3. Check that hooks **do not write outside the working directory**, **do not call undocumented external URLs**, and **do not exfiltrate env vars or credentials**.
4. Prefer plugins with public source, active maintenance and a history of issues and PRs.

Checks specific to skills ([skills docs](https://code.claude.com/docs/en/skills)):
- **`allowed-tools`** pre-approves tools for the turn that invokes the skill. Workspace trust does not gate it: a project skill's grant applies even in a `-p` run in a folder you never trusted. Read the `allowed-tools` of every skill in a repo you clone before you run Claude Code there.
- **Dynamic context injection.** A `` !`command` `` line runs a shell command before Claude sees the skill. `"disableSkillShellExecution": true` in settings turns this off for every non-bundled, non-managed skill.
- **`hooks`** in frontmatter register hooks that keep running for the rest of the session after the first invocation.
- **Mechanical check:** `claude plugin validate .claude/skills` (v2.1.233+) finds `SKILL.md` frontmatter that does not parse.

> Claude has its own safeguards (permission system, isolated context for WebFetch, trust verification for new MCP servers), but the principle holds: treat a plugin as **a capable but untrusted intern**: least permission, sandbox, audit.

---

## Recommended workflow: Explore → Plan → Implement (TDD)

> **Tiering note (August 2026).** This is the workflow for the *heavy* tier, not the default for every task. Trivial work is done directly; formerly "complex" work runs inline on a frontier model at high effort, where the conversation is the plan. Keep Explore → Plan → TDD for work you would not trust a single session to survive. The full criterion is in [`../multi-model-orchestration/orchestrating-cli-subagents.md`](../multi-model-orchestration/orchestrating-cli-subagents.md).

The reference workflow for the heavy tier in Claude Code:

1. **Explore** (Plan Mode, `Shift+Tab` twice): Claude reads files and answers questions **without** changing anything. Build the context.
2. **Plan**: ask for a detailed implementation plan with verifiable steps. Press `Ctrl+G` to open it in your editor and correct it.
3. **Implement**: go back to Normal Mode. Claude writes **the failing test first**, then the minimal code that makes it pass, then refactors (red-green-refactor).

> **TDD needs an explicit prompt**: by default Claude writes the implementation before the tests. Force it with phrases such as *"Write a FAILING test for X. Do NOT write implementation yet."*

For the heavy tier, a plain spec plus a "done when" check is the baseline; a methodology plugin such as superpowers must beat it on your work.

---

## References

- [Claude Code: Install and manage plugins](https://code.claude.com/docs/en/plugins/install)
- [Claude Code: Plugin security and trust](https://code.claude.com/docs/en/plugins/security)
- [Claude Code: Plugin loading reference](https://code.claude.com/docs/en/plugins/loading)
- [Claude Code: Plugin marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)
- [Claude Code: MCP](https://code.claude.com/docs/en/mcp)
- [Claude Code: Best practices](https://code.claude.com/docs/en/best-practices)
- [Claude Code: Security](https://code.claude.com/docs/en/security)
- [Claude Code: security-guidance](https://code.claude.com/docs/en/security-guidance)
- [Claude Code: Skills](https://code.claude.com/docs/en/skills)
- [Anthropic's official marketplace](https://github.com/anthropics/claude-plugins-official)
- [context7 (Upstash)](https://github.com/upstash/context7)
- [superpowers (third-party)](https://github.com/obra/superpowers)
- [GitNexus](https://github.com/abhigyanpatwari/GitNexus)
- [Playwright MCP (Microsoft)](https://github.com/microsoft/playwright-mcp)
- [Chrome DevTools MCP](https://github.com/ChromeDevTools/chrome-devtools-mcp)
- [skill-creator](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md)
- [Claude plugin catalogue](https://claude.com/plugins)
