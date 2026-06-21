# Export-for-Second-Brain Prompt

> **For the operator.** `/crystallize` captures reasoning that happened **inside a Claude Code session**. This file is the other half: a prompt you paste into **Claude.ai (web)**, **ChatGPT**, **Gemini**, or **NotebookLM** to extract a conversation or notebook into a single markdown file, shaped so `/ingest` can file it cleanly.

## When to use this

- You had a long Claude.ai / ChatGPT / Gemini conversation that produced durable insight and you want it in the vault.
- A NotebookLM notebook has synthesised something you want to preserve as a source.
- You're migrating historical chat threads from one platform into the vault.

**When not to use.** Trivial back-and-forth, one-shot Q&A lookups, anything secret (health, credentials, financial). Those don't belong in the vault regardless of platform.

## How to use this

1. Open the third-party platform where the conversation lives.
2. Make sure the full conversation is loaded in the UI (scroll to the top if the platform lazy-loads).
3. Copy the block between `===== BEGIN PROMPT =====` and `===== END PROMPT =====` below and paste it as your next message into that platform.
4. The platform will reply with a single markdown document.
5. **Save the reply.** Copy the entire markdown output into a new file at `~/second-brain/inbox/<platform>-<short-slug>-YYYY-MM-DD.md`. Example: `inbox/chatgpt-pricing-model-debate-2026-04-23.md`.
6. In Claude Code (from the vault root): `/ingest inbox/`. The ingest skill will process the file like any other source — finding the wiki pages it touches, extracting claims, filing the raw under `raw/conversations/YYYY-MM/`, committing.

**NotebookLM note.** NotebookLM isn't a chat — it's a notebook with sources and synthesised answers. Paste the prompt into the notebook's chat pane; it will export the notebook's synthesised knowledge plus the source citations. The resulting markdown lists the underlying sources so `/ingest` can flag which raw documents you may want to pull in separately.

---

===== BEGIN PROMPT =====

I need you to export the full substantive content of this conversation (or, if this is a NotebookLM notebook, the notebook's synthesised knowledge and source inventory) into a single markdown document that I will file in a long-lived personal knowledge vault. Another system will ingest your output — so the format below is a **contract**, not a suggestion. Follow it exactly.

## Invariants (do not violate)

1. **Preserve substantive detail.** This is a high-fidelity export, not an executive summary. A 90-minute conversation should compile to 120–250 lines of markdown. If you strip out a decision, a framework, a named entity, a number, a rationale, a verbatim turn-of-phrase that carries reasoning — you've failed. The downstream vault explicitly rejects shallow captures.
2. **Do not invent content.** If a section below has no material in the conversation, write `_(none in this conversation)_` and move on. Fabrication, paraphrase-drift, or adding plausible-sounding content I didn't say is the single worst failure mode. If you're unsure whether something was actually said vs implied, prefer `_(implied but not stated: …)_` over silently asserting it.
3. **Attribute.** When a specific claim came from me (the user), tag it `[user]`. When it came from you (the assistant), tag it `[assistant]`. When it came from a cited source inside the conversation (a paper, article, link I pasted), tag it `[source: <title or URL>]`. Unattributed claims are worse than missing ones.
4. **Preserve numbers and names verbatim.** Entities, people, org names, product names, metrics, dates, €/$ figures, percentages, code identifiers — copy as they appeared. Do not round, translate, or normalise.
5. **Output a single markdown document.** No preamble, no "Here's your export:", no trailing commentary. The document starts with `---` (YAML frontmatter) and ends with the last content line. I will copy your entire reply verbatim into a file.

## Output format — follow this exactly

```markdown
---
type: conversation
source_platform: [claude.ai | chatgpt | gemini | notebooklm | other]
source_title: "<the conversation's title or a ≤80-char descriptive title you generate>"
source_url: "<permalink if the platform exposes one, else empty string>"
participants: ["user", "assistant"]
conversation_date_range: "YYYY-MM-DD to YYYY-MM-DD"
exported_at: "YYYY-MM-DD"
domain_hint: "<your best guess at the life-compartment: work, startup, freelance, tech-news, personal, or global. If unclear, write 'unclear'.>"
status: active
tags: [kebab-case, kebab-case]
---

# <Same as source_title>

## Summary

2–4 dense paragraphs covering: what we were trying to figure out, how the conversation evolved, what we converged on (or failed to converge on). A reader who never saw the conversation should understand the arc.

## Key claims

Bulleted list of the load-bearing claims the conversation produced. Each claim is one sentence, standalone, with attribution in brackets. Aim for 5–20 claims depending on conversation length — do not pad, do not compress.

- Claim one, as a standalone proposition. [user]
- Claim two, with a specific number or name preserved. [assistant]
- Claim three, citing an external source introduced mid-conversation. [source: Karpathy LLM Wiki gist, https://…]

## Decisions

Any choice made during the conversation. Format: **Decision:** one line. **Rationale:** one line. **Alternatives considered:** one line (or `_(none stated)_`).

- **Decision:** We'll use X for Y. **Rationale:** Z. **Alternatives considered:** W, rejected because V. [user]

If no decisions were made, write `_(no decisions in this conversation)_`.

## Action items

Checkboxes of anything I committed to do, or that I said someone else would do. One owner per item; if no owner was named, write `[owner: unspecified]`. Include deadlines verbatim.

- [ ] Write the spec for the pricing tier system. [owner: me] [by: end of week]
- [ ] Ask <colleague> about the Q2 budget. [owner: me] [by: unspecified]

If none, write `_(no action items in this conversation)_`.

## Open questions / unresolved threads

Questions we raised but did not answer. These feed the vault's open-threads tracker.

- Does X actually hold when Y is at scale?
- Who owns the decision on Z?

If none, write `_(all threads resolved)_`.

## Entities mentioned

Flat list, one line per entity, in the form `- <name> — <role/type>, <one-line context>`. Include people, organisations, products, projects, places, papers. If the same entity is mentioned many times, list once.

- Karpathy — person, introduced the "LLM wiki" concept that shaped our file-format decision.
- Acme — organisation, the employer context anchoring the discussion.

If none, write `_(none)_`.

## Concepts introduced or clarified

Any framework, distinction, term, or mental model that crystallised during the conversation. One line each.

- **Hot cache vs cold cache:** the distinction between per-session recent-context files and the full wiki.
- **30-minute stall rule:** stub work and move on rather than letting perfectionism block compounding.

If none, write `_(none)_`.

## Verbatim excerpts worth preserving

When a turn of phrase carries the reasoning and paraphrasing would lose it — quote it. Each excerpt: 1–6 lines, attributed, with 1 line of context before. Aim for 0–5 excerpts per conversation. Do not dump the whole transcript here.

> **Context:** we were debating whether to ingest aggressively or curate tightly.
>
> [user] "The vault is not a museum. If it's not compounding, it's rotting."

## Contradictions with anything stated

Did the conversation flag a contradiction with something I had said earlier (in this conversation or in a linked document)? Or between two sources we discussed?

- **Contradiction:** Claim A from source X conflicts with claim B from source Y on the question of Z. Unresolved.

If none, write `_(none flagged)_`.

## Next steps / follow-on work

What I said I'd do next, or what the conversation implies should happen next. Distinct from action items in that these can be fuzzy ("think more about X") rather than committed tasks.

- Explore whether the 3-citation trend-promotion rule should apply to personal-domain clusters too.

If none, write `_(none)_`.

## Raw transcript (optional — include only if the conversation is ≤20 turns OR is a high-signal short exchange)

If the conversation is short enough to preserve in full, include it here under a `### Turn N [role]` heading per turn. If the conversation is long (>20 turns), omit this section entirely — the sections above carry the signal, and dumping thousands of lines of raw transcript dilutes the export.

---

## NotebookLM variant (only if this is a NotebookLM notebook)

Replace the "Raw transcript" section with:

## Source inventory

Flat list of every source the notebook draws on. One line each.

- <source title> — <type: pdf | article | transcript | webpage>, <URL or "local file">, <1-line summary of what it contributes to the notebook>.

## Notebook-generated syntheses

The notebook's own generated answers / briefings / study guides worth preserving. Each: a heading, 1–3 paragraphs of the synthesis content, and a line naming which sources grounded it.
```

## Process these final checks before replying

- Did you preserve numbers, names, and specific claims verbatim?
- Did you attribute every claim?
- Is anything in your output something that wasn't actually in the conversation? If yes, delete it.
- Is the output a single markdown document with no surrounding prose?
- If this is a long conversation (>30 turns), did you resist the urge to summarise too aggressively? The target is 120–250 lines for a substantive conversation, not 40.
- Include raw snippets where appropriate for grounding

Reply with the markdown document only. Begin with `---` and end with the final content line. No preamble, no closing remarks.

===== END PROMPT =====

---

## After the export lands in `inbox/`

1. **Sanity check before ingest.** Open the saved file. Spot-check 3 claims against what you actually said. If the export fabricated or drifted, delete it and re-run the prompt with the amendment *"Your previous export invented content in section X. Redo, strict."* The invariants above catch most drift but not all.
2. **Rename to the filing convention.** `inbox/<platform>-<short-slug>-YYYY-MM-DD.md`. The `/ingest` skill uses the filename to pick a raw destination and seed the source page title.
3. **Run `/ingest inbox/`.** The conversation will be processed like any other source: claims become wiki updates, entities become people/entity pages, open questions land in `wiki/questions/` or `OPEN-THREADS.md`, the raw file moves to `raw/conversations/YYYY-MM/`, and a commit lands with `ingest(<domain>): <title> — N pages (+M new)`.
4. **Source page.** `/ingest` creates a `wiki/sources/<slug>.md` for the conversation. Because the source is a 3rd-party chat — not something a reader can cite without access to your account — the source page's `source_path:` frontmatter points at the local `raw/conversations/YYYY-MM/<file>.md`. Provenance holds.

## Platform-specific notes

- **Claude.ai (web).** Paste the prompt as a follow-up message. Claude web preserves the full conversation context; the export will be high-fidelity. If the conversation used Projects with attached knowledge, ask it to list the project's knowledge sources under "Source inventory".
- **ChatGPT.** Paste as a follow-up. For conversations with custom GPTs or memory enabled, the export will reflect whatever was actually surfaced in the chat — memory contents GPT didn't verbalise won't be captured. That's fine; you only want what was discussed.
- **Gemini.** Works the same. Gemini is more prone to mild paraphrase-drift; double-check direct quotes in the "Verbatim excerpts" section.
- **NotebookLM.** Paste into the notebook chat pane. NotebookLM will use the NotebookLM-variant section (source inventory + syntheses). The underlying PDFs/articles the notebook was built on are **not** exported — you'll need to drop those into `inbox/` separately if you want them ingested as first-class sources.

## Failure modes

- **Export looks summary-sized (40 lines for a 90-min conversation).** The invariant against shallow capture was ignored. Re-run with: *"Your previous export was too compressed. The target is 120–250 lines for a substantive conversation. Preserve named entities, numbers, rationales, and the reasoning arc. Redo."*
- **Export invents content.** Rerun with the stricter amendment above. If it happens twice, the model is leaking its own knowledge into the export — switch platforms (Claude.ai tends to be most faithful here, GPT and Gemini more prone to drift on long threads).
- **Export omits attribution.** Rerun with: *"Every claim needs [user], [assistant], or [source: …] attribution. Redo."*
- **Export includes preamble or closing ("Here's your markdown export:")**. Strip manually; not worth a rerun. Note this is why step 5 in "How to use" says to copy *the markdown document* into the file, not the entire reply.

---

*This prompt is intentionally platform-agnostic. If a new platform (Perplexity, Poe, etc.) enters your workflow, it'll work there too — the invariants and output contract are the load-bearing parts. For the vault's own in-session equivalent, see `/crystallize` (documented in `second-brain-tutorial.md`).*
