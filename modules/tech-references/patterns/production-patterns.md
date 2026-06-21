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
const Body = z.object({ email: z.string().email(), age: z.number().int() })

async function handler(req) {
  const input = Body.parse(await req.json())  // throws on bad input
  // input is now typed and trusted everywhere below
  return service.register(input)
}
```

Apply the same rule to model output: the LLM is an untrusted boundary. Parse its JSON before you act on it.

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

type Invoice = z.infer<typeof Invoice>        // static type
Invoice.parse(payload)                         // runtime validation
generateObject({ schema: Invoice, prompt })    // structured-output constraint
```

**When to skip it.** A type that exists only at compile time and never touches a runtime boundary or a model — a pure internal helper shape can stay a plain type.

---

## 4. Human-in-the-loop via atomic DB RPC

**What.** When an action needs human approval before it executes, gate it behind a single database function (RPC/stored procedure) that checks the approval state and performs the write in one transaction. The approve-and-execute step is atomic: either the action was approved and it ran, or neither happened.

**Why it pays off.** Approval logic split across application code — read the pending action, check it's approved, then write — has a race window. Two clicks, a retry, or a concurrent worker can double-execute or execute something that was just rejected. Pushing the check-and-act into one transaction makes "approved exactly once, executed exactly once" a property the database enforces, not a hope.

**Minimal generic example.**

```sql
create function approve_and_execute(action_id uuid, approver uuid)
returns void language plpgsql as $$
begin
  update pending_action
     set status = 'executed', approved_by = approver
   where id = action_id and status = 'pending';   -- atomic guard

  if not found then
    raise exception 'action not pending';
  end if;

  -- perform the gated write in the same transaction
  insert into ledger (...) values (...);
end $$;
```

**When to skip it.** Actions that are idempotent and reversible (re-running causes no harm), or approvals with no concurrency — a single operator, one action at a time, where a double-execute is impossible.

---

## 5. Provider-wrapper pattern

**What.** Put every model or external provider behind a thin adapter that exposes your own interface, and persist the raw request and response JSON on the way through. The app calls your adapter; the adapter calls the vendor and logs both sides verbatim.

**Why it pays off.** Vendor SDKs change shape, providers get swapped, and models get deprecated — a thin adapter contains that blast radius to one file. Persisting raw request/response gives you replay (re-run a failed call without the live API), audit (what exactly did the model see and return), and debugging (reproduce a bad output from stored bytes instead of guessing).

**Minimal generic example.**

```
interface CompletionProvider {
  complete(req): Promise<Result>
}

class VendorAdapter implements CompletionProvider {
  async complete(req) {
    const raw = await vendor.call(req)
    await store.put({ request: req, response: raw })  // verbatim, for replay/audit
    return normalize(raw)                             // your shape, not theirs
  }
}
```

**When to skip it.** A one-off integration you'll never replay or swap, where the vendor shape is already exactly what you want. But persisting raw I/O is cheap insurance — drop the adapter before you drop the logging.

---

## 6. Event-driven background work

**What.** Move slow, retriable, or fan-out work out of the request path into a background pipeline driven by events. Run one pipeline per concern, key concurrency per tenant so one customer can't starve another, and make retries explicit rather than implicit.

**Why it pays off.** Doing slow work inline ties up request capacity, couples user-facing latency to third-party speed, and loses the work entirely if the request dies mid-flight. An event pipeline gives you durability (the event survives a crash), backpressure (concurrency limits per tenant), and honest retry semantics instead of a user mashing refresh.

**Minimal generic example.**

```
// Request path just emits an event and returns.
emit("invoice.received", { tenantId, invoiceId })

// A separate function consumes it, scoped per tenant.
on("invoice.received", {
  concurrency: { key: event.tenantId, limit: 5 },
  retries: 3,
}, async (event) => {
  await processInvoice(event.invoiceId)
})
```

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
// Register only where the tracer is supported; no-op elsewhere.
export function register() {
  if (runtime === "nodejs") registerOTel({ serviceName })
}

// Enable the spans that explain agent behavior.
generateText({ ..., experimental_telemetry: { isEnabled: true } })
```

**When to skip it.** Local-only throwaway code, or a single runtime where unconditional setup genuinely can't break anything. The conditional guard costs nothing, though — it's usually worth keeping.
