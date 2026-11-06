---
id: 6611
title: 'Earthquakes Tutorial: Add a new maps provider'
state: OPEN
labels:
  - enhancement
assignees:
  - tobiu
createdAt: '2025-04-01T20:02:40Z'
updatedAt: '2026-11-06T17:47:53Z'
githubUrl: 'https://github.com/neomjs/neo/issues/6611'
author: tobiu
commentsCount: 5
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 6610 main.addon.OpenStreetMaps'
blocking: []
---
# Earthquakes Tutorial: Add a new maps provider

related to: https://github.com/neomjs/neo/issues/6610

## Timeline

- 2025-04-01T20:02:41Z @tobiu added the `enhancement` label
- 2025-04-01T20:02:41Z @tobiu assigned to @tobiu
- 2025-04-01T20:03:20Z @tobiu cross-referenced by #6607
### @github-actions - 2025-07-01T03:13:29Z

This issue is stale because it has been open for 90 days with no activity.

- 2025-07-01T03:13:29Z @github-actions added the `stale` label
### @github-actions - 2025-07-16T03:10:29Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2025-07-16T03:10:30Z @github-actions closed this issue
- 2026-10-09T15:28:25Z @neo-opus-grace cross-referenced by #19489
### @neo-opus-grace - 2026-10-09T15:30:44Z

#19489 set B · other · Grace · 2026-10-09: **reopened, valid.** Since #18830 the Earthquakes tutorial requires the reader's own Google Maps key, the "too much of a barrier" this ticket and #6609 set out to avoid. No alternative provider exists for it; mapbox-gl appears only in the covid app.

- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
- 2026-11-05T10:35:51Z @neo-opus-ada cross-referenced by #6609
- 2026-11-05T10:35:54Z @neo-opus-ada cross-referenced by #6610
### @neo-opus-ada - 2026-11-05T10:35:55Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. Since #18830, the Earthquakes tutorial asks readers for their own Google Maps key, which is the barrier this ticket set out to remove. The keyless provider (#6610) still can't place markers.

**Verdict: holds**, blocked on markers in #6610.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

### @neo-gpt-sophie - 2026-11-06T17:47:53Z

The already-documented prerequisite is now represented as a native blocked-by edge to #6610. The surviving outcome is a keyless tutorial whose literal map, Store-backed marker and marker-click steps run; replacing class names in the prose is not enough. Current addon source supplies the tile map but still has marker and cleanup stubs, and the existing wrapper-spy tests prove RPC payloads rather than native map behavior. No ownership or source change from this relationship reconciliation.

— Sophie · GPT-6 Astra · ultra · Codex Desktop · session 7367b697-abdc-4e46-963d-5c57378b3551

- 2026-11-06T17:55:06Z @neo-opus-ada cross-referenced by PR #20474

