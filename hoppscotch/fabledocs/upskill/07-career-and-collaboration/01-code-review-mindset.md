# Code Review Mindset

## The five layers (review in this order; stop early only downward)

1. **Does it work?** — happy path, obvious breakage.
2. **Is it correct?** — edge cases, concurrency, error paths. *This repo's specifics: ordering writes without the lock; Either Lefts unmapped; publish inside transactions.*
3. **Will it stay correct?** — tests that would fail on regression; constraints; types at boundaries.
4. **Does it fit?** — the module's existing patterns (guard parity, `cast()`, error constants, `do-*` scripts, versioned data changes).
5. **Is it kind to future maintainers?** — names that tell the truth ([the `hashedRefreshToken` counterexample](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L335)), comments that explain *why* ([the mock-server defense comments](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L142-L144) are the gold standard — they saved Kata 6's reviewer).

## Repo-specific review checklist

- ☐ New resolver/route: guards match most-restrictive sibling; `teamID`-vs-`collectionID` guard variant correct ([authz map](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)).
- ☐ Ordering/tree writes: inside transaction, behind the lock, retry allow-list respected.
- ☐ Events: publish after commit; topic string matches `TopicDef`; payload contains no secrets.
- ☐ Errors: constants from `errors.ts`; service returns Either; controller/resolver translates once.
- ☐ Data-shape changes: schema.prisma delta reviewed as SQL; blob-shape changes get a data-package version, never an in-place edit.
- ☐ Security-relevant deletion or GET-with-side-effect: escalate, don't approve solo.
- ☐ Tests: the *not-called* assertions present on rejection paths ([Recipe 2](../05-quality-engineering/02-writing-tests-here.md#recipe-2-validation-failure)).

## Comment language that works

Pattern: **observation (anchored) → risk → path → question mark where honest.**

> This `updateMany` runs outside the lock that `createCollection` takes (L479) — two concurrent reorders could interleave. Could we move it inside the locked transaction like the delete path does (L563)?

> Nit (non-blocking): `hashedRefreshToken` here actually holds the raw cookie token — rename to avoid the next reader's 10 confused minutes?

Anti-patterns: "this is wrong" (no path), "why didn't you just…" (status move), style comments during a correctness review (wrong layer), approving with 12 unresolved "thoughts" (decide: blocking or not).

Calibration drill: re-grade your [review katas](../04-code-reading-gym/04-review-katas.md) answers against layer numbers — every Blocking finding should live in layers 1–2, Important in 3–4, Optional in 5. If your Blockings are layer-5 items, you're the reviewer people dread; recalibrate.
