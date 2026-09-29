# Runtime and Tooling Map

## Package manager and orchestration

- **pnpm 10** enforced by `npx only-allow pnpm` preinstall ([package.json#L10](../../../package.json#L10)); workspace = `packages/**`.
- Root scripts fan out recursively: `pnpm dev` → `pnpm -r do-dev`, same for `test`/`lint`/`typecheck` ([package.json#L11-L21](../../../package.json#L11-L21)). Each package opts in by defining the `do-*` alias — a lightweight task-runner convention (no turbo/nx).
- `pnpm.overrides` pins vulnerable/incompatible transitive deps ([package.json#L36-L52](../../../package.json#L36-L52)); `onlyBuiltDependencies` allow-lists which deps may run install scripts (L53-L68) — supply-chain control worth naming in interviews.

## Build and dev servers

| Package | Dev | Build | Notes |
| --- | --- | --- | --- |
| common / selfhost-web / sh-admin | Vite + parallel `graphql-codegen --watch` ([common package.json](../../../packages/hoppscotch-common/package.json#L6-L10)) | Vite (`--max_old_space_size=8192` for selfhost-web — the bundle is big) | Vue 3 + TS |
| backend | `nest start --watch` | `nest build` + mailer template copy | Express 5 platform |
| data / kernel / js-sandbox | library builds | tsup/vite-style lib output | consumed via workspace links |
| cli | tsc/bundler build; binary `hopp` | | |
| desktop / agent | Tauri (Rust toolchain required) | separate CI workflows | out of scope for this curriculum |

## Codegen — the hidden coupling

Two generators keep frontend and backend contracts in sync; know them or suffer:

1. **Backend GraphQL SDL emit**: `pnpm gen-gql` boots Nest with `GENERATE_GQL_SCHEMA=true` and writes `gql-gen/backend-schema.gql` ([root package.json#L13](../../../package.json#L13), [backend package.json generate-gql-sdl](../../../packages/hoppscotch-backend/package.json)). Code-first: resolvers/decorators are the source of truth.
2. **Frontend operation types**: `graphql-codegen` in common/selfhost-web/sh-admin consumes that schema and generates typed operations at `postinstall` and in dev watch. A backend schema change ripples to frontend compile errors — that's the *point*.

Also: `prisma generate` (client into `src/generated/prisma` — note the non-default output path, [schema.prisma#L1-L4](../../../packages/hoppscotch-backend/prisma/schema.prisma#L1-L4)) runs on backend postinstall with a placeholder `DATABASE_URL`.

## Runtime boundaries

| Boundary | Runtime | Constraints to respect |
| --- | --- | --- |
| Browser app | Vue 3 SPA | CORS limits direct sends → interceptor system; storage = localStorage + backend sync |
| Web worker | sandbox scripts | no DOM, structured-clone messaging only ([Flow 2](05-key-flows.md#flow-2-running-a-user-test-script-sandboxasync-flow)) |
| Node server | NestJS/Express 5 | libuv threadpool for argon2/bcrypt; in-memory pubsub = single-instance assumption |
| Node CLI | commander | exit code is the contract; node sandbox variant |
| Tauri/native | Rust + webview | kernel contracts ([hoppscotch-kernel](../../../packages/hoppscotch-kernel/src)) mediate |

## Environment variables (high level, no secrets)

- Frontend build-time config uses `VITE_*` vars (base URLs, allowed auth providers) — read via `import.meta.env` and, backend-side, mirrored through ConfigService as `INFRA.*` / `VITE_*` keys (e.g., [auth.service.ts#L105](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L105)).
- Backend requires `DATABASE_URL`, `DATA_ENCRYPTION_KEY` (CI shows the minimum viable set — [tests.yml](../../../.github/workflows/tests.yml)), JWT/token validity settings, mailer config.
- Many settings live in the **InfraConfig DB table** rather than env ([schema.prisma#L215-L223](../../../packages/hoppscotch-backend/prisma/schema.prisma#L215-L223)) so admins can change them at runtime; `lastSyncedEnvFileValue` tracks drift between file and DB — read [infra-config](../../../packages/hoppscotch-backend/src/infra-config) before touching config behavior.
- `.env.example` at root is the canonical inventory; CI literally renames it to `.env`.

## Lint / format / hooks

husky + lint-staged + commitlint (conventional commits) at the root ([package.json#L24-L33](../../../package.json#L24-L33), [commitlint.config.js](../../../commitlint.config.js)); per-package eslint with a `HOPP_LINT_FOR_PROD` strict mode ([common package.json#L13](../../../packages/hoppscotch-common/package.json#L13)).

Interview angle: "Walk me through what happens from `git push` to deploy" — for this repo: commitlint gate → CI `pnpm test` on Node 22 → Docker workflows ([release-push-docker.yml](../../../.github/workflows/release-push-docker.yml)) → self-hosters run compose/AIO. Noticing that **typecheck and lint are not in the tests.yml gate** (only `pnpm test`) is a legitimate observation — verify before asserting in a PR, then propose it.

Drill: run `pnpm -r --list` (or read each package.json) and build your own table of which packages define `do-test`. That's the real test surface CI covers.
