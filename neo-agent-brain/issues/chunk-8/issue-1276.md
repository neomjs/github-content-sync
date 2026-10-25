---
id: 1276
title: Family resolution mistakes inherited properties for identities
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-25T19:16:29Z'
updatedAt: '2026-10-25T19:46:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1276'
author: neo-gpt-sophie
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
closedAt: '2026-10-25T19:44:50Z'
---
# Family resolution mistakes inherited properties for identities

## Context
During the current family-admission work, an import-safe probe of `agentFamilyResolution.mjs` at Brain `3c4f22735b8e257bcc843777bd5783b51fee58a8` reproduced inherited JavaScript properties being treated as identity/family entries. Grace independently confirmed the counter defect in [her policy read](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19182069). This ticket restores the existing resolver contract; it does not implement that Discussion's ungraduated publication protocol.

## The Problem
With an empty family map, reviewer logins `constructor`, `toString` and `__proto__` report `classified:true`; author lookup likewise returns inherited functions/objects. The social-name parser also returns those inherited values instead of null. An APPROVED review by unrostered `constructor` consequently makes a known GPT author's `resolveCrossFamilyVerdict` return true in the pure probe.

With an explicit own map entry whose family is `constructor` or `toString`, review accumulation produces a string instead of a number. `__proto__` creates no own count. Even an empty count object exposes inherited values to the budget consumer's ordinary lookup. No live review or production graph was mutated to reproduce this.

## The Architectural Reality
Design authority: the module's owner contract resolves the canonical roster; `parseSelfIdLogin` promises null for an unregistered social name, and `resolveReviewerFamily` explicitly distinguishes unclassifiable reviewers for fail-closed budget handling.

The owning module is `ai/services/graph/agentFamilyResolution.mjs`: social lookup at 300, author lookups at 333–334 and 371–372, reviewer lookup at 413, and accumulation at 439–444. `PullRequestService.mjs:3386` reads the returned dictionary; `GoldenPathSynthesizer` delegates author/review classification. Caller census found no dependency on prototype methods/equality. `FleetRegistryService.mjs:1296–1336` already uses null-prototype records for the same arbitrary-key hazard. Structure map ran successfully; the existing graph module/spec own this repair.

## The Fix
Restrict social-name and family-map resolution to own entries. Keep the existing self-id/opener precedence and reviewer `@` normalization. Return a null-prototype family-count dictionary so both writes and absent-key reads are safe. Extend the existing resolver spec with the negative and positive controls below. No new source file or service boundary is needed.

| Target surface | Authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Social/author/reviewer resolution | Existing module JSDoc | Only own entries classify | Existing null/undefined/unclassified outcomes | Owning JSDoc | AC-1, AC-3 |
| Review family counts | Existing budget grouping contract | Exact own family strings receive numeric counts | Absent family remains undefined, including an empty population | Grouping JSDoc | AC-2 |
| Cross-family verdict | Existing approval and unresolved policy | Inherited entries supply no approving family | Existing unresolved-author and `unknown` behavior unchanged | Existing verdict JSDoc | AC-3 |

Decision Record impact: none. This is a current-behavior defect repair, independent of D811 graduation.

## Acceptance Criteria
- [x] AC-1 Built-in and custom inherited entries do not resolve as reviewer/author identities; unregistered prototype-named social authors return null. Explicit own login entries still resolve.
- [x] AC-2 Exact family keys including `constructor`, `toString`, `__proto__`, mixed case and whitespace count numerically across repeated reviews. Empty/missing counts, enumeration and JSON retain ordinary dictionary semantics.
- [x] AC-3 Regressions fail on the base and pass after repair: an inherited approver cannot establish cross-family coverage; real own entries, author precedence, `unknown`, unresolved and approval-state controls retain their existing policy.

Source delivery completed by [PR #1277](https://github.com/neomjs/neo-agent-brain/pull/1277), human-merged by @tobiu at 2026-10-25 19:44:48Z as `45deef149a27802212ed878bb30f8160ca33c606`. The two changed files are byte-identical to reviewed head `2623daa3c9e111e6615013cc079f84263370cfa9`. Grace's exact-head approval independently reproduced five regression failures on base and none on head; 24 CI checks passed. This closes the bounded source repair, not #700's admission or installed witness.

## Out of Scope
Family admission, eras, model observation, graph storage, roster additions, allowed-family policy, trimming/coercing values, and the installed outside-roster witness. No claim that arbitrary truthy non-string own values become schema-valid.

## Avoided Traps
A family-name denylist would reject legitimate own values and leave custom prototypes uncovered. Guarding only accumulation misses empty-map reads. Replacing the dictionary with `Map` changes the caller contract unnecessarily. A new provisioning layer cannot fix this pure lookup defect.

## Related
#700 (the consumer outcome, which stays open); #34 (existing review budget authority).

Live latest-open sweep and A2A in-flight scan: latest 20 open issues and 30 messages checked immediately before creation; no equivalent issue or claim. Exact all-state `prototype family` search found none. Own-assignment sweep: three open issues; `#700` owns the broader outcome, the other two are unrelated. MC problem sweep recovered the original fail-closed budget rationale (memory `90f201cf-5f40-4e90-8ab3-e61f41da2d38`); no prior prototype-key repair surfaced. KB could not resolve the helper, so current source and executable probes are the authority.

Origin Session ID: 5032dade-507a-4d14-ac0f-15d78670a93c
Retrieval Hint: `agentFamilyResolution inherited properties prototype review counts`



## Timeline

- 2026-10-25T19:16:29Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-10-25T19:16:30Z @neo-gpt-sophie added the `bug` label
- 2026-10-25T19:16:31Z @neo-gpt-sophie added the `ai` label
- 2026-10-25T19:16:31Z @neo-gpt-sophie added the `testing` label
- 2026-10-25T19:22:57Z @neo-gpt-sophie cross-referenced by PR #1277
- 2026-10-25T19:44:50Z @tobiu referenced in commit `45deef1` - "fix(identity): resolve only own family entries (#1276) (#1277)"
- 2026-10-25T19:44:50Z @tobiu closed this issue

