# Frontend Framework Question Cards

12 cards, Vue-centric with React-comparative framing (interviewers often let you answer in your strongest framework — knowing both models is the differentiator). Round: **frontend**.

## Q1: How does Vue know what to re-render? Contrast with React.
Repo anchor: [kernel-interceptor.service.ts#L74-L87](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L74-L87) — `reactive` state + `computed` views.
Junior: "Vue watches data."
Mid: proxies record property reads per effect; only effects that read a changed property re-run — no VDOM-diff-everything by default; React re-runs the component function and diffs.
Senior: consequences — Vue makes fine-grained derived state cheap (`current`, `available` here); the cost is edge cases (destructuring kills tracking, `markRaw` needed for non-data objects at [L130](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L130)) — and can explain *why* markRaw is there (proxying component/function-bearing objects breaks them and wastes tracking).
Follow-ups: "When does Vue's model hurt?" (huge reactive graphs; identity-sensitive libs).

## Q2: Where should client state live, and when does it leave components?
Repo anchor: the three-tier reality — component-local, dioc services ([services/](../../../packages/hoppscotch-common/src/services)), legacy RxJS stores ([newstore/](../../../packages/hoppscotch-common/src/newstore)).
Junior: "put it in a store."
Mid: state escalates only when shared across routes/components or must outlive mounting; names this repo's service pattern (class + reactive + computed, injected).
Senior: the two-generation situation and its tax ([framework models](../02-stack-and-language-mastery/02-framework-mental-models.md#rxjs-and-the-two-state-generations)); rule they'd set: new state in services, bridges explicit, no third pattern.
Follow-ups: "How would you migrate?" ([Kata 7 of review katas](../04-code-reading-gym/04-review-katas.md#kata-7-migrate-settings-store-to-dioc) is the trap catalog).

## Q3: A user closes the tab mid-request. Walk through what should happen.
Repo anchor: cancel function returned by the stream ([network.ts#L68-L79](../../../packages/hoppscotch-common/src/helpers/network.ts#L68-L79)); `cancelCalled` checks between pipeline stages ([RequestRunner.ts#L526](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L526)); subscription cleanup ([L688](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L688)).
Junior: "the request is cancelled."
Mid: enumerates the resources: in-flight transport call (cancel via interceptor contract), the RxJS subscription (unsubscribe), pending sandbox work; notes cancellation *races* the response.
Senior: the write-after-death problem — results landing on `tab.value.document` for a closed tab; asks what guards that (good honest "I'd verify" moment) and generalizes: every async completion needs an "is my target still alive?" story.

## Q4: Why run user scripts in a Web Worker instead of `eval` in the page?
Repo anchor: [web/test-runner/index.ts#L21-L41](../../../packages/hoppscotch-js-sandbox/src/web/test-runner/index.ts#L21-L41).
Junior: "workers are separate threads."
Mid: isolation (no DOM/cookies/app globals), crash containment, structured-clone boundary forces data-only exchange.
Senior: what workers *don't* give (CPU limits — no built-in timeout; same-origin network with the app's cookies if `fetch` were exposed — capability injection is the real control) and the per-run-worker vs pooled tradeoff.

## Q5: Explain optimistic UI vs server-confirmed updates using collection sync.
Repo anchor: local store mutates immediately; sync layer replays to backend and maps IDs ([gqlCollections.sync.ts#L62-L65](../../../packages/hoppscotch-selfhost-web/src/platform/collections/web/gqlCollections.sync.ts#L62-L65)); dedupe helper import at [L1-L5](../../../packages/hoppscotch-selfhost-web/src/platform/collections/web/gqlCollections.sync.ts#L1-L5) as evidence of the failure mode.
Junior: "update the UI first, then the server."
Mid: the reconciliation problem — local identity (path/index) vs server identity (ID), the mapper, and what happens on failure (rollback? retry? duplicate).
Senior: proposes client-generated stable IDs (`_ref_id` exists in the data model) as the structural fix; names the interview-classic: optimistic UI is easy, optimistic *identity* is hard.

## Q6: How do you keep a huge list (collection tree, console) fast?
Repo anchor: conceptual — no direct anchor verified; the collection tree components in [components/collections](../../../packages/hoppscotch-common/src/components/collections) are the subject.
Junior: "virtualize it."
Mid: render cost = nodes × update frequency; virtualization, `computed` granularity so one rename doesn't re-render the tree, keys stable across reorders (orderIndex changes! — key by ID, never by index, and this repo's reorder semantics make that concrete).
Senior: measurement first (Vue devtools flame), and the data-shape angle — a normalized map + child-ID lists re-renders less than nested arrays.

## Q7: What belongs in a composable/hook vs a service vs a component?
Repo anchor: [composables/](../../../packages/hoppscotch-common/src/composables) vs [services/](../../../packages/hoppscotch-common/src/services) split in this repo.
Mid: composables = reusable *per-component* stateful logic (lifecycle-bound); services = app-wide singletons (lifecycle-free); components = orchestration + template only.
Senior: the litmus — "if two components use it, do they share state (service) or each get their own (composable)?"; misplacing this is the root of "why is my state shared?!" bugs.

## Q8: How does env-variable substitution get into the request URL — and where would you add secret masking?
Repo anchor: [EffectiveURL.ts#L226-L252](../../../packages/hoppscotch-common/src/helpers/utils/EffectiveURL.ts#L226-L252); masking flags already thread through `parseTemplateString(..., showKeyIfSecret)` ([L280-L281](../../../packages/hoppscotch-common/src/helpers/utils/EffectiveURL.ts#L280-L281)).
Mid: template parse at send time producing the effective request; masking = same parse, different render target (display vs wire).
Senior: single-parser principle — preview and send must share one substitution engine or they *will* diverge ([M4](../06-contribution-practice/02-mid-level-feature-tickets.md#m4-request-level-resolved-effective-request-preview) is exactly this).

## Q9: Where do you put error UI for many failure kinds?
Repo anchor: `HoppRESTResponse` union variants each get distinct UI; interceptor errors carry their own display components ([kernel-interceptor.service.ts#L38-L47](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L38-L47)).
Mid: discriminated unions map 1:1 to UI states; no boolean `isError`.
Senior: the errors-as-props-with-components pattern here — error *sources* supply their own rendering, so adding an interceptor never touches central error UI. Open/closed principle with a real example.

## Q10: i18n — what does it change about how you write components?
Repo anchor: interceptor names/descriptions are functions of `t` ([kernel-interceptor.service.ts#L60-L64](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L60-L64)), not strings.
Mid: no hardcoded strings; messages as keys; pluralization/interpolation via the i18n layer.
Senior: why *functions of t* rather than resolved strings in data structures — resolution deferred to render time so language switches propagate; a design most codebases get wrong by caching resolved strings.

## Q11: Accessibility of a keyboard-heavy tool — what do you check?
Repo anchor: conceptual — the repo ships extensive keybindings ([helpers/keybindings.ts](../../../packages/hoppscotch-common/src/helpers/keybindings.ts)).
Mid: focus management on tab switches/modals, visible focus, shortcuts not trapping, semantic roles on the tree.
Senior: shortcut systems need an *inventory and conflict policy* (this file is one); screen-reader path for the response viewer; and "a11y regressions need tests or they return."

## Q12: How would you test frontend logic here without a browser?
Repo anchor: vitest suites in common; the runner pipeline's pure segments (env combination, translation functions in [RequestRunner.ts](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts)).
Mid: extract pure logic from components; test services/helpers headless; component tests only for interaction contracts.
Senior: the testability critique — big orchestration functions writing directly to tab state resist testing; the [K1 split](../06-contribution-practice/04-refactor-and-design-katas.md#k1-split-requestrunnerts-boundary-kata--2h-design-only) is *motivated* by testability, which is how seniors justify refactors economically.

---

Coverage: rendering model (Q1), state (Q2, Q7), effects/lifecycle/async (Q3), security (Q4), data fetching/sync (Q5), performance (Q6), templating (Q8), error UI (Q9), i18n (Q10), a11y (Q11), testing (Q12). 10 of 12 repo-anchored.
