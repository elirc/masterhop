# Domain Glossary

Product nouns, their meaning here, and where they live in code. Confusables are flagged — misusing these in a PR or interview answer signals you haven't read the code.

| Term | Means here | Code home |
| --- | --- | --- |
| **Request** | A saved, versioned description of an HTTP/GraphQL call (not an in-flight call) | [hoppscotch-data/src/rest](../../../packages/hoppscotch-data/src/rest); persisted as Json ([TeamRequest.request](../../../packages/hoppscotch-backend/prisma/schema.prisma#L64)) |
| **Effective request** | The request *after* env substitution and script mutation — what actually goes on the wire | [EffectiveURL.ts](../../../packages/hoppscotch-common/src/helpers/utils/EffectiveURL.ts) |
| **Collection** | Ordered tree of folders + requests; exists in *user* and *team* flavors with separate tables | [UserCollection](../../../packages/hoppscotch-backend/prisma/schema.prisma#L197-L213) / [TeamCollection](../../../packages/hoppscotch-backend/prisma/schema.prisma#L42-L57) |
| **Environment** | Named set of variables (`<<var>>` syntax); *global* env is a special one; also user vs team flavors | [hoppscotch-data/src/environment](../../../packages/hoppscotch-data/src/environment); [UserEnvironment.isGlobal](../../../packages/hoppscotch-backend/prisma/schema.prisma#L168) |
| **Workspace** | Where you're working: personal or a team — an enum, not a table | [WorkspaceType](../../../packages/hoppscotch-backend/prisma/schema.prisma#L321-L324); frontend [workspace.service.ts](../../../packages/hoppscotch-common/src/services/workspace.service.ts) |
| **Team** | The tenancy unit; members carry a `TeamAccessRole` | [schema.prisma#L10-L28](../../../packages/hoppscotch-backend/prisma/schema.prisma#L10-L28) |
| **Access role** | OWNER / EDITOR / VIEWER — per team, not global | [TeamAccessRole](../../../packages/hoppscotch-backend/prisma/schema.prisma#L331-L335) |
| **Interceptor** | A transport strategy for sending requests (browser/extension/agent/native) — *not* a NestJS interceptor | [kernel-interceptor.service.ts#L56-L69](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L56-L69) |
| **Kernel** | The typed contract layer between the shared app and its shells (relay/io/store) | [hoppscotch-kernel/src](../../../packages/hoppscotch-kernel/src) |
| **Relay** | The kernel's request-execution capability; `RelayRequest`/`RelayResponse` are the wire-level shapes | [hoppscotch-kernel/src/relay](../../../packages/hoppscotch-kernel/src/relay) |
| **Agent** | A local desktop daemon acting as an interceptor (bypasses CORS) | [hoppscotch-agent](../../../packages/hoppscotch-agent) |
| **Pre-request script / test script** | User JS run before/after a request, in the sandbox; "test" ≈ post-request assertions | [RequestRunner.ts#L520-L605](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L520-L605) |
| **Sandbox / cage** | Isolated script runtime; "cage" = faraday-cage instance with injected modules | [hoppscotch-js-sandbox](../../../packages/hoppscotch-js-sandbox) |
| **Lens** | A response-viewer strategy (JSON viewer, image viewer, …) | [hoppscotch-common/src/helpers/lenses](../../../packages/hoppscotch-common/src/helpers/lenses) |
| **Shortcode** | A shareable short ID for a request snapshot (`hopp.sh/r/abc`) | [shortcode module](../../../packages/hoppscotch-backend/src/shortcode); [Shortcode model](../../../packages/hoppscotch-backend/prisma/schema.prisma#L75-L85) |
| **Mock server** | User-defined fake API served publicly by the backend under a subdomain or `/mock/{id}` path | [mock-server module](../../../packages/hoppscotch-backend/src/mock-server) |
| **Published docs** | Generated, versioned API documentation from a collection | [PublishedDocs](../../../packages/hoppscotch-backend/prisma/schema.prisma#L299-L319) |
| **Magic link** | Passwordless email sign-in; token bound to a `deviceIdentifier` | [auth.service.ts#L206-L326](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L206-L326) |
| **PAT / infra token** | PersonalAccessToken = user-scoped API token (CLI auth); InfraToken = instance-admin token | [schema.prisma#L225-L244](../../../packages/hoppscotch-backend/prisma/schema.prisma#L225-L244) |
| **InfraConfig** | Runtime instance configuration stored in DB (not env vars), some encrypted | [schema.prisma#L215-L223](../../../packages/hoppscotch-backend/prisma/schema.prisma#L215-L223); [infra-config module](../../../packages/hoppscotch-backend/src/infra-config) |
| **Self-host / AIO** | Deployment modes; AIO = all-in-one container ([aio_run.mjs](../../../aio_run.mjs), Caddyfiles at root) | root deploy files |

## Confusables

- **Interceptor** (transport strategy, frontend) vs **NestJS `@UseInterceptors`** (server middleware, e.g., [UserLastLoginInterceptor](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L116)). Same word, unrelated concepts, both in this repo.
- **`newstore`** is the *legacy* state layer; **`services/` (dioc)** is the newer one. The name lies. ([newstore/](../../../packages/hoppscotch-common/src/newstore) vs [services/](../../../packages/hoppscotch-common/src/services))
- **Request** (saved entity) vs **effective request** (wire-ready) vs **RelayRequest** (kernel format) — three shapes, two transformations ([RESTRequest.toRequest](../../../packages/hoppscotch-common/src/helpers/network.ts#L25)).
- **User collection** vs **team collection** — parallel table hierarchies and parallel backend modules (`user-collection/`, `team-collection/`); features often land in one and lag in the other. Check both before claiming something is missing.
- **Test** (user's post-request script) vs **test** (vitest/jest). "The tests fail" is ambiguous in this repo — say which.
- **`v` in data** (schema version of an entity) vs **API version** (`/v1/auth`). Unrelated version counters.

Drill: without looking, write the storage table for: a starred history entry, a team env variable, a mock server hit log. Then verify ([UserHistory](../../../packages/hoppscotch-backend/prisma/schema.prisma#L152-L161), [TeamEnvironment](../../../packages/hoppscotch-backend/prisma/schema.prisma#L87-L93), [MockServerLog](../../../packages/hoppscotch-backend/prisma/schema.prisma#L267-L286)).
