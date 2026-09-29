# Tier 2 Mid-Level Missions

### Mission 9: State Has a Home and a Reason

**Tier:** Mid-Level
**Time Estimate:** 45 minutes
**Goal:** Explain why history state lives in `newstore/history.ts`.
**The Concept:** State is inventory. The request UI should not keep the only copy of the receipt drawer.
**Design Intent Before You Read the Code:** History state must be observable, mutable through known actions, and syncable. Poor design would scatter array mutations across components.
**Find It In The Code:** `packages/hoppscotch-common/src/newstore/history.ts:13-28`, `134-192`, `257-372`.

```ts
// history.ts:134-146
addEntry(currentVal, { entry }) {
  return { state: [entry, ...currentVal.state].slice(0, HISTORY_LIMIT) }
}
// The newest entry is prepended and capped in one store action.
```

**The Aha Moment:** A store is not just storage; it encodes allowed state transitions.
**Socratic Checkpoint:** What are the allowed REST history actions? What is derived as a stream? What side effect writes into the store? Why cap at 50? What risk exists in `removeDuplicateEntry` mutating state?
**How to Self-Grade:** Strong answers distinguish store state, dispatchers, streams, and side effects.
**Connects To:** Mission 10 and Mission 11.

### Mission 10: The Custom Hook Or Reusable Abstraction Ecosystem

**Tier:** Mid-Level
**Time Estimate:** 50 minutes
**Goal:** Read custom Vue composables and service abstractions as reusable contracts.
**The Concept:** Abstractions are reusable tools in the API workshop.
**Design Intent Before You Read the Code:** GraphQL behavior, polling, subscriptions, and service state should be reusable. Poor design duplicates loading/error logic in every component.
**Find It In The Code:** `packages/hoppscotch-common/src/composables/graphql.ts:47-236`, `packages/hoppscotch-common/src/services/workspace.service.ts:36-228`.

```ts
// composables/graphql.ts:47-60
export const useGQLQuery = <DocType, DocVarType, DocErrorType>(_args) => {
  const loading = ref(true)
  const data: Ref<E.Either<GQLError<DocErrorType>, DocType>> = ref() as any
}
// The composable standardizes query loading and typed error shape.
```

**The Aha Moment:** Reuse is most valuable when it hides repeated coordination, not business meaning.
**Socratic Checkpoint:** What does `useGQLQuery` own? What does `WorkspaceService` own? Why is `Either` useful? What does polling add? What should remain outside the abstraction?
**How to Self-Grade:** Strong answers cite loading/data/pause/execute from `graphql.ts:226-235`.
**Connects To:** Mission 12.

### Mission 11: Side Effects Are Promises To The System

**Tier:** Mid-Level
**Time Estimate:** 45 minutes
**Goal:** Identify side effects and their cleanup responsibilities.
**The Concept:** A side effect is a promise that something outside your function will change.
**Design Intent Before You Read the Code:** Subscriptions, timers, auth events, and store sync must start and stop predictably. Poor design leaks listeners.
**Find It In The Code:** `history/web/index.ts:42-70`, `72-95`, `composables/graphql.ts:82-94`, `136-200`.

```ts
// history/web/index.ts:72-95
const userHistoryCreatedSub = setupUserHistoryCreatedSubscription()
return () => { subs.forEach((sub) => sub.unsubscribe()) }
// Subscription setup returns cleanup. That is the side-effect contract.
```

**The Aha Moment:** Senior debugging often starts by asking, "Who started this side effect, and who stops it?"
**Socratic Checkpoint:** Which code starts history sync? Which code listens to login/logout? Which code unsubscribes GraphQL subscriptions? Which code clears polling intervals? What leak would show up first?
**How to Self-Grade:** Strong answers mention `onInvalidate` in `graphql.ts:89-91` and unsubscribe cleanup in `history/web/index.ts:92-94`.
**Connects To:** Mission 14.

### Mission 12: The Full API Or Module Contract

**Tier:** Mid-Level
**Time Estimate:** 55 minutes
**Goal:** Map the frontend GraphQL history API wrapper to backend resolver names.
**The Concept:** API contracts are the menu between frontend and backend kitchens.
**Design Intent Before You Read the Code:** Frontend should call generated documents through small functions; backend should expose typed resolver methods. Poor design spreads raw query strings.
**Find It In The Code:** `history/web/api.ts:35-127`, `user-history.resolver.ts:26-170`, `user-history.model.ts:4-57`.

```ts
// history/web/api.ts:51-64
export const createUserHistory = (reqData, resMetadata, reqType) =>
  runMutation(CreateUserHistoryDocument, { reqData, resMetadata, reqType })()
// The wrapper names the operation and hides generated document details.
```

**The Aha Moment:** The contract is not one file; it is generated documents, wrapper functions, resolver args, and GraphQL object models.
**Socratic Checkpoint:** Which wrapper creates history? Which mutation handles it? Which model field stores request JSON? How do subscriptions connect? What happens on auth failure?
**How to Self-Grade:** Strong answers connect `CreateUserHistoryDocument` to `createUserHistory` resolver and `UserHistory` model.
**Connects To:** Mission 13 and Mission 14.

### Mission 13: The Middleware, Pipeline, Or Boundary Chain

**Tier:** Mid-Level
**Time Estimate:** 50 minutes
**Goal:** Trace auth, throttling, validation, and GraphQL context boundaries.
**The Concept:** Boundaries are checkpoints before data reaches business logic.
**Design Intent Before You Read the Code:** The backend should validate/transform DTOs, throttle requests, parse cookies, attach auth, and guard GraphQL fields. Poor design lets unauthenticated or malformed traffic reach services.
**Find It In The Code:** `backend/src/main.ts:72-89`, `backend/src/app.module.ts:48-95`, `backend/src/guards/gql-auth.guard.ts:5-11`, `user-history.resolver.ts:16-30`.

```ts
// main.ts:76-80
app.useGlobalPipes(new ValidationPipe({ transform: true }))
// Controller DTOs are transformed/validated before route logic.
```

**The Aha Moment:** A resolver method is only safe if its surrounding boundary chain is safe.
**Socratic Checkpoint:** Where is CORS configured? Where is validation configured? How do subscription connections get auth? What does `GqlAuthGuard.getRequest` return? Which guards wrap user history?
**How to Self-Grade:** Strong answers cite `app.module.ts:59-88` for subscription auth and `user-history.resolver.ts:16-30` for guards.
**Connects To:** Mission 22.

### Mission 14: End-to-End Feature Trace

**Tier:** Mid-Level
**Time Estimate:** 75 minutes
**Goal:** Trace REST history persistence from UI to database and back to UI.
**The Concept:** End-to-end tracing follows the package from counter to receipt archive to live update.
**Design Intent Before You Read the Code:** Each layer should have one job. Poor design makes the UI know database details or the database know UI tab state.
**Find It In The Code:** `Request.vue:340-441`, `history.ts:354-372`, `sync.ts:26-60`, `api.ts:51-91`, `resolver.ts:26-57`, `service.ts:60-93`, `schema.prisma:152-161`, `history/web/index.ts:153-190`.

```ts
// sync.ts:34-45
const res = await createUserHistory(JSON.stringify(entry.request), JSON.stringify(entry.responseMeta), ReqType.Rest)
if (E.isRight(res)) {
  entry.id = res.right.createUserHistory.id
  removeDuplicateRestHistoryEntry(entry.id)
}
// Local store change becomes a backend mutation, then avoids duplicate subscription echo.
```

**The Aha Moment:** A full-stack feature is a chain of small contracts, not one giant block.
**Socratic Checkpoint:** Where does the user action start? Where does the local history snapshot get created? Where is it serialized? Where is auth checked? Where is it persisted? How does another client receive it?
**How to Self-Grade:** Strong answers can draw the chain without looking and cite at least six file ranges.
**Connects To:** Mission 15 and Mission 23.

### Mission 15: Read The Diff Like An Engineer

**Tier:** Mid-Level
**Time Estimate:** 45 minutes
**Goal:** Practice reviewing a hypothetical history change.
**The Concept:** A diff is a proposed story about the system. Your job is to find missing chapters.
**Design Intent Before You Read the Code:** A change to persisted request data may need schema, UI, sync, backend, and tests.
**Find It In The Code:** `rest/v/17.ts:5-18`, `history.ts:13-28`, `sync.ts:26-60`, `user-history.service.ts:60-93`, `user-history.service.spec.ts:143-218`.

```ts
// rest/v/17.ts:13-18
up(old) { return { ...old, v: "17", description: null } }
// Schema migrations are part of reviewing persisted-data changes.
```

**The Aha Moment:** Good review checks every boundary touched by the data, not just the edited file.
**Socratic Checkpoint:** What tests should change if request shape changes? What generated files may update? What import/export path could break? What backend parse risk exists? What UI might need a fallback?
**How to Self-Grade:** Strong answers include a review checklist and at least one missing-test suggestion.
**Connects To:** Mission 19.

### Mission 16: Composition Over Inheritance

**Tier:** Mid-Level
**Time Estimate:** 40 minutes
**Goal:** See how Hoppscotch composes modules, platform capabilities, and services.
**The Concept:** Composition is building a request station from replaceable parts.
**Design Intent Before You Read the Code:** The app composes modules and platform services rather than subclassing app variants. Poor design would fork web and desktop apps.
**Find It In The Code:** `common/src/index.ts:65-66`, `selfhost-web/src/main.ts:171-203`, `backend/src/app.module.ts:42-138`.

```ts
// common/src/index.ts:65-66
HOPP_MODULES.forEach((mod) => mod.onVueAppInit?.(app))
platformDef.addedHoppModules?.forEach((mod) => mod.onVueAppInit?.(app))
// Core and platform-added modules compose at startup.
```

**The Aha Moment:** The repo prefers plugging capabilities together over inheritance trees.
**Socratic Checkpoint:** Where are modules composed? Where are backend modules composed? What does platform composition buy desktop? What risk does composition introduce? How do you find a capability's implementation?
**How to Self-Grade:** Strong answers connect frontend module composition to Nest module composition without pretending they are the same mechanism.
**Connects To:** Mission 18.

### Mission 17: TypeScript's Hidden Work

**Tier:** Mid-Level
**Time Estimate:** 50 minutes
**Goal:** Identify where TypeScript prevents invalid flows and where runtime validation still matters.
**The Concept:** TypeScript is a safety rail, not a locked door.
**Design Intent Before You Read the Code:** Compile-time types help within the app; Zod/GraphQL/DTOs are needed at external boundaries.
**Find It In The Code:** `rest/index.ts:80-115`, `workspace.service.ts:16-27`, `GQLClient.ts:197-270`, `user-history.service.ts:69-77`.

```ts
// workspace.service.ts:16-27
export type Workspace = PersonalWorkspace | TeamWorkspace
// The `type` field lets code narrow what fields are available.
```

**The Aha Moment:** Strong TypeScript code still needs runtime checks where data enters from storage, network, or user input.
**Socratic Checkpoint:** Which types are runtime-backed? Which are compile-only? Where is GraphQL typed? Where can malformed JSON still crash? What would a safer boundary look like?
**How to Self-Grade:** Strong answers separate TypeScript, Zod/verzod, GraphQL decorators, class-validator, and Prisma.
**Connects To:** Mission 23.

