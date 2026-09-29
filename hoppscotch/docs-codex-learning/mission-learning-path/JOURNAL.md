# Mission Journal

## Design Rationale

The missions are ordered from orientation to ownership. The first missions teach where the app starts and how files are organized. The middle missions teach state, side effects, GraphQL contracts, and end-to-end tracing. The final missions ask you to think like an owner: find bugs before users do, audit security, write missing tests, and infer codebase evolution.

## Chosen Code Paths

The primary path is REST history sync because it is compact but full-stack:

- UI: `packages/hoppscotch-common/src/components/http/Request.vue:58-80`, `340-441`
- State: `packages/hoppscotch-common/src/newstore/history.ts:13-28`, `134-192`, `354-372`
- Sync/API: `packages/hoppscotch-selfhost-web/src/platform/history/web/sync.ts:26-60`, `api.ts:35-91`
- Backend: `packages/hoppscotch-backend/src/user-history/user-history.resolver.ts:26-170`, `user-history.service.ts:28-93`
- Database: `packages/hoppscotch-backend/prisma/schema.prisma:152-161`

Supporting paths include startup, routing, auth, GraphQL client helpers, workspace state, data schemas, and tests.

## Tier Skills

- Junior missions build navigation confidence and file recognition.
- Mid-level missions build cause/effect tracing and boundary awareness.
- Senior missions build risk detection, testing judgment, performance/security review, and architecture storytelling.

## Adapted Gaps

This repo does not use React, Redux, Zustand, Express-first REST, Next.js, TypeORM, or React Query as the main stack. Missions focus on Vue 3, custom stores, URQL, NestJS, GraphQL, Prisma, Zod/verzod, and Vitest/Jest instead.

