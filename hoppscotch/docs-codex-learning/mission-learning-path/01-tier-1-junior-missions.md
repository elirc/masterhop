# Tier 1 Junior Missions

### Mission 1: The App's Heartbeat

**Tier:** Junior

**Time Estimate:** 35 minutes

**Goal:** Identify how the app starts and where platform capabilities enter.

**The Concept:** Startup code is the front door. In Hoppscotch, the self-host shell is like the front desk deciding whether the visitor gets web or desktop equipment before entering the shared workshop.

**Design Intent Before You Read the Code:** `selfhost-web/src/main.ts` chooses platform definitions; `common/src/index.ts` mounts the shared Vue app. Poor design here would hard-code web or desktop behavior into shared UI.

**Find It In The Code:** Open `packages/hoppscotch-selfhost-web/src/main.ts:42-204` and `packages/hoppscotch-common/src/index.ts:24-70`.

```ts
// packages/hoppscotch-selfhost-web/src/main.ts:42-78
const PLATFORM_CONFIG = { web: {...}, desktop: {...} }
// The shell selects capabilities before the common app starts.

// packages/hoppscotch-common/src/index.ts:36-68
setPlatformDef(platformDef)
const app = createApp(App)
HOPP_MODULES.forEach((mod) => mod.onVueAppInit?.(app))
app.mount(el)
// Common app receives capabilities, initializes modules, then mounts.
```

**The Aha Moment:** The shared app is reusable because platform behavior is injected, not scattered through every component.

**Socratic Checkpoint:** What calls `createHoppApp`? What does `PlatformDef` provide? What initializes before mounting? What runs after mounting? Why is this better than importing web auth directly in common components?

**How to Self-Grade:** Strong answers cite `main.ts:155-204` and `index.ts:24-70`, and explain dependency direction: shell depends on common, common depends on abstract platform.

**Connects To:** Mission 2 because package boundaries make the startup design easier to navigate.

### Mission 2: The Folder Mental Map

**Tier:** Junior

**Time Estimate:** 45 minutes

**Goal:** Build a package-level map of the monorepo.

**The Concept:** A monorepo is a city. You do not memorize every street first; you learn districts.

**Design Intent Before You Read the Code:** Root `package.json` and package folders show ownership. Poor navigation leads to changing a shell package when the reusable feature belongs in common.

**Find It In The Code:** Open `package.json:1-36`, `pnpm-workspace.yaml:1-2`, and inspect package folders.

```json
// package.json:8-18
"packageManager": "pnpm@10.33.2",
"scripts": { "dev": "pnpm -r do-dev", "test": "pnpm -r do-test" }
// Root scripts delegate to every workspace package.
```

**The Aha Moment:** Most feature work starts by deciding which package owns the change.

**Socratic Checkpoint:** Which package owns shared UI? Which owns backend GraphQL? Which owns request schemas? Which owns self-host sync? Which owns CLI tests?

**How to Self-Grade:** Strong answers name `hoppscotch-common`, `hoppscotch-backend`, `hoppscotch-data`, `hoppscotch-selfhost-web`, and `hoppscotch-cli` with one responsibility each.

**Connects To:** Mission 3 because schemas live in their own package.

### Mission 3: TypeScript Is a Contract

**Tier:** Junior

**Time Estimate:** 40 minutes

**Goal:** Understand `HoppRESTRequest` as a runtime schema and TypeScript type.

**The Concept:** A request object is a passport. The schema checks it can cross boundaries.

**Design Intent Before You Read the Code:** `@hoppscotch/data` uses versioned schemas so old saved/imported requests can become current requests. Without that, saved collections would break after shape changes.

**Find It In The Code:** Open `packages/hoppscotch-data/src/rest/index.ts:75-115`, `242-260`, and `packages/hoppscotch-data/src/rest/v/17.ts:5-18`.

```ts
// packages/hoppscotch-data/src/rest/index.ts:80-115
export const HoppRESTRequest = createVersionedEntity({ latestVersion: 17, versionMap: {...} })
export type HoppRESTRequest = InferredEntity<typeof HoppRESTRequest>
// One declaration gives runtime parsing and compile-time typing.
```

**The Aha Moment:** TypeScript safety here comes from schema-backed data, not just annotations.

**Socratic Checkpoint:** What is the latest REST schema version? How does v17 add `description`? What does `safeParse` protect? Why is `getDefaultRESTRequest` important? What breaks if a migration is missing?

**How to Self-Grade:** Strong answers mention `latestVersion: 17`, `V17_SCHEMA`, and the difference between runtime validation and editor autocomplete.

**Connects To:** Mission 6 because props and parameters use these contracts.

### Mission 4: Your First UI Component

**Tier:** Junior

**Time Estimate:** 35 minutes

**Goal:** Read a Vue component by separating template, imports, state, and methods.

**The Concept:** The request bar is a control panel: method selector, URL input, send button, save actions.

**Design Intent Before You Read the Code:** `Request.vue` should render controls and delegate execution. If it owned networking directly, testing and platform behavior would be harder.

**Find It In The Code:** Open `packages/hoppscotch-common/src/components/http/Request.vue:1-80`, `238-340`, `340-441`.

```vue
<!-- Request.vue:58-67 -->
<SmartEnvInput v-model="tab.document.request.endpoint" @enter="newSendRequest" />
<!-- The endpoint lives in the tab document, not hidden local state. -->
```

**The Aha Moment:** The component edits request state but delegates sending to `runRESTRequest$`.

**Socratic Checkpoint:** What prop does the component receive? Where is URL state stored? What triggers send? What shows loading? What function normalizes missing `http://`?

**How to Self-Grade:** Strong answers cite `Request.vue:294-312`, `340-365`, and `443-454`.

**Connects To:** Mission 7 because sending creates data that flows into history.

### Mission 5: Your First Backend Route Or Core Function

**Tier:** Junior

**Time Estimate:** 45 minutes

**Goal:** Read a NestJS GraphQL mutation from decorator to service call.

**The Concept:** A resolver is a receiving desk. It checks credentials, accepts a form, then hands the work to a specialist.

**Design Intent Before You Read the Code:** Resolvers should be thin and services should hold business logic. Poor design puts database details inside decorators.

**Find It In The Code:** Open `packages/hoppscotch-backend/src/user-history/user-history.resolver.ts:16-57` and `packages/hoppscotch-backend/src/user-history/user-history.service.ts:60-93`.

```ts
// user-history.resolver.ts:26-57
@UseGuards(GqlAuthGuard)
async createUserHistory(@GqlUser() user: User, @Args("reqData") reqData: string) {
  const createdHistory = await this.userHistoryService.createUserHistory(user.uid, reqData, ...)
}
// Auth and argument extraction happen here; persistence is delegated.
```

**The Aha Moment:** Backend boundaries are layered: guard -> resolver -> service -> Prisma.

**Socratic Checkpoint:** Which guard protects this mutation? Where does `user.uid` come from? Which service method writes data? What does the resolver do on `E.left`? What type does it return?

**How to Self-Grade:** Strong answers name `GqlAuthGuard`, `@GqlUser`, `throwErr`, and `UserHistory`.

**Connects To:** Mission 13 because guards are part of the boundary chain.

### Mission 6: Props, Inputs, Or Parameters Are Contracts

**Tier:** Junior

**Time Estimate:** 30 minutes

**Goal:** See how frontend props and backend args define valid interaction.

**The Concept:** Contracts are order slips. If the order slip is unclear, the kitchen guesses.

**Design Intent Before You Read the Code:** Vue props type the tab input; GraphQL args type backend mutation inputs. Weak contracts produce runtime surprises.

**Find It In The Code:** Open `Request.vue:294-297`, `user-history.resolver.ts:30-48`, and `user-history.model.ts:21-40`.

```ts
// Request.vue:294-297
const props = defineProps<{ modelValue: HoppTab<HoppRequestDocument> }>()
const tab = useVModel(props, "modelValue", emit)
// Parent and child share a typed tab contract.
```

**The Aha Moment:** Contract strength varies: Vue and GraphQL are typed, but request history payloads are still JSON strings.

**Socratic Checkpoint:** What does `modelValue` contain? What does `reqData` guarantee? What does it not guarantee? Why is JSON string weaker than a structured GraphQL input? Where is `responseMetadata` typed?

**How to Self-Grade:** Strong answers distinguish TypeScript compile-time contracts from runtime validation boundaries.

**Connects To:** Mission 12 because API contracts are richer than single functions.

### Mission 7: Following Data Into The App

**Tier:** Junior

**Time Estimate:** 45 minutes

**Goal:** Trace how completed requests become history.

**The Concept:** History is the receipt printer after a request is sent.

**Design Intent Before You Read the Code:** Request execution should not require the UI to manually remember every successful response. A stream can centralize that side effect.

**Find It In The Code:** Open `Request.vue:365-441` and `history.ts:354-372`.

```ts
// history.ts:354-372
executedResponses$.subscribe((res) => {
  const { _ref_id, id, ...request } = res.req
  addRESTHistoryEntry(makeRESTHistoryEntry({ request, responseMeta: {...}, star: false }))
})
// Completed responses are transformed into history snapshots.
```

**The Aha Moment:** The UI sends; a store-level subscriber records history.

**Socratic Checkpoint:** Why remove `_ref_id` and `id`? What response metadata is stored? Why is `star` false? What limits history size? Where does `executedResponses$` come from?

**How to Self-Grade:** Strong answers cite `history.ts:129-146` for the 50-entry limit and `history.ts:354-372` for snapshot creation.

**Connects To:** Mission 9 because this introduces state ownership.

### Mission 8: Navigation, Routing, Or Execution Flow Is The App's Skeleton

**Tier:** Junior

**Time Estimate:** 35 minutes

**Goal:** Understand route generation and route lifecycle hooks.

**The Concept:** Routing is the hallway map of the app.

**Design Intent Before You Read the Code:** File-based pages are generated, layouts are applied, and modules can react before/after route changes. Poor routing would make every feature manually register routes.

**Find It In The Code:** Open `packages/hoppscotch-common/src/modules/router.ts:1-111`.

```ts
// router.ts:7-12
import { setupLayouts } from "virtual:generated-layouts"
import generatedRoutes from "virtual:generated-pages"
const routes = setupLayouts(generatedRoutes)
// Pages become routes through Vite plugins, then get layouts.
```

**The Aha Moment:** Route files are source material; the router module is the runtime coordinator.

**Socratic Checkpoint:** Where do routes come from? What preserves desktop `org` query state? Which hooks run before navigation? Which hooks run after? Where is analytics logged?

**How to Self-Grade:** Strong answers cite `router.ts:50-73`, `77-89`, and `94-110`.

**Connects To:** Mission 14 because end-to-end feature tracing begins at route/page ownership.

