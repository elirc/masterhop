# Mid-Level Engineer Guide

## Architecture Diagram

```text
User
  -> Vue route page (`common/src/pages/index.vue`)
  -> REST request component (`common/src/components/http/Request.vue`)
  -> Request runner / kernel / interceptor layer
  -> `executedResponses$`
  -> local history store (`common/src/newstore/history.ts`)
  -> self-host sync adapter (`selfhost-web/src/platform/history/web/sync.ts`)
  -> URQL GraphQL client (`common/src/helpers/backend/GQLClient.ts`)
  -> NestJS GraphQL resolver (`backend/src/user-history/user-history.resolver.ts`)
  -> service (`backend/src/user-history/user-history.service.ts`)
  -> Prisma model (`backend/prisma/schema.prisma`)
  -> PostgreSQL
```

## Type System Deep Dive

`HoppRESTRequest` is versioned with `verzod` and Zod in `packages/hoppscotch-data/src/rest/index.ts:80-115`. That means persisted or imported request objects can be parsed and upgraded rather than trusted blindly.

```ts
// packages/hoppscotch-data/src/rest/v/17.ts:5-18
export const V17_SCHEMA = V16_SCHEMA.extend({
  v: z.literal("17"),
  description: z.string().nullable().catch(null),
})
// A new field is added through a migration step, so old requests can become new requests.
```

Mid-level insight: the type is not just IDE autocomplete. It is a compatibility boundary for saved requests, imports, history snapshots, and collection sync.

## State Management Deep Dive: History

History has a typed entry and a dispatcher-backed store:

- Type: `packages/hoppscotch-common/src/newstore/history.ts:13-28`
- Dispatchers: `packages/hoppscotch-common/src/newstore/history.ts:134-192`
- Store/stream export: `packages/hoppscotch-common/src/newstore/history.ts:257-268`
- Completed-response listener: `packages/hoppscotch-common/src/newstore/history.ts:354-372`

`DispatchingStore` gives typed dispatch payloads in `packages/hoppscotch-common/src/newstore/DispatchingStore.ts:24-32`, then emits new state through a `BehaviorSubject` in `DispatchingStore.ts:42-77`.

## API Contract Map

- Frontend generated documents are called through small wrappers in `packages/hoppscotch-selfhost-web/src/platform/history/web/api.ts:35-91`.
- The low-level URQL client attaches auth headers and handles refresh in `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts:71-118`.
- Backend GraphQL schema is generated from decorators in `packages/hoppscotch-backend/src/user-history/user-history.model.ts:4-57` and resolvers in `user-history.resolver.ts:26-170`.
- Database shape is Prisma: `packages/hoppscotch-backend/prisma/schema.prisma:152-161`.

## Component Interaction Map

`pages/index.vue` owns tab-level composition. It passes each active request tab into `HttpRequestTab` in `packages/hoppscotch-common/src/pages/index.vue:58-63`. The request toolbar inside `Http/Request.vue` uses `v-model` on `tab.document.request.endpoint` in `packages/hoppscotch-common/src/components/http/Request.vue:58-67`, then emits state changes through the tab service indirectly.

## Full-Stack Feature Trace: REST History Persistence

1. User edits method/URL in `Request.vue:17-27` and `Request.vue:58-67`.
2. User sends request through `newSendRequest` in `Request.vue:340-441`.
3. Request result eventually appears on `executedResponses$`; `history.ts:354-372` strips collection identifiers and creates a history snapshot.
4. `restHistoryStore` dispatches `addEntry` from `history.ts:277-282`.
5. Self-host sync observes store dispatches and runs `createUserHistory` in `sync.ts:26-46`.
6. `api.ts:51-64` sends the GraphQL mutation.
7. `user-history.resolver.ts:26-57` requires `GqlAuthGuard`, extracts the current user, and calls service logic.
8. `user-history.service.ts:60-93` validates `ReqType`, parses JSON request/metadata, creates the Prisma row, stringifies the row for GraphQL, and publishes a subscription.
9. `schema.prisma:152-161` stores `request` and `responseMetadata` as JSON under `UserHistory`.
10. Other clients receive subscriptions from `user-history.resolver.ts:121-170`; the frontend applies them in `history/web/index.ts:153-316`.

## Diff Reading Exercise

Hypothetical change: "Add a `description` field to history entries."

Read the diff in this order:

1. Data schema: Is `HoppRESTRequest.description` already present? Yes, `packages/hoppscotch-data/src/rest/v/17.ts:5-18`.
2. Local history: Does `RESTHistoryEntry` carry the whole request? Yes, `history.ts:13-28`.
3. Sync: Does JSON stringify preserve it? Yes, `sync.ts:34-37`.
4. Backend: Does Prisma store request JSON? Yes, `schema.prisma:152-161`.
5. UI: Where should description be displayed or edited? likely request save/edit components under `components/collections`.

Review questions: Did the change update import/export paths? Did it preserve old saved requests? Did it add tests for migration and display?

## Non-Obvious Patterns

- Platform injection lets common Vue code run in web and desktop shells. Evidence: `selfhost-web/src/main.ts:171-203` passes auth/sync/interceptors; `common/src/index.ts:24-68` consumes them.
- Store dispatches can be synced externally. Evidence: history store in `history.ts:257-268`, sync definition in `sync.ts:26-60`.
- Backend GraphQL uses subscriptions to keep multiple clients aligned. Evidence: `user-history.resolver.ts:121-170`.
- Prisma stores flexible request JSON because API-client requests evolve often. Evidence: `schema.prisma:152-161` plus versioned request schemas in `hoppscotch-data`.

## Mid-Level Socratic Checkpoint

Questions:

1. What is the boundary between `hoppscotch-common` and `hoppscotch-selfhost-web`?
2. Why does history sync need duplicate-removal logic?
3. Where are auth headers added to GraphQL operations?
4. What parts of request history are strongly typed and what parts are JSON?
5. How would a backend validation failure surface to the frontend?
6. Why might JSON request persistence be acceptable here?
7. What would break if GraphQL subscriptions fired duplicate events?
8. Where would you add a test for malformed history JSON?

How to self-grade: strong answers trace from caller to callee and include both benefit and risk. A senior-leaning answer notes that `removeDuplicateRestHistoryEntry` in `sync.ts:40-45` handles same-client subscription echo, but malformed JSON should probably be guarded in `user-history.service.ts:69-77`.

