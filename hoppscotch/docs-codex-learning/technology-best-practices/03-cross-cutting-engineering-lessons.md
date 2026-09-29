# Cross-Cutting Engineering Lessons

## Separation Of Concerns

Concept: each layer should own a clear decision.

Repo evidence: platform selection in `packages/hoppscotch-selfhost-web/src/main.ts:42-204`; shared app creation in `packages/hoppscotch-common/src/index.ts:24-70`; backend resolver/service split in `user-history.resolver.ts:26-57` and `user-history.service.ts:60-93`.

Junior miss: seeing many files as complexity rather than ownership.

Senior notice: package boundaries enable web/desktop/self-host variants without copying UI.

Practice: explain why `hoppscotch-common` should not import `@app/platform/history/web`.

## Type Safety

Concept: types are contracts, but runtime data still needs validation.

Repo evidence: `HoppRESTRequest` in `packages/hoppscotch-data/src/rest/index.ts:80-115`; workspace union in `workspace.service.ts:16-27`; GraphQL `Either` in `GQLClient.ts:197-270`.

Junior miss: believing TypeScript protects JSON from the network.

Senior notice: `user-history.service.ts:69-77` parses JSON and needs runtime error handling.

Practice: list three compile-time contracts and three runtime boundaries.

## Validation Boundaries

Concept: validate at the edge where data enters the trusted system.

Repo evidence: Nest `ValidationPipe` in `backend/src/main.ts:76-80`; Zod schemas in `data/src/rest/index.ts:80-115`; direct JSON parsing in `user-history.service.ts:69-77`.

Junior miss: only checking forms.

Senior notice: GraphQL `String` args are not structured validation.

Practice: design `safeParseJSON` for history service and test it.

## API Design

Concept: API shape controls how safely frontend and backend evolve.

Repo evidence: API wrappers in `history/web/api.ts:35-91`; resolver args in `user-history.resolver.ts:30-48`; model fields in `user-history.model.ts:21-40`.

Junior miss: thinking the API is only the resolver.

Senior notice: generated documents, wrapper functions, resolver decorators, and model fields form one contract.

Practice: map `toggleHistoryStarStatus` from UI wrapper to service.

## State Management

Concept: state needs a home, allowed mutations, and observation strategy.

Repo evidence: `DispatchingStore.ts:34-77`; history dispatchers in `history.ts:134-192`; streams in `history.ts:267-268`.

Junior miss: mutating arrays directly from UI.

Senior notice: sync depends on store dispatch semantics, so action naming matters.

Practice: add a documentation-only `starredCount$` selector plan.

## Data Fetching

Concept: fetching should centralize auth, loading, errors, and retries.

Repo evidence: URQL auth exchange in `GQLClient.ts:71-118`; query wrapper in `GQLClient.ts:210-270`; Vue composable in `composables/graphql.ts:47-236`.

Junior miss: copy-pasting fetch logic.

Senior notice: `network-only` is conservative but potentially expensive.

Practice: identify one query that could be cache-safe and defend your reasoning.

## Database Access

Concept: services should own persistence and enforce authorization assumptions.

Repo evidence: Prisma model in `schema.prisma:152-161`; create path in `user-history.service.ts:60-93`; Prisma connection setup in `prisma.service.ts:14-47`.

Junior miss: seeing Prisma as just CRUD.

Senior notice: update/delete by ID only in `user-history.service.ts:101-162` may under-enforce ownership.

Practice: write a test plan for mismatched `uid`.

## Error Handling

Concept: expected failures should become typed control flow.

Repo evidence: `Either` in `user-history.service.ts:66-93`; resolver `throwErr` in `user-history.resolver.ts:49-56`; frontend `Either` in `GQLClient.ts:210-270`.

Junior miss: throwing everywhere.

Senior notice: direct JSON parse can bypass the intended `Either` style.

Practice: find one throw and decide whether it is expected or exceptional.

## Testing Strategy

Concept: tests should protect behavior at the smallest useful boundary.

Repo evidence: workspace service tests in `workspace.service.spec.ts:72-140`; backend user history tests in `user-history.service.spec.ts:143-218`; CLI request runner tests in `requestRunner.spec.ts:27-108`.

Junior miss: testing implementation details.

Senior notice: service tests are good, but cross-layer sync tests may be missing.

Practice: write a test title for subscription echo prevention.

## Production Readiness

Concept: production systems need security, observability, migration discipline, and setup reliability.

Repo evidence: CORS and logging in `main.ts:58-89`; token hashing in `auth.service.ts:103-151`; migrations under `packages/hoppscotch-backend/prisma/migrations`; setup scripts in root/package JSON.

Junior miss: assuming passing tests means production-ready.

Senior notice: setup friction and missing ownership checks are production concerns.

Practice: draft a one-page release checklist for a history service change.

