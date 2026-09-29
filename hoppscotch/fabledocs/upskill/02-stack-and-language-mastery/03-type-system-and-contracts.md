# Type System and Contracts

TypeScript in this repo does four jobs: narrow unions, derive types from runtime schemas, constrain generics, and carry generated contracts. Each below: concept → repo usage → failure mode → drill.

## 1. Discriminated unions and narrowing

Concept: a shared literal field (`type`, `_tag`, `v`) lets the compiler prove which variant you hold.
Repo: `HoppRESTResponse` (`loading | success | fail | network_fail | interceptor_error | script_fail`) narrowed via `filter` at [RequestRunner.ts#L587-L590](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L587-L590); fp-ts's own `_tag` checked raw at [network.ts#L45](../../../packages/hoppscotch-common/src/helpers/network.ts#L45) (`res._tag === "Right"` — bypassing `E.isRight`; it works, but couples to fp-ts internals — reviewable).
Failure mode: adding a union member and missing a switch site. Defense: exhaustiveness checks (`never` assertion) — the repo largely relies on discipline instead.
Drill: list every `type` value of `HoppRESTResponse` from its definition file (grep `HoppRESTResponse`) and find one consumer per variant.

## 2. Runtime schemas as the source of static types

Concept: TS types vanish at runtime; zod schemas exist at runtime; `z.infer`/`InferredEntity` derives the static type so there's one source of truth.
Repo: [rest/index.ts#L80-L115](../../../packages/hoppscotch-data/src/rest/index.ts#L80-L115) — `HoppRESTRequest` the *value* is a verzod entity (runtime), `HoppRESTRequest` the *type* is inferred from it (L115 — note the deliberate name doubling). Validation is dispatch-by-version, then migration ([Pattern 1](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-1-versioned-entity-with-explicit-migrations-verzod)).
Failure mode: hand-written interfaces drifting from the schema; or validating in some load paths but not others (imports! — check [helpers/import-export](../../../packages/hoppscotch-common/src/helpers/import-export)).
Drill: find where `fixBrokenRequestVersion.ts` ([helpers](../../../packages/hoppscotch-common/src/helpers/fixBrokenRequestVersion.ts)) is used — a real scar from versioning going wrong; write one sentence on what it repairs.

## 3. fp-ts: `Either`/`Option`/`TaskEither` as contract vocabulary

Concept: `Option<T>` = presence contract; `Either<L, R>` = failure contract; `TaskEither` = async failure contract. The signature *is* the documentation.
Repo: backend services return `Either<ErrorCode, T>` ([team.service.ts#L205-L208](../../../packages/hoppscotch-backend/src/team/team.service.ts#L205-L208) even spells the return type `Promise<E.Left<string> | E.Right<boolean>>` — more literal than idiomatic `Promise<Either<...>>`, another review conversation); lookups return `Option` ([auth.service.ts#L81-L95](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L81-L95)); the frontend network strategy type is a `TaskEither` ([network.ts#L11-L13](../../../packages/hoppscotch-common/src/helpers/network.ts#L11-L13)).
Failure mode: Either-in-name-only — code that constructs Eithers but every caller immediately throws on Left gains ceremony without safety. Judge each layer: here the controller boundary translation ([auth.controller.ts#L69](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L69)) is exactly right.
Drill: rewrite `leaveTeam`'s signature three ways (throwing, Either, result-object) and write one caller for each; feel the difference.

## 4. Generated types as enforced contracts

Concept: when a contract has two sides (GraphQL schema ↔ client operations; DB schema ↔ query client), generate one side from the other so drift becomes a compile error.
Repo: Prisma client generated into [src/generated/prisma](../../../packages/hoppscotch-backend/prisma/schema.prisma#L1-L4); GraphQL SDL emitted from resolvers, consumed by `graphql-codegen` in every frontend package ([runtime map](../01-codebase-cartography/04-runtime-and-tooling-map.md#codegen--the-hidden-coupling)). Distinct model classes for the API layer ([team.model.ts](../../../packages/hoppscotch-backend/src/team/team.model.ts)) keep DB types from leaking — with explicit mapping ([team.service.ts#L194-L198](../../../packages/hoppscotch-backend/src/team/team.service.ts#L194-L198)).
Failure mode: committing stale generated artifacts; or editing generated files (never — they're overwritten).
Drill: trace one GraphQL mutation type from resolver decorator → emitted SDL → frontend generated hook (grep the mutation name in `selfhost-web/src/api`).

## 5. Where `any` leaks and what it costs

Repo reality check: `transformCollectionForBackend(collection: HoppCollection): any` ([gqlCollections.sync.ts#L39](../../../packages/hoppscotch-selfhost-web/src/platform/collections/web/gqlCollections.sync.ts#L39)) returns `any` — every consumer of the sync payload is unchecked from there on. Contrast with the CLI's `unknown[]` + narrowing at [cli test.ts#L48-L73](../../../packages/hoppscotch-cli/src/commands/test.ts#L48-L73), the disciplined version.
Policy worth adopting (and stating in interviews): `any` may appear in test doubles; never in a return type; `unknown` at trust boundaries with explicit narrowing.

## Interview angle

- Narrowing/unions → [Q3](../08-interview-prep/01-js-ts-node-deep-dive.md#q3-what-is-a-discriminated-union-and-why-would-you-type-api-results-with-one)
- zod vs TS types → [Q5](../08-interview-prep/01-js-ts-node-deep-dive.md#q5-how-do-zod-or-similar-runtime-schemas-relate-to-typescript-types)
- Generics with `TopicDef` → [Q6](../08-interview-prep/01-js-ts-node-deep-dive.md#q6-explain-generics-with-a-real-example-that-isnt-arrayt)
- `unknown` vs `any` → [Q7](../08-interview-prep/01-js-ts-node-deep-dive.md#q7-unknown-vs-any--where-does-it-matter-in-real-code)
- Structural typing hazards → [Q13](../08-interview-prep/01-js-ts-node-deep-dive.md#q13-how-does-structural-typing-bite-you-when-two-same-shape-models-diverge)
