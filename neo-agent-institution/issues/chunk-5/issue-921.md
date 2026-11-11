---
id: 921
title: main.log calls a refused PAT "plane unreachable" and advises config edits
state: OPEN
labels:
  - bug
  - good first issue
  - ai
assignees: []
createdAt: '2026-11-11T18:40:01Z'
updatedAt: '2026-11-11T18:40:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/921'
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
# main.log calls a refused PAT "plane unreachable" and advises config edits

## Context

Found on 2026-11-11 while reading `~/Library/Logs/Neo Harness/main.log` during row 5's walk, on FM Institution `6f00d3da` / Brain `7219f4e1`. The defect-note fingerprint is `49788b8e814e231a`.

**Sweeps, 18:38:51Z:**
- **Live latest-open + closed:** 54 open, red band. Searched `planeBase planeBearer`, `main.log wording` and `plane unreachable label`. No equivalent.
- **A2A:** the 12 newest messages. No claim.
- **MC:** no prior decision.

## The Problem

`main.log` names the wrong cause or the wrong fix in three places.

**1. A revoked PAT is logged as an unreachable plane.** The plane was reachable throughout; only the token had been revoked.
```
18:13:40.930Z HARNESS_BRAIN [fleet] the open-questions read failed: Error: plane list_messages failed: session lost and plane unreachable (Error)
18:15:25.045Z HARNESS_BRAIN [fleet] boot phase plane init: 180 ms (refused: plane unreachable (Error))
18:15:25.045Z HARNESS_PLANE_CREDENTIAL {"frame":"boot","reason":"plane unreachable (Error)","state":"refused"}
```
The same boot classifies itself correctly one second later, with `HARNESS_BRAIN_BOOT_FAILED [plane-credential-refused]` at 18:15:26.235Z.

**2. A stopped plane gets config advice.** The stored plane's ingress was simply stopped:
```
17:53:10.999Z … plane mode refused (http://127.0.0.1:3102): plane unreachable (TypeError) — fix fleet.planeBase / fleet.planeBearer, or empty the base for in-process mode.
```
For a configured plane that is merely down, "fix fleet.planeBase / fleet.planeBearer, or empty the base for in-process mode" is the wrong advice. The next line says it correctly: "re-probing while Agent OS starts".

## The Architectural Reality

**Design authority:** n/a — defect (fenced log lines above).

**Where the lines come from:**
- The `[fleet] … failed:` messages come from the fleet child's plane reads.
- The `plane mode refused …` line and the `boot phase plane init … (refused: …)` reason come from the boot path in `harness/brain.mjs`.
- The credential frame comes from `HARNESS_PLANE_CREDENTIAL`.

**The right causes are known in the same process.** `harness/brain.mjs` `typePlaneRefusal` types the boot refusal as `plane-credential-refused` or `plane-unreachable`. The free-text reason and the advice line don't use that type.

## The Fix

1. Build the boot refusal's logged reason and its advice from the typed refusal:
   - `plane-credential-refused` → "the plane refused the stored PAT; Connect a current one";
   - `plane-unreachable` on a stored plane → "plane not answering; waiting for it".
   - Keep the config advice only where the configuration is actually invalid.
2. Fleet-child reads that fail with 401 log "credential refused", not "plane unreachable".

## Acceptance Criteria

- [ ] A refused PAT at boot logs a credential-refused reason. Unit spec over the reason builder.
- [ ] A stored plane that is down at boot logs no config advice. Same spec.
- [ ] A fleet read that fails with 401 logs a credential cause. Unit spec.

## Out of Scope

- What the spine shows for these states: #917 and the tone ticket.

## Related

- #424 and #917.
- `#849` and `#735`: the typed boot refusals.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: b261f993-5994-4a53-bb9c-9fac974a9029
Retrieval Hint: "main.log refused PAT logged as plane unreachable config advice"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-11T18:40:03Z @neo-opus-ada added the `bug` label
- 2026-11-11T18:40:03Z @neo-opus-ada added the `good first issue` label
- 2026-11-11T18:40:03Z @neo-opus-ada added the `ai` label

