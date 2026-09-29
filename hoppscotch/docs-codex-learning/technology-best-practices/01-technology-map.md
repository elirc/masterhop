# Technology Map

## pnpm Workspaces

**Role in This Project:** Coordinates many packages under one repo.

**Where It Appears:** `package.json:8-21`, `pnpm-workspace.yaml:1-2`, package `package.json` files.

**Why It Matters:** Root scripts run package scripts recursively; understanding package ownership prevents wrong-file edits.

**What to Study First:** `package.json`, `packages/hoppscotch-common/package.json`, `packages/hoppscotch-backend/package.json`.

**Related Technologies:** Vite, Nest CLI, Prisma generation, GraphQL codegen.

**Learning Priority:** High, because it shapes every workflow.

## Vue 3 And Vite

**Role in This Project:** Main frontend UI framework and build tooling.

**Where It Appears:** `packages/hoppscotch-common/src/App.vue`, `pages/index.vue`, `components/http/Request.vue`, `modules/router.ts`.

**Why It Matters:** Most user-visible features are Vue components composed through file-based routes and shared modules.

**What to Study First:** `packages/hoppscotch-common/src/App.vue:1-52`, `packages/hoppscotch-common/src/pages/index.vue:1-220`, `packages/hoppscotch-common/src/components/http/Request.vue:1-520`.

**Related Technologies:** Vue Router, Composition API, Tailwind, Hoppscotch UI, Vite plugins.

**Learning Priority:** High.

## TypeScript

**Role in This Project:** Defines contracts across UI props, stores, GraphQL wrappers, backend services, and shared domain data.

**Where It Appears:** Nearly every `.ts` and `<script setup lang="ts">` file.

**Why It Matters:** The repo relies on typed boundaries, but you must know where runtime validation is still needed.

**What to Study First:** `packages/hoppscotch-data/src/rest/index.ts:80-115`, `packages/hoppscotch-common/src/services/workspace.service.ts:16-27`, `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts:197-270`.

**Related Technologies:** Zod, GraphQL codegen, Prisma generated client, Vue.

**Learning Priority:** High.

## Zod And Verzod

**Role in This Project:** Runtime validation and version migration for request/collection data.

**Where It Appears:** `packages/hoppscotch-data/src/rest`, `graphql`, `collection`.

**Why It Matters:** Saved/imported API request data evolves over time.

**What to Study First:** `packages/hoppscotch-data/src/rest/v/17.ts:5-18`, `packages/hoppscotch-data/src/rest/index.ts:75-115`, `packages/hoppscotch-data/src/collection/index.ts:27-74`.

**Related Technologies:** TypeScript, persistence, import/export.

**Learning Priority:** High.

## Custom Stores And RxJS

**Role in This Project:** Manages app state and streams without Redux/Zustand.

**Where It Appears:** `packages/hoppscotch-common/src/newstore`.

**Why It Matters:** Features such as history and collections use store dispatches and streams.

**What to Study First:** `DispatchingStore.ts:1-77`, `history.ts:13-372`.

**Related Technologies:** Vue composables, sync adapters, RxJS.

**Learning Priority:** High.

## GraphQL And URQL

**Role in This Project:** Main frontend-backend API for user data and sync.

**Where It Appears:** `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts`, `packages/hoppscotch-selfhost-web/src/platform/*/api.ts`, backend resolvers.

**Why It Matters:** User history, collections, teams, settings, and subscriptions cross this boundary.

**What to Study First:** `GQLClient.ts:71-118`, `history/web/api.ts:35-127`, `user-history.resolver.ts:26-170`.

**Related Technologies:** GraphQL codegen, NestJS GraphQL, subscriptions.

**Learning Priority:** High.

## NestJS

**Role in This Project:** Backend framework for modules, REST controllers, GraphQL resolvers, guards, services, and pipes.

**Where It Appears:** `packages/hoppscotch-backend/src`.

**Why It Matters:** Backend features follow Nest module/resolver/service patterns.

**What to Study First:** `main.ts:42-106`, `app.module.ts:42-138`, `user-history.resolver.ts:16-170`, `user-history.service.ts:14-233`.

**Related Technologies:** Prisma, class-validator, Passport/JWT, Apollo GraphQL.

**Learning Priority:** High.

## Prisma And PostgreSQL

**Role in This Project:** Database schema, generated client, migrations, persistence.

**Where It Appears:** `packages/hoppscotch-backend/prisma/schema.prisma`, `src/prisma/prisma.service.ts`, service files.

**Why It Matters:** User, team, collection, history, token, mock server, and published docs data persist here.

**What to Study First:** `schema.prisma:95-210`, `prisma.service.ts:14-47`, `user-history.service.ts:28-93`.

**Related Technologies:** PostgreSQL, Nest services, migrations.

**Learning Priority:** High.

## Testing

**Role in This Project:** Validates services, persistence migration, sandbox behavior, CLI request running, and backend logic.

**Where It Appears:** `__tests__`, `.spec.ts`, backend Jest config in package JSON.

**Why It Matters:** Tests show intended behavior and safe extension points.

**What to Study First:** `workspace.service.spec.ts:72-140`, `user-history.service.spec.ts:143-218`, `requestRunner.spec.ts:27-108`.

**Related Technologies:** Vitest, Jest, mocks, `jest-fp-ts`.

**Learning Priority:** High.

