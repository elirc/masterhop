# Refactor and Design Katas

Paper/branch exercises — the deliverable is the design artifact, graded by you against the criteria. Timebox each; over-polishing is its own failure mode.

## K1: Split `RequestRunner.ts` (boundary kata) — 2h, design only
Produce: a 3-module decomposition (script execution / env resolution / response application) with exact function signatures and which module owns tab-state writes.
Self-grade: Strong = no module imports both Vue reactivity and the sandbox; the tab-write surface shrank to one function; you stated what you're *not* extracting and why.

## K2: Outbox conversion (eventing kata) — 3h, design + one fake migration
Convert `createCollection`'s publish ([team-collection.service.ts#L511-L514](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L511-L514)) to a transactional outbox on paper: event table DDL, relay loop, delivery semantics statement, subscriber impact (none — same topics).
Self-grade: Strong = you addressed relay crash/redelivery (at-least-once now — subscribers must tolerate dupes; name which current subscriber wouldn't).

## K3: De-fp-ts one service (readability kata) — 2h, branch
Rewrite `leaveTeam` with plain discriminated-union results (`{ok:true,...}|{ok:false,error}`) preserving behavior and tests. Then argue *against* merging your own PR (consistency across 30 services beats local taste).
Self-grade: Strong = your argument against yourself cites the real cost of a two-idiom codebase ([framework models](../02-stack-and-language-mastery/02-framework-mental-models.md#rxjs-and-the-two-state-generations) — the repo already pays this tax in state management).

## K4: Type-safety kata — 2–3h, branch
Eliminate the `any` return from `transformCollectionForBackend` ([gqlCollections.sync.ts#L39](../../../packages/hoppscotch-selfhost-web/src/platform/collections/web/gqlCollections.sync.ts#L39)) by naming the wire type (derive from the generated GraphQL input types if possible).
Self-grade: Strong = zero `as` casts survive; a caller bug becomes a compile error you can demonstrate.

## K5: Migration design kata — 2h, paper
Design "orderIndex → fractional order key" for TeamCollection as a zero-downtime migration: dual-write phase, backfill, read-switch, constraint swap, revert points at each phase.
Self-grade: Strong = every phase is independently deployable and revertable; you identified that the [lock machinery](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L476-L505) becomes deletable only at the end — and whether the `@@unique` survives (it changes meaning).

## K6: N+1 hunt (evidence kata) — 2h, running system required
With `DEBUG=prisma:query`, open a 4-level-deep collection tree and count queries; if super-linear, write (don't submit) the recursive-CTE fix with before/after counts.
Self-grade: Strong = numbers in hand, and a stated threshold for when the fix is worth its raw-SQL maintenance cost.

## K7: RFC kata — 3h, paper
Write the RFC for [P1 Redis pubsub](03-senior-build-projects.md#p1-redis-backed-pubsub-multi-instance-real-time--2-weeks) using the [RFC template](../07-career-and-collaboration/02-writing-prs-and-rfcs.md#the-rfc-template): context, two alternatives (Redis pub/sub vs Postgres LISTEN/NOTIFY — take the second seriously; the DB is already there), decision, risks, rollout.
Self-grade: Strong = the alternative analysis could convince a maintainer either way; delivery-guarantee implications stated for both.

## K8: Review a flawed PR under time pressure — 30 min hard stop
Take [Kata 6 (mock-server MIME downgrade removal)](../04-code-reading-gym/04-review-katas.md#kata-6-support-html-preview-for-mock-responses) cold, write the full review in 30 minutes including the alternative you'd propose.
Self-grade: Strong = you found the security regression in the first 10 minutes *and* your proposed alternative (subdomain URLs / sandboxed preview origin) is concrete enough to unblock the author — review that only blocks is half a review.
