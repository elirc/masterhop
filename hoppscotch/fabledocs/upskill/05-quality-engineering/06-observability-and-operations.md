# Observability and Operations

## What exists

- **Logging**: `console.log/error/debug` in the paths read ([team-collection.service.ts#L596-L611](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L596-L611), [auth flows](../../../packages/hoppscotch-backend/src/auth/auth.service.ts), [pubsub init](../../../packages/hoppscotch-backend/src/pubsub/pubsub.service.ts#L15)). No structured logger observed in files read (**verify repo-wide before asserting** — grep `Logger` for NestJS's built-in usage).
- **Health checks**: a dedicated [health module](../../../packages/hoppscotch-backend/src/health) using `@nestjs/terminus` (dep list), plus root [healthcheck.sh](../../../healthcheck.sh) for containers — read both when wiring deploys.
- **Product telemetry**: [posthog module](../../../packages/hoppscotch-backend/src/posthog) — analytics, not ops.
- **Error boundaries (frontend)**: script failures become data (`scriptError: true`, [RequestRunner.ts#L667-L685](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L667-L685)); interceptor errors carry human-message components ([kernel-interceptor.service.ts#L38-L47](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L38-L47)) — errors-as-UI-props is a pattern worth stealing.
- **Deploy/rollback**: Docker images via [release-push-docker.yml](../../../.github/workflows/release-push-docker.yml); self-hosters run [docker-compose.deploy.yml](../../../docker-compose.deploy.yml) or the AIO container ([aio_run.mjs](../../../aio_run.mjs)); DB rollback = forward migrations only ([data model](../03-architecture-and-patterns/02-data-model-and-persistence.md#how-to-safely-change-this-schema)).

## "How would I know this broke?" — per key flow

| Flow | Current answer | What a senior would add |
| --- | --- | --- |
| Request send (client) | user sees error state | frontend error telemetry with interceptor id + error class |
| Sandbox scripts | user sees `scriptError` | counter: executions, failures, (future) timeouts — per runtime |
| Auth | console errors, users email support | login funnel metrics: signin→verify conversion, token-refresh failure rate (spikes = clock skew, cookie misconfig, or an attack) |
| Collection ordering | `console.debug` retries, 500 after exhaustion | retry counter + alert on exhaustion; that counter is your deadlock canary |
| Subscriptions | silent staleness (worst kind) | gauge: active sockets per instance; synthetic canary — publish/receive round-trip |
| Mock server | `MockServerLog` rows ([schema.prisma#L267-L286](../../../packages/hoppscotch-backend/prisma/schema.prisma#L267-L286)) — genuinely good, *per-feature* observability with indexes | wire p95 `responseTime` from those rows to ops dashboards |
| Email | request fails inline | delivery failure counter by template |

The mock server is the teachable contrast: it logs every hit with timing, status, IP because logs are *its product feature*. Ops observability for everything else lagging behind product observability is extremely common — name that dynamic in interviews.

## The minimum viable upgrade (what you'd actually PR)

1. NestJS structured logger (pino) with request IDs — one module, thread it into the retry and auth paths first.
2. Three counters before any dashboards: order-retry exhaustions, sandbox failures, refresh failures.
3. A `/health` deep check that includes DB reachability (terminus likely has it — verify then extend).

Interview angle: "How do you debug production?" — the honest structure: *logs to locate, metrics to quantify, traces to connect; and if a flow has none of the three, that absence is the first bug.* Use the subscriptions row as your example of a silent-failure mode you'd instrument before it pages anyone — it ties to the [multi-instance scenario](03-systematic-debugging.md#scenario-4-after-deploying-a-second-backend-replica-live-collaboration-randomly-breaks).
