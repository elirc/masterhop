# How To Use Codex With These Stories

## Start With Story 1

Read the story, open the listed files, and write your own plan. Before coding, ask Codex:

```text
I am working on Story 1. Here is my plan: [...]. Compare it to existing Hoppscotch patterns. Do not write code yet.
```

## Ask For Hints Instead Of Full Solutions

```text
I am stuck finding where request descriptions are saved. Give me three hints in increasing specificity. Do not show code unless I ask.
```

## Ask Codex To Explain Existing Patterns First

```text
Explain how `packages/hoppscotch-common/src/components/http/Request.vue` uses `v-model` and tab state. Cite line ranges. Do not propose changes yet.
```

## Ask Codex To Review Your Implementation

```text
Review my diff for Story X. Prioritize bugs, missed tests, package-boundary violations, and behavior regressions.
```

## Ask Codex To Create Tests After You Implement

```text
I implemented Story X. Study nearby tests first, then suggest the smallest useful test. Explain why this test belongs there.
```

## Ask Codex To Explain A Diff

```text
Explain this diff as if I need to defend it in code review. What changed, why does it belong in these files, and what could break?
```

## Prevent Unrelated Changes

Tell Codex:

```text
Documentation and unrelated formatting are off limits. Touch only the files needed for Story X. If another file seems necessary, explain why before editing.
```

## Checklist Before Accepting AI Code

- Does it follow an existing pattern from a cited file?
- Does it touch only expected files?
- Does it preserve package boundaries?
- Does it include or justify tests?
- Does it handle loading/error/empty states?
- Does it avoid logging or storing secrets?
- Can you explain every line?

## Checklist Before Opening A PR

- Read your diff top to bottom.
- Run focused tests or document why not.
- Run typecheck for the touched package if practical.
- Confirm manual acceptance criteria.
- Remove debug logs.
- Write a PR summary with files changed and risks.

## Example Prompts

Understanding a file:

```text
Walk me through `packages/hoppscotch-common/src/newstore/history.ts`. Focus on state ownership and side effects. Ask me questions after each section.
```

Tracing a bug:

```text
History entries duplicate after sending a request. Help me trace the flow from `Request.vue` to subscriptions. Ask for observations before giving fixes.
```

Planning a change:

```text
I want to add backend validation for malformed history JSON. Help me plan files and tests. Do not implement.
```

Reviewing a diff:

```text
Review this diff like a senior engineer. Findings first. Focus on security, test gaps, and boundary mistakes.
```

Writing tests:

```text
Study `user-history.service.spec.ts` and propose tests for ownership enforcement. Use existing mocking style.
```

Asking for hints only:

```text
I am stuck on the sync layer. Give me one hint at a time and wait for my answer.
```

