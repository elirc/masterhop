# Reading Map

## One-Paragraph Mental Model

Hoppscotch is an API workbench: the Vue app lets users compose requests, run them through kernel/interceptor layers, store local state, and optionally sync authenticated data to a NestJS GraphQL backend backed by Prisma/PostgreSQL. Think of a request as a parcel: the UI labels it, shared data schemas define what a valid parcel looks like, the runner sends it, history state records the delivery receipt, sync adapters decide whether the receipt leaves the browser, and backend services persist it for the user's account.

## Top 10 Files To Read In Order

| Order | File | Why Read It Now | Key Lines |
|---|---|---|---|
| 1 | `package.json` | Establish monorepo scripts and pnpm workspace expectations. | `package.json:1-36` |
| 2 | `packages/hoppscotch-selfhost-web/src/main.ts` | Shows platform wiring for web versus desktop shells. | `packages/hoppscotch-selfhost-web/src/main.ts:42-204` |
| 3 | `packages/hoppscotch-common/src/index.ts` | Shows shared app creation, service initialization, module registration, and mount. | `packages/hoppscotch-common/src/index.ts:24-70` |
| 4 | `packages/hoppscotch-common/src/modules/router.ts` | Shows file-generated routes and route lifecycle hooks. | `packages/hoppscotch-common/src/modules/router.ts:12-111` |
| 5 | `packages/hoppscotch-common/src/pages/index.vue` | Shows the REST workspace page and tab rendering. | `packages/hoppscotch-common/src/pages/index.vue:1-124`, `135-220` |
| 6 | `packages/hoppscotch-common/src/components/http/Request.vue` | Shows request method, URL, send, save, and runner integration. | `packages/hoppscotch-common/src/components/http/Request.vue:58-80`, `340-441` |
| 7 | `packages/hoppscotch-common/src/newstore/history.ts` | Shows history state shape, dispatchers, and completed-response subscription. | `packages/hoppscotch-common/src/newstore/history.ts:13-28`, `134-192`, `354-372` |
| 8 | `packages/hoppscotch-selfhost-web/src/platform/history/web/sync.ts` | Shows how local history dispatches become backend mutations. | `packages/hoppscotch-selfhost-web/src/platform/history/web/sync.ts:26-60` |
| 9 | `packages/hoppscotch-backend/src/user-history/user-history.resolver.ts` | Shows GraphQL mutations/subscriptions and auth guards. | `packages/hoppscotch-backend/src/user-history/user-history.resolver.ts:16-57`, `121-170` |
| 10 | `packages/hoppscotch-backend/src/user-history/user-history.service.ts` | Shows Prisma persistence, JSON conversion, and pubsub. | `packages/hoppscotch-backend/src/user-history/user-history.service.ts:28-49`, `60-93` |

Before reading each file, know whether you are looking at a shell, shared UI, state store, sync adapter, API boundary, service, or database model. After reading each file, explain what it owns and what it deliberately delegates.

## Three Most Important Data Flows

1. REST request execution -> history: `Request.vue:340-441` runs the request, `history.ts:354-372` subscribes to completed responses and adds snapshots.
2. History sync -> backend persistence: `sync.ts:26-60` serializes history, `api.ts:51-64` calls GraphQL, `user-history.resolver.ts:26-57` receives it, `user-history.service.ts:60-93` writes Prisma.
3. Startup -> platform capabilities: `selfhost-web/src/main.ts:42-204` builds the platform definition, then `common/src/index.ts:24-70` injects it and initializes modules/services.

## Pre-Reading Checklist

- Can I name the package I am reading?
- Is this file runtime code, config, generated code, schema, or test?
- Does this code run in browser, desktop, backend, CLI, or shared library?
- What data type crosses this boundary?
- What package owns that data type?
- Is this code reactive, synchronous, async, or event/subscription based?
- What happens on failure?
- What auth context is assumed?
- What state is local-only versus synced?
- What test would catch a regression here?

## Architecture and Review Red Flags

- A backend mutation accepts a user ID but does not include it in the database `where`.
- A JSON string crosses a boundary without parse validation or safe error handling.
- A frontend store mutation causes a backend mutation, but subscription echo handling is missing.
- A component both renders UI and owns too much business logic.
- Generated GraphQL types are bypassed with `any`.
- Platform-specific code leaks into `hoppscotch-common`.
- Route assumptions depend on plugin-generated names without tests.
- A migration changes JSON shape without updating shared `@hoppscotch/data` schemas.
- Auth refresh logic retries forever or signs users out unexpectedly.
- A test mocks the entire behavior being tested rather than the boundary around it.

