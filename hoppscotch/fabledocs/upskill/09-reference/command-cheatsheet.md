# Command Cheatsheet

All commands **inferred** from package manifests and CI config unless marked verified; none were executed while authoring (see [verification-log.md](verification-log.md)). Run from repo root unless noted. Node 22 + pnpm 10 ([tests.yml](../../../.github/workflows/tests.yml), [package.json#L8](../../../package.json#L8)).

## Install / bootstrap

| Command | Does | Status |
| --- | --- | --- |
| `pnpm install` | install + postinstall codegen (prisma generate with placeholder DB URL, GraphQL SDL + client types) | inferred |
| `mv .env.example .env` (or copy on Windows) | minimum env; CI does exactly this | inferred from CI |
| `docker compose up` | full self-host stack (Postgres, backend, apps) per [docker-compose.yml](../../../docker-compose.yml) | inferred |

## Dev / build

| Command | Does | Status |
| --- | --- | --- |
| `pnpm dev` | `pnpm -r do-dev` — all packages' dev servers (Vite apps run codegen watchers in parallel) | inferred |
| `pnpm --filter hoppscotch-backend start:dev` | backend only, watch mode | inferred |
| `pnpm --filter hoppscotch-selfhost-web dev` | web app only | inferred |
| `pnpm generate` | production builds (`do-build-prod`) | inferred |
| `pnpm start` | serve built selfhost-web on :3000 via http-server | inferred |

## Test / quality

| Command | Does | Status |
| --- | --- | --- |
| `pnpm test` | all suites (backend jest, others vitest) — the CI gate | inferred |
| `cd packages/hoppscotch-backend && pnpm test -- team.service` | one backend spec by pattern | inferred |
| `pnpm --filter hoppscotch-backend test:watch` / `test:cov` / `test:debug` | watch / coverage / inspector (`--runInBand`) | inferred |
| `pnpm --filter hoppscotch-common test` | common vitest (`vitest --run`) | inferred |
| `pnpm typecheck` | `do-typecheck` everywhere (vue-tsc for apps) | inferred |
| `pnpm lint` / `pnpm lintfix` | prod-mode eslint / autofix | inferred |
| `pnpm pre-commit` | lint + typecheck (what husky runs) | inferred |

## Codegen / schema

| Command | Does | Status |
| --- | --- | --- |
| `pnpm gen-gql` | boot Nest, emit GraphQL SDL to `gql-gen/backend-schema.gql` | inferred |
| `pnpm --filter hoppscotch-common gql-codegen` | regenerate frontend operation types | inferred |
| `cd packages/hoppscotch-backend && npx prisma generate` | regen Prisma client (needs `DATABASE_URL`, placeholder ok) | inferred |
| `npx prisma migrate dev --name <name>` | create+apply a migration (real DB required) | inferred |
| `DEBUG=prisma:query pnpm --filter hoppscotch-backend start:dev` | log every SQL query — the N+1 probe | inferred (standard Prisma) |

## CLI package

| Command | Does | Status |
| --- | --- | --- |
| `hopp test <collection.json>` | run a collection's tests, exit non-zero on failure | inferred from [test.ts](../../../packages/hoppscotch-cli/src/commands/test.ts) |
| `hopp test <file> --env <envs.json> --reporter-junit [path]` | with env + JUnit output for CI | inferred |
| `hopp test <file> --iteration-data data.csv --iteration-count 5 --delay 100` | data-driven runs | inferred |
| `--legacy-sandbox` | old node sandbox implementation | inferred |

## Gotchas (from static reading)

- Backend postinstall *requires* some `DATABASE_URL` even placeholder — a bare env will fail install at prisma generate.
- Generated dirs (`src/generated/prisma`, frontend gql types) are build artifacts: when types look insane, regenerate before debugging.
- CI pins Node 22 because of known Node 24 test failures — match it locally.
- The `nul` file and home-dir git situation on this Windows clone are environment noise, not repo features.
