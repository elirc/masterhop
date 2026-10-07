# Maintainer Communication

Open source (and every senior colleague relationship) runs on one currency: **evidence of your own work before you spend their attention.**

## Asking for help without outsourcing thinking

Bad: "The backend won't start, help?"
Good structure — context, what you tried, hypothesis, narrow question:

> Setting up per CONTRIBUTING on Node 22 / pnpm 10. `pnpm install` fails in hoppscotch-backend's postinstall at `prisma generate` with <exact error>. I've confirmed the placeholder DATABASE_URL is set as in the script; generation succeeds if I pin prisma to X locally. Hypothesis: engine download blocked by my proxy. Is there a documented offline/proxy path, or is vendoring the engine the expected workaround?

Every sentence carries information; the question is answerable in one line. That's the whole art.

## Reporting a bug (repro report template)

```markdown
**What happens**: concurrent drag-reorders intermittently 500.
**Expected**: both reorders apply (any order).
**Repro**: steps or script — minimal; two parallel updateCollectionOrder
mutations against one parent. Fails ~1/5 runs.
**Evidence**: backend log excerpt: "Retrying updateOrderIndex... (3)"
then TEAM_COL_REORDERING_FAILED.
**Environment**: commit/tag, Node, single instance, Postgres 16.
**Suspicion (labeled)**: retry exhaustion under contention —
team-collection.service.ts#L602-L608. Happy to test a patch.
```

Minimal repro beats prose; a *labeled* suspicion beats a confident wrong diagnosis; "happy to test" converts you from cost to asset.

## Proposing a feature

Lead with the problem, not the solution: user story → evidence it's common (issues, forum links) → sketch of a fit-the-codebase approach ([M3's cap proposal](../06-contribution-practice/02-mid-level-feature-tickets.md#m3-cap-and-validate-mock-server-delayinms) is the model) → offer to implement behind a flag. Accept "no" gracefully — a rejected proposal with a good conversation still builds the relationship (and is itself a STAR story about disagreement).

## Responding to review

- Every comment gets a response: fixed (with commit link), pushback (with evidence), or question. Silence reads as ignored.
- Pushback template: *"Went the other way because <evidence: existing pattern / measurement / failing case>. Happy to switch if you still prefer X — you know the codebase's direction better."* Firm on evidence, soft on ego, explicit about who owns the tiebreak.
- Batch nit-fixes into one commit ("address review"); never force-push away the history a reviewer already read (append, then squash at merge if the repo prefers).

## Respectful disagreement (the escalation ladder)

1. Clarify — most disagreements are missing context.
2. Evidence — a failing test or benchmark ends debates prose can't.
3. Cost it — "your way adds a migration; mine adds a flag; both work" reframes from taste to tradeoff.
4. Concede or escalate *explicitly* — "not blocking for me, your call" or "I think this risks data loss; can we get a second maintainer?" Both are wins; simmering is the only loss.

Interview mapping: "tell me about disagreeing with a senior engineer" wants exactly ladder steps 1–3 with a concrete artifact. Build one by actually doing [Ticket 16](../06-contribution-practice/01-good-first-tickets.md#ticket-16-improve-the-verifyadmin-response-contract) (a design conversation, not a code PR) against upstream.
