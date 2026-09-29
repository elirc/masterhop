# Trace Tables

Fill each table yourself first (columns: step, file/line, value shape, owner, transformation, risk), then compare. Do them on paper — the friction is the exercise.

## Trace 1 (UI→network): `<<baseUrl>>/users` from editor to wire

Scenario: selected env has `baseUrl = https://api.example.com`; user hits Send.

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | tab document | `HoppRESTRequest{endpoint:"<<baseUrl>>/users"}` | tab service | none | stale tab state |
| 2 | [RequestRunner.ts#L495-L499](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L495-L499) | + inherited auth/headers | runner | merge collection-level props | wrong inheritance order |
| 3 | [RequestRunner.ts#L520-L524](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L520-L524) | script result Either | sandbox | script may rewrite endpoint/envs | script_fail aborts |
| 4 | [RequestRunner.ts#L566-L581](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L566-L581) | combined env list → `EffectiveHoppRESTRequest{effectiveFinalURL:"https://api.example.com/users"}` | env engine | `<<var>>` substitution ([EffectiveURL.ts#L226-L252](../../../packages/hoppscotch-common/src/helpers/utils/EffectiveURL.ts#L226-L252)) | unresolved vars pass through literally |
| 5 | [network.ts#L25](../../../packages/hoppscotch-common/src/helpers/network.ts#L25) | `RelayRequest` | kernel adapter | Hopp shape → kernel shape | lossy conversion for exotic bodies |
| 6 | interceptor | bytes on wire | selected transport | serialization | CORS (browser transport only) |

## Trace 2 (persistence): `deleteCollection(collectionID)` with 3 siblings

Start state: siblings at orderIndex 1,2,3; deleting index 2.

| Step | File/line | What happens | DB state after | Risk |
| --- | --- | --- | --- | --- |
| 1 | [team-collection.service.ts#L624-L626](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L624-L626) | fetch collection or Left | 1,2,3 | not found |
| 2 | [L563-L570](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L563-L570) | tx opens; sibling rows locked | 1,2,3 (locked) | lock wait |
| 3 | [L572-L580](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L572-L580) | delete row (cascade takes subtree) | 1,3 | already deleted → treated as success |
| 4 | [L582-L589](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L582-L589) | `updateMany` orderIndex>2 → decrement | 1,2 | bulk update inside lock — safe |
| 5 | [L595](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L595) / retry loop | commit or retry on deadlock | 1,2 committed | retry exhaustion → Left |
| 6 | [L636-L639](../../../packages/hoppscotch-backend/src/team-collection/team-collection.service.ts#L636-L639) | publish `coll_removed` | — | crash before publish = silent UI staleness |

## Trace 3 (auth): refresh-token rotation

| Step | File/line | Cookie holds | DB holds | Risk |
| --- | --- | --- | --- | --- |
| 1 | login completes, [auth.service.ts#L135-L151](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L135-L151) | RT₁ (JWT) | argon2(RT₁) on user row ([L114-L119](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L114-L119)) | — |
| 2 | `GET /auth/refresh`, [auth.controller.ts#L87-L100](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L87-L100) | RT₁ sent | argon2(RT₁) | RT guard validates JWT signature+expiry first |
| 3 | [auth.service.ts#L344-L352](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L344-L352) | — | verify(hash, RT₁) | mismatch → 404-coded error (why not 401? worth debating) |
| 4 | [L355-L362](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L355-L362) | RT₂ set | argon2(RT₂) overwrites | RT₁ now dead everywhere — replay of RT₁ fails at step 3 |

Question the table forces: what happens to a *second device* holding RT₁? (Dead session — single stored hash means single active refresh chain. Investigate whether that's intended product behavior.)

## Trace 4 (authz denial): VIEWER calls `renameTeam`

| Step | File/line | Outcome |
| --- | --- | --- |
| 1 | [team.resolver.ts#L243-L244](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L243-L244) | metadata: requires OWNER |
| 2 | `GqlAuthGuard` | JWT ok → user attached |
| 3 | [gql-team-member.guard.ts#L37-L38](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L37-L38) | membership found (VIEWER) |
| 4 | [L40-L42](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L40-L42) | role not in {OWNER} → throw `TEAM_NOT_REQUIRED_ROLE` |
| 5 | GraphQL layer | error response; resolver body **never ran** |

Contrast row: same user, `teamID` of a team they're *not* in → step 3 throws `TEAM_MEMBER_NOT_FOUND` instead. Two different errors — does that leak team existence to non-members? (Both fire only with a valid teamID guess; assess and argue.)

## Trace 5 (error path): pre-request script throws

| Step | File/line | Value | UI consequence |
| --- | --- | --- | --- |
| 1 | sandbox run | `E.left(errMsg)` | — |
| 2 | [RequestRunner.ts#L528-L531](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L528-L531) | logged, `E.left("script_fail")` | request never sent — no network step |
| 3 | caller of runner | Left propagates | toast/error state; response pane untouched |

Compare with *post*-request script failure ([L661-L686](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L661-L686)): the response **is** shown, testResults get `scriptError: true`. Asymmetry is correct — pre-fail prevents the send; post-fail must not hide a response that already happened. Being able to defend that asymmetry is a Strong grade.

## Self-grade (all traces)

Basic: your table matches on steps. Solid: your value-shape column was right before checking. Strong: you filled the risk column with at least one entry per trace that this file doesn't list.
