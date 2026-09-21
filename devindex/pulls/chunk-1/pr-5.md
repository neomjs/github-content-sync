---
number: 5
title: Enroll in the canonical agent-skill substrate
author: tobiu
state: CLOSED
createdAt: '2026-08-25T22:57:39Z'
updatedAt: '2026-08-25T23:09:47Z'
closedAt: '2026-08-25T23:09:47Z'
mergedAt: null
head: feat/substrate-sync-17784
base: main
url: 'https://github.com/neomjs/devindex/pull/5'
contentTrust:
  projected: true
  quarantined: 0
  signals: []
---
Refs neomjs/neo#17784

Enrolls this repository in the canonical agent-skill substrate.

Until now this repo was not *behind* canonical — it had no `.agents/skills` at all, and nothing reported that. Invisible absence is the failure mode the canonical store exists to close: a repo that never synced looks identical to a repo that needs nothing.

## What lands

| path | what |
|---|---|
| `.agents/skills/` | the canonical tree — 38 skills + manifest + schema, byte-identical to canonical |
| `AGENT_SUBSTRATE_REVISION.json` | receipt pinning `canonical@1b7cecd`, tree hash `13d8e935…` |
| `.claude/skills/` | the manifest-derived façade — 37 links; the one declared opt-out is correctly absent |
| `.github/workflows/substrate-sync.yml` | calls the reusable guard so future divergence is reported |

## Verification

The guard was run against this branch before it was pushed:

```
leg A — skill tree matches canonical@receipt (13d8e935…)
leg B — façade matches the manifest projection (37 projected, 1 declared opt-out correctly absent)
substrate-sync: GREEN
```

The same guard measured this repo **RED** before the sync, failing on the absent receipt and an underivable projection. Green here is the same instrument reporting the state changed — not a different instrument reporting a different opinion.

## Not in scope

`AGENTS.md` is untouched. The committed constitution is the **contributor** surface and keeps separate custody from the maintainer constitution, which projects into seat substrate rather than into a repo's tree. Its divergence from canonical is a different ticket, deliberately not folded in here.

Authored by Grace (Claude Opus 5, Claude Code). Session f27af939-3cec-4f52-a67d-e4e8786fed08.


## Comments

### `@neo-opus-grace` commented on 2026-08-25T23:09:46Z

Reopening under my own identity — this was created with the wrong GitHub account; same branch, same commits (authored `Grace <neo-claude-opus@neomjs.com>`), no content change.

---

