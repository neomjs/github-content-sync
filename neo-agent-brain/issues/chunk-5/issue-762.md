---
id: 762
title: The heartbeat digest renders a seat's open work from one projection
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-02T14:36:52Z'
updatedAt: '2026-10-02T14:36:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/762'
author: neo-opus-grace
commentsCount: 0
parentIssue: 759
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 760 One producer observes every open PR and projects each seat''s open work'
blocking: []
---
# The heartbeat digest renders a seat's open work from one projection

## Context

This is a reader leaf of #759, graduated from neomjs/neo#19122 (OQ4's second reader). The heartbeat digest renders a seat's open work from the producer's projection instead of deriving any of it.

## The Problem

The heartbeat digest names no seat's open PRs, red heads or due reviews. A seat finds them only by polling GitHub.

## The Architectural Reality

- `SwarmHeartbeatService` (`ai/daemons/orchestrator/services`) renders the digest. Its only GitHub source is the host account's notifications (`emitGitHubNotificationWakes`).
- The projection lives host-side in `fleetOpenWorkSource` (the observing leaf). The digest runs on the plane, so it needs the projection published there.

## The Fix

1. The producer publishes each seat's open work to the plane, alongside its transitions.
2. An MC read verb answers one seat's open work with the producer's freshness: its high-water time, and `ok · stale · unavailable`.
3. The heartbeat digest renders that verb's answer: open PRs (CI, review state) and requested reviews, or "stale since <time>" or "unavailable", never an empty list in their place.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Producer → plane publish | `fleetOpenWorkSource` | per-seat open work with the high-water time | a failed publish leaves the plane copy stale | JSDoc | unit |
| MC read verb (open work) | the plane copy | one seat's open work + freshness | no copy → `unavailable` | OpenAPI description + JSDoc | unit |
| Digest section | the verb | renders open work, or stale/unavailable | never an empty list in their place | JSDoc | unit |

## Acceptance Criteria

- [ ] AC-1: The verb answers one seat's open work with its freshness, and `unavailable` with no copy (unit).
- [ ] AC-2: The digest renders open PRs and requested reviews from the verb, and renders stale or unavailable as such, never as no work (unit).
- [ ] AC-3: Nothing in the digest or the verb reads GitHub (unit: no GitHub client in their module graph).

## Out of Scope

- The producer, the wakes, the plane's PR lane and the cockpit card.

## Decision Record impact

None.

## Related

#759 (parent) · neomjs/neo#19122 OQ4.

Sweeps: as on the observing leaf (2026-10-02T14:35Z). No equivalent.

unowned-rationale: filed at graduation for any seat to claim. It is blocked by the observing leaf.

Origin Session ID: 31c9ca1a-ded8-4b19-8d99-682d259efeca


## Timeline

- 2026-10-02T14:36:53Z @neo-opus-grace added the `enhancement` label
- 2026-10-02T14:36:54Z @neo-opus-grace added the `ai` label
- 2026-10-02T14:36:54Z @neo-opus-grace added the `agent-os` label
- 2026-10-02T14:37:18Z @neo-opus-grace added parent issue #759
- 2026-10-02T14:37:26Z @neo-opus-grace marked this issue as being blocked by #760
- 2026-10-02T14:37:48Z @neo-opus-grace cross-referenced by #759

