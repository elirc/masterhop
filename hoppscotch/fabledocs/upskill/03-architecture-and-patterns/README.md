# 03 — Architecture and Patterns

The mid-level module: not "where is it" but "is it the right shape, and what breaks it."

1. [01-boundaries-and-layers.md](01-boundaries-and-layers.md) — who owns what; good boundaries and leaks
2. [02-data-model-and-persistence.md](02-data-model-and-persistence.md) — schema, invariants, transactions, safe change
3. [03-validation-auth-and-permissions.md](03-validation-auth-and-permissions.md) — every validation layer; authn vs authz; isolation
4. [04-side-effects-async-and-reliability.md](04-side-effects-async-and-reliability.md) — events, retries, idempotency, failure visibility
5. [05-pattern-catalog.md](05-pattern-catalog.md) — 18 recognition cards
6. [06-architecture-critique.md](06-architecture-critique.md) — strengths, risks, what I'd change owning this for 3 months (doubles as system-design interview prep)

Vocabulary contract for the module: **invariant** (must stay true), **boundary** (ownership/trust change), **contract** (agreed shape across a boundary), **blast radius** (what breaks if this breaks), **idempotency** (repeat = same result), **isolation** (tenant A can't see B). Use them precisely; interviewers do.
