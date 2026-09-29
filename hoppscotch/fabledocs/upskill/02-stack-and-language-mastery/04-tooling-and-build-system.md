# Tooling and Build System

Shorter than the others because [01-.../04-runtime-and-tooling-map.md](../01-codebase-cartography/04-runtime-and-tooling-map.md) covers the inventory; this file covers the *judgment*.

## The `do-*` convention vs a build tool

The root orchestrates with `pnpm -r do-dev` etc. ([package.json#L11-L21](../../../package.json#L11-L21)) — each package aliases its real scripts to `do-*` names. This is a hand-rolled, zero-dependency alternative to turborepo/nx. What you give up: task graph awareness (no "build data before common"), caching, affected-only runs. Why it still works: pnpm workspace linking plus postinstall codegen makes most packages independently runnable, and CI just runs everything. Interview-ready judgment: "I'd reach for turbo when CI time or cross-package build ordering starts hurting; before that, the convention is honest and debuggable."

## Codegen ordering — the real build graph

The *actual* dependency chain, invisible in scripts, is:

```
nest resolvers ──gen-gql──▶ gql-gen/backend-schema.gql ──graphql-codegen──▶ typed frontend ops
prisma/schema.prisma ──prisma generate──▶ src/generated/prisma ──▶ backend compiles
```

Both run at postinstall ([backend package.json postinstall](../../../packages/hoppscotch-backend/package.json); [common package.json#L17](../../../packages/hoppscotch-common/package.json#L17)), which is why a fresh `pnpm install` needs the placeholder `DATABASE_URL` trick and why deleting `node_modules` "fixes" mysterious type errors — it reruns codegen. When frontend types don't match a backend change you just made: regenerate, don't hand-edit.

## Vite specifics worth knowing here

- Worker inlining: `import Worker from "./worker?worker&inline"` ([web/test-runner/index.ts#L19](../../../packages/hoppscotch-js-sandbox/src/web/test-runner/index.ts#L19)) — Vite bundles the worker as a data URL so the sandbox works without extra deployed assets. The `?worker` suffix is a Vite convention, not standard JS.
- selfhost-web builds with `--max_old_space_size=8192` ([package.json#L10](../../../packages/hoppscotch-selfhost-web/package.json#L10)) — the app is big enough to OOM default Node heaps; that flag is a smell worth mentioning (bundle size, [performance guide](../05-quality-engineering/04-performance-thinking.md)).
- Env at build time: `VITE_*` vars are baked in; changing them means rebuilding — relevant to self-hosters and why `InfraConfig` (runtime DB config) exists for the backend.

## Quality gates and their gaps

Local: husky pre-commit → lint-staged; commit messages gated by commitlint (conventional commits — your PR titles matter here).
CI ([tests.yml](../../../.github/workflows/tests.yml)): `pnpm install` + `pnpm test`, Node 22 pinned. Observations a senior would raise (verify freshness before citing): typecheck/lint absent from the test workflow; no e2e suite in this workflow; desktop/agent built in separate workflows. "What CI doesn't gate" is as informative as what it does.

## Drills

1. Break codegen deliberately: rename a field in a backend model class, run `pnpm gen-gql` + frontend codegen (inferred commands), and read the resulting frontend error. Revert. You now know what schema drift looks like *before* it happens on a real PR.
2. Time `pnpm -r do-typecheck` on your machine; identify the slowest package; hypothesize why (hint: vue-tsc + component count).

## Interview angle

- "Monorepo tooling tradeoffs" → the `do-*` vs turbo argument above.
- "How do you keep FE/BE types in sync?" → the codegen chain; [Q10](../08-interview-prep/01-js-ts-node-deep-dive.md#q10-what-actually-happens-during-pnpm-install-in-a-monorepo-like-this) for the install story.
- "What would you add to this CI?" → typecheck gate, affected-only test runs, one smoke e2e — in that order, with cost reasoning.
