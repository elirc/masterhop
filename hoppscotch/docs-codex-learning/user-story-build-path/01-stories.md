# User Stories

## Story 1: Show Request Description In The Save Menu

**Difficulty:** Easy

**Estimated Time:** 1.5 hours

**Skills You'll Practice:** Vue templates, v-model, request schema reading

**The Story:** As an API developer, I want to see and edit a request description near the request name so that saved requests are easier to recognize later.

**Why This Story Matters:** REST request schema v17 already includes `description`; surfacing it teaches how UI follows domain schema.

**Acceptance Criteria:**
- [ ] Save/options menu shows a description field for REST requests.
- [ ] Empty descriptions are stored as `null` or the existing schema-accepted empty value.
- [ ] Existing send/save behavior still works.
- [ ] No backend or database changes are required.

**Files You'll Likely Touch:**
- `packages/hoppscotch-common/src/components/http/Request.vue` - save/options menu and request object binding.

**Relevant Existing Patterns:**
- `Request.vue:178-187` edits `tab.document.request.name`.
- `packages/hoppscotch-data/src/rest/v/17.ts:5-18` defines `description`.
- `packages/hoppscotch-data/src/rest/index.ts:242-260` builds default REST requests.

**High-Level Implementation Plan:**
1. In `Request.vue`, study the save dropdown around `178-211`.
2. Add a compact input or textarea bound to `tab.document.request.description`.
3. Normalize empty string to `null` in a small handler or computed setter.
4. Confirm `HoppRESTRequest` accepts the value.

**Testing and Verification Plan:** Manually create a request, type description, open/close save menu, save as collection if setup allows. Add a component-level test only if this repo has nearby component testing patterns; otherwise document manual verification.

**Tips:**
- Keep the field inside the existing dropdown, not a new modal.
- Follow existing `input` class usage in `Request.vue`.
- Do not change `@hoppscotch/data`; the schema already exists.

**What Could Go Wrong:**
- Empty string may violate the intended `nullable` shape; inspect `V17_SCHEMA`.
- A large textarea may break compact toolbar layout.
- Saving may serialize `undefined`; use `null` if unsure.

**Stretch Goal:** Show description in collection request previews.

**Connects To:** Story 2 because both improve request editing without backend changes.

## Story 2: Warn Before Sending A Request With Only Whitespace In Method

**Difficulty:** Easy

**Estimated Time:** 1 hour

**Skills You'll Practice:** validation, Vue event handling, toasts

**The Story:** As an API developer, I want the app to reject a blank custom method so that I do not send invalid requests accidentally.

**Why This Story Matters:** `Request.vue` already validates blank endpoints; this extends the same guard to method state.

**Acceptance Criteria:**
- [ ] Sending with blank/whitespace method shows an error toast.
- [ ] Valid custom methods still send.
- [ ] Existing endpoint validation remains unchanged.

**Files You'll Likely Touch:**
- `packages/hoppscotch-common/src/components/http/Request.vue` - `newSendRequest`, `updateMethod`, method input.

**Relevant Existing Patterns:**
- Endpoint guard in `Request.vue:340-344`.
- Method update in `Request.vue:512-519`.
- Toast usage in `Request.vue:290-292`, `340-343`.

**High-Level Implementation Plan:**
1. In `newSendRequest`, add a method trim check after endpoint validation.
2. Use the existing `toast.error` pattern.
3. Ensure `updateMethod` does not mask invalid input before validation.

**Testing and Verification Plan:** Manually select custom method, clear method, press Send, verify toast and no request run.

**Tips:**
- Keep validation close to send, not only input, because state can be changed programmatically.
- Use `newMethod.value`.
- Do not alter method color helper.

**What Could Go Wrong:**
- Blocking all custom methods if validation only accepts known verbs.
- Showing two toasts if endpoint is blank too.

**Stretch Goal:** Add a tiny helper function for request-toolbar validation.

**Connects To:** Story 3 because it also improves request safety.

## Story 3: Add A Safer Curl Paste Detection

**Difficulty:** Easy

**Estimated Time:** 2 hours

**Skills You'll Practice:** string validation, UI behavior, focused testing

**The Story:** As an API developer, I want curl import to trigger only for real curl commands so that pasted URLs containing the word curl do not open the import modal.

**Why This Story Matters:** The current detector is intentionally simple and may be too broad.

**Acceptance Criteria:**
- [ ] Pasting `curl https://example.com` opens import modal.
- [ ] Pasting `https://example.com/search?q=curl` does not.
- [ ] Previous endpoint text is preserved when curl modal opens.

**Files You'll Likely Touch:**
- `packages/hoppscotch-common/src/components/http/Request.vue` - `onPasteUrl` and `isCURL`.

**Relevant Existing Patterns:**
- `Request.vue:456-468` handles paste and curl detection.

**High-Level Implementation Plan:**
1. Replace `curl.includes("curl ")` with a stricter regex for line-start or whitespace-delimited `curl`.
2. Keep `onPasteUrl` behavior unchanged.
3. Add a small unit test only if there is an easy helper extraction pattern; otherwise manually verify.

**Testing and Verification Plan:** Paste several curl and non-curl strings into the URL input.

**Tips:**
- Avoid a complex parser; this is just modal detection.
- Preserve old behavior for uppercase/lowercase only if product expects it.
- Consider leading whitespace.

**What Could Go Wrong:**
- False negatives for multi-line curl commands.
- False positives for prose text beginning with "curling".

**Stretch Goal:** Extract `isCURL` to a helper and unit test it.

**Connects To:** Story 4 because parsing user input grows into reusable UI behavior.

## Story 4: Add Recent History Endpoint Count In The Request Bar

**Difficulty:** Medium

**Estimated Time:** 3 hours

**Skills You'll Practice:** computed state, store streams, compact UI

**The Story:** As an API developer, I want to see how many recent history entries match my current endpoint so that I know whether I have called it before.

**Why This Story Matters:** This reuses local history without new backend work.

**Acceptance Criteria:**
- [ ] Request bar shows a small count when current endpoint appears in history.
- [ ] Count updates after a request completes.
- [ ] Count does not disrupt mobile layout.

**Files You'll Likely Touch:**
- `packages/hoppscotch-common/src/components/http/Request.vue` - history stream and toolbar UI.

**Relevant Existing Patterns:**
- History stream consumption in `Request.vue:328-332`.
- History entry shape in `history.ts:13-28`.
- Completed response history insertion in `history.ts:354-372`.

**High-Level Implementation Plan:**
1. Add computed `matchingHistoryCount` from `history.value`.
2. Compare normalized endpoint strings carefully.
3. Render a compact label near URL input or send actions.
4. Verify layout at narrow width.

**Testing and Verification Plan:** Send the same endpoint twice and confirm count increases. Clear history if UI provides it and confirm count resets.

**Tips:**
- Use the existing `history` readonly stream.
- Avoid scanning backend history directly.
- Do not mutate history entries in the component.

**What Could Go Wrong:**
- Comparing before `ensureMethodInEndpoint` causes mismatch.
- Count includes current unsent edits incorrectly.
- UI crowding in toolbar.

**Stretch Goal:** Show the last status code for the matching endpoint.

**Connects To:** Story 5 because both build history-aware UI.

## Story 5: Add A History Store Status Indicator

**Difficulty:** Medium

**Estimated Time:** 4 hours

**Skills You'll Practice:** platform feature state, Vue computed, settings/status UX

**The Story:** As a signed-in user, I want to know whether history sync is enabled so that I understand whether my request history is local or cloud-backed.

**Why This Story Matters:** Sync status already exists but is easy to overlook.

**Acceptance Criteria:**
- [ ] UI shows enabled/loading/error state for history sync.
- [ ] Signed-out users see a local-history state.
- [ ] Indicator reads from existing platform history status.

**Files You'll Likely Touch:**
- `packages/hoppscotch-common/src/components/history/index.vue` or nearby history component - display status.
- `packages/hoppscotch-common/src/platform/history.ts` - study interface if needed.
- `packages/hoppscotch-selfhost-web/src/platform/history/web/index.ts` - existing status refs.

**Relevant Existing Patterns:**
- `history/web/index.ts:318-329` exports `requestHistoryStore` status refs.
- Platform injection in `selfhost-web/src/main.ts:174-179`.

**High-Level Implementation Plan:**
1. Find the history panel component and existing status display conventions.
2. Read platform history definition.
3. Bind to `platform.sync.history.requestHistoryStore` or the actual exposed interface.
4. Render a compact, non-blocking status.

**Testing and Verification Plan:** Test signed out, signed in with backend available, and backend error if possible.

**Tips:**
- Keep this in common only if the platform interface exposes needed state.
- Do not import self-host web files into common.
- Follow existing i18n patterns for labels.

**What Could Go Wrong:**
- Violating package boundary by importing `@app/platform/history/web`.
- Showing stale status before auth finishes.

**Stretch Goal:** Link disabled state to settings if an existing settings route exists.

**Connects To:** Story 6 because status UI prepares you for sync/API changes.

## Story 6: Add Backend Validation For Malformed History JSON

**Difficulty:** Medium

**Estimated Time:** 5 hours

**Skills You'll Practice:** NestJS service validation, Either errors, Jest tests

**The Story:** As a backend maintainer, I want malformed history JSON to return a controlled error so that bad clients cannot crash request history creation.

**Why This Story Matters:** `JSON.parse` is a runtime boundary and currently appears unguarded.

**Acceptance Criteria:**
- [ ] Malformed `reqData` returns a typed error.
- [ ] Malformed `resMetadata` returns a typed error.
- [ ] Prisma create is not called for malformed JSON.
- [ ] Existing valid history tests still pass.

**Files You'll Likely Touch:**
- `packages/hoppscotch-backend/src/user-history/user-history.service.ts` - parse boundary.
- `packages/hoppscotch-backend/src/errors.ts` or relevant errors file - new error constant if needed.
- `packages/hoppscotch-backend/src/user-history/user-history.service.spec.ts` - service tests.

**Relevant Existing Patterns:**
- `user-history.service.ts:60-93` creates history.
- Invalid req type test in `user-history.service.spec.ts:208-216`.
- Resolver error handling in `user-history.resolver.ts:49-56`.

**High-Level Implementation Plan:**
1. Add safe parse helper inside service or nearby utility.
2. Return `E.left` with a named error on parse failure.
3. Use parsed values in Prisma create.
4. Add Jest tests for invalid request and metadata JSON.

**Testing and Verification Plan:** Run backend user-history service tests.

**Tips:**
- Preserve `Either` style.
- Do not throw from service if surrounding pattern returns `E.left`.
- Assert Prisma is not called.

**What Could Go Wrong:**
- Throwing before resolver can convert into generic 500.
- Returning a string not recognized by error helpers.

**Stretch Goal:** Add structured GraphQL input types later instead of JSON strings.

**Connects To:** Story 8 because both harden backend history.

## Story 7: Add A Team Workspace Polling Visibility Toggle

**Difficulty:** Medium

**Estimated Time:** 5 hours

**Skills You'll Practice:** services, Vue refs, tests, workspace state

**The Story:** As a team user, I want a visible way to pause team list polling so that I can reduce background network activity during debugging.

**Why This Story Matters:** `WorkspaceService` has polling locks; exposing a controlled toggle teaches service ownership.

**Acceptance Criteria:**
- [ ] Toggle pauses team list polling for the active UI scope.
- [ ] Toggle resumes polling.
- [ ] Existing workspace behavior remains unchanged by default.
- [ ] Unit tests cover pause/resume behavior.

**Files You'll Likely Touch:**
- `packages/hoppscotch-common/src/services/workspace.service.ts` - polling lock behavior.
- `packages/hoppscotch-common/src/components/workspace/Selector.vue` - likely UI surface.
- `packages/hoppscotch-common/src/services/__tests__/workspace.service.spec.ts` - tests.

**Relevant Existing Patterns:**
- Polling setup in `workspace.service.ts:61-108`.
- `acquireTeamListAdapter` in `workspace.service.ts:217-227`.
- Polling tests in `workspace.service.spec.ts:142-230`.

**High-Level Implementation Plan:**
1. Decide whether toggle belongs in service or component state.
2. Reuse lock semantics rather than inventing a global timer.
3. Add UI control in workspace selector.
4. Extend tests for paused state.

**Testing and Verification Plan:** Run workspace service tests and manually confirm no polling calls fire while paused.

**Tips:**
- Do not break `tryOnScopeDispose`.
- Keep default behavior unchanged.
- Prefer service API over direct timer mutation.

**What Could Go Wrong:**
- Global pause affects other scopes unexpectedly.
- Timer resumes with stale interval.

**Stretch Goal:** Persist the debugging preference locally.

**Connects To:** Story 9 because it prepares for cross-layer state changes.

## Story 8: Enforce User Ownership On History Update/Delete

**Difficulty:** Hard

**Estimated Time:** 6 hours

**Skills You'll Practice:** security review, Prisma queries, backend tests

**The Story:** As a security-conscious maintainer, I want history update/delete operations to enforce ownership so that users cannot modify another user's history by ID.

**Why This Story Matters:** Guards authenticate users, but service/database predicates should enforce ownership too.

**Acceptance Criteria:**
- [ ] Toggle star only succeeds for the authenticated user's own history.
- [ ] Delete only succeeds for the authenticated user's own history.
- [ ] Tests cover mismatched `uid`.
- [ ] Existing subscription behavior remains correct.

**Files You'll Likely Touch:**
- `packages/hoppscotch-backend/src/user-history/user-history.service.ts` - ownership predicates.
- `packages/hoppscotch-backend/src/user-history/user-history.service.spec.ts` - tests.
- Possibly `packages/hoppscotch-backend/prisma/schema.prisma` only if a compound unique key is chosen.

**Relevant Existing Patterns:**
- Current update/delete paths in `user-history.service.ts:101-162`.
- Authenticated resolver calls in `user-history.resolver.ts:63-94`.
- Prisma model in `schema.prisma:152-161`.

**High-Level Implementation Plan:**
1. Write failing tests for mismatched user.
2. Update lookup to find by both `id` and `userUid`.
3. Ensure update/delete uses the verified record.
4. Keep pubsub payloads unchanged.

**Testing and Verification Plan:** Run backend user-history tests.

**Tips:**
- Start with service tests before changing code.
- Avoid changing GraphQL API shape.
- Preserve existing error type for not found if appropriate.

**What Could Go Wrong:**
- Updating by `id` after a secure find still has a race window; decide if acceptable.
- Changing Prisma schema requires migration and more review.

**Stretch Goal:** Add a compound unique constraint for `(id, userUid)`.

**Connects To:** Story 10 because both involve production readiness.

## Story 9: Persist Request Description Through User Collections

**Difficulty:** Hard

**Estimated Time:** 8 hours

**Skills You'll Practice:** schema tracing, collection save flow, GraphQL sync, tests

**The Story:** As an API developer, I want request descriptions to persist when saving to personal collections so that documentation survives across sessions.

**Why This Story Matters:** It forces a full persistence trace beyond local editing.

**Acceptance Criteria:**
- [ ] Description survives saving a REST request to a user collection.
- [ ] Description loads after refresh/sync.
- [ ] Old requests without description still load.
- [ ] Tests or manual verification cover save/load.

**Files You'll Likely Touch:**
- `packages/hoppscotch-common/src/components/collections/SaveRequest.vue` - save UI flow.
- `packages/hoppscotch-common/src/newstore/collections.ts` - collection state updates.
- `packages/hoppscotch-selfhost-web/src/platform/collections/web/index.ts` - sync/load mapping.
- Backend user collection/request files if persistence mapping drops the field.

**Relevant Existing Patterns:**
- `HoppRESTRequest` schema in `rest/index.ts:80-115`.
- Collection schema in `collection/index.ts:27-74`.
- Collection parsing in `selfhost-web/src/platform/collections/web/index.ts:158-164`.

**High-Level Implementation Plan:**
1. Trace save request flow from modal to collection store.
2. Confirm the request object is stored whole.
3. Trace self-host collection sync mapping for REST requests.
4. Fix any mapping that strips `description`.
5. Add a regression test near the smallest stripping point.

**Testing and Verification Plan:** Save request with description, reload app, confirm description remains.

**Tips:**
- Do not add duplicate description fields outside `request`.
- The likely bug, if present, is mapping/serialization, not schema.
- Study GraphQL collection APIs before touching backend.

**What Could Go Wrong:**
- Description persists locally but not after backend sync.
- Old imported requests fail if `description` is assumed string.

**Stretch Goal:** Show descriptions in generated documentation.

**Connects To:** Story 10 because it practices full-stack data preservation.

## Story 10: Add Audit Logging For History Mutations

**Difficulty:** Expert

**Estimated Time:** 12-16 hours

**Skills You'll Practice:** architecture planning, Prisma modeling, backend modules, security, tests

**The Story:** As an organization administrator, I want audit logs for history create/update/delete events so that sensitive request activity can be reviewed.

**Why This Story Matters:** This introduces a new production-readiness domain concept and forces careful boundaries.

**Acceptance Criteria:**
- [ ] Backend records create/star/delete/all-delete history actions.
- [ ] Audit rows include actor user ID, action, target history ID when available, timestamp, and request type.
- [ ] User-facing behavior does not change.
- [ ] Tests cover service audit creation for key actions.
- [ ] No sensitive request bodies are logged unless explicitly designed and reviewed.

**Files You'll Likely Touch:**
- `packages/hoppscotch-backend/prisma/schema.prisma` - new audit model.
- `packages/hoppscotch-backend/src/user-history/user-history.service.ts` - emit audit records.
- `packages/hoppscotch-backend/src/user-history/user-history.service.spec.ts` - tests.
- Possibly new backend module/service if audit logging should be reusable.

**Relevant Existing Patterns:**
- Prisma models in `schema.prisma:95-161`.
- User history service mutations in `user-history.service.ts:60-205`.
- Pubsub after mutation in `user-history.service.ts:86-90`, `123-128`, `154-159`.
- Backend module composition in `app.module.ts:105-130`.

**High-Level Implementation Plan:**
1. Write an architecture note before coding: audit model fields, retention, privacy, query needs.
2. Add Prisma model and migration.
3. Decide whether audit logging lives in `UserHistoryService` or a reusable `AuditLogService`.
4. Add audit writes inside the same logical mutation path.
5. Avoid logging full request JSON initially.
6. Add tests for create, star toggle, delete.
7. Run Prisma generation and backend tests.

**Testing and Verification Plan:** Unit test service calls with mocked Prisma audit model; integration test later if database test setup exists.

**Tips:**
- Treat request bodies as sensitive.
- Keep audit failure behavior explicit: should failed audit block user action?
- Do not expose admin UI until backend semantics are stable.

**What Could Go Wrong:**
- Logging sensitive data creates compliance risk.
- Audit writes outside transactions can diverge from actual mutations.
- Adding a Prisma model without generated client updates breaks backend build.

**Stretch Goal:** Add admin GraphQL query for audit logs with pagination and role guard.

**Connects To:** This is the capstone: it uses schema, service, security, testing, and architecture judgment.

