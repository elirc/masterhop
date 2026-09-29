# Two-Week Cram Plan

Assumes ~2.5h/day weekdays, 4h/day weekends. Everything references materials in this curriculum. **Aloud** means spoken, standing, timed — silent reading does not build interview fluency.

## Week 1 — build the evidence base

**Day 1** — [00-fast-track.md](../00-fast-track.md) sections 1–3 (skip the code change). Rehearse the [60-second repo summary](README.md#your-60-second-repo-summary-rehearse-until-fluid) aloud ×5.
**Day 2** — [Key Flows](../01-codebase-cartography/05-key-flows.md) 1, 3, 5 with code open. Trace Table [1](../04-code-reading-gym/02-trace-tables.md#trace-1-uinetwork-baseurlusers-from-editor-to-wire) and [3](../04-code-reading-gym/02-trace-tables.md#trace-3-auth-refresh-token-rotation) on paper.
**Day 3** — JS/TS cards [Q1–Q7](01-js-ts-node-deep-dive.md): read anchor → answer aloud 90s each → check. Log weak ones.
**Day 4** — JS/TS cards Q8–Q14 same method. Redo Day 3's weak ones.
**Day 5** — API/data cards [Q1–Q8](03-api-and-data-modeling-questions.md). Then Flow 4 + the [validation map](../03-architecture-and-patterns/03-validation-auth-and-permissions.md) — authz must be reflexive by now.
**Day 6 (weekend)** — Full [system design walkthrough](04-system-design-from-this-repo.md): whiteboard it alone, 40 min timed, then read the file and diff your run. Afternoon: STAR stories [1, 4, 5](06-behavioral-star-stories.md) drafted in your own words.
**Day 7 — checkpoint** — Self-assess against [learning rubrics](../09-reference/learning-rubrics.md): ☐ repo summary fluid ☐ 3 flows traceable from memory ☐ 10+ cards at Solid ☐ 2 STAR stories under 2 min. Anything unchecked is Day 8–9's priority. Rest otherwise.

## Week 2 — pressure and polish

**Day 8** — API/data cards Q9–Q15. Frontend cards [Q1–Q6](02-frontend-framework-questions.md).
**Day 9** — Frontend Q7–Q12. Timed debugging round [D1](05-debugging-and-code-review-rounds.md#d1-env-var-works-in-url-not-in-script-25-min), aloud, 25 min hard stop.
**Day 10** — Timed rounds [D4](05-debugging-and-code-review-rounds.md#d4-replica-deploy-breaks-live-updates-25-min) + review [R2](05-debugging-and-code-review-rounds.md#r2-the-mime-downgrade-removal-pr-30-min). These two are the highest-yield simulations — D4 chains into system design, R2 into security.
**Day 11** — Mock system design **with a human** (or recorded): base walkthrough + two variation prompts from [Step 5](04-system-design-from-this-repo.md#step-5--scaling-and-evolution-prompts). Watch/listen to yourself once. Painful; do it anyway.
**Day 12** — STAR stories [3, 9](06-behavioral-star-stories.md) + full behavioral run: 6 prompts from the coverage map, 2 min each, aloud. Re-run weakest technical cards.
**Day 13 (weekend)** — Dress rehearsal, full loop order: 60s summary → 6 mixed cards → one 25-min debugging round ([D2](05-debugging-and-code-review-rounds.md#d2-intermittent-500-on-concurrent-reorder-25-min)) → 40-min system design → 3 STAR prompts. ~3h with breaks. Note every stumble.
**Day 14 — final checkpoint + taper** — Fix only the top 3 stumbles from Day 13. Re-verify: ☐ summary ☐ critique P1/P2 explainable with fixes ☐ golden rule reflex (example + tradeoff + failure mode in every answer). Then stop — sleep beats one more card.

## Standing rules

- Every answer, every day: **example + tradeoff + failure mode**. If you gave a definition, redo it.
- When a card exposes a gap, open the anchor — never patch with memorized prose; the anchor is why your answers sound different from everyone else's.
- Keep a stumble log; study the log, not your comfort zone.
- Day-of: 60s summary in the car, nothing else.
