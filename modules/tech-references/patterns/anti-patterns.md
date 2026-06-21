# Anti-patterns (each paired with the better pattern)

> The traps that keep showing up in AI/agentic web builds, with the production pattern that fixes each one. The **tell** is the cheapest way to notice you're already in the trap.

| Anti-pattern | Why it hurts | Better pattern |
|---|---|---|
| Two competing cache stories | State diverges; freshness becomes unknowable | One server-state cache, one transient UI store |
| ORM calls scattered across the codebase | Can't swap, mock, or test the data layer | [Repository-everything](production-patterns.md#1-repository-everything-orm-nowhere-else) |
| ORM-generated-and-forgotten migrations | Drift between schema and reality | Hand-authored SQL + committed journal |
| RLS used as application authorization | RLS guards the auto-exposed surface, not your business rules | [Defense-in-depth auth](production-patterns.md#9-defense-in-depth-auth) |
| Loading every tool + every result into context | Context rot, cost, latency | [Retrieve-don't-load](production-patterns.md#7-retrieve-dont-load-at-scale) |
| Shipping on an EOL runtime/lint major | Security + support cliff | Track the watch-list; migrate before EOL |
| Prompt-in-a-loop without state | No memory, no progress, no exit | [Event-driven background work](production-patterns.md#6-event-driven-background-work) |
| Treating an LLM judge as ground truth | A noisy estimate gets reported as a fact | [Boundary parsing](production-patterns.md#2-boundary-parsing) + human spot-checks |

---

## Two competing cache stories

**Why it hurts.** When the same server data lives in a real query cache *and* in an ad-hoc fetch wrapper or a context provider, the two copies drift. One updates after a mutation and the other doesn't, so what the user sees depends on which copy a component happened to read. Freshness stops being a property you can reason about and becomes a coin flip.

**The better pattern.** Pick one server-state cache as the single source of truth for fetched data, and one transient store for UI-only state (open menus, form drafts). Server data invalidates through the query cache; UI state never pretends to be server data. This is the cache discipline behind the [reference stack](../reference-stack.md): one query cache, one UI store, no overlap.

**Tell.** You're holding the same record in two places and writing code to keep them in sync — or you've shipped a "why is this stale?" bug and had to ask which cache the component read from.

---

## ORM calls scattered across the codebase

**Why it hurts.** Once route handlers, components, and jobs each reach for the ORM directly, the data layer has no seam. You can't mock it in a test, you can't swap a slow query for raw SQL without hunting every call site, and a schema change means grepping the whole app. Cross-cutting concerns like tenant scoping and tracing get reimplemented at each call site, inconsistently.

**The better pattern.** [Repository-everything, ORM-nowhere-else](production-patterns.md#1-repository-everything-orm-nowhere-else). The ORM lives inside repository classes behind interfaces; everything else depends on the interface. One place to mock, one place to optimize, one place to enforce scoping.

**Tell.** An import of the ORM or a table object appears in a file whose name isn't a repository — a route handler, a React component, a job runner.

---

## ORM-generated-and-forgotten migrations

**Why it hurts.** Letting the ORM auto-generate migrations and never reading them means the schema in your head, the schema in the migration history, and the schema actually in the database slowly diverge. Auto-generated diffs miss intent (a rename looks like a drop-plus-add and silently loses data), and nobody reviews the SQL that ships to production.

**The better pattern.** Hand-author the SQL and commit a migration journal you actually read in review. Use the ORM's tooling for studio and to *suggest* a diff, but the migration that ships is human-authored and version-controlled, so the database state is always reconstructable from the repo.

**Tell.** Nobody on the team can say what the last migration did without opening the database, or a "drop column" showed up in a diff that was supposed to be a rename.

---

## RLS used as application authorization

**Why it hurts.** Row-level security guards the database's auto-exposed data surface — it's a deny-list on rows, not a model of your business rules. Treating it as *the* authorization layer means complex rules ("a manager can approve but only below their limit, and not their own request") get crammed into row policies where they're hard to read, hard to test, and easy to get subtly wrong. And if any code path reaches the data outside that surface, the rule isn't enforced at all.

**The better pattern.** [Defense-in-depth auth](production-patterns.md#9-defense-in-depth-auth): real authorization in application code where it can be read and tested, with RLS kept as a backstop deny-list so a mistake in app code still can't leak another tenant's rows. Two layers, both enforcing, neither load-bearing alone.

**Tell.** Your business authorization logic lives mostly in row policies, or you'd be exposed the moment a query ran with a service role.

---

## Loading every tool and every result into context

**Why it hurts.** Registering dozens of low-level tools and feeding every intermediate result back into the prompt buries the signal. The model loses the thread (context rot), every turn costs more tokens, and each call gets slower — and more tools means more chances to pick the wrong one. The agent gets dumber as the conversation gets longer.

**The better pattern.** [Retrieve-don't-load at scale](production-patterns.md#7-retrieve-dont-load-at-scale): expose fewer, higher-level tools, and keep large or intermediate results out of context behind a fetch-on-demand handle. The working context stays small and on-task; the model pulls only what the current step needs.

**Tell.** Context length grows every turn regardless of the task, or you're registering more tools to fix behavior instead of fewer.

---

## Shipping on an EOL runtime or lint major

**Why it hurts.** An end-of-life runtime or tooling major stops getting security patches and support. You're one disclosed CVE away from an emergency upgrade under pressure, and the longer you wait the bigger the jump — a forced migration across two majors at the worst possible time.

**The better pattern.** Track a standing watch-list of version lines and their EOL dates (see the [reference stack](../reference-stack.md)), and migrate *before* the cliff, on your schedule, in small steps. Upgrades are routine maintenance, not a fire drill.

**Tell.** A dependency's EOL date is in the past (or this quarter) and nobody has a migration plan, or your CI is pinned to a major that no longer receives patches.

---

## Prompt-in-a-loop without state

**Why it hurts.** Wrapping a model call in a `while` loop and re-prompting until it "looks done" produces an agent with no memory of what it tried, no record of progress, and no principled exit. It repeats failed actions, can't resume after a crash, and either loops forever or stops on a vibe. There's nothing to inspect when it goes wrong.

**The better pattern.** Drive multi-step work through [event-driven background work](production-patterns.md#6-event-driven-background-work) with explicit, durable state: each step is recorded, retries are bounded, and the loop has a real termination condition tied to state rather than to the model's mood. A crash resumes from the last committed step instead of starting over.

**Tell.** Your agent is a bare `while` loop with no persisted step record, or you can't answer "what did it already try, and why did it stop?"

---

## Treating an LLM judge as ground truth

**Why it hurts.** Using a model to grade output (an "LLM-as-judge") gives you a useful but noisy estimate. Reporting that score as if it were a measured fact hides the noise: the judge has its own biases, drifts between model versions, and can be gamed by the very system it's grading. Decisions then rest on a number that looks rigorous and isn't.

**The better pattern.** Treat the judge's output as untrusted model output — [parse it at the boundary](production-patterns.md#2-boundary-parsing), constrain it to a defined shape, and calibrate it against a human-labeled sample before you trust the trend. Report it as an estimate with known error, and keep a human spot-check in the loop for anything consequential.

**Tell.** A judge score is quoted as a hard metric in a decision, with no human-labeled baseline behind it and no note on which model produced it.
