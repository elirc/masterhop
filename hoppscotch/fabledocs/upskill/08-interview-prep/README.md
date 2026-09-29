# 08 — Interview Prep

Target: mid-level fullstack JS loops. Typical structure and where this module arms you:

| Round | Typical form | Your prep |
| --- | --- | --- |
| Screen | 30 min, resume + one technical thread | rehearsed repo summary (below) + 2 STAR stories |
| Technical deep-dive | 45–60 min JS/TS/Node | [01-js-ts-node-deep-dive.md](01-js-ts-node-deep-dive.md) (14 cards) |
| Frontend | framework mental models | [02-frontend-framework-questions.md](02-frontend-framework-questions.md) (12 cards) |
| API & data | REST/GraphQL, schema, transactions | [03-api-and-data-modeling-questions.md](03-api-and-data-modeling-questions.md) (15 cards) |
| System design | 40 min whiteboard | [04-system-design-from-this-repo.md](04-system-design-from-this-repo.md) |
| Practical / debugging / review | live exercises | [05-debugging-and-code-review-rounds.md](05-debugging-and-code-review-rounds.md) |
| Behavioral | STAR prompts | [06-behavioral-star-stories.md](06-behavioral-star-stories.md) (9 worksheets) |
| Two weeks out? | | [07-two-week-cram-plan.md](07-two-week-cram-plan.md) |

## The golden rule

Answer with **a concrete example, a tradeoff, and a failure mode** — never a definition alone. This repo is your example bank: 41 of the 50 question cards here are anchored to code you can describe from memory. "In a codebase I've studied deeply…" is a legitimate and strong framing — you don't need to have merged the code to reason about it honestly (and say which it is if asked).

## Your 60-second repo summary (rehearse until fluid)

*"I did a deep study of Hoppscotch — the open-source Postman alternative. It's a pnpm monorepo: Vue 3 core shared across web/desktop shells via dependency-injected platform implementations, a NestJS + GraphQL + Prisma backend for teams and sync, a versioned schema package that migrates 18 generations of saved request formats, and a sandbox package that runs user scripts in workers in the browser and isolated runtimes in the CLI. The parts I know best are the auth flow — magic links with rotated, hashed refresh tokens — the team RBAC guards, and the collection-ordering system, which uses row locks plus unique constraints plus bounded retries. I can also tell you the three things I'd fix."*

That last sentence baits the exact follow-up you want ([critique P1–P3](../03-architecture-and-patterns/06-architecture-critique.md)).
