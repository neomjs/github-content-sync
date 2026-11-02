---
id: 62
title: 'Corpus-loss attribution: a before-and-after corpus series beside the bridge''s death record'
state: OPEN
labels:
  - enhancement
  - ai
assignees: []
createdAt: '2026-08-05T23:03:19Z'
updatedAt: '2026-11-02T18:49:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/62'
author: neo-opus-vega
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
# Corpus-loss attribution: a before-and-after corpus series beside the bridge's death record

**Rewritten 2026-11-02 against Brain `dev` `92a7ec44`** for the #1438 backlog sweep. The original body (filed 2026-08-05) is in this issue's edit history.

## What changed since filing

- **Death capture shipped for plane services** (#466, closed 2026-09-25). The deployment-state bridge reads Docker's `oom` and `die` events per service from a cursor on every snapshot cycle (`ai/daemons/orchestrator/services/DeploymentStateBridgeService.mjs:852`–`:905`; on by default, `ai/configBase.mjs:1698`). It keeps the last 10 deaths per service as `{at, exitCode, oomKilled}`, and both MCP healthchecks report the last one as `lastDeath`.
- **The retention premise is sharper.** The original body read Docker's event retention as zero. #466 found a bounded buffer instead, one that a 1 Hz `docker exec` sampler rotated within minutes. That is why the bridge reads a short window from a cursor rather than reconstructing a death after the fact.

## What remains

The bridge records why a container died. It does not record what the corpus looked like before and after the death. The bridge also runs in the orchestrator, which is itself one of the suspects. Still missing:

1. A continuous series of per-collection count **and** collection id, sampled through a container that is not a suspect. A changed id is what tells delete-and-recreate apart from emptied-in-place.
2. Host memory pressure beside that series, because a host-level reclaim kills a container without setting `OOMKilled`.
3. An owner that survives the orchestrator's own death and a host restart. That is likely a host-edge lane, so ADR-0014's re-review trigger fires: classify the lane against the current host-edge / container-plane vocabulary (`ai/daemons/orchestrator/taskAuthority.mjs`) before implementation.
4. A retention bound for the series, and retirement of the unversioned stopgap `~/.neo-ai/diagnostics/kb-loss-watch.sh`, which is still on the host and was last written 2026-08-05.

## Acceptance Criteria

- [ ] If the owner is a scheduler lane, an ADR-0014 amendment classifies it (kind, authority class, disable toggle), with the in-plane-observability constraint as its rationale.
- [ ] A loss produces one attributable record that joins the bridge's death record with the corpus series: per-collection count and id before and after, and host pressure.
- [ ] The owner survives a host restart and the orchestrator's own death without manual intervention.
- [ ] The series has a retention bound, and the host stopgap is retired, with its replacement named.

## Out of scope

- Diagnosing any past loss: #66.
- Health degradation on an empty KB: neomjs/neo#16512.
- Death capture for plane services: shipped by #466.

Related: #466, #66, neomjs/neo#16571 (ADR-0014 currency), neomjs/neo#16166 (the host-edge / container-plane projection).

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-08-05T23:03:20Z @neo-opus-vega added the `enhancement` label
- 2026-08-05T23:03:20Z @neo-opus-vega added the `ai` label
- 2026-08-26T15:06:51Z @tobiu added the `enhancement` label
- 2026-08-26T15:06:51Z @tobiu added the `ai` label
- 2026-11-02T18:14:10Z @neo-opus-vega cross-referenced by #15000
- 2026-11-02T18:19:26Z @neo-opus-vega cross-referenced by #1438
- 2026-11-02T18:49:30Z @neo-opus-ada changed title from **Corpus-loss cause capture must be a live event stream — the daemon retains no history to reconstruct from** to **Corpus-loss attribution: a before-and-after corpus series beside the bridge's death record**

