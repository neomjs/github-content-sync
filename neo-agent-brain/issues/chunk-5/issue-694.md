---
id: 694
title: OwnAgentTeam.md still makes identityRoots the identity authority
state: CLOSED
labels:
  - bug
  - documentation
  - ai
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-10-01T14:56:33Z'
updatedAt: '2026-10-01T16:12:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/694'
author: neo-fable-clio
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
closedAt: '2026-10-01T16:12:08Z'
---
# OwnAgentTeam.md still makes identityRoots the identity authority

## Context

`learn/agentos/OwnAgentTeam.md` — the guide an outside team follows to stand up its own agents — still says (section *Define Identity Roots*) "the identity root source of truth is `ai/graph/identityRoots.mjs`. Add one `AgentIdentity` entry per teammate", makes *Seed The Graph* a required step, lists "the `@<login>` node exists in `ai/graph/identityRoots.mjs`" as a blocker in *Verify A Teammate*, and opens *Bring Up The Team* with "Add one identity root". The operator ruled otherwise on 2026-10-01 (and on 2026-08-21 before it): that file WAS the authority before dockerization and the ingress; identity is **credential-derived now** — the seat's PAT (or OIDC subject) names the login — and the file is Neo's own team roster, which other teams do not have and do not need. Captured as a defect-note by @neo-gpt-emmy (2026-10-01 14:52Z, while reviewing #681) and routed to the onboarding lane; promoted here.

## The Problem

A reader who follows the guide today treats a missing roster entry as a login failure and edits a file that is not theirs to edit. Sophie's first day was that reader: her seat authenticated but the write guard refused her as "unmappable in identityRoots" — a guard defect fixed by #663 / PR #665 (merged, in the installed app since the 2026-10-01 interim install). The guide was never corrected to match the ruling or the repair, so it now prescribes the exact detour the repair removed.

## The Architectural Reality

- `ai/configBase.mjs:624` — `auth.mode` selects the identity source: `'oidc'` (default) | `'gitlab-pat'` | `'github-pat'` | `'local-bearer'` | `'seat-token'`; `ai/mcp/server/shared/services/AuthService.mjs` validates the credential and derives the login from it.
- Identities that are not in the static roster are **runtime-provisioned at request time** and live only in the graph (`ai/services/graph/agentFamilyResolution.mjs:126`; the mailbox and wake readers carry the retirement-gated fallback: `MailboxService.mjs:231`, `WakeSubscriptionService.mjs:1071`).
- `ai/graph/assertExpectedIdentity.mjs` (after #665) compares the reference identity with the authenticated login without any roster read.
- `ai/graph/identityRoots.mjs` is the Neo team's roster: display name, co-author email (`agentCoAuthorEmails.mjs`), trust tier, participation status, character choices — enrichment, never admission (its header says so since Sophie's #661 / PR #693 draft).
- The guide's *Bind Harnesses* section (NEO_AGENT_IDENTITY as the local stdio pin) stays correct; so does the Fleet's injection of the seat's PAT.

## The Fix

Rewrite the four sections and the two rows that depend on them, nothing else:
1. **Identity Layers** — the "Graph identity" row: the `AgentIdentity` node is provisioned from the authenticated login at first contact; `identityRoots.mjs` is optional roster metadata.
2. **Define Identity Roots** → *Where identity comes from*: the seat's own credential (`auth.mode`); no file entry is required; then the optional roster paragraph (what it adds, who edits it — Neo's own team), keeping the example as an *optional* enrichment.
3. **Seed The Graph** → optional, only when a roster is kept; the script stays documented as the refresh path for that case.
4. **Verify A Teammate** — the blocker list verifies the credential path (`healthcheck.identity` bound from the credential; `gh api user` through the seat's PAT); the roster node is optional and never a blocker.
5. **Bring Up The Team** — step 1 becomes "give the seat its own PAT (the Fleet injects it)"; the roster steps become one optional step.
6. **Source Anchors** — add `ai/configBase.mjs` (`auth.mode`), `AuthService.mjs`, `assertExpectedIdentity.mjs` (#665); keep the roster anchors with their optional role.

No section outside these moves; #681's hunks (lines 239–340, the memory recipe) are untouched, so the two PRs merge independently.

## Acceptance Criteria

- [ ] AC-1 No sentence in the guide names `identityRoots.mjs` as a source of truth, a prerequisite, or a verification blocker (`grep -n "identityRoots" learn/agentos/OwnAgentTeam.md` in the PR body, every hit read as optional).
- [ ] AC-2 The credential-derived path is the one the guide verifies: `auth.mode` named with its five values and the file that validates it; *Verify A Teammate* lists the credential check first and the roster not at all as a blocker.
- [ ] AC-3 The roster paragraph states what the file adds and that outside teams need no entry; the example is marked optional.
- [ ] AC-4 Docs-only diff inside the named sections; the `#681` hunks untouched (diff hunk headers in the PR body).

## Out of Scope

The roster file itself (#661 / PR #693, Sophie); the write guard (#665, merged); the README roster (neo#19345, merged); any change to `NEO_AGENT_IDENTITY` binding.

## Avoided Traps

Deleting the roster from the guide (it is still Neo's own team's file and the co-author email map's source); turning the correction into a bootstrap path that seeds every outside operator (the #663 anti-pattern); editing #681's sections in the same PR.

## Related

#663 · PR #665 (the guard repair) · #661 / PR #693 (Sophie's roster entry) · #681 (the same file, other sections) · neo#15990 (fail-closed provider-PAT auth, rosterless profile) · the operator's rulings 2026-08-21 and 2026-10-01

Decision Record impact: aligned-with ADR 0019 (`auth.mode` is a declared leaf); none amended.

Sweeps: live latest-open sweep — the latest 20 open Brain issues at 2026-10-01T14:55:06Z (newest #690), no equivalent; A2A in-flight sweep — the last messages at 14:53Z: Emmy's defect-note (the capture this promotes), Sophie's #661 resume, no claim on the guide; Memory Core rationale sweep (`query_raw_memories`) — the trail: operator 2026-08-21 (via Vega) and 2026-10-01 rulings, Mnemosyne's 2026-06-10 finding (the roster is wrong-for-tenants: universal core vs deployment roster + auto-registration), Euclid's 2026-07-26 PAT→login read, Emmy's #663 session; own-assignment sweep — #37, #50, #51, #53, none on this; structure map — N/A (docs).

Origin Session ID: 6682a116-897e-4c18-925e-4320d0489481
Retrieval Hint: "OwnAgentTeam identityRoots optional roster credential-derived identity auth.mode guide correction"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 6682a116-897e-4c18-925e-4320d0489481

## Timeline

- 2026-10-01T14:56:34Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-01T14:56:36Z @neo-fable-clio added the `bug` label
- 2026-10-01T14:56:36Z @neo-fable-clio added the `documentation` label
- 2026-10-01T14:56:36Z @neo-fable-clio added the `ai` label
- 2026-10-01T14:56:36Z @neo-fable-clio added the `agent-os` label
- 2026-10-01T14:58:49Z @neo-fable-clio cross-referenced by PR #695
- 2026-10-01T15:15:27Z @neo-fable-clio cross-referenced by #696
- 2026-10-01T15:16:05Z @neo-fable-clio cross-referenced by #697
- 2026-10-01T15:19:51Z @neo-fable-clio referenced in commit `108835c` - "docs(guide): OwnAgentTeam derives a teammate's identity from its credential; the roster is optional (#694)"
- 2026-10-01T15:19:51Z @neo-fable-clio referenced in commit `8db9b38` - "docs(guide): OwnAgentTeam qualifies identity provisioning and verification by auth mode (#694)"
- 2026-10-01T15:32:51Z @neo-opus-grace cross-referenced by #700
- 2026-10-01T15:38:49Z @neo-fable-clio referenced in commit `429ea38` - "docs(guide): OwnAgentTeam derives a teammate's identity from its credential; the roster is optional (#694)"
- 2026-10-01T15:38:49Z @neo-fable-clio referenced in commit `22f8b66` - "docs(guide): OwnAgentTeam qualifies identity provisioning and verification by auth mode (#694)"
- 2026-10-01T15:38:49Z @neo-fable-clio referenced in commit `0c5a245` - "docs(guide): OwnAgentTeam states provisioning by listed source and limits the identity pin to stdio (#694)"
- 2026-10-01T16:12:08Z @tobiu referenced in commit `70fdb3c` - "docs(guide): OwnAgentTeam derives a teammate's identity from its credential; the roster is optional (#694) (#695)

* docs(guide): OwnAgentTeam derives a teammate's identity from its credential; the roster is optional (#694)

* docs(guide): OwnAgentTeam qualifies identity provisioning and verification by auth mode (#694)

* docs(guide): OwnAgentTeam states provisioning by listed source and limits the identity pin to stdio (#694)"
- 2026-10-01T16:12:08Z @tobiu closed this issue
- 2026-10-01T17:29:15Z @neo-gpt-sophie cross-referenced by #708
- 2026-10-01T17:54:04Z @neo-fable-clio cross-referenced by PR #709

