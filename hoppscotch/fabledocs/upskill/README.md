# Hoppscotch Upskill Curriculum

A training lab built from the real Hoppscotch codebase for one learner: a junior fullstack JS engineer (React/Node/TS-style CRUD background) who wants to reach mid-level fast, build senior judgment, and pass interviews for mid-level fullstack roles.

Every page teaches two things at once:

1. **This codebase** — where things live, how its real flows work, with exact file/line anchors.
2. **The transferable skill** — why the pattern exists, when it fails, and how to explain it in an interview.

## What this repo is

Hoppscotch is an open-source API development platform (the popular Postman alternative). It is a pnpm monorepo of 12 packages: a Vue 3 frontend core ([hoppscotch-common](../../packages/hoppscotch-common)) shared by web, desktop, and admin shells; a NestJS + Apollo GraphQL + Prisma/PostgreSQL backend ([hoppscotch-backend](../../packages/hoppscotch-backend)) powering accounts, teams, cloud sync, real-time collaboration, and a public mock-server surface; a versioned data-schema package ([hoppscotch-data](../../packages/hoppscotch-data)) that migrates 18 generations of request formats; a sandbox ([hoppscotch-js-sandbox](../../packages/hoppscotch-js-sandbox)) that safely executes user-written pre-request/test scripts; a CI-friendly CLI ([hoppscotch-cli](../../packages/hoppscotch-cli)); and a Tauri-based desktop/interceptor stack (kernel, relay, agent, desktop). The domain is unusually instructive: the product itself is about HTTP, so the codebase is dense with auth flows, header handling, sandboxing, and security boundaries — the exact topics interviews probe.

Frontend state mixes two generations (legacy RxJS `newstore` dispatchers and newer `dioc` services), error handling is `fp-ts` Either/Option throughout, and persistence enforces ordering invariants with row locks and retries. You will see both strong patterns and honest legacy seams — both are teaching material.

## How to use it

- **One weekend** — do [00-fast-track.md](00-fast-track.md) only.
- **Two weeks** — fast track, then [01-codebase-cartography](01-codebase-cartography/README.md), [05-key-flows](01-codebase-cartography/05-key-flows.md), the [pattern catalog](03-architecture-and-patterns/05-pattern-catalog.md), and 3–4 drills from [04-code-reading-gym](04-code-reading-gym/README.md).
- **Eight weeks** — modules in order, one per ~5 days, doing every drill and 2 tickets from [06-contribution-practice](06-contribution-practice/README.md).
- **Ongoing contributor** — live in [06-contribution-practice](06-contribution-practice/README.md) and [07-career-and-collaboration](07-career-and-collaboration/README.md), returning to module 03 and 05 as references.

### Recommended paths by profile

| You are | Start | Then |
| --- | --- | --- |
| Brand-new junior | [00-fast-track.md](00-fast-track.md) | 01 → 02 → 04 (drills) → easy tickets |
| Junior who knows Vue/Nest | [01-.../05-key-flows.md](01-codebase-cartography/05-key-flows.md) | 03 → 04 → 05 → mid tickets |
| Mid-level, new to this repo | [01-.../01-system-map.md](01-codebase-cartography/01-system-map.md) | 03-.../06-architecture-critique.md → senior projects |
| Senior doing architecture review | [03-.../06-architecture-critique.md](03-architecture-and-patterns/06-architecture-critique.md) | [09-reference/risk-register.md](09-reference/risk-register.md) |
| **Interview in two weeks** | [08-interview-prep/07-two-week-cram-plan.md](08-interview-prep/07-two-week-cram-plan.md) | Follow it day by day |

## Conventions

- **Anchors**: `path#Lx-Ly` links point at real code, verified at writing time (see [verification log](09-reference/verification-log.md)). Line numbers drift as the repo moves; the function name is your fallback.
- **Fake code**: every invented snippet begins with `// Illustrative fake code: not from this repo`. Everything else quoted is real.
- **Verified vs inferred**: commands marked **verified** were run; **inferred** come from scripts/CI config and were not executed here.
- **Drills and self-grading**: most sections end with a drill and a Basic/Solid/Strong rubric. Grade yourself honestly — the rubric is the point, not the answer.
- **Investigate labels**: suspicious code is labeled "investigate" or "possible risk," never asserted as a bug unless confirmed.

## The mindset ladder

- **Junior asks**: "How do I make it work?"
- **Mid-level asks**: "Is this the right pattern? What breaks it?"
- **Senior asks**: "What does this commit us to, who pays the cost, and how do we reduce the risk?"

Mid-level interview loops are testing exactly the second and third questions. When a page here asks you to name an invariant, a failure mode, or an alternative design, that is interview practice, not ceremony.

## Module index

| Module | What you get |
| --- | --- |
| [00-fast-track.md](00-fast-track.md) | Weekend orientation: run it, trace two flows, make one safe change |
| [01-codebase-cartography](01-codebase-cartography/README.md) | System map, reading order, glossary, tooling map, 7 key flows |
| [02-stack-and-language-mastery](02-stack-and-language-mastery/README.md) | JS/TS runtime, Vue mental models, type contracts, build tooling |
| [03-architecture-and-patterns](03-architecture-and-patterns/README.md) | Boundaries, data model, auth, async/reliability, 16 pattern cards, critique |
| [04-code-reading-gym](04-code-reading-gym/README.md) | Annotation drills, trace tables, fake-code contrasts, review katas |
| [05-quality-engineering](05-quality-engineering/README.md) | Testing strategy/recipes, debugging method, performance, security, observability |
| [06-contribution-practice](06-contribution-practice/README.md) | Junior tickets, mid-level features, senior projects, refactor katas |
| [07-career-and-collaboration](07-career-and-collaboration/README.md) | Review mindset, PR/RFC writing, maintainer communication |
| [08-interview-prep](08-interview-prep/README.md) | 50+ question cards anchored to this repo, system design walkthrough, STAR stories, cram plan |
| [09-reference](09-reference/verification-log.md) | Command cheatsheet, risk register, rubrics, verification log |
