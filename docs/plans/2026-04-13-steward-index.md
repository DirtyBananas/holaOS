# Steward: Spec Index

*Spec-driven development artifacts for [DirtyBananas/Steward](https://github.com/DirtyBananas/Steward) — a local-first personal memory and retrieval system.*

---

## Documents

| Document | Purpose |
|----------|---------|
| [Vision](2026-04-13-steward-vision.md) | Why Steward exists. The fragmentation diagnosis, prospector psychology, core principles, and philosophical anchors. Read this first. |
| [Requirements](2026-04-13-steward-requirements.md) | What Steward must do. User stories, functional and non-functional requirements, source matrix, v1 acceptance criteria. |
| [Architecture](2026-04-13-steward-architecture.md) | How Steward is built. 5-layer architecture, normalized item schema, SQLite schema, retrieval pipeline, HTTP API surface, source adapter contract, technology choices. |
| [Implementation Plan](2026-04-13-steward-implementation-plan.md) | When to build what. Phases 0-7 with validation gates, effort estimates, tech stack per phase, anti-patterns to avoid. |
| [Risks and Glossary](2026-04-13-steward-risks-and-glossary.md) | What could go wrong and what the words mean. 12 risks with mitigations, 13 open questions, 36 glossary terms. |

## Reading Order

1. **Vision** — understand the problem and the principles
2. **Requirements** — understand what success looks like
3. **Architecture** — understand the technical shape
4. **Implementation Plan** — understand the build sequence
5. **Risks and Glossary** — understand the hazards and the vocabulary

## Origin

These documents were distilled from a long-form conversation about LLM productivity, digital fragmentation, and personal memory systems. The conversation surfaced the core insight that the user is a *prospector* (past work has genuine operational value) suffering from *fragmentation* (information scattered across ~60 possible locations along channel, context, and time axes), and that the solution is a *retrieval and integration layer* that meets data where it lives rather than demanding migration.

## Key Principle

> The prospector log is the signal. The automation is the plumbing.
