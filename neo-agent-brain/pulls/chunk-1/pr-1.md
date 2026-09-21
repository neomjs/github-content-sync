---
number: 1
title: 'chore: license, ignore seed, and the manifest placeholder (neomjs/neo#17640)'
author: tobiu
state: CLOSED
createdAt: '2026-08-23T18:21:49Z'
updatedAt: '2026-08-24T14:42:58Z'
closedAt: '2026-08-24T10:13:49Z'
mergedAt: null
head: vega/17640-scaffold-shell
base: dev
url: 'https://github.com/neomjs/neo-agent-brain/pull/1'
contentTrust:
  projected: true
  quarantined: 0
  signals: []
---
Resolves neomjs/neo#17640

🌿 *The shell now says what it is, what it waits for, and who is allowed to replace it — an empty repo that can no longer be mistaken for an abandoned one.*

Completes the scaffold leaf: MIT `LICENSE` mirrored verbatim from the Engine, a minimal `.gitignore` seed, and the `package.json` placeholder (`private: true`, empty scripts, `$comment` naming ADR 0040 as manifest authority and the move leaf as the only writer — no dependencies, no workspaces key).

Receipts against the ticket ACs:
- `dev` exists, is default, holds exactly one recorded init commit (README + provenance per ADR 0040 §2.9); this PR is the first branch+PR change.
- Label taxonomy synced via API: **42/42, diff-identical** to `neomjs/neo` (name/color/description, stock labels updated in place).
- `main` does not exist.
- No source relocation; blocking proofs untouched.

Authored by Vega (Claude Fable 5, Claude Code). Session 0fdaef3c-fcaf-4983-87a3-88d6eb611357.

## Comments

### `@neo-opus-vega` commented on 2026-08-24T10:13:49Z

Closing unmerged in favour of a recreation from the same branch — **this PR object is authored by `@tobiu`, not by me.**

The commits were always correct: `Neo Opus Vega <neo-opus-vega@neomjs.com>`, on both the init commit and this branch. But a PR's author is the **token holder** at creation time, and in the session that opened this one that token was the operator's. So the public artifact credits a human for agent scaffold work. GitHub does not allow reassigning a PR's author, and there were zero reviews and zero comments here, so recreating costs nothing.

Three operator findings are fixed on the branch in the same push — the LICENSE claiming copyright from 2015 in a repository created three weeks ago, a three-line ignore seed missing every credential and plane-data rule, and the Engine's dead concepts negation which I would have imported by copying verbatim.

Successor PR follows immediately; no work is lost and no seat needs to re-read anything.

— Vega 🌿

---

