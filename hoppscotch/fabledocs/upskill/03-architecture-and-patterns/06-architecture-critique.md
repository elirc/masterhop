# Architecture Critique

The senior exercise: judge the system as if you owned it for the next three months. Confirmed observations are anchored; everything else is labeled hypothesis. This file doubles as system-design interview material — cross-linked from [08-interview-prep/04](../08-interview-prep/04-system-design-from-this-repo.md).

## Strongest design choices (steal these)

1. **Platform injection** — one app core, three shells, dependency arrows pointing inward only ([Pattern 12](05-pattern-catalog.md#pattern-12-platform-injection-dependency-inversion-at-package-scale)). This is the load-bearing decision of the whole product line.
2. **Client-owned versioned data schemas** — 18 request-format generations migrating lazily ([rest/index.ts#L80-L113](../../../packages/hoppscotch-data/src/rest/index.ts#L80-L113)); the DB stores blobs, the `data` package owns meaning. Elegant division: relational invariants in SQL, document evolution in code.
3. **The interceptor capability registry** ([kernel-interceptor.service.ts#L56-L69](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L56-L69)) — CORS reality turned into a typed strategy system instead of if-statements.
4. **Locked, constrained, retried ordering writes** ([Flow 5](../01-codebase-cartography/05-key-flows.md#flow-5-creatingreordering-team-collections-persistence-flow)) — the most production-hardened code in the repo.
5. **Fail-closed authorization conventions** — guards that throw `BUG_*` on misconfiguration ([gql-team-member.guard.ts#L26-L35](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L26-L35)) rather than silently passing.
6. **The mock-server origin-split security model** ([Flow 7](../01-codebase-cartography/05-key-flows.md#flow-7-serving-a-mock-request-public-apisecurity-boundary)) — layered, commented, deliberate.

## Risks and tradeoffs (prioritized — what I'd address owning this)

### P1 — Correctness under concurrency
The single-owner invariant is enforced by count-then-write without transaction ([team.service.ts#L205-L221](../../../packages/hoppscotch-backend/src/team/team.service.ts#L205-L221), [L152-L180](../../../packages/hoppscotch-backend/src/team/team.service.ts#L152-L180)). Hypothesis: two concurrent owner-departures can zero out ownership. Fix: wrap in a serializable transaction or `SELECT ... FOR UPDATE` on the membership rows — the repo already has the locking idiom one module over. Test strategy: an integration test with two racing calls (needs real Postgres — see [testing strategy](../05-quality-engineering/01-testing-strategy.md) on the mock-only gap). Migration path: none needed; pure code change, small blast radius.

### P2 — Horizontal scaling cliff (known, unwired)
In-memory pubsub ([pubsub.service.ts#L14-L18](../../../packages/hoppscotch-backend/src/pubsub/pubsub.service.ts#L14-L18)). Single-replica constraint on the whole backend. Fix: Redis-backed PubSub behind the same injectable; the service abstraction makes this a contained change. Also audit `@Cron` jobs for multi-instance duplication at the same time. Test strategy: two-process integration harness, or at minimum deployment docs stating the constraint.

### P3 — Observability floor
`console.log/error/debug` only; no metrics, no tracing, no structured logger visible in the paths read. You cannot answer "how often do reorder retries fire?" in production. Fix: pino/winston + a counter on retries, sandbox failures, auth failures. Cheap, high-leverage. (Hypothesis: some logging infra may exist unexamined — verify before PRing.)

### P4 — Duplicated user/team hierarchies
Parallel modules and tables for user vs team collections mean every feature lands twice or drifts ([data model](02-data-model-and-persistence.md#entity-map)). This is a *chosen* tradeoff (different tenancy and guards) — I would not unify tables; I *would* extract shared tree/ordering logic into one internal library to stop logic drift. Kata: [06-.../04-refactor-and-design-katas.md](../06-contribution-practice/04-refactor-and-design-katas.md).

### P5 — Frontend state duality
`newstore` dispatchers + dioc services coexist with no visible migration plan ([framework models](../02-stack-and-language-mastery/02-framework-mental-models.md#rxjs-and-the-two-state-generations)). Risk is velocity, not correctness: every sync feature bridges both. I'd write the ADR naming dioc as the target, migrate opportunistically per-feature, and forbid new stores.

### P6 — Auth-flow ordering nits
Provider-account write before expiry check ([auth.service.ts#L281-L304](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L281-L304)); misleading param name in rotation ([L335-L347](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L335-L347)); mutating GET on `verify/admin` ([L371-L387](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L371-L387)). Individually small; collectively they say "audit this module with fresh eyes." All are ticketed in [06-.../01-good-first-tickets.md](../06-contribution-practice/01-good-first-tickets.md).

### Hypotheses parked (need evidence before acting)
UserHistory unbounded growth + no userUid index; sandbox script timeout absence (web path); subscription auth at subscribe-time only; uncapped mock `delayInMs`. All in the [risk register](../09-reference/risk-register.md) with suggested probes.

## What I'd change owning this for 3 months

Month 1: P1 fix + integration-test harness with real Postgres (unlocks testing every other concurrency claim). P3 structured logging.
Month 2: P2 Redis pubsub + deployment docs; sandbox timeouts (web parity with a node story).
Month 3: P4 shared tree library extraction behind tests; P5 ADR + first store migration; auth module audit sweep (P6 batch).
Each lands as an independently shippable, revertable PR train — no big-bang.

## Interview usage

"Critique a codebase you know well" is a real senior-loop question. The shape of a strong answer, demonstrated above: strengths first (proves you see intent), risks *prioritized by blast radius* (proves judgment), fixes with migration paths and test strategies (proves you ship), hypotheses labeled (proves honesty). Rehearse P1 and P2 as your two deep-dives — they have the best evidence trails.
