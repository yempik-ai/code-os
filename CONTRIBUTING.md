# Contributing to `code-os`

> `code-os` is **structure + prompts + grounded references**, not an app. There is nothing
> to `npm install` and no runtime to ship. Contributions that keep it simple, generic, and
> drop-in adoptable are welcome.

`code-os` is the engineer's counterpart to [`cowork-os`](https://github.com/yempik-ai/cowork-os) —
two systems, one hat: yempik. This guide covers contributing to `code-os` (engineers running
agents on real codebases). For the business/ops side, see the `cowork-os` repo.

Read [`index.md`](index.md) first for the full repo map and the five-module model.

---

## What good contributions look like

- **Harness / vault discipline that is genuinely reusable.** New survival-kit files, hooks, or
  `CLAUDE.md` patterns for `modules/agentic-harness/` or `modules/second-brain/` — but only if
  they generalize across any git repo or any vault, not just yours.
- **A tighter spec → plan → execute loop.** Improvements to the bootstrap flow
  (`DAY-0-CHECKLIST.md` → `BOOTSTRAP.md` → spec → tutorial), the `docs/agent-memory/` survival
  kit, or the `docs/superpowers/` loop the harness installs.
- **Better-grounded references.** Additions to `modules/tech-references/` that come with a
  source: a changelog, a release note, a version tag, or a fetched URL. Folklore without a
  citation does not land here (see Grounding, below).
- **New curated entries.** Skills/plugins/MCPs in `modules/skills-guide/useful-skills.md`,
  tools in `modules/ai-tools/useful-ai-tools.md`, or people in
  `modules/personas/people-to-follow.md` — with a one-line reason that earns the slot.
- **Clearer wording.** Sharper, more concrete English. Cut hype, name the real problem, show
  the fix.

Name a real problem, then show the fix. Results over adjectives.

---

## Rules

### 1. Never commit private or employer-derived material

This is the hard line, and it is non-negotiable.

- **No real client or employer names**, no client/employer codebase audits, no file paths or
  commit SHAs from private repos, no internal auth domains.
- **No salary, rates, or financials** — yours or anyone's.
- **No API keys, tokens, or credentials.**
- **No PII** — yours or third parties'.

Material produced in an employment context may be the **employer's IP**, not yours to publish.
When in doubt, it does not go in the public tree.

Private material goes in **`_private/`**, which `.gitignore` blocks (along with `*-grounded.md`
and `.remember/`). If your contribution started life as a real-world artifact, genericize it:
replace real names, rates, and provenance with neutral placeholders before it goes anywhere near
the public tree.

### 2. Keep modules self-contained

Each of the five modules under `modules/` stands alone — someone uses one without the others.
**Do not cross-wire one module's spec into another.** The `agentic-harness` spec and the
`second-brain` spec share a family resemblance (both build on the same agent-memory
discipline), but each is tailored to its target — a codebase vs. a vault. Keep them that way.
If you find yourself reaching across modules, you probably want a new self-contained doc, not a
dependency.

### 3. Preserve the grounding discipline (`tech-references`)

`modules/tech-references/` is citation-heavy on purpose. Every public claim is tied to a
**version source or a fetched URL**. If you add or change a reference:

- Cite the source inline — a changelog, release note, version tag, or URL.
- Keep the **peer-reviewed / preprint / vendor-marketing** distinctions explicit. A vendor blog
  post and a peer-reviewed result are not the same kind of evidence; label them so.
- No superlatives without a source. "Best/fastest/first" needs a number, and the number needs a
  citation.

### 4. Preserve the deferred-tool discipline

Code-intelligence tooling, custom memory layers, and speculative MCPs are **deferred off Day 1**
by design — earned once the repo is stable, never imposed on a fresh setup. If a contribution
proposes wiring one of these into the Day-0 path of either setup module, push back. Keep the
first run lean.

### 5. Format

- **Markdown only.** No binaries, no images-as-source, no `.DS_Store`.
- **English only.** `code-os` is English-first. The single exception is the brand-claim line —
  "In produzione, non in slide." — which stays as-is where a colophon is natural.
- Keep the **spec → checklist → bootstrap → tutorial** shape intact for the two setup modules.
- One module per pull request when you can. It keeps review honest.

---

## Before you open a pull request

1. Re-read the diff yourself as if you were a stranger seeing the repo for the first time. Would
   any of it leak a private name, a rate, a key, or an employer artifact? If yes, it goes in
   `_private/`.
2. Confirm `git check-ignore _private` reports the path as ignored before you commit anything
   near it.
3. Check that every relative link and file reference you added points at a file that exists.
4. **Open an issue first for anything large** — a new module, a structural rewrite, a new
   reference subsection — so we can agree on the shape before you write it.

File issues and pull requests at <https://github.com/yempik-ai/code-os>. Reach the maintainer: [Simone Bova](https://www.linkedin.com/in/simone-bova/) · simone@yempik.ai.

---

<sub>In produzione, non in slide. · by yempik. · maintained by Simone Bova</sub>
