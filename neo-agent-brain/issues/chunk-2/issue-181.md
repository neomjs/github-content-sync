---
id: 181
title: Give the Brain package its own discovery identity
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - refactoring
  - agent-os
assignees:
  - neo-gpt
createdAt: '2026-08-27T08:49:31Z'
updatedAt: '2026-08-27T09:54:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/181'
author: neo-gpt
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
closedAt: '2026-08-27T09:54:40Z'
---
# Give the Brain package its own discovery identity

## Context

Brain `dev@cd251a90f6` has become the canonical Agent OS repository, but its `package.json` remains a move-first transition manifest: `version: 0.0.0`, `private: true`, a migration `$comment`, a one-line description, and **zero keywords**. Engine `dev@915c1a01e7` still holds all 66 pre-split discovery terms, including 26 Brain-specific identities.

Brain `#13` deliberately delivered only the minimum installable receive and deferred package refinement. Brain `#12` owns Host/Cloud/deployment topology. This ticket owns structured identity/discovery metadata only.

Live latest-open sweep: checked the latest 20 open Brain issues immediately before creation; no equivalent Brain package-identity ticket exists. A2A in-flight sweep: latest 30 messages across read states contained no competing Brain package-metadata claim.

## The Problem

Package registries, agents and cold readers cannot discover what the Brain contains from its metadata. The description does not name Memory Core, Knowledge Base, Native Edge Graph, A2A, GitHub Workflow, DreamService, multi-agent institutional memory or deployment modes. The missing keywords invert custody: Brain code moved, while its discovery vocabulary stayed in Engine.

The current manifest also lacks the durable structured metadata already present in Engine (`author`, `bugs`, `homepage`, keywords), while GitHub repository description/topics are separately blank.

## The Architectural Reality

`package.json` is a Class-3 identity surface under ADR 0018. Brain is one product repository inside the Neo organism, so its metadata must be Brain-specific while remaining compatible with the shared apex. It consumes Engine and Skills; it does not own Institution or Skills identity.

Package topology is not metadata. `version`, `private`, scripts, dependencies, lock state, Host/Cloud manifests and the transition pin remain governed by Brain `#12`, `#13`, and `#179` follow-through.

Structure-map evidence: scoped `ai:structure-map --root learn --files --loc` completed; this ticket creates no path or `.mjs` placement.

## The Fix

Edit Brain `package.json` only:

1. Replace the minimal description with a concise Agent OS / Brain package description.
2. Add a priority-ordered keyword set containing the 26 Brain-specific terms currently stranded in Engine plus justified shared organism/Neural-Link terms.
3. Add durable package identity fields that have clear existing authority (`author`, `bugs`, `homepage`) without inventing release/product facts.
4. Document exact intentional divergence from Engine package metadata in the PR body.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `package.json#description` | Brain README + canonical organization Introduction + ADR 0018 | Concisely identifies Brain as the Agent OS and names its primary mechanisms | Keep minimal description only if a cold reader can identify the product/capabilities from it | PR body | Cold-reader comparison |
| `package.json#keywords` | post-split subject custody + completed Engine `#12241/#12250` priority discipline | Brain terms first; shared terms intentional; no Engine UI keyword stuffing | Disputed term omitted pending evidence | PR body classification | Exact set/count diff + JSON parse |
| package identity fields | existing GitHub/npm coordinates | Adds only verifiable author/bugs/homepage values | Omit any field without live authority | package JSON | URL/readback validation |

## Decision Record impact

Aligned-with ADR 0018 and ADR 0040. No ADR change.

## Acceptance Criteria

- [ ] `package.json` remains valid JSON and no lockfile change is introduced.
- [ ] Description names Brain/Agent OS and its primary public capabilities.
- [ ] A priority-ordered Brain keyword list exists; every term has Brain/shared rationale.
- [ ] The 26 Brain-specific concepts currently stranded in Engine are dispositioned explicitly.
- [ ] Author/bugs/homepage fields, if added, resolve to live canonical coordinates.
- [ ] No change to `version`, `private`, scripts, dependencies, lock pin, Host/Cloud topology, or release mechanics.
- [ ] Cross-family identity review completes before merge.

## Out of Scope

README · `learn/tree.json` · GitHub repository description/topics · package topology · Engine metadata · npm publishing · dependency pin advancement.

## Avoided Traps

- Copying all 66 Engine keywords into Brain and preserving monorepo ambiguity.
- Smuggling Host/Cloud refactoring into metadata work.
- Inventing a public version/release state inside an identity ticket.

## Related

Receive baseline: Brain `#13`  
Package/deployment topology: Brain `#12`  
Pin/consumer closure: Brain `#179`  
Learning/front door: Brain `#10`

Origin Session ID: 14f81b5f-435b-4f00-92fc-67503eb0602e

Retrieval Hint: `Brain package description zero keywords Agent OS discovery identity`


## Timeline

- 2026-08-27T08:49:33Z @neo-gpt added the `documentation` label
- 2026-08-27T08:49:33Z @neo-gpt added the `enhancement` label
- 2026-08-27T08:49:33Z @neo-gpt added the `ai` label
- 2026-08-27T08:49:34Z @neo-gpt added the `refactoring` label
- 2026-08-27T08:49:34Z @neo-gpt added the `agent-os` label
- 2026-08-27T09:00:46Z @neo-gpt assigned to @neo-gpt
- 2026-08-27T09:06:02Z @tobiu cross-referenced by PR #183
- 2026-08-27T09:19:59Z @neo-gpt-emmy cross-referenced by #184
- 2026-08-27T09:43:23Z @neo-opus-vega cross-referenced by PR #185
- 2026-08-27T09:54:40Z @tobiu referenced in commit `e1b2436` - "Merge pull request #183 from neomjs/codex/181-brain-package-identity

docs(package): give Brain its discovery identity (#181)"
- 2026-08-27T09:54:40Z @tobiu closed this issue

