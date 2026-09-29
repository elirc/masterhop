# System Map

## Shape: pnpm monorepo, 12 packages

Declared in [pnpm-workspace.yaml](../../../pnpm-workspace.yaml) (`packages/**`), orchestrated by recursive scripts in the root [package.json](../../../package.json#L9-L22): every package exposing `do-dev`/`do-test`/`do-lint`/`do-typecheck` participates in `pnpm dev/test/lint/typecheck`.

```
                        ┌────────────────────────────┐
                        │      hoppscotch-data       │  versioned domain schemas (zod+verzod)
                        └─────┬────────────┬─────────┘
                              │            │
        ┌─────────────────────▼──┐   ┌─────▼──────────────────┐
        │   hoppscotch-common    │   │  hoppscotch-js-sandbox │  user-script execution
        │  (Vue 3 app core: UI,  │◄──┤  (web worker / node)   │
        │  services, stores)     │   └─────┬──────────────────┘
        └──┬──────────┬──────┬───┘         │
           │          │      │             │
 ┌─────────▼───┐ ┌────▼────────────┐ ┌─────▼─────────────┐
 │ selfhost-web│ │hoppscotch-      │ │  hoppscotch-cli   │  `hopp test` for CI
 │ (web shell +│ │desktop (Tauri)  │ └───────────────────┘
 │ GQL sync)   │ │ + kernel/relay/ │
 └──────┬──────┘ │ agent (Rust/TS) │        ┌──────────────────┐
        │        └─────────────────┘        │ hoppscotch-sh-   │
        │  GraphQL over HTTP/WS             │ admin (Vue admin)│
        ▼                                   └────────┬─────────┘
 ┌─────────────────────────────────────────┐         │
 │           hoppscotch-backend            │◄────────┘
 │  NestJS + Apollo GraphQL + Prisma       │
 │  auth (REST) · teams · sync · pubsub    │
 │  mock-server (public REST) · admin API  │
 └───────────────────┬─────────────────────┘
                     ▼
               PostgreSQL
```

Plus [codemirror-lang-graphql](../../../packages/codemirror-lang-graphql) (editor language support) and [hoppscotch-kernel](../../../packages/hoppscotch-kernel) (typed contracts — `relay`, `io`, `store` — between common and native shells).

## Ownership map

| Concern | Lives in | Notes |
| --- | --- | --- |
| UI components | [hoppscotch-common/src/components](../../../packages/hoppscotch-common/src/components) | Domain-grouped: `http/`, `collections/`, `environments/`, `teams/`, `mockServer/`… |
| Client state (new) | [hoppscotch-common/src/services](../../../packages/hoppscotch-common/src/services) | `dioc` service classes (e.g., [kernel-interceptor.service.ts](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts)) |
| Client state (legacy) | [hoppscotch-common/src/newstore](../../../packages/hoppscotch-common/src/newstore) | RxJS `DispatchingStore` dispatchers — despite the name, this is the *older* generation |
| Domain models | [hoppscotch-data/src](../../../packages/hoppscotch-data/src) | `rest/`, `graphql/`, `collection/`, `environment/` — all versioned entities |
| Request execution | [hoppscotch-common/src/helpers/RequestRunner.ts](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts) + `helpers/network.ts` + kernel interceptors | |
| API (GraphQL) | [hoppscotch-backend/src/*/\*.resolver.ts](../../../packages/hoppscotch-backend/src) | Code-first Nest resolvers; schema emitted via `gen-gql` |
| API (REST) | `auth`, `mock-server`, `team-collection` controllers, `access-token` | REST where cookies/redirects/public surface demand it |
| Domain/business logic | `*.service.ts` per backend module | Services return fp-ts `Either`; resolvers translate |
| Persistence | [prisma/schema.prisma](../../../packages/hoppscotch-backend/prisma/schema.prisma) + PrismaService | Migrations in `prisma/migrations` |
| Workers/async | [pubsub](../../../packages/hoppscotch-backend/src/pubsub) (subscriptions), `@nestjs/schedule` crons, mailer | No queue system — worth knowing |
| CLI | [hoppscotch-cli/src](../../../packages/hoppscotch-cli/src) | commander-based, `commands/test.ts` is the entry |
| Tests | co-located `*.spec.ts` (backend, jest), `__tests__/` (common/cli, vitest) | CI: [tests.yml](../../../.github/workflows/tests.yml) |
| Docs | root markdown + this curriculum | `docs-codex-learning/` is a separate, earlier learning suite |

## Public interfaces vs private internals

**Public (contracts; changing them = breaking someone):** the GraphQL schema (generated SDL), REST routes under `/v1/auth`, `/mock`, access-token routes, the CLI's flags and exit codes, `@hoppscotch/data` exported types (consumed by external tooling and storage formats), the mock-server URL patterns.

**Private (refactor freely):** service internals, stores/services inside common, sandbox internals, `cast()` mappers.

The sneaky middle: **pubsub topic strings** (`team/{id}/member_added`) — technically internal, but every subscriber resolver and frontend expectation rides on them, and persisted **JSON formats** — the `request` Json column ([schema.prisma#L64](../../../packages/hoppscotch-backend/prisma/schema.prisma#L64)) is governed by `hoppscotch-data` versioning, so "private" DB data carries a public-grade compatibility burden.

Drill: pick any file you haven't opened, predict its owner row in the table above from its path alone, then open it and check. Ten reps.
