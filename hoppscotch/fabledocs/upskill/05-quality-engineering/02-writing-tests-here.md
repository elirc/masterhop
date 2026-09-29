# Writing Tests Here — Recipes

Seven recipes in this repo's actual idioms. Commands are **inferred** from package scripts. Backend: `cd packages/hoppscotch-backend && pnpm test -- <pattern>`. Frontend/data/cli: `pnpm --filter <package> test`.

## Recipe 1: Happy-path service test (backend)

Model: [team.service.spec.ts#L63-L80](../../../packages/hoppscotch-backend/src/team/team.service.spec.ts#L63-L80).

```ts
// Illustrative fake code: not from this repo (follows the repo's spec idiom)
const mockPrisma = mockDeep<PrismaService>()
const service = new TeamService(mockPrisma as any, mockUserService as any, mockPubSub as any)
beforeEach(() => mockReset(mockPrisma))

test("renameTeam updates and returns the team", async () => {
  mockPrisma.team.update.mockResolvedValue({ id: "t1", name: "new" } as any)
  const result = await service.renameTeam("t1", "new")
  expect(E.isRight(result)).toBe(true)
})
```

Idiom notes: assert on `E.isRight/isLeft` + payload; assert error *constants* from [errors.ts](../../../packages/hoppscotch-backend/src/errors.ts), never string literals.

## Recipe 2: Validation failure

Target: `createCollection` short-title rejection ([team-collection.service.ts#L458-L459](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L458-L459)). Arrange nothing (validation precedes DB); act with a 2-char title; assert `E.left(TEAM_COLL_SHORT_TITLE)` **and** `expect(mockPrisma.teamCollection.create).not.toHaveBeenCalled()` — the not-called assertion is the half juniors skip, and it's the one that catches ordering regressions (compare [Ticket 3](../06-contribution-practice/01-good-first-tickets.md#ticket-3-reorder-the-expiry-check-in-magic-link-verification)).

## Recipe 3: Permission failure (invariant)

Target: single-owner rule ([team.service.ts#L205-L221](../../../packages/hoppscotch-backend/src/team/team.service.ts#L205-L221)). Mock `teamMember.count` → 1 and `getTeamMember` → OWNER; assert `E.left(TEAM_ONLY_ONE_OWNER)`, `delete` not called, `publish` not called. The publish-not-called assertion documents the event contract.

## Recipe 4: Cross-tenant rejection

Target: parent-ownership check ([team-collection.service.ts#L462-L465](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L462-L465)). Mock the parent lookup to return a collection whose `teamID` ≠ input team (drive `isOwnerCheck` to `O.none` — read [L429-L442](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L429-L442) first to mock the right call). Assert `E.left(TEAM_NOT_OWNER)`. This is your IDOR regression template.

## Recipe 5: Async side effect (pubsub)

Target: any mutation. Assert topic string exactly:
```ts
// Illustrative fake code: not from this repo
expect(mockPubSub.publish).toHaveBeenCalledWith(`team/t1/member_removed`, "u2")
```
Topic strings are contracts ([Pattern 8](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-8-typed-pubsub-topics)); a typo here breaks live UIs silently — this test is the only compile-independent guard on the subscribe side today.

## Recipe 6: Data-format migration (the repo's crown-jewel test type)

Target: `hoppscotch-data` version chain. Recipe: take a **frozen literal** of an old export (a real v9 request JSON, pasted verbatim), run `HoppRESTRequest.safeParse` (verzod), assert it lands on `v: 17` with fields correctly transformed. Never construct old versions via current helpers — the point is that ancient bytes still parse. Freezing a fixture per released version is how you make "we never break saved data" enforceable. Model on the existing suites in [hoppscotch-data/src/__tests__](../../../packages/hoppscotch-data/src).

## Recipe 7: Time-dependent behavior without flakes

Target: magic-link expiry ([auth.service.ts#L299-L304](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L299-L304)). Use jest fake timers or mock the token row with `expiresOn: new Date(Date.now() - 1)` — expired by construction, zero sleeping. Assert `MAGIC_LINK_EXPIRED` and (currently) that the provider account **was** created before the check — writing the test that documents today's ordering is how you later prove the reorder PR changed it deliberately.

## Naming and structure conventions observed

`describe` per method, `test('resolves to ...')` phrasing, fixtures as top-level consts ([team.service.spec.ts#L37-L61](../../../packages/hoppscotch-backend/src/team/team.service.spec.ts#L37-L61)). Match them — reviewers notice.
