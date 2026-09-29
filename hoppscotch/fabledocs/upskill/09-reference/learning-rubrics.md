# Learning Rubrics

Observable behaviors, not vibes. Grade against what you can *do*, demonstrated on this repo. The **Interview-ready** column is the externally-visible version of each skill.

## Codebase navigation

| Level | Observable behavior | Interview-ready when |
| --- | --- | --- |
| Junior | Finds where a feature lives within 15 min using grep + the system map | You can narrate your search strategy aloud |
| Mid | Predicts file locations from conventions before searching; knows which of the 12 packages owns any concern | 60-second repo summary is fluid, unprompted |
| Senior | Reads a new module and produces its boundary statement ("owns X, must not know Y") in one pass | You can do [R3 (cold review)](../08-interview-prep/05-debugging-and-code-review-rounds.md#r3-live-review-of-real-code-30-min-no-prep) at Strong |

## Language & stack (JS/TS/Vue/Nest)

| Level | Observable behavior | Interview-ready when |
| --- | --- | --- |
| Junior | Explains async ordering in [network.ts](../../../packages/hoppscotch-common/src/helpers/network.ts) correctly after study | 8+ of the [JS/TS cards](../08-interview-prep/01-js-ts-node-deep-dive.md) at Mid level |
| Mid | Writes a typed, Either-returning service method matching house style without copying; explains every guard/decorator on a resolver | All 14 cards at Mid, 5+ at Senior |
| Senior | Argues framework tradeoffs with costs (RxJS vs promises, guards-vs-service checks) and knows when the house pattern is wrong | You volunteer failure modes without being asked |

## Data & persistence

| Level | Observable behavior | Interview-ready when |
| --- | --- | --- |
| Junior | Reads schema.prisma and states each `@@unique` as an English invariant | Can draw the entity map from memory |
| Mid | Designs an additive zero-downtime migration ([drill](../03-architecture-and-patterns/02-data-model-and-persistence.md#drills)); explains the lock/retry machinery | [Q7–Q10 API cards](../08-interview-prep/03-api-and-data-modeling-questions.md) at Mid |
| Senior | Completes [K5 (fractional-key migration)](../06-contribution-practice/04-refactor-and-design-katas.md#k5-migration-design-kata--2h-paper) at Strong; can defend the Json-column split | You can teach the NULL-in-unique caveat correctly |

## Debugging

| Level | Observable behavior | Interview-ready when |
| --- | --- | --- |
| Junior | Follows a written narrowing path ([scenarios](../05-quality-engineering/03-systematic-debugging.md)) and lands the root cause | You state hypotheses as "bug is in X because Y" |
| Mid | Constructs the narrowing path yourself on scenarios 1–3; names the cheapest probe first | Timed [D1/D2](../08-interview-prep/05-debugging-and-code-review-rounds.md) pass grade |
| Senior | Recognizes not-a-bug (Scenario 5) fast; ends every debug with a regression test and a systemic follow-up | D4 lands the architecture conversation unprompted |

## Review & collaboration

| Level | Observable behavior | Interview-ready when |
| --- | --- | --- |
| Junior | Findings correct but unlayered; some blocking/optional confusion | Comments follow observation→risk→path |
| Mid | [Kata](../04-code-reading-gym/04-review-katas.md) headline issues caught; severity calibrated to layers | Katas 1–4 at Solid |
| Senior | Catches the judgment katas (2, 5, 6, 8) including when to *unblock*; writes RFC-quality tradeoff analyses | Kata 8 in 30 min at Strong |

## Security

| Level | Observable behavior | Interview-ready when |
| --- | --- | --- |
| Junior | Explains authn vs authz with the guard example | — |
| Mid | Applies the [pre-merge checklist](../05-quality-engineering/05-security-checklist.md#pre-merge-security-checklist-use-on-every-pr-you-write-here) unprompted; explains the mock-server defenses | Three security exhibits rehearsed |
| Senior | Reasons about storage asymmetries (hashed RT vs raw PAT), origin splits, and staleness windows as design dimensions | You can red-team your own feature designs ([M2's secret-payload question](../06-contribution-practice/02-mid-level-feature-tickets.md#m2-team-environment-change-events)) |

## Self-assessment checklist (run at week 2, 4, 8)

- ☐ I can trace 3 key flows from memory, naming files.
- ☐ I can state 5 invariants and where each is enforced.
- ☐ I've completed ≥4 annotation drills at Solid+.
- ☐ I've done ≥2 tickets end to end with design notes.
- ☐ I can deliver the system-design walkthrough in 40 min.
- ☐ I have 4 rehearsed STAR stories under 2 minutes each.
- ☐ My answers reflexively include example + tradeoff + failure mode.

Mindset check per the ladder: if your notes say *how* more than *whether* and *what it costs*, you're still practicing junior questions on senior material.
