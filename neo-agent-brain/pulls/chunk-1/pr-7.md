---
number: 7
title: 'Revert "Merge pull request #5 from neomjs/feat/substrate-sync-17784"'
author: neo-opus-grace
state: MERGED
createdAt: '2026-08-26T08:23:05Z'
updatedAt: '2026-08-26T08:34:03Z'
closedAt: '2026-08-26T08:33:59Z'
mergedAt: '2026-08-26T08:33:59Z'
head: revert/pr-5-substrate-sync
base: dev
url: 'https://github.com/neomjs/neo-agent-brain/pull/7'
contentTrust:
  projected: true
  quarantined: 0
  signals: []
---
Reverts #5 — a true `git revert -m 1 aa77b99`, so the history records the undo rather than a hand-built deletion.

This repo is the AgentOS extraction **destination**, not a stale consumer. It was created 2026-08-23 as a scaffold (neomjs/neo#17640) awaiting the cut, so its lack of `.agents/skills` was content it had not received yet — not the invisible staleness the canonical store exists to close. I enrolled it on that misreading.

Leaving canonical-synced, receipt-pinned, CI-guarded bytes on `.agents/**` here while neomjs/neo#17786 and #17788 prepare to populate this repo puts two mechanisms on one path during the cut window.

Restores the repo to its scaffold state: `.gitignore`, `LICENSE`, `README.md`, `package.json`. Removes `.agents/skills` (38), the `.claude/skills` façade, `AGENT_SUBSTRATE_REVISION.json`, and `.github/workflows/substrate-sync.yml`. 172 files, 9,447 deletions. Nothing else touched.

Canonical `enrollment.json` already carries this repo as an explicit **excluded** row with the reason, so it reports as a deliberate exclusion rather than an unlisted gap.

Supersedes #6, which did the same removal by hand; closing that in favour of this.

Authored by Grace (Claude Opus 5, Claude Code). Session f27af939-3cec-4f52-a67d-e4e8786fed08.
