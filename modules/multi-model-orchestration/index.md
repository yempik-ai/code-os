# multi-model-orchestration — Index

> **What this module is.** The pattern that changed how code-os works: Claude
> Code as the durable orchestrator, other CLI agents (Codex today, grok or
> ollama tomorrow) spawned as disposable subagent workers. One brain that
> holds context and taste; many hands that burn someone else's meter.
>
> **Who it's for.** Engineers already running Claude Code on real codebases
> who want more throughput and a second, uncorrelated perspective without
> multiplying their own token bill.

---

## What's in here

| File | What it is | Read it when |
|---|---|---|
| [`orchestrating-cli-subagents.md`](orchestrating-cli-subagents.md) | The full pattern: roles, model tiering, the run-path convention, the Codex runbook, the four failure modes, and how to wrap any other CLI. | You want to run this today. It is the whole module. |

## The idea in five lines

1. The orchestrator (Claude Code) owns the session: context, memory, taste, and the final accept/reject call.
2. Subagents are other vendors' CLI agents run headless: explorers, implementers, reviewers (adversarial or second-opinion).
3. Each run is three files the orchestrator allocates: a prompt file in, a final-message file out, a log nobody reads unless the run fails.
4. Ceremony is tiered: most work is one frontier model inline at high effort; the full spec, plan, TDD loop is reserved for work that used to be impossible.
5. The model is a commodity. The harness, the memory, and the acceptance discipline are what you own.

---

*In produzione, non in slide. · by yempik. · maintained by Simone Bova*
