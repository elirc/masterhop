# Key Flows

Seven end-to-end flows, each traced against real code. These rotate through the rest of the curriculum — drills, tickets, and interview cards reference them by name.

Vocabulary used below (all interview vocabulary): an **invariant** is a condition the system must keep true (e.g., sibling order indexes are unique); a **boundary** is a line where ownership and trust change (UI→API, API→DB, app→user script); a **contract** is the agreed shape crossing a boundary.

---

## Flow 1: Sending a REST request (UI/client flow)

Why this flow matters: it is the product. Every other feature exists to feed or consume this path. It also demonstrates streams-as-results, pluggable transports, and sandboxed user code in one trace.

Open these files first:
- [RequestRunner.ts#L490-L605](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L490-L605) — orchestration
- [network.ts#L15-L80](../../../packages/hoppscotch-common/src/helpers/network.ts#L15-L80) — stream creation
- [kernel-interceptor.service.ts#L56-L69](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L56-L69) — the transport contract

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | UI tab | [RequestRunner.ts#L490-L518](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L490-L518) | Request resolved with inherited auth/headers/scripts from collection hierarchy | `HoppRESTRequest` + inherited props | Inheritance bugs surface here, not in the editor |
| 2 | Sandbox | [RequestRunner.ts#L520-L531](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L520-L531) | Pre-request script(s) run; may mutate envs/request/cookies | fp-ts `Either<err, {updatedEnvs, updatedRequest, ...}>` | Script failure aborts with `script_fail` |
| 3 | Env engine | [RequestRunner.ts#L576-L581](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L576-L581) | `getEffectiveRESTRequest` substitutes `<<var>>` templates ([EffectiveURL.ts#L226-L252](../../../packages/hoppscotch-common/src/helpers/utils/EffectiveURL.ts#L226-L252)) | `EffectiveHoppRESTRequest` | Secret masking vs real values |
| 4 | Network | [network.ts#L15-L39](../../../packages/hoppscotch-common/src/helpers/network.ts#L15-L39) | Request converted to kernel format, handed to active interceptor; `BehaviorSubject` starts at `{type:"loading"}` | `RelayRequest` | Conversion failure → `network_fail` |
| 5 | Interceptor | [kernel-interceptor.service.ts#L156-L172](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L156-L172) | Selected transport (browser fetch / extension / agent / native) executes; returns `{cancel, response: Promise<Either>}` | `Either<KernelInterceptorError, RelayResponse>` | Throws if no interceptor active |
| 6 | Stream | [network.ts#L41-L66](../../../packages/hoppscotch-common/src/helpers/network.ts#L41-L66) | Right → success/fail response emitted; Left → `interceptor_error`; stream completes | `HoppRESTResponse` union | One-shot stream; late subscribers get last value |
| 7 | Sandbox | [RequestRunner.ts#L587-L605](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L587-L605) | Post-request (test) script runs against status/body/headers | `SandboxTestResult` | Test script errors must not crash the tab |
| 8 | UI state | [RequestRunner.ts#L607-L660](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L607-L660) | Response + test results written to the tab; env diffs applied only if scripts changed envs; cookie jar rebuilt | tab document mutation | Concurrent tabs: response written per-tab on purpose (comment at L608) |

Validation and authorization: none server-side — this flow can run fully client-side against any URL. The trust boundary is the **sandbox** (user script vs app) and the **interceptor** (app vs network).

Persistence and side effects: history entry, env updates, cookie jar ([RequestRunner.ts#L644-L660](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L644-L660)).

Tests that cover it: `hoppscotch-common` has vitest suites for helpers (`src/helpers/__tests__`), but no end-to-end test of this exact pipeline was found — **investigate**.

What juniors usually miss: the request that is sent is *not* the request in the editor — scripts and env substitution produce an "effective request" ([EffectiveURL.ts](../../../packages/hoppscotch-common/src/helpers/utils/EffectiveURL.ts)).

What seniors notice: cancellation is a first-class part of the transport contract (`ExecutionResult.cancel`, [kernel-interceptor.service.ts#L49-L54](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L49-L54)); the response is a discriminated union, not a thrown exception.

Interview angle: "Design an HTTP client with pluggable transports" and "How do you run untrusted user code?" both map directly here.

Drill: without running the app, predict what happens if the pre-request script throws. Then read [RequestRunner.ts#L526-L531](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L526-L531) and check yourself.
Self-grade — Basic: you can name the 8 steps. Solid: you can name the two trust boundaries and the effective-request transformation. Strong: you can argue why the response is modeled as a stream (loading state, cancellation, one-shot completion) and what a Promise-only design would lose.

---

## Flow 2: Running a user test script (sandbox/async flow)

Why this flow matters: executing user-authored JS inside your product is a maximal-blast-radius feature. The repo isolates it behind a worker + capability-injection library (`faraday-cage`).

Open these files first:
- [js-sandbox/src/web/test-runner/index.ts#L1-L41](../../../packages/hoppscotch-js-sandbox/src/web/test-runner/index.ts#L1-L41) — web worker path
- [js-sandbox/src/node](../../../packages/hoppscotch-js-sandbox/src/node) — node runner (legacy + experimental)

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Runner | [RequestRunner.ts#L593-L605](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L593-L605) | Test script + envs + response snapshot passed to sandbox | script string + `TestResponse` | Response body must be serializable |
| 2 | Sandbox host | [web/test-runner/index.ts#L21-L41](../../../packages/hoppscotch-js-sandbox/src/web/test-runner/index.ts#L21-L41) | New `Worker` spawned; payload posted; resolves on first message | `postMessage` clone | Worker per run: isolation over reuse |
| 3 | Cage | [web/test-runner/index.ts#L43-L60](../../../packages/hoppscotch-js-sandbox/src/web/test-runner/index.ts#L43-L60) | `faraday-cage` executes script with injected modules (`pw`/`hopp` APIs, console capture) | `TestDescriptor` tree | Bootstrap errors trigger cage reset + retry |
| 4 | Result | back in RequestRunner | `Either<string, SandboxTestResult>` returned; env diff computed; UI updated | expect results + env diff | Script errors → `scriptError: true` block, not a crash |

The same package ships a **node** implementation for the CLI ([src/node/test-runner](../../../packages/hoppscotch-js-sandbox/src/node/test-runner)) with `legacy.ts` and `experimental.ts` variants — the CLI exposes `--legacy-sandbox` ([cli test.ts#L29](../../../packages/hoppscotch-cli/src/commands/test.ts#L29)). One contract, two runtimes: that is why the sandbox is its own package.

What juniors usually miss: `console.log` inside user scripts is captured and replayed into the UI console ([RequestRunner.ts#L612-L621](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L612-L621)) — the sandbox has no direct I/O.

What seniors notice: a worker is spawned per execution with no visible timeout in the web path — **investigate**: what stops `while(true){}` in a test script? (The node experimental runner comments on `faraday-cage`'s keepAlive loop swallowing rejections at [experimental.ts#L100](../../../packages/hoppscotch-js-sandbox/src/node/test-runner/experimental.ts#L100).)

Interview angle: "How would you safely run user-provided code?" — answer with: separate process/worker, capability injection instead of ambient globals, serialize inputs/outputs, treat results as data (`Either`), plan for runaway scripts.

Drill: list three capabilities a test script has (env read/write, assertions, console) and three it must never have (DOM, cookies of the host app, network via host credentials). Verify by reading the module list imported at [web/test-runner/index.ts#L5](../../../packages/hoppscotch-js-sandbox/src/web/test-runner/index.ts#L5).
Self-grade — Basic: you know scripts run in a worker. Solid: you can explain capability injection. Strong: you can propose a timeout/kill design and its UX tradeoff.

---

## Flow 3: Magic-link sign-in (auth/security boundary)

Why this flow matters: passwordless auth, token rotation, and cookie handling in one path — the densest security material in the repo.

Open these files first:
- [auth.controller.ts#L51-L100](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L51-L100) — endpoints
- [auth.service.ts#L206-L326](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L206-L326) — the logic

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Controller | [auth.controller.ts#L51-L71](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L51-L71) | `POST /auth/signin`: provider allow-list checked, email validated | `SignInMagicDto` | Provider check is config-driven |
| 2 | Service | [auth.service.ts#L206-L221](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L206-L221) | User found-or-created by email | `AuthUser` | Account enumeration is accepted by design (same response either way) |
| 3 | Service | [auth.service.ts#L50-L73](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L50-L73) | `VerificationToken` row created: bcrypt-salt `deviceIdentifier` + cuid token, expiry default 24h | DB row | Expiry parsed from config with NaN fallback |
| 4 | Mailer | [auth.service.ts#L238-L244](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L238-L244) | Email sent with `${url}/enter?token=...`; `deviceIdentifier` returned to *this browser* | magic link | Device binding: link only works where sign-in started |
| 5 | Controller | [auth.controller.ts#L76-L81](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L76-L81) | `POST /auth/verify` with token + deviceIdentifier | `VerifyMagicDto` | — |
| 6 | Service | [auth.service.ts#L257-L326](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L257-L326) | Token pair looked up, user loaded, provider account ensured, **expiry checked**, tokens issued, verification row deleted (single-use) | `AuthTokens` | ⚠ expiry checked at L299-L304 *after* provider-account creation at L290-L297 — investigate ordering |
| 7 | Cookies | [auth.controller.ts#L80](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L80) | `authCookieHandler` sets `access_token` + `refresh_token` cookies | JWTs | Cookie flags live in `auth/helper.ts` |
| 8 | Refresh | [auth.controller.ts#L87-L100](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L87-L100) + [auth.service.ts#L335-L363](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L335-L363) | Refresh-token rotation: cookie token verified against argon2 hash stored on the user row; new pair issued | rotated `AuthTokens` | Only the *hash* is stored ([L114](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L114)) — DB leak ≠ session theft |

Validation and authorization: DTO validation via class-validator; rate limiting via `ThrottlerBehindProxyGuard` on the whole controller ([auth.controller.ts#L34](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L34)); SSO callbacks skip throttle ([L114](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L114)).

Tests that cover it: [auth.service.spec.ts](../../../packages/hoppscotch-backend/src/auth/auth.service.spec.ts) exists; coverage of the expiry-ordering edge was not confirmed — **investigate**.

What juniors usually miss: the `deviceIdentifier` — the emailed link alone is insufficient; you must present the identifier issued to the original browser ([schema.prisma#L134-L142](../../../packages/hoppscotch-backend/prisma/schema.prisma#L134-L142), unique on the pair).

What seniors notice: first-registered user is auto-promoted to admin when the user count is exactly 1 ([auth.service.ts#L371-L387](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L371-L387)) — a bootstrap convenience with a security story to tell.

Interview angle: "Design passwordless login." Junior describes the email link. Mid adds single-use tokens, expiry, device binding. Senior adds rotation, hashed refresh tokens, throttling, and enumeration tradeoffs.

Drill: write down what an attacker who steals (a) the emailed link, (b) the DB, (c) the refresh cookie can and cannot do. Anchor each claim.
Self-grade — Basic: (a). Solid: (a)+(b) with the argon2 hash argument. Strong: all three plus the rotation replay window.

---

## Flow 4: Team RBAC on a GraphQL mutation (authorization flow)

Why this flow matters: this is the repo's multi-tenant isolation story — the code that stops me from deleting *your* team.

Open these files first:
- [gql-team-member.guard.ts#L21-L43](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L21-L43)
- [team.resolver.ts#L215-L238](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L215-L238) — `removeTeamMember`

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Resolver decl | [team.resolver.ts#L215-L219](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L215-L219) | `@UseGuards(GqlAuthGuard, GqlTeamMemberGuard)` + `@RequiresTeamRole(OWNER)` | metadata | Forgetting the decorator = guard throws `BUG_*` |
| 2 | AuthN | `GqlAuthGuard` | JWT from cookie → `req.user` | `AuthUser` | — |
| 3 | AuthZ | [gql-team-member.guard.ts#L22-L35](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L22-L35) | Reads required roles from reflector; extracts `teamID` **from GraphQL args** | roles + teamID | Guard hard-fails with `BUG_TEAM_NO_TEAM_ID` if the mutation's arg isn't literally named `teamID` — convention as contract |
| 4 | AuthZ | [gql-team-member.guard.ts#L37-L42](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L37-L42) | Membership row fetched; role must be in required set | `TeamMember` | Non-member and wrong-role both → error (no info leak about team existence) |
| 5 | Service | [team.service.ts#L205-L240](../../../packages/hoppscotch-backend/src/team/team.service.ts#L205-L240) | `leaveTeam` enforces the single-owner invariant: last OWNER cannot be removed | `Either<string, boolean>` | ⚠ count-then-delete is not transactional — possible TOCTOU, see [risk register](../09-reference/risk-register.md) |
| 6 | Event | [team.service.ts#L237](../../../packages/hoppscotch-backend/src/team/team.service.ts#L237) | `team/{id}/member_removed` published for live UIs | uid | Publish after commit — good ordering |

What juniors usually miss: authorization happens **twice** — the guard checks role, the service re-checks the business invariant. Guards answer "may this user call this?"; services answer "is this state change legal?"

What seniors notice: the guard couples to the literal argument name `teamID` via `gqlExecCtx.getArgs()` ([L34](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L34)). Rename the arg and authorization breaks loudly (a `BUG_` error) — deliberate fail-closed design.

Interview angle: IDOR questions. "How do you prevent a user from accessing another tenant's data?" — point at declarative role metadata + a guard that resolves membership per request, and the fail-closed convention.

Drill: find one more mutation using `@RequiresTeamRole` with a *different* role set (e.g., collection edits allowing EDITOR) and explain why the roles differ.
Self-grade — Basic: found one. Solid: articulated the role hierarchy OWNER > EDITOR > VIEWER. Strong: identified where VIEWER-permitted reads still expose data you'd debate (e.g., member emails).

---

## Flow 5: Creating/reordering team collections (persistence flow)

Why this flow matters: the cleanest example in the repo of protecting an ordering invariant under concurrency — transactions, explicit row locks, retry with backoff.

Open these files first:
- [schema.prisma#L42-L57](../../../packages/hoppscotch-backend/prisma/schema.prisma#L42-L57) — `@@unique([teamID, parentID, orderIndex])`
- [team-collection.service.ts#L452-L517](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L452-L517) — create
- [team-collection.service.ts#L555-L616](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L555-L616) — delete + sibling reindex

Trace (create):

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Service | [L458-L465](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L458-L465) | Title length validated; parent must belong to the same team (`isOwnerCheck`) | `Either` | Cross-team parenting = data exfiltration vector, blocked here |
| 2 | Service | [L467-L472](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L467-L472) | `data` JSON parsed/validated | JSON string | Malformed JSON rejected before touching DB |
| 3 | Tx | [L476-L505](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L476-L505) | Transaction: **lock sibling rows** (`lockTeamCollectionByTeamAndParent`) → read max `orderIndex` → insert with `max+1` | DB row | Without the lock, concurrent creates race to the same index |
| 4 | Event | [L511-L514](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L511-L514) | `team_coll/{teamID}/coll_added` published | GraphQL model via `cast()` | Publish after commit |

Delete ([L555-L616](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L555-L616)) adds two more ideas: **idempotency** (a concurrent delete's P2025 "record not found" is treated as success, L576-L580) and **bounded retries with linear backoff** for deadlocks/constraint conflicts (L560-L612, `retryCount * 100` ms).

Consistency expectation: siblings are always a gapless 1..n sequence per `(teamID, parentID)`; the DB constraint is the last line of defense, the lock is the first.

Tests that cover it: [team-collection.service.spec.ts](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.spec.ts) exists (mocked Prisma). Concurrency behavior is untestable with mocks — **investigate** whether any integration test exercises the lock.

What juniors usually miss: `onDelete: Cascade` on the self-relation ([schema.prisma#L51](../../../packages/hoppscotch-backend/prisma/schema.prisma#L51)) means deleting a folder deletes an entire subtree in the DB layer — no app code walks children.

What seniors notice: the retry loop only retries specific Prisma error codes (deadlock, unique violation, timeout) and fails fast on everything else ([L602-L608](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L602-L608)) — retrying unknown errors would mask bugs.

Interview angle: "How do you implement drag-to-reorder persistently?" Junior: store an index. Mid: unique constraint + transaction. Senior: locks vs optimistic retries, gapless vs fractional indexes (e.g., `orderKey` strings), and why this repo chose pessimistic locking.

Drill: sketch the alternative fractional-index design ("insert between 3 and 4 as 3.5") and list two problems it trades for (key growth, rebalancing).
Self-grade — Basic: you can explain `max+1`. Solid: you can explain the lock. Strong: you can compare both designs and name when each wins.

---

## Flow 6: Real-time team updates (background/async flow)

Why this flow matters: collaborative UIs need server push; this repo uses GraphQL subscriptions over an in-process pubsub — simple, with a scaling cliff worth understanding.

Open these files first:
- [pubsub.service.ts#L10-L27](../../../packages/hoppscotch-backend/src/pubsub/pubsub.service.ts#L10-L27)
- [team.resolver.ts#L305-L326](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L305-L326) — `teamMemberAdded`

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Client | frontend GraphQL client | Subscribes `teamMemberAdded(teamID)` over WebSocket | subscription op | Auth on connect |
| 2 | Guard | [team.resolver.ts#L310-L316](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L310-L316) | Same `GqlTeamMemberGuard` + role decorator as mutations — subscriptions are authorized too | — | Forgetting guards on subscriptions leaks a live feed |
| 3 | Resolver | [team.resolver.ts#L317-L326](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L317-L326) | Returns `pubsub.asyncIterator('team/{id}/member_added')` | AsyncIterator | Topic string is the contract |
| 4 | Producer | [team.service.ts#L200](../../../packages/hoppscotch-backend/src/team/team.service.ts#L200), [L237](../../../packages/hoppscotch-backend/src/team/team.service.ts#L237) | Mutations publish typed payloads (`TopicDef` maps topic→payload type, [pubsub.service.ts#L24-L26](../../../packages/hoppscotch-backend/src/pubsub/pubsub.service.ts#L24-L26)) | `TeamMember` / uid | Type-checked topics prevent payload drift |
| 5 | Transport | in-memory `graphql-subscriptions` PubSub | Event fans out to sockets **on this process only** | — | ⚠ Multi-instance deploys silently drop cross-instance events — the L5-L8 comment plans Redis; none is wired. Investigate |

What juniors usually miss: authorization is checked at *subscribe time*, not per event. A member removed from the team keeps receiving events until the socket re-subscribes — **possible risk**, verify before claiming.

What seniors notice: topic names encode tenancy (`team/{teamID}/...`), so isolation rides on string construction; a typo subscribes you to nothing (fail-closed) but a bug in the publisher could cross tenants (fail-open). Grep both sides when reviewing.

Interview angle: "How would you scale WebSocket subscriptions horizontally?" — sticky sessions don't fix fan-out; you need a shared broker (Redis pub/sub) exactly where this repo left a TODO comment.

Drill: list every `pubsub.publish` topic in `team.service.ts` and `team-collection.service.ts` (grep `pubsub.publish`) and build the topic→payload table yourself.
Self-grade — Basic: table built. Solid: you spot the naming convention. Strong: you can design the Redis migration and its delivery-guarantee change (at-most-once stays at-most-once).

---

## Flow 7: Serving a mock request (public API/security boundary)

Why this flow matters: the mock server executes *user-defined responses* on the product's own domain — a textbook stored-XSS surface, and the code visibly fights it.

Open these files first:
- [mock-server.controller.ts#L39-L59](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L39-L59) — dual routing
- [mock-server.controller.ts#L107-L182](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L107-L182) — defenses

Trace:

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Guard | [mock-server.controller.ts#L52-L59](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L52-L59) | `MockRequestGuard` resolves mock server from subdomain (`id.mock.host`) or path (`/mock/id/...`) | `MockServer` on `req` | Two origins, two threat models |
| 2 | Service | [mock-server.controller.ts#L89-L103](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L89-L103) | Path+method matched against stored mock examples | `Either<err, response>` | 404 on no match |
| 3 | Headers | [L107-L133](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L107-L133) | User headers applied **except** a blocklist (`set-cookie`, CSP, `x-frame-options`…); only string/number values | header map | Blocklist at [L18-L25](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L18-L25) |
| 4 | Delay | [L135-L140](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L135-L140) | Configured artificial latency via `setTimeout` | — | Held connections; throttler is the backstop |
| 5 | XSS defense | [L142-L154](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L142-L154) | Same-origin (path-based) responses: script-capable content types (`text/html`, `image/svg+xml`, `*+xml`…) downgraded to `text/plain` | — | Subdomain access keeps real types — different origin, safe |
| 6 | Hardening | [L177-L182](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L177-L182) | `nosniff` always; CSP `default-src 'none'; sandbox` + `X-Frame-Options: DENY` on same-origin | — | Defense in depth: downgrade *and* CSP |

What juniors usually miss: why subdomain vs path access get different treatment — the browser's same-origin policy is the real security boundary; `id.mock.host` HTML can't touch `app.host` cookies, but `app.host/mock/id/` HTML could.

What seniors notice: content-type detection for the default ([L157-L176](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L157-L176)) tries `JSON.parse` on the body — a body of `"123"` becomes `application/json`. Harmless? Probably. Worth a test.

Interview angle: this flow *is* the answer to "explain XSS and how you'd let users host content safely." Cite the blocklist, the MIME downgrade, the CSP sandbox, and the origin split.

Drill: for each header in the blocklist, write one sentence on the attack it blocks.
Self-grade — Basic: 3 of 5. Solid: all 5. Strong: you can also explain why `Content-Disposition` is there (download vs inline rendering).

---

## Flow 7½ (bonus): `hopp test` in CI (CLI flow)

The CLI ([cli/src/commands/test.ts#L21-L114](../../../packages/hoppscotch-cli/src/commands/test.ts#L21-L114)) parses a collection (file path or shared ID), optional env file, optional CSV iteration data (Papa Parse, L53-L93), then hands everything to `collectionsRunner` and exits non-zero on failure (L105-L107) — that exit code is the entire CI contract. It reuses `hoppscotch-data` schemas and the node sandbox, which is the monorepo's reuse story in one sentence: **the CLI is the app minus the UI**.
