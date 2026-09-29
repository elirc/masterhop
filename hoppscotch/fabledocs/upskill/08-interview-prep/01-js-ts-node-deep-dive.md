# JS / TS / Node Deep-Dive Question Cards

14 cards. Every one is anchored to code you have read, so you answer with evidence instead of trivia. Round for all: **JS/TS deep-dive**. Practice drill for each unless noted: answer aloud in 90 seconds, then open the anchor and check yourself.

## Q1: Walk me through what the event loop does when you `await` inside an async function.

What it's really testing: microtask vs macrotask model, not vocabulary.
Repo anchor: [network.ts#L25-L66](../../../packages/hoppscotch-common/src/helpers/network.ts#L25-L66) — a promise chain (`execResult.then`) started before `service` is even assigned at L39, which still works because the `.then` callback runs in a later microtask.
Junior answer sounds like: "await pauses the function."
Mid-level adds: await suspends the function, control returns to the caller synchronously; the continuation is queued as a microtask when the promise settles; microtasks drain before the next macrotask/render.
Senior includes: uses the anchor — explains why `service` being declared *after* its use in the `.then` at L36 is safe (TDZ applies at reference time; the callback references it only after synchronous execution finishes) and still calls it a readability defect worth fixing.
Likely follow-ups: "Where would `setTimeout(fn, 0)` land relative to that `.then`?" / "What starves if microtasks loop?"

## Q2: Promises vs RxJS observables — when is a promise not enough?

What it's really testing: whether you've felt the limits of one-shot async.
Repo anchor: [network.ts#L15-L80](../../../packages/hoppscotch-common/src/helpers/network.ts#L15-L80) — request lifecycle as `BehaviorSubject` emitting `loading → success|fail`, plus a separate cancel function.
Junior: "observables emit multiple values."
Mid: promises can't represent intermediate states or cancellation; here the UI needs `loading` immediately and a terminal state later, and `BehaviorSubject` replays the latest state to late subscribers (a tab re-render resubscribing gets the current state free).
Senior: also argues the cost — RxJS teaches everyone reading the file; a discriminated-union state + AbortController would serve today's usage; keeps it because history/streams integrate elsewhere. Tradeoff, not dogma.
Likely follow-ups: "Implement cancellation with promises." (AbortController; compare with `ExecutionResult.cancel` at [kernel-interceptor.service.ts#L49-L54](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L49-L54).)

## Q3: What is a discriminated union and why would you type API results with one?

What it's really testing: TS narrowing in practice.
Repo anchor: `HoppRESTResponse` usage — [RequestRunner.ts#L587-L590](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L587-L590) filters `res.type === "success" || res.type === "fail"` and TS narrows the payload inside.
Junior: defines a union.
Mid: the `type` field lets the compiler prove which fields exist per branch; impossible states (success with an error object) become unrepresentable.
Senior: connects to `Either` (Q4) as the same idea, and notes the design rule: model *expected* failures in the type, keep exceptions for bugs.
Likely follow-ups: "How does `filter` preserve the narrowing?" (type predicate signatures).

## Q4: Your service returns `Either<Error, T>` instead of throwing. Defend it. Attack it.

What it's really testing: judgment, not fp-ts trivia.
Repo anchor: [auth.service.ts#L257-L326](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L257-L326) — six typed failure exits; boundary translation at [auth.controller.ts#L77-L81](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L77-L81).
Junior: "Either has Left and Right."
Mid: defend — failure paths are in the signature, callers must handle them, error codes are greppable constants ([errors.ts](../../../packages/hoppscotch-backend/src/errors.ts)). Attack — ceremony, interop with throwing libs (see the try/catch wrapping Prisma calls), team onboarding cost.
Senior: states the boundary rule this repo follows: Either *inside* services, converted to HTTP/GraphQL errors exactly once at the controller/resolver edge — and flags mixing (`throwErr` inside resolvers) as the consistency cost to watch.
Likely follow-ups: "Where would you allow throwing?" (programmer errors — see the guard's `BUG_*` throws, [gql-team-member.guard.ts#L26-L35](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L26-L35)).

## Q5: How do zod (or similar) runtime schemas relate to TypeScript types?

What it's really testing: compile-time vs runtime boundary awareness.
Repo anchor: [rest/index.ts#L75-L115](../../../packages/hoppscotch-data/src/rest/index.ts#L75-L115) — zod schema parses untrusted persisted JSON; `InferredEntity<typeof HoppRESTRequest>` derives the static type from the runtime schema (L115).
Junior: "zod validates data."
Mid: TS types are erased; anything crossing a trust boundary (localStorage, DB JSON, network) needs runtime validation, and `z.infer` keeps the two in lockstep — one source of truth.
Senior: points at the version discriminator (L102-L112): validation isn't a boolean, it's *dispatch* — parse decides which schema generation applies, then migrations normalize.
Likely follow-ups: "Where would you NOT validate?" (internal function calls — cost without a trust boundary).

## Q6: Explain generics with a real example that isn't `Array<T>`.

What it's really testing: whether you can *design* with generics.
Repo anchor: [pubsub.service.ts#L24-L26](../../../packages/hoppscotch-backend/src/pubsub/pubsub.service.ts#L24-L26) — `publish<T extends keyof TopicDef>(topic: T, payload: TopicDef[T])`: the payload type is *looked up* from the topic argument.
Junior: containers of T.
Mid: explains the indexed-access constraint — pass topic `'team/x/member_added'`, and the compiler demands the matching payload shape; producer/consumer drift becomes a compile error.
Senior: notes the asymmetry — `asyncIterator<T>` (L20-L22) takes an unconstrained T, so the subscribe side is unchecked; proposes typing it and what breaks (topic strings are dynamic — template literal types can fix it).
Likely follow-ups: "Write that signature." Do it on paper.

## Q7: `unknown` vs `any` — where does it matter in real code?

What it's really testing: type-safety hygiene.
Repo anchor: [cli test.ts#L48-L73](../../../packages/hoppscotch-cli/src/commands/test.ts#L48-L73) — CSV rows come in as `unknown[]`, then are narrowed via `as Record<string, unknown>` before use.
Junior: "any disables checking, unknown is safer."
Mid: `unknown` forces a narrowing step at the parse boundary; `any` silently propagates. Notes the repo *does* use `as any` in test files and sync code ([gqlCollections.sync.ts#L38](../../../packages/hoppscotch-selfhost-web/src/platform/collections/web/gqlCollections.sync.ts#L38) returns `any`) — real codebases leak.
Senior: policy answer — `any` acceptable in test doubles, banned on data crossing boundaries; a returned `any` (the sync transform) is worse than a local one because it infects every caller.
Likely follow-ups: "Refactor that `any`-returning function's signature."

## Q8: What does a compound unique constraint give you that app-level checks don't?

What it's really testing: Node devs who understand their database.
Repo anchor: [`@@unique([teamID, parentID, orderIndex])`](../../../packages/hoppscotch-backend/prisma/schema.prisma#L56) plus the lock at [team-collection.service.ts#L476-L505](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L476-L505).
Junior: "prevents duplicates."
Mid: app checks race; the constraint is atomic and survives every code path including manual SQL. Belt (lock) and suspenders (constraint).
Senior: the NULL caveat — Postgres treats NULLs as distinct in unique indexes, so root collections (`parentID NULL`) are *not* fully protected by the constraint; that's why the explicit lock isn't optional. (Label as investigate-and-verify if challenged.)
Likely follow-ups: "What error surfaces when it's violated and how should code react?" (P2002-class → the retry allow-list, [L602-L608](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L602-L608)).

## Q9: How do you run untrusted JavaScript safely in the browser? In Node?

What it's really testing: security model of the runtime.
Repo anchor: [js-sandbox/src/web/test-runner/index.ts#L21-L41](../../../packages/hoppscotch-js-sandbox/src/web/test-runner/index.ts#L21-L41) (Worker) and [src/node/test-runner](../../../packages/hoppscotch-js-sandbox/src/node/test-runner) (isolated runtimes, legacy vs experimental).
Junior: "eval is dangerous, use a sandbox."
Mid: web — Worker gives a separate global scope and thread, communicate via structured clone; Node — separate isolate/process; both: inject capabilities explicitly, return results as data.
Senior: threats beyond globals — prototype pollution through injected objects, unbounded CPU (no timeout visible in the web path — real finding), and why the sandbox is a separate package (one contract, two runtimes, CLI parity).
Likely follow-ups: "How do you kill a runaway script?" (worker.terminate() on a deadline — design it).

## Q10: What actually happens during `pnpm install` in a monorepo like this?

What it's really testing: tooling literacy beyond `npm i`.
Repo anchor: [root package.json#L9-L22](../../../package.json#L9-L22) (`pnpm -r do-*` orchestration), backend `postinstall` running `prisma generate` + GraphQL SDL emit ([hoppscotch-backend/package.json](../../../packages/hoppscotch-backend/package.json)).
Junior: "installs dependencies."
Mid: workspace linking (packages resolve each other via symlinks), lifecycle scripts run codegen so generated clients exist before first build; `only-allow pnpm` guards the package manager; `pnpm.overrides` pins transitive versions ([package.json#L36-L52](../../../package.json#L36-L52)).
Senior: supply-chain angle — `onlyBuiltDependencies` allow-lists which packages may run build scripts ([package.json#L53-L68](../../../package.json#L53-L68)); that's a deliberate defense against postinstall malware, worth saying in any 2026 interview.
Likely follow-ups: "Why do the `do-dev`/`do-test` indirection scripts exist?" (uniform recursive invocation across heterogeneous packages).

## Q11: Explain closure capture bugs with a UI example.

What it's really testing: closures under async timing.
Repo anchor: [RequestRunner.ts#L520-L585](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L520-L585) — `cancelCalled` and `cancelFunc` are closure variables checked/assigned across multiple async stages (L526 checks a flag set by a cancel that may fire mid-pipeline).
Junior: recites the loop-variable example.
Mid: identifies stale-capture risk — every `.then` stage sees current variable values, so the flag pattern works; capturing a *snapshot* (e.g., destructured field) instead would freeze it.
Senior: relates to Vue refs — `tab.value.document` is deliberately re-read at write time (L609) so results land on the tab even after user navigation; discusses what breaks if the tab was closed (ticket-worthy question).
Likely follow-ups: "How does this interact with cancellation racing the response?"

## Q12: Node's `argon2`/`bcrypt` are async — why, and what happens if you block?

What it's really testing: event-loop + libuv understanding attached to real security code.
Repo anchor: [auth.service.ts#L50-L53](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L50-L53) (bcrypt salt), [L114](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L114) (argon2 hash), [L344-L347](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L344-L347) (argon2 verify).
Junior: "they're slow so they're async."
Mid: they're CPU-intensive *by design* (work factor); the native implementations run on libuv's threadpool so the event loop keeps serving; a sync variant would freeze every concurrent request for tens of ms.
Senior: capacity math — threadpool defaults to 4; a login storm queues hashing; mention `UV_THREADPOOL_SIZE`, rate limiting (this controller is throttled, [auth.controller.ts#L34](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L34)), and why the *work factor* is config (`TOKEN_SALT_COMPLEXITY`).
Likely follow-ups: "Why argon2 for tokens but bcrypt for the salt here?" (honest answer: mixed legacy; argon2 is the modern pick — noticing the inconsistency is the point).

## Q13: How does structural typing bite you when two "same-shape" models diverge?

What it's really testing: TS's structural model + boundary mapping discipline.
Repo anchor: DB `TeamMember` vs GraphQL `TeamMember` — mapped explicitly at [team.service.ts#L194-L198](../../../packages/hoppscotch-backend/src/team/team.service.ts#L194-L198) (`id` → `membershipID`).
Junior: "TS compares shapes not names."
Mid: while shapes coincide, TS lets DB rows flow into API types unnoticed; the explicit `cast`/mapping functions ([Pattern 10](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-10-boundary-mapping-cast-db-model--api-model)) are the firewall.
Senior: proposes branded types or separate generated types (Prisma client types vs GraphQL model classes — which this repo does have) as structural-typing countermeasures.
Likely follow-ups: "Show how a leaked DB field becomes a security bug." (e.g., `refreshToken` on User — [schema.prisma#L101](../../../packages/hoppscotch-backend/prisma/schema.prisma#L101) — must never reach a resolver return type.)

## Q14: What is TDZ / hoisting, and have you seen code that depends on it?

What it's really testing: language mechanics with judgment about readability.
Repo anchor: [network.ts#L36-L39](../../../packages/hoppscotch-common/src/helpers/network.ts#L36-L39) — `service.execute(...)` appears textually before `const service = ...`; legal only because the reference executes inside a later microtask.
Junior: "var hoists, let/const don't."
Mid: `const` bindings are hoisted but in the temporal dead zone until initialization; *referencing* early throws, but a closure that merely *contains* the reference is fine if invoked after initialization.
Senior: verdict — correct but hostile to readers; would reorder in review and cite it as an example that "works" ≠ "reviewable."
Likely follow-ups: "What exactly would throw if `.then` ran synchronously?" (ReferenceError — and promise semantics guarantee it can't.)

---

**Coverage check**: event loop (Q1, Q12), async patterns (Q2, Q11), closures (Q11, Q14), modules/monorepo (Q10), error handling (Q4), TS types/narrowing/generics (Q3, Q5, Q6, Q7, Q13), Node runtime (Q9, Q12), DB-adjacent Node (Q8). All 14 repo-anchored.
