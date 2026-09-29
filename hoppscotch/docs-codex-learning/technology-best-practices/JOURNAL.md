# Technology Journal

## Technologies Identified

- pnpm workspaces and monorepo scripts.
- Vue 3 single-file components, Composition API, Vue Router, Vite.
- TypeScript across frontend, backend, data packages, and CLI.
- Zod and `verzod` for versioned request/collection schemas.
- RxJS streams and a custom `DispatchingStore`.
- URQL GraphQL client with generated typed documents.
- NestJS backend with GraphQL, REST controllers, guards, validation pipes, throttling, and modules.
- Prisma 7 with PostgreSQL and migrations.
- `fp-ts` `Either`/`Option` for explicit success/failure paths.
- Vitest for common package tests and Jest for backend/CLI tests.
- Tailwind and Hoppscotch UI components for styling.
- Tauri/Rust/desktop support and kernel abstractions, lower priority for this learning pass.

## Priorities

High priority: Vue, TypeScript, state, GraphQL, NestJS, Prisma, validation, tests.

Medium priority: platform injection, kernel/interceptors, CLI runner, sandbox.

Low priority for first study: Rust relay internals and desktop Tauri plugin code, unless your goal becomes desktop systems work.

## Strong Patterns

- Versioned request schemas in `packages/hoppscotch-data/src/rest/index.ts:80-115`.
- Platform injection in `packages/hoppscotch-selfhost-web/src/main.ts:42-204`.
- Typed store dispatchers in `packages/hoppscotch-common/src/newstore/DispatchingStore.ts:16-32`.
- GraphQL client auth handling in `packages/hoppscotch-common/src/helpers/backend/GQLClient.ts:71-118`.
- Backend module composition in `packages/hoppscotch-backend/src/app.module.ts:42-138`.

## Weak Or Risky Patterns

- JSON parsing in `packages/hoppscotch-backend/src/user-history/user-history.service.ts:69-77` needs safer error handling.
- History update/delete should enforce `uid` in service/database queries in `user-history.service.ts:101-162`.
- Setup knowledge is distributed across root/package scripts and `.env.example`.
- Some UI components carry a lot of behavior, such as `Request.vue:238-520`.

## What Developers Might Misunderstand

- This is not a React/Redux app; Vue and custom stores do the main work.
- TypeScript is strongest where backed by schemas or generated GraphQL types.
- `hoppscotch-common` is shared code; do not import self-host implementation details into it.
- GraphQL models returning strings can still contain JSON that needs runtime validation.

