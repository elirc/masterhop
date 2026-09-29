# System Design From This Repo

The whiteboard exercise: **"Design a collaborative API client — Postman."** You have studied a production implementation; this file converts that into a 40-minute interview performance. Cross-reference: [architecture critique](../03-architecture-and-patterns/06-architecture-critique.md) for the risk discussion.

## How to run the 40 minutes

1. Requirements (5 min) → 2. API sketch (7 min) → 3. Data model (10 min) → 4. The hard parts: sandboxing, real-time, ordering (12 min) → 5. Scaling & tradeoffs (6 min). At every step below: what Hoppscotch actually chose (with anchor), what a junior/mid/senior answer sounds like, and a stronger-or-simpler alternative.

---

## Step 1 — Requirements

Functional: compose and send HTTP/GraphQL requests; save into nested collections; environments with variable substitution; user scripts before/after requests; teams with roles; real-time collaboration; share/mocks; CLI for CI.
Non-functional: requests must be sendable **without any backend** (local-first); user scripts must be isolated; team data must be tenant-isolated; collaboration should feel live.

- Junior: lists features.
- Mid: separates local-first from cloud features — the app works logged-out; the backend adds sync/teams. Hoppscotch's proof: the frontend has a full local persistence layer and the backend is optional ([platform injection](../03-architecture-and-patterns/05-pattern-catalog.md), Pattern 12).
- Senior: identifies the defining constraint early — *the client sends arbitrary HTTP, so the browser's CORS model is your enemy*; that single fact forces the interceptor architecture.

## Step 2 — API sketch

Hoppscotch chose **GraphQL** for the team/sync API (queries, mutations, subscriptions in one schema — resolvers like [team.resolver.ts](../../../packages/hoppscotch-backend/src/team/team.resolver.ts)) plus **REST** for auth ([auth.controller.ts](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts) — cookies and OAuth redirects are awkward in GraphQL) plus a **public catch-all REST surface** for mocks ([mock-server.controller.ts#L52](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L52)).

- Junior: one REST API for everything.
- Mid: notices auth wants REST (redirects, Set-Cookie) while collaborative data wants GraphQL subscriptions; can sketch `createTeam`, `teamMemberAdded(teamID)` ([team.resolver.ts#L305-L326](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L305-L326)).
- Senior: talks contract governance — schema is code-generated to clients (`pnpm gen-gql`, [root package.json#L13](../../../package.json#L13)), so breaking changes are caught at frontend compile time.
- Simpler alternative: REST + polling. Defensible for v1; you lose live cursors/updates and pay refetch costs.

## Step 3 — Data model

Draw this (Hoppscotch's actual shape, [schema.prisma](../../../packages/hoppscotch-backend/prisma/schema.prisma)):

```
User ─< TeamMember(role) >─ Team ─< TeamCollection (self-referencing tree, orderIndex)
                                        └─< TeamRequest (request Json, orderIndex)
Team ─< TeamEnvironment(variables Json)
User ─< PersonalAccessToken / Account(provider) / VerificationToken
MockServer ─< MockServerLog
```

Three decisions to defend:

1. **Requests are a `Json` column** ([schema.prisma#L64](../../../packages/hoppscotch-backend/prisma/schema.prisma#L64)), not normalized columns. Why: the request format evolves fast (18 schema versions! — [rest/index.ts#L80-L113](../../../packages/hoppscotch-data/src/rest/index.ts#L80-L113)) and the DB never queries inside it. The *client* owns that schema via versioned migrations. Tradeoff: no SQL-level querying/validation of request internals.
2. **Tree as adjacency list** (`parentID` self-relation, [schema.prisma#L51-L52](../../../packages/hoppscotch-backend/prisma/schema.prisma#L51-L52)) with `onDelete: Cascade` doing subtree deletion. Alternative: materialized path or closure table for cheap subtree reads — Hoppscotch instead walks recursively in app code (see `fetchCollectionParentTree`, [team-collection.service.ts#L1286](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L1286)), fine at collection-tree scale.
3. **Sibling order as integer `orderIndex`** with [`@@unique([teamID, parentID, orderIndex])`](../../../packages/hoppscotch-backend/prisma/schema.prisma#L56), maintained under row locks with retries ([Flow 5](../01-codebase-cartography/05-key-flows.md#flow-5-creatingreordering-team-collections-persistence-flow)). Alternative: fractional/lexicographic order keys — no reindex on insert, but keys grow and need rebalancing.

- Junior: gets entities and foreign keys.
- Mid: gets the unique constraints as invariants and can explain cascade deletes.
- Senior: leads with *who owns which schema* — DB owns tenancy/ordering; the versioned JSON blob belongs to the client package; that split is the design.

## Step 4 — The hard parts (where the interview is won)

### 4a. Sending requests without CORS pain

Browser JS cannot send arbitrary cross-origin requests. Hoppscotch's answer: the **interceptor strategy registry** ([kernel-interceptor.service.ts#L56-L69](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L56-L69)) — browser fetch (limited), browser extension, local agent process, or native code in the desktop app, all behind one interface with declared capabilities. Mid answer: proxy server. Senior answer: the capability-declaring registry, because "which transports support client certs / custom DNS?" is a product question, not an if-statement.

### 4b. User scripts

Run pre/post scripts in a **worker-isolated sandbox with injected capabilities** ([Flow 2](../01-codebase-cartography/05-key-flows.md#flow-2-running-a-user-test-script-sandboxasync-flow)); same contract executes in node for the CLI. Mid: "web worker." Senior: capability injection (no ambient authority), serializable results as `Either`, a runaway-script kill story, and the observation that the sandbox being its own package is what lets CLI and web share it.

### 4c. Real-time collaboration

Hoppscotch: GraphQL subscriptions over an **in-process pubsub** ([pubsub.service.ts#L14-L18](../../../packages/hoppscotch-backend/src/pubsub/pubsub.service.ts#L14-L18)), topics namespaced `team/{id}/...`, authorized at subscribe time by the same RBAC guard ([team.resolver.ts#L310-L316](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L310-L316)).
This is also the **scaling cliff**: two backend replicas = events published on instance A never reach sockets on instance B. The code's own comment plans Redis ([pubsub.service.ts#L5-L8](../../../packages/hoppscotch-backend/src/pubsub/pubsub.service.ts#L5-L8)). Offering this unprompted — "the design is correct single-node and has a named, bounded migration to multi-node" — is a senior move.

### 4d. Multi-tenancy / authorization

Declarative role metadata + membership guard ([gql-team-member.guard.ts#L21-L43](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L21-L43)), business invariants re-checked in services (single-owner rule, [team.service.ts#L205-L221](../../../packages/hoppscotch-backend/src/team/team.service.ts#L205-L221)). Name the known weakness honestly if pressed: the owner-count check is not transactional (TOCTOU) — and say how you'd fix it (serializable transaction or a `SELECT ... FOR UPDATE` on members).

## Step 5 — Scaling and evolution prompts

Prepared answers for the classic follow-ups:

1. **"Now make it 10× traffic."** Stateless API scales horizontally *except* subscriptions (Redis pub/sub migration) and the mock server's artificial delay holding connections ([mock-server.controller.ts#L135-L140](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L135-L140) — cap it, or move mocks to an edge worker). DB: `UserHistory` is append-heavy with no retention policy visible — partition or prune.
2. **"Now add real-time co-editing of a request body."** Subscriptions broadcast whole entities; co-editing needs OT/CRDT — scope it to presence + field-level locks first; full CRDT (Yjs) only if the product demands it.
3. **"Now add an audit log."** The pubsub publish points ([Pattern 9](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-9-publish-after-commit-event-ordering)) are exactly where audit events belong; note the at-most-once gap and introduce a transactional outbox if audit must be complete.
4. **"Offline-first sync conflicts?"** Point at the mapper design ([Pattern 15](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-15-sync-mapper--local-identity-vs-remote-identity)) and its honest weakness (path-based local identity); propose stable client-generated IDs (`_ref_id` already exists in the data model — [rest/index.ts#L141](../../../packages/hoppscotch-data/src/rest/index.ts#L141)) as the migration.
5. **"Self-host vs cloud from one codebase?"** Platform injection (Pattern 12) + `InfraConfig` table for runtime config ([schema.prisma#L215-L223](../../../packages/hoppscotch-backend/prisma/schema.prisma#L215-L223)) instead of env-vars-only — admins reconfigure without redeploys.

## Grading yourself

- Basic: you can draw steps 1–3 and name the interceptor idea.
- Solid: you can defend the three data-model decisions with tradeoffs and raise the pubsub scaling cliff yourself.
- Strong: you can run every step 4 subsection with anchors from memory, disclose one real weakness (TOCTOU or at-most-once events) *with its fix*, and adapt to all five variation prompts without notes.
