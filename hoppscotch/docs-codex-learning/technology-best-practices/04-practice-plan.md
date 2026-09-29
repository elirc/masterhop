# 30-Day Technology Practice Plan

## Days 1-5: Orientation

Day 1: Focus package map. Read `package.json`, `pnpm-workspace.yaml`. Exercise: write one sentence per package. Self-check: which package owns shared UI? Prompt: "Quiz me on package ownership."

Day 2: Focus startup. Read `selfhost-web/src/main.ts:42-204`. Exercise: draw web vs desktop platform config. Self-check: what changes between web and desktop?

Day 3: Focus app mount. Read `common/src/index.ts:24-70`. Exercise: list startup steps. Self-check: what runs before mount?

Day 4: Focus routing. Read `common/src/modules/router.ts:12-111`. Exercise: explain generated routes. Self-check: how is `org` preserved?

Day 5: Focus root UI. Read `common/src/App.vue:1-52`. Exercise: explain crash handler. Self-check: when does `RouterView` render?

## Days 6-10: Vue And Request UI

Day 6: Read `pages/index.vue:1-124`. Exercise: map tab document types. Self-check: what renders a request tab?

Day 7: Read `Request.vue:1-80`. Exercise: annotate method/URL/send controls. Self-check: where is endpoint state?

Day 8: Read `Request.vue:238-340`. Exercise: list refs/computed values. Self-check: what is local state versus tab state?

Day 9: Read `Request.vue:340-441`. Exercise: trace send behavior. Self-check: what happens on script failure?

Day 10: Small exercise: implement mentally a stricter curl detector. Prompt: "Give me hints only for testing curl detection."

## Days 11-15: State And Schemas

Day 11: Read `DispatchingStore.ts:1-77`. Exercise: explain typed dispatch. Self-check: how is payload type inferred?

Day 12: Read `history.ts:13-192`. Exercise: list dispatchers and invariants. Self-check: what caps history?

Day 13: Read `history.ts:257-372`. Exercise: trace completed response to history. Self-check: why strip `_ref_id`?

Day 14: Read `rest/index.ts:75-115` and `rest/v/17.ts:5-18`. Exercise: explain version migration. Self-check: type vs runtime validation?

Day 15: Read `collection/index.ts:27-74`. Exercise: compare collection schema to request schema. Self-check: why generate `_ref_id`?

## Days 16-20: GraphQL And Backend

Day 16: Read `GQLClient.ts:71-118`. Exercise: explain auth exchange. Self-check: when does refresh happen?

Day 17: Read `GQLClient.ts:210-333`. Exercise: compare query and subscription helpers. Self-check: where are errors reported?

Day 18: Read `history/web/api.ts:35-127`. Exercise: map wrapper names to operations. Self-check: which function creates history?

Day 19: Read `backend/src/main.ts:42-106` and `app.module.ts:42-138`. Exercise: list backend global concerns. Self-check: where is validation configured?

Day 20: Read `user-history.resolver.ts:16-170`. Exercise: trace mutations and subscriptions. Self-check: which guards apply?

## Days 21-25: Prisma, Auth, And Tests

Day 21: Read `user-history.service.ts:28-93`. Exercise: trace fetch/create. Self-check: where does JSON parse happen?

Day 22: Read `user-history.service.ts:101-233`. Exercise: identify ownership risk. Self-check: which methods receive `uid`?

Day 23: Read `schema.prisma:95-210`. Exercise: draw User, UserHistory, UserCollection, UserRequest. Self-check: which fields are JSON?

Day 24: Read `auth.controller.ts:51-100` and `auth.service.ts:103-151`. Exercise: explain refresh token flow. Self-check: where is token hashed?

Day 25: Read `user-history.service.spec.ts:143-218`. Exercise: design two missing tests. Self-check: what mocks are needed?

## Days 26-30: Review, Debug, Build Judgment

Day 26: Review a hypothetical diff adding a request field. Exercise: list touched layers. Self-check: did you include schema migration?

Day 27: Debug missing history. Exercise: trace from `Request.vue` to Prisma. Self-check: where would you log first?

Day 28: Security review. Exercise: write ownership-enforcement plan for history update/delete. Self-check: what should fail for mismatched uid?

Day 29: Performance review. Exercise: identify network-only calls and possible indexes. Self-check: what would you measure?

Day 30: Interview walkthrough. Exercise: explain Hoppscotch architecture in 3 minutes. Prompt: "Ask me follow-up interview questions about this repo and grade my answers."

