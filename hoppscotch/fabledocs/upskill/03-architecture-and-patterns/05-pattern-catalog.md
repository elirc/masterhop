# Pattern Catalog

18 patterns this repo actually uses. The goal is **recognition** — being able to spot the shape in any codebase and name what it buys and what it costs. Each card ends with the interview question it answers.

---

## Pattern 1: Versioned entity with explicit migrations (verzod)

Problem it solves: persisted user data (requests saved years ago in localStorage/DB/exports) must keep loading as the schema evolves.
General shape: every schema version is a module with a zod schema + an `up()` migration; a discriminator field (`v`) selects the version; loading = parse at detected version, migrate step-by-step to latest.
Real example: [hoppscotch-data/src/rest/index.ts#L80-L113](../../../packages/hoppscotch-data/src/rest/index.ts#L80-L113) — 18 versions (0→17), `getVersion` handles the v0 case that predates the `v` field (L102-L112).
Second example: [hoppscotch-data/src/environment](../../../packages/hoppscotch-data/src/environment) uses the same machinery.
Why this implementation works: versions are append-only modules; old code is never edited, so migrations stay auditable. `RESTReqSchemaVersion = "17"` ([L145](../../../packages/hoppscotch-data/src/rest/index.ts#L145)) pins the write version.
Failure modes: a migration that loses data silently; forgetting the version bump when changing the latest schema; O(versions) load cost on hot paths.
Use it when: data outlives code (exports, local storage, embedded JSON columns). Avoid when: the DB owns the schema and SQL migrations suffice.
Interview angle: "How do you evolve a data format without breaking old clients?" — this is a complete answer with evidence.
Drill: open [rest/v/17](../../../packages/hoppscotch-data/src/rest/v) and find what v17 added over v16 by diffing the two modules.

## Pattern 2: Errors as values — fp-ts `Either`/`Option`

Problem it solves: exceptions hide failure paths from the type system; services need callers to *handle* failure.
General shape: services return `E.Left<errorCode> | E.Right<value>`; controllers/resolvers translate Left → HTTP/GraphQL error at the boundary.
Real example: [auth.service.ts#L257-L326](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L257-L326) returns typed Left at six distinct failure points; the controller converts via `throwHTTPErr` ([auth.controller.ts#L77-L81](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L77-L81)).
Second example: frontend request pipeline ([RequestRunner.ts#L526-L531](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L526-L531)).
Why this implementation works: error codes are constants in [src/errors.ts](../../../packages/hoppscotch-backend/src/errors.ts), so the full failure vocabulary is greppable.
Failure modes: forgetting to check `isLeft` (type system catches most); mixing thrown exceptions and Eithers in one layer (see the try/catch around transactions — the seam shows).
Use it when: failures are expected outcomes (not found, invalid, forbidden). Avoid when: truly exceptional conditions — let those throw.
Interview angle: "Exceptions vs result types?" — argue costs both ways with these anchors.
Drill: count the distinct `E.left` returns in `verifyMagicLinkTokens` and name the HTTP status each maps to.

## Pattern 3: Declarative authorization — guard + metadata decorator

Problem it solves: per-resolver role checks written inline get forgotten and drift.
General shape: `@RequiresTeamRole(...)` writes metadata; a guard reads it via `Reflector`, resolves the caller's membership, and fails closed.
Real example: [gql-team-member.guard.ts#L21-L43](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L21-L43) with usage at [team.resolver.ts#L215-L219](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L215-L219).
Second example: REST variant [rest-team-member.guard.ts](../../../packages/hoppscotch-backend/src/team/guards/rest-team-member.guard.ts).
Why this implementation works: missing metadata throws `BUG_TEAM_NO_REQUIRE_TEAM_ROLE` (L26) — misconfiguration is loud, not silently permissive.
Failure modes: the guard reads `teamID` from GraphQL args by literal name (L34); an arg renamed to `id` breaks authorization (loudly). Resources addressed by non-team IDs need a different guard (see collection guards in [team-collection/guards](../../../packages/hoppscotch-backend/src/team-collection/guards)).
Use it when: role checks are uniform per-endpoint. Avoid when: authorization depends on row-level data (do it in the service).
Interview angle: "Where do you put authorization in a NestJS/Express app?"
Drill: find the guard used by mutations that receive a `collectionID` instead of a `teamID`, and trace how it resolves the team.

## Pattern 4: Invariant re-check in the service (defense in depth)

Problem it solves: guards authorize the *call*; only the service can protect *state* invariants.
General shape: even after RBAC passes, the mutation re-validates business rules against current data.
Real example: single-owner rule in [team.service.ts#L205-L221](../../../packages/hoppscotch-backend/src/team/team.service.ts#L205-L221) — an OWNER may call `leaveTeam`, but not if they're the last owner.
Second example: parent-ownership check in [team-collection.service.ts#L462-L465](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L462-L465).
Failure modes: the check here is count-then-write without a transaction — **investigate** TOCTOU (two owners leaving simultaneously could both pass the `ownerCount === 1` check... actually both see count=2 and both leave, leaving zero owners). See [risk register](../09-reference/risk-register.md#r1).
Use it when: any invariant spans rows. Avoid: duplicating *authentication* in services — that's the guard's job.
Interview angle: "What's the difference between authorization and validation?" — this pattern is the crisp answer.
Drill: write the SQL/transaction shape that would make the single-owner check race-free.

## Pattern 5: Pessimistic row locking for ordering invariants

Problem it solves: `orderIndex = max(siblings) + 1` computed concurrently produces duplicates.
General shape: open transaction → lock the sibling set (`SELECT ... FOR UPDATE` behind `lockTeamCollectionByTeamAndParent`) → read max → write.
Real example: [team-collection.service.ts#L476-L505](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L476-L505); constraint backstop [`@@unique([teamID, parentID, orderIndex])`](../../../packages/hoppscotch-backend/prisma/schema.prisma#L56).
Second example: same lock in the delete/reindex path ([L563-L589](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L563-L589)).
Why this implementation works: lock ordering is consistent (always by team+parent), which is what prevents deadlocks between these paths.
Failure modes: forgetting the lock in one write path reintroduces the race; long-held locks under slow queries throttle throughput.
Use it when: contention is low and correctness must be absolute. Avoid when: hot rows — prefer optimistic retries or fractional order keys.
Interview angle: "Pessimistic vs optimistic concurrency — when each?"
Drill: enumerate every method in this service that takes the lock (grep `lockTeamCollectionByTeamAndParent`) and check each mutates sibling order.

## Pattern 6: Bounded retry with backoff on *specific* errors

Problem it solves: deadlocks and unique-constraint collisions are transient; other errors are bugs.
General shape: retry loop with max attempts, linear backoff, and an allow-list of retryable error codes.
Real example: [team-collection.service.ts#L560-L612](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L560-L612) — retries only `UNIQUE_CONSTRAINT_VIOLATION`, `TRANSACTION_DEADLOCK`, `TRANSACTION_TIMEOUT`; backoff `retryCount * 100`ms.
Second example: no second instance found in files read — grep `MAX_RETRIES` to confirm.
Failure modes: retrying non-idempotent work (this path is safe: the delete tolerates already-deleted); retrying every error masks real bugs — the allow-list is the point.
Interview angle: "How do you handle deadlocks?" — detection + bounded retry + jitter (this repo uses linear, no jitter — a fair critique to offer).
Drill: explain why the code `break`s on success rather than returning inside the loop, and what the post-loop `E.right(true)` returns if retries exhaust (trick: it can't — trace the control flow carefully).

## Pattern 7: Idempotent mutation under concurrency

Problem it solves: two clients delete the same thing; the second should not error out the user.
General shape: catch the "not found" error class and treat as success.
Real example: [team-collection.service.ts#L572-L580](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L572-L580) — P2025 during delete → `return` (success).
Why this implementation works: **idempotency** — the post-state is identical whether you deleted it or someone else did.
Failure modes: swallowing not-found on *update* paths where the caller needs to know; conflating "never existed" with "already deleted" when auditing matters.
Interview angle: "What does idempotency mean and where have you seen it?"
Drill: is `renameCollection` ([L527-L546](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L527-L546)) idempotent? Retryable? Answer both, they differ.

## Pattern 8: Typed pub/sub topics

Problem it solves: string-topic pubsub lets publishers and subscribers drift in payload shape.
General shape: a `TopicDef` type maps topic-name patterns to payload types; `publish<T extends keyof TopicDef>` enforces it at compile time.
Real example: [pubsub.service.ts#L24-L26](../../../packages/hoppscotch-backend/src/pubsub/pubsub.service.ts#L24-L26) with [topicsDefs.ts](../../../packages/hoppscotch-backend/src/pubsub/topicsDefs.ts).
Second example: consumers at [team.resolver.ts#L325](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L325).
Failure modes: `asyncIterator` (subscribe side, L20-L22) is *not* typed against TopicDef — half the contract is unchecked. Good ticket material.
Interview angle: "How do you keep event producers and consumers in sync?"
Drill: write the type signature that would type the subscribe side too.

## Pattern 9: Publish-after-commit event ordering

Problem it solves: emitting events for state that later rolls back creates ghost updates in live UIs.
General shape: `await transaction; publish(event)` — never publish inside the transaction.
Real example: [team-collection.service.ts#L505-L514](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L505-L514) — publish sits after the `$transaction` block.
Second example: [team.service.ts#L237](../../../packages/hoppscotch-backend/src/team/team.service.ts#L237).
Failure modes: crash *between* commit and publish loses the event (at-most-once). The industrial fix is a transactional outbox — this repo accepts the gap because subscribers can refetch.
Interview angle: "Exactly-once event publishing?" — explain outbox vs this repo's accepted at-most-once.
Drill: find one publish and describe what a subscriber sees if the process dies 1ms before it.

## Pattern 10: Boundary mapping (`cast()` DB model → API model)

Problem it solves: leaking DB rows (with `parentID`, internal fields) straight into the public GraphQL contract couples storage to API.
General shape: a private `cast(dbRow): ApiModel` at the service edge.
Real example: [team-collection.service.ts#L279](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L279) (`cast`), used before every publish/return (e.g., [L513-L516](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L513-L516)).
Second example: `TeamMember` mapping in [team.service.ts#L194-L198](../../../packages/hoppscotch-backend/src/team/team.service.ts#L194-L198).
Failure modes: mapping in some paths but not others; the map silently dropping new columns you meant to expose.
Interview angle: "What is an anti-corruption layer / DTO for?"
Drill: list the fields `cast` drops or renames and say why each is hidden.

## Pattern 11: Strategy registry — pluggable interceptors

Problem it solves: one "send request" action must work over browser fetch, a browser extension, a local agent, or native Tauri code, chosen at runtime.
General shape: a registry service holds implementations of one interface (`KernelInterceptor`); a reactive `current` selection; callers call `service.execute(req)` and never know which transport ran.
Real example: interface [kernel-interceptor.service.ts#L56-L69](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L56-L69); registry/selection [L129-L153](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L129-L153); auto-fallback when the current one becomes unselectable [L97-L127](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L97-L127).
Second example: platform injection (Pattern 12) is the same idea at package scale.
Failure modes: capability drift — interceptors declare `RelayCapabilities` (L67) precisely because not every transport supports every feature (e.g., custom certs).
Interview angle: "Explain the strategy pattern with a real example" — this beats the textbook duck example.
Drill: `execute` throws if nothing is selected ([L161-L172](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L161-L172)). Argue throw vs Either here — why is throwing defensible for a programmer-error state?

## Pattern 12: Platform injection (dependency inversion at package scale)

Problem it solves: `hoppscotch-common` must run in web, desktop, and self-host builds with different auth/sync/storage backends.
General shape: common defines platform interfaces; each shell ([selfhost-web/src/platform](../../../packages/hoppscotch-selfhost-web/src/platform)) supplies implementations (`auth`, `collections`, `environments`, `history`, ...) at boot.
Real example: [selfhost-web/src/platform/collections/web/gqlCollections.sync.ts#L39-L65](../../../packages/hoppscotch-selfhost-web/src/platform/collections/web/gqlCollections.sync.ts#L39-L65) — the web shell's GraphQL-backed collection sync; a sibling `desktop/` dir provides the desktop variant.
Why this implementation works: common never imports a shell; the dependency arrow points inward only.
Failure modes: interface bloat (platform contracts grow until every shell must implement everything); silent behavioral drift between shells.
Interview angle: "How do you share 90% of an app across web and desktop?"
Drill: list the platform sub-modules in `selfhost-web/src/platform` and match each to what the logged-out app does instead.

## Pattern 13: Response-as-stream with discriminated unions

Problem it solves: a request has lifecycle states (loading → success/fail/cancelled), and consumers need all of them, typed.
General shape: `BehaviorSubject<Union>` seeded with `{type:"loading"}`; emit terminal state; `complete()`.
Real example: [network.ts#L15-L80](../../../packages/hoppscotch-common/src/helpers/network.ts#L15-L80); consumed with a type-narrowing `filter` at [RequestRunner.ts#L587-L590](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L587-L590).
Failure modes: forgetting `complete()` leaks subscriptions; `BehaviorSubject` replays the last value to late subscribers — feature here, bug elsewhere.
Interview angle: "Why RxJS over promises?" — lifecycle states + cancellation + replay semantics; also be ready to argue the reverse.
Drill: rewrite this function's signature promise-only and enumerate what the UI loses (spinner state, cancel).

## Pattern 14: Sandbox via worker + capability injection

Problem it solves: user scripts must not touch the app's DOM, cookies, or credentials.
General shape: run in a Worker (web) or isolated runtime (node); inject explicit capabilities (`pw.*` APIs, captured console) via `faraday-cage` modules; results come back as data.
Real example: [js-sandbox/src/web/test-runner/index.ts#L21-L41](../../../packages/hoppscotch-js-sandbox/src/web/test-runner/index.ts#L21-L41).
Second example: node runners ([src/node/test-runner](../../../packages/hoppscotch-js-sandbox/src/node/test-runner)) with a `--legacy-sandbox` escape hatch in the CLI.
Failure modes: capability leaks through injected objects (prototype chains!); no timeout on runaway scripts (**investigate**, see Flow 2).
Interview angle: "How would you build a plugin system safely?"
Drill: find where console output is captured and how it travels back to the UI console.

## Pattern 15: Sync mapper — local identity vs remote identity

Problem it solves: offline-first data (collections in localStorage, addressed by path/index) must reconcile with server rows (addressed by IDs).
General shape: a mapper table `localIdentifier ↔ backendID` maintained during sync; local mutations dispatch GraphQL calls, responses update the mapper.
Real example: [gqlCollections.sync.ts#L62-L65](../../../packages/hoppscotch-selfhost-web/src/platform/collections/web/gqlCollections.sync.ts#L62-L65) (`createMapper<string, string>()`), transform at [L39-L59](../../../packages/hoppscotch-selfhost-web/src/platform/collections/web/gqlCollections.sync.ts#L39-L59).
Failure modes: mapper desync (rename local path while a create is in flight); this whole area is where "my collection duplicated" bugs live — note `removeDuplicateGraphqlCollectionOrFolder` imported at [L1-L5](../../../packages/hoppscotch-selfhost-web/src/platform/collections/web/gqlCollections.sync.ts#L1-L5): the defensive function's existence is evidence.
Interview angle: "How do you sync offline-first state to a server?"
Drill: why does the local side key requests by `collectionPath/requestIndex` instead of an ID? What breaks when items reorder?

## Pattern 16: Security header blocklist + MIME downgrade

Problem it solves: users control mock responses; some headers/content-types weaponize the platform's origin.
General shape: strip a blocklist of headers; downgrade script-capable MIME types on same-origin routes; always add `nosniff`/CSP.
Real example: [mock-server.controller.ts#L18-L37](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L18-L37) + [L142-L182](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L142-L182).
Failure modes: blocklists are fail-open — a *new* dangerous header (or a creative MIME like `text/xsl`… which they did catch) walks through. An allowlist is fail-closed but breaks legitimate mocks. This repo chose usability and compensated with CSP — a defensible, articulable tradeoff.
Interview angle: "Blocklist vs allowlist?" — never answer abstractly again; cite this file.
Drill: propose one header not on the list that you'd consider adding, and the attack it enables (e.g., `Location` + 3xx for open-redirect chains — **investigate** whether status codes are constrained).

## Pattern 17: Refresh-token rotation with hashed storage

Problem it solves: long-lived refresh tokens are theft targets; storage-side leaks must not equal session theft.
General shape: refresh token is a JWT; only its argon2 hash is stored ([auth.service.ts#L114-L119](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L114-L119)); every refresh verifies-then-rotates ([L335-L363](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L335-L363)).
Failure modes: single stored hash means one active session per user — signing in elsewhere invalidates the old refresh token (**investigate**: confirm this is intended single-session behavior); no reuse-detection (a rotated-out token being replayed isn't flagged as theft, standard in stricter implementations).
Interview angle: "Why hash refresh tokens? Why rotate them?"
Drill: diagram the token lifecycle across signin → refresh → refresh, marking what's in cookie vs DB at each step.

## Pattern 18: Compound unique constraints as executable invariants

Problem it solves: invariants enforced only in app code die the day someone writes a script against the DB.
General shape: encode the invariant in the schema: [`@@unique([teamID, userUid])`](../../../packages/hoppscotch-backend/prisma/schema.prisma#L27) (one membership per user per team), [`@@unique([teamID, inviteeEmail])`](../../../packages/hoppscotch-backend/prisma/schema.prisma#L38) (no duplicate invites), [`@@unique([teamID, parentID, orderIndex])`](../../../packages/hoppscotch-backend/prisma/schema.prisma#L56) (gapless ordering), [`@@unique([deviceIdentifier, token])`](../../../packages/hoppscotch-backend/prisma/schema.prisma#L141) (device-bound magic links), [`@@unique([slug, version])`](../../../packages/hoppscotch-backend/prisma/schema.prisma#L317) (published docs versioning).
Failure modes: `NULL` in compound uniques — Postgres treats NULLs as distinct, so `[teamID, parentID(NULL), orderIndex]` does **not** dedupe root collections the way you'd assume. **Investigate**: this is exactly why the app-level lock exists. This nuance is senior-signal gold.
Interview angle: "Where do you enforce uniqueness — app or DB?" Answer: both, and know the NULL caveat.
Drill: list every `@@unique` in [schema.prisma](../../../packages/hoppscotch-backend/prisma/schema.prisma) and state each invariant in one English sentence.
