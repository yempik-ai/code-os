# tech-references — Index

> **What this module is.** Grounded engineering references for building production AI/agentic web apps: a reference stack (tech for the whole stack), a flagship tool-orchestration compendium, and a brief-driven workflow that turns the references into an executable plan. Every claim is tied to a version source or a fetched URL — these are *references*, not setups.
>
> **Who it's for.** Engineers and team leads who ship through long-running AI agents — and Claude itself when pointed at this repo to plan work.

---

## What's in here

| File | What it is | Read it when |
|---|---|---|
| [`reference-stack.md`](reference-stack.md) | The **reference stack, layer by layer** (framework → runtime → AI layer → data → infra → dev/test), each choice with rationale and a June-2026 version line. | You're starting a new app, or deciding "what do we build this with?" |
| [`tool-orchestration-best-practices-and-resources.md`](tool-orchestration-best-practices-and-resources.md) | **SOTA tool-orchestration compendium** (June 2026): universal principles → per-provider deep dives (Anthropic, OpenAI, Google, Vercel) → MCP → frameworks → failure modes → evaluation. Adversarially verified, fully cited. | You're designing how an agent uses tools, picks them at scale, or orchestrates multiple agents. |
| [`patterns/`](patterns/README.md) | **Vendor-neutral production patterns & anti-patterns** distilled from real codebase audits — no client names, paths, or SHAs. | You want operational patterns worth copying, and the traps to avoid. |

> The original audits these patterns were distilled from were grounded in private production codebases and are **not** part of this public release.

---

## How to use this module — the documentation-driven brief workflow

The point of these references is **not** to read them cover-to-cover. It's to make Claude plan your next piece of work *grounded in them*, then hand you a brief you execute against. The loop:

```text
┌─ 1. Clone/pull this repo locally (or open it next to your product repo)
│
├─ 2. In Claude Code, point at this folder and ask for a grounded roadmap
│      (use the prompt template below)
│
├─ 3. Claude reads the references, proposes a roadmap WITH reasoning,
│      and flags anything outdated instead of trusting the docs blindly
│
├─ 4. Export the final answer as a brief:  brief-<feature>.md
│
└─ 5. In the PRODUCT repo, start from the brief — it's the spec the
       agentic harness (../agentic-harness/) executes against.
```

### Copy-paste prompt template

```text
I need to build: <X — feature/goal, constraints, deadline>.

Based ONLY on the documentation in tech-references/ (reference-stack.md,
tool-orchestration-best-practices-and-resources.md, and patterns/):

1. What's the recommended roadmap and WHY? Justify every technical choice
   by anchoring it to a file/section of this documentation.
2. Don't assume no better option exists: if you know a more suitable or
   more recent one, propose it explicitly.
3. If the documentation is outdated (EOL versions, superseded libraries,
   deprecated patterns), FLAG it with the delta and the updated source.
4. List the risks and the open decisions that need my input.

Output: a Markdown brief ready to save as brief-<feature>.md, with sections
Roadmap / Rationale / Stack / Risks / Open decisions.
```

### Why this works

- **Grounded, not hallucinated.** Claude reasons from verified version lines and real production patterns, not training-data folklore.
- **Self-correcting.** The "flag if outdated" instruction turns the reference into a *living* baseline — the stack drifts weekly (see the watch-list in `reference-stack.md`), and Claude has Context7 + web search to check.
- **One artifact, two repos.** The brief is the contract between *thinking* (here) and *doing* (the product repo). The agentic harness in [`../agentic-harness/`](../agentic-harness/index.md) is built to execute exactly this kind of spec → plan → execute brief.

---

## Notes for Claude / agents

- **These are references, not specs to install.** Don't "set up" anything from this folder; cite it.
- **Preserve the grounding discipline.** When extending any file here, tie every non-trivial claim to a version source or fetched URL, and keep the **peer-reviewed / preprint / vendor-marketing** distinction explicit (see the caveats ledger in `tool-orchestration-best-practices-and-resources.md`).
- **`reference-stack.md` is dated.** Treat its versions as a June-2026 snapshot; re-verify "latest" against current docs before pinning, and flag deltas.
