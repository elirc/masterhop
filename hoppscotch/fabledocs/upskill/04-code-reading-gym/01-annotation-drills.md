# Annotation Drills

For each excerpt: open the anchor, then annotate **inputs, outputs, dependencies, invariants, side effects, failure modes** before reading the "what to find" notes. Grade with the rubric at the bottom. Do at most two per sitting — depth beats coverage.

---

## Drill 1 — `createCollection`
Anchor: [team-collection.service.ts#L452-L517](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L452-L517)
What to find: 2 validations before the transaction; the lock; why `orderIndex` is computed *inside* the transaction; the publish *outside* it; the `Either` return.
Trap: `parentID: parentID ? parentID : undefined` (L497) — why `undefined` and not `null`? (Prisma create semantics for optional relations.)

## Drill 2 — `verifyMagicLinkTokens`
Anchor: [auth.service.ts#L257-L326](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L257-L326)
What to find: six failure exits and their HTTP statuses; the single-use token deletion; the side effect (provider-account creation) that happens *before* the expiry check; the fire-and-forget `updateUserLastLoggedOn` (L323 — no await: intentional or bug? argue it).

## Drill 3 — `createRESTNetworkRequestStream`
Anchor: [network.ts#L15-L80](../../../packages/hoppscotch-common/src/helpers/network.ts#L15-L80)
What to find: why `cloneDeep` (L23); the three terminal states; where the stream completes; the TDZ curiosity (`service` used at L36, declared L39); why cancel swallows errors (L70-L78).

## Drill 4 — `GqlTeamMemberGuard.canActivate`
Anchor: [gql-team-member.guard.ts#L21-L43](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L21-L43)
What to find: the three `BUG_*` throws vs the two user-facing errors — different audiences; the `headers.user` vs `req.user` split (subscriptions vs queries); the implicit contract that the GraphQL arg is named `teamID`.

## Drill 5 — `deleteCollectionAndUpdateSiblingsOrderIndex`
Anchor: [team-collection.service.ts#L555-L616](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L555-L616)
What to find: the retry allow-list; linear backoff; the idempotent-delete escape (L576-L580); what happens on the final failed retry; whether the sibling `updateMany` can itself violate the unique constraint mid-flight (it can — decrementing in bulk while another insert holds an index; hence the lock).

## Drill 6 — `handleMockRequest` header handling
Anchor: [mock-server.controller.ts#L107-L133](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L107-L133)
What to find: JSON.parse in a try/catch that only logs (headers silently dropped on malformed JSON — good or bad?); the array-check (L112-L116) blocking `["evil"]` payloads; the string/number-only value filter and the "type bypass" comment.

## Drill 7 — `refreshAuthTokens` + `generateRefreshToken`
Anchor: [auth.service.ts#L103-L127](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L103-L127) and [L335-L363](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L335-L363)
What to find: what's stored vs what's in the cookie; the rotation (every refresh re-hashes and overwrites); the misleading parameter name; single-active-session implication (one stored hash per user).

## Drill 8 — CLI iteration-data transform
Anchor: [cli test.ts#L53-L93](../../../packages/hoppscotch-cli/src/commands/test.ts#L53-L93)
What to find: file-exists and extension checks before read; Papa Parse with `header: true`; empty-string values dropped per-row; whole rows dropped when empty; the shape produced (`IterationDataItem[][]` — array per iteration).

## Drill 9 — `HoppRESTRequest` versioned entity
Anchor: [rest/index.ts#L75-L113](../../../packages/hoppscotch-data/src/rest/index.ts#L75-L113)
What to find: the `v` discriminator as stringified number (L75-L78 regex+transform); v0's schema-sniffing fallback; what `null` from `getVersion` means downstream (unparseable data — find the caller's behavior in verzod docs or usage).

## Drill 10 — Post-request env writeback
Anchor: [RequestRunner.ts#L623-L660](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L623-L660)
What to find: env updates applied **only if** scripts changed envs (`hasEnvironmentChanges` guard L630-L642 — why? avoids clobbering concurrent user edits); the cookie jar rebuilt as a whole Map (last-writer-wins across tabs — risk); test results translated before display.

---

## Self-grading rubric (per drill)

- **Basic**: you correctly listed inputs/outputs and at least one side effect. You can say what the function does.
- **Solid**: you named the invariant it protects, every failure exit, and traced one input to its output shape. You found the trap without the hint.
- **Strong**: you also identified something the code *commits the team to* (a contract, a convention, a scaling property), proposed a test that doesn't exist yet, and can argue one design alternative and why the author didn't choose it.

Log your grades; when three consecutive drills come out Strong, move to [review katas](04-review-katas.md).
