# Good First Tickets

16 junior-scoped tickets, spread across backend, frontend, data, sandbox, and CLI. None ships a product decision; all follow an existing pattern. Do the **Read these anchors first** step every time — that habit is the actual skill.

Conventions: every ticket ends with *Interview story potential* — the STAR story finishing it would produce (see [08-interview-prep/06-behavioral-star-stories.md](../08-interview-prep/06-behavioral-star-stories.md)).

---

## Ticket 1: Test the single-owner invariant in `leaveTeam`
Difficulty: Easy — 2–3h. Skills: jest, mocked Prisma, invariants.
Story: As a maintainer, I want the last-owner rule regression-tested so refactors can't drop it.
Why good: pure test addition; zero runtime blast radius; the invariant is real ([team.service.ts#L205-L221](../../../packages/hoppscotch-backend/src/team/team.service.ts#L205-L221)).
Acceptance: ☐ test: last OWNER leaving → `TEAM_ONLY_ONE_OWNER` ☐ test: OWNER leaves when 2 owners exist → success ☐ suite green.
Anchors first: [team.service.spec.ts#L15-L35](../../../packages/hoppscotch-backend/src/team/team.service.spec.ts#L15-L35) (mock style), the service method.
Files touched: `team.service.spec.ts`.
What could go wrong: mocking `teamMember.count` but forgetting `getTeamMember` — read what the method calls, in order.
Review questions: does your test assert the *error code constant*, not a string literal?
Interview story potential: "I found an unprotected business invariant and locked it in with tests."

## Ticket 2: Type the subscribe side of PubSub
Difficulty: Medium — 3–5h. Skills: TS generics, indexed access types.
Story: As a maintainer, I want `asyncIterator` payloads type-checked against `TopicDef` like `publish` already is.
Why good: compile-time-only change; drift between publisher and subscriber payloads is a real bug class.
Anchors first: [pubsub.service.ts#L20-L26](../../../packages/hoppscotch-backend/src/pubsub/pubsub.service.ts#L20-L26), [topicsDefs.ts](../../../packages/hoppscotch-backend/src/pubsub/topicsDefs.ts), one consumer ([team.resolver.ts#L317-L326](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L317-L326)).
Files touched: `pubsub.service.ts`, possibly resolver call sites if inference needs help.
Plan: template-literal topic types → `asyncIterator<T extends keyof TopicDef>(topic: T): AsyncIterator<TopicDef[T]>`; dynamic `team/${id}/...` strings will need the TopicDef keys to be template-literal patterns.
What could go wrong: over-constraining breaks resolvers that build topic strings dynamically — typecheck the whole backend (`pnpm typecheck`, inferred).
Rejection risk: maintainers may not want type churn across many resolvers; keep the diff minimal.
Interview story potential: "I extended a half-typed contract to both sides of an event bus."

## Ticket 3: Reorder the expiry check in magic-link verification
Difficulty: Easy–Medium — 2–4h (mostly discussion). Skills: auth reasoning, safe refactor.
Story: As a maintainer, I want token expiry checked *before* any side effect, so expired links cause zero writes.
Why good: [auth.service.ts#L281-L304](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L281-L304) creates the provider account (L290-L297) before checking `expiresOn` (L299-L304). Moving the check earlier is a pure ordering fix. **File as a question first** — there may be a reason; that's contribution realism.
Acceptance: ☐ expiry checked immediately after token lookup ☐ existing tests pass ☐ new test: expired token → `MAGIC_LINK_EXPIRED` and **no** account row created.
What could go wrong: behavioral change for users whose account was being created by an expired-link flow (unlikely, but state it in the PR).
Interview story potential: "I spotted a side-effect-before-validation ordering issue in an auth flow and fixed it with tests."

## Ticket 4: Rename the misleading `hashedRefreshToken` parameter
Difficulty: Easy — 1h. Skills: careful reading, naming.
Story: As a reader, I want parameter names to tell the truth: [auth.service.ts#L335-L347](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L335-L347) receives the *raw* cookie token; the *stored* value is the hash.
Acceptance: ☐ rename to `refreshTokenFromCookie` (or similar) ☐ update JSDoc ☐ no behavior change.
Rejection risk: pure-rename PRs can be seen as noise — bundle with Ticket 3 or a test.
Interview story potential: small, but feeds the "naming as correctness documentation" review talking point.

## Ticket 5: Add a timeout to web sandbox script execution
Difficulty: Medium — 4–8h. Skills: workers, promises, UX of failure.
Story: As a user, an accidental `while(true)` in my test script should fail after N seconds, not hang the run.
Why good: real gap (see [Flow 2](../01-codebase-cartography/05-key-flows.md#flow-2-running-a-user-test-script-sandboxasync-flow)); worker-per-run design makes `terminate()` safe.
Anchors first: [web/test-runner/index.ts#L21-L41](../../../packages/hoppscotch-js-sandbox/src/web/test-runner/index.ts#L21-L41).
Plan: `Promise.race([workerResult, deadline])`; on deadline → `worker.terminate()` → `E.left("Script execution timed out")`.
Illustrative fake code shape:
```ts
// Illustrative fake code: not from this repo
const deadline = new Promise<never>((_, rej) =>
  setTimeout(() => rej(new Error("timeout")), SCRIPT_TIMEOUT_MS))
```
What could go wrong: legitimate slow scripts (crypto in pre-request); make the limit generous and/or configurable.
Rejection risk: maintainers will ask for the desktop/node parity story — scope the PR to web and say so.
Interview story potential: "I added a kill-switch for runaway user scripts in a sandboxed execution engine."

## Ticket 6: JUnit reporter docs for `hopp test`
Difficulty: Easy — 2h. Skills: CLI, docs.
Story: As a CI user, I want `--reporter-junit` documented with an example pipeline snippet.
Anchors first: [cli test.ts#L28](../../../packages/hoppscotch-cli/src/commands/test.ts#L28), [L105](../../../packages/hoppscotch-cli/src/commands/test.ts#L105); CLI README.
Files touched: `packages/hoppscotch-cli/README.md` (or equivalent docs location — follow existing convention).
Interview story potential: minor; pairs with "how the CLI's exit code is the CI contract."

## Ticket 7: Reject non-CSV iteration files with a better error
Difficulty: Easy — 2–3h. Skills: input validation UX.
Story: As a CLI user pointing `--iteration-data` at a `.json` file, I want the error to tell me only CSV is supported *and where I passed it*.
Anchors first: [cli test.ts#L53-L93](../../../packages/hoppscotch-cli/src/commands/test.ts#L53-L93) — existing `INVALID_DATA_FILE_TYPE` error path.
Acceptance: ☐ error message includes the offending path (it already passes `data:` — improve the rendered text in the error catalog) ☐ CLI tests updated.
Interview story potential: error-message empathy — small but reviewers love it.

## Ticket 8: Test `updateTeamAccessRole` demotion guard
Difficulty: Easy — 2h. Skills: jest, RBAC edge cases.
Story: As a maintainer, I want a test that the sole OWNER cannot demote themselves ([team.service.ts#L152-L180](../../../packages/hoppscotch-backend/src/team/team.service.ts#L152-L180)).
Acceptance: ☐ ownerCount=1 + newRole=EDITOR → `TEAM_ONLY_ONE_OWNER` ☐ ownerCount=2 → success + pubsub publish asserted ([L200](../../../packages/hoppscotch-backend/src/team/team.service.ts#L200)).
Interview story potential: same family as Ticket 1; together they make one solid story.

## Ticket 9: Document the mock-server security model
Difficulty: Easy–Medium — 3h. Skills: security writing.
Story: As a contributor touching mocks, I want the origin-split threat model written down next to the code.
Why good: the code embodies a sophisticated model ([mock-server.controller.ts#L18-L46](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L18-L46), [L142-L182](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L142-L182)) explained only in scattered comments.
Files touched: a `SECURITY.md`-style note in the mock-server module or expanded header comment.
Rejection risk: docs placement is maintainer preference — ask first in an issue.
Interview story potential: "I reverse-engineered and documented an XSS defense model" — excellent security-round material.

## Ticket 10: Add `expiresOn` display logic test for PersonalAccessToken
Difficulty: Easy — 2h. Skills: nullable date handling.
Story: As a maintainer, I want tests around never-expiring tokens (`expiresOn: null`, [schema.prisma#L230](../../../packages/hoppscotch-backend/prisma/schema.prisma#L230)) in the access-token service.
Anchors first: [src/access-token](../../../packages/hoppscotch-backend/src/access-token) service + spec.
Acceptance: ☐ null expiry treated as valid ☐ past expiry rejected.
Interview story potential: nullable-field edge discipline.

## Ticket 11: Surface interceptor capability mismatch to the user
Difficulty: Medium — 4–6h. Skills: Vue, UX of degraded modes.
Story: As a user whose selected interceptor lacks a capability the request needs (e.g., custom certs), I want a hint instead of a generic failure.
Anchors first: `RelayCapabilities` on the interceptor contract ([kernel-interceptor.service.ts#L67](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L67)); the `unselectable`/reason machinery (L18-L36) shows the intended UX language.
Files touched: interceptor selection UI under `hoppscotch-common/src/components/settings` (locate via grep for `interceptor`).
Rejection risk: product-surface change — open an issue with a screenshot proposal first.
Interview story potential: "I turned a silent capability failure into an actionable UX."

## Ticket 12: Gapless-order assertion helper for collection tests
Difficulty: Medium — 3–5h. Skills: test utilities, invariants.
Story: As a maintainer, I want a reusable assertion `expectSiblingOrderGapless(rows)` used by collection reorder tests, so every ordering test checks the invariant, not just the case at hand.
Anchors first: [team-collection.service.spec.ts](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.spec.ts), the invariant ([schema.prisma#L56](../../../packages/hoppscotch-backend/prisma/schema.prisma#L56)).
Interview story potential: "I encoded an invariant as a reusable test assertion."

## Ticket 13: Normalize header casing in mock request matching tests
Difficulty: Easy — 2–3h. Skills: HTTP semantics.
Story: As a mock user, matching must be case-insensitive for header names; controller lowercases at [mock-server.controller.ts#L78-L87](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L78-L87) — add spec cases proving `X-Api-Key` == `x-api-key` end to end in [mock-server.service.spec.ts](../../../packages/hoppscotch-backend/src/mock-server/mock-server.service.spec.ts).
Interview story potential: "HTTP header semantics" trivia turned into a test.

## Ticket 14: Add a `data-testid` pass to one high-traffic component
Difficulty: Easy — 2h. Skills: testability, Vue.
Story: As a test author, I want stable selectors on the request method/URL bar component (locate under `hoppscotch-common/src/components/http`).
Rejection risk: check whether the repo prefers another selector convention first (grep `data-testid` — if absent, open an issue instead of inventing a convention).
Interview story potential: minor; feeds "how do you make UI testable."

## Ticket 15: Prune expired VerificationTokens
Difficulty: Medium — 4–6h. Skills: cron jobs, NestJS schedule.
Story: As an operator, expired magic-link tokens ([schema.prisma#L134-L142](../../../packages/hoppscotch-backend/prisma/schema.prisma#L134-L142)) should be deleted periodically instead of accumulating.
Anchors first: check `@nestjs/schedule` usage (it's in backend deps) — grep `@Cron` to find the existing pattern and follow it.
Acceptance: ☐ daily job deletes `expiresOn < now()` ☐ logged count ☐ test with mocked prisma.
What could go wrong: mass delete on a huge backlog — batch it (`deleteMany` with a limit loop) and say so in the PR.
Interview story potential: "I added a data-retention job with batching" — operations signal.

## Ticket 16: Improve the `verify/admin` response contract
Difficulty: Easy — 1–2h. Skills: API hygiene.
Story: As a client dev, `GET /auth/verify/admin` ([auth.controller.ts#L193-L199](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L193-L199)) silently auto-promotes the only user to admin ([auth.service.ts#L371-L387](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L371-L387)) — a surprising side effect on a "verify" endpoint. Propose (issue, not PR): move the bootstrap promotion to signup, or document it loudly.
Why good: teaches "GET must not mutate" — and the honest conversation that fixing it is a behavior change requiring maintainer buy-in.
Interview story potential: "I found a mutating GET and drove the design conversation" — strong judgment story even if the code never changes.

---

## How to work any of these

1. Read the anchors. 2. Write the failing test or the issue text *first*. 3. Smallest possible diff. 4. Run the package's own suite (`pnpm --filter <pkg> test`, inferred). 5. PR description per [07-.../02-writing-prs-and-rfcs.md](../07-career-and-collaboration/02-writing-prs-and-rfcs.md).
