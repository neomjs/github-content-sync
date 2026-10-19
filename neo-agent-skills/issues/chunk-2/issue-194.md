---
id: 194
title: The automerge contract pins fetch-metadata@v2 and fails its own bump
state: OPEN
labels:
  - bug
  - ai
  - testing
  - build
assignees:
  - neo-opus-ada
createdAt: '2026-10-19T12:52:53Z'
updatedAt: '2026-10-19T12:56:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/194'
author: neo-opus-ada
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# The automerge contract pins fetch-metadata@v2 and fails its own bump

## Context

Dependabot PR #173 (`dependabot/fetch-metadata` v2 → v3, plus this repo's reusable-workflow tag) has had a red `corpus` job since 2026-10-13. Dependabot rebased it today and it failed again ([run 39226257431](https://github.com/neomjs/neo-agent-skills/actions/runs/39226257431)) at `scripts/test-reusable-dependabot-automerge.mjs`:

```
AssertionError: canonical workflow violates its own contract
  actual: [ 'a failed metadata read fails the job' ], expected: []
```

Vega triaged the cause on #173 the same day. Nobody filed it, so the PR has waited six days. This ticket files that triage.

## The Problem

`validateReusableDependabotAutomerge` checks the property *a failed metadata read does not fail the job* by matching the literal `'        continue-on-error: true\n        uses: dependabot/fetch-metadata@v2'` (`scripts/test-reusable-dependabot-automerge.mjs:284`). The version token has nothing to do with that property. So every major bump of the action, which is exactly the update Dependabot exists to make, reports the property as lost while `continue-on-error: true` is untouched. The check describes what the file says today, not what it has to guarantee (the class #191 names).

## The Architectural Reality

- `.github/workflows/reusable-dependabot-automerge.yml` — the eligibility job's `metadata` step: `if:` limits it to Dependabot's own events, then `continue-on-error: true`, then `uses: dependabot/fetch-metadata@v2`. The comment above the step says why a failed read must still reach the decision.
- The workflow reads only `steps.metadata.outputs.dependency-names` and `steps.metadata.outputs.update-type`. fetch-metadata v3's `action.yml` still exports both. Its one breaking change is the `node24` Actions runtime (release notes on #173), so v3 is safe to take.
- Design authority: the check names its own property, "a failed metadata read fails the job" (`:284`), and the step comment gives the reason. Neither names a major version. No other contract line in the validator pins a version.
- The mutation control (`:364`) deletes `continue-on-error: true\n` and expects this failure. It must stay red under the fix.

## The Fix

In `validateReusableDependabotAutomerge`, match the property for any ref: in the eligibility job, `continue-on-error: true` directly followed by `uses: dependabot/fetch-metadata@<ref>`, e.g. `/^ {8}continue-on-error: true\n {8}uses: dependabot\/fetch-metadata@\S+$/m`. Add a control arm that moves the canonical source to a ref it will never carry (`@v99`) and still passes. `@v3` would stop testing anything once #173 merges. Bump `package.json` / `package-lock.json` (`scripts/check-version-bump.mjs`: a maintainer PR carries a new release).

## Acceptance Criteria

- [ ] A canonical source whose metadata step uses another ref (`dependabot/fetch-metadata@v99`) passes the contract (new arm: red on `dev`, green at head).
- [ ] The `:364` mutation (no `continue-on-error`) still reports `a failed metadata read fails the job`.
- [ ] `node scripts/test-reusable-dependabot-automerge.mjs` and the full `corpus` job pass.
- [ ] Post-merge validation: #173 rebased → `corpus` green → its automerge handoff runs.

## Out of Scope

- Taking fetch-metadata v3 itself: that is #173's diff. This ticket only stops the check from refusing it.
- Changing the step's `if:` or `continue-on-error` semantics.

## Related

- #173 (the blocked bump) and Vega's triage comment on it · #191 (tests derive from what the module should do) · #159 / #160 (where the automerge contract started)

Live latest-open sweep: latest 20 open issues read 2026-10-19T12:53Z, no equivalent found. A2A sweep: last 30 min of claims, none overlapping. MC sweep: `test-reusable-dependabot-automerge fetch-metadata@v2 continue-on-error canonical workflow violates its own contract`, 6 results, no prior decision found. Own-assignment sweep: 1 open (#102), not overlapping.

Origin Session ID: 17ba709b-6759-4aa3-b484-e78a178b7f19
Retrieval Hint: "automerge contract fetch-metadata major pin corpus red"


## Timeline

- 2026-10-19T12:52:53Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-19T12:52:55Z @neo-opus-ada added the `bug` label
- 2026-10-19T12:52:55Z @neo-opus-ada added the `ai` label
- 2026-10-19T12:52:56Z @neo-opus-ada added the `testing` label
- 2026-10-19T12:52:56Z @neo-opus-ada added the `build` label
- 2026-10-19T12:57:07Z @neo-opus-ada cross-referenced by PR #195
- 2026-10-19T12:58:24Z @neo-opus-ada cross-referenced by #19863

