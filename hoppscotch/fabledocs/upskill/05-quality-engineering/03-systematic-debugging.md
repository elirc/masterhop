# Systematic Debugging

The method: **reproduce → narrow → hypothesize → test cheaply → fix the root cause → add regression coverage.** Each scenario below is realistic for this repo and names the actual tools. The interview version of each: narrate the *narrowing*, not the answer — interviewers grade the search, not the destination.

Tools of this codebase: browser devtools + Vue devtools (frontend), the app's own console (script `console.log` is captured — [RequestRunner.ts#L612-L621](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L612-L621)), NestJS console logs, Prisma query logging (`DEBUG=prisma:query`, inferred standard Prisma), jest `--runInBand` + `test:debug` script ([backend package.json](../../../packages/hoppscotch-backend/package.json)), CI logs from [tests.yml](../../../.github/workflows/tests.yml).

---

## Scenario 1: "My environment variable works in the URL but not in the pre-request script"

Reproduction: set `<<baseUrl>>` in a selected env; use it in the endpoint (works) and read it via the script API (undefined).
First question: is the bug in env *resolution* (frontend state) or env *injection into the sandbox* (boundary)?
Narrowing path:
1. In the request editor, does the URL substitute correctly on send? If yes, resolution works — [EffectiveURL.ts#L226-L252](../../../packages/hoppscotch-common/src/helpers/utils/EffectiveURL.ts#L226-L252) is fine.
2. Check what env set is passed to the sandbox: [RequestRunner.ts#L520-L524](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L520-L524) — pre-request scripts get `initialEnvs` captured *before* substitution.
3. If the variable is a **secret** or has empty `currentValue`, check the filtering step ([RequestRunner.ts#L572-L574](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L572-L574) `filterNonEmptyEnvironmentVariables`).
Useful probes: `console.log(pw.env...)` in the script (captured to UI console); Vue devtools on the environment store.
Likely root causes: secret masking; variable defined in an env that isn't selected/global; empty current value filtered out.
Regression test: vitest case around the env-combining helper.
Senior lesson: two features consuming "the same" data through different paths is a bug factory — name the single source of truth.
Interview narration: "URL works, script doesn't → the divergence is after resolution, at the sandbox boundary → diff what each consumer receives."

## Scenario 2: Two teammates drag collections simultaneously; one gets a 500

Reproduction: needs concurrency — simulate with two parallel GraphQL mutations against a dev backend (a small script, or two tabs).
First question: is it the transaction (expected, retried) or the retry exhaustion (bug/load)?
Narrowing path:
1. Backend logs: `Error from TeamCollectionService.updateOrderIndex` + `Retrying updateOrderIndex... (n)` ([team-collection.service.ts#L596-L611](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L596-L611)) tell you retries fired.
2. If retries exhausted: which Prisma code? Deadlock (P2034-class) vs unique violation — the allow-list at [L602-L608](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L602-L608).
3. Check both mutations lock via `lockTeamCollectionByTeamAndParent` — a write path missing the lock is the root-cause candidate.
Useful probes: `DEBUG=prisma:query` to see lock acquisition order; Postgres `pg_locks` view during repro.
Likely root causes: legitimate contention (raise MAX_RETRIES? add jitter?) or an unlocked write path violating lock discipline.
Regression test: hard with mocks — this is where you argue for a Postgres-backed integration test (testcontainers-style), knowing the suite currently mocks Prisma ([team.service.spec.ts#L15](../../../packages/hoppscotch-backend/src/team/team.service.spec.ts#L15)).
Senior lesson: retry logs are *signal*, not noise — count them in production before they become 500s (observability gap: they're `console.debug`).

## Scenario 3: A user reports "my test script passes locally but fails in CI via `hopp test`"

First question: same sandbox? The CLI has two (`--legacy-sandbox` flag, [cli test.ts#L29](../../../packages/hoppscotch-cli/src/commands/test.ts#L29); node runners in [js-sandbox/src/node/test-runner](../../../packages/hoppscotch-js-sandbox/src/node/test-runner)); the web app uses the web/faraday-cage path.
Narrowing path:
1. Reproduce with the CLI locally, same flags as CI. Divergence gone? → env/data differences, not sandbox.
2. Still diverges? Toggle `--legacy-sandbox`. If that fixes it → an API implemented differently between runners; diff `node/test-runner/experimental.ts` vs `legacy.ts` for the API in question.
3. Check for web-only globals the script uses (e.g., `btoa`, `fetch` semantics).
Useful probes: `console.log` in the script (CLI prints captured console entries); node version (CI pins Node 22 for a reason — [tests.yml](../../../.github/workflows/tests.yml) comment).
Regression test: a fixture collection in the CLI's `__tests__` exercising the exact API.
Senior lesson: "one contract, two runtimes" (the sandbox package's whole point) still needs conformance tests, or the contract is aspirational.
Interview narration: this is a *matrix* bug — hold the script constant, vary runtime and flags one axis at a time.

## Scenario 4: After deploying a second backend replica, live collaboration "randomly" breaks

Reproduction: two browsers on the same team, load-balanced to different instances; member added in browser A; browser B doesn't update until refresh.
First question: is the event not *published*, not *delivered*, or not *rendered*?
Narrowing path:
1. Same-instance pair works (sticky session test) → publishing and rendering fine.
2. Cross-instance fails → delivery. Read [pubsub.service.ts#L14-L18](../../../packages/hoppscotch-backend/src/pubsub/pubsub.service.ts#L14-L18): in-memory `graphql-subscriptions` PubSub — events never cross processes. Root cause is architectural, and the comment at L5-L8 shows it's known.
3. Confirm with logs on both instances: publish happens on A; B's socket never receives.
Fix root cause: Redis-backed pubsub implementation behind the same `PubSubService` interface (its existence as an injectable service is what makes the fix tractable).
Regression coverage: an integration test spinning two app instances is heavy; at minimum, a deployment doc stating the single-instance constraint.
Senior lesson: "works on my machine" has an infra twin — "works with one replica." State scaling assumptions where deployers will read them.
Interview narration: this is the debugging-round version of the system-design pubsub question — connecting the two impresses.

## Scenario 5: Mock server returns the right body but the browser downloads it instead of rendering

Reproduction: create a mock with `Content-Type: text/html`, fetch it via the *path-based* URL (`/mock/{id}/...`) in a browser tab.
First question: is the header wrong at storage, at retrieval, or rewritten at response time?
Narrowing path:
1. Check the stored mock example (DB row / API response) — content type intact? Yes →
2. Curl the endpoint and inspect actual headers: `Content-Type: text/plain`, plus CSP + `X-Frame-Options`.
3. Read [mock-server.controller.ts#L142-L154](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L142-L154): same-origin responses deliberately downgrade script-capable MIME types. **Not a bug — a security control.** Subdomain access preserves the type (different origin).
Fix: none to code; fix the *user's mental model* (use subdomain URL for HTML mocks) and maybe the docs (see [Ticket 9](../06-contribution-practice/01-good-first-tickets.md#ticket-9-document-the-mock-server-security-model)).
Regression test: spec asserting the downgrade on path-based and preservation on subdomain-based access.
Senior lesson: some "bugs" are controls working as intended; the debugging deliverable is a docs PR. Recognizing this fast is judgment.
Interview narration: perfect "debugging a 'bug' that isn't" story — shows you verify intent (comments, tests, git blame) before "fixing."

---

## The habit

Before touching code, write one sentence: *"The bug is in [layer] because [evidence]."* If you can't fill in "because," you're guessing — go collect one more observation instead. Cheap observations in this repo, in order: UI console → network tab → backend console → Prisma query log → targeted unit test.
