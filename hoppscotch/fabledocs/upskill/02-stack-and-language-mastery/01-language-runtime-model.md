# Language & Runtime Model

## The event loop, using this repo's code

Mental model: one call stack; when it empties, drain **all** microtasks (promise continuations); then one macrotask (timer, I/O callback); repeat. `await` = "suspend me, queue my continuation as a microtask when the promise settles."

Where the repo leans on it:

- [network.ts#L25-L39](../../../packages/hoppscotch-common/src/helpers/network.ts#L25-L39): `RESTRequest.toRequest(req).then(...)` references `service`, declared *after* it at L39. Safe only because the `.then` callback runs in a later microtask — synchronous module execution finishes first. Works; hostile to readers; know exactly why it works ([Q14](../08-interview-prep/01-js-ts-node-deep-dive.md#q14-what-is-tdz--hoisting-and-have-you-seen-code-that-depends-on-it)).
- [mock-server.controller.ts#L135-L140](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L135-L140): artificial delay via awaited `setTimeout` promise — a macrotask. The request handler suspends; the server keeps serving. Failure mode: thousands of held sockets under load — delay features need caps.
- Fire-and-forget: [auth.service.ts#L323](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L323) calls `updateUserLastLoggedOn` **without await**. Latency win; the cost is an unhandled-rejection risk and no error propagation. Recognize the pattern and always ask "was the missing await chosen or forgotten?" (Here: plausibly chosen — it's telemetry-ish. Say that out loud in review.)

Serial vs parallel work: the repo mostly chains sequentially. When you see independent awaits in sequence (e.g., counting owners then fetching membership in [team.service.ts#L209-L217](../../../packages/hoppscotch-backend/src/team/team.service.ts#L209-L217)), `Promise.all` is the obvious optimization — and the wrong one whenever step 2 depends on step 1 or transaction semantics matter. Optimizing away sequencing is a classic mid-level PR mistake; check dependencies first.

## CPU-bound work in Node

argon2/bcrypt hashing ([auth.service.ts#L50-L53](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L50-L53), [L114](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L114), [L344](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L344)) is *designed* slow. Native async implementations run on libuv's threadpool (default 4 threads) so the event loop survives — but a login storm still queues. Mitigations visible here: throttling on the auth controller ([auth.controller.ts#L34](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L34)), configurable work factor. Full card: [Q12](../08-interview-prep/01-js-ts-node-deep-dive.md#q12-nodes-argon2bcrypt-are-async--why-and-what-happens-if-you-block).

## Workers and isolation

The web sandbox spawns a `Worker` per script run ([web/test-runner/index.ts#L21-L41](../../../packages/hoppscotch-js-sandbox/src/web/test-runner/index.ts#L21-L41)): separate thread, separate global scope, structured-clone messaging (no shared references — hence `preventCyclicObjects` and serializability constraints). Node gets a different isolation mechanism ([node/test-runner](../../../packages/hoppscotch-js-sandbox/src/node/test-runner)) behind the same interface. Pitfall checklist: ☐ non-serializable payloads (functions, DOM nodes) throw on postMessage ☐ worker leaks if never terminated ☐ no timeout = runaway scripts (real gap here — [Ticket 5](../06-contribution-practice/01-good-first-tickets.md#ticket-5-add-a-timeout-to-web-sandbox-script-execution)).

## Browser vs Node constraints — the repo's defining split

The same conceptual operation ("send this HTTP request") has different powers per runtime: browser fetch obeys CORS and can't touch some headers; the desktop/agent native path can do raw sockets, custom certs, any header. That asymmetry is *why* the interceptor abstraction exists and why interceptors declare capabilities ([kernel-interceptor.service.ts#L67](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L67)). Generalize: when one interface spans runtimes, capabilities must be data, not documentation.

## Drills

1. Predict the output ordering of `console.log` statements placed at [network.ts](../../../packages/hoppscotch-common/src/helpers/network.ts) L23, L36, L39, and inside the L44 `.then` — write it down, then justify each hop (sync / microtask / after-await).
2. Find one more fire-and-forget call in the backend (grep for un-awaited `this.` calls in services) and judge: chosen or forgotten?
3. `deleteCollectionAndUpdateSiblingsOrderIndex` awaits `delay(retryCount * 100)` ([team-collection.service.ts#L610](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L610)). What else can the process do during that delay? (Everything — that's the point of async backoff.)

## Interview angle

- "Explain the event loop" → [Q1](../08-interview-prep/01-js-ts-node-deep-dive.md#q1-walk-me-through-what-the-event-loop-does-when-you-await-inside-an-async-function)
- "Promises vs observables" → [Q2](../08-interview-prep/01-js-ts-node-deep-dive.md#q2-promises-vs-rxjs-observables--when-is-a-promise-not-enough)
- "CPU-bound work in Node" → [Q12](../08-interview-prep/01-js-ts-node-deep-dive.md#q12-nodes-argon2bcrypt-are-async--why-and-what-happens-if-you-block)
- "Running untrusted code" → [Q9](../08-interview-prep/01-js-ts-node-deep-dive.md#q9-how-do-you-run-untrusted-javascript-safely-in-the-browser-in-node)
