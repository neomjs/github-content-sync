---
id: 182
title: Map the Neo organization from the Brain README
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-gpt
createdAt: '2026-08-27T08:49:33Z'
updatedAt: '2026-08-27T10:12:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/182'
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
closedAt: '2026-08-27T10:12:17Z'
---
# Map the Neo organization from the Brain README

## Context

Brain `README.md` is currently a 23-line migration shell. It identifies the repository as Agent OS, links Engine, states the temporary SHA rule and points at three guides, but it does not explain the parent Neo organism, the product relationship, the organization repository map, operating modes or a coherent learning path.

The operator directed that the mature Engine README be used literally as the baseline—not that Brain invent a new isolated identity. Brain then segments that shared narrative for its own audience and carries a visible “you are here” marker.

Live latest-open sweep: checked the latest 20 open Brain issues immediately before creation; no README ticket exists. Brain `#10` owns received guide/link/index integrity; this ticket owns the repository front door only. A2A in-flight sweep: latest 30 messages across read states contained no competing Brain README claim.

## The Problem

A cold visitor cannot understand from the README:

- what Neo is as a parent organism;
- how Brain relates to Engine, Institution, DevIndex and Skills;
- whether this repository is the Agent OS product, an internal package, or a migration staging area;
- which install/Host/Cloud/learning door to follow.

The README also over-indexes the temporary cut pin, while durable product framing and repository navigation are absent.

## The Architectural Reality

README is the ADR 0018 apex/framing surface for this repository. Audience segmentation permits Brain-specific install and architecture content, but it must remain compatible with the canonical organization Introduction. The Introduction stays canonical; copying it as a second independently edited file would recreate SSOT drift.

Brain `#10` separately owns `learn` links, entrypoint/index and fresh-clone readability. Brain `#12` owns final Host/Cloud package/deployment topology. This ticket links those surfaces; it does not implement them.

## The Fix

Replace the transition README using Engine README as the baseline:

1. Preserve/adapt the organization-level “What is Neo?” narrative and link the canonical Introduction.
2. Add the five-repository map:
   - `neomjs/neo` — Body / Engine;
   - `neomjs/neo-agent-brain` — Brain / Agent OS **← you are here**;
   - `neomjs/neo-agent-institution` — Institution application;
   - `neomjs/devindex` — developer/repository intelligence;
   - `neomjs/neo-agent-skills` — canonical Skills substrate.
3. Replace Engine-specific quickstart/details with Brain-specific installation, local/cloud operating modes, Host/Cloud architecture status, MCP services and deployment/learning doors.
4. Keep the temporary Engine SHA rule as a clearly bounded transition note rather than the README's identity center.
5. Route learning through the Brain README/index authority owned by Brain `#10`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| README apex | Engine README baseline + canonical Introduction + ADR 0018 | Parent organism first, Brain audience segmentation second | Preserve exact baseline paragraphs when no segmentation is required | README | Side-by-side framing audit |
| organization map | live neomjs product repositories + operator direction | Five links, one-line ownership, Brain-only marker | None: missing sibling is incomplete | README | Live link check |
| Brain quickstart/doors | Brain package, `#10`, `#12` | Honest current install/operate/learn routes with transition status named | If a final Host/Cloud path is not shipped, label it pending rather than inventing | README | Fresh clone + command/link validation |

## Decision Record impact

Aligned-with ADR 0018 and ADR 0040. No ADR change.

## Acceptance Criteria

- [ ] README is based on the mature Engine README organization narrative rather than the current migration shell.
- [ ] Five repositories are linked with accurate ownership and Brain alone marked “you are here.”
- [ ] Canonical organization Introduction is linked, not forked into a second independently maintained copy.
- [ ] Brain install, operating-mode, Agent OS architecture and learning doors are explicit and honest about transitional state.
- [ ] Temporary SHA-pin prose is bounded as transition information rather than the primary identity.
- [ ] Brain `#10` remains owner of `learn` index/tree/link implementation; Brain `#12` remains owner of Host/Cloud package/deployment implementation.
- [ ] All links and documented commands pass a fresh-clone/readability validation.
- [ ] Cross-family identity review completes before merge.

## Out of Scope

`learn/tree.json` implementation · guide rewrites · Brain `package.json` · Host/Cloud refactor · Portal/Pages/middleware · Engine README changes · generated SEO.

## Avoided Traps

- Keeping a 23-line repository shell because it satisfied the literal move ticket.
- Copying the organization Introduction into Brain as a divergent SSOT.
- Copying Engine quickstarts without replacing their product subject.
- Pretending deferred Host/Cloud topology is already shipped.

## Related

Learning/index owner: Brain `#10`  
Literal receive baseline: Brain `#13`  
Package/deployment topology: Brain `#12`  
Identity authority: ADR 0018

Origin Session ID: 14f81b5f-435b-4f00-92fc-67503eb0602e

Retrieval Hint: `Brain README Engine baseline five repository map you are here Agent OS`


## Timeline

- 2026-08-27T08:49:34Z @neo-gpt added the `documentation` label
- 2026-08-27T08:49:34Z @neo-gpt added the `enhancement` label
- 2026-08-27T08:49:34Z @neo-gpt added the `ai` label
- 2026-08-27T08:49:35Z @neo-gpt added the `agent-os` label
- 2026-08-27T09:19:01Z @neo-gpt assigned to @neo-gpt
- 2026-08-27T09:28:44Z @tobiu cross-referenced by PR #185
- 2026-08-27T10:02:35Z @tobiu referenced in commit `b905d8b` - "docs(readme): map the Brain in the Neo organization (#182)"
- 2026-08-27T10:12:17Z @tobiu referenced in commit `864ead2` - "Merge pull request #185 from neomjs/codex/182-brain-readme

docs(readme): map the Brain in the Neo organization (#182)"
- 2026-08-27T10:12:17Z @tobiu closed this issue
- 2026-08-27T14:11:48Z @neo-gpt cross-referenced by #186

