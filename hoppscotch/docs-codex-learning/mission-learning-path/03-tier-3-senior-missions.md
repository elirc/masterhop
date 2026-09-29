# Tier 3 Senior Missions

### Mission 18: Reverse-Engineer The Architecture Decisions

**Tier:** Senior
**Time Estimate:** 75 minutes
**Goal:** Infer why the repo uses platform injection, versioned schemas, and sync adapters.
**The Concept:** Architecture is the residue of constraints.
**Design Intent Before You Read the Code:** The app supports web, desktop, self-host, evolving request formats, and authenticated sync. A simpler single-app design would be easier today but worse over time.
**Find It In The Code:** `selfhost-web/src/main.ts:42-204`, `common/src/index.ts:24-70`, `data/src/rest/index.ts:80-115`, `history/web/sync.ts:26-60`.

```ts
// selfhost-web/src/main.ts:174-179
sync: { environments: config.environments, collections: config.collections, settings: config.settings, history: config.history }
// Sync is a platform capability, so common state can be reused with different persistence backends.
```

**The Aha Moment:** The most important architecture choices support product variants and long-lived user data.
**Socratic Checkpoint:** What constraints does platform injection solve? What constraints do versioned schemas solve? Why are sync adapters outside common? What trade-off does JSON persistence make? What would be simpler but less durable?
**How to Self-Grade:** Strong answers argue trade-offs, not just benefits.
**Connects To:** Mission 19.

### Mission 19: Find The Bugs Before They Happen

**Tier:** Senior
**Time Estimate:** 60 minutes
**Goal:** Identify likely defects from boundaries and invariants.
**The Concept:** Senior intuition is pattern matching against broken promises.
**Design Intent Before You Read the Code:** Authenticated data mutations should enforce ownership; JSON parsing should be safe; subscription echoes should not double-apply.
**Find It In The Code:** `user-history.service.ts:101-162`, `user-history.service.ts:69-77`, `history/web/index.ts:193-231`, `sync.ts:40-45`.

```ts
// user-history.service.ts:108-115
await this.prisma.userHistory.update({ where: { id }, data: { isStarred: ... } })
// The method receives uid but does not enforce it in this query.
```

**The Aha Moment:** Bugs often hide where the function signature promises more safety than the implementation enforces.
**Socratic Checkpoint:** Which methods receive `uid`? Which queries use it? What happens if JSON is malformed? Why does duplicate removal exist? What test would prove the bug?
**How to Self-Grade:** Strong answers produce at least one concrete test per risk.
**Connects To:** Mission 20 and Mission 22.

### Mission 20: The Bug Injection Challenge

**Tier:** Senior
**Time Estimate:** 60 minutes
**Goal:** Practice debugging from symptoms without modifying production code.
**The Concept:** Inject the bug in your head, then predict where evidence appears.
**Design Intent Before You Read the Code:** You should trace symptoms to boundaries: UI state, store dispatch, sync, API, service, database.
**Find It In The Code:** `Request.vue:340-441`, `history.ts:354-372`, `sync.ts:26-60`, `history/web/index.ts:153-231`, `user-history.service.ts:60-162`.

```ts
// history/web/index.ts:207-214
if (existingEntry && existingEntry.star !== isStarred) {
  toggleRESTHistoryEntryStar(existingEntry)
}
// Remove this guard mentally: starring would flip twice on subscription echo.
```

**The Aha Moment:** Good debugging starts with a predicted failure point.
**Socratic Checkpoint:** Where would you log first for missing history? For duplicate history? For wrong star state? For auth failure? For database write failure?
**How to Self-Grade:** Strong answers name an observation, expected value, and next branch in the trace.
**Connects To:** Mission 23.

### Mission 21: Performance X-Ray

**Tier:** Senior
**Time Estimate:** 55 minutes
**Goal:** Audit likely performance pressure points.
**The Concept:** Performance work starts by finding loops, network calls, and unbounded growth.
**Design Intent Before You Read the Code:** Local history is capped, backend fetches are limited, but network-only GraphQL and missing indexes can hurt at scale.
**Find It In The Code:** `history.ts:129-146`, `user-history.service.ts:28-38`, `GQLClient.ts:217-220`, `composables/graphql.ts:117-120`, `schema.prisma:152-161`.

```ts
// history.ts:145
state: [entry, ...currentVal.state].slice(0, HISTORY_LIMIT)
// Local state is intentionally bounded.
```

**The Aha Moment:** Bounded client memory does not automatically mean bounded backend cost.
**Socratic Checkpoint:** What is bounded? What is network-only? Which query may need an index? What would you measure before optimizing? What caching risk matters in an API client?
**How to Self-Grade:** Strong answers pair each concern with measurement, not premature refactor.
**Connects To:** Mission 24.

### Mission 22: The Security Audit

**Tier:** Senior
**Time Estimate:** 70 minutes
**Goal:** Audit auth, ownership, CORS, tokens, and validation boundaries.
**The Concept:** Security is making invalid paths boring and unsuccessful.
**Design Intent Before You Read the Code:** Guards should authenticate, services should enforce ownership, tokens should rotate, and CORS should be environment-aware.
**Find It In The Code:** `main.ts:58-70`, `auth.controller.ts:87-100`, `auth.service.ts:103-151`, `gql-auth.guard.ts:5-11`, `user-history.service.ts:101-162`.

```ts
// auth.service.ts:114-126
const refreshTokenHash = await argon2.hash(refreshToken)
await this.usersService.updateUserRefreshToken(refreshTokenHash, userUid)
// Refresh tokens are hashed before storage.
```

**The Aha Moment:** Auth at the resolver is necessary but not the same as ownership enforcement in the service query.
**Socratic Checkpoint:** Where is CORS strict? Where are refresh tokens generated and hashed? What does `GqlAuthGuard` extract? Which history methods should use `uid` in queries? What validation exists before controllers?
**How to Self-Grade:** Strong answers identify one positive pattern and one concrete hardening task.
**Connects To:** Mission 23.

### Mission 23: Write The Test That Doesn't Exist

**Tier:** Senior
**Time Estimate:** 75 minutes
**Goal:** Design tests for risky untested behavior.
**The Concept:** A good test guards a promise the codebase quietly depends on.
**Design Intent Before You Read the Code:** Unit tests should isolate service logic; integration tests should verify boundaries. Poor tests mock the behavior they claim to validate.
**Find It In The Code:** `user-history.service.spec.ts:143-218`, `workspace.service.spec.ts:72-140`, `persistence/__tests__/index.spec.ts:202-240`.

```ts
// user-history.service.spec.ts:208-216
await userHistoryService.createUserHistory(..., "INVALID")
// Existing tests prove invalid reqType handling; add similar tests for bad JSON and ownership.
```

**The Aha Moment:** The best missing test often mirrors an existing test shape but targets a neglected branch.
**Socratic Checkpoint:** What is already tested? What is not? Which mocks are needed? What should not be mocked? What assertion proves the risk is closed?
**How to Self-Grade:** Strong answers produce a test name, setup, action, assertions, and why it would fail today.
**Connects To:** Mission 24.

### Mission 24: The Git History Tells A Story

**Tier:** Senior
**Time Estimate:** 45 minutes
**Goal:** Use commit history to infer system evolution.
**The Concept:** Git history is archaeology for architecture.
**Design Intent Before You Read the Code:** Recent commits reveal which parts are active, risky, or repeatedly fixed.
**Find It In The Code:** Run `git log --oneline --decorate -n 30` locally, then inspect touched files with `git show --stat <sha>`.

```text
# Suggested read pattern
git log --oneline --decorate -n 30
git show --stat <sha>
git show --name-only <sha>
```

**The Aha Moment:** Repeated changes are signals: unstable boundary, active product area, or recurring bug class.
**Socratic Checkpoint:** Which package changes most? Are migrations frequent? Are auth files active? Are tests added with fixes? What commit themes would affect your feature plan?
**How to Self-Grade:** Strong answers identify five commit themes and connect each to a current architecture risk or learning area.
**Connects To:** Mission 25 in the reference artifacts.

