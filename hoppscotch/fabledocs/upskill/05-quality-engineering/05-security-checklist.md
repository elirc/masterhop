# Security Checklist

Each risk class mapped to where this repo handles it (or where to verify). This file assumes [03-.../03-validation-auth-and-permissions.md](../03-architecture-and-patterns/03-validation-auth-and-permissions.md).

| Class | This repo | Status |
| --- | --- | --- |
| Authorization / IDOR | Guard-per-endpoint + resource→tenant resolution ([validation map](../03-architecture-and-patterns/03-validation-auth-and-permissions.md#authz-tenantresource-isolation-the-idor-section)) | Strong pattern; per-endpoint audit still required on every new resolver |
| Input validation | DTOs (REST), GraphQL types, service-level checks, zod at data boundaries | Layered; JSON blob contents validated client-side only — server stores blind (**by design**, know why) |
| XSS | Mock server: header blocklist + MIME downgrade + CSP sandbox + nosniff ([mock-server.controller.ts#L18-L182](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L18-L182)); frontend: Vue escaping by default — audit any `v-html` (grep it) | The mock path is the crown exhibit; frontend `v-html` audit is homework |
| CSRF | Cookie-based auth ⇒ CSRF-relevant; mitigations to verify: cookie `SameSite` flags in `authCookieHandler` ([auth/helper.ts](../../../packages/hoppscotch-backend/src/auth/helper.ts)) and CORS origin allow-list (`WHITELISTED_ORIGINS`) | **Verify before claiming safe** — read helper.ts cookie options yourself |
| SSRF | The product's whole purpose is "server sends requests to arbitrary URLs"? No — *clients/agents* send; the backend itself doesn't fetch user URLs in the paths read. Mock server responds, never fetches | Re-audit if any server-side "call this URL" feature lands (webhooks!) |
| Open redirect | Desktop auth callback restricted to localhost URIs ([redirect-uri.validator.ts](../../../packages/hoppscotch-backend/src/auth/redirect-uri.validator.ts), used at [auth.controller.ts#L208-L213](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L208-L213)); SSO `redirect_uri` flows through state — verify allow-listing in [stateless-state-store.ts](../../../packages/hoppscotch-backend/src/auth/stateless-state-store.ts) | Half verified; finish the audit |
| Injection (SQL) | Prisma parameterization everywhere; raw SQL exists only in the lock helper (`lockTeamCollectionByTeamAndParent` — read [prisma service](../../../packages/hoppscotch-backend/src/prisma) and confirm parameters aren't interpolated) | Audit the one raw query — that's always the rule |
| Secrets at rest | Refresh tokens argon2-hashed ([auth.service.ts#L114](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L114)); InfraConfig rows flagged `isEncrypted` with `DATA_ENCRYPTION_KEY` ([schema.prisma#L221](../../../packages/hoppscotch-backend/prisma/schema.prisma#L221)); PAT/InfraToken values stored **raw** ([schema.prisma#L229](../../../packages/hoppscotch-backend/prisma/schema.prisma#L229), [L240](../../../packages/hoppscotch-backend/prisma/schema.prisma#L240) — `@unique` lookup requires it) — DB leak exposes live tokens. **Team env variables** (`variables Json`, [L91](../../../packages/hoppscotch-backend/prisma/schema.prisma#L91)) may contain API keys stored unencrypted — investigate | The PAT-vs-refresh-token asymmetry is a top-tier interview observation: hashed tokens can't be looked up by value; lookup-by-token forces raw or HMAC-indexed storage |
| Rate limiting | `ThrottlerBehindProxyGuard` on auth and mock controllers ([auth.controller.ts#L34](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L34), [mock-server.controller.ts#L47](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L47)); `@SkipThrottle` on SSO callbacks and subscriptions | "Behind proxy" = trusts forwarded IP headers — misconfigured proxy = bypassable limits; deployment concern |
| Dependency risk | pnpm `overrides` pinning vulnerable transitive deps + `onlyBuiltDependencies` install-script allow-list ([package.json#L36-L68](../../../package.json#L36-L68)) | Genuinely good; cite it |
| Uploads | Import flows parse user JSON/CSV client-side (zod) and CLI-side ([cli test.ts#L53-L93](../../../packages/hoppscotch-cli/src/commands/test.ts#L53-L93)) | Parser DoS (huge files) mostly a local concern |
| Untrusted code | The sandbox ([Flow 2](../01-codebase-cartography/05-key-flows.md#flow-2-running-a-user-test-script-sandboxasync-flow)); gap: no web-path timeout | The product's hardest security surface |
| Cookies | flags set centrally in `authCookieHandler` | Read it; know httpOnly/secure/sameSite values cold before any auth interview |

## Pre-merge security checklist (use on every PR you write here)

- ☐ Every new resolver/route: guard list matches its most-restrictive sibling; resource IDs resolve to a tenant before use.
- ☐ Any new query: parameterized; if raw, reviewed by a second person.
- ☐ Any new stored value: classified (secret? hash it / encrypt it / justify raw) — cite the PAT asymmetry if raw.
- ☐ Any user-controlled output: escaping/content-type story written down (mock-server file is the template).
- ☐ Any new redirect/callback: target allow-listed.
- ☐ Any new endpoint: throttle posture chosen deliberately (inherit / skip / custom) and stated in the PR.
- ☐ GET endpoints mutate nothing (the repo has one counterexample — don't add a second).
- ☐ Errors reveal no more to non-members than to members (existence leaks).

Interview angle: this table *is* the "security round" for a mid-level fullstack loop. Rehearse three exhibits: mock-server XSS defenses, refresh-token hashing vs PAT raw storage (and *why* the asymmetry is forced), and the IDOR guard pattern.
