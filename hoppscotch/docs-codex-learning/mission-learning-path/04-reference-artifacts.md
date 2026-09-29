# Reference Artifacts

### Mission 25: Write The Docs That Don't Exist

**Tier:** Senior

**Time Estimate:** 90 minutes

**Goal:** Produce a concise architecture note from your own trace.

**The Concept:** Writing is engineering compression. If you can explain a system clearly, you probably understand its boundaries.

**Design Intent Before You Read the Code:** Good internal docs cite real files, explain trade-offs, and tell future engineers where to look first.

**Find It In The Code:** Use the REST history path: `Request.vue:340-441`, `history.ts:354-372`, `sync.ts:26-60`, `api.ts:51-91`, `user-history.resolver.ts:26-170`, `user-history.service.ts:60-93`, `schema.prisma:152-161`.

```text
UI -> state -> sync -> GraphQL -> resolver -> service -> Prisma -> subscription -> UI
```

**The Aha Moment:** Documentation is strongest when it follows a real feature flow.

**Socratic Checkpoint:** What did you omit? What would a new engineer misunderstand? What risk did you identify? What line ranges prove your claims? What test would protect the flow?

**How to Self-Grade:** Strong docs are accurate, cited, short enough to use, and honest about uncertainty.

**Connects To:** The user-story suite because clear docs become better implementation plans.

## Doc 1: Junior Onboarding Checklist

- Read `package.json:9-18`.
- Identify package ownership before editing.
- Trace one route from `modules/router.ts:12-111`.
- Read `Request.vue:58-80` and explain method/URL/send.
- Run package-level tests before root-wide tests when possible.
- First PR checklist: one behavior change, one test or clear test rationale, no unrelated formatting.
- Questions to ask: Which package owns this? Is data local or synced? What is the nearest existing pattern?

## Doc 2: Architecture Guide For New Engineers

Navigate by layer:

- Shell: `hoppscotch-selfhost-web`
- Shared UI/state: `hoppscotch-common`
- Domain schemas: `hoppscotch-data`
- Backend: `hoppscotch-backend`
- Persistence: Prisma schema and migrations

Do not get lost by following every import. Follow one user action at a time.

## Doc 3: Code Review Checklist

- Check package boundary first.
- Check schema/version changes.
- Check auth/ownership.
- Check local store and sync echo behavior.
- Check generated GraphQL contract.
- Check tests near the behavior.
- Check failure states and loading states last.

## Doc 4: Debugging Playbook

For missing history:

1. Does `newSendRequest` run? `Request.vue:340-441`
2. Does `executedResponses$` add history? `history.ts:354-372`
3. Does sync call mutation? `sync.ts:26-46`
4. Does GraphQL return error? `GQLClient.ts:210-270`
5. Does backend create Prisma row? `user-history.service.ts:69-77`

## Doc 5: Change Playbook

Branch -> locate owner -> study nearest pattern -> plan files -> implement narrow change -> add tests -> run package test/typecheck -> review diff -> open PR.

For this repo, watch generated GraphQL files and postinstall codegen. Avoid manual edits to generated outputs unless the repo expects them.

## Doc 6: Senior Ownership Notes

Monitor:

- Auth refresh failures.
- GraphQL subscription duplication.
- Prisma migration quality.
- History sync and collection sync regressions.
- Setup/codegen friction.

Improve:

- Ownership predicates in service methods.
- Safe JSON parse handling.
- Local setup docs.
- Integration coverage for sync flows.

Leave alone unless measured:

- Broad store rewrites.
- Replacing Vue/URQL/custom stores with trendier libraries.
- Normalizing all request JSON into relational tables.

## Doc 7: Interview Walkthrough

Practice answer:

"I studied Hoppscotch, a full-stack pnpm monorepo. The shared Vue app composes API request screens, platform shells inject web/desktop capabilities, shared Zod/verzod schemas maintain request compatibility, and a NestJS GraphQL backend persists user data through Prisma. I traced REST history from the send button to local state, GraphQL sync, backend resolver/service, Prisma JSON storage, and subscriptions back to clients."

Follow-up trade-off:

"JSON storage supports flexible API request shapes, while versioned schemas preserve compatibility. The risk is weaker database validation, so I would harden parse errors, ownership checks, and indexes before changing the storage model."

