# Real Code Patterns

## Technology: Vue 3

### Where It Is Used

Shared app UI under `packages/hoppscotch-common/src`, self-host shell under `packages/hoppscotch-selfhost-web/src`, admin shell under `packages/hoppscotch-sh-admin/src`, and agent UI under `packages/hoppscotch-agent/src`.

### Important Files

- `packages/hoppscotch-common/src/App.vue` - root component.
- `packages/hoppscotch-common/src/pages/index.vue` - REST workspace route.
- `packages/hoppscotch-common/src/components/http/Request.vue` - request toolbar and send behavior.

### Best-Practice Example 1: Route Page Composes Feature Tabs

File path: `packages/hoppscotch-common/src/pages/index.vue:47-63`

```vue
<HttpExampleResponseTab v-if="tab.document.type === 'example-response'" />
<HttpTestRunner v-if="tab.document.type === 'test-runner'" />
<HttpRequestTab v-if="tab.document.type === 'request'" />
<!-- The discriminated document type decides which child component owns the UI. -->
```

What it does: keeps the page responsible for layout and tab selection, while specialized components own content.

What juniors miss: `v-if` is not just conditional rendering; here it enforces a domain split.

What seniors notice: this is maintainable as long as document type remains explicit and tab state is tested.

### Best-Practice Example 2: Request UI Delegates Execution

File path: `packages/hoppscotch-common/src/components/http/Request.vue:340-365`

```ts
const newSendRequest = async () => {
  if (newEndpoint.value === "" || /^\s+$/.test(newEndpoint.value)) {
    toast.error(`${t("empty.endpoint")}`)
    return
  }

  const [cancel, streamPromise] = runRESTRequest$(tab)
}
// The component validates obvious UI input, then delegates the real runner.
```

Why it is good: request execution stays outside the component, so platform/interceptor behavior can evolve.

### Weak or Risky Pattern

`Request.vue` is large and owns UI state, validation, analytics, running, response updates, save/share/curl/codegen modal state, and cancellation. See `Request.vue:238-520`. This is understandable for a central control surface, but future work should extract only when duplication or test pain appears.

### Anti-Patterns to Avoid

- Importing self-host platform implementations into common components.
- Adding network calls directly to UI components when helpers/services already exist.
- Hiding request state in local refs when the tab document is the source of truth.

### Interview Talking Points

- "The REST route composes tab content by document type."
- "The request toolbar edits typed tab state and delegates execution to a runner."
- "The shared UI is platform-agnostic because platform services are injected at startup."

### Mini Practice Tasks

- Add a small computed display based on `restHistory$`.
- Extract and test `isCURL` only if you can preserve behavior.
- Explain `pages/index.vue` in a 2-minute walkthrough.

## Technology: TypeScript, Zod, And Versioned Schemas

### Where It Is Used

`packages/hoppscotch-data` defines shared domain schemas; frontend/backend packages consume generated or inferred types.

### Important Files

- `packages/hoppscotch-data/src/rest/index.ts`
- `packages/hoppscotch-data/src/rest/v/17.ts`
- `packages/hoppscotch-data/src/collection/index.ts`

### Best-Practice Example 1: Runtime Schema Plus Inferred Type

File path: `packages/hoppscotch-data/src/rest/index.ts:80-115`

```ts
export const HoppRESTRequest = createVersionedEntity({ latestVersion: 17, versionMap: {...} })
export type HoppRESTRequest = InferredEntity<typeof HoppRESTRequest>
// The same source gives runtime parsing and TypeScript shape.
```

Why it is good: saved/imported request objects can be upgraded and checked, not merely assumed.

### Best-Practice Example 2: Version Migration

File path: `packages/hoppscotch-data/src/rest/v/17.ts:5-18`

```ts
export const V17_SCHEMA = V16_SCHEMA.extend({
  v: z.literal("17"),
  description: z.string().nullable().catch(null),
})
// Old requests gain a safe default for the new field.
```

### Weak or Risky Pattern

Backend history GraphQL exposes `request` and `responseMetadata` as strings in `packages/hoppscotch-backend/src/user-history/user-history.model.ts:21-29`, then parses in `user-history.service.ts:69-77`. This boundary needs safe parse handling.

### Anti-Patterns to Avoid

- Adding persisted fields without schema version migration.
- Treating TypeScript types as runtime validation.
- Using `any` to bypass generated GraphQL or schema types.

### Interview Talking Points

- "Hoppscotch uses versioned runtime schemas to keep old saved requests valid."
- "TypeScript is strongest where runtime schema or generated GraphQL types back it."

### Mini Practice Tasks

- Trace how v16 REST requests become v17.
- Add a documentation-only migration plan for a hypothetical `tags` field.
- Find one JSON boundary and propose validation.

## Technology: Custom Store And RxJS

### Where It Is Used

State under `packages/hoppscotch-common/src/newstore`, including history, collections, settings, local state, realtime sessions.

### Important Files

- `packages/hoppscotch-common/src/newstore/DispatchingStore.ts`
- `packages/hoppscotch-common/src/newstore/history.ts`

### Best-Practice Example 1: Typed Dispatch Union

File path: `packages/hoppscotch-common/src/newstore/DispatchingStore.ts:24-32`

```ts
type Dispatch<StoreType, DispatchersType> = {
  [Dispatcher in keyof DispatchersType]: {
    dispatcher: Dispatcher
    payload: Parameters<DispatchersType[Dispatcher]>[1]
  }
}[keyof DispatchersType]
// The dispatcher name determines the payload type.
```

### Best-Practice Example 2: Store Side Effect From Completed Requests

File path: `packages/hoppscotch-common/src/newstore/history.ts:354-372`

```ts
executedResponses$.subscribe((res) => {
  const { _ref_id, id, ...request } = res.req
  addRESTHistoryEntry(makeRESTHistoryEntry({ request, responseMeta: {...}, star: false }))
})
// Centralizes history capture instead of making every UI caller remember it.
```

### Weak or Risky Pattern

`removeDuplicateEntry` mutates `currentVal.state` with `splice` in `history.ts:178-190`. It returns state afterward, but mutation inside dispatcher can surprise readers and tests.

### Anti-Patterns to Avoid

- Directly mutating store state from components.
- Adding sync behavior inside generic UI components.
- Forgetting unsubscribe/cleanup for stream subscribers.

### Interview Talking Points

- "The custom store provides typed dispatcher payloads and RxJS streams."
- "History sync observes local store dispatches rather than living in the button component."

### Mini Practice Tasks

- Add a derived selector for starred history count.
- Write a test for `addEntry` capping history at 50.

## Technology: GraphQL And URQL

### Where It Is Used

Frontend API calls and subscriptions use URQL and generated GraphQL documents; backend uses Nest GraphQL decorators.

### Important Files

- `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts`
- `packages/hoppscotch-common/src/composables/graphql.ts`
- `packages/hoppscotch-selfhost-web/src/platform/history/web/api.ts`
- `packages/hoppscotch-backend/src/user-history/user-history.resolver.ts`

### Best-Practice Example 1: Auth Exchange

File path: `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts:71-118`

```ts
authExchange(async () => ({
  addAuthToOperation(operation) {
    const authHeaders = platform.auth.getBackendHeaders()
    return makeOperation(operation.kind, operation, { ...operation.context, fetchOptions: { headers: { ...authHeaders } } })
  },
  async refreshAuth() { await authRetryGuard.execute(() => refresh.call(platform.auth)) },
}))
// Auth is centralized in the GraphQL client, not repeated in every query.
```

### Best-Practice Example 2: Thin API Wrapper

File path: `packages/hoppscotch-selfhost-web/src/platform/history/web/api.ts:51-64`

```ts
export const createUserHistory = (reqData, resMetadata, reqType) =>
  runMutation(CreateUserHistoryDocument, { reqData, resMetadata, reqType })()
```

### Weak or Risky Pattern

`GQLClient.ts:28` has `TODO: Implement caching`, and many queries use `network-only` in `GQLClient.ts:217-220`. This favors correctness/freshness but may cost performance.

### Anti-Patterns to Avoid

- Bypassing generated documents with raw strings.
- Handling auth refresh separately in each feature.
- Ignoring subscription duplicate events.

### Interview Talking Points

- "The frontend wraps generated GraphQL operations in small platform API functions."
- "URQL auth exchange centralizes token attachment and refresh."

### Mini Practice Tasks

- Trace `createUserHistory` from frontend wrapper to backend resolver.
- Find one query using `useGQLQuery` and explain loading/error behavior.

## Technology: NestJS And Prisma

### Where It Is Used

Backend package under `packages/hoppscotch-backend`.

### Important Files

- `src/main.ts`
- `src/app.module.ts`
- `src/user-history/user-history.resolver.ts`
- `src/user-history/user-history.service.ts`
- `prisma/schema.prisma`
- `src/prisma/prisma.service.ts`

### Best-Practice Example 1: Backend Bootstrap

File path: `packages/hoppscotch-backend/src/main.ts:58-89`

```ts
app.enableCors(...)
app.enableVersioning({ type: VersioningType.URI })
app.useGlobalPipes(new ValidationPipe({ transform: true }))
app.use(morgan(":remote-addr :method :url :status - :response-time ms"))
// Cross-cutting backend concerns are configured once at startup.
```

### Best-Practice Example 2: Resolver Delegates To Service

File path: `packages/hoppscotch-backend/src/user-history/user-history.resolver.ts:26-57`

```ts
@UseGuards(GqlAuthGuard)
async createUserHistory(@GqlUser() user: User, ...) {
  const createdHistory = await this.userHistoryService.createUserHistory(user.uid, ...)
  if (E.isLeft(createdHistory)) throwErr(createdHistory.left)
  return createdHistory.right
}
```

### Best-Practice Example 3: Prisma Service Encapsulates Connection

File path: `packages/hoppscotch-backend/src/prisma/prisma.service.ts:14-47`

```ts
const pool = new pg.Pool({ connectionString: parsed.connectionString, max: parsed.connectionLimit ?? 20 })
const adapter = new PrismaPg(pool, { schema: parsed.schema })
super({ adapter, transactionOptions: { maxWait: 5000, timeout: 10000 } })
```

### Weak or Risky Pattern

History update/delete should enforce ownership in database logic. See `user-history.service.ts:101-162`.

### Anti-Patterns to Avoid

- Putting Prisma queries in resolvers.
- Trusting authenticated user context without ownership checks.
- Adding migrations without generated client and tests.

### Interview Talking Points

- "Nest modules compose backend domains."
- "Resolvers stay thin; services own business logic and Prisma calls."
- "Prisma schema stores flexible request data as JSON."

### Mini Practice Tasks

- Add a documentation-only service test for malformed history JSON.
- Draw the `User` -> `UserHistory` Prisma relationship.

## Technology: Testing

### Where It Is Used

Vitest in common package; Jest in backend and CLI; many sandbox tests.

### Important Files

- `packages/hoppscotch-common/src/services/__tests__/workspace.service.spec.ts`
- `packages/hoppscotch-backend/src/user-history/user-history.service.spec.ts`
- `packages/hoppscotch-cli/src/__tests__/functions/request/requestRunner.spec.ts`

### Best-Practice Example 1: Mock Boundaries, Test Service Behavior

File path: `packages/hoppscotch-common/src/services/__tests__/workspace.service.spec.ts:52-80`

```ts
const container = new TestContainer()
const service = container.bind(WorkspaceService)
expect(service.currentWorkspace.value).toEqual({ type: "personal" })
// The test verifies service default behavior through the DI container.
```

### Best-Practice Example 2: Backend Service Tests

File path: `packages/hoppscotch-backend/src/user-history/user-history.service.spec.ts:143-218`

```ts
await userHistoryService.createUserHistory("abc", JSON.stringify([{}]), JSON.stringify([{}]), "REST")
// The test checks service result and mock Prisma behavior without starting Nest.
```

### Weak or Risky Pattern

Some high-value integration flows, such as frontend history sync through backend subscriptions, are not obvious from inspected tests. Unit coverage is useful but may not catch cross-layer contract drift.

### Anti-Patterns to Avoid

- Mocking the exact behavior you want to verify.
- Only testing success paths.
- Skipping ownership/security tests because guards exist.

### Interview Talking Points

- "The repo uses small unit tests around services and utilities."
- "A senior improvement would add cross-layer tests for sync flows."

### Mini Practice Tasks

- Write a failing test for malformed history JSON.
- Write a failing test for mismatched `uid` deletion.

