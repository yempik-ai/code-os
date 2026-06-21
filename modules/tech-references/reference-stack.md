# Reference Stack — production AI/agentic web apps (June 2026)

> [!note] Vendor-neutral distillation
> A generalized reference stack: version lines and the *reasons* behind them, with no
> client- or vendor-specific provenance. Treat each version as a grounded default to
> re-verify (changelog / release note) before you pin it — not a mandate.

This is a **grounded default, not a mandate.** Deviate when you have a reason, and write it down (ADR). "Latest" = current stable as of **2026-06-03**; re-verify before pinning.

---

## The stack at a glance

| Layer | Choice | Line (Jun 2026) | Why / pattern (one-liner) |
|---|---|---|---|
| App framework | **Next.js** | 16.2.x (App Router only) | React Compiler on by default; `middleware.ts` → `proxy.ts`. |
| UI runtime | **React + React DOM** | 19.2.x | Server/client components, `useActionState`, `use()`. |
| Package manager | **pnpm** | 11.x | Needs Node 22+; supply-chain `minimumReleaseAge` guard. |
| Node runtime | **Node 24 LTS** | 24.x (Active LTS) | ⚠️ Do not ship on Node 25 (EOL 2026-06-01). |
| Language | **TypeScript** | 6.0.x | Last JS-based release before Go-native TS 7; faster builds. |
| Lint | **ESLint** + `eslint-config-next` | 9.x → migrate to 10.x | ⚠️ ESLint v9 EOL 2026-08-06. Flat config default. |
| Validation | **Zod** | 4.x | Boundary-parsing only — parse at the edges, trust internally. |
| Styling | **Tailwind CSS** | 4.3.x | CSS-first config (`@theme inline`, `@source`); no JS config file. |
| UI primitives | **Radix UI + shadcn/ui** | current | `cva` + `tailwind-merge`; package your own design system. |
| Animation | **Motion** (was framer-motion) | 12.x | `LayoutGroup` + `layoutId` for shared-element transitions. |
| Server cache | **TanStack Query** | 5.x | Server-state cache; pair with a transient UI store. |
| Client store | **Zustand** | 5.x | Transient UI state, `persist`-partialized. Pick one cache story. |
| Forms | **react-hook-form** + resolvers | RHF 7.x (8 in beta) | Standard-Schema resolvers integrate Zod 4 identically. |
| Multi-provider AI SDK | **Vercel AI SDK** (`ai`) | 6.x | `streamText`, `UIMessage.parts`, `stepCountIs`; provider-agnostic. |
| Direct provider | **OpenAI Responses API** + strict Structured Outputs | `openai-node` 6.x | Schema-first; strip unsupported JSON-Schema keywords in strict mode. |
| Database | **Supabase Postgres** | 17.x | Postgres + RLS + Realtime + Auth + Storage. |
| ORM | **Drizzle** | 0.45.x | Repository-everything; `drizzle-kit` for migrate/studio only. |
| Migrations | **Hand-authored SQL + committed journal** | — | Don't let the ORM generate-and-forget. |
| Background work | **Inngest** | 4.x | Event-driven; `concurrency.key` per tenant + retries. |
| Auth | **NextAuth / Auth.js** | v4 (plan v5) | Domain-lock + allow-list; defense-in-depth at proxy *and* callback. |
| Hosting (web) | **Vercel** | — | Preview-per-PR. Long-running services → a container runtime. |
| Observability | **OpenTelemetry** (`@vercel/otel`) | — | Respect the runtime; enable AI SDK telemetry for tool-call spans. |
| Media | **fal.ai** | — | FLUX.2 (image), Veo 3.1 / Kling 3.0 Pro (video). |
| Web data | **Tavily** | — | Crawl + Search + Extract; persist provider request/response JSON. |
| Test runner | **Vitest** | 4.1.x | Node-env, no jsdom; tests live next to code. |
| CI | **GitHub Actions** | — | CI enforces the knowledge layer, not just the build. |

---

## SDK selection — which agent SDK, and when

| SDK | Pick it for | Trade-off |
|---|---|---|
| **Vercel AI SDK + ai-elements** | Web apps & complex systems with little code; provider-agnostic; quick MVPs | The default for most web work. |
| **Anthropic Agents SDK** | Anthropic-ecosystem lock-in, complex multi-agent systems, long-running workflows | Best when you commit to Claude. |
| **Google Agents SDK / ADK** | Leveraging the Google ecosystem; betting on its platform/maintenance | Gemini models lag for now — choose for ecosystem, not raw model quality. |

**Rule of thumb:** Vercel for web apps & quick MVPs (provider-agnostic) · Anthropic for complex multi-agent + long-horizon workflows · Google when you're already all-in on GCP.

---

## Standing watch-list (re-check before pinning)

- **Node 25 → 24 LTS** (Node 25 EOL'd 2026-06-01) — highest priority.
- **ESLint 9 → 10** before EOL 2026-08-06.
- **TypeScript 6.0 line** — verify exact patch; TS 7 (Go-native) on the horizon.
- **react-hook-form 7 → 8** — 8 in beta, stay on 7 for now.
- Always re-verify "latest" at install time — this file is a snapshot dated 2026-06-03.

> Versions drift weekly. Ask Claude to re-check against current docs (Context7 / web search) and **flag this file as outdated** — see the workflow in [`index.md`](index.md).
