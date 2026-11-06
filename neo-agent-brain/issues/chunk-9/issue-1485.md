---
id: 1485
title: Candidate readers take who exists from the plane's identity nodes
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-05T16:39:10Z'
updatedAt: '2026-11-05T19:10:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1485'
author: neo-opus-vega
commentsCount: 1
parentIssue: 875
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-05T18:08:39Z'
---
# Candidate readers take who exists from the plane's identity nodes

## Context

This is the second half of class A in #875's census: who exists. #1484 stops the boot seed from provisioning our roster. Four readers still take "who exists" from `identityRoots.IDENTITIES`, so on another operator's plane our agents count and theirs do not.

The prescription was checked module by module on #1484 (comment 6663307247). The split follows the rule that a PR resolves one ticket.

## The Problem

| Module | Read | Effect on another operator's plane |
|---|---|---|
| `ai/daemons/orchestrator/scheduling/swarmHeartbeat.mjs:198` | `active-local-team` is the roster's AgentIdentity entries; its doc (`:73-76`) tells forks to edit `identityRoots.mjs` | under that opt-in source, their agents never become heartbeat targets. The default source, `active-a2a-participants`, is already graph-derived |
| `ai/services/graph/issueFocusSections.mjs:79` `getStaleAssignmentMaintainers()` | maintainer logins for stale-assignment acknowledgements | their maintainers' progress comments never count |
| `ai/services/ingestion/IssueIngestor.mjs:208` `internalAuthorLogins` | built from the roster at import | every ticket their team writes is boosted as community-authored |
| `ai/services/fleet/wireFleetOpenWorkSource.mjs:38` `seatIdentities` | `byLogin` falls back to the roster after the registry; `byName`, which resolves the PR body's author line, uses the roster only | their PRs resolve to no seat (`unowned`) |

## The Architectural Reality

- Design authority: #875's recorded ruling (the operator, 2026-10-05, on #874), *"agent os and MCP tools should never use them. this only works for OUR own team, but can not work for other operators who simply do not even have that file for them."*
- `ai/graph/agentIdentityParticipation.mjs` is the plane-side reader: `readAgentIdentityNodes(db)`, `participationByIdentity` and `participationStatusOf`. Wake delivery, the heartbeat's participation gate and issue focus already read through it.
- `SwarmHeartbeatService` already passes `resolveTargets` a `participationProvider` that reads those nodes.
- `issueFocusSections.mjs:1162` already reads the nodes for stall inference, while the maintainer list beside it comes from the roster.
- `IssueIngestor` writes through `Memory_GraphService`, so the graph is mounted where it runs.
- The open-work producer runs in the Fleet server, which reads the plane through `who_is_online` (`readPresence`) every pulse, as `benchedIdentities` already does. Its rows carry `identity` and `name`.
- `active-local-team` is also the default team of `checkAllAgentIdle.mjs`, the all-agent-idle detector, so it is a production path on our plane too.
- On our plane the nodes are not the same set as the roster: the graph holds 18 AgentIdentity nodes, the roster 14 (read 2026-11-05T17:2xZ).
  - Three of the extra nodes have empty properties: `@agent-a`, `@agent-b` and `@neo-test-writer`.
  - The fourth, the retired `@neo-opus`, is still `active` (defect-note fb9537b3).
- Membership therefore needs a fact on the node, and `accountType` is that fact. Auto-provisioned principals carry `accountType: 'agent'` (`Server.ensureAgentIdentityForAuthContext`).

## The Fix

1. `active-local-team`: the candidates are the plane's nodes with `accountType: 'agent'`, gated by participation, and the doc describes the graph as the team. The human owner stops being a pulse target, since keeping it was an artifact of the roster.
2. `getStaleAssignmentMaintainers(nodes)`: maintainers come from AgentIdentity nodes whose `accountType` is `agent` or `human`. Callers pass the nodes they already read.
3. `internalAuthorLogins`: read from the nodes (`githubLogin` plus `formerLogins`) when an ingest runs, not at import.
4. `seatIdentities`: `byName` and `byLogin` resolve through the plane's identity rows from the pulse's presence read, never through the roster.

## Acceptance Criteria

- [ ] AC-1: For each reader, a unit arm shows an AgentIdentity node outside the roster counted:
  - as a heartbeat target under `active-local-team`;
  - as a stale-assignment maintainer;
  - as an internal author;
  - as a seat, by name and by login.

  A roster entry with no node is counted as none of these.
- [ ] AC-2: None of the four modules imports `identityRoots` at runtime. The runtime modules that import it, outside `ai/scripts/`, drop by four from #1484's 16.
- [ ] AC-3 (post-merge): on our own plane, the `active-local-team` set is the roster's agents, read from the nodes. `@neo-opus` is in it only until it is benched. Issue-focus maintainers are unchanged.

## Out of Scope

- The boot seed (#1484).
- Classes B (trust tier), C (family: #700), D (display) and E (vocabulary, team tooling).
- The Fleet admission writer (#1297), which gives a fresh plane's agents their nodes.
- Deleting existing nodes.

## Decision Record impact

`aligned-with` #875's ruling. No ADR changes.

## Related

#875 (parent) · #1484 (sibling: the boot seed) · #700 / #1296 / #1297 (class C and admitted membership) · #663 (precedent: a write guard that stopped requiring a roster entry)

Live latest-open + closed sweep:
- Live: the latest 20 open Brain issues at 2026-11-05T16:38Z.
- Closed searches: "heartbeat candidates identity nodes", "issue focus maintainers roster", `internalAuthorLogins`, "open-work seat identities roster", `active-local-team`.
- No equivalent ticket. I read the Out of Scope sections of #880 and #1008; neither covers who counts as a candidate.
- A2A: no claim on these readers in the latest 30 messages.
- Memory Core: no prior decision on them.
- Own open assignments on this surface: #875 (the parent) and #1484 (the boot seed).

Ownership: mine (`@neo-opus-vega`). The class A outcome under #875 is mine, and the prescription is already checked on #1484.

Retrieval Hint: "who exists candidate readers heartbeat issue focus maintainers internal authors open-work seat identity nodes class A"

Authored by Vega (Opus 5.5, Claude Code)


## Timeline

- 2026-11-05T16:39:12Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-05T16:39:12Z @neo-opus-vega added the `enhancement` label
- 2026-11-05T16:39:13Z @neo-opus-vega added the `ai` label
- 2026-11-05T16:39:13Z @neo-opus-vega added the `architecture` label
- 2026-11-05T16:39:13Z @neo-opus-vega added the `agent-os` label
- 2026-11-05T16:39:25Z @neo-opus-vega cross-referenced by #1484
- 2026-11-05T16:39:29Z @neo-opus-vega added parent issue #875
- 2026-11-05T16:39:52Z @neo-opus-vega cross-referenced by #875
- 2026-11-05T16:49:08Z @neo-opus-vega cross-referenced by PR #1486
- 2026-11-05T17:43:56Z @neo-opus-vega cross-referenced by PR #1487
- 2026-11-05T18:08:39Z @tobiu referenced in commit `7519932` - "fix(graph): candidate readers take who exists from the plane's nodes (#1485) (#1487)

Four readers still took "who exists" from identityRoots.IDENTITIES, so on
another operator's plane our team counted and theirs never did:
- heartbeat active-local-team, also checkAllAgentIdle's default team;
- stale-assignment maintainers;
- IssueIngestor's internal authors;
- the open-work seat lookup.

Membership is now a fact on the node: agentIdentitiesOf() keeps the
AgentIdentity nodes whose accountType is agent, and memberLoginsOf() the
agents and people with their logins. A node that records no account type
is no member, which keeps out the empty fixture nodes our own plane holds.

- resolveTargets takes a teamProvider for active-local-team.
  SwarmHeartbeatService and checkAllAgentIdle pass the plane's agents.
  Without one, the source falls back to self.
- getStaleAssignmentMaintainers(nodes); buildStaleAssignmentCandidates
  reads the plane when no maintainers are passed.
- IssueIngestor.readInternalAuthorLogins() reads the nodes once per ingest
  run instead of at import.
- seatIdentities resolves through the plane's who_is_online rows. The
  pulse's one presence read keeps them for the lookups that follow.

The specs build their teams from nodes they seed rather than from the
roster boot seeds."
- 2026-11-05T18:08:41Z @tobiu closed this issue
### @neo-opus-vega - 2026-11-05T19:10:19Z

**AC-3 receipt (post-merge), 2026-11-05: the team half passes; the maintainers half is not yet verified.** The plane was cut from Brain `15dbfb38` to dev `5b042434` at 19:09Z, with all four services healthy at that revision.

**`active-local-team`,** read through its other consumer, `checkAllAgentIdle` (`docker exec … node ai/scripts/lifecycle/checkAllAgentIdle.mjs`, which prints its `identities`). The live heartbeat's `targetSource` is `active-a2a-participants`, so the pulse itself doesn't show this set.
- **Before, on `15dbfb38`, roster-based:** 11 identities, the 10 roster agents plus `@tobiu`.
- **After, on `5b042434`, node-based:** 11 identities: `@neo-fable-clio`, `@neo-fable-mnemosyne`, `@neo-gemini-julia`, `@neo-gemini-maryam`, `@neo-gpt-emmy`, `@neo-gpt-euclid`, `@neo-gpt-sophie`, `@neo-opus-ada`, `@neo-opus-grace`, `@neo-opus-vega`, and `@neo-opus`.

That is the roster's agents read from the nodes, plus `@neo-opus` until it is benched (Brain #28). `@tobiu` is no longer a pulse target, and the three fixture nodes with no `accountType` stay out, as the PR said.

**"Issue-focus maintainers are unchanged":** not verified. I took no before-reading of that set before the cut. The probe is the next Dream-pipeline handoff: compare its issue-focus maintainers with the last pre-cut handoff's. Until then, AC-3 stays unticked.

— Vega (Opus 5.5, Claude Code) 🌿


- 2026-11-05T19:12:54Z @neo-opus-vega cross-referenced by #1490

