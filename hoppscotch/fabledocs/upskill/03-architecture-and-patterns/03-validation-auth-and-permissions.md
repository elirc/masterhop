# Validation, Auth, and Permissions

## The validation layers, mapped

A request into the backend crosses up to five checks. Know each one's job:

| Layer | Mechanism | Example | Catches |
| --- | --- | --- | --- |
| 1. Rate limit | `ThrottlerBehindProxyGuard` (controller-wide) | [auth.controller.ts#L34](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L34) | abuse volume |
| 2. Shape/DTO | class-validator DTOs (REST), GraphQL scalar/type system | [SignInMagicDto](../../../packages/hoppscotch-backend/src/auth/dto/signin-magic.dto.ts); GraphQL args typed at [team.resolver.ts#L188-L190](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L188-L190) | malformed input |
| 3. AuthN | `GqlAuthGuard` / `JwtAuthGuard` (JWT from cookie) | [team.resolver.ts#L186](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L186) | anonymous callers |
| 4. AuthZ | `GqlTeamMemberGuard` + `@RequiresTeamRole` | [gql-team-member.guard.ts#L21-L43](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L21-L43) | wrong tenant / wrong role |
| 5. Business validation | service checks | title length + parent ownership ([team-collection.service.ts#L458-L465](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L458-L465)); JSON validity (L467-L472); single-owner rule ([team.service.ts#L219-L221](../../../packages/hoppscotch-backend/src/team/team.service.ts#L219-L221)) | legal-but-wrong operations |

The frontend ALSO validates (forms, zod parsing of imports) — treat that as UX, never as security. Say this sentence in every interview: **client validation is a convenience; server validation is the contract.**

## AuthN: how identity is established

- Cookie-carried JWTs (`access_token` ~1 day, `refresh_token` ~7 days) set by `authCookieHandler` ([auth.controller.ts#L80](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L80)); Passport strategies in [auth/strategies](../../../packages/hoppscotch-backend/src/auth/strategies) resolve them to `req.user`.
- Refresh rotation with argon2-hashed storage ([Pattern 17](05-pattern-catalog.md#pattern-17-refresh-token-rotation-with-hashed-storage)).
- Magic-link + Google/GitHub/Microsoft SSO; provider identities in `Account` rows keyed `[provider, providerAccountId]` ([schema.prisma#L131](../../../packages/hoppscotch-backend/prisma/schema.prisma#L131)).
- Non-browser callers: PersonalAccessTokens for the CLI ([access-token module](../../../packages/hoppscotch-backend/src/access-token)); InfraTokens for instance admin APIs.
- Desktop gets tokens via a localhost redirect, validated by [redirect-uri.validator.ts](../../../packages/hoppscotch-backend/src/auth/redirect-uri.validator.ts) (checked at [auth.controller.ts#L208-L213](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L208-L213)) — the validator exists to stop open-redirect token theft; it even has its own spec file. That's what "security-critical 20 lines" looks like.

## AuthZ: tenant/resource isolation (the IDOR section)

The isolation question: *when a request names a resource ID, who checks the caller may touch it?*

- Team-scoped ops with an explicit `teamID` arg: the guard resolves caller membership ([gql-team-member.guard.ts#L34-L42](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L34-L42)).
- Ops naming a `collectionID` instead: a collection-specific guard must first resolve collection→team, then check membership ([team-collection/guards](../../../packages/hoppscotch-backend/src/team-collection/guards)) — **the classic IDOR seam**: any resolver that takes a child-resource ID and forgets the ownership walk is a cross-tenant hole. Review checklist: for every new mutation, name the ID it receives and point at the line resolving it to a tenant.
- Cross-team writes blocked in services too: `createCollection` verifies the parent belongs to the same team ([team-collection.service.ts#L462-L465](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L462-L465)) — without this, you could graft your collection into someone else's tree.
- Subscriptions carry the same guards ([team.resolver.ts#L310-L316](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L310-L316)) — checked at subscribe time, not per event (**possible risk** — removed members and live sockets; verify before claiming).

## What a junior misses vs what a senior checks

| Junior misses | Senior checks |
| --- | --- |
| authz on *reads* ("it's just a query") | every query resolver's guard list |
| child-resource IDOR (collectionID, requestID) | the ID→tenant resolution line, per endpoint |
| GET endpoints with side effects | [verify/admin auto-promotion](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L371-L387) — found one |
| trusting arg names | the guard's hard dependency on the literal arg `teamID` |
| "validation = zod" | which layer *owns* each rule, and rules enforced twice on purpose |

## Drills

1. Pick any resolver in [team-request](../../../packages/hoppscotch-backend/src/team-request) — write down its five-layer table row by row from the code.
2. Red-team exercise (on paper only): construct the GraphQL call a malicious EDITOR of team A would send to read team B's environment, and find the exact line that stops it.
3. Explain why `logout` ([auth.controller.ts#L186-L191](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L186-L191)) needs no guard — and what a guard there would break (logging out with an expired access token).

Interview angle: authn vs authz definitions with the guard/service split; IDOR walk-through; the whole of [05-quality-engineering/05-security-checklist.md](../05-quality-engineering/05-security-checklist.md) builds on this file.
