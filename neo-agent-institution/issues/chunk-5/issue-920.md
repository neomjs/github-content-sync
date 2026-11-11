---
id: 920
title: A refused PAT reads yellow while last-known data shows
state: OPEN
labels:
  - enhancement
  - ai
  - design
assignees: []
createdAt: '2026-11-11T18:39:47Z'
updatedAt: '2026-11-11T18:39:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/920'
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
# A refused PAT reads yellow while last-known data shows

## Context

The operator noticed this during row 5's H3 on 2026-11-11, on FM Institution `6f00d3da`: "pat refused showed up. in yellow, not red (left border color)." The design note is on [#424 comment 6803845378](https://github.com/neomjs/neo-agent-institution/issues/424#issuecomment-6803845378).

**Sweeps, 18:38:51Z:**
- **Live latest-open + closed:** 54 open, red band. Searched `pat refused tone`, `degraded cold tone` and `banner tone`. No equivalent.
- **A2A:** the 12 newest messages. No claim.
- **MC:** no prior decision.

## The Problem

The same refusal shows in two tones:
- **While the FM runs, the tone is yellow.** It is the `fm-spine-banner-degraded` class: "pat refused · The plane refused that PAT. Connect again with a current one." The roster still shows last-known data, header "0 started · 13 UNKNOWN".
- **After a relaunch, the tone is red.** The same text uses `fm-spine-banner-cold`, with an empty roster.

So the tone tracks whether data is on screen. But in both cases nothing updates until the operator acts: the plane accepts no reads until a new PAT is connected. A yellow refusal reads as "partially working", which it is not.

## The Architectural Reality

**Design authority**, from `apps/agentos/util/SpineBanner.mjs` `PLANE_REFUSALS`:
> "the credential's own two need a PAT, so they offer Connect"

That gives the action. No record I found sets the tone of an action-required refusal: neither #477 (a reason and a next step) nor `learn/CockpitStateCensus.md` names one. The tone comes from the generic data-presence split, degraded when data is shown and cold when there is none.

## The Fix

Recommendation: give the credential refusals (`plane-credential-refused`, `plane-identity-changed`) one action-required tone, the cold/red one, whatever the data on screen. Keep the last-known roster visible beneath it.

Record the rule in `learn/CockpitStateCensus.md`: a state that needs the operator's action wears the cold tone.

## Acceptance Criteria

- [ ] The decision is recorded in `learn/CockpitStateCensus.md`, either this rule or the reason to keep data-presence tones.
- [ ] If adopted: a `pat refused` banner over a last-known roster renders cold. Spine banner spec.

## Out of Scope

- The refusal's text and its Connect action, which are unchanged.
- The setup card after Connect, #918.

## Related

- #424 and #477.
- `#828` and `#845`: a revoked PAT reaching the spine.

unowned-rationale: A design call first; then a small, contributor-ready change.

Origin Session ID: b261f993-5994-4a53-bb9c-9fac974a9029
Retrieval Hint: "pat refused banner tone yellow degraded versus red cold"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-11T18:39:49Z @neo-opus-ada added the `enhancement` label
- 2026-11-11T18:39:49Z @neo-opus-ada added the `ai` label
- 2026-11-11T18:39:50Z @neo-opus-ada added the `design` label

