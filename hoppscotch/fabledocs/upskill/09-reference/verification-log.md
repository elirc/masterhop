# Verification Log

Running log of what was inspected while authoring this curriculum. Anchors in the curriculum were confirmed against these reads. Date: 2026-07-09.

> **Status note (2026-10-06, frozen record).** The tables below record the 2026-07-09 pass and are kept as written. A static re-check on 2026-10-06 (no installs, servers or tests) found: every line count in the "Files read" table still matches (schema 342, `auth.service.ts` 392, `auth.controller.ts` 230, guard 44, `pubsub.service.ts` 27, mock-server controller 194, `network.ts` 80, `kernel-interceptor.service.ts` 173, CLI `test.ts` 114); root `package.json` is still version 3.0.1 on `pnpm@10.33.2`; suspicions 2 and 4 below were re-read and still hold (provider account created before the expiry check; `PubSubService` only ever constructs the local `graphql-subscriptions` PubSub). Five links in `07-career-and-collaboration/` that pointed at `06-contribution-practice/` files without the directory prefix were repaired. The "no own `.git`" row is superseded: the clone now has its own repository (a single snapshot commit, so still no upstream history).

## Method

- Static reading only. **No commands that mutate state were run.** No dev server, DB, or test suite was executed (Windows host, no local Postgres provisioned for this repo). All run/test commands in the curriculum are marked **inferred** from `package.json` scripts and CI config unless stated otherwise.
- Line anchors were re-read immediately before writing.

## Files read (full or targeted excerpts)

| File | What was verified |
| --- | --- |
| `package.json` (root) | Workspace scripts (`pnpm dev/test/typecheck/lint`), pnpm 10, overrides, version 3.0.1 |
| `pnpm-workspace.yaml` | `packages/**` workspace layout |
| `.github/workflows/tests.yml` | CI runs `pnpm test` on Node 22 (pinned due to Node 24 test failures), sets `DATABASE_URL` + `DATA_ENCRYPTION_KEY` env |
| `packages/hoppscotch-backend/prisma/schema.prisma` | Full 342-line schema: all models, `@@unique` constraints, cascade rules, enums |
| `packages/hoppscotch-backend/src/auth/auth.service.ts` | Full 392 lines: magic-link flow, token generation/rotation, argon2/bcrypt usage, fp-ts Either/Option |
| `packages/hoppscotch-backend/src/auth/auth.controller.ts` | Full 230 lines: all auth routes, guards, cookie handling, throttling |
| `packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts` | Full 44 lines: RBAC guard implementation |
| `packages/hoppscotch-backend/src/team/team.resolver.ts` | L150–373: queries, mutations, role decorators, subscriptions |
| `packages/hoppscotch-backend/src/team/team.service.ts` | L150–249: `updateTeamAccessRole`, `leaveTeam`, `createTeam`, single-owner invariant |
| `packages/hoppscotch-backend/src/team-collection/team-collection.service.ts` | Method inventory (grep) + L440–660 read: `createCollection` transaction/locking, delete + sibling reindex with retry/backoff |
| `packages/hoppscotch-backend/src/pubsub/pubsub.service.ts` | Full 27 lines: in-memory `graphql-subscriptions` PubSub |
| `packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts` | Full 194 lines: catch-all mock route, security header blocklist, content-type downgrade |
| `packages/hoppscotch-backend/src/team/team.service.spec.ts` | L1–80: jest + `jest-mock-extended` `mockDeep<PrismaService>()` test style |
| `packages/hoppscotch-common/src/helpers/network.ts` | Full 80 lines: `createRESTNetworkRequestStream`, BehaviorSubject response stream |
| `packages/hoppscotch-common/src/services/kernel-interceptor.service.ts` | Full 173 lines: dioc service, interceptor registry/selection/execute |
| `packages/hoppscotch-common/src/helpers/RequestRunner.ts` | L490–689 read + grep of script entry points: pre-request → effective request → stream → post-request script pipeline |
| `packages/hoppscotch-data/src/rest/index.ts` | L1–150: verzod `createVersionedEntity`, versions 0–17, `getVersion` discriminator |
| `packages/hoppscotch-js-sandbox/src/web/test-runner/index.ts` | L1–60: web worker + faraday-cage execution paths |
| `packages/hoppscotch-cli/src/commands/test.ts` | Full 114 lines: `hopp test` command flow, iteration data CSV parsing |
| `packages/hoppscotch-selfhost-web/src/platform/collections/web/gqlCollections.sync.ts` | L1–70: store→backend sync, mappers, JSON transform |
| `packages/hoppscotch-common/package.json`, `hoppscotch-selfhost-web/package.json`, `hoppscotch-backend/package.json` | Test/dev/codegen scripts; vitest (common), jest (backend) |
| Directory listings | `hoppscotch-backend/src` modules, `hoppscotch-common/src` (components/services/newstore/helpers), `hoppscotch-js-sandbox/src` (node vs web), `hoppscotch-kernel/src`, mock-server module files, auth guards/strategies |

## Commands run

| Command | Result |
| --- | --- |
| `ls` / `wc -l` / `grep -n` over the files above | Structure and line counts as recorded |
| `git rev-parse --show-toplevel` | Repo is nested inside a home-directory git repo (`C:/Users/Owner`); the hoppscotch clone itself carries no own `.git` — treat git history as unavailable |

## Explicitly NOT verified (labeled inferred wherever used)

- Runtime behavior: no server started, no request sent, no test suite executed.
- The desktop/agent/relay (Rust/Tauri) code paths — only directory shape inspected.
- `hoppscotch-sh-admin` internals beyond directory existence.
- Prisma migrations folder contents (only `schema.prisma` read).
- Whether `pnpm install` succeeds on this machine (postinstall requires a `DATABASE_URL` placeholder and runs codegen).
- EffectiveURL.ts was inspected via targeted grep (template parsing call sites at L226–L330), not a full read.

## Uncertainties / suspicions carried into the curriculum (labeled "investigate", not bugs)

1. `leaveTeam` / `updateTeamAccessRole` count-then-write without a transaction — possible TOCTOU on the single-owner invariant (`team.service.ts#L205-L240`).
2. `verifyMagicLinkTokens` creates the provider account *before* checking token expiry (`auth.service.ts#L281-L304`) — ordering smell.
3. `refreshAuthTokens(hashedRefreshToken, ...)` parameter actually receives the raw cookie token (it is the *stored* token that is hashed) — misleading name (`auth.service.ts#L335-L352`).
4. `PubSubService` is in-memory only — subscriptions cannot fan out across multiple backend instances (`pubsub.service.ts#L14-L18`, comment at L5-L8 says Redis "for production" but none is wired).
5. `UserHistory` has no visible retention/pruning (schema only; service not fully read).
