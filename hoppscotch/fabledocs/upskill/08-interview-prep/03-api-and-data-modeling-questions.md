# API and Data-Modeling Question Cards

15 cards. Round: **API/data**. All anchored to this repo's actual API and schema.

## Q1: GraphQL vs REST — when would you use both in one system?
Anchor: this backend does — GraphQL for team/sync data ([team.resolver.ts](../../../packages/hoppscotch-backend/src/team/team.resolver.ts)), REST for auth ([auth.controller.ts](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts)) and the public mock surface.
Mid: auth needs redirects + Set-Cookie (browser primitives, awkward in GraphQL); collaborative data wants selective queries + subscriptions; public mock endpoints must accept *any* method/path — a catch-all REST controller ([mock-server.controller.ts#L52](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L52)).
Senior: contract governance per surface — GraphQL schema is generated and type-checked to clients; REST contracts rely on discipline; the mock surface is deliberately contract-free. Three surfaces, three governance modes, one system.

## Q2: How do you paginate? Defend cursor vs offset.
Anchor: [team.resolver.ts#L160-L175](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L160-L175) — `cursor` (last-seen ID) + `take` (default 10).
Mid: offset breaks under concurrent inserts (skips/dupes) and costs O(offset); cursor is stable and indexed.
Senior: cursor requires a total order (ID here — creation-ordered cuid-ish; is that the order users want? name ordering vs stable pagination tension); also: `take` unbounded above? Check for a max — unbounded take is a DoS knob.

## Q3: Where does validation live for a mutation like `createCollection`?
Anchor: the five-layer table in [validation map](../03-architecture-and-patterns/03-validation-auth-and-permissions.md#the-validation-layers-mapped); service checks at [team-collection.service.ts#L458-L472](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L458-L472).
Mid: shape at the edge (GraphQL types/DTOs), semantics in the service (title length, parent ownership, JSON validity).
Senior: *why* semantics can't move to the edge — parent ownership needs a DB read and tenant context; the rule: validation requiring state lives with state.

## Q4: Authn vs authz — implement both for a multi-tenant API.
Anchor: `GqlAuthGuard` then `GqlTeamMemberGuard` + `@RequiresTeamRole` ([team.resolver.ts#L215-L219](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L215-L219); guard internals [gql-team-member.guard.ts#L21-L43](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L21-L43)).
Mid: identity (JWT cookie → user) vs permission (membership row + role set); order matters.
Senior: fail-closed misconfiguration handling (`BUG_*` throws), the child-resource IDOR seam, and business invariants re-checked in services ([Pattern 4](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-4-invariant-re-check-in-the-service-defense-in-depth)).

## Q5: Design "leave team" correctly.
Anchor: [team.service.ts#L205-L240](../../../packages/hoppscotch-backend/src/team/team.service.ts#L205-L240).
Mid: the invariant (≥1 OWNER always), the check, the event publish.
Senior: the TOCTOU honestly disclosed + the transactional fix ([critique P1](../03-architecture-and-patterns/06-architecture-critique.md#p1--correctness-under-concurrency)). This card is a gift: a real subtle bug you can explain end to end.

## Q6: Where do cross-cutting concerns (rate limiting, last-login stamping) go?
Anchor: `ThrottlerBehindProxyGuard` controller-wide ([auth.controller.ts#L34](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L34)), `@SkipThrottle` exceptions (L114), `UserLastLoginInterceptor` (L116).
Mid: guards/middleware/interceptors, not handler bodies; opt-out explicit and visible.
Senior: "behind proxy" trust chain (forwarded IPs) — infra config becomes part of the security model; and the *audit* habit: every `@SkipThrottle` needs a reason you can recite.

## Q7: Model a folder tree with ordered children in SQL.
Anchor: [TeamCollection](../../../packages/hoppscotch-backend/prisma/schema.prisma#L42-L57) — adjacency list + `orderIndex` + compound unique.
Mid: adjacency vs materialized path vs closure table; integer index + unique constraint; cascade for subtree delete.
Senior: gapless-integer maintenance cost (reindex on delete/move → the whole [lock/retry machinery](../01-codebase-cartography/05-key-flows.md#flow-5-creatingreordering-team-collections-persistence-flow)) vs fractional keys (no reindex, key growth); *and* the Postgres NULL-in-unique caveat for root nodes.

## Q8: When do you store JSON columns instead of normalized tables?
Anchor: `request Json` ([schema.prisma#L64](../../../packages/hoppscotch-backend/prisma/schema.prisma#L64)) + the client-side versioning that makes it safe ([Pattern 1](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-1-versioned-entity-with-explicit-migrations-verzod)).
Mid: JSON when the DB never queries inside and the shape evolves fast; normalized when you filter/join/constrain on fields.
Senior: the ownership statement — someone must own the blob's schema (here: the `data` package with 18 versions); JSON without an owner is a swamp. Cite `fixBrokenRequestVersion.ts` as the scar of getting it briefly wrong.

## Q9: Design schema migrations for zero downtime.
Anchor: [prisma/migrations](../../../packages/hoppscotch-backend/prisma/migrations); recipe in [data model](../03-architecture-and-patterns/02-data-model-and-persistence.md#how-to-safely-change-this-schema).
Mid: additive → tolerate both → backfill → tighten; forward-only rollback.
Senior: the two-migration-systems insight — SQL migrations (deploy-time, DBA-visible) vs lazy data-package migrations (read-time, client-side) — and which failure modes each has (locked table vs stragglers on old versions forever).

## Q10: Transactions — when and how here?
Anchor: interactive transaction + explicit lock ([team-collection.service.ts#L476-L505](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L476-L505)).
Mid: multi-row invariants need atomicity; single-row writes don't; keep transactions short.
Senior: what does NOT belong inside — the pubsub publish ([Pattern 9](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-9-publish-after-commit-event-ordering)), external calls, hashing; and retry design for serialization failures.

## Q11: Idempotency for API design.
Anchor: concurrent-delete tolerance ([team-collection.service.ts#L572-L580](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L572-L580)).
Mid: definition + which verbs are naturally idempotent; why DELETE returning success-if-gone is right.
Senior: idempotency *keys* for creates (this repo doesn't need them — UI-driven; a public API would), and retries-require-idempotency as the design law.

## Q12: How do subscriptions/real-time change API design?
Anchor: [team.resolver.ts#L305-L326](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L305-L326); [pubsub.service.ts](../../../packages/hoppscotch-backend/src/pubsub/pubsub.service.ts).
Mid: events as first-class contract (topic + payload types); authz at subscribe.
Senior: delivery guarantees (at-most-once here), the horizontal-scaling cliff, subscribe-time-vs-per-event authorization staleness, and payload-vs-refetch for secret-bearing entities ([M2](../06-contribution-practice/02-mid-level-feature-tickets.md#m2-team-environment-change-events)).

## Q13: Design token auth for a CLI against your API.
Anchor: [PersonalAccessToken](../../../packages/hoppscotch-backend/prisma/schema.prisma#L225-L234); raw storage forced by lookup-by-token; contrast hashed refresh tokens ([security checklist](../05-quality-engineering/05-security-checklist.md)).
Mid: PAT with label + optional expiry, revocable, sent as header.
Senior: the storage asymmetry and its fix-if-needed (store HMAC of token, look up by HMAC) — this exact asymmetry question separates senior candidates.

## Q14: Where do error contracts live in an API?
Anchor: [errors.ts](../../../packages/hoppscotch-backend/src/errors.ts) constants; Either→HTTP/GraphQL translation at controllers/resolvers; [Kata 4](../04-code-reading-gym/04-review-katas.md#kata-4-nicer-error-for-expired-magic-links)'s lesson.
Mid: stable machine-readable codes; human copy client-side.
Senior: error taxonomy as versioned contract — renaming a code is a breaking change; also the info-leak audit (member vs non-member error differences).

## Q15: Rate limit a public, user-programmable endpoint (the mock server).
Anchor: [mock-server.controller.ts#L47](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L47) + `delayInMs` connection-holding ([L135-L140](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L135-L140)) + `hitCount` tracking ([schema.prisma#L257](../../../packages/hoppscotch-backend/prisma/schema.prisma#L257)).
Mid: per-IP throttling, per-mock quotas, isActive kill-switch ([schema.prisma#L256](../../../packages/hoppscotch-backend/prisma/schema.prisma#L256)).
Senior: the *resource dimensions* beyond request-rate — held connections (delay), response size, log-storage growth (`MockServerLog` per hit) — each needs its own limit; rate limiting is a vector, not a scalar.
