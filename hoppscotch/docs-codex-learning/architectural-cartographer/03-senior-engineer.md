# Senior Engineer Guide

## Architectural Critique

| Area | Score | Rationale |
|---|---:|---|
| Scalability | 4 | Package boundaries and platform injection scale well. JSON history storage is flexible but can complicate querying. |
| TypeScript discipline | 4 | Strong schema-backed data package and generated GraphQL types. Some `any`/JSON string boundaries remain. |
| Separation of concerns | 4 | Shell/common/backend/data split is strong. Some UI components own substantial behavior. |
| Testability | 3 | Many unit tests exist, especially services and sandbox. Full end-to-end flows are harder to verify locally. |
| Maintainability | 4 | Versioned schemas and package boundaries help. Setup and generated-code flow are not beginner-obvious. |
| Security posture | 3 | Auth guards, token refresh, throttling, and CORS exist. Some service methods should enforce ownership in database predicates. |
| Performance | 3 | History capped locally at 50. GraphQL is network-only in several paths; duplicate subscriptions are explicitly noted. |
| Developer experience | 3 | pnpm scripts exist, but first setup requires stitching root/package/env/Docker/codegen details together. |

## Performance Audit

Finding: GraphQL queries use `network-only` in `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts:217-220` and `useGQLQuery` in `packages/hoppscotch-common/src/composables/graphql.ts:117-120`. This is simple and fresh, but repeated polling or route changes can increase backend load.

Possible corrected pattern for low-risk read-only data:

```ts
client.value?.executeQuery(request.value, {
  requestPolicy: args.cacheSafe ? "cache-and-network" : "network-only",
})
```

Finding: Local history caps entries at 50 in `packages/hoppscotch-common/src/newstore/history.ts:129-146`, which is good for UI memory. Backend `fetchUserHistory` uses `take` and `orderBy` in `packages/hoppscotch-backend/src/user-history/user-history.service.ts:28-38`, but `UserHistory` lacks an explicit index on `(userUid, reqType, executedOn)` in `schema.prisma:152-161`. Add an index if history reads become hot.

## Security Audit

Finding: `toggleHistoryStarStatus` receives `uid`, but updates by `id` only in `packages/hoppscotch-backend/src/user-history/user-history.service.ts:101-128`. `removeRequestFromHistory` does the same delete-by-id in `user-history.service.ts:140-162`. A safer shape is ownership-enforced lookup or composite predicate.

Suggested service pattern:

```ts
// Documentation-only example, not applied.
await this.prisma.userHistory.update({
  where: { id },
  data: { isStarred: !userHistory.value.isStarred },
})
// Improve by first finding with { id, userUid: uid } or defining a unique compound key.
```

Finding: `createUserHistory` calls `JSON.parse` directly in `user-history.service.ts:69-77`. GraphQL argument typing proves only that values are strings. Safer code would catch parse failures and return a typed error.

## TypeScript Discipline Review

Strong:

- Versioned request schema: `packages/hoppscotch-data/src/rest/index.ts:80-115`.
- Workspace discriminated union: `packages/hoppscotch-common/src/services/workspace.service.ts:16-27`.
- Typed GraphQL helper result using `Either`: `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts:197-270`.

Watch:

- JSON strings in GraphQL models: `packages/hoppscotch-backend/src/user-history/user-history.model.ts:21-29`.
- Event target `any` in `Request.vue:516-519`.
- Store dispatchers may mutate arrays in place for duplicate removal in `history.ts:178-190`.

## Custom Abstractions Inventory

- `PlatformDef`: capability injection for auth/sync/history/interceptors.
- `DispatchingStore`: typed dispatcher store over RxJS `BehaviorSubject`.
- `useGQLQuery`: Vue composable wrapping URQL, polling, subscriptions, and `Either`.
- `@hoppscotch/data` versioned entities: migration-aware domain schemas.
- Kernel modules: web/desktop IO, relay, store, log abstractions.
- Sync adapters: map local store dispatches to backend mutations and subscriptions.

## Testing Assessment

Good anchors:

- Workspace service tests in `packages/hoppscotch-common/src/services/__tests__/workspace.service.spec.ts:72-140`.
- Persistence migration tests in `packages/hoppscotch-common/src/services/persistence/__tests__/index.spec.ts:202-240`.
- User history service tests in `packages/hoppscotch-backend/src/user-history/user-history.service.spec.ts:143-218`.
- CLI request runner tests in `packages/hoppscotch-cli/src/__tests__/functions/request/requestRunner.spec.ts:27-108`.

Missing high-value test: malformed JSON and ownership enforcement in user history service.

Example test to add later, not added to repo:

```ts
test("createUserHistory returns a typed error for malformed request JSON", async () => {
  const result = await userHistoryService.createUserHistory(
    "user-1",
    "{not-json",
    JSON.stringify({ duration: 10, statusCode: 200 }),
    "REST"
  )

  expect(result).toEqualLeft(USER_HISTORY_INVALID_JSON)
  expect(mockPrisma.userHistory.create).not.toHaveBeenCalled()
})
```

## Bug Injection Exercise

1. Symptom: starring a history item flips back. Test: simulate own-client subscription echo after `toggleStar`. Inspect `history/web/index.ts:193-231`.
2. Symptom: another user's history item can be deleted if ID is known. Test: call service delete with mismatched `uid`. Inspect `user-history.service.ts:140-162`.
3. Symptom: login succeeds but history does not load after token refresh. Test: emit auth events and assert `loadHistoryEntries`. Inspect `history/web/index.ts:53-69`.
4. Symptom: old REST requests lose `description`. Test: parse v16 request through `HoppRESTRequest`. Inspect `rest/v/17.ts:5-18`.
5. Symptom: GraphQL subscription errors disappear silently. Test: force `runGQLSubscription` error and assert `gqlClientError$`. Inspect `GQLClient.ts:272-333`.

## Git History Learning Exercise

I did not run an expensive history analysis. Use this local command when you study:

```bash
git log --oneline --decorate -n 30
```

Realistic commit themes to look for:

1. Prisma migration commits: imply evolving domain and persistence constraints.
2. Auth/token commits: imply security hardening and production incidents.
3. Desktop platform commits: imply platform boundary pressure.
4. Request schema version commits: imply backwards compatibility concerns.
5. Test regression commits: imply brittle behavior worth understanding.

## If I Owned This Codebase

| Item | Effort | Impact | Risk | Why It Matters |
|---|---|---:|---|---|
| Add ownership predicates to user history mutations | S | High | Low | Prevents cross-user access if IDs leak. |
| Add safe JSON parse errors in history service | S | Medium | Low | Converts crashes into typed client errors. |
| Add Prisma index for history reads | S | Medium | Low | Protects hot history queries at scale. |
| Write an official local setup doc | M | High | Low | Speeds onboarding and reduces support load. |
| Audit GraphQL network-only query usage | M | Medium | Medium | Reduces backend load without stale UI. |
| Add integration test for history sync | L | High | Medium | Catches store/API/subscription regressions. |

