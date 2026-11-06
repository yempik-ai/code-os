# Tool Orchestration Best Practices and Resources for LLM Agents

> Snapshot first compiled **June 2026**. Re-verified **2026-11-06** against the live docs linked inline: §1, §2, §3.3, §3.5, §3.6, §4.1–§4.2, §5 and §6.1 were rewritten where claims no longer held; items carried over unchanged from June are labelled *(June 2026 snapshot)*. Re-verify against live docs before pinning anything. The principles hold; the pins drift.
> Scope note: this compendium covers orchestration *inside the products you build*. For orchestrating your own development workflow across vendor CLIs, see `../multi-model-orchestration/`.
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
- **Combine functions that are always called in sequence, and do not make the model fill arguments your code already knows:** `submit_refund()` takes no parameters, and your code passes the order id ([OpenAI function calling](https://developers.openai.com/api/docs/guides/function-calling)).
- **Make invalid states unrepresentable** with enums and object structure: `toggle_light(on: bool, off: bool)` admits a call that is both.
- **Keep the set visible at the start of a turn small.** OpenAI: fewer than 20 functions, as a soft suggestion. Google: 10–20 active tools. Defer the rest (§6).

Sources: Anthropic, _Writing effective tools for AI agents_; Anthropic, _Effective context engineering for AI agents_; Anthropic, _Building Effective Agents_; OpenAI function-calling guide.

### 1.2 Context and token management

- **Make tool responses concise.** Return only relevant fields. Use pagination, range selection, filtering, and/or truncation with sensible defaults. (Anthropic's Slack example cut a response from 206 to ~72 tokens, ~⅔ fewer.) When you truncate, say so in the result and tell the agent how to narrow the query. Claude Code itself caps MCP tool output at 25,000 tokens by default and warns above 10,000 (`MAX_MCP_OUTPUT_TOKENS`, [MCP docs](https://code.claude.com/docs/en/mcp)).
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
- **Maximize prompt/KV-cache hits** (stable system prompt + tool definitions at the front; append-only history) and compact context around ~80% window usage. Where the API offers it, restrict callable tools with an allow-list that leaves the definitions in place (OpenAI `allowed_tools`, §3.3); removing tools from the request per step (AI SDK `activeTools`, §5.2) narrows selection but changes the cached prefix.
- **Make every side-effecting tool idempotent or approval-gated.** A retried step must not double-charge, double-send or double-write.

---

## 2. Anthropic / Claude

Anthropic's engineering publications are the most-cited primary sources in this space and define much of the cross-provider consensus in §1.

### 2.1 Tool naming and namespacing

- **Namespacing conventions have measurable, non-trivial effects on tool-use eval performance.** Prefix/service namespacing (`asana_search`) versus suffix/resource namespacing (`asana_projects_search`) can change results. Anthropic frames this explicitly as a measured eval finding and advises validating with **your own evals**, because effects vary by model.

### 2.2 Multi-agent orchestration on Claude

- Use **specialized sub-agents with clean context windows, coordinated by a main (lead) agent.** Each sub-agent returns only a condensed, distilled summary — **often 1,000–2,000 tokens**. The main agent holds the high-level plan; sub-agents do deep technical work in isolation.
- This orchestrator-worker pattern (lead plans → spins up 3–5 parallel specialized sub-agents, each with its own context window, each returning a condensed summary) is the basis of Anthropic's multi-agent research system. On Anthropic's internal research eval it outperformed single-agent Claude Opus 4 by 90.2%.
- **Cost scales with it.** Agents use about 4× the tokens of chat; multi-agent systems about 15×. On BrowseComp, token usage alone explained 80% of the performance variance. Use multi-agent only where the work splits into independent breadth worth the tokens. Early prompt changes have large effects (30% to 80% success), and about 20 realistic queries are enough to see them ([multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system); vendor, internal eval).

### 2.3 Code execution with MCP — the headline token saving

- Having the agent **write code that calls MCP servers** and filter/transform results in the execution environment (rather than passing raw tool output through the model) keeps intermediate results out of context. A documented Google Drive → Salesforce workflow reduced token usage from **150,000 → 2,000 tokens (98.7%)**: the large transcript never entered the model context; only the small final result did.
- This is the recommended pattern for high-volume, multi-tool workflows. *(Anthropic's own illustrative example — accurately attributed, not an independently audited benchmark.)*

### 2.4 Advanced tool use (provider features worth knowing)

- **Tool Search Tool.** Mark tools `defer_loading: true`; Claude sees only the search tool plus the non-deferred tools and loads 3–5 relevant definitions on demand. In Anthropic's example, context spent before any work drops from about 77K tokens to about 8.7K. Keep your three to five most-used tools always loaded and defer the rest ([advanced tool use](https://www.anthropic.com/engineering/advanced-tool-use); vendor example).
- Claude exposes **programmatic / code-execution-callable tools**: a tool can be marked callable from within the code-execution sandbox, so the model orchestrates many tool calls in code without round-tripping each through the context window. Provider-executed tool version strings change often; re-verify before you pin.

### 2.5 Writing an MCP server for Claude Code

Claude Code defers MCP tool definitions by default: at session start only tool names and server instructions load, and Claude fetches schemas through tool search ([MCP docs](https://code.claude.com/docs/en/mcp)).

- Server instructions decide whether Claude searches for your tools. State the category of tasks, when to search, and the key capabilities.
- Claude Code cuts server instructions and upfront tool descriptions at 4,096 characters, and descriptions loaded through tool search at 16,384. Tool search matches against the text within the shorter limit, so put the words a user would search for first.
- Set `"anthropic/alwaysLoad": true` in a tool's `_meta` for the one or two tools that must always be visible. Leave everything else deferred.

Sources: Anthropic, _Writing effective tools for AI agents_; _Effective context engineering for AI agents_; _Code execution with MCP_ (Nov 2025); _Building Effective Agents_; _Advanced tool use_; _Multi-agent research system_ (anthropic.com/engineering/*, anthropic.com/research/building-effective-agents); Claude Code MCP docs.

---

## 3. OpenAI

OpenAI's tool-orchestration stack as of mid-2026 has three layers: the **Responses API** (the function/tool-calling primitive and host for built-in tools), the open-source **Agents SDK** (runner loop, handoffs, guardrails, sessions, tracing — the production successor to the experimental *Swarm*), and **AgentKit** (a higher-level visual/managed layer announced at DevDay, Oct 6 2025). Build new work on Responses.

### 3.1 Two orchestration models (OpenAI explicitly recommends mixing them)

- **LLM-driven orchestration** — the model plans, reasons, and chooses steps via instructions, tools, and handoffs. Best for open-ended tasks. OpenAI's stated best practices: invest in good prompts, use specialized (not generalist) agents, let agents introspect/self-improve, and "invest in evals."
- **Code/deterministic orchestration** — flow controlled in code for predictable speed/cost/quality. Patterns: classify with structured outputs then branch; chain agents transforming each output into the next input; `while` feedback loops; parallel execution via `asyncio.gather`.
- Core design principle (conservative): *"Start with one agent whenever you can. Add specialists only when they materially improve capability isolation, policy isolation, prompt clarity, or trace legibility."*

### 3.2 Agents-as-tools vs. handoffs — *who owns the final response*

- **Agents as tools (manager pattern):** a manager agent stays in control and calls specialists as bounded capabilities via `Agent.as_tool()` (TS: `agent.asTool()`); the manager synthesizes the final answer. `as_tool()` parameters: `tool_name`, `tool_description`, `parameters` (Pydantic input), `custom_output_extractor`, `needs_approval` (approval gate), `is_enabled` (bool or callable), `on_stream`.
- **Handoffs:** a triage/router agent transfers control; the specialist *becomes the active agent* for the rest of the turn and owns the user-facing response. Declared via the `handoffs` parameter; keep `handoff_description` concise; split only when branches truly need different instructions/tools/policies. Handoffs can carry structured metadata and filtered conversation history.

### 3.3 Responses API function/tool-calling model

- **Strict mode / structured outputs for tool args:** `strict: true` guarantees schema adherence (requires `additionalProperties: false`, all fields in `required`, optional fields modeled as `["string","null"]`). If you omit `strict`, Responses *attempts* to normalise the schema to strict and falls back to best effort when it cannot, shown as `strict: false` on the tool; Chat Completions stays non-strict. Caveat: with a fine-tuned model, parallel calls disable strict for those calls ([function calling](https://developers.openai.com/api/docs/guides/function-calling)).
- **Parallel tool calls:** `parallel_tool_calls` defaults to `true`; set `false` to force at most one tool per turn.
- **`tool_choice`:** `"auto"` (default), `"required"` (force ≥1), `"none"` (forbid), named function (force exactly one), and **`allowed_tools`** to restrict the model to a subset while keeping the full tool definitions cached (prompt-cache-friendly).
- **Wire format:** model emits `function_call` items in the `output` array with `call_id`/`name`/`arguments`; you append `function_call_output` items with the matching `call_id`. Streaming uses `response.function_call_arguments.delta` events.

### 3.4 Built-in hosted tools *(June 2026 snapshot)*

Configured in the `tools` array; the model decides whether to invoke (overridable via `tool_choice`): `web_search`, `file_search` (over OpenAI Vector Stores), `code_interpreter` (hosted sandbox), `image_generation`, `mcp` (remote MCP servers), `function` (custom). Computer use exists as a separate tool; check the current docs for its model and API support before you build on it.

### 3.5 Tool selection at scale

1. **Tool search / deferred loading** — dynamically load only relevant tool definitions instead of front-loading all tools. Group tools into **namespaces** (`crm`, `billing`) and mark them `defer_loading` behind `tool_search`; only `gpt-5.4` and later support `tool_search`. Keep namespace descriptions short and put the detail in function descriptions. Tool definitions are billed as input tokens ([function calling](https://developers.openai.com/api/docs/guides/function-calling)). In the Agents SDK: `ToolSearchTool`, `@function_tool(defer_loading=True)`, and `tool_namespace(name, description, tools=[...])` *(SDK symbols: June 2026 snapshot)*. *(The same deferred-tool pattern Claude Code uses for MCP, §2.5.)*
2. **Model-routed multi-tool orchestration** — the cookbook _Multi-Tool Orchestration with RAG_ shows the model routing between `web_search` and a custom vector-DB function purely from instructions. *(Illustrative demo, not a benchmark.)*

### 3.6 Runner loop, sessions, guardrails, tracing (Agents SDK)

- **Runner:** `Runner.run()` (async → `RunResult`), `run_sync()`, `run_streamed()`. Loop: call LLM → if final output, stop; if handoff, swap active agent; if tool calls, execute, append, continue.
- **`max_turns`:** exceeding raises `MaxTurnsExceeded`; `max_turns=None` disables the cap. `error_handlers={"max_turns": ...}` returns a controlled final output instead (the §1.4 degrade path); the other keys are `"model_refusal"` and `"invalid_final_output"` ([running agents](https://openai.github.io/openai-agents-python/running_agents/)).
- **`RunConfig`:** overrides for model/provider/settings, input/output guardrails, tracing (`tracing_disabled`, `workflow_name`, `trace_id`), `call_model_input_filter`.
- **Guardrails:** `@input_guardrail` / `@output_guardrail` (+ tool-level `@tool_input_guardrail` / `@tool_output_guardrail`); return `GuardrailFunctionOutput(output_info, tripwire_triggered)`; a tripwire raises `InputGuardrailTripwireTriggered` / `OutputGuardrailTripwireTriggered`. Input guardrails can run **in parallel** with the agent (lower latency, may burn tokens before tripping) or blocking; output guardrails run only on the final-output agent.
- **Sessions:** pluggable cross-turn memory — SQLite, SQLAlchemy, Redis, MongoDB, encrypted variants.
- **Tracing:** built-in spans for the agent flow, feeding evals and fine-tuning.

### 3.7 Evals for orchestration

- **AgentKit "Evals for Agents":** step-by-step **trace grading**, per-component datasets, automated prompt optimization, running evals against external models.
- Cookbook **Macro Evals for Agentic Systems** is the most concrete orchestration-eval guidance: distinguishes **per-agent evals** (routing, tool appropriateness, handoff timing, policy compliance) from **macro evals** that cluster patterns across many traces (embeddings → UMAP → HDBSCAN), score `impact_score = prevalence_share × severity_weighted_prevalence`, and diagnose via backward execution-graph traversal. Framing: *"a final answer is only the last event in a longer workflow"* — failures are usually wrong routing / mistimed handoffs, not a single bad response.

**Flags:** AgentKit ROI/benchmark claims are marketing (only a dev-speed anecdote — Ramp "built a procurement agent in a few hours" — could be verified). Cookbook routing results are illustrative, not benchmarked. The `max_turns` default is not stated in the docs. The official AgentKit page 403'd; those details rely on secondary sources *(June 2026 snapshot)*.

**Sources:** openai.github.io/openai-agents-python/{multi_agent,running_agents,guardrails,tools} · developers.openai.com/api/docs/guides/{agents/orchestration,function-calling,tools,tools-computer-use} · developers.openai.com/cookbook/examples/{responses_api/responses_api_tool_orchestration,partners/macro_evals_for_agentic_systems} · openai.com/index/new-tools-for-building-agents · techcrunch.com/2025/10/06/openai-launches-agentkit-…

---

## 4. Google / Gemini

### 4.1 Gemini API function calling (core orchestration)

Google's current request surface for agents is the **Interactions API** ([function calling](https://aistudio.google.com/docs/function-calling), [latest model](https://aistudio.google.com/docs/latest-model)).

- **Function declarations use an OpenAPI 3.0.3 *subset*** (`name`, `description`, `parameters`). Very large/deeply nested schemas may be rejected; remedy is shorter names, less nesting, fewer declarations. Practical guidance: keep **10–20 active tools max** for reliability. *(June 2026 snapshot.)*
- **Function-calling modes** via `tool_choice` in `generation_config`:
  - `auto` — default; model chooses text or a call.
  - `any` — always predicts a function call.
  - `none` — prohibits calls.
  - `validated` — ensures function schema adherence.
  - `allowed_tools` (a `mode` plus a list of `tools`) restricts the callable set.
- **Parallel function calling:** multiple independent calls in one turn; results may arrive in any order. Match each `function_result` to its call by `call_id`.
- **Compositional / sequential function calling:** outputs of one call feed the next; Google's SDKs can run the loop for you.
- **Reliability levers:** detect malformed or invalid calls before executing; validate consequential calls before executing.

### 4.2 Thinking / reasoning interplay with tools

- **Keep conversation state on the server.** Use `previous_interaction_id` for multi-turn and function-calling conversations. Google's SDKs handle thought signatures automatically; if you hand-roll requests, follow Google's thought-signatures guide exactly, because a missing signature on a function-calling turn fails the request.
- **Sampling parameters:** on Google's newest model, `temperature`, `top_p` and `top_k` are ignored; remove them. Do **not** port the old "temperature 0 for tool calls" habit.
- **Thinking control:** `thinking_level` is `low`, `medium` (default) or `high`; test all three and pick the lowest that meets your quality bar. Thinking spends the output budget: Google recommends `max_output_tokens` of at least 32,768, and client timeouts of at least 15 minutes or background execution.
- Gemini supports **multimodal function responses** (nested images/PDF/text). *(June 2026 snapshot.)*

### 4.3 Built-in tools and 2026 tooling updates *(June 2026 snapshot)*

- **Tool combination (Mar 18, 2026):** built-in tools (Google Search grounding, Maps grounding, URL Context, Code Execution, File Search) can now combine with custom function declarations **in a single request**, removing a separate orchestration hop. Claimed latency benefit is **unquantified marketing**.
- **Context circulation:** every built-in tool call/response is preserved so later steps (incl. custom tools) can reason over it; recommended via the **Interactions API**.
- **Grounding with Google Maps** extended to the Gemini 3 family. **URL Context** tool launched experimental. **File Search / RAG** updated (May 5, 2026) for multimodal search + visual citations (`media_id`, `page_numbers` in grounding metadata). **Batch API** moved to **event-driven Webhooks** (May 4, 2026).
- **Interactions API breaking change** (eff. May 26, 2026): `outputs` → `steps`. **Managed Agents** entered public preview (May 19, 2026). *(Model-naming details from the changelog fetch were not independently re-verified field-by-field.)*

### 4.4 Agent Development Kit (ADK) — orchestration primitives *(June 2026 snapshot)*

- **Workflow agents:** `SequentialAgent`, `ParallelAgent`, `LoopAgent`; plus `LlmAgent` (model-driven) and custom `BaseAgent`.
- **Multi-agent hierarchies & agents-as-tools:** agents compose into hierarchies; a sub-agent can be wrapped and exposed as a tool to a parent.
- **Authoring:** YAML or drag-and-drop visual UI; deploy to Cloud Run with one command.
- **A2A (Agent2Agent) protocol:** cross-framework collaboration — agents discover each other via standardized **Agent Cards**, authenticate cryptographically, exchange tasks over **gRPC** (Python, Go, Java). *(Exact symbols like `RemoteA2aAgent` / `to_a2a()` could not be verified from live docs — unverified.)*

### 4.5 Vertex AI Agent Engine / Agent Builder (production) *(June 2026 snapshot)*

- **Agent Engine:** managed runtime — autoscaling, session management, long-running execution, evaluation, **Sessions**, **Memory Bank**, Code Execution; deploy via `adk deploy`.
- **Sessions vs. Memory Bank:** Sessions hold within-conversation state; Memory Bank persists across conversations. Both **GA, billed separately at $0.25 per 1,000 events/memories from Jan 28, 2026.**
- Agent Builder added **enhanced tool governance** (enterprise control over callable tools) — feature, not benchmarked.

### 4.6 Tool-calling reliability vs. competitors (honest assessment)

- **Berkeley Function-Calling Leaderboard (BFCL)** is the standard AST-based eval (serial + parallel calls across Python/Java/JS/REST; V4 extends to agentic/multi-turn).
- **Honest signal:** frontier models ace single-shot calls but **degrade on multi-turn, long-context, and "when not to act" decisions**. Per-model leaderboard figures date within months; read the live leaderboard instead of copying numbers, and Google publishes no quantified tool-reliability numbers in its own announcements. *(Model-specific BFCL figures from June 2026 were cut: they named a previous model generation.)*

**Sources:** ai.google.dev/gemini-api/docs/{function-calling,gemini-3,changelog,maps-grounding} · blog.google/innovation-and-ai/technology/developers-tools/gemini-api-tooling-updates · developers.googleblog.com/en/agent-development-kit-… · adk.dev/a2a · google.github.io/adk-docs/sessions/memory · cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview · cloud.google.com/agent-builder/agent-engine/memory-bank/overview · gorilla.cs.berkeley.edu/leaderboard.html

---

## 5. Vercel AI SDK (v7)

AI SDK 7 is the current major release. Names below follow the [v7 migration guide](https://ai-sdk.dev/docs/migration-guides/migration-guide-7-0), [loop control](https://ai-sdk.dev/docs/agents/loop-control), [tool calling](https://ai-sdk.dev/docs/ai-sdk-core/tools-and-tool-calling) and [MCP](https://ai-sdk.dev/docs/ai-sdk-core/mcp-tools) docs. Node 22+, ESM only.

### 5.1 The Agent abstraction

`ToolLoopAgent` encapsulates the loop (call model → execute tools → append → repeat).

```ts
import { ToolLoopAgent, isStepCount, hasToolCall } from 'ai';

const agent = new ToolLoopAgent({
  model: '<provider>/<model-id>', // a gateway string, or a provider instance
  instructions: 'You answer weather questions.', // was `system` before v7
  tools: { weather: weatherTool, done: doneTool },
  stopWhen: [isStepCount(20), hasToolCall('done')], // any match stops; set it explicitly
});
const { text } = await agent.generate({ prompt: '…' });
```

### 5.2 Loop control: stopWhen, prepareStep

- **Stop conditions:** `isStepCount(n)` (was `stepCountIs`), `hasToolCall(...names)`, `isLoopFinished()` (removes the step limit; pair it with another bound), or a custom predicate over `steps`. `stopWhen` accepts one condition or an array (any match stops).
- **`prepareStep`** runs before each step. It can override `model` (send cheap tool-dispatch steps and expensive synthesis steps to different models), `activeTools` (narrow the tool surface per step), `toolChoice`, `messages` (trim or compact) and call settings. In v7, returned `instructions` and `messages` persist for later steps until another override.
- **Callbacks and results:** `onStepEnd` (was `onStepFinish`), `onEnd` (was `onFinish`); the full event stream is `stream` (was `fullStream`).

### 5.3 Tool definition: `tool()` helper

- `tool({ description, inputSchema, execute })`. `strict: true` is per tool, on providers that support it. `inputExamples` attaches examples; `toModelOutput` transforms a result before the model sees it. `dynamicTool` covers tools without static types (MCP).
- **Execution context** is `context`, split into `runtimeContext` and per-tool `toolsContext` (was `experimental_context`), typed with `contextSchema`.
- **Human in the loop:** the `toolApproval` setting on the call or agent; per-tool `needsApproval` is deprecated.
- **Structured output:** `generateText({ output: Output.object({ schema }) })`, then read `result.output` (`experimental_output` is removed).
- **Provider-executed tools** (Anthropic code execution, OpenAI shell, and others): version strings change often; re-verify them before you pin.

### 5.4 Telemetry

OpenTelemetry span collection is no longer built into the `ai` package. Install `@ai-sdk/otel` and call `registerTelemetry(new OpenTelemetry())` once at startup (in Next.js, in `instrumentation.ts`).

### 5.5 MCP client + provider-agnostic routing

- **MCP:** `createMCPClient` from `@ai-sdk/mcp`. Use HTTP transport (`transport: { type: 'http', url, authProvider }`) in production; stdio is for local servers only. Pass `schemas` to expose a typed subset (reduces tool surface). Redirects default to `'error'` to prevent SSRF; allow-list the OAuth origins you trust. Close the client when done.
- **Routing:** a bare `creator/model` string routes through **Vercel AI Gateway** (budgets, fallbacks, provider order). Combined with `prepareStep`'s `model` override, different loop steps can go to different models.

**Sources:** ai-sdk.dev/docs/{migration-guides/migration-guide-7-0,agents/loop-control,ai-sdk-core/tools-and-tool-calling,ai-sdk-core/mcp-tools} · vercel.com/docs/ai-gateway

---

## 6. Model Context Protocol (MCP)

MCP is the cross-provider connective tissue for tool exposure (Anthropic, OpenAI, Google, and Vercel all consume it). The dominant 2025–2026 research/engineering theme is **scaling tool selection without prompt bloat.**

### 6.1 The core scaling problem

- **Loading all MCP tool definitions upfront increases response time and cost** as tools/servers grow; with thousands of tools, agents process hundreds of thousands of tokens before reading a request. In Anthropic's five-server example, 58 tools consume about 55K tokens before the conversation starts ([advanced tool use](https://www.anthropic.com/engineering/advanced-tool-use); vendor example).
- **The industry answer is deferral, and it has converged.** Claude Code tool search (§2.5), the Claude API Tool Search Tool (§2.4), OpenAI `tool_search` with namespaces (§3.5) and LangGraph's `langgraph-bigtool` (§7.4) all load a small relevant subset on demand. Design servers for it: clear server instructions, distinct tool names, search words early in each description.
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

- **τ-bench** (Yao, Shinn, Razavi, Narasimhan; `arXiv:2406.12045`, Jun 2024; ICLR/OpenReview): dynamic conversations between a simulated user and a tool-equipped agent with written policy, scored by comparing the *final database state* to an annotated goal (not text). Introduces the **pass^k** metric — probability of success on the *same* task across all k independent trials. Headline: SOTA function-calling agents (gpt-4o at the time) succeeded on **<50% of tasks** and were **inconsistent (pass^8 <25% in retail)**.
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
| AI SDK 7 migration guide + ai-sdk.dev docs | Vercel | `ToolLoopAgent`, `stopWhen`, `prepareStep`, `toolApproval` |
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
- **API surface drifts fast.** §1–§2, §3.3, §3.5, §3.6, §4.1–§4.2, §5 and §6.1 were re-checked against live docs on 2026-11-06. Items labelled *(June 2026 snapshot)*, provider tool version strings, the `max_turns` default (not stated in the docs) and exact ADK A2A symbols were not, and should be re-checked before you pin them in code. Model-gated features (OpenAI `tool_search`, Gemini sampling behaviour) depend on the model you call.
- **The through-line that *is* well-supported:** fewer/higher-level tools; keep intermediate results out of context; retrieve tools rather than load all; start single-agent and add agents only for real isolation; instrument handoffs; measure reliability (pass^k) on your own evals, not one-shot accuracy. These are corroborated across multiple independent primary and peer-reviewed sources.
