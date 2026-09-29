# Performance Thinking

Rule zero: **measure first**. Every hotspot below is a *hypothesis with evidence pointers*, not a confirmed problem. The skill being trained: knowing where each class of cost lives in this architecture and which probe confirms it.

## Performance domains in this system

| Domain | Where it lives here | First probe |
| --- | --- | --- |
| Render | Vue components (tabs, huge response bodies, collection trees) | Vue devtools profiler |
| Network (app) | GraphQL ops + sync chatter | devtools network tab |
| Server | resolvers, guards (per-request membership query), hashing | request logs + timing middleware (add one) |
| DB | unindexed scans, N+1 in tree walks | `DEBUG=prisma:query`, `EXPLAIN ANALYZE` |
| Bundle | selfhost-web build needs 8GB heap ([package.json#L10](../../../packages/hoppscotch-selfhost-web/package.json#L10)) | `vite build` + analyzer plugin |
| Memory | worker-per-script-run, response bodies held in tabs | heap snapshots |

## Hypothesis inventory (each with its confirming probe)

1. **Guard query per request.** Every team-scoped call runs `getTeamMember` ([gql-team-member.guard.ts#L37](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L37)) — indexed unique lookup, so probably fine; confirm by counting queries per GraphQL op under `DEBUG=prisma:query`. Caching it is a security tradeoff, not a free win ([Kata 5](../04-code-reading-gym/04-review-katas.md#kata-5-cache-team-membership-in-the-guard)).
2. **Tree operations as sequential awaits.** Parent-tree fetches are recursive per level ([fetchCollectionParentTree, team-collection.service.ts#L1286](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L1286)) — classic N+1-by-depth. Probe: query log while opening a deeply nested collection. Fix shape if confirmed: recursive CTE (raw SQL) — a **measured** PR, not a drive-by.
3. **UserHistory scans.** No index on `userUid` visible ([schema.prisma#L152-L161](../../../packages/hoppscotch-backend/prisma/schema.prisma#L152-L161)); history grows per request executed. Probe: `EXPLAIN` the history list query at 100k rows. Fix: index + retention.
4. **Giant response bodies in tab state.** Responses live on tab documents ([RequestRunner.ts#L609](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L609)); a 50MB JSON response is held, rendered by a lens, possibly persisted. Probe: heap snapshot after several large responses; UI frame times in the JSON lens. Mitigations to know: size caps on render, virtualization, detached storage.
5. **Sync chatter on bulk import.** Import used to fan out one mutation per node; the sync layer comments say it was optimized to a bulk call ([gqlCollections.sync.ts#L66-L68](../../../packages/hoppscotch-selfhost-web/src/platform/collections/web/gqlCollections.sync.ts#L66-L68) — "Optimized implementation using importUserCollectionsFromJSON"). Read that as: this *was* a real incident. Probe on regression: network tab during a 500-request import.
6. **Serial awaits that could parallelize** — but see [runtime model](../02-stack-and-language-mastery/01-language-runtime-model.md#the-event-loop-using-this-repos-code) for why dependency analysis precedes `Promise.all`.
7. **Bundle size.** The 8GB build heap and a Postman-class feature set imply a heavy main chunk. Probe: bundle analyzer; look for monaco/codemirror duplication across lazy chunks.

## How to find each classic problem here

- **N+1**: `DEBUG=prisma:query` + one user action; if query count scales with item count, you have one. Highest-suspicion areas: collection tree walks, search parent-tree assembly ([searchByTitle helpers](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L1274-L1407)).
- **Expensive renders**: Vue devtools "highlight updates"; collection tree + tab bar are the usual suspects; check `computed` granularity and `v-memo` absence.
- **Missing indexes**: cross-reference every `where` clause in hot services against schema indexes ([data model](../03-architecture-and-patterns/02-data-model-and-persistence.md#indexes-and-access-paths)).
- **Unbounded queries**: grep `findMany` without `take` in list endpoints; check pagination on team queries (cursor+take pattern exists — [team.resolver.ts#L160-L175](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L160-L175) — verify all lists follow it).

## Interview angle

"How do you find a performance problem?" — answer as a decision tree, not a tool list: *reproduce with numbers → attribute to a domain (render/network/server/DB/bundle) → probe that domain's native tool → fix → re-measure → guard with a budget.* Then give hypothesis 2 or 5 above as your worked example — hypothesis 5 is especially strong because the codebase itself records the fix.
