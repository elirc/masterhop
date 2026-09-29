# Data Model and Persistence

One schema file governs everything: [schema.prisma](../../../packages/hoppscotch-backend/prisma/schema.prisma), 342 lines, PostgreSQL, client generated to a custom path (L1-L4).

## Entity map

```
User (uid) ─┬─< TeamMember(role) >── Team ──< TeamCollection(tree) ──< TeamRequest
            ├─< UserCollection(tree) ──< UserRequest        └──< TeamEnvironment
            ├─< UserEnvironment / UserHistory / UserSettings
            ├─< Account(provider) / VerificationToken / PersonalAccessToken
            └─< Shortcode / MockServer ──< MockServerLog / MockServerActivity
InfraConfig / InfraToken / InvitedUsers / PublishedDocs   (instance-level)
```

Two parallel hierarchies — user vs team collections ([L197-L213](../../../packages/hoppscotch-backend/prisma/schema.prisma#L197-L213) vs [L42-L73](../../../packages/hoppscotch-backend/prisma/schema.prisma#L42-L73)) — duplicated on purpose: different owners (uid vs teamID), different guards, same shape. Cost: every feature ships twice (see `mockExamples Json?` present on both request tables, L65 and L186).

## Invariants encoded in the schema

| Constraint | Invariant (say it in English) |
| --- | --- |
| [`TeamMember @@unique([teamID, userUid])`](../../../packages/hoppscotch-backend/prisma/schema.prisma#L27) | one membership per user per team |
| [`TeamInvitation @@unique([teamID, inviteeEmail])`](../../../packages/hoppscotch-backend/prisma/schema.prisma#L38) | can't invite the same email twice |
| [`TeamCollection @@unique([teamID, parentID, orderIndex])`](../../../packages/hoppscotch-backend/prisma/schema.prisma#L56) | sibling order is collision-free (NULL parent caveat — [Pattern 18](05-pattern-catalog.md#pattern-18-compound-unique-constraints-as-executable-invariants)) |
| [`VerificationToken @@unique([deviceIdentifier, token])`](../../../packages/hoppscotch-backend/prisma/schema.prisma#L141) | magic links are device-bound |
| [`Account @@unique([provider, providerAccountId])`](../../../packages/hoppscotch-backend/prisma/schema.prisma#L131) | one row per external identity |
| [`PublishedDocs @@unique([slug, version])`](../../../packages/hoppscotch-backend/prisma/schema.prisma#L317) | doc versions are addressable |
| [`MockServer.subdomain @unique`](../../../packages/hoppscotch-backend/prisma/schema.prisma#L249) | subdomain routing is unambiguous |

Cascade topology: nearly everything cascades from `User` or `Team` deletion; `MockServer.creatorUid` uses `SetNull` instead ([L262](../../../packages/hoppscotch-backend/prisma/schema.prisma#L262)) — mocks outlive their creator. Ask "delete a user: what survives?" — answer: their mock servers (ownerless), nothing else. Collection trees cascade internally ([L51](../../../packages/hoppscotch-backend/prisma/schema.prisma#L51)): folder delete = subtree delete, zero app code.

## The Json columns decision

`request Json` ([L64](../../../packages/hoppscotch-backend/prisma/schema.prisma#L64)), `variables Json` ([L91](../../../packages/hoppscotch-backend/prisma/schema.prisma#L91)), `data Json?` on collections, sessions on User (L102-L103). The DB stores these blind; the **client-side versioned schema** ([Pattern 1](05-pattern-catalog.md#pattern-1-versioned-entity-with-explicit-migrations-verzod)) owns their meaning. Consequences to be able to recite: no SQL queries into request internals (search is done via app code — see `searchByTitle`, [team-collection.service.ts#L1133](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L1133)); no DB-level validation; migrations of blob shape happen lazily at read time in clients, not in SQL. **Consistency expectation**: rows are internally consistent, but blob contents are eventually-migrated.

## Transactions and consistency

- Interactive transactions (`prisma.$transaction(async (tx) => ...)`) wrap every ordering mutation, always preceded by the row lock ([team-collection.service.ts#L476-L505](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L476-L505)).
- Retry-on-conflict wraps the transaction, not the statements ([L560-L612](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L560-L612)).
- **Where transactions are missing and might matter** (investigate, don't assert): the owner-count checks in [team.service.ts#L152-L221](../../../packages/hoppscotch-backend/src/team/team.service.ts#L152-L221) read counts outside any transaction before writing — see [risk register R1](../09-reference/risk-register.md).

## Indexes and access paths

Explicit `@@index` is rare: invitations by team ([L39](../../../packages/hoppscotch-backend/prisma/schema.prisma#L39)), mock logs by server and by server+time ([L284-L285](../../../packages/hoppscotch-backend/prisma/schema.prisma#L284-L285)), docs by collection ([L318](../../../packages/hoppscotch-backend/prisma/schema.prisma#L318)). Everything else rides on `@unique`-implied indexes. Query patterns to sanity-check against them: history per user (`UserHistory` has **no index on userUid** — sequential-ish scans as history grows; investigate real query plans before filing).

## How to safely change this schema

1. Additive first: new nullable column / new table; deploy code that tolerates both states; backfill; then tighten (NOT NULL, constraint).
2. `prisma migrate dev` generates SQL into [prisma/migrations](../../../packages/hoppscotch-backend/prisma/migrations) — read the generated SQL every time; Prisma's guess at a rename is a drop+create (data loss) unless corrected.
3. Blob-shape changes don't touch this schema at all — they're a new version module in `hoppscotch-data` + migration ([drill in Pattern 1](05-pattern-catalog.md#pattern-1-versioned-entity-with-explicit-migrations-verzod)).
4. Rollback story: migrations here have no down-scripts (Prisma default) — rollback = new forward migration. Say that phrase in interviews; it's the modern default and interviewers probe it.

## Drills

1. Write the SQL you'd expect for "move collection X under parent Y" preserving both siblings' invariants — then compare with `changeParentAndUpdateOrderIndex` ([team-collection.service.ts#L651](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L651)).
2. Design the migration to add `archivedAt DateTime?` to TeamCollection with zero-downtime deploy ordering (schema → code reads → code writes → UI).
3. From the schema alone, list every table an account-deletion GDPR request touches, and which rows survive.

Interview angle: [03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md) Q7–Q12 are built from this file.
