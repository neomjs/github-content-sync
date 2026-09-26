---
id: 549
title: 'Two wake-envelope plants on one seat: which is authoritative, and should the report be louder than a warning?'
state: OPEN
labels:
  - ai
assignees: []
createdAt: '2026-09-26T13:48:21Z'
updatedAt: '2026-09-26T13:48:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/549'
author: neo-preview
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
# Two wake-envelope plants on one seat: which is authoritative, and should the report be louder than a warning?

## Problem

Found while implementing #532: the wake-envelope plant has no provisioning path, so it was installed by
hand. On the seat where that was measured, **two** plants sit in the same plugins directory:

```
neo-wake-envelope.js    16129B  Aug  1 19:00
neo-wake-envelope.mjs   19028B  Sep 26 09:20
```

Neither carries a `GENERATED` marker and they differ by ~3 KB. OpenCode loads every plugin it finds in
that directory, and nothing in the system records which one won or that they disagree.

The missing installer was the ticket's framing. This is the sharper half: hand-installing has **already
produced an ambiguous install**, and no check surfaces it. #548 makes the hook *report* sibling plants
rather than resolve them, deliberately — which file is authoritative is a maintainer decision.

## The decision this ticket needs

1. **Which file is authoritative** for a seat that has both — or should provisioning refuse to proceed
   until the ambiguity is reconciled?
2. **Should an ambiguous install block seat launch**, or warn and continue? Today it warns per launch.
   The cost of a warning nobody reads is that the next seat starts the same way.
3. **Is `.js` a legacy name with no loader**, or is it genuinely loaded? That determines whether the
   Aug 1 file is inert or actively shadowing the current plant. I could not determine this from the seat
   alone, and guessing is how the ambiguity got here.
4. **Should the generator ever reconcile rather than report?** #548 deliberately never clobbers or
   deletes. If the answer is "the generated artifact always wins", that is a defensible position and it
   is the opposite of what shipped — so it should be a decision on the record, not a default.

## Why this is a ticket and not a line in #532

Picking an authoritative file means choosing between a file I did not write and one I did, on a seat
whose loader behaviour I cannot observe from here. That is a maintainer call, and #532's scope is the
installer.

## AC

- [ ] It is established which of the two files the harness actually loads, with evidence rather than
      inference from file names.
- [ ] A maintainer decision is recorded for which file is authoritative, and what provisioning does when
      it finds an ambiguous install.
- [ ] A seat carrying two plants is brought to a single, recorded state.

Refs #532

## Timeline

- 2026-09-26T13:48:22Z @neo-preview added the `ai` label

