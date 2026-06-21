# code-os (plugin)

The installable, runnable core of [`code-os`](https://github.com/yempik-ai/code-os) — the engineer's operating system for Claude Code. Install it once and Claude Code gets an always-on core skill plus five slash commands that drive the work. No manual setup, no files to copy.

`code-os` is the builder's half of a pair with [`cowork-os`](https://github.com/yempik-ai/cowork-os) (Raffaele Zarrelli's, for business and ops teams). Two founders, two systems, one hat: yempik. This plugin is the Claude Code surface.

Built by yempik. for Claude Code. MIT.

## Install

In Claude Code:

```bash
# Add the marketplace
/plugin marketplace add yempik-ai/code-os
# Install the plugin
/plugin install code-os@code-os
```

That's it. The core skill loads automatically; the commands are available immediately.

## What you get

An always-on **core skill** plus five slash commands.

| Name | What it does |
|---|---|
| `code-os-core` *(skill, always on)* | Keeps the agent honest on real work: separates facts from assumptions, refuses to claim work done without verification, keeps deferred tools deferred until the repo is stable, and routes you to the right module for the job. |
| `/code-os:install` | Guided router. Asks what you want to do, then routes you to the right module and command. |
| `/code-os:harness` | Drives the agentic-harness install into your current repo: survival-kit memory, the spec→plan→execute loop, enforcement hooks, and a tailored `CLAUDE.md`. |
| `/code-os:second-brain` | Drives the second-brain setup: an Obsidian knowledge vault run from Claude Code. |
| `/code-os:skills` | Surfaces the curated skills / plugins / MCP guide and walks the prompt-injection security-audit procedure before you install anything. |
| `/code-os:stack` | The brief-driven tech-references workflow: reference stack plus tool-orchestration patterns, turned into a plan. |

The harness and the second brain are real setups, not chat — each starts from a human pre-step checklist, runs a paste-in bootstrap prompt, and reads an authoritative spec during the work. The commands drive those flows; you stay in the loop.

## Plugin vs full kit

This plugin is the **runnable core**: the core skill and the five commands. The [full repository](https://github.com/yempik-ai/code-os) also ships the complete module folders under `modules/`, the `DAY-0-CHECKLIST.md` → `BOOTSTRAP.md` setup prompts, and the authoritative specs the agent reads during setup (the ~1,200-line `KNOWLEDGE_MGMT_SETUP_SPEC.md` and `second-brain-spec.md`).

Use the plugin to drive the work from inside Claude Code. Clone the repo when you want the full scaffold — every module file, every bootstrap prompt, every spec — to read, copy, or adapt by hand.

## Customize it

These are field-tested starting points, not a finished house. They get more useful when you adapt them to your repo: your stack, your conventions, your `CLAUDE.md`. Edit the command and skill markdown files in this plugin, or run `/code-os:install` and let it route you to the module that fits, then walk the setup with your real answers.

---

<sub>In produzione, non in slide. · by yempik. · maintained by Simone Bova</sub>
