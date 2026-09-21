---
id: 292
title: Retire the twelve Engine-owned src/ai unit specs the split left in Brain
state: CLOSED
labels:
  - testing
  - tech-debt
assignees:
  - neo-opus-grace
createdAt: '2026-09-01T03:12:29Z'
updatedAt: '2026-09-01T10:21:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/292'
author: neo-opus-grace
commentsCount: 0
parentIssue: 191
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-01T10:21:17Z'
---
# Retire the twelve Engine-owned src/ai unit specs the split left in Brain

## Problem

Twelve unit specs for **Engine-owned** `src/ai/**` subjects live in this repository at paths identical to the ones the Engine deleted in `c623b2f63c`:

```
test/playwright/unit/ai/LockRegistry.spec.mjs        test/playwright/unit/ai/client/ComponentService.spec.mjs
test/playwright/unit/ai/TransactionService.spec.mjs  test/playwright/unit/ai/client/DockService.spec.mjs
test/playwright/unit/ai/WriteGuard.spec.mjs          test/playwright/unit/ai/client/InstanceService.spec.mjs
test/playwright/unit/ai/admitWrite.spec.mjs          test/playwright/unit/ai/client/RuntimeService.spec.mjs
test/playwright/unit/ai/deriveSubtreePath.spec.mjs   test/playwright/unit/ai/client/TourRunner.spec.mjs
test/playwright/unit/ai/parseAgentEnvelope.spec.mjs  test/playwright/unit/ai/resolveWriteLock.spec.mjs
```

All twelve verified present on `origin/dev@a212acc`. They import the Engine as a package (`neo.mjs/src/ai/…`), so they assert the Engine's own unit contracts from a consumer repository.

Two things follow, and only the first is a Brain defect:

1. **These are Engine-subject suites sitting in Brain.** ADR 0040 keeps `src/ai/**` Engine-owned and puts test custody with the *subject*. Consuming the published Engine does not make its unit contracts Brain-owned.
2. **They are never executed here anyway.** `brain-unit.yml` collects the corpus with `--list`, then runs a four-file smoke that contains none of them. That half is **already owned by #201** — this ticket does not restate or re-fix it, and deliberately does not wait on it.

Point 2 is what makes point 1 cheap to resolve: because Brain CI never executes these twelve, deleting them costs this repository **zero executed coverage**. There is no window where Brain loses a signal it was actually getting.

## Intended solution shape

Delete exactly these twelve paths from Brain, **sequenced behind** the Engine restoration in neomjs/neo PR #18002 so the assertions exist in an executing suite before they leave here. Engine CI runs `npm run test-${{ matrix.suite }}` with no file list, so the restored specs execute there on landing — the coverage genuinely moves rather than evaporating.

If a specific Brain **consumer** contract needs protection — that Brain's own call sites drive these Engine services correctly across the package boundary — that is a narrow integration arm observing Brain's usage, authored deliberately. It is not a second copy of the Engine's subject suite, and it is not a prerequisite for this deletion.

The drift is already visible and is the argument against leaving them: the Brain DockService/TourRunner copies still name pre-v13.2 dock modules and schemas, while the Engine copies restored in PR #18002 are ported to `Operations`, `Persistence`, `PerspectiveLibrary` and `neo.dock.*`. Two authorities for one contract have already diverged, and the un-executed one is the stale one.

## Acceptance criteria

- [ ] The twelve paths above are absent from Brain, and each subject's assertions exist in the Engine suite at the merge head — i.e. the move is complete in both directions, not a delete that drops coverage.
- [ ] The Engine PR (#18002) has landed first; this PR states the Engine commit it sequences behind.
- [ ] `npm run test-unit -- --list` collects cleanly after the deletion — no dangling import or helper left behind by removing the twelve.
- [ ] No new Brain-side copy of an Engine subject suite is introduced as a replacement. Any retained arm is justified in the PR as a Brain **consumer** boundary and names the Brain call site it protects.
- [ ] The deletion does not touch #201's smoke list or the Brain-tier gate; if either needs to change, that is #201's lane and is linked rather than absorbed.

## Out of scope

- **Binding the full retained Brain suite in CI** — #201 owns it. This ticket neither depends on nor anticipates it.
- **The other specs retired by the split.** This is a named twelve-path population, not a census of Brain's retained corpus.
- **Authoring Brain consumer-integration coverage for `src/ai/**`.** Legitimate follow-up if a real call site warrants it; it is not required to complete this move and must not be used to justify keeping the copies.
- **Any change to the Engine specs themselves** — their content is PR #18002's business.

## Avoided traps

- **Deleting before the Engine half lands.** That is the one sequencing that creates a real coverage gap, however brief, and it is the reason this is explicitly Engine-first.
- **Keeping the copies "until #201 binds the suite."** That inverts the dependency: once #201 lands, these twelve would begin executing *in the wrong repository*, cementing the duplicate authority instead of retiring it. Engine-subject suites should leave Brain before Brain's suite becomes real, not after.
- **Reading "they pass locally" as "they are covering something."** They do pass when run by hand; no pipeline runs them. Local green is not a signal this repository is receiving.
- **Replacing the twelve with an inventory of the twelve.** Per #191's own avoided-traps: the deletion is the artifact.

## Related

- Parent: #191 (delete legacy Brain surfaces within domain slices).
- Sequenced behind: neomjs/neo PR #18002, neomjs/neo#17968.
- Adjacent, deliberately not depended on: #201 (run the retained Brain unit suite in CI).
- Authority: ADR 0040 §2.7 — test custody follows the subject.
- Required by: RA-1 of @neo-gpt's review on neomjs/neo PR #18002.

Origin Session ID: c5d6f187-307b-49b6-af0d-9c4503c71a19

Retrieval Hint: "twelve Engine src/ai unit specs still in Brain after the split" · Engine-first paired custody move


## Timeline

- 2026-09-01T03:12:31Z @neo-opus-grace added the `testing` label
- 2026-09-01T03:12:32Z @neo-opus-grace added the `tech-debt` label
- 2026-09-01T03:14:49Z @neo-opus-grace cross-referenced by PR #18002
- 2026-09-01T03:40:59Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-01T03:43:04Z @neo-opus-grace cross-referenced by PR #293
- 2026-09-01T10:21:17Z @tobiu referenced in commit `ac56359` - "Merge pull request #293 from neomjs/grace/292-retire-engine-ai-specs

test(brain): retire the twelve Engine-owned src/ai specs (#292)"
- 2026-09-01T10:21:17Z @tobiu closed this issue

