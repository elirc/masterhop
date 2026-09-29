# Debugging and Code-Review Rounds — Timed Simulations

Seven simulations converted from earlier modules. Run them **timed, aloud**, ideally with a friend playing interviewer using the provided follow-ups. Interviewers grade the *search process* and communication, not just the answer.

## Format

Debugging rounds: 25 min — 5 restating/questions, 15 narrowing aloud, 5 fix + regression test. Review rounds: 30 min — read 10, write findings 15, discuss 5.

---

## D1: Env var works in URL, not in script (25 min)
Source: [Scenario 1](../05-quality-engineering/03-systematic-debugging.md#scenario-1-my-environment-variable-works-in-the-url-but-not-in-the-pre-request-script).
Interviewer follow-ups: "You can't reproduce locally — what do you ask the user for?" / "Now it's only secrets that fail — new hypothesis?"
Grading: named the boundary (sandbox injection vs resolution) in first 5 min = pass; diffed the two consumers' inputs = strong.

## D2: Intermittent 500 on concurrent reorder (25 min)
Source: [Scenario 2](../05-quality-engineering/03-systematic-debugging.md#scenario-2-two-teammates-drag-collections-simultaneously-one-gets-a-500).
Follow-ups: "Logs show retries exhausted at 3 — raise MAX_RETRIES?" (trap: maybe, but first ask *why* contention rose; jitter; and is a write path missing the lock) / "How do you repro concurrency deterministically in a test?"
Grading: strong = distinguishes expected-retry from exhaustion, proposes the barrier-based race test.

## D3: Works in app, fails in CI CLI (25 min)
Source: [Scenario 3](../05-quality-engineering/03-systematic-debugging.md#scenario-3-a-user-reports-my-test-script-passes-locally-but-fails-in-ci-via-hopp-test).
Follow-ups: "Same node version, same flags, still differs" (→ sandbox implementation diff; conformance-test pitch) / "You find both runners correct but docs ambiguous — what ships?"
Grading: strong = matrix method stated explicitly (hold script, vary runtime/flags one at a time).

## D4: Replica deploy breaks live updates (25 min)
Source: [Scenario 4](../05-quality-engineering/03-systematic-debugging.md#scenario-4-after-deploying-a-second-backend-replica-live-collaboration-randomly-breaks) — the best one to rehearse: it ends in an architecture conversation.
Follow-ups: "Redis is approved — what's your rollout?" / "What else breaks with 2 replicas?" (crons!)
Grading: strong = localized to delivery (not publish/render) via sticky-session experiment before reading any code.

## R1: The membership-cache PR (30 min)
Source: [Review Kata 5](../04-code-reading-gym/04-review-katas.md#kata-5-cache-team-membership-in-the-guard).
Follow-ups: "Author says 60s staleness is fine, product agrees — now what?" (bound it: invalidate on member_removed publish; per-instance caveat; document the window).
Grading: strong = identified authz staleness as *the* issue in 10 min and still offered a shippable path.

## R2: The MIME-downgrade removal PR (30 min)
Source: [Review Kata 6](../04-code-reading-gym/04-review-katas.md#kata-6-support-html-preview-for-mock-responses).
Follow-ups: "The CSP header still exists — isn't that enough?" (defense in depth; sandbox CSP bypass surface; nosniff interplay) / "Author is frustrated — deescalate."
Grading: strong = reconstructed *why the defense exists* from the code comments before objecting.

## R3: Live review of real code (30 min, no prep)
Open [team-collection.service.ts#L893-L1055](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L893-L1055) (`updateCollectionOrder` + `moveCollection` region — *not* covered in this curriculum's drills) cold and review it as if it were a PR. This simulates the "here's unfamiliar code, review it" round with genuinely unfamiliar code.
Grading: apply the [five layers](../07-career-and-collaboration/01-code-review-mindset.md#the-five-layers-review-in-this-order-stop-early-only-downward); strong = you checked it against the module's own conventions (lock? retry? publish placement? cast?) rather than abstract standards.

---

## Narration formula (all rounds)

1. Restate + one clarifying question. 2. State the layer split and which side evidence favors. 3. Name the cheapest next observation before making it. 4. Say findings as you go — silence reads as flailing. 5. End with: root cause, fix, regression test, and one systemic follow-up ("I'd also add a counter here"). That last beat is what converts a pass into a strong-hire signal.
