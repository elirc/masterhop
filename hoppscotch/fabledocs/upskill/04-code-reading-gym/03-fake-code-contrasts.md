# Fake-Code Contrasts

Ten pairs. Every snippet below is **illustrative fake code, not from this repo** (labeled per snippet as well); each pair ends with the real repo code that embodies the better shape. Read bad → diagnose → read better → open the anchor.

## 1. Ordering without atomicity

```ts
// Illustrative fake code: not from this repo
const last = await db.collection.findFirst({ where: { parentID }, orderBy: { orderIndex: "desc" } })
await db.collection.create({ data: { ...input, orderIndex: (last?.orderIndex ?? 0) + 1 } })
```
Diagnosis: two concurrent calls read the same `last` → duplicate index (or constraint error with no recovery).
Better: read-and-write inside one transaction with the sibling set locked, constraint as backstop, bounded retry.
Real: [team-collection.service.ts#L476-L505](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L476-L505).

## 2. Missing tenant check on a child resource (IDOR)

```ts
// Illustrative fake code: not from this repo
@Mutation() async renameCollection(@Args("collectionID") id: string, @Args("title") t: string) {
  return this.svc.rename(id, t)  // whoever you are
}
```
Diagnosis: authenticated ≠ authorized; any user can rename any team's collection by guessing IDs.
Better: guard resolves resource→team→membership before the handler runs.
Real: guard stack pattern at [team.resolver.ts#L215-L219](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L215-L219); collection-ID guards in [team-collection/guards](../../../packages/hoppscotch-backend/src/team-collection/guards).

## 3. Publish inside the transaction

```ts
// Illustrative fake code: not from this repo
await db.$transaction(async (tx) => {
  const coll = await tx.collection.create({ data })
  await pubsub.publish(`coll_added`, coll)   // tx may still roll back!
})
```
Diagnosis: subscribers can receive events for state that never commits.
Better: publish after the transaction resolves.
Real: [team-collection.service.ts#L505-L514](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L505-L514).

## 4. Retrying everything

```ts
// Illustrative fake code: not from this repo
for (let i = 0; i < 5; i++) { try { return await doWrite() } catch { await sleep(100) } }
```
Diagnosis: retries validation bugs and auth failures alike; hides real errors for 500ms × N; no backoff growth.
Better: allow-list transient error codes; growing backoff; log each retry.
Real: [team-collection.service.ts#L596-L612](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L596-L612).

## 5. Coupling UI shape to DB shape

```ts
// Illustrative fake code: not from this repo
return await prisma.teamMember.findMany() // straight to the resolver return
```
Diagnosis: DB column names become API contract; adding `refreshToken`-style columns later leaks them.
Better: explicit mapping at the service edge.
Real: [team.service.ts#L194-L198](../../../packages/hoppscotch-backend/src/team/team.service.ts#L194-L198); `cast()` in [team-collection.service.ts#L279](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L279).

## 6. Swallowed errors

```ts
// Illustrative fake code: not from this repo
try { await sendMail(user.email, link) } catch {} // best effort ¯\_(ツ)_/¯
```
Diagnosis: user waits for an email that will never come; nothing logged; support ticket unexplainable.
Better: let expected failures become typed values the caller must route; log at the swallow point when tolerating *is* correct.
Real contrast: the deliberate, *commented* tolerance of P2025 in concurrent delete ([team-collection.service.ts#L576-L580](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L576-L580)) — swallowing with a reason and a scope, vs the empty catch above. Also the cancel-error swallow with explanation at [network.ts#L70-L78](../../../packages/hoppscotch-common/src/helpers/network.ts#L70-L78).

## 7. Boolean-blind results

```ts
// Illustrative fake code: not from this repo
async function verifyMagic(dto): Promise<boolean> { ... } // true/false, good luck knowing why
```
Diagnosis: caller can't distinguish expired vs invalid vs user-missing → one generic error toast, no correct statuses.
Better: `Either<TypedError, Tokens>` with one boundary translation.
Real: [auth.service.ts#L257-L326](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L257-L326) — six distinct Lefts with statuses.

## 8. Secrets stored raw

```ts
// Illustrative fake code: not from this repo
await db.user.update({ where: { uid }, data: { refreshToken } }) // the actual token
```
Diagnosis: DB dump = every user's live session.
Better: store a slow hash; verify on use; rotate on refresh.
Real: [auth.service.ts#L114-L119](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L114-L119), [L344-L352](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L344-L352).

## 9. Any-typed boundary data

```ts
// Illustrative fake code: not from this repo
const rows: any = Papa.parse(csv).data
rows.forEach(r => envs.push({ key: r.key, value: r.val })) // r.val is undefined, silently
```
Diagnosis: `any` erases the misspelling; empty env vars flow into requests.
Better: `unknown` + explicit narrowing + filtering.
Real: [cli test.ts#L48-L93](../../../packages/hoppscotch-cli/src/commands/test.ts#L48-L93).

## 10. User content served trustingly

```ts
// Illustrative fake code: not from this repo
res.set(userMock.headers).send(userMock.body) // including Set-Cookie and text/html, same origin
```
Diagnosis: stored XSS + session fixation on your own domain, delivered as a feature.
Better: header blocklist, value-type filtering, MIME downgrade on same-origin, CSP sandbox, nosniff.
Real: [mock-server.controller.ts#L107-L182](../../../packages/hoppscotch-backend/src/mock-server/mock-server.controller.ts#L107-L182).

---

Self-grade: for each pair, before reading the diagnosis — Basic: you found *a* problem. Solid: you found *the* problem (the one that pages someone). Strong: your proposed fix matched the repo's actual shape, or you can argue yours is better.
