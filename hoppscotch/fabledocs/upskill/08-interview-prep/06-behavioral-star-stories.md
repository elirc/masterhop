# Behavioral STAR Story Worksheets

Nine worksheets. Sources are the tickets/projects/katas in this curriculum plus the study itself — **do the work first, then fill the worksheet the same day.** Honesty rule: frame study-derived stories as "in a codebase I studied/contributed to," never invent employment context. Each: prompts answered, S/T/A/R skeleton, evidence, senior-signal details, resume bullet, rehearsal check (under 2 min? concrete? ends with impact?).

## Story 1: The race condition nobody had hit yet
Prompts: "a time you found a bug others missed" / "attention to detail" / "disagreement over severity".
Source: [P3 / risk R1](../06-contribution-practice/03-senior-build-projects.md#p3-transactional-single-owner-invariant--membership-hardening--2-4-days) — the leaveTeam TOCTOU.
S: multi-tenant team system with a "last owner can't leave" rule. T: verify the invariant held under concurrency. A: traced count-then-write ([team.service.ts#L205-L221](../../../packages/hoppscotch-backend/src/team/team.service.ts#L205-L221)), built a two-call race test, demonstrated zero-owner state, fixed with row-locked transaction mirroring the codebase's own idiom. R: invariant now provably atomic; race test in suite.
Senior signals: proved before claiming; reused existing idiom over inventing; sized blast radius.
Resume bullet: *Identified and fixed a TOCTOU race in team-ownership invariants; added deterministic concurrency regression tests.*

## Story 2: Adding a kill-switch for untrusted code
Prompts: "shipped something security-relevant" / "balancing UX and safety".
Source: [Ticket 5 / P4](../06-contribution-practice/03-senior-build-projects.md#p4-sandbox-execution-budget-web--node-parity--1-2-weeks).
Skeleton: user scripts ran with no time budget → runaway script hangs runs/CI → Promise.race + terminate, generous default, distinct error state → hangs impossible, failure visible.
Senior signals: chose the limit from user data (slow-but-legit scripts), parity plan across two runtimes.
Resume bullet: *Added execution budgets to a user-script sandbox across browser and Node runtimes.*

## Story 3: The optimization I argued against
Prompts: "disagreed with a teammate" / "performance work".
Source: [Review Kata 2](../04-code-reading-gym/04-review-katas.md#kata-2-speed-up-owner-check) + [Kata 5](../04-code-reading-gym/04-review-katas.md#kata-5-cache-team-membership-in-the-guard).
Skeleton: PR cached authorization lookups for speed → I quantified the staleness window (revoked members keep access 60s) → reframed from perf to security tradeoff, product decided with eyes open → bounded version shipped with event-driven invalidation.
Senior signals: didn't say "no," said "here's the real price"; escalation ladder step 3 ([maintainer communication](../07-career-and-collaboration/03-maintainer-communication.md#respectful-disagreement-the-escalation-ladder)).
Resume bullet: *Surfaced hidden security tradeoffs in a proposed authz cache; drove bounded design with explicit staleness guarantees.*

## Story 4: Learning a 12-package codebase fast
Prompts: "ramping up on unfamiliar code" / "learning something complex quickly".
Source: this curriculum's own method.
Skeleton: needed working knowledge of a large production monorepo → systematic cartography: manifests → schema → one write path → one auth path, notes with file:line anchors → could trace 7 end-to-end flows and critique the architecture within two weeks → produced a documented map others could use.
Senior signals: method over heroics; verified claims before repeating them; distinguished confirmed findings from hypotheses.
Resume bullet: *Reverse-engineered and documented the architecture of a 12-package open-source platform (Vue/NestJS/Prisma), producing an onboarding curriculum.*

## Story 5: The "bug" that was a security control
Prompts: "a time you were wrong" / "customer-reported issue".
Source: [Scenario 5](../05-quality-engineering/03-systematic-debugging.md#scenario-5-mock-server-returns-the-right-body-but-the-browser-downloads-it-instead-of-rendering).
Skeleton: user report: HTML mocks render as plain text → traced to a deliberate MIME downgrade with XSS rationale in comments → shifted deliverable from code fix to docs + supported-path guidance (subdomain origin).
Senior signals: checked intent (comments/tests) before "fixing"; converted a complaint into documentation.
Resume bullet: *Turned a recurring bug report into security-model documentation after tracing it to an intentional XSS control.*

## Story 6: Migrating a live data format
Prompts: "backwards compatibility" / "a technically complex project".
Source: [Pattern 1](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-1-versioned-entity-with-explicit-migrations-verzod) study + [M7](../06-contribution-practice/02-mid-level-feature-tickets.md#m7-collection-export-includes-mock-examples) or [Recipe 6](../05-quality-engineering/02-writing-tests-here.md#recipe-6-data-format-migration-the-repos-crown-jewel-test-type) work.
Skeleton: saved user data spans 18 format generations → extend format without breaking any of them → additive field + version bump + frozen-fixture tests per released version → old exports keep importing forever, provably.
Senior signals: "data outlives code" framing; fixture-freezing discipline.
Resume bullet: *Evolved a versioned persistence format with frozen-fixture compatibility tests across 18 schema generations.*

## Story 7: Living with two state systems
Prompts: "dealing with legacy code" / "technical debt prioritization".
Source: [framework models](../02-stack-and-language-mastery/02-framework-mental-models.md#rxjs-and-the-two-state-generations) + [Review Kata 7](../04-code-reading-gym/04-review-katas.md#kata-7-migrate-settings-store-to-dioc).
Skeleton: RxJS stores and DI services coexist; a partial migration created dual sources of truth → defined the bridge rule (single adapter, no third pattern), migrated one feature completely rather than five halfway → seam contained; playbook written for the rest.
Senior signals: "complete one migration" over "start five"; the ADR habit.
Resume bullet: *Defined and executed a contained migration path between coexisting frontend state architectures.*

## Story 8: Making the untestable testable
Prompts: "improved engineering quality" / "influenced beyond your task".
Source: [M5 / P2](../06-contribution-practice/03-senior-build-projects.md#p2-real-database-integration-test-suite--1-2-weeks).
Skeleton: concurrency machinery (locks/retries/constraints) had zero real-DB coverage → built Postgres-backed integration job with deterministic race harness → proved value by disabling a lock and watching the suite catch it → concurrency regressions now falsifiable; suite became required CI.
Senior signals: mutation-testing mindset ("prove the test can fail"); infra cost argued explicitly.
Resume bullet: *Introduced real-database integration testing that made concurrency invariants falsifiable in CI.*

## Story 9: Scaling real-time past one box
Prompts: "biggest technical challenge" / "designing for scale".
Source: [P1](../06-contribution-practice/03-senior-build-projects.md#p1-redis-backed-pubsub-multi-instance-real-time--2-weeks) + [D4 rehearsal](05-debugging-and-code-review-rounds.md#d4-replica-deploy-breaks-live-updates-25-min).
Skeleton: in-process pubsub silently capped deployment at one replica → diagnosed via sticky-session experiment, RFC'd Redis vs LISTEN/NOTIFY, kept service interface stable → multi-replica deploys with unchanged subscriber contracts.
Senior signals: experiment-first diagnosis; interface-stability as the migration enabler; delivery-guarantee documentation.
Resume bullet: *Designed the horizontal-scaling migration for a GraphQL-subscriptions pubsub layer.*

---

## Prompt coverage map

conflict (3), ambiguity (4, 7), mistake/wrong (5), technical tradeoff (2, 3, 9), complex learning (4), quality influence (8), backwards compat (6), scale (9), detail/rigor (1). Every common prompt has ≥1 story; rehearse your top four (recommended: 1, 4, 5, 9 — they span find/learn/humility/design).
