# Junior Engineer Guide

## Setup And Run

Found facts:

- Package manager: pnpm, declared as `pnpm@10.33.2` in `package.json:8`.
- Workspaces: root `package.json:19-21` and `pnpm-workspace.yaml:1-2`.
- Root scripts: `dev`, `lint`, `typecheck`, `test`, `generate`, `start` in `package.json:9-18`.
- Backend scripts: Nest build/dev/test and Prisma generation in `packages/hoppscotch-backend/package.json:9-24`.
- Self-host web scripts: Vite dev/build and GraphQL codegen in `packages/hoppscotch-selfhost-web/package.json:7-23`.
- Environment example: `.env.example` exists at repo root.
- Database: PostgreSQL via Prisma in `packages/hoppscotch-backend/prisma/schema.prisma:6-8`.

Reconstructed local path, based on scripts and config:

```bash
pnpm install
cp .env.example .env
docker compose up -d
pnpm -r generate-gql-sdl
pnpm dev
```

Assumptions: Docker Compose is the intended local dependency path; `.env.example` contains the required service URLs/secrets; package postinstall codegen may require the backend schema and placeholder database URL. If codegen fails, inspect `packages/hoppscotch-common/package.json:10-23`, `packages/hoppscotch-selfhost-web/package.json:8-23`, and `packages/hoppscotch-backend/package.json:19`.

Test/build commands:

```bash
pnpm test
pnpm typecheck
pnpm lint
pnpm generate
```

## Folder Orientation

- `packages/hoppscotch-common`: shared Vue app, components, pages, composables, stores, services, platform interfaces.
- `packages/hoppscotch-selfhost-web`: web/desktop shell wiring for auth, sync, history, collections, settings, and instance behavior.
- `packages/hoppscotch-backend`: NestJS backend, GraphQL resolvers, REST controllers, guards, Prisma, auth, teams, history.
- `packages/hoppscotch-data`: versioned domain schemas for REST requests, GraphQL requests, collections, environments.
- `packages/hoppscotch-kernel`: IO, relay, store, log abstractions for web/desktop.
- `packages/hoppscotch-cli`: command-line collection runner and tests.
- `packages/hoppscotch-js-sandbox`: sandbox/test-script compatibility layer with many tests.
- `packages/hoppscotch-agent`, `packages/hoppscotch-desktop`, `packages/hoppscotch-relay`: supporting app/desktop/Rust relay packages.

Three important frontend folders:

- `packages/hoppscotch-common/src/pages`: file-based route pages. Example REST route: `pages/index.vue`.
- `packages/hoppscotch-common/src/components`: reusable Vue UI grouped by domain (`http`, `graphql`, `collections`, `teams`, `app`).
- `packages/hoppscotch-common/src/newstore`: custom state stores using `DispatchingStore`, RxJS streams, and typed dispatcher functions.

Three important backend folders:

- `packages/hoppscotch-backend/src/user-history`: compact example of GraphQL resolver -> service -> Prisma.
- `packages/hoppscotch-backend/src/auth`: REST auth controller, token generation, SSO, magic link, refresh token rotation.
- `packages/hoppscotch-backend/src/guards`: authentication, admin, team, throttling, and feature flag boundaries.

## Entry Point Walkthroughs

Frontend shell:

```ts
// packages/hoppscotch-selfhost-web/src/main.ts:42-78
const PLATFORM_CONFIG = {
  web: {
    auth: webAuth,
    history: webHistory,
    defaultInterceptor: "browser",
    cookiesEnabled: false,
  },
  desktop: {
    auth: desktopAuth,
    history: desktopHistory,
    defaultInterceptor: "native",
    cookiesEnabled: true,
  },
}
// The shell chooses capabilities. The common app does not hard-code
// whether it is running in a browser tab or desktop webview.
```

Shared app creation:

```ts
// packages/hoppscotch-common/src/index.ts:24-43
export async function createHoppApp(el: string | Element, platformDef: PlatformDef) {
  initKernel(getKernelMode())       // initialize runtime kernel
  setPlatformDef(platformDef)       // inject platform services
  const app = createApp(App)        // create Vue app
  const initService = getService(InitializationService)
  await initService.initPre()       // run pre-mount setup
}
```

Backend entry:

```ts
// packages/hoppscotch-backend/src/main.ts:72-80
app.enableVersioning({ type: VersioningType.URI })
app.use(cookieParser())
app.useGlobalPipes(new ValidationPipe({ transform: true }))
// REST endpoints are versioned as /v1/..., cookies are parsed globally,
// and DTO validation/transformation runs before controllers receive data.
```

## Component Anatomies

Simple root component:

```vue
<!-- packages/hoppscotch-common/src/App.vue:1-12 -->
<RouterView v-else />
<Toaster rich-colors />
<!-- The root delegates real screens to Vue Router and keeps global toast UI mounted. -->
```

REST page composition:

```vue
<!-- packages/hoppscotch-common/src/pages/index.vue:47-63 -->
<HttpExampleResponseTab v-if="tab.document.type === 'example-response'" />
<HttpTestRunner v-if="tab.document.type === 'test-runner'" />
<HttpRequestTab v-if="tab.document.type === 'request'" />
<!-- A tab document type decides which feature surface renders. -->
```

Request send bar:

```vue
<!-- packages/hoppscotch-common/src/components/http/Request.vue:58-80 -->
<SmartEnvInput v-model="tab.document.request.endpoint" @enter="newSendRequest" />
<HoppButtonPrimary @click="!isTabResponseLoading ? newSendRequest() : cancelRequest()" />
<!-- URL editing and request execution meet here, but the runner is still delegated. -->
```

## Backend Route/Resolver Anatomies

Health-style REST controller exists at `packages/hoppscotch-backend/src/app.controller.ts:4-8`.

Auth REST route:

```ts
// packages/hoppscotch-backend/src/auth/auth.controller.ts:51-70
@Post("signin")
async signInMagicLink(@Body() authData: SignInMagicDto, @Query("origin") origin: string) {
  const deviceIdToken = await this.authService.signInMagicLink(authData.email, origin)
  if (E.isLeft(deviceIdToken)) throwHTTPErr(deviceIdToken.left)
  return deviceIdToken.right
}
```

GraphQL mutation:

```ts
// packages/hoppscotch-backend/src/user-history/user-history.resolver.ts:26-57
@Mutation(() => UserHistory)
@UseGuards(GqlAuthGuard)
async createUserHistory(@GqlUser() user: User, @Args("reqData") reqData: string) {
  const createdHistory = await this.userHistoryService.createUserHistory(user.uid, reqData, ...)
  if (E.isLeft(createdHistory)) throwErr(createdHistory.left)
  return createdHistory.right
}
```

Prisma service method:

```ts
// packages/hoppscotch-backend/src/user-history/user-history.service.ts:69-77
const history = await this.prisma.userHistory.create({
  data: {
    userUid: uid,
    request: JSON.parse(reqData),
    responseMetadata: JSON.parse(resMetadata),
    reqType: requestType.right,
    isStarred: false,
  },
})
```

## TypeScript Orientation

- Versioned domain types are generated from schemas, not hand-written interfaces: `HoppRESTRequest` in `packages/hoppscotch-data/src/rest/index.ts:80-115`.
- Frontend workspace state uses discriminated unions: `Workspace` in `packages/hoppscotch-common/src/services/workspace.service.ts:16-27`.
- GraphQL client helpers return `Either` so callers must handle success/failure: `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts:210-270`.

## Glossary

- Platform definition: injected capability object for auth, sync, history, collections, interceptors.
- Interceptor: request execution strategy, such as browser, proxy, agent, native, or extension.
- History entry: snapshot of a request plus response metadata after execution.
- Collection: saved tree of requests/folders.
- Workspace: either personal or team context.
- Resolver: NestJS GraphQL handler.
- Service: NestJS or DI class that owns business logic.
- Prisma model: database entity definition.
- Subscription echo: backend subscription event received by the same client that caused it.

## Junior Socratic Checkpoint

Questions:

1. Why does `hoppscotch-selfhost-web` pass platform definitions into `createHoppApp`?
2. What renders the REST request page?
3. Where is the request URL stored while you edit it?
4. What function is called when you press Send?
5. What file records completed REST responses into history?
6. What backend model stores user history?
7. Why is `HoppRESTRequest` a schema-backed type instead of a plain interface?

How to self-grade: strong answers name files, line ranges, and the direction of dependency. For example, "the send button calls `newSendRequest` in `Request.vue:340-441`, which delegates execution to `runRESTRequest$`; completed responses later feed `executedResponses$` in `history.ts:354-372`."

