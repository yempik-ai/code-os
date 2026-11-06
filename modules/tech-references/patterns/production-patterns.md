# Production patterns worth lifting

> Patterns that earned their place in shipped AI/agentic web apps. Each one is here because skipping it caused a specific, recurring failure. Lift them into a new build as defaults; the **When to skip it** line tells you where the cost outweighs the payoff.

---

## 1. Repository-everything, ORM-nowhere-else

**What.** Keep every ORM call inside repository classes that sit behind interfaces. The rest of the app — route handlers, services, background jobs — depends on the interface, never on the ORM. The repository is the only code that knows a table exists.

**Why it pays off.** When the ORM leaks into handlers and components, you cannot mock the data layer in a test, swap a query for a raw SQL statement, or change the schema without grepping the whole codebase. Centralizing data access also gives you one place to add caching, tenant scoping, and tracing instead of scattering them across call sites.

**Minimal generic example.**

```
interface UserRepository {
  findById(id): User | null
  save(user): void
}

// One implementation knows about the ORM. Nothing else does.
class DrizzleUserRepository implements UserRepository { ... }

// Callers depend on the interface:
function loadProfile(repo: UserRepository, id) {
  return repo.findById(id)
}
```

Folder shape: `db/schema.ts` (tables) · `repositories/*.ts` (the only ORM consumers) · everything else imports repositories.

**When to skip it.** A throwaway script or a single-table prototype with no tests and no second data path. The moment you write the second query against the same table, introduce the interface.

---

## 2. Boundary parsing

**What.** Parse and validate untrusted input at the system's edges — HTTP bodies, query params, webhook payloads, tool arguments, model output, third-party API responses — with a schema. Once data is parsed, it carries a trusted type and you never re-validate it internally.

**Why it pays off.** Validation sprinkled through business logic is both incomplete and redundant: some paths check, some don't, and the type system can't tell them apart. Parsing once at the boundary means a malformed payload fails loudly at the door with a clear error, and everything downstream operates on data the compiler already trusts.

**Minimal generic example.**

```
const Body = z.object({ email: z.email(), age: z.number().int() })

async function handler(req) {
  const parsed = Body.safeParse(await req.json())
  if (!parsed.success) return Response.json(z.treeifyError(parsed.error), { status: 400 })
  // parsed.data is now typed and trusted everywhere below
  return Response.json(await service.register(parsed.data))
}
```

`safeParse` turns bad input into a 400; an uncaught `parse` throw usually surfaces as a 500. Zod 4 deprecates `z.string().email()` for `z.email()` and `.format()` for `z.treeifyError()` ([Zod 4 changelog](https://zod.dev/v4/changelog), vendor docs).

Apply the same rule to model output: the LLM is an untrusted boundary. Parse its JSON before you act on it, and treat text it read from the web or a document as data, never as instructions.

**When to skip it.** Data that never crossed a trust boundary — values you constructed yourself in the same process. Don't re-parse your own internal objects on every function call.

---

## 3. Schema-first contracts

**What.** Write one schema and derive everything from it: the static type, the runtime validator, and the structured-output constraint you hand to the model. The schema is the single source of truth for the shape of a thing.

**Why it pays off.** When types, validators, and the model's output format are authored separately, they drift. A field gets added to the type but not the validator; the model returns a shape the validator rejects. One schema means a change propagates to all three at once, and "what the model is allowed to return" is the same object as "what the code expects."

**Minimal generic example.**

```
const Invoice = z.object({
  total: z.number(),
  currency: z.enum(["USD", "EUR"]),
})

type Invoice = z.infer<typeof Invoice>                               // static type
Invoice.parse(payload)                                                // runtime validation
generateText({ model, prompt, output: Output.object({ schema: Invoice }) }) // structured-output constraint
const InvoiceResponse = Invoice.pick({ total: true })                 // boundary projection
```

Derive each boundary's shape (API response, database row, model input) as an explicit projection of the one schema, not a hand-written copy. `generateObject`/`streamObject` are deprecated since AI SDK 6 in favour of `generateText`/`streamText` with `output` ([AI SDK 6 migration guide](https://ai-sdk.dev/docs/migration-guides/migration-guide-6-0), vendor docs).

**When to skip it.** A type that exists only at compile time and never touches a runtime boundary or a model — a pure internal helper shape can stay a plain type.

---

## 4. Human-in-the-loop via atomic DB RPC

**What.** When an action needs human approval before it executes, gate it behind a single database function (RPC/stored procedure) that checks the approval state and performs the write in one transaction. The approve-and-execute step is atomic: either the action was approved and it ran, or neither happened.

**Why it pays off.** Approval logic split across application code — read the pending action, check it's approved, then write — has a race window. Two clicks, a retry, or a concurrent worker can double-execute or execute something that was just rejected. Pushing the check-and-act into one transaction makes "approved exactly once, executed exactly once" a property the database enforces, not a hope.

**Minimal generic example.**

```sql
create function approve_and_execute(p_action_id uuid)
returns void language plpgsql as $$
begin
  update pending_action
     set status = 'executed', approved_by = app_user_id(), executed_at = now()
   where id = p_action_id
     and status = 'pending'                             -- atomic guard: one caller wins
     and app_can_approve(app_user_id(), p_action_id);   -- placeholders for your authz

  if not found then
    raise exception 'action % is not pending or not approvable', p_action_id;
  end if;

  -- perform the gated write in the same transaction
  insert into ledger (...) values (...);
end $$;

revoke execute on function approve_and_execute(uuid) from public;
grant execute on function approve_and_execute(uuid) to approver_role;
```

Derive the approver from the session, never from a parameter: a caller can pass any id. Postgres grants `EXECUTE` on new functions to `PUBLIC` by default, so revoke it and grant only the roles that may approve ([Postgres privileges](https://www.postgresql.org/docs/current/ddl-priv.html), vendor docs).

**When to skip it.** Actions that are idempotent and reversible (re-running causes no harm), or approvals with no concurrency — a single operator, one action at a time, where a double-execute is impossible.

---

## 5. Provider-wrapper pattern

**What.** Put every model or external provider behind a thin adapter that exposes your own interface, and record each request and response on the way through. The app calls your adapter; the adapter calls the vendor and records both sides.

**Why it pays off.** Vendor SDKs change shape, providers get swapped, and models get deprecated — a thin adapter contains that blast radius to one file. The request/response record gives you replay (re-run a failed call without the live API), audit (what exactly did the model see and return), and debugging (reproduce a bad output from stored bytes instead of guessing).

**Minimal generic example.**

```
interface CompletionProvider {
  complete(req): Promise<Result>
}

class VendorAdapter implements CompletionProvider {
  async complete(req) {
    const raw = await vendor.call(req)
    await store.put({ request: redact(req), response: redact(raw) })  // for replay/audit
    return normalize(raw)                                             // your shape, not theirs
  }
}
```

Keep the record minimal: redact personal data and secrets before it is written, encrypt what stays sensitive, and set a retention period. A verbatim prompt is personal data you later have to erase. Keep model ids in one config module.

**When to skip it.** A one-off integration you'll never replay or swap, where the vendor shape is already exactly what you want. But a redacted I/O record is cheap insurance — drop the adapter before you drop the record.

---

## 6. Event-driven background work

**What.** Move slow, retriable, or fan-out work out of the request path into a background pipeline driven by events. Run one pipeline per concern, key concurrency per tenant so one customer can't starve another, and make retries explicit rather than implicit.

**Why it pays off.** Doing slow work inline ties up request capacity, couples user-facing latency to third-party speed, and loses the work entirely if the request dies mid-flight. An event pipeline gives you durability (the event survives a crash), backpressure (concurrency limits per tenant), and honest retry semantics instead of a user mashing refresh.

**Minimal generic example.**

```
// Request path just emits an event and returns.
emit("invoice.received", { tenantId, invoiceId })

// A separate function consumes it, scoped per tenant. Each step's result is recorded.
on("invoice.received", {
  concurrency: { key: "event.data.tenantId", limit: 5 },  // an expression, evaluated per event
  retries: 3,
}, async ({ event, step }) => {
  const parsed = await step.run("parse", () => parseInvoice(event.data.invoiceId))
  await step.run("book", () => book(parsed, { idempotencyKey: event.id }))
})
```

Record each step's result, so a crash or retry resumes from the last completed step instead of starting over. Every side effect takes an idempotency key, because retries will happen. An agent loop runs the same way: steps recorded, retries bounded, a termination condition tied to state. The concurrency key shape follows Inngest's CEL expression ([Inngest concurrency](https://www.inngest.com/docs/guides/concurrency), vendor docs); map it onto your runner.

**When to skip it.** Work that is fast, must complete synchronously for the user to proceed, and has no retry value (e.g. a cheap in-memory transform). Don't add a queue to defer a 2ms operation.

---

## 7. Retrieve-don't-load at scale

**What.** Keep large documents and intermediate tool results out of the model's context. Store them, hand the model a reference or a summary, and let it fetch the full content on demand through a tool only when it actually needs it.

**Why it pays off.** Stuffing every document and every prior tool result into context causes context rot (the model loses the thread in a wall of text), inflates token cost on every turn, and slows each call. Retrieval keeps the working context small and relevant: the model pulls the one chunk it needs instead of re-reading everything it has ever seen.

**Minimal generic example.**

```
// Don't: dump the whole 50-page doc into the prompt.
// Do: store it, expose a fetch tool, pass a handle.
tools: {
  getSection: ({ docId, section }) => store.read(docId, section),
}
// The model receives "docId: 7, sections: [intro, pricing, terms]"
// and calls getSection only for what the task needs.
```

Retrieval is authorised (the caller may read it), bounded (a limit on results), and carries provenance (source and date), so the answer can cite it and admit gaps.

**When to skip it.** Small inputs that fit comfortably and are needed every turn — a short system prompt, a handful of records. Retrieval adds a round-trip; don't pay it for data the model always needs anyway.

---

## 8. Docs-as-code enforced in CI

**What.** Treat specs, architecture decisions, and other knowledge-layer artifacts as code that lives in the repo, and have CI fail when they go stale. A change that touches a contract but not its spec, or adds a decision-shaped change with no ADR, doesn't merge.

**Why it pays off.** Documentation that isn't enforced rots, and rotted docs are worse than none — they mislead. Wiring the knowledge layer into CI makes "the docs match reality" a merge gate rather than a good intention, so the spec is trustworthy enough to onboard a human or prime an agent from.

**Minimal generic example.**

```
# CI step: fail if a public contract changed without its spec.
- run: check-contracts-have-specs   # diff schema/ against docs/specs/
- run: check-adr-present            # flag decision-shaped diffs missing an ADR
- run: lint-docs                    # links resolve, required sections present
```

The point isn't a specific tool; it's that the knowledge layer is checked, not just the build.

**When to skip it.** A solo prototype with no second reader and no agents consuming the docs. The payoff scales with the number of people and tools that trust the documentation.

---

## 9. Defense-in-depth auth

**What.** Enforce the same authorization rule in more than one layer, so a single misconfiguration doesn't open the door. A request is checked at the edge (proxy/redirect) and again at the destination (callback/handler), and both must agree.

**Why it pays off.** Auth that lives in exactly one place fails open the moment that place is bypassed — a route that skips the middleware, a direct call that dodges the redirect. Repeating the check at the boundary and at the handler means an attacker has to defeat every layer, and a refactor that drops one guard still leaves another standing.

**Minimal generic example.**

```
// Layer 1 — edge: unauthenticated requests never reach the app.
proxy: if (!session(req)) redirect("/login")

// Layer 2 — handler: re-check identity and authorization on the action itself.
async function handler(req) {
  const user = requireUser(req)            // not just "is logged in"
  assert(user.canAccess(resource))         // business-rule authz, in app code
}
```

Keep real authorization in application code; don't lean on an auto-exposed data-layer surface for business rules.

**When to skip it.** Truly public, read-only, non-sensitive endpoints with nothing to protect. Everywhere identity or ownership matters, layer it.

---

## 10. Telemetry that respects the runtime

**What.** Register tracing conditionally, in the runtimes where it's supported, instead of importing an instrumentation stack unconditionally. Turn on the spans that earn their keep — model calls, tool invocations, background steps — and let the rest stay off.

**Why it pays off.** Instrumentation that assumes one runtime breaks builds or balloons cold-start when it lands somewhere it doesn't fit. Conditional registration keeps the edge fast and the build green while still giving you the spans that matter — a slow tool call or a retried model request shows up as a span instead of a mystery.

**Minimal generic example.**

```
// instrumentation.ts — Next.js calls register() in every runtime.
export async function register() {
  if (process.env.NEXT_RUNTIME === "nodejs") await import("./instrumentation-node")
  if (process.env.NEXT_RUNTIME === "edge") await import("./instrumentation-edge")
}

// AI SDK 7: register @ai-sdk/otel once in instrumentation-node;
// model and tool calls then emit spans by default.
```

AI SDK 7 moved OpenTelemetry out of core into `@ai-sdk/otel` and makes telemetry opt-out once registered; drop `experimental_telemetry` from calls ([AI SDK 7 migration guide](https://ai-sdk.dev/docs/migration-guides/migration-guide-7-0), vendor docs; [Next.js instrumentation](https://nextjs.org/docs/app/guides/instrumentation), vendor docs).

**When to skip it.** Local-only throwaway code, or a single runtime where unconditional setup genuinely can't break anything. The conditional guard costs nothing, though — it's usually worth keeping.

---

## 11. Hand-authored migrations

**What.** Write every migration as SQL by hand, commit it, and review it like code. The ORM's tooling may suggest a diff; the migration that ships is human-authored. Every schema change goes through a migration.

**Why it pays off.** Auto-generated diffs miss intent: a rename looks like drop-plus-add and silently loses data. Unread migrations let the schema in your head, in the history, and in the database drift apart.

**Minimal generic example.**

```sql
-- 0042_rename_customer_name.sql — expand, migrate, contract across three deploys
alter table customer add column full_name text;                 -- 1. expand
update customer set full_name = name where full_name is null;   -- 2. migrate (backfill)
-- alter table customer drop column name;                       -- 3. contract, after no code reads it
-- Rollback: drop column full_name (safe until step 3 ships).
```

- Ship a breaking change as **expand → migrate → contract**, so old and new code both run during a rolling deploy.
- Before applying a migration that recreates an existing function, view, or policy, diff its body against the live definition. A stale local copy silently reverts someone else's fix.
- Each migration PR names its rollback. "Re-run with -1" is not a plan.

**When to skip it.** Never, for a database other people depend on.

---

## 12. Lifecycles as explicit state machines

**What.** Model a persistent lifecycle (a job, an agent run, an approval, an import) as one `status` column with named states and allowed transitions. Make each transition a guarded update that names the state it leaves.

**Why it pays off.** Booleans multiply into impossible combinations (`isRunning` and `isDone` both true), and every reader re-derives the state differently. A discriminated union plus a guarded transition makes illegal states unrepresentable in code and in the database, and a lost race fails visibly instead of overwriting.

**Minimal generic example.**

```ts
type Run =
  | { status: "queued" }
  | { status: "running"; attempt: number; startedAt: Date }
  | { status: "awaiting_approval"; actionId: ActionId }
  | { status: "succeeded"; resultId: ResultId }
  | { status: "failed"; error: string; retryable: boolean }

// UPDATE runs SET status = 'running', ... WHERE id = $1 AND status = 'queued'
// Zero rows updated = someone else moved it first. Re-read; don't overwrite.
```

**When to skip it.** The object has no lifecycle.

---

## Past MVP

Production-readiness invariants (audit log, one request id end to end, error taxonomy, idempotent writes, erasure, secrets, migrations, one deployment target) and the MVP → enterprise scaling path (stage map, seams to keep open, compliance table, restore drills, scaling knobs) are covered in the agentic harness spec; see also [§17–§18](../../agentic-harness/KNOWLEDGE_MGMT_SETUP_SPEC.md#17-production-readiness-invariants-past-mvp).
