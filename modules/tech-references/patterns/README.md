# patterns — vendor-neutral production patterns

> Two complete reference pages distilled from real production AI/agentic codebases — with all client names, file paths, and commit SHAs removed. Each entry is a well-known engineering practice stated plainly: the *what*, the *why it pays off* (or *why it hurts*), a minimal generic example, and where to skip it. Use them as defaults for a new build.

- [`production-patterns.md`](production-patterns.md) — patterns worth lifting into a new build.
- [`anti-patterns.md`](anti-patterns.md) — traps to avoid, each paired with the better pattern.

Past-MVP invariants (audit log, request id, error taxonomy, idempotent writes, erasure, secrets) and the MVP → enterprise scaling path: see also the [agentic harness spec §17–§18](../../agentic-harness/KNOWLEDGE_MGMT_SETUP_SPEC.md#17-production-readiness-invariants-past-mvp).
