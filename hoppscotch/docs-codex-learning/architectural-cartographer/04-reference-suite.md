# Reference Suite

## Doc 1: Junior Onboarding Guide

Start with the root scripts in `package.json:9-18`, then identify the package you are working in. For the main app, read `packages/hoppscotch-selfhost-web/src/main.ts:42-204`, `packages/hoppscotch-common/src/index.ts:24-70`, and `packages/hoppscotch-common/src/modules/router.ts:12-111`.

Day-one rule: do not chase every package. Pick one user flow. For this repo, use REST history because it crosses UI, state, sync, GraphQL, service, and database.

Setup is reconstructed from scripts: `pnpm install`, copy `.env.example` to `.env`, start dependencies with Docker Compose if needed, run codegen/build/dev scripts. Treat this as an assumption until verified locally.

## Doc 2: Mid-Level Architecture Guide

System design:

- Shell package chooses platform capabilities.
- Common package owns shared UI and client behavior.
- Data package owns versioned request/collection schemas.
- Self-host package owns sync adapters to backend APIs.
- Backend package owns auth, GraphQL, REST, services, and Prisma.

Key flow: `Request.vue:340-441` -> `history.ts:354-372` -> `sync.ts:26-60` -> `api.ts:51-91` -> `user-history.resolver.ts:26-170` -> `user-history.service.ts:60-93` -> `schema.prisma:152-161`.

## Doc 3: Senior Ownership Guide

Technical debt register:

- JSON string parse boundary in history service.
- Ownership enforcement in history update/delete service methods.
- Setup knowledge spread across many files.
- Potential missing index for history reads.
- GraphQL caching strategy likely conservative.

Upgrade path: secure backend predicates first, add tests, document setup, then optimize query/cache behavior from measured usage.

## Doc 4: Code Review Guide

Review order:

1. Does the change touch a domain schema in `packages/hoppscotch-data`?
2. Does it affect platform-specific code or common code?
3. Does it cross GraphQL? Check generated types and API wrappers.
4. Does backend code enforce auth/ownership at guard and service/database levels?
5. Are local store dispatches synced? Check subscription echo behavior.
6. Are tests added near the smallest reliable boundary?

Example: a history feature must consider `history.ts`, `history/web/sync.ts`, generated GraphQL documents, `user-history.resolver.ts`, `user-history.service.ts`, and `schema.prisma`.

## Doc 5: Debugging Guide

For a REST request that does not appear in history:

1. Check the UI sends: `Request.vue:340-441`.
2. Check completed responses emit into history: `history.ts:354-372`.
3. Check local store state through `restHistory$`: `history.ts:267-268`.
4. Check sync enabled and mutation call: `sync.ts:26-46`.
5. Check GraphQL client auth headers: `GQLClient.ts:71-118`.
6. Check backend resolver guard and args: `user-history.resolver.ts:26-57`.
7. Check Prisma create: `user-history.service.ts:69-77`.

Likely logs/errors: browser console GraphQL errors, network `/graphql` response, Nest morgan logs from `main.ts:87`, Prisma connection failures from `prisma.service.ts:130-139`.

## Doc 6: Change Playbook

To add a feature safely:

1. Identify owning package.
2. Find the closest existing pattern.
3. Update shared data schemas first if persisted/imported shape changes.
4. Update UI component and store.
5. Update sync/API wrapper.
6. Update backend resolver/service/model if needed.
7. Add focused tests.
8. Run package-level test/typecheck before root-wide commands.

For removal, reverse the flow and search with `rg` for generated GraphQL documents, route pages, store actions, and Prisma fields.

## Doc 7: Interview Walkthrough

Concise explanation:

"Hoppscotch is a pnpm monorepo. The shared Vue app lives in `hoppscotch-common`, platform shells inject web or desktop capabilities, the backend is NestJS with GraphQL and REST endpoints, and Prisma persists user/team/request data in PostgreSQL. A REST request starts in a Vue tab, runs through a request runner and interceptor strategy, records completed responses into a local history store, syncs through URQL GraphQL mutations, and persists as JSON in Prisma."

Trade-off answer:

"Storing request snapshots as JSON gives flexibility for evolving API-client shapes, and the `@hoppscotch/data` package mitigates that with versioned schemas. The risk is weaker database-level validation and harder querying, so backend parse validation and indexes matter."

