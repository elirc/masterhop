# Framework Mental Models

Three frameworks carry this repo: Vue 3 (frontend), RxJS (legacy state + streams), NestJS (backend). The transferable skill is recognizing each framework's *one core idea* and where the repo bends it.

## Vue 3: reactivity is dependency tracking, not re-rendering

Mental model: `reactive()`/`ref()` wrap values in proxies; `computed()` and `watchEffect()` record which properties they *read* and re-run when exactly those change. Unlike React, there is no "component re-runs top to bottom" — the template's render function is just another tracked effect.

Where the repo uses it well: [KernelInterceptorService](../../../packages/hoppscotch-common/src/services/kernel-interceptor.service.ts#L74-L127) — a plain class holding `reactive` state, exposing `computed` views (`current`, `available`), and a `watchEffect` that auto-heals selection when the current interceptor becomes unselectable (L97-L108). Note `markRaw` on registered interceptors (L130): they contain components/functions that must **not** be proxied — knowing *when to opt out* of reactivity is the mid-level skill.

Sharp edges to check in any Vue code here:
- Destructuring reactive state kills tracking (`const { currentId } = this.state` gives a dead value).
- Mutating deeply into big objects (tab documents — `tab.value.document.request.requestVariables = ...` at [RequestRunner.ts#L564](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L564)) works but hides the write-path — grep for who else writes the same path before adding another writer.
- `watchEffect` over-triggering when it reads more than it needs.

## RxJS and the two state generations

The repo contains **two frontend state systems** — say this in interviews; the analysis is senior-signal:

1. **Legacy: `newstore/` dispatching stores** ([newstore/](../../../packages/hoppscotch-common/src/newstore)) — RxJS `BehaviorSubject` + dispatcher functions (Redux-ish, pre-dating modern Vue). Sync code subscribes to these ([gqlCollections.sync.ts#L1-L9](../../../packages/hoppscotch-selfhost-web/src/platform/collections/web/gqlCollections.sync.ts#L1-L9) imports `graphqlCollectionStore`).
2. **Current: `dioc` services** ([services/](../../../packages/hoppscotch-common/src/services)) — class-based DI containers with Vue reactivity inside.

Why it matters: every feature touching both layers pays a translation tax, and bugs breed at the seam (store updated, service not, or vice versa). When adding state, follow the *new* pattern unless the data you extend already lives in a store. The honest architecture note: no visible migration plan — a normal condition in real codebases, and a great "how do you handle legacy?" interview story.

Streams proper: request lifecycle as `BehaviorSubject` ([network.ts#L18-L21](../../../packages/hoppscotch-common/src/helpers/network.ts#L18-L21)), consumed with `filter` + narrow + `unsubscribe` ([RequestRunner.ts#L587-L590](../../../packages/hoppscotch-common/src/helpers/RequestRunner.ts#L587-L590), L688). Rule: every `subscribe` needs an ownership answer — who unsubscribes, when?

## NestJS: everything is a provider; requests flow through a decorated pipeline

Mental model: modules wire providers into an injector; a request passes **guards → interceptors → pipes → handler → interceptors**; decorators attach metadata that guards/interceptors read via `Reflector`.

The repo's cleanest exhibits:
- DI: [AuthService's constructor](../../../packages/hoppscotch-backend/src/auth/auth.service.ts#L35-L42) — six injected collaborators, all mockable ([team.service.spec.ts#L15-L30](../../../packages/hoppscotch-backend/src/team/team.service.spec.ts#L15-L30) builds services by hand with mocks — no TestingModule ceremony).
- Guard pipeline with metadata: `@RequiresTeamRole` + `GqlTeamMemberGuard` ([Pattern 3](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-3-declarative-authorization--guard--metadata-decorator)). Guard order matters: `@UseGuards(GqlAuthGuard, GqlTeamMemberGuard)` — authn must populate `req.user` before authz reads it ([team.resolver.ts#L218](../../../packages/hoppscotch-backend/src/team/team.resolver.ts#L218)).
- Cross-cutting via interceptor: [UserLastLoginInterceptor](../../../packages/hoppscotch-backend/src/auth/auth.controller.ts#L116) stamps login time on SSO callbacks without polluting handlers.
- Code-first GraphQL: resolvers + decorators *are* the schema; SDL is emitted, not hand-written ([runtime map](../01-codebase-cartography/04-runtime-and-tooling-map.md#codegen--the-hidden-coupling)).

Sharp edges: guards throwing plain `Error` in GraphQL context produce generic errors (see the `BUG_*` convention softening this); decorator metadata is invisible in call stacks — a forgotten `@RequiresTeamRole` fails at runtime, not compile time (the guard's hard throw at [gql-team-member.guard.ts#L26](../../../packages/hoppscotch-backend/src/team/guards/gql-team-member.guard.ts#L26) is the mitigation).

## Drills

1. In `KernelInterceptorService`, predict what happens if you `register()` two interceptors and then `setActive(null)` — trace L137-L153 before answering.
2. Find one component in [components/http](../../../packages/hoppscotch-common/src/components/http) that reads a `newstore` store and one that injects a dioc service; note how each subscribes.
3. Write (on paper) the guard execution order for `removeTeamMember` and what each guard adds to the request context.

## Interview angle

- Vue reactivity vs React re-render → [02-frontend-framework-questions.md](../08-interview-prep/02-frontend-framework-questions.md) Q1–Q4
- "Where do you put cross-cutting concerns in Node?" → guards/interceptors, [03-api-and-data-modeling-questions.md](../08-interview-prep/03-api-and-data-modeling-questions.md) Q6
- "How do you handle legacy patterns coexisting with new ones?" → the two-state-generations story, [06-behavioral-star-stories.md](../08-interview-prep/06-behavioral-star-stories.md) Story 7
