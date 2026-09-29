# Review Katas

Eight fake PRs against this codebase. For each: read the intent and diff summary, write your review (findings graded **Blocking / Important / Optional**), then compare. All diffs are **illustrative fake code, not from this repo**, designed to resemble real files. Practice kind, specific language — a good comment names the risk, anchors the evidence, and offers a path.

## Kata 1: "Add collection description editing"

Author intent: let users edit a description on team collections.
Fake diff summary: adds `description` arg to `updateTeamCollection` resolver; service writes `data.description` into the Json column; no guard changes; publishes `coll_updated`.
Files this resembles: [team-collection.service.ts#L1087](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L1087), [team-collection.resolver.ts](../../../packages/hoppscotch-backend/src/team-collection/team-collection.resolver.ts).
Expected findings — Blocking: resolver added without `@RequiresTeamRole`/guard parity with sibling mutations (check what `updateTeamCollection` requires — likely EDITOR+). Important: Json `data` merge semantics — does the write clobber sibling keys (auth/headers/variables) stored in `data`? Needs read-modify-write or a typed merge + a test. Optional: description length limit consistent with `TITLE_LENGTH` convention.
Good comment example:
> Blocking: this mutation writes `data` wholesale — a concurrent auth-config save would be lost. Can we merge into the existing `data` (and add a test like the clobber case in X)? Also flagging that the sibling mutations all carry `@RequiresTeamRole(OWNER, EDITOR)` — this one needs the same.

## Kata 2: "Speed up owner check"

Author intent: performance — replace two sequential awaits in `leaveTeam` with `Promise.all`.
Fake diff summary: `const [ownerCount, member] = await Promise.all([count(), getTeamMember()])`.
Files this resembles: [team.service.ts#L205-L221](../../../packages/hoppscotch-backend/src/team/team.service.ts#L205-L221).
Expected findings — Blocking: none strictly (the reads were already non-transactional). Important: this PR *narrows nothing but the window* — the real issue is the TOCTOU on the invariant; suggest converting to a transaction with locking instead, otherwise the "optimization" cements a race. Optional: measured impact? (Two indexed reads — this is optimizing noise.)
Lesson: reviews should surface when a PR polishes code that needs *restructuring* — kindly. "Is this path actually hot?" is a legitimate review question.

## Kata 3: "Add request duplication endpoint"

Author intent: `POST /v1/team-collection/:id/duplicate-request`.
Fake diff summary: new REST controller method; reads request, creates copy with `orderIndex: siblings.length + 1` outside any transaction; returns Prisma row directly.
Expected findings — Blocking: ordering write without lock/transaction (violates the module's own invariant discipline — cite [createCollection](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L476-L505)); raw DB row in the response (leaks shape; `cast()` convention exists). Important: `siblings.length + 1` ≠ `max + 1` after deletions. Important: guard? REST side needs [rest-team-member.guard.ts](../../../packages/hoppscotch-backend/src/team/guards/rest-team-member.guard.ts)-style protection.
Lesson: "does it follow the module's existing invariant machinery" is the first question for any write-path PR.

## Kata 4: "Nicer error for expired magic links"

Author intent: friendlier UX message.
Fake diff summary: in `verifyMagicLinkTokens`, replaces `MAGIC_LINK_EXPIRED` string with a prose sentence inline.
Files this resembles: [auth.service.ts#L299-L304](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L299-L304), [errors.ts](../../../packages/hoppscotch-backend/src/errors.ts).
Expected findings — Blocking: error *codes* are a contract — the frontend and tests match on constants from `errors.ts`; inline prose breaks matching. Important: user-facing copy belongs in the client's i18n layer, not the API error. Optional: if copy must change, change it client-side keyed on the stable code.
Lesson: distinguish machine-readable error contracts from human copy. Cheap review win, very common real PR.

## Kata 5: "Cache team membership in the guard"

Author intent: cut a DB query per request by caching `getTeamMember` results in-process for 60s.
Fake diff summary: `Map<teamID+uid, member>` with TTL inside `GqlTeamMemberGuard`.
Files this resembles: [gql-team-member.guard.ts#L37-L42](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L37-L42).
Expected findings — Blocking: authorization decisions served stale — a removed member keeps access up to 60s; role demotions delayed. This is a security-latency tradeoff that needs product sign-off, not a perf PR. Important: unbounded Map growth; multi-instance incoherence. Optional: if approved, invalidate on the `member_removed/updated` publish points.
Lesson: caching authz is *possible* but never a drive-by. Name the staleness window explicitly in review.

## Kata 6: "Support HTML preview for mock responses"

Author intent: users complain path-based mock HTML renders as plain text (see [Scenario 5](../05-quality-engineering/03-systematic-debugging.md#scenario-5-mock-server-returns-the-right-body-but-the-browser-downloads-it-instead-of-rendering)); PR removes the MIME downgrade.
Fake diff summary: deletes the `ACTIVE_CONTENT_TYPES` downgrade block.
Files this resembles: [mock-server.controller.ts#L142-L154](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L142-L154).
Expected findings — Blocking: reintroduces same-origin stored XSS — the downgrade exists *because* path-based mocks share the API origin; the CSP header alone shouldn't carry the whole defense. Point to the subdomain URL as the supported HTML path, or propose serving previews from a sandboxed origin. Important: security-relevant deletion with no security reviewer tagged, no test removed/updated (there should be a test asserting the downgrade — is there?).
Lesson: when a PR deletes a defense to close a UX ticket, the review must reconstruct why the defense exists. Comments in code (L142-L144 here) are what make that survivable.

## Kata 7: "Migrate settings store to dioc"

Author intent: move `newstore/settings.ts` to a dioc service.
Fake diff summary: new `SettingsService`; old store kept but marked deprecated; 14 call sites updated; sync layer (`selfhost-web`) untouched "because it still works via the old store."
Expected findings — Blocking: two live sources of truth — sync writes the old store, UI reads the new service; settings changed on another device won't render. The bridge (old store → service, or a single adapter) must ship in the same PR. Important: migration ADR/plan reference; test covering the bridge. Optional: deprecation timeline.
Lesson: partial state migrations are the most dangerous "safe" PRs. The seam ([framework models](../02-stack-and-language-mastery/02-framework-mental-models.md#rxjs-and-the-two-state-generations)) must be explicit.

## Kata 8: "Add retry to email sending"

Author intent: transient SMTP failures shouldn't fail signin.
Fake diff summary: wraps `mailerService.sendEmail` in the same retry helper as collections (5 attempts, backoff), still awaited inline in `signInMagicLink`.
Files this resembles: [auth.service.ts#L238-L244](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L238-L244).
Expected findings — Important: 5 retries × backoff happen *inside the HTTP request* — signin latency balloons to seconds; retry budget must fit the request budget, or email must go async (which changes the "did we email you?" contract). Important: is sending idempotent? Duplicate emails on retry after timeout-but-sent. Optional: single retry with short backoff is a fair inline compromise; log the failure regardless.
Lesson: retries move latency; the same pattern that's right for a 10ms DB conflict is wrong for a 2s SMTP call. Context, not recipes.

---

## Grading your reviews

Basic: you caught each kata's headline Blocking issue. Solid: your comments cite a real repo convention as evidence (guards parity, cast(), error constants, lock discipline). Strong: your comments are ones you'd be glad to receive — specific, anchored, offering a path — and you correctly *withheld* Blocking status where the issue was Important-but-shippable (katas 2, 8 hinge on that judgment).

Convert katas 5 and 6 into timed interview rounds via [08-.../05-debugging-and-code-review-rounds.md](../08-interview-prep/05-debugging-and-code-review-rounds.md).
