# Mid-Level Feature Tickets

10 cross-layer tickets. Rule: **write the design note first** (½–1 page: data model delta, API delta, UI delta, test plan, risk, rollback) and grade it against the ticket before coding. Risk/rollback sections are mandatory — that's the mid-level bar.

## M1: Archive (soft-hide) a team collection
Layers: schema → GraphQL → UI → tests. ~3–5 days.
Design constraints: nullable `archivedAt` on TeamCollection (additive migration — [safe-change recipe](../03-architecture-and-patterns/02-data-model-and-persistence.md#how-to-safely-change-this-schema)); archived items excluded from default tree queries ([getChildrenOfCollection, team-collection.service.ts#L341](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L341)) but **orderIndex handling on archive/unarchive is the hard part** — decide: keep index (gaps) or reindex (reuse the delete/reindex machinery at [L555-L616](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L555-L616)); defend the choice.
Risk: every existing query that counts/walks siblings. Rollback: feature-flag the UI; column stays (harmless).
Interview story potential: "I extended an ordering-invariant system without breaking its concurrency guarantees."

## M2: Team-environment change events
Layers: backend pubsub → GraphQL subscription → frontend live update.
Today: check whether team environments publish on update (grep `team_environment` topics in [team-environments](../../../packages/hoppscotch-backend/src/team-environments)); wire any missing mutation→topic→subscription→UI path end to end, following the collection topics as template ([Pattern 8](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-8-typed-pubsub-topics)/[9](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-9-publish-after-commit-event-ordering)).
Risk: payloads may contain secrets (env values!) — decide what subscribers receive vs refetch. Rollback: drop the subscription; publishes are inert without consumers.
Interview story potential: "I designed the payload-vs-refetch tradeoff for live updates of secret-bearing data."

## M3: Cap and validate mock-server `delayInMs`
Layers: GraphQL input validation → service → docs.
Enforce a max (e.g., 60s) at creation/update, plus a runtime clamp at [mock-server.controller.ts#L135-L140](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L135-L140) for pre-existing rows. Migration question: what about existing rows over the cap? (Clamp at runtime; don't rewrite data silently — argue it.)
Risk: users with legitimate long delays (timeout testing). Rollback: raise the constant.
Interview story potential: "I closed a resource-holding DoS vector while preserving the feature's purpose."

## M4: Request-level "resolved effective request" preview
Layers: frontend only (helpers + UI).
A panel showing the request *after* env substitution (mask secrets) before sending — surfacing [EffectiveURL.ts](../../../packages/hoppscotch-common/src/helpers/utils/EffectiveURL.ts) output. Reuse `getEffectiveRESTRequest` without running scripts (scripts mutate state — the preview must be side-effect-free; that constraint is the design work).
Risk: secret leakage in the preview; divergence between preview and actual send path. Rollback: UI flag.
Interview story potential: "I built a debugging surface reusing the production code path read-only."

## M5: Integration-test job for ordering invariants
Layers: CI + test infra + a first test.
Postgres service container in a new workflow job; run migrations; two racing `createCollection` calls; assert gapless siblings. This is [testing strategy](../05-quality-engineering/01-testing-strategy.md#the-strategic-gap-say-this-carefully-and-its-a-senior-statement)'s gap made concrete (scoped-down version of [senior project 2](03-senior-build-projects.md)).
Risk: CI time; flaky concurrency tests (design for determinism: barrier the two calls, retry-assert). Rollback: job is non-required initially.
Interview story potential: "I introduced real-database integration testing to cover what mocks can't."

## M6: Per-user active-session list + revoke
Layers: schema (multi-session refresh tokens) → REST/GraphQL → settings UI.
Today one hashed refresh token per user ([auth.service.ts#L114-L119](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L114-L119)) implies single-session ([Trace 3](../04-code-reading-gym/02-trace-tables.md#trace-3-auth-refresh-token-rotation)). Move to a `Session` table (token hash, device, createdOn, lastUsed), verify-against-any, revoke-one.
Risk: auth-path change = maximum blast radius; ship behind a flag, migrate by "new logins create sessions; legacy column honored until empty."
Interview story potential: the strongest one here — "I migrated a live auth system from single- to multi-session with zero forced logouts."

## M7: Collection export includes mock examples
Layers: backend export → data schema → CLI import parity.
`exportCollectionToJSONObject` ([team-collection.service.ts#L111](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L111)) — check whether `mockExamples` ([schema.prisma#L65](../../../packages/hoppscotch-backend/prisma/schema.prisma#L65)) round-trips through export→import; if not, add it **with a data-package version bump if the export format is versioned** (investigate first — that investigation is half the ticket).
Risk: format compatibility with older CLI versions. Rollback: additive field, old readers ignore it.
Interview story potential: "I evolved a persisted format without breaking old readers."

## M8: Structured logging for the retry/auth paths
Layers: backend infra.
Introduce pino (or Nest Logger consistently), replace console calls in [team-collection retries](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L596-L611) and auth failures, add request IDs. Scope discipline: two modules only, pattern documented for the rest.
Risk: log-volume cost; PII in logs (emails!) — write the redaction rule. Rollback: logger config.
Interview story potential: "I established an observability baseline and its PII policy."

## M9: `hopp test --env` inline overrides
Layers: CLI options → env merge → docs + tests.
Add `--env-var key=value` (repeatable) merged over `--env` file, following the existing option/parse structure ([options/test/env.ts](../../../packages/hoppscotch-cli/src/options/test/env.ts), [cli test.ts#L44-L46](../../../packages/hoppscotch-cli/src/commands/test.ts#L44-L46)). Define precedence (CLI > file > collection) and test it — precedence tables are where CLI bugs live.
Risk: secrets in shell history — note it in docs. Rollback: additive flag.
Interview story potential: "I designed configuration precedence for a CI tool."

## M10: Team-collection search respects role visibility
Layers: backend search → guard alignment → tests.
Audit `searchByTitle` ([team-collection.service.ts#L1133](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L1133)): what guard protects its resolver, and can a VIEWER see titles they should? (Likely fine — all roles can read; the *exercise* is proving it and writing the cross-tenant rejection test.) Then add the test that a non-member gets nothing ([Recipe 4](../05-quality-engineering/02-writing-tests-here.md#recipe-4-cross-tenant-rejection)).
Risk: none if findings are clean; a finding = responsible disclosure practice (report, don't post).
Interview story potential: "I security-audited a search path and locked in isolation with tests."

---

Grading your design notes (before coding): Basic — layers and files identified. Solid — migration ordering, test plan per layer, one named risk with mitigation. Strong — rollback story, blast-radius statement, and an explicit "what I am NOT doing" scope fence.
