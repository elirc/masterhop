# Testing Strategy

## What exists

| Layer | Tool | Where | What it covers |
| --- | --- | --- | --- |
| Backend unit | jest + `jest-mock-extended` | co-located `*.spec.ts` ([team.service.spec.ts](../../../packages/hoppscotch-backend/src/team/team.service.spec.ts)) | service logic against `mockDeep<PrismaService>()` |
| Backend e2e scaffold | jest e2e config | [test/app.e2e-spec.ts](../../../packages/hoppscotch-backend/test) | minimal; not in the main CI path (verify) |
| Frontend unit | vitest | `__tests__/` dirs in common (`pnpm --filter hoppscotch-common test`, inferred) | helpers, services |
| Data package | vitest | [hoppscotch-data/src/__tests__](../../../packages/hoppscotch-data/src) | schema migrations — the highest-value tests in the repo relative to size |
| CLI | vitest | [hoppscotch-cli/src/__tests__](../../../packages/hoppscotch-cli/src/__tests__) | command behavior |
| CI | [tests.yml](../../../.github/workflows/tests.yml) | `pnpm test` on Node 22 | all of the above; **no live database** |

The backend idiom to internalize ([team.service.spec.ts#L15-L35](../../../packages/hoppscotch-backend/src/team/team.service.spec.ts#L15-L35)): no `TestingModule` ceremony — construct the service directly with `mockDeep(prisma)` + hand-rolled collaborator mocks, `mockReset` in `beforeEach`. Fast, isolated, and it makes constructor DI earn its keep.

## What each layer should and shouldn't test

- Unit (mocked Prisma): branching, invariant checks, error mapping, pubsub publish calls. **Not**: SQL correctness, constraint behavior, transactions, locks.
- Data package: every version migration up-chain. **Not**: UI rendering of the entities.
- CLI: flag parsing, exit codes, report shapes with fixture collections. **Not**: real network calls.
- What does NOT belong anywhere: testing Prisma itself, testing NestJS wiring (a controller that only delegates needs no unit test — the e2e/smoke layer owns it).

## The strategic gap (say this carefully and it's a senior statement)

The repo's hardest-won correctness properties — row locks, retry-on-deadlock, unique-constraint interplay ([Flow 5](../01-codebase-cartography/05-key-flows.md#flow-5-creatingreordering-team-collections-persistence-flow)), cascade behavior — are exactly the properties **mocked Prisma cannot test**. A `mockDeep` suite proves the service *asks* for a transaction; it cannot prove the transaction *works*. Missing layer: integration tests against a real Postgres (testcontainers or CI service container — CI already provisions `DATABASE_URL`-style env, so the infrastructure distance is small). This is the single highest-leverage testing investment in the repo, and [senior project 2](../06-contribution-practice/03-senior-build-projects.md) builds it.

## Fixtures, isolation, and flake prevention

- Time: token expiry logic compares `new Date()` ([auth.service.ts#L299-L300](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L299-L300)) — tests must inject/freeze time (jest fake timers) rather than sleep. Any test that sleeps to "wait for" expiry is a flake seed.
- Randomness/IDs: cuid/uuid defaults come from the DB layer; with mocks you control returns — with a real DB, assert on shapes not values.
- Isolation: `mockReset(mockPrisma)` per test ([spec#L33-L35](../../../packages/hoppscotch-backend/src/team/team.service.spec.ts#L33-L35)); with a real DB, per-test truncation or per-suite schema.
- Worker/sandbox tests: the js-sandbox suites ([src/__tests__](../../../packages/hoppscotch-js-sandbox/src/__tests__)) are the model for testing async isolation — deterministic scripts, result-shape assertions.

## Interview angle

"Describe your testing strategy" — the strong answer is layer-shaped with named tradeoffs: *"Unit with deep mocks for logic breadth; schema-migration tests because persisted data outlives code; the gap I'd close is real-DB integration for concurrency invariants, because mocks can't falsify those."* That sentence, with this repo as evidence, is [Q-round material](../08-interview-prep/05-debugging-and-code-review-rounds.md).
