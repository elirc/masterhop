# Codex Learning Documentation for Hoppscotch

This folder is a documentation-only learning project created to help you study this repository as a working engineer, not as a tourist. The repo is a pnpm monorepo for the Hoppscotch API development ecosystem: a Vue/TypeScript frontend, a NestJS/GraphQL backend, Prisma/PostgreSQL persistence, shared data schemas, desktop and self-host shells, CLI tooling, a JavaScript sandbox, and supporting packages.

No existing docs were changed. All notes here are intentionally separate from the product docs.

## The Four Suites

- `architectural-cartographer/` is the top-down map. Use it when you want to understand how the system is shaped, where to read first, and how frontend, sync, API, and database pieces connect.
- `mission-learning-path/` is active training. Use it when you want exercises that force tracing, annotation, debugging, review, and explanation.
- `user-story-build-path/` is a ticket ladder. Use it when you want realistic features to implement yourself without having Codex do the whole job.
- `technology-best-practices/` is the stack study guide. Use it when you want to learn Vue, TypeScript, GraphQL, NestJS, Prisma, RxJS-style stores, validation, and tests from real code in this repo.

## Recommended Reading Order

1. Read `architectural-cartographer/00-reading-map.md`.
2. Work through `architectural-cartographer/01-junior-engineer.md`.
3. Do Missions 1-8 in `mission-learning-path/01-tier-1-junior-missions.md`.
4. Read `technology-best-practices/01-technology-map.md`.
5. Pick Story 1 from `user-story-build-path/01-stories.md` and plan it before touching code.
6. Return to the mid-level and senior docs once the basic file map feels familiar.

## How to Use These Docs With Codex

Use Codex as a mentor and reviewer before using it as an implementer. A strong learning loop is:

1. Read the cited files yourself.
2. Write your own 5-10 sentence trace.
3. Ask Codex to challenge the trace.
4. Implement a small change yourself.
5. Ask Codex to review the diff for bugs, tests, and architecture drift.

Useful prompt:

```text
I am studying this Hoppscotch code path: [files]. Ask me questions that reveal whether I understand the flow. Do not give me the answer first. Give hints in stages.
```

## How to Use the User Stories Without Outsourcing the Thinking

For each story, write your own implementation plan before asking Codex. Then ask Codex to compare your plan against existing patterns. If you ask for code, request one file or one function at a time and require citations to existing patterns.

## 30-Day Self-Study Shape

- Days 1-5: Entry points, package map, routing, one simple component.
- Days 6-10: REST request UI, tabs, local state, request execution.
- Days 11-15: History sync, GraphQL client, backend resolver/service/database flow.
- Days 16-20: Auth, guards, validation, Prisma models, tests.
- Days 21-25: Review hypothetical diffs, inject bugs mentally, write missing tests in a scratch branch.
- Days 26-30: Explain the system out loud, plan a feature, defend trade-offs, and review your own implementation.

