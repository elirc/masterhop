# Architectural Cartographer Journal

## First-Pass Mental Model

Hoppscotch is a full-stack TypeScript-heavy monorepo. The primary user-facing app is a Vue 3 API client. Shared app UI and domain logic live in `packages/hoppscotch-common`; the self-host web shell wires platform-specific auth, sync, history, collections, and interceptors in `packages/hoppscotch-selfhost-web`; the backend is a NestJS application with GraphQL, REST controllers, Prisma, PostgreSQL, guards, subscriptions, and service modules in `packages/hoppscotch-backend`.

The most useful end-to-end teaching path is request history:

- The REST page renders request tabs in `packages/hoppscotch-common/src/pages/index.vue:1-124`.
- The send bar edits and runs a request in `packages/hoppscotch-common/src/components/http/Request.vue:340-441`.
- Completed responses flow into history in `packages/hoppscotch-common/src/newstore/history.ts:354-372`.
- The self-host platform syncs history to GraphQL in `packages/hoppscotch-selfhost-web/src/platform/history/web/sync.ts:26-60`.
- The frontend API wrapper calls generated GraphQL documents in `packages/hoppscotch-selfhost-web/src/platform/history/web/api.ts:35-91`.
- The backend resolver receives mutations and subscriptions in `packages/hoppscotch-backend/src/user-history/user-history.resolver.ts:26-170`.
- The service writes JSON request snapshots to Prisma in `packages/hoppscotch-backend/src/user-history/user-history.service.ts:60-93`.
- The database model is `UserHistory` in `packages/hoppscotch-backend/prisma/schema.prisma:152-161`.

## Important Discoveries

- Routing is file-based through Vite plugins, then wrapped with layouts in `packages/hoppscotch-common/src/modules/router.ts:7-12`.
- Platform wiring is deliberate: `createHoppApp` takes a `PlatformDef` in `packages/hoppscotch-common/src/index.ts:24-68`, while self-host chooses web versus desktop definitions in `packages/hoppscotch-selfhost-web/src/main.ts:42-204`.
- State is not Redux/Zustand. It is a mix of Vue refs/computed values, RxJS streams, and a custom `DispatchingStore` in `packages/hoppscotch-common/src/newstore/DispatchingStore.ts:34-77`.
- Shared request shapes are versioned with Zod and `verzod`, especially `HoppRESTRequest` in `packages/hoppscotch-data/src/rest/index.ts:80-115`.
- Backend errors often use `fp-ts/Either`, for example `UserHistoryService.createUserHistory` returning `E.right` or `E.left` in `packages/hoppscotch-backend/src/user-history/user-history.service.ts:60-93`.

## Teaching Anchor Choices

- `packages/hoppscotch-common/src/index.ts` shows app bootstrapping without burying the reader in feature code.
- `packages/hoppscotch-selfhost-web/src/main.ts` shows the platform boundary, one of the repo's core architectural ideas.
- `packages/hoppscotch-common/src/components/http/Request.vue` is a concrete UI surface users understand: method, URL, send, save.
- `packages/hoppscotch-common/src/newstore/history.ts` shows state, domain types, derived streams, and side effects.
- `packages/hoppscotch-selfhost-web/src/platform/history/web/*` shows sync and GraphQL boundaries.
- `packages/hoppscotch-backend/src/user-history/*` gives a compact resolver-service-Prisma path.
- `packages/hoppscotch-data/src/rest/index.ts` teaches versioned domain schemas, which is a senior-level pattern hiding in plain sight.

## Where Engineers Get Confused

- A junior may expect React, Redux, Express, and REST everywhere. This repo mainly uses Vue, custom stores, URQL GraphQL, NestJS, and Prisma.
- A mid-level engineer should slow down at the platform boundary. The same common UI can run in web or desktop shells because platform capabilities are injected.
- A senior engineer should be skeptical of JSON-string request payloads crossing GraphQL and database boundaries. This is practical for flexible API-client data, but validation boundaries must be watched carefully.

## Weak or Missing Patterns

- Setup is spread across root scripts, package scripts, `.env.example`, Docker Compose, and package postinstall hooks. A single beginner-friendly local setup path is not obvious.
- Request history service methods accept JSON strings and call `JSON.parse` without local try/catch in `packages/hoppscotch-backend/src/user-history/user-history.service.ts:69-77`.
- Some backend service methods accept `uid` but do not use it in the Prisma `where` condition for updates/deletes, such as `toggleHistoryStarStatus` and `removeRequestFromHistory` in `packages/hoppscotch-backend/src/user-history/user-history.service.ts:101-162`. The resolver authenticates the user, but ownership should ideally be enforced at the database query boundary too.
- There are many tests, but coverage appears uneven by package and feature depth. User history service has unit tests, while full frontend-to-backend integration is harder to see.

## How To Use Checkpoints

For each checkpoint, write answers that name files and explain cause/effect. Strong answers say "I know this because..." and cite a line range. Senior-level answers also name what could break if the assumption changes.

