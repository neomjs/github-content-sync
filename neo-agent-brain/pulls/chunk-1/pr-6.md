---
number: 6
title: Unenroll from canonical skill sync until after the cut
author: neo-opus-grace
state: CLOSED
createdAt: '2026-08-26T08:20:10Z'
updatedAt: '2026-08-26T08:23:07Z'
closedAt: '2026-08-26T08:23:07Z'
mergedAt: null
head: revert/substrate-sync-premature
base: dev
url: 'https://github.com/neomjs/neo-agent-brain/pull/6'
contentTrust:
  projected: true
  quarantined: 0
  signals: []
---
Reverts #5.

**This repo is the AgentOS extraction destination, not a stale consumer.** It was created 2026-08-23 as a scaffold (neomjs/neo#17640) awaiting the cut. Its lack of `.agents/skills` was content it had not received yet — not the invisible staleness the canonical store exists to close. I enrolled it on that misreading; @tobiu caught it after merge.

**Why it matters now rather than later:** neomjs/neo#17786 and #17788 are preparing to populate this repo. Leaving canonical-synced, receipt-pinned, CI-guarded bytes on `.agents/**` here means two mechanisms claim the same paths during the cut window — the one window where that is most expensive.

**The deeper miss:** ADR 0040 never dispositions `.agents/skills` at all. My §2.7 amendment asserted that skill-tree custody "is decided in ADR 0041" — answering a custody question that belongs to the cut, without routing it to @neo-opus-vega or @neo-gpt. That question is now open rather than silently settled.

Canonical's `enrollment.json` already carries this repo as an **explicit excluded row with a reason** (AC-6's requirement), so it reports as a deliberate exclusion rather than an unlisted gap. Re-evaluate enrollment once the cut lands.

Removes: `.agents/skills` (38), `.claude/skills` façade, `AGENT_SUBSTRATE_REVISION.json`, `.github/workflows/substrate-sync.yml`. Nothing else in the repo is touched.

Authored by Grace (Claude Opus 5, Claude Code). Session f27af939-3cec-4f52-a67d-e4e8786fed08.

## Comments

### `@neo-opus-grace` commented on 2026-08-26T08:23:06Z

Superseded by the true `git revert -m 1` of the merge commit — same removal, but the history records it as a revert of #5 rather than a hand-built deletion.

---

