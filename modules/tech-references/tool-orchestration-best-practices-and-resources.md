# Tool Orchestration Best Practices and Resources for LLM Agents

> State of the art as of **June 2026**.
> Synthesized from primary engineering publications (Anthropic, OpenAI, Google, Vercel, LangChain) and peer-reviewed / arXiv research.
> Core provider-agnostic and MCP claims passed **3-vote adversarial verification** (deep-research workflow, 27 sources fetched, 133 claims extracted, 25 verified 3-0, 0 killed). Per-provider deep dives (OpenAI, Google/Gemini, Vercel, LangGraph, Evaluation) were researched by dedicated agents and are anchored to fetched primary docs; marketing vs. measured results are flagged inline.
> Purpose: a durable, non-superficial reference for agentic-engineering teams.

**How to read this:** §1 is the cross-provider foundation — read it first; everything else builds on it. §2–§5 are vendor deep dives. §6 (MCP) is the cross-provider tool protocol. §7 covers orchestration frameworks. §8–§9 are the research/science layer (failure taxonomies, benchmarks, context degradation). §10 is a source/researcher index. §11 is the honesty ledger of caveats.

---

## 1. Provider-Agnostic / Universal Principles

These hold across model providers and orchestration frameworks. They are the foundation; provider-specific sections build on them.

### 1.1 Tool design: fewer, higher-level, consolidated tools beat tool proliferation

- **More tools do not reliably improve outcomes.** Consolidating multiple operations into fewer, higher-level tools (e.g. a single `schedule_event` instead of `list_users` + `list_events` + `create_event`, or a single `get_customer_context` instead of `get_customer_by_id` + `list_transactions` + `list_notes`) improves agent performance and reduces context waste.
- **Bloated, overlapping tool sets are one of the most common agent failure modes.** Aim for a *minimal viable set* with unambiguous selection: if a human engineer cannot definitively say which tool applies in a given situation, the agent cannot be expected to either.
- **Tools must be self-contained, robust to error, and clearly scoped**, with descriptive, unambiguous input parameters that play to the inherent strengths of the model.
- **The agent-computer interface (ACI) deserves the same engineering investment as a human-computer interface (HCI).** Write tool documentation like a great docstring for a junior developer: example usage, edge cases, input-format requirements, and clear boundaries from other tools.
- **Return error *strings*, not thrown exceptions.** The model can only reason about a failure it can read. A descriptive error result ("rate limited, retry after 30s") is recoverable; an exception that aborts the loop is not.

Sources: Anthropic, _Writing effective tools for AI agents_; Anthropic, _Effective context engineering for AI agents_; Anthropic, _Building Effective Agents_.

### 1.2 Context and token management

- **Make tool responses concise.** Return only relevant fields. Use pagination, range selection, filtering, and/or truncation with sensible defaults. (Anthropic's Slack example cut a response from 206 to ~72 tokens, ~⅔ fewer.)
- **Passing every intermediate tool result through the model is a primary source of token waste** in multi-tool orchestration. Filter/transform data in the execution environment before returning it.
- **Compaction** — summarizing a conversation nearing the context limit and reinitializing a fresh window, *prioritizing recall first* — is the recommended technique for long-horizon tasks. (Used in production in Claude Code: preserve architectural decisions, unresolved bugs, and implementation details; discard redundant tool outputs.)
- **More context is not free** (see §9). Each added tool schema and each verbose tool output measurably erodes selection and reasoning accuracy. Curate aggressively; do not "load everything."

Sources: Anthropic, _Writing effective tools for AI agents_; Anthropic, _Code execution with MCP_; Anthropic, _Effective context engineering for AI agents_.

### 1.3 Orchestration: start simple, escalate complexity only when justified

- **Distinguish workflows from agents.** *Workflows* orchestrate LLMs and tools through predefined code paths; *agents* let the LLM dynamically direct its own process and tool usage. Choose the simplest approach that works; add agentic complexity **only when it demonstrably improves outcomes**.
- **The five composable orchestration patterns** are the standard taxonomy for multi-step LLM systems (Anthropic, _Building Effective Agents_):
  1. **Prompt chaining** — decompose a task into a sequence of steps, each processing the previous output.
  2. **Routing** — classify an input and direct it to a specialized follow-up task.
  3. **Parallelization** — *sectioning* (independent subtasks in parallel) and *voting* (same task run multiple times for diverse outputs).
  4. **Orchestrator-workers** — a central LLM dynamically breaks down tasks, delegates to workers, and synthesizes results.
  5. **Evaluator-optimizer** — one LLM generates while another evaluates and gives feedback in a loop.

### 1.4 The agent loop is a solved problem; the engineering *around* the loop is where decisions live

A cross-framework consensus (articulated well by Steve Kinney, _Agent Loops_, 2026-03-19, and reflected in every SDK below) on production hardening:

- **Defense in depth on termination:** iteration caps (≈15–25 steps), wall-clock timeouts (≈300s), token/cost budgets, loop detection via request fingerprinting, and error classification (retry 429/5xx; terminate on 401/403).
- **On hitting the step cap, do a final tool-free synthesis call** rather than returning nothing — give the model a last turn with tools disabled to summarize what it has.
- **Maximize prompt/KV-cache hits** (stable system prompt + tool definitions at the front; append-only history) and compact context around ~80% window usage.

---

## 2. Anthropic / Claude

Anthropic's engineering publications are the most-cited primary sources in this space and define much of the cross-provider consensus in §1.

### 2.1 Tool naming and namespacing

- **Namespacing conventions have measurable, non-trivial effects on tool-use eval performance.** Prefix/service namespacing (`asana_search`) versus suffix/resource namespacing (`asana_projects_search`) can change results. Anthropic frames this explicitly as a measured eval finding and advises validating with **your own evals**, because effects vary by model.

### 2.2 Multi-agent orchestration on Claude

- Use **specialized sub-agents with clean context windows, coordinated by a main (lead) agent.** Each sub-agent returns only a condensed, distilled summary — **often 1,000–2,000 tokens**. The main agent holds the high-level plan; sub-agents do deep technical work in isolation.
- This orchestrator-worker pattern (lead plans → spins up 3–5 parallel specialized sub-agents, each with its own context window, each returning a condensed summary) is the basis of Anthropic's multi-agent research system.

### 2.3 Code execution with MCP — the headline token saving

- Having the agent **write code that calls MCP servers** and filter/transform results in the execution environment (rather than passing raw tool output through the model) keeps intermediate results out of context. A documented Google Drive → Salesforce workflow reduced token usage from **150,000 → 2,000 tokens (98.7%)**: the large transcript never entered the model context; only the small final result did.
- This is the recommended pattern for high-volume, multi-tool workflows. *(Anthropic's own illustrative example — accurately attributed, not an independently audited benchmark.)*

### 2.4 Advanced tool use (provider features worth knowing)

- Claude exposes **programmatic / code-execution-callable tools**: a tool can be marked callable from within the code-execution sandbox, so the model orchestrates many tool calls in code without round-tripping each through the context window (see §5 for the Vercel AI SDK wiring of this, `allowedCallers: ['code_execution_…']`).
- Claude reports **pass^k** reliability metrics in model cards (see §9) — evidence the reliability-not-accuracy framing is now an industry standard.

Sources: Anthropic, _Writing effective tools for AI agents_; _Effective context engineering for AI agents_; _Code execution with MCP_ (Nov 2025); _Building Effective Agents_; _Advanced tool use_; _Multi-agent research system_ (anthropic.com/engineering/*, anthropic.com/research/building-effective-agents).

---

## 3. OpenAI

OpenAI's tool-orchestration stack as of mid-2026 has three layers: the **Responses API** (the function/tool-calling primitive and host for built-in tools), the open-source **Agents SDK** (runner loop, handoffs, guardrails, sessions, tracing — the production successor to the experimental *Swarm*), and **AgentKit** (a higher-level visual/managed layer announced at DevDay, Oct 6 2025). The Assistants API is on a deprecation path (target sunset mid-2026); Responses is the forward direction.

### 3.1 Two orchestration models (OpenAI explicitly recommends mixing them)

- **LLM-driven orchestration** — the model plans, reasons, and chooses steps via instructions, tools, and handoffs. Best for open-ended tasks. OpenAI's stated best practices: invest in good prompts, use specialized (not generalist) agents, let agents introspect/self-improve, and "invest in evals."
- **Code/deterministic orchestration** — flow controlled in code for predictable speed/cost/quality. Patterns: classify with structured outputs then branch; chain agents transforming each output into the next input; `while` feedback loops; parallel execution via `asyncio.gather`.
- Core design principle (conservative): *"Start with one agent whenever you can. Add specialists only when they materially improve capability isolation, policy isolation, prompt clarity, or trace legibility."*

### 3.2 Agents-as-tools vs. handoffs — *who owns the final response*

- **Agents as tools (manager pattern):** a manager agent stays in control and calls specialists as bounded capabilities via `Agent.as_tool()` (TS: `agent.asTool()`); the manager synthesizes the final answer. `as_tool()` parameters: `tool_name`, `tool_description`, `parameters` (Pydantic input), `custom_output_extractor`, `needs_approval` (approval gate), `is_enabled` (bool or callable), `on_stream`.
- **Handoffs:** a triage/router agent transfers control; the specialist *becomes the active agent* for the rest of the turn and owns the user-facing response. Declared via the `handoffs` parameter; keep `handoff_description` concise; split only when branches truly need different instructions/tools/policies. Handoffs can carry structured metadata and filtered conversation history.

### 3.3 Responses API function/tool-calling model

- **Strict mode / structured outputs for tool args:** `strict: true` guarantees schema adherence (requires `additionalProperties: false`, all fields in `required`, optional fields modeled as `["string","null"]`). The **Responses API normalizes to strict automatically**; Chat Completions defaults to non-strict. Caveat: fine-tuned models disable strict when emitting multiple parallel calls.
- **Parallel tool calls:** `parallel_tool_calls` defaults to `true`; set `false` to force at most one tool per turn.
- **`tool_choice`:** `"auto"` (default), `"required"` (force ≥1), `"none"` (forbid), named function (force exactly one), and **`allowed_tools`** to restrict the model to a subset while keeping the full tool definitions cached (prompt-cache-friendly).
- **Wire format:** model emits `function_call` items in the `output` array with `call_id`/`name`/`arguments`; you append `function_call_output` items with the matching `call_id`. Streaming uses `response.function_call_arguments.delta` events.

### 3.4 Built-in hosted tools

Configured in the `tools` array; the model decides whether to invoke (overridable via `tool_choice`): `web_search`, `file_search` (over OpenAI Vector Stores), `code_interpreter` (hosted sandbox), `image_generation`, `mcp` (remote MCP servers), `function` (custom). **Computer use** is a separate tool tied to the `computer-use-preview` model, **Responses-API-only** (returns UI actions against screenshots your harness executes; OpenAI recommends sandboxing + human-in-the-loop).

### 3.5 Tool selection at scale

1. **Tool search / deferred loading** — dynamically load only relevant tool definitions instead of front-loading all tools. In the Agents SDK: `ToolSearchTool`, `@function_tool(defer_loading=True)`, and `tool_namespace(name, description, tools=[...])` to group large tool surfaces. *(This is the same deferred-tool pattern this very Claude Code environment uses.)*
2. **Model-routed multi-tool orchestration** — the cookbook _Multi-Tool Orchestration with RAG_ shows the model routing between `web_search` and a custom vector-DB function purely from instructions. *(Illustrative demo, not a benchmark.)*

### 3.6 Runner loop, sessions, guardrails, tracing (Agents SDK)

- **Runner:** `Runner.run()` (async → `RunResult`), `run_sync()`, `run_streamed()`. Loop: call LLM → if final output, stop; if handoff, swap active agent; if tool calls, execute, append, continue.
- **`max_turns`:** exceeding raises `MaxTurnsExceeded`; `max_turns=None` disables the cap. *(Default value not stated in the docs read — unverified.)*
- **`RunConfig`:** overrides for model/provider/settings, input/output guardrails, tracing (`tracing_disabled`, `workflow_name`, `trace_id`), `call_model_input_filter`.
- **Guardrails:** `@input_guardrail` / `@output_guardrail` (+ tool-level `@tool_input_guardrail` / `@tool_output_guardrail`); return `GuardrailFunctionOutput(output_info, tripwire_triggered)`; a tripwire raises `InputGuardrailTripwireTriggered` / `OutputGuardrailTripwireTriggered`. Input guardrails can run **in parallel** with the agent (lower latency, may burn tokens before tripping) or blocking; output guardrails run only on the final-output agent.
- **Sessions:** pluggable cross-turn memory — SQLite, SQLAlchemy, Redis, MongoDB, encrypted variants.
- **Tracing:** built-in spans for the agent flow, feeding evals and fine-tuning.

### 3.7 Evals for orchestration

- **AgentKit "Evals for Agents":** step-by-step **trace grading**, per-component datasets, automated prompt optimization, running evals against external models.
- Cookbook **Macro Evals for Agentic Systems** is the most concrete orchestration-eval guidance: distinguishes **per-agent evals** (routing, tool appropriateness, handoff timing, policy compliance) from **macro evals** that cluster patterns across many traces (embeddings → UMAP → HDBSCAN), score `impact_score = prevalence_share × severity_weighted_prevalence`, and diagnose via backward execution-graph traversal. Framing: *"a final answer is only the last event in a longer workflow"* — failures are usually wrong routing / mistimed handoffs, not a single bad response.

**Flags:** AgentKit ROI/benchmark claims are marketing (only a dev-speed anecdote — Ramp "built a procurement agent in a few hours" — could be verified). Cookbook routing results are illustrative, not benchmarked. `max_turns` default unverified. The official AgentKit page 403'd; those details rely on secondary sources.

**Sources:** openai.github.io/openai-agents-python/{multi_agent,running_agents,guardrails,tools} · developers.openai.com/api/docs/guides/{agents/orchestration,function-calling,tools,tools-computer-use} · developers.openai.com/cookbook/examples/{responses_api/responses_api_tool_orchestration,partners/macro_evals_for_agentic_systems} · openai.com/index/new-tools-for-building-agents · techcrunch.com/2025/10/06/openai-launches-agentkit-…

---

## 4. Google / Gemini

### 4.1 Gemini API function calling (core orchestration)

- **Function declarations use an OpenAPI 3.0.3 *subset*** (`name`, `description`, `parameters`). Very large/deeply nested schemas may be rejected; remedy is shorter names, less nesting, fewer declarations. Practical guidance: keep **10–20 active tools max** for reliability.
- **Function-calling modes** via `tool_config.function_calling_config.mode`:
  - `AUTO` — default; model chooses text or a call.
  - `ANY` — forces a (schema-compliant) function call ("forced function calling").
  - `NONE` — prohibits calls.
  - `VALIDATED` — newer; default when built-in tools combine with custom functions; constrains output to schema-adherent calls or text.
  - `allowed_function_names` restricts the callable set (used with `ANY`/`VALIDATED` to force a specific tool).
- **Parallel function calling:** multiple independent calls in one turn; results may return out of order, each matched by `id`.
- **Compositional / sequential function calling:** outputs of one call feed the next; Python SDK supports *automatic* function calling (executes the loop for you).
- **Call IDs are mandatory plumbing:** echo the exact `id` in the matching `functionResponse`. Critical for parallel calls and cross-tool context.
- **Reliability levers:** check `finishReason` to detect malformed/invalid calls; validate consequential calls before executing.

### 4.2 Thinking / reasoning interplay with tools (Gemini 3.x)

- **Thought signatures are load-bearing.** Gemini 3 returns encrypted `thought_signature` parts that must be sent back **verbatim, in original order**. For **function calling this is strict — a missing signature returns HTTP 400.** Patterns: single call → one signature; **parallel calls → only the first carries a signature, but return all parts in received order**; sequential → each step has its own, accumulate and return all. Google's GenAI SDKs handle this automatically; raw REST callers must manage it.
- **Thinking control:** `thinking_level` ∈ `minimal`/`low`/`medium`/`high` (high default). Legacy `thinking_budget` deprecated; setting both → 400.
- **Temperature guidance flipped between generations — gotcha.** The function-calling page still advises low/0 temperature, but the **Gemini 3 guide strongly recommends keeping temperature at the default 1.0**, warning that lowering it causes looping/degraded performance. Do **not** carry over the old "temperature=0 for tools" habit on Gemini 3.x.
- Gemini 3 supports **multimodal function responses** (nested images/PDF/text via `inlineData`).

### 4.3 Built-in tools and 2026 tooling updates (verified via official blog + changelog)

- **Tool combination (Mar 18, 2026):** built-in tools (Google Search grounding, Maps grounding, URL Context, Code Execution, File Search) can now combine with custom function declarations **in a single request**, removing a separate orchestration hop. Claimed latency benefit is **unquantified marketing**.
- **Context circulation:** every built-in tool call/response is preserved so later steps (incl. custom tools) can reason over it; recommended via the **Interactions API**.
- **Grounding with Google Maps** extended to the Gemini 3 family. **URL Context** tool launched experimental. **File Search / RAG** updated (May 5, 2026) for multimodal search + visual citations (`media_id`, `page_numbers` in grounding metadata). **Batch API** moved to **event-driven Webhooks** (May 4, 2026).
- **Interactions API breaking change** (eff. May 26, 2026): `outputs` → `steps`. **Managed Agents** entered public preview (May 19, 2026). *(Model-naming details from the changelog fetch were not independently re-verified field-by-field.)*

### 4.4 Agent Development Kit (ADK) — orchestration primitives

- **Workflow agents:** `SequentialAgent`, `ParallelAgent`, `LoopAgent`; plus `LlmAgent` (model-driven) and custom `BaseAgent`.
- **Multi-agent hierarchies & agents-as-tools:** agents compose into hierarchies; a sub-agent can be wrapped and exposed as a tool to a parent.
- **Authoring:** YAML or drag-and-drop visual UI; deploy to Cloud Run with one command.
- **A2A (Agent2Agent) protocol:** cross-framework collaboration — agents discover each other via standardized **Agent Cards**, authenticate cryptographically, exchange tasks over **gRPC** (Python, Go, Java). *(Exact symbols like `RemoteA2aAgent` / `to_a2a()` could not be verified from live docs — unverified.)*

### 4.5 Vertex AI Agent Engine / Agent Builder (production)

- **Agent Engine:** managed runtime — autoscaling, session management, long-running execution, evaluation, **Sessions**, **Memory Bank**, Code Execution; deploy via `adk deploy`.
- **Sessions vs. Memory Bank:** Sessions hold within-conversation state; Memory Bank persists across conversations. Both **GA, billed separately at $0.25 per 1,000 events/memories from Jan 28, 2026.**
- Agent Builder added **enhanced tool governance** (enterprise control over callable tools) — feature, not benchmarked.

### 4.6 Tool-calling reliability vs. competitors (honest assessment)

- **Berkeley Function-Calling Leaderboard (BFCL)** is the standard AST-based eval (serial + parallel calls across Python/Java/JS/REST; V4 extends to agentic/multi-turn).
- Secondary-sourced BFCL figures (treat as **approximate**): Gemini 3.1 Pro ≈ 71.2% (v3); Claude Opus 4.1 ≈ 70.36%, Sonnet 4 ≈ 70.29%; GPT-5 ≈ 59.22%. **The official Gemini 3.x standing on gorilla.cs.berkeley.edu was not clearly visible in searches — head-to-head ranking unverifiable from primary source.**
- **Honest signal:** all frontier models ace single-shot calls but **degrade on multi-turn, long-context, and "when not to act" decisions**; Claude (Sonnet 4.5 / Opus 4) is repeatedly cited as leading **τ-bench multi-turn**. Gemini's *single-call* function calling is competitive, but **multi-turn/compositional reliability appears to lag the strongest competitors**, and Google publishes no quantified tool-reliability numbers in its own announcements.

**Sources:** ai.google.dev/gemini-api/docs/{function-calling,gemini-3,changelog,maps-grounding} · blog.google/innovation-and-ai/technology/developers-tools/gemini-api-tooling-updates · developers.googleblog.com/en/agent-development-kit-… · adk.dev/a2a · google.github.io/adk-docs/sessions/memory · cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview · cloud.google.com/agent-builder/agent-engine/memory-bank/overview · gorilla.cs.berkeley.edu/leaderboard.html

---

## 5. Vercel AI SDK

AI SDK 6 is the current major release. It builds on the v5 redesign (mid-2025) and, unlike v4→v5, ships with minimal breaking changes. The biggest tool-orchestration shifts came in two waves: the `maxSteps`→`stopWhen` redesign (v5), and the `Agent` abstraction + default-loop changes (v6).

### 5.1 The Agent abstraction (v6)

`Agent` is an **interface**; `ToolLoopAgent` is the default production implementation. It encapsulates the loop (call model → execute tools → append → repeat).

```ts
import { ToolLoopAgent, stepCountIs } from 'ai';
const agent = new ToolLoopAgent({
  model: 'anthropic/claude-sonnet-4.5', // provider-agnostic string (routed via AI Gateway)
  instructions: 'You are a helpful weather assistant.', // was `system` in v5
  tools: { weather: weatherTool },
  stopWhen: stepCountIs(20),
});
const { text, steps } = await agent.generate({ prompt: '...' });
```

Options: `model`, `tools`, `instructions`, `stopWhen`, `prepareStep`, `toolChoice`, `output` (structured), plus `callOptionsSchema` + `prepareCall` for per-invocation config. **Version note (v5→v6):** `Experimental_Agent` → `ToolLoopAgent`; config key `system` → `instructions`.

### 5.2 Loop control: stopWhen, stepCountIs, prepareStep

- Built-in stop conditions: `stepCountIs(n)`, `hasToolCall(name)`, `isLoopFinished()` (run until model stops requesting tools), custom `({ steps }) => boolean`. `stopWhen` accepts one condition or an array (any match stops).
- **`prepareStep`** runs **between steps** and can override `model` (switch models mid-run), `messages` (trim/compact), `activeTools` (narrow the tool surface per step — the primitive for tool-selection-at-scale), and `toolChoice`.
- `onStepFinish({ stepNumber, text, toolCalls, toolResults, finishReason, usage })` fires after each step.
- **Critical version change (flag):** v4 used `maxSteps`; v5 removed it for `stopWhen` and **defaulted to single-step** (`stepCountIs(1)` — you opt into looping); **v6 changed the default to `stepCountIs(20)`** — agents loop by default now. Watch this on upgrade.

### 5.3 Tool definition: `tool()` helper

- `description`, `inputSchema` (Zod or JSON; **renamed from `parameters` in v4**), `execute(input, { toolCallId, messages, abortSignal, experimental_context })`.
- `strict: true` — per-tool native provider strict-schema mode (v6 moved this from a global option to per-tool).
- `needsApproval: boolean | async (input) => boolean` — human-in-the-loop; produces `tool-approval-request` parts requiring a follow-up `tool-approval-response`.
- `toModelOutput` (transform result before returning to model; **v6 receives `{ output }`, may be async**), `inputExamples`, streaming callbacks `onInputStart`/`onInputDelta`/`onInputAvailable`.
- **`dynamicTool({...})`** for tools unknown at compile time (e.g. MCP tools without static schemas). **`toolChoice`:** `'auto'` (default), `'required'`, `'none'`, `{ type: 'tool', toolName }`.
- **Provider-executed tools (v6):** Anthropic programmatic tool calling via `providerOptions.anthropic.allowedCallers: ['code_execution_20250825']` + `anthropic.tools.codeExecution_20250825()` + a `prepareStep` forwarding the container ID. OpenAI/Google expose built-ins (`shell`, `applyPatch`, `fileSearch`, `googleMaps`, `vertexRagStore`). *(Exact tool version strings — re-verify against live provider docs before pinning.)*

### 5.4 streamText / generateText, telemetry, UI parts

- Both accept `tools`, `toolChoice`, `stopWhen`, `prepareStep`, `onStepFinish`. After a run, append `response.messages` to history. `result.staticToolCalls` exposes client-side tools (no `execute`).
- **Telemetry (OpenTelemetry):** `experimental_telemetry: { isEnabled, functionId, metadata, recordInputs, recordOutputs, tracer }`. Tool spans `ai.toolCall` record name/id/args/result, nested under `ai.generateText.doGenerate` / `ai.streamText.doStream`. `recordInputs`/`recordOutputs` suppress sensitive payloads.
- **UIMessage parts model:** assistant messages carry `parts[]` — statically typed `tool-{toolName}`, dynamic `dynamic-tool`, `step-start` boundaries. State machine: `input-streaming` → `input-available` → `output-available`/`output-error`, plus `approval-requested`. Client helpers `addToolOutput` (replaced v5's `addToolResult`) and `addToolApprovalResponse({ approved })`. The **ai-elements** component library renders these on top of `useChat`.

### 5.5 MCP client + provider-agnostic routing

- Package **`@ai-sdk/mcp`** (v6 moved MCP out of core): `createMCPClient({ transport: { type: 'http', url, headers, authProvider } })`. Transports: **HTTP/Streamable HTTP** (recommended for remote/prod), **SSE**, **stdio** (local). `mcpClient.tools()` auto-discovers schemas, or `tools({ schemas })` for explicit typed subsets (recommended — reduces tool surface). Always `await mcpClient.close()`. v6 adds OAuth (`authProvider`), **elicitation**, resource discovery (`listResources`/`readResource`), experimental prompts.
- **Provider-agnostic routing:** a bare `creator/model` string (e.g. `anthropic/claude-sonnet-4.5`, `openai/gpt-5.5`) routes through **Vercel AI Gateway** — one endpoint across providers with budgets/load-balancing/fallbacks, controlled via `providerOptions.gateway` (`order`, `only`, `sort`). Combined with `prepareStep`'s `model` override, you can route different loop steps to different models (cheap model for tool-dispatch, frontier model for synthesis).

**Flags:** provider tool version strings and the full OpenAI/Google built-in list come from the v6 blog summary (re-verify before pinning); the telemetry `integrations`/lifecycle-hook API is tentative; AI SDK 6 GA-vs-beta status as of June 2026 not separately confirmed (docs present it as current).

**Sources:** vercel.com/blog/ai-sdk-6 · ai-sdk.dev/docs/{agents/overview,agents/loop-control,ai-sdk-core/tools-and-tool-calling,ai-sdk-core/telemetry,ai-sdk-core/mcp-tools,ai-sdk-ui/chatbot-tool-usage,migration-guides/migration-guide-5-0,migration-guides/migration-guide-6-0} · vercel.com/docs/ai-gateway · stevekinney.com/writing/agent-loops (2026-03-19, third-party)

---

## 6. Model Context Protocol (MCP)

MCP is the cross-provider connective tissue for tool exposure (Anthropic, OpenAI, Google, and Vercel all consume it). The dominant 2025–2026 research/engineering theme is **scaling tool selection without prompt bloat.**

### 6.1 The core scaling problem

- **Loading all MCP tool definitions upfront increases response time and cost** as tools/servers grow; with thousands of tools, agents process hundreds of thousands of tokens before reading a request. *(Independent corroboration: teams report 3 MCP servers consuming 143k/200k tokens = 72% of context; GitHub's official MCP server ≈17,600 tokens/request.)*
- **Injecting all tool descriptions degrades tool-selection accuracy** as the count grows — paralleling Needle-in-a-Haystack / lost-in-the-middle degradation. **Larger context windows do not solve the underlying selection/reasoning degradation** (see §9).

### 6.2 Code execution with MCP

Having the agent **write code that calls MCP servers** and filter/transform results in the execution environment keeps intermediate results out of context (see the 98.7% reduction in §2.3). Recommended for high-volume multi-tool workflows.

### 6.3 RAG-MCP — retrieval-based tool selection

`arXiv:2505.03275`, _RAG-MCP: Mitigating Prompt Bloat in LLM Tool Selection via Retrieval-Augmented Generation_ (Tiantian Gan, Qiyao Sun; May 2025):

- **Semantic retrieval pre-selects only the most relevant tool descriptions** from an external index and passes *only those* to the LLM.
- **Result:** more than **tripled tool-selection accuracy — 43.13% vs. 13.62% baseline** ("Blank Conditioning," all N tools at once); prompt tokens cut >50%.
- *Caveats:* non-peer-reviewed preprint; single LLM (Qwen-max-0125), single benchmark (web-search subset of MCPBench); the 13.62% baseline is an aggregate over a 1–100 candidate stress test; a third baseline ("Actual Match," 18.20%) is omitted from the headline framing.

### 6.4 ScaleMCP — dynamic, agentic, auto-synchronizing tool selection

`arXiv:2505.06416`, _ScaleMCP_ (Elias Lumer, Anmol Gulati, Vamse Kumar Subbiah, Pradeep Honaganahalli Basavaraju, James A. Burke — PwC; May 2025):

- **Dynamic agentic retrieval:** equips the agent with a dedicated **MCP Retrieval Tool**, giving it autonomy to add tools to its own memory and iteratively search/evaluate/invoke over multiple steps **during multi-turn interactions** (rather than statically fixing selection up front); re-queries when no match is found.
- **Auto-synchronizing storage:** treats **MCP servers as the single source of truth**, using CRUD ops and **SHA-256 hashing of tool metadata** to detect/reflect changes automatically — eliminating error-prone manual updates.
- **Eval at scale:** dataset of **5,000 financial-metric MCP servers** + **140,000 query instances**, across 10 LLMs, 5 embedding models, 5 retriever configs. *(Authors' own dataset; "substantial improvements" is their characterization.)*

### 6.5 Benchmarks for MCP tool selection (see also §9)

- **MCP-Bench** (`arXiv:2508.20453`, Accenture; NeurIPS 2025 workshop) — tool retrieval from **fuzzy instructions without explicit tool names**, multi-hop planning, cross-domain orchestration over real MCP servers; 20 LLMs show persistent failures.
- **MCPAgentBench** (`arXiv:2512.24565`) — extends to real-world MCP tasks.

Sources: Anthropic, _Code execution with MCP_; `arXiv:2505.03275`; `arXiv:2505.06416`; `arXiv:2508.20453`.

---

## 7. Orchestration Frameworks (LangGraph and peers)

### 7.1 LangGraph — why a graph / state-machine model

LangGraph models an agent as an explicit **`StateGraph`** of nodes and edges over a typed, shared **state**, rather than a free-form "LLM-in-a-loop." Execution advances in discrete **super-steps** (one tick where all scheduled nodes run, possibly in parallel); control flow is expressed via normal edges, **conditional edges**, and the **`Command`** primitive. Trade-off vs. free-form loops: less autonomy/terseness in exchange for deterministic, inspectable routing, durable resumability, and interrupt/replay — the reasons LangChain positions it for "controllable" production orchestration.

### 7.2 Persistence, durable execution, HITL, time-travel

- **Checkpointer:** compiling with `InMemorySaver` / `SqliteSaver` / `PostgresSaver` (all `BaseCheckpointSaver`) snapshots state at every super-step into **threads** keyed by `thread_id`. Read via `graph.get_state(config)` / `get_state_history(config)` → `StateSnapshot`. Subgraphs namespaced via `checkpoint_ns`.
- **Durability modes:** `"exit"` (persist at completion — fastest, no mid-run recovery), `"async"` (write while next step runs), `"sync"` (write before proceeding — highest durability). The crash-recovery knob.
- **Human-in-the-loop:** pause via `interrupt(...)` / `interrupt_before=[...]`; human mutates state with `graph.update_state(...)`; resume by invoking with the thread (`graph.invoke(None, config)`).
- **Time travel:** re-invoke with a specific `checkpoint_id` to replay; `update_state()` from a past checkpoint **forks** a new branch without mutating the original.

### 7.3 Tool execution: ToolNode, create_react_agent, errors, parallelism

- **`create_react_agent`** (`langgraph.prebuilt`) wires an LLM node to a **`ToolNode`** — the standard ReAct loop.
- **`ToolNode`** executes calls, formats outputs as tool messages, handles errors via `handle_tool_errors`; **`GraphInterrupt` is always re-raised** even with error handling on (preserves HITL).
- **Parallel tools:** a single assistant message can emit multiple `tool_calls`; `ToolNode` runs them and returns all results in one batch. Finer-grained fan-out via the **`Send`** API.

### 7.4 Tool selection at scale — RAG over the tool catalog

For agents with hundreds/thousands of tools, the official pattern is **`langgraph-bigtool`**: register tools into LangGraph's long-term **store** (`store.put(("tools",), tool_id, {...})`), then **semantically retrieve** a small relevant subset per query (`store.search(("tools",), query=..., limit=...)`) before model invocation. Custom logic via `retrieve_tools_function`/`retrieve_tools_coroutine`; backends `InMemoryStore`, `PostgresStore`. The "narrow to ~3–5 tools rather than 50–100+" guidance is a third-party heuristic, not an official number.

### 7.5 Multi-agent architectures

Official taxonomy (LangChain docs): **subagents** (coordinator calls subagents as tools; all routing through it), **handoffs** (agents transfer control via tool calls), **router** (classifier dispatches to specialists), **skills**, **custom workflows**.

- **`Command` primitive** drives handoffs: a node/tool returns `Command(goto=..., update=...)`; `Command(goto=..., graph=Command.PARENT)` routes to a parent-graph node — cross-agent navigation in one step.
- **`langgraph-supervisor`:** `create_supervisor()` + generated handoff tools (`create_handoff_tool()`, `create_forward_message_tool()`, `with_agent_name`). Centralized — the supervisor always mediates.
- **`langgraph-swarm`:** `create_swarm()`, `create_handoff_tool()`, `add_active_agent_router()`, `SwarmState` (tracks the active agent). Decentralized — any agent can hand off directly.
- **Guidance (third-party):** start with supervisor (simpler to build/debug); move to swarm when latency data shows the centralized hop is the bottleneck and misrouting is rare. (A cited "~40% latency reduction" is a single-case marketing claim, unverified.)

### 7.6 Observability & eval: LangSmith

**LangSmith** traces the full execution tree (every LLM call, tool invocation, retrieval, and parameters), integrating natively with LangGraph and via SDKs for any stack; adds dataset-driven **evaluation** (offline + online). In 2026 it is positioned as an "agent engineering platform" (observe → evaluate → deploy). Adoption figures ("~89% have observability, 52% evals") are from LangChain's own _State of Agent Engineering_ survey — vendor-sourced.

### 7.7 Framework comparison (orchestration-control lens — third-party, directional)

| Framework | Model | Pick when |
|---|---|---|
| **LangGraph** | Directed graph + conditional edges; checkpointing/time-travel; highest control + production tooling | Explicit control flow, long-running/durable workflows, HITL approval gates |
| **OpenAI Agents SDK** | Explicit handoffs, built-in tracing/guardrails; ephemeral state; OpenAI-centric | OpenAI-native, lighter orchestration |
| **CrewAI** | Role-based crews / process types; lowest setup; limited checkpointing | Straightforward role-defined multi-agent tasks |
| **AutoGen / AG2** | Conversational `GroupChat` + selector logic; in-memory history | Conversation-driven agent collectives |
| **Vercel AI SDK / Mastra** | TypeScript-first; AI SDK strongest for streaming web UIs | TS/JS web stacks |

Recurring 2026 theme across these roundups: framework boundaries are blurring toward multi-framework compositions.

### 7.8 Named people / 2026 direction

**Harrison Chase** (LangChain co-founder/CEO) is pushing **"ambient agents"** — long-running, event-driven agents acting in the background rather than 1:1 chat — backed by LangGraph (orchestration + persistence), **LangGraph Platform** (scale), and LangSmith (observability), with an **"agent inbox"** prototype for human approvals. Enterprise guidance: invest in frameworks that persist state across interactions, support HITL at any step, and give visibility into agent actions.

**Sources:** docs.langchain.com/oss/python/langgraph/persistence · reference.langchain.com/python/{langgraph.prebuilt/tool_node/ToolNode,langgraph-supervisor,langgraph-swarm} · github.com/langchain-ai/langgraph-bigtool · docs.langchain.com/oss/python/langchain/multi-agent · langchain.com/{langsmith/observability,state-of-agent-engineering} · sequoiacap.com/podcast/training-data-harrison-chase{,-2} · speakeasy.com/blog/ai-agent-framework-comparison (third-party comparisons)

---

## 8. Multi-Agent Orchestration: Failure Modes (research layer)

### 8.1 Why multi-agent systems fail (MAST taxonomy)

`arXiv:2503.13657`, _Why Do Multi-Agent LLM Systems Fail?_ (Cemri, Pan, Yang et al. — incl. Klein, Zaharia, Gonzalez, Stoica; UC Berkeley Sky Computing Lab; **NeurIPS 2025, peer-reviewed**):

- The **MAST taxonomy identifies 14 distinct failure modes** in 3 categories: (1) system/specification design, (2) inter-agent misalignment, (3) task verification/termination. Built from 1,600+ annotated traces across 7 MAS frameworks (Cohen's Kappa 0.88).
- **Multi-agent systems often deliver minimal gains over single-agent baselines.** Failures are largely **architectural/design-driven, not base-model capability limits** — headroom from better MAS design persists across model families (GPT-4, Claude 3, Qwen2.5, CodeLlama), meaning failures are fixable via design.

### 8.2 Hierarchical orchestration

`arXiv:2506.12508`, _AgentOrchestra_ (June 2025): a **hierarchical framework** where a **central planner coordinates specialized sub-agents** (deep researcher, browser-use, deep analyzer, tool manager) and **dynamically extends capabilities at runtime**, built on a Tool-Environment-Agent (TEA) protocol.

> **Practical synthesis:** §8.1 + the τ²-bench dual-control results (§9.1) + the BFCL multi-turn degradation (§4.6) all point the same way — *the model is rarely the bottleneck; the orchestration design is.* Default to a single agent (per OpenAI §3.1 and Anthropic), add agents only for genuine capability/policy isolation, and instrument every handoff (§3.7, §7.6) because mis-routing and mistimed handoffs are the dominant failure class.

Sources: `arXiv:2503.13657`; `arXiv:2506.12508`.

---

## 9. Evaluation and Context Management (the scientific layer)

Tool-using agents fail in ways single-run demos hide: wrong tool, lost thread across turns, degradation as tool defs and outputs accumulate. The rigorous literature converges on three claims — (1) measure *reliability*, not one-shot success; (2) more context is not free and often hurts; (3) trust only evals you built and validated yourself.

### 9.1 Reliability, not single-run accuracy: the τ-bench / τ²-bench line

- **τ-bench** (Yao, Shinn, Razavi, Narasimhan; `arXiv:2406.12045`, Jun 2024; ICLR/OpenReview): dynamic conversations between a simulated user and a tool-equipped agent with written policy, scored by comparing the *final database state* to an annotated goal (not text). Introduces the **pass^k** metric — probability of success on the *same* task across all k independent trials. Headline: SOTA function-calling agents (gpt-4o at the time) succeeded on **<50% of tasks** and were **inconsistent (pass^8 <25% in retail)**. Anthropic now reports pass^k in model cards — the metric became an industry reliability standard.
- **τ²-bench** (Victor Barres, Honghua Dong, Soham Ray, Xujie Si, Karthik Narasimhan; `arXiv:2506.07982`, Jun 2025): extends to a **dual-control** setting modeled as a **Dec-POMDP** where *both* agent and user hold tools and act on shared dynamic state. Finding: **up to a 25-point drop** in task success moving from solo to interactive (dual-control) mode, even for GPT-4.1 and o4-mini — isolating *tool orchestration under coordination* as a distinct, harder failure mode.
- Both from **Sierra Research** (co-founded by **Bret Taylor** and **Clay Bavor**; **Karthik Narasimhan** is a benchmark *author*, not a Sierra founder — do not conflate). Amazon released **tau2-bench-verified**, a corrected dataset — even rigorous benchmarks carry annotation noise.

### 9.2 Context Rot: more context degrades performance, even on trivial tasks

**Context Rot** (Kelly Hong, Anton Troynikov, Jeff Huber; Chroma; July 2025) — non-peer-reviewed industry report, methodologically careful. Across **18 models** (Claude 4/3.7/3.5/Haiku; GPT-4.1/4o/o3/4-Turbo/3.5; Gemini 2.5 Pro/Flash, 2.0 Flash; Qwen3 235B/32B/8B): **"models do not use their context uniformly; performance grows increasingly unreliable as input length grows"** — even on tasks trivial at short length. **Even a single distractor lowers accuracy, amplifying with length.** Direct implication: stuffing many tool definitions and long tool outputs into the window is not free — argues for tool filtering/retrieval, output summarization, aggressive curation.

### 9.3 Long context windows do NOT fix tool-selection degradation

- **"Lost in the Middle"** (Nelson F. Liu, Kevin Lin, John Hewitt, Ashwin Paranjape, Michele Bevilacqua, Fabio Petroni, Percy Liang; **TACL 2023, peer-reviewed**; `arXiv:2307.03172`): a **U-shaped curve** — accuracy highest when relevant info is at the start/end, degrading badly in the middle, **even for explicitly long-context models.**
- **LongFuncEval** (IBM Research — Kate, Pedapati, Basu, Rizk, et al.; `arXiv:2505.10570`, preprint): isolates three long-context stressors for function calling — **large tool catalogs → 7–85% drop**; **long API responses → 7–91% drop**; **extended multi-turn → 13–40% drop**. Conclusion: "LLMs still struggle with long context in tool calling settings." With Context Rot + Lost-in-the-Middle, this is the rigorous basis that bigger windows ≠ better tool selection.

### 9.4 Benchmarks and reliability metrics (2025–2026)

- **Berkeley Function-Calling Leaderboard (BFCL)** — Gorilla team, UC Berkeley (Shishir G. Patil, Huanzhi Mao, Fanjia Yan, Charlie Cheng-Jie Ji); paper in PMLR (proceedings.mlr.press/v267/patil25a.html). AST-match over simple/parallel/multiple calls + **irrelevance detection**. **V3** added multi-turn/multi-step **state-based evaluation**; **V4** adds agentic categories (web search, memory) and the ability to **abstain**.
- **MCP-Bench / MCPAgentBench** — see §6.5.
- **ReliabilityBench** (`arXiv:2601.06112`, Aayush Gupta, single author, Jan 3 2026, preprint — **low provenance, flagged**): formalizes three axes — **consistency (pass^k)**, **robustness to semantic perturbations (ε)**, **fault tolerance under injected tool/API failures (λ)**. Useful framing; its specific numbers should be independently verified before citing.

### 9.5 Practical eval guidance: build (and validate) your own evals

- **Evaluate with your own evals.** Public leaderboards measure generic capability, not *your* tool catalog/policies/failure economics. Use the τ-bench template: define success as a **checkable state change**, run many trials, report **pass^k** (production reliability is the all-k-succeed probability), not a single pass rate.
- **LLM-as-judge is biased — know the three failure modes:** **position bias** (verdict flips in ~10–30% of comparisons when you swap order; worse with more candidates), **verbosity bias** (prefers longer answers), **self-preference** (favors own/lower-perplexity outputs — GPT-4 shown ~10% higher win rate on its own). Sources: "Self-Preference Bias in LLM-as-a-Judge" (`arXiv:2410.21819`), "Justice or Prejudice?" (`arXiv:2410.02736`). Mitigations: randomize order, normalize length, **never let the same model generate and judge**, validate the judge against human labels.
- **For tool selection specifically:** prefer programmatic, state- or AST-based checks (BFCL, τ-bench) over LLM-judged free text wherever the outcome is verifiable; reserve LLM-as-judge for genuinely subjective dimensions and audit it.

**Sources:** github.com/sierra-research/tau2-bench · arxiv.org/abs/{2406.12045,2506.07982,2307.03172,2505.10570,2601.06112,2508.20453,2512.24565,2410.21819,2410.02736} · sierra.ai/blog/benchmarking-agents-in-collaborative-real-world-scenarios · trychroma.com/research/context-rot · gorilla.cs.berkeley.edu/leaderboard.html · proceedings.mlr.press/v267/patil25a.html

---

## 10. Key Sources and Named Researchers (index)

### Primary engineering guides
| Source | Org | Note |
|---|---|---|
| _Writing effective tools for AI agents_ | Anthropic Engineering | Tool design canon |
| _Effective context engineering for AI agents_ | Anthropic Engineering | Compaction, sub-agent summaries |
| _Code execution with MCP_ (Nov 2025) | Anthropic Engineering | 98.7% token-saving example |
| _Building Effective Agents_ | Anthropic Research | The 5-pattern taxonomy |
| _Advanced tool use_ / _Multi-agent research system_ | Anthropic Engineering | Programmatic tools, orchestrator-worker |
| OpenAI Agents SDK docs + Orchestration guide + Macro-Evals cookbook | OpenAI | Handoffs, guardrails, trace grading |
| Gemini function-calling / Gemini 3 / tooling-updates | Google | Modes, thought signatures, built-in tools |
| AI SDK 6 blog + ai-sdk.dev docs | Vercel | `ToolLoopAgent`, `stopWhen`, `prepareStep` |
| LangGraph persistence / multi-agent / supervisor / swarm | LangChain | Graph orchestration, durable execution |

### Research / papers
| Work | Authors | Venue |
|---|---|---|
| _Why Do Multi-Agent LLM Systems Fail?_ (MAST) `2503.13657` | Cemri, Pan, Yang, …, Klein, Zaharia, Gonzalez, Stoica (UC Berkeley Sky Lab) | **NeurIPS 2025** |
| _Lost in the Middle_ `2307.03172` | Liu, Lin, Hewitt, Paranjape, Bevilacqua, Petroni, Liang | **TACL 2023** |
| _τ-bench_ `2406.12045` | Yao, Shinn, Razavi, Narasimhan (Sierra) | ICLR/OpenReview |
| _τ²-bench_ `2506.07982` | Barres, Dong, Ray, Si, Narasimhan (Sierra) | preprint |
| _BFCL / From Tool Use to Agentic Evaluation_ | Patil, Mao, Yan, Ji (Berkeley Gorilla) | PMLR/ICML 2025 |
| _RAG-MCP_ `2505.03275` | Gan, Sun | preprint |
| _ScaleMCP_ `2505.06416` | Lumer, Gulati, Subbiah, Basavaraju, Burke (PwC) | preprint |
| _AgentOrchestra_ `2506.12508` | — | preprint |
| _LongFuncEval_ `2505.10570` | Kate, Pedapati, Basu, Rizk, … (IBM Research) | preprint |
| _MCP-Bench_ `2508.20453` | Accenture | NeurIPS 2025 workshop |
| _Context Rot_ | Hong, Troynikov, Huber (Chroma) | industry report |

### Practitioners / voices
- **Harrison Chase** (LangChain) — ambient agents, agent inbox, durable orchestration.
- **Bret Taylor** & **Clay Bavor** (Sierra) — reliability-first agent evaluation (τ-bench line).
- **Steve Kinney** — _Agent Loops_ (production loop-hardening synthesis).

---

## 11. Caveats (honesty ledger)

- **Vendor examples are directional, not audited.** Anthropic's 98.7% token reduction, OpenAI's AgentKit ROI, Google's latency claims, and LangChain's adoption survey are first-party. Treat magnitudes as illustrative.
- **Preprint vs. peer-reviewed.** Only MAST (NeurIPS 2025), Lost-in-the-Middle (TACL 2023), and BFCL (PMLR) are peer-reviewed here. RAG-MCP, ScaleMCP, τ²-bench, LongFuncEval, AgentOrchestra, MCPAgentBench, and the LLM-judge-bias papers are preprints. **ReliabilityBench (`2601.06112`) is a single-author Jan-2026 preprint — low provenance; do not cite its numbers as established.**
- **Single-model / single-benchmark results** (notably RAG-MCP's 43% vs 13%) may not generalize.
- **API surface drifts fast.** Provider tool version strings (`code_execution_20250825`, etc.), `max_turns` default, Gemini 3.x BFCL standing, exact ADK A2A symbols, and AI SDK 6 GA status were flagged unverified in research and should be re-checked against live docs before you pin them in code.
- **The through-line that *is* well-supported:** fewer/higher-level tools; keep intermediate results out of context; retrieve tools rather than load all; start single-agent and add agents only for real isolation; instrument handoffs; measure reliability (pass^k) on your own evals, not one-shot accuracy. These are corroborated across multiple independent primary and peer-reviewed sources.
