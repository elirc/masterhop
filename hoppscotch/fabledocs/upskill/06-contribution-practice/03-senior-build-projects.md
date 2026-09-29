# Senior Build Projects

Six projects, 2 days–4 weeks. Each is something upstream might actually accept (or that teaches the equivalent judgment). Required artifacts per project: design doc → migration plan → test plan → security note → rollout/rollback → then code.

## P1: Redis-backed PubSub (multi-instance real-time) — ~2 weeks
Problem: in-process pubsub caps the backend at one replica ([pubsub.service.ts#L14-L18](../../../packages/hoppscotch-backend/src/pubsub/pubsub.service.ts#L14-L18); the code comments already intend this).
Product value: horizontal scaling for self-hosted/cloud deployments.
Design checklist: same `PubSubService` interface (publish/asyncIterator); `graphql-redis-subscriptions` or hand-rolled; config-driven fallback to local for dev; topic namespace unchanged (they're contracts); delivery stays at-most-once — document it.
Likely files: `pubsub/pubsub.service.ts`, `pubsub/pubsub.module.ts`, config, docker-compose (Redis service), deployment docs.
Test plan: contract tests run against both implementations; two-process manual verification script.
Security: Redis auth/TLS in deploy docs; no payload changes.
Rollout: env-flag selection, local default; rollback = flip flag.
Open questions: cron-job duplication across instances (fix in same PR or explicitly defer); subscription reconnect behavior during deploys.
Interview story potential: textbook "how did you scale a real-time system" with before/after architecture.

## P2: Real-database integration test suite — ~1–2 weeks
Problem: the concurrency machinery (locks, retries, cascades, constraints) is untested by the mock-based suite ([testing strategy gap](../05-quality-engineering/01-testing-strategy.md#the-strategic-gap-say-this-carefully-and-its-a-senior-statement)).
Design checklist: testcontainers vs CI service container; migration-per-suite; deterministic concurrency harness (start-barrier + Promise.all + assert invariants); which 10 behaviors first (create/move/reorder/delete collections, leaveTeam race, cascade topology, magic-link single-use).
Test plan *for the tests*: prove they fail — comment out the lock, watch the suite catch it (mutation-testing mindset; put that in the PR description).
Rollout: separate CI job, non-required → required after a week of stability.
Interview story potential: "I made concurrency bugs falsifiable" — exceptional senior signal.

## P3: Transactional single-owner invariant + membership hardening — ~2–4 days
Problem: count-then-write races in `leaveTeam`/`updateTeamAccessRole` ([risk R1](../09-reference/risk-register.md)).
Design checklist: `SELECT ... FOR UPDATE` on the team's membership rows inside a transaction (the idiom exists one module over); audit *all* owner-count call sites; error semantics unchanged.
Test plan: P2's harness racing two departures; unit tests unchanged.
Rollout: pure code change; measure lock wait (negligible — membership rows per team are few).
Interview story potential: a complete TOCTOU story — find, prove, fix, verify. Rehearse it as your debugging-round centerpiece.

## P4: Sandbox execution budget (web + node parity) — ~1–2 weeks
Problem: no visible timeout on user scripts ([Flow 2](../01-codebase-cartography/05-key-flows.md#flow-2-running-a-user-test-script-sandboxasync-flow)); a hung script hangs the run (and in CLI, the CI job).
Design checklist: web — `Promise.race` + `worker.terminate()`; node — inspect what faraday-cage/isolated runtime offers (the experimental runner's keepAlive comment at [experimental.ts#L100](../../../packages/hoppscotch-js-sandbox/src/node/test-runner/experimental.ts#L100) suggests sharp edges); unified config (default + per-run override for CLI `--script-timeout`); error surfaced as a distinct `TestResult` state, not a generic failure.
Security note: timeouts are also a DoS control for shared runners.
Rollout: generous default (30s), telemetry-if-available on how often it fires before tightening.
Interview story potential: "I put resource budgets around untrusted code execution across two runtimes."

## P5: Shared tree/ordering library for user+team collections — ~3–4 weeks
Problem: two parallel implementations of tree+ordering logic drift ([critique P4](../03-architecture-and-patterns/06-architecture-critique.md#p4--duplicated-userteam-hierarchies)).
Design checklist: extract *logic*, not tables — a module parameterized by (model delegate, scope key) offering create/move/reorder/delete-with-reindex; adopt in team-collection first (richest), then user-collection; behavior-preserving (P2 suite is the safety net — sequence this after P2).
Migration plan: strangler — new module used by one method at a time, old code deleted per method.
What a maintainer would reject: a big-bang rewrite PR; submit as 5+ small PRs with an RFC first ([writing RFCs](../07-career-and-collaboration/02-writing-prs-and-rfcs.md)).
Interview story potential: "I removed duplicated invariant logic across tenancy models via strangler refactor."

## P6: Mock-server response templating (dynamic mocks) — ~3–4 weeks, product-scoped
Problem/value: static mock bodies limit realism; users want `{{request.query.id}}` echoes and randomized fields (competitive parity feature).
Design checklist: templating language choice (safe subset — **no** arbitrary JS unless it goes through the existing sandbox; that decision is the heart of the design); where templates render ([mock-server.service.handleMockRequest](../../../packages/hoppscotch-backend/src/mock-server/mock-server.service.ts)); template errors → 500 vs literal passthrough; size/depth limits.
Security plan: templating engines are injection surfaces — no access to process/env; render output still passes the header/MIME defenses ([mock-server.controller.ts#L107-L182](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L107-L182)); CSP posture unchanged.
Rollout: feature flag per mock server; docs; log template-render failures to `MockServerLog`.
Open questions: versioning stored mock examples (the `mockExamples Json` shape — does it need the verzod treatment?).
Interview story potential: a full product-feature design under security constraints — your best "design a feature end to end" story.

---

Sequencing advice: P3 → P2 → P1 builds each on the last's safety net. P4 anytime. P5 only after P2. P6 standalone.
