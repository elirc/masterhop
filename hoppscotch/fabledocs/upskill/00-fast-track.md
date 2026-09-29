# Fast Track — One Weekend in Hoppscotch

Goal: by Sunday night you can run the app, trace two end-to-end flows out loud, and have made one safe change with a passing test.

## 1. Install and run (Saturday morning)

All commands **inferred** from [package.json](../../package.json#L9-L22) and [tests.yml](../../.github/workflows/tests.yml) — not executed while writing this doc. Node 22 is what CI pins (Node 24 has known test failures per the workflow comment).

```bash
pnpm install          # postinstall runs prisma generate + GraphQL codegen; needs a DATABASE_URL, see below
pnpm dev              # runs `do-dev` in every package that defines it
pnpm test             # all package test suites (backend: jest, common: vitest)
pnpm typecheck        # vue-tsc / tsc across packages
```

Notes that save you an hour:

- The backend `postinstall` ([hoppscotch-backend/package.json](../../packages/hoppscotch-backend/package.json)) runs `prisma generate` against a placeholder `DATABASE_URL` — you don't need a live database just to install. You do need Postgres to actually run the backend; `docker-compose.yml` at the root defines the full self-host stack.
- CI ([tests.yml](../../.github/workflows/tests.yml)) does exactly: `mv .env.example .env`, `pnpm install`, `pnpm test`. If you can only afford one runtime verification this weekend, replicate that.
- Frontend-only exploration works without any backend: the web app degrades to local-storage persistence when logged out.

## 2. The first 10 files to open, in order

| # | File | Why |
| --- | --- | --- |
| 1 | [package.json](../../package.json#L9-L22) | Workspace scripts — how everything is orchestrated via `pnpm -r do-*` |
| 2 | [pnpm-workspace.yaml](../../pnpm-workspace.yaml) | The monorepo is just `packages/**` |
| 3 | [packages/hoppscotch-data/src/rest/index.ts](../../packages/hoppscotch-data/src/rest/index.ts#L80-L113) | The core domain object (`HoppRESTRequest`), versioned 0→17 |
| 4 | [packages/hoppscotch-common/src/helpers/RequestRunner.ts](../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L490-L605) | The heart of the product: how a request actually runs |
| 5 | [packages/hoppscotch-common/src/helpers/network.ts](../../packages/hoppscotch-common/src/helpers/network.ts#L15-L80) | Request → interceptor → response stream |
| 6 | [packages/hoppscotch-common/src/services/kernel-interceptor.service.ts](../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L56-L69) | The interceptor abstraction (browser/extension/proxy/native) |
| 7 | [packages/hoppscotch-backend/prisma/schema.prisma](../../packages/hoppscotch-backend/prisma/schema.prisma#L10-L73) | Teams, members, collections, requests — the whole data model in one file |
| 8 | [packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts](../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L21-L43) | How every team-scoped API call is authorized |
| 9 | [packages/hoppscotch-backend/src/auth/auth.controller.ts](../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L51-L100) | Magic-link + SSO + refresh-token endpoints |
| 10 | [packages/hoppscotch-backend/src/team-collection/team-collection.service.ts](../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L452-L517) | A production-grade write path: transaction, row lock, ordering invariant |

## 3. Two flows to trace (Saturday afternoon)

Do these with the code open, writing each step down. Full trace tables live in [05-key-flows.md](01-codebase-cartography/05-key-flows.md).

**Flow A — pressing "Send":** start at `RequestRunner.ts` [L520-L585](../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L520-L585). Pre-request script runs in a sandbox → environment variables are substituted into an "effective request" → `createRESTNetworkRequestStream` hands it to the active kernel interceptor → the response comes back on an RxJS stream → the test script runs against it → env diffs and cookies are written back.

**Flow B — creating a team collection:** GraphQL mutation → [team-collection.service.ts createCollection](../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L452-L517). Title validated → parent ownership checked → transaction opens → sibling rows locked → `orderIndex = last + 1` → created → event published to subscribers.

Pause-and-predict while tracing: *before* reading `createCollection`'s transaction, ask why a simple `count(siblings) + 1` outside a transaction would be wrong. (Answer: two concurrent creates would compute the same index and violate [`@@unique([teamID, parentID, orderIndex])`](../../packages/hoppscotch-backend/prisma/schema.prisma#L56).)

## 4. One small safe change (Sunday)

Pick one — both are docs-of-behavior changes with tiny blast radius:

1. In [team.service.ts](../../packages/hoppscotch-backend/src/team/team.service.ts#L205-L240), write a new jest test in `team.service.spec.ts` asserting `leaveTeam` returns `TEAM_ONLY_ONE_OWNER` when the last owner tries to leave. Model it on the existing `mockDeep<PrismaService>()` style at [team.service.spec.ts#L15-L35](../../packages/hoppscotch-backend/src/team/team.service.spec.ts#L15-L35). Run: `cd packages/hoppscotch-backend && pnpm test -- team.service` (**inferred**).
2. Add one missing JSDoc `@returns` description to a public method in `team-collection.service.ts` and confirm `pnpm lint` passes.

## 5. Teach-back (Sunday night)

Explain Flow A aloud in under 3 minutes as if an interviewer asked: *"Walk me through how an HTTP client app you've studied sends a request."* You must name: where user scripts run and why they're sandboxed, what an "effective request" is, why interceptors are pluggable, and where results/env changes land. Record yourself. If you say "and then it just sends it," you skipped the interceptor boundary — do it again.

## What the fast path skips

Real-time GraphQL subscriptions, the mock-server security boundary, the desktop/Tauri kernel stack, the CLI, schema migrations, and all of interview prep. That's the rest of the curriculum.
