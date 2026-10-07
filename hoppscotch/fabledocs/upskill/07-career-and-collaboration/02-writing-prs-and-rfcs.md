# Writing PRs and RFCs

## Commits

This repo enforces conventional commits via commitlint ([commitlint.config.js](../../../commitlint.config.js), husky hooks): `feat(scope): ...`, `fix:`, `chore:`, etc. Scope by package or module (`fix(hoppscotch-backend): ...` — check `git log` upstream for the house style). One logical change per commit; the message's body answers *why*, the diff answers *what*.

## PR description template (use every time)

```markdown
## What
One paragraph. The user-visible or maintainer-visible change.

## Why
The problem/ticket, with evidence (anchor the code or issue).

## How
Design notes a reviewer needs BEFORE reading the diff: which pattern
followed (e.g., "same lock/retry machinery as createCollection"),
what was deliberately NOT done.

## How tested
Exact commands + what they prove. "pnpm --filter hoppscotch-backend test --
team.service" + new cases listed. Manual verification steps if UI.

## Risks & rollback
Blast radius in one sentence. How to revert (clean revert? flag? data?).

## Follow-ups
Deferred items, filed as issues, linked.
```

The "How tested" section is where junior PRs die. "Tests pass" is not testing evidence; *which* test would have failed without your change?

## When to RFC instead of PR

RFC first when any of: touches a contract (GraphQL schema, CLI flags, stored formats, pubsub topics); changes a security posture; introduces infrastructure (Redis, queues); spans >1 package in behavior (not just types). In this repo's terms: [P1/P5/P6](../06-contribution-practice/03-senior-build-projects.md) are RFC-first; [Ticket 3's ordering fix](../06-contribution-practice/01-good-first-tickets.md#ticket-3-reorder-the-expiry-check-in-magic-link-verification) is issue-first; a test addition is PR-only.

## The RFC template (tailored to this repo)

```markdown
# RFC: <title>
Status: draft | Discussion issue: #
## Context
What's true today, with anchors (e.g., pubsub.service.ts#L14-L18 is
in-process; deploys are single-replica because of it).
## Goals / Non-goals
Non-goals stop scope creep in review.
## Proposal
Design with the repo's vocabulary: which module, which interface stays
stable (PubSubService signature), what config gates it.
## Alternatives
At least one you'd genuinely accept. State the tradeoff axis
(e.g., Postgres LISTEN/NOTIFY: no new infra vs 8KB payload limit).
## Compatibility & migration
Contracts affected; deploy ordering; rollback point per phase.
## Risks
Ranked. Include "reviewer time" honestly for big diffs.
## Test & rollout plan
How we know it works; how we know it broke (observability!).
```

## Judgment calls that mark seniority

- Small PRs are a *courtesy with compound interest* — the strangler sequencing in [P5](../06-contribution-practice/03-senior-build-projects.md#p5-shared-treeordering-library-for-userteam-collections--3-4-weeks) exists for reviewers, not for git.
- Never mix a rename/format sweep with a behavior change (the diff hides the bug).
- If the PR needs a paragraph of context per file, it needed an RFC; write it retroactively as the PR description rather than making reviewers reverse-engineer.
