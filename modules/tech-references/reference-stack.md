# Reference Stack — production AI/agentic web apps (November 2026)

> [!note] Vendor-neutral distillation
> A generalized reference stack: version lines and the *reasons* behind them, with no
> client- or vendor-specific provenance. Treat each version as a grounded default to
> re-verify (changelog / release note) before you pin it — not a mandate.

This is a **grounded default, not a mandate.** Deviate when you have a reason, and write it down (ADR). "Latest" = current stable as of **2026-11-06** (checked with `npm view <pkg> version`); re-verify before pinning.

---

## The stack at a glance

| Layer | Choice | Line (Nov 2026) | Why / pattern (one-liner) |
|---|---|---|---|
| App framework | **Next.js** | 16.4.x (App Router only) | React Compiler is opt-in (`babel-plugin-react-compiler` + `reactCompiler: true`, [docs](https://nextjs.org/docs/app/api-reference/config/next-config-js/reactCompiler)); `middleware.ts` → `proxy.ts`, Node.js runtime by default ([docs](https://nextjs.org/docs/app/api-reference/file-conventions/proxy)). |
| UI runtime | **React + React DOM** | 19.3.x | Server/client components, `useActionState`, `use()`. |
| Package manager | **pnpm** | 12.x | `minimumReleaseAge` defaults to 1440 min: a version must be a day old before install ([docs](https://pnpm.io/settings/dependency-resolution)). Keep the supply-chain guard on. |
| Node runtime | **Node 26 LTS** | 26.x (Active LTS since 2026-10-28) | Node 24 is in maintenance (EOL 2028-04-30); Node 20 EOL'd 2026-04-30 ([schedule](https://github.com/nodejs/Release/blob/main/schedule.json)). Pin local and CI to the same major. |
| Language | **TypeScript** | 6.0.x → 7.x | TS 7 (native Go port) is stable (7.0.x); 6.0 is the last JS-based line. Move once your lint, test and framework tooling supports 7. |
| Lint | **ESLint** + `eslint-config-next` | 10.x | ESLint 9 reached EOL 2026-08-06 ([endoflife.date](https://endoflife.date/eslint)). Flat config default. |
| Validation | **Zod** | 4.x | Boundary-parsing only — parse at the edges, trust internally. |
| Styling | **Tailwind CSS** | 4.3.x | CSS-first config (`@theme inline`, `@source`); no JS config file. |
| UI primitives | **Radix UI + shadcn/ui** | current | `cva` + `tailwind-merge`; package your own design system. |
| Animation | **Motion** (was framer-motion) | 14.x | `LayoutGroup` + `layoutId` for shared-element transitions. |
| Server cache | **TanStack Query** | 5.x | Server-state cache; pair with a transient UI store. |
| Client store | **Zustand** | 5.x | Transient UI state, `persist`-partialized. Pick one cache story. |
| Forms | **react-hook-form** + resolvers | RHF 7.x (8 in beta) | Standard-Schema resolvers integrate Zod 4 identically. |
| Multi-provider AI SDK | **Vercel AI SDK** (`ai`) | 7.x | Node 22+, ESM-only. v7 renames `system`→`instructions`, `onFinish`→`onEnd`, `needsApproval`→`toolApproval`, `stepCountIs`→`isStepCount`; set `stopWhen` explicitly ([v7 migration guide](https://ai-sdk.dev/docs/migration-guides/migration-guide-7-0)). |
| Direct provider | **OpenAI Responses API** + strict Structured Outputs | `openai-node` 7.x | Schema-first; strip unsupported JSON-Schema keywords in strict mode. |
| Database | **Supabase Postgres** | 17.x | Postgres + RLS + Realtime + Auth + Storage. |
| ORM | **Drizzle** | 0.45.x | Repository-everything; `drizzle-kit` for migrate/studio only. |
| Migrations | **Hand-authored SQL + committed journal** | — | Don't let the ORM generate-and-forget. |
| Background work | **Inngest** | 4.x | Event-driven; `concurrency.key` per tenant + retries. |
| Auth | **Better Auth** (self-hosted) or a managed provider | 1.x | Auth.js is maintained by the Better Auth team since 2025-09, which recommends Better Auth for new projects ([announcement](https://www.better-auth.com/blog/authjs-joins-better-auth)); keep working Auth.js v4 setups. Domain-lock + allow-list; check authz in every route handler and Server Function, not only `proxy.ts`: a matcher change can silently remove proxy coverage ([Next.js docs](https://nextjs.org/docs/app/api-reference/file-conventions/proxy#execution-order)). |
| Hosting (web) | **Vercel** | — | Preview-per-PR. Long-running services → a container runtime. |
| Observability | **OpenTelemetry** (`@vercel/otel`) | — | Respect the runtime; register `@ai-sdk/otel` for model and tool-call spans (moved out of `ai` core in v7). |
| Media | **fal.ai** | — | One API across many image/video models. Model names change monthly: keep them in config, not code. |
| Web data | **Tavily** | — | Crawl + Search + Extract; persist provider request/response JSON. |
| Test runner | **Vitest** | 5.x | Needs Node `^22.12`, `^24` or `>=26`. Node-env, no jsdom; tests live next to code. |
| CI | **GitHub Actions** | — | CI enforces the knowledge layer, not just the build. |

---

## SDK selection — which agent SDK, and when

| SDK | Pick it for | Trade-off |
|---|---|---|
| **Vercel AI SDK + ai-elements** | Web apps & complex systems with little code; provider-agnostic; quick MVPs | The default for most web work. |
| **Claude Agent SDK** (`@anthropic-ai/claude-agent-sdk`) | Long-running agents on Claude Code's harness: built-in tools, permissions, hooks, subagents, sessions ([overview](https://code.claude.com/docs/en/agent-sdk/overview)) | Claude only. |
| **Google ADK** | Products that live in the Google Cloud ecosystem | Most of its value comes from Google Cloud deployment and managed services; choose it for the ecosystem. |

**Rule of thumb:** Vercel for web apps & quick MVPs (provider-agnostic) · Claude Agent SDK for long-horizon agents on Claude Code's harness · Google ADK when you're already all-in on GCP.

---

## Standing watch-list (re-check before pinning)

- **Node 24 → 26 LTS** for new work; move existing services off 24 before EOL 2028-04-30. Anything still on 20 or 25 is past EOL — highest priority.
- **ESLint 9 → 10** — 9 is past EOL (2026-08-06).
- **TypeScript 6.0 → 7** — once lint, test and framework tooling support 7.
- **AI SDK 6 → 7** — migrate deprecated aliases (`system`, `onFinish`, `needsApproval`) while the diff is small.
- **react-hook-form 7 → 8** — 8 in beta, stay on 7 for now.
- Always re-verify "latest" at install time — this file is a snapshot dated 2026-11-06.

> Versions drift weekly. Ask Claude to re-check against current docs (Context7 / web search) and **flag this file as outdated** — see the workflow in [`index.md`](index.md).
