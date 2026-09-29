# File Reading Order

28 files. Read in order within your track; each row says why it matters, what to look for, what to skip. Don't read linearly top-to-bottom of each file — read the parts named.

## Junior path (files 1–12): "can I locate and explain?"

| # | File | Why / look for | Ignore |
| --- | --- | --- | --- |
| 1 | [package.json](../../../package.json) | `do-*` orchestration, pnpm overrides | dep versions |
| 2 | [pnpm-workspace.yaml](../../../pnpm-workspace.yaml) | one line — the whole monorepo | — |
| 3 | [hoppscotch-data/src/rest/index.ts](../../../packages/hoppscotch-data/src/rest/index.ts#L80-L115) | the core entity + 18 versions | the export list |
| 4 | [hoppscotch-data/src/rest/v/17](../../../packages/hoppscotch-data/src/rest/v) | what one version module looks like | older versions |
| 5 | [prisma/schema.prisma](../../../packages/hoppscotch-backend/prisma/schema.prisma) | every model; read `@@unique` lines twice | — |
| 6 | [app.module.ts](../../../packages/hoppscotch-backend/src/app.module.ts) | how Nest wires modules together | config details |
| 7 | [team.resolver.ts](../../../packages/hoppscotch-backend/src/team/team.resolver.ts) | resolver = guards + args + service call, nothing else | arg decorators' descriptions |
| 8 | [team.service.ts#L150-L249](../../../packages/hoppscotch-backend/src/team/team.service.ts#L150-L249) | Either returns, invariants, pubsub | pagination helpers |
| 9 | [gql-team-member.guard.ts](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts) | 44 lines of authorization | — |
| 10 | [RequestRunner.ts#L490-L690](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L490-L690) | the product's main function | env-capture helpers |
| 11 | [network.ts](../../../packages/hoppscotch-common/src/helpers/network.ts) | 80 lines, stream pattern | — |
| 12 | [components/http](../../../packages/hoppscotch-common/src/components/http) — pick the URL-bar component | how a Vue component triggers the runner | styling |

## Mid path (files 13–22): "can I change it safely across layers?"

| # | File | Why / look for | Ignore |
| --- | --- | --- | --- |
| 13 | [team-collection.service.ts#L440-L660](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L440-L660) | transactions, locks, retries — the repo's best persistence code | search/tree helpers (first pass) |
| 14 | [auth.controller.ts](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts) | route inventory, guard usage, throttling | SSO duplication |
| 15 | [auth.service.ts](../../../packages/hoppscotch-backend/src/auth/auth.service.ts) | magic-link + rotation, full read | — |
| 16 | [kernel-interceptor.service.ts](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts) | the strategy registry + dioc service shape | i18n typing noise |
| 17 | [js-sandbox/src/web/test-runner/index.ts](../../../packages/hoppscotch-js-sandbox/src/web/test-runner/index.ts) | worker + cage execution | — |
| 18 | [pubsub.service.ts](../../../packages/hoppscotch-backend/src/pubsub/pubsub.service.ts) + [topicsDefs.ts](../../../packages/hoppscotch-backend/src/pubsub/topicsDefs.ts) | typed topics; in-memory limits | — |
| 19 | [mock-server.controller.ts](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts) | the security boundary, full read | — |
| 20 | [gqlCollections.sync.ts](../../../packages/hoppscotch-selfhost-web/src/platform/collections/web/gqlCollections.sync.ts) | offline↔server sync + mappers | GraphQL op details |
| 21 | [team.service.spec.ts](../../../packages/hoppscotch-backend/src/team/team.service.spec.ts) | the testing idiom (mockDeep) | every case |
| 22 | [cli/src/commands/test.ts](../../../packages/hoppscotch-cli/src/commands/test.ts) | CLI as thin shell over shared packages | CSV details |

## Senior path (files 23–28): "what does this commit us to?"

| # | File | Why / look for |
| --- | --- | --- |
| 23 | [newstore/](../../../packages/hoppscotch-common/src/newstore) any dispatcher (e.g., `collections.ts`) vs a [dioc service](../../../packages/hoppscotch-common/src/services) | two state generations coexisting — migration strategy, or lack of one |
| 24 | [hoppscotch-kernel/src](../../../packages/hoppscotch-kernel/src) (`relay`, `io`, `store` types) | the contract that keeps web/desktop shells honest |
| 25 | [selfhost-web/src/platform](../../../packages/hoppscotch-selfhost-web/src/platform) directory shape | platform injection — what varies per shell |
| 26 | [prisma/migrations](../../../packages/hoppscotch-backend/prisma/migrations) — skim names chronologically | schema evolution story; find the mock-server and published-docs arrivals |
| 27 | [.github/workflows/tests.yml](../../../.github/workflows/tests.yml) | what CI actually guarantees (spoiler: `pnpm test` only — no e2e, no typecheck gate; verify lint claims yourself) |
| 28 | [docker-compose.yml](../../../docker-compose.yml) + [aio_run.mjs](../../../aio_run.mjs) | deploy topology; where the single-instance pubsub assumption bites |

Habit to build: for every file, write one line — "this file owns X and must never know about Y." If you can't fill in Y, you haven't found the boundary yet.
