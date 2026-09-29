# Side Effects, Async, and Reliability

Inventory of every side-effect channel in the system, then the reliability concepts each one exercises.

## Side-effect map

| Channel | Trigger | Code | Reliability posture |
| --- | --- | --- | --- |
| Email (magic links, invites) | auth/invitation flows | [auth.service.ts#L238-L244](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L238-L244); [mailer module](../../../packages/hoppscotch-backend/src/mailer) | awaited inline; failure fails the request — no queue/retry |
| PubSub events → GraphQL subscriptions | team/collection mutations | [Pattern 9](05-pattern-catalog.md#pattern-9-publish-after-commit-event-ordering) | at-most-once, in-process only |
| Scheduled jobs | `@nestjs/schedule` crons | grep `@Cron` in backend | fire on whichever instance hosts the scheduler — multi-instance duplication risk (investigate) |
| Mock server logging | every mock hit | [MockServerLoggingInterceptor](../../../packages/hoppscotch-backend/src/mock-server/mock-server-logging.interceptor.ts) | async log write; also `hitCount`/`lastHitAt` counters ([schema.prisma#L257-L258](../../../packages/hoppscotch-backend/prisma/schema.prisma#L257-L258)) |
| External APIs | SSO providers, PostHog telemetry ([posthog module](../../../packages/hoppscotch-backend/src/posthog)) | | third-party latency inside auth callbacks |
| Frontend → backend sync | store mutations replayed as GraphQL calls | [gqlCollections.sync.ts](../../../packages/hoppscotch-selfhost-web/src/platform/collections/web/gqlCollections.sync.ts) | optimistic local write, async remote — the mapper is the consistency mechanism |
| User scripts | request runs | [Flow 2](../01-codebase-cartography/05-key-flows.md#flow-2-running-a-user-test-script-sandboxasync-flow) | isolated; results-as-data |

Notably **absent**: a job queue (no BullMQ/SQS), an outbox, webhooks out. Absence is architecture too — this system leans synchronous request/response plus best-effort events, which is honest for its scale and a named upgrade path when it isn't.

## The reliability concepts, taught from this code

**Idempotency.** Concurrent-delete tolerance ([team-collection.service.ts#L572-L580](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L572-L580)). Litmus test to reuse: *if this ran twice, would state differ?* Create-with-`max+1` is NOT idempotent — which is precisely why it's locked and unique-constrained instead.

**Retries — only on transient, only bounded.** The allow-list retry loop ([L560-L612](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L560-L612)); linear backoff, no jitter (fine at this contention level; add jitter when retry storms correlate).

**Ordering: commit before publish.** Every `pubsub.publish` sits after its transaction ([team.service.ts#L200](../../../packages/hoppscotch-backend/src/team/team.service.ts#L200), [team-collection.service.ts#L511-L514](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L511-L514)). The gap — crash between commit and publish — loses the event. Upgrade path when events become load-bearing: transactional outbox (write event row in the same tx; a relay publishes). Be able to draw it.

**Timeouts and cancellation.** Frontend: cancellation is part of the transport contract ([kernel-interceptor.service.ts#L49-L54](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L49-L54)); cancel races the response and is checked between pipeline stages ([RequestRunner.ts#L526](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L526)). Backend mock delay: unbounded `setTimeout` up to `delayInMs` ([mock-server.controller.ts#L135-L140](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L135-L140)) — what caps `delayInMs`? Investigate the resolver's validation; if uncapped, that's a resource-holding risk ([risk register R6](../09-reference/risk-register.md)).

**Backpressure.** None needed at current shape (no queues); the closest thing is the throttler. Know how you'd answer "what happens if mock traffic 100×?" — throttle limits, connection caps, then move mocks off the main API process.

**Failure visibility.** Retries log at `console.debug`/`console.error` ([L596-L611](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L596-L611)); no metrics/counters. "How would I know this broke?" — currently: users tell you. See [observability](../05-quality-engineering/06-observability-and-operations.md).

## Side effects in risky places (flag list)

- Provider-account creation before expiry validation in magic-link verify ([auth.service.ts#L281-L304](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L281-L304)) — write before final validation.
- Auto-admin promotion inside a GET-handled verify endpoint ([auth.service.ts#L374-L384](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L374-L384)).
- Un-awaited `updateUserLastLoggedOn` ([auth.service.ts#L323](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L323)) — acceptable-if-chosen; must never grow real logic.

## Drills

1. Build the full pubsub topic inventory (grep `pubsub.publish` across backend) into a topic → payload → subscribers table. This is *the* map for any real-time feature work.
2. Design the outbox version of `createCollection`: what's in the event row, who relays, what changes for subscribers? One page.
3. For each side-effect channel in the table, write its "how would I know it broke?" answer in one line. Three will be "I wouldn't" — those are your observability tickets.

Interview angle: idempotency/retries/outbox are the core of [03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md) Q13–Q15 and the scaling half of the [system-design walkthrough](../08-interview-prep/04-system-design-from-this-repo.md#step-5--scaling-and-evolution-prompts).
