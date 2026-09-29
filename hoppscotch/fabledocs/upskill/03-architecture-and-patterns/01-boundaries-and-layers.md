# Boundaries and Layers

A boundary is where ownership and trust change. Layers are boundaries stacked. This file names each layer in Hoppscotch, what it owns, what it must not own, and where the lines blur.

## Backend layers

| Layer | Owns | Must not own | Exhibit |
| --- | --- | --- | --- |
| Controller / Resolver | HTTP/GraphQL shape, guards, arg extraction, error translation | business rules, DB access | [team.resolver.ts#L183-L195](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L183-L195) — 12 lines: decorate, delegate, translate |
| Guard | "may this caller invoke this?" | state-change legality | [gql-team-member.guard.ts](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts) |
| Service | business rules, invariants, transactions, events | HTTP concepts (status codes...) | [team-collection.service.ts#L452-L517](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L452-L517) |
| Prisma/DB | constraints, cascades, atomicity | business vocabulary | [schema.prisma](../../../packages/hoppscotch-backend/prisma/schema.prisma) |

**A leak worth studying**: services returning `RESTError` objects with `statusCode` ([auth.service.ts#L120-L124](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L120-L124)) — HTTP status is a *controller* concept that has crept into the service layer. Compare with team services returning bare error-code strings and letting resolvers map them. Both exist in one codebase; naming which is cleaner and why (services should be transport-agnostic — the same service serves GraphQL and REST) is exactly a mid-level interview answer.

**A deliberate double-check that is NOT a leak**: guards check membership, then services re-check business invariants (single-owner rule). Different questions, different layers ([Pattern 4](05-pattern-catalog.md#pattern-4-invariant-re-check-in-the-service-defense-in-depth)).

## Frontend layers

| Layer | Owns | Must not own | Exhibit |
| --- | --- | --- | --- |
| Components | rendering, user intent | network calls, business transforms | [components/http](../../../packages/hoppscotch-common/src/components/http) |
| Services (dioc) / stores (newstore) | client state, orchestration | transport details | [kernel-interceptor.service.ts](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts) |
| Helpers | pure-ish domain logic (env substitution, runner pipeline) | component state | [RequestRunner.ts](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts) |
| Platform (per-shell) | auth/sync/storage implementations | UI | [selfhost-web/src/platform](../../../packages/hoppscotch-selfhost-web/src/platform) |
| Kernel | typed contracts to native/transport | anything Vue | [hoppscotch-kernel/src](../../../packages/hoppscotch-kernel/src) |

**The strongest boundary in the repo**: common → platform. `hoppscotch-common` never imports a shell; shells inject implementations ([Pattern 12](05-pattern-catalog.md#pattern-12-platform-injection-dependency-inversion-at-package-scale)). This single decision is what makes web/desktop/self-host from one codebase possible.

**A blurry seam**: `RequestRunner.ts` is a 1100-line helper that orchestrates UI state (tab documents), sandbox execution, env stores, and networking. It respects package boundaries but concentrates four concerns in one file — the "helper that became an organ." Refactor kata material ([06-.../04-refactor-and-design-katas.md](../06-contribution-practice/04-refactor-and-design-katas.md)).

## Package-level dependency rules

Allowed direction (verify with imports): `data ← common ← shells`; `data ← js-sandbox ← common/cli`; `kernel ← common/desktop`. `data` imports nothing internal — it is the bottom. The backend shares **no runtime code** with the frontend except through the GraphQL contract and the `data` package's JSON formats. That independence is why backend refactors don't ripple into the UI — and why the *JSON in the DB* (owned by `data`, stored by backend, interpreted by frontend) is the subtlest shared contract in the system.

## Drills

1. Find one more service that mentions HTTP status codes (grep `HttpStatus.` in `src/*/**.service.ts`) and one that doesn't; write a one-line rule you'd propose in a team style guide.
2. Attempt to name what would break if `hoppscotch-common` imported from `hoppscotch-selfhost-web` (circular workspace dep; desktop build carrying web sync code; the platform contract dissolving).
3. For `RequestRunner.ts`, sketch a 3-module split (script-execution, env-resolution, response-application) and the function signatures between them.

Interview angle: "Describe a well-layered backend" — use the 4-row table verbatim. "Tell me about a boundary violation you've seen" — the `statusCode`-in-service leak, including why it's *mild* (it's still data, not behavior).
