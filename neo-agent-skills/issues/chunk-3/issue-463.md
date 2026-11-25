---
id: 463
title: pr-review misses closed PRs and ADR-less VDom update-path diffs
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
assignees:
  - neo-opus-ada
createdAt: '2026-11-25T08:52:32Z'
updatedAt: '2026-11-25T11:11:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/463'
author: neo-opus-ada
commentsCount: 1
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[x] 449 Add plugins/context-compaction-memory magnet and standalone fallbacks'
---
# pr-review misses closed PRs and ADR-less VDom update-path diffs

Owned follow-through on an operator rule from 2026-11-25. #465 implements it; the operator confirms or rescopes the path list at his merge gate (AC-4).

## Context

On 2026-11-25 the operator ruled on VDom engine changes. Grace quotes him verbatim in neomjs/neo#22178 (issuecomment-7157404019): they are *"as sensitive, fragile and CRUCIAL to get right as the neo core itself"*, and *"changes on this gravity NEED AN ADR … this has the potential to SLOW DOWN HOTPATHS … changes like this OVERCOMPLICATE NEO"*.

At about 09:45Z he tightened the order. Grace relayed it in her `[operator-rule]` A2A: *"vdom needs: short but fully on point guides about how the vdom engine works inside the app worker, vdom worker, main threads. the design must be crystal clear. THEN we need ADRs."* So the engine guides come first, then an ADR built on them, then code. Until the guides land, no seat changes the update path. Grace owns the guides.

The review that should have caught neomjs/neo#22178 did not. For neomjs/neo#21974, that PR rebuilt the design the operator had rejected two days earlier on neomjs/neo#22117 ([closing record](https://github.com/neomjs/neo/pull/22117#issuecomment-7115040888)), in the same two engine files: `src/manager/VDomUpdate.mjs` and `src/mixin/VdomLifecycle.mjs`. My Round-1 review (Ada) requested changes on two test items only and wrote "the source shape is right… no source change is required". I have since retracted it ([retraction](https://github.com/neomjs/neo/pull/22178#issuecomment-7157427765)). Two gaps in pr-review let it through:

1. Nothing sends a reviewer to the PRs that already tried the close-target. §0 item 1 lists "the ticket/issue, the changed-file list, the current `dev` source …, sibling precedent, and the source-of-authority substrate", and the SKILL.md prior-art gate is a `query_raw_memories` / `query_summaries` sweep. `gh pr list --repo neomjs/neo --state all --search 21974` lists neomjs/neo#22117 as closed, with its closing record; the memory sweep did not surface it.
2. The premise catalog has no row for an engine update-path diff, so such a diff gets scored on tests and CI like any bug fix.

Grace files the author and intake side separately: pull-request (closing a PR on a dead premise closes or rewrites its ticket in the same act) and ticket-intake (a ticket whose PR was closed on premise rejection is not `valid-as-written` until that record is answered).

## The Architectural Reality

- `.agents/skills/pr-review/references/pr-review-guide.md` §0 item 1: the patch-blind inputs.
- `.agents/skills/pr-review/SKILL.md`: the prior-art sweep gate.
- `.agents/skills/pr-review/audits/cycle-1-premise-preflight.md`: the Trigger Catalog. It has eight rows, and a firing row ends the review at one premise RA, Drop+Supersede by default. The file is 3,332 B.
- `scripts/lint-skill-corpus.mjs` `COMBINED_BUDGETS` → `pr-review loaded surface`: guide (32,810 B) + circuit breaker (2,440 B) = 35,250 B, against a largest legal sum of 41,356 B. The catalog is outside that budget.

Design authority: "changes on this gravity NEED AN ADR" — the operator, 2026-11-25, quoted on neomjs/neo#22178 (issuecomment-7157404019); the guides-first order is his 09:45Z tightening, quoted under Context.

## The Fix

1. **§0 item 1:** add, in one clause, "every PR that named the close-target (`gh pr list --state all --search "<#N>"`), with each closed one's closing comment". A design rejected there ends the review at that record, as one premise RA.
2. **Premise catalog:** add one row, "Engine update path without guides and ADR". Its question: does the diff change the VDom update path without building on the engine guides (App worker, VDom worker, Main) and on an accepted ADR built on them? The ADR states the researched update model, the measured hot-path cost, and a real app flow that fails on `dev`. Until the guides land, every such diff fires the row. Once they exist, the row links them by path. The interim path list (Grace and Ada's reading; the operator's word decides):
   - `src/vdom/**`, `src/util/vdom/**`, `src/worker/VDom.mjs`
   - `src/main/DeltaUpdates.mjs` and the Main render queue
   - `src/manager/VDomUpdate.mjs`, `src/mixin/VdomLifecycle.mjs`, the Container `needsVdomUpdate` cascade
3. **Bytes:** fold the §0 clause into item 1's existing sentence. Offset the catalog row by tightening a neighbouring row, or state its retirement trigger.

## Acceptance Criteria

- [ ] §0 item 1 names the closed-PR sweep for the close-target, and what a rejected-design hit does.
- [ ] The premise catalog carries the update-path row: the path list, the order (guides → ADR → code), the ADR's three contents, and the operator rule's anchors.
- [ ] `npm run lint` passes, including the combined pr-review budget, and the PR states the byte delta per file.
- [ ] The operator confirms or rescopes the path list at his merge gate: his merge confirms it, and a rescope before it amends the catalog note.

## Post-Merge Validation

- [ ] When neomjs/neo#22189's guides land: the path list moves into them and the catalog note links them (Ada).

## Out of Scope

- The author and intake side (Grace's ticket).
- The engine guides themselves (Grace's lane).
- A path-keyed CI lint. Agreed: only if a skill rule demonstrably fails to fire, since the operator named overcomplication as a cost.
- The release-gate wording in neomjs/neo `ROADMAP.md` (Ada, at the pre-cut refresh).

## Avoided Traps

- **A memory-only rule.** Two seats had the record within reach, Grace on neomjs/neo#21974's body and Ada on the review, and no slot fired for either.
- **An "an ADR exists" check.** The operator doubts an ADR alone is enough. The row asks whether the ADR builds on the guides.
- **Expanding §0 with prose.** The pr-review surface is byte-governed, and one clause carries the sweep.

## Related

neomjs/neo#21974 (closed, not planned), neomjs/neo#22117, neomjs/neo#22178 (closed), and neomjs/neo#22168, held on `src/main/DeltaUpdates.mjs` under the same rule. The engine guides are Grace's lane, not filed yet.

Decision Record impact: none. This encodes an operator rule in a review payload; no ADR is amended.

Live latest-open + closed sweep: latest open issues (newest #451) + open and closed searches ("VDom ADR", "closed PR close-target", "premise catalog engine", "ADR premise review vdom", "prior PR sweep review", "update path ADR"; no same-surface hit) at 2026-11-25T08:51Z; no equivalent or prior exclusion found. Open count 49 (orange band); no owning ticket to fold into.
A2A in-flight sweep: all read-states since 08:00Z at 08:51Z. Grace and Ada agreed this split (Grace files author + intake); no competing claim.
MC sweep: "review approved engine change that operator had rejected before; closed PR for same ticket missed; VDom change needs ADR" → Grace's 11-25 diary only (same incident).
Own-assignment sweep: #445, #298 (other surfaces).
Meta-skill sweep: `create-skill/SKILL.md` consulted. Payload-only edits (a reference + an audit); the `SKILL.md` router is untouched.
Structure map: N/A in neo-agent-skills (the script is Brain-hosted).

Origin Session ID: 69b62d78-792a-4fa8-ba57-b8d6cfab8843
Retrieval Hint: `query_raw_memories("VDom engine change needs an ADR, review missed the closed PR 22117 for 21974")`


## Timeline

- 2026-11-25T08:52:34Z @neo-opus-ada added the `enhancement` label
- 2026-11-25T08:52:34Z @neo-opus-ada added the `ai` label
- 2026-11-25T08:52:34Z @neo-opus-ada added the `architecture` label
- 2026-11-25T08:52:36Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-25T08:52:43Z @github-actions added the `needs-design-authority` label
- 2026-11-25T09:52:31Z @neo-opus-grace cross-referenced by #22189
### @neo-opus-ada - 2026-11-25T10:40:42Z

**Contract Ledger (T3)**, posted before the PR opens. Implementation starts now. AC-4, the operator's word on the path list, gates the merge, not the edit.

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `pr-review/references/pr-review-guide.md` §0 item 1 (the patch-blind inputs) | Operator rule of 2026-11-25 ([neo#22178](https://github.com/neomjs/neo/pull/22178#issuecomment-7157404019)), this ticket | The inputs include every PR that named the close-target (`gh pr list --state all --search <N>`). A design rejected in a closed PR's closing record ends the review there, as one premise RA. | If no PR names the target, the input is empty and the review proceeds. | The guide itself | skill-corpus lint (combined pr-review budget), the diff |
| `pr-review/audits/cycle-1-premise-preflight.md` Trigger Catalog row "Engine update path without guides and ADR" | Same rule, and the guides lane neo#22189 | Fires on a diff that changes VDom update-path behavior without the engine guides (App worker, VDom worker, Main) and an accepted ADR built on them. The ADR must state the researched update model, the measured hot-path cost, and a real app flow failing on `dev`. A firing row means one premise RA, Drop+Supersede by default. | Until the guides land, every such diff fires the row. Once they land, the path list moves into them and the note links them. | The catalog note lists the paths: `src/vdom/**`, `src/util/vdom/**`, `src/worker/VDom.mjs`, `src/main/DeltaUpdates.mjs`, the `src/Main.mjs` read/write queues, `src/manager/VDomUpdate.mjs`, `src/mixin/VdomLifecycle.mjs`, and the Container `needsVdomUpdate` cascade | The diff now. After merge, the row's first firing on a real update-path review. |

Surface-anchor V-B-A: every named path resolves on neo `dev` at `6e77da5d72` (`ls`). The render queue is `src/Main.mjs` `readQueue`/`writeQueue` via `processQueue`. `needsVdomUpdate` is in `src/container/Base.mjs`.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-25T10:41:19Z @neo-opus-ada cross-referenced by PR #465
- 2026-11-25T10:42:50Z @neo-opus-ada referenced in commit `bdf2e8d` - "chore(release): 2.8.5 carries the pr-review premise row (#463)"
- 2026-11-25T11:10:48Z @neo-opus-ada marked this issue as being blocked by #450
- 2026-11-25T11:11:00Z @github-actions removed the `needs-design-authority` label
- 2026-11-25T11:14:24Z @neo-gemini-maryam cross-referenced by PR #22192
- 2026-11-25T11:21:27Z @neo-opus-ada removed the block by #450
- 2026-11-25T11:21:48Z @neo-opus-ada referenced in commit `949d643` - "chore(release): 2.8.2 lands #463 ahead of the plugin stack (#463)"
- 2026-11-25T11:29:14Z @neo-gemini-julia marked this issue as blocking #449
- 2026-11-25T11:31:34Z @neo-gemini-julia cross-referenced by PR #460
- 2026-11-25T11:33:10Z @neo-gemini-julia cross-referenced by PR #461
- 2026-11-25T11:33:14Z @neo-gemini-julia cross-referenced by PR #462
- 2026-11-25T11:54:55Z @neo-opus-ada referenced in commit `f6b2ee2` - "Merge origin/dev into ada/463 after the 2.8.3 release (#463)"

