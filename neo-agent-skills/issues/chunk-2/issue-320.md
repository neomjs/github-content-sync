---
id: 320
title: 'The guide lint runs in the shared PR baseline, not from a Brain checkout'
state: CLOSED
labels:
  - enhancement
  - ai
  - build
  - needs-design-authority
assignees:
  - neo-opus-grace
createdAt: '2026-11-08T18:42:31Z'
updatedAt: '2026-11-08T20:30:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/320'
author: neo-opus-grace
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
blocking:
  - '[ ] 1518 The Brain''s PR baseline still pins v0.1.84, and every check name repeats its caller'
closedAt: '2026-11-08T20:30:45Z'
---
# The guide lint runs in the shared PR baseline, not from a Brain checkout

## Context

The Engine's `lint-guides.yml` checks out `neo-agent-brain` at `dev` to run the Brain's `ai/scripts/lint/lint-guides.mjs --root engine` (neomjs/neo#20177, with `--root` from neomjs/neo-agent-brain#1366). That makes an Engine pull request's verdict depend on the Brain's `dev`. It also breaks neomjs/neo#20328's predicate that no Engine CI job checks out the Brain. The fix is tracked in neomjs/neo#20597. The shape, agreed with @neo-opus-ada:
- the lint ships here;
- the Engine runs it at its pinned release;
- the Brain runs the Brain-data rules over the Engine's guides at its own `neo.mjs` pin.

## The Problem

The lint has one home, the Brain. Any other repository has to check the Brain out to run it, and a consumer pinned to a skills release cannot run it at all.

A `guards` entry cannot carry it either. That job runs the consumer's own install, and the Engine pins `neo-agent-skills` 0.1.90, which has no such bin. The `PR baseline` job already installs this package's guards at the workflow's own tag (`Install immutable guards`). A step there runs exactly the pinned release, with no consumer install and no second version pin.

## The Architectural Reality

- **The lint:** Brain `ai/scripts/lint/lint-guides.mjs` (Brain `dev`, 651 lines) imports only Node built-ins and the Brain's `isEntryModule`.
- **HARD rules:** `mermaid-reserved-word`, `mermaid-self-loop`, `dead-link`, `dead-script-ref`, `openapi-tool-parity`, `ticket-id`.
- **WARN rules:** `no-mermaid`, `mermaid-lr-squish`, `feature-list-heading`, `identity-framework`.
- **Scope:** every rule runs over the top-level `learn/agentos` and `learn/benefits` guides; `ticket-id` also runs over nested `learn/guides`.
- **Data inputs:**
  - `dead-script-ref` reads the scripts in the root's `package.json` and `cloud/package.json`;
  - `openapi-tool-parity` reads `ai/mcp/server/*/openapi.yaml`, and with no such files it is a no-op.
- **The `PR baseline` job** (`reusable-pr-baseline.yml`) runs no pull-request code. Its guards come from `runner.temp` at `package.json`'s version. Every step runs under `!cancelled()`, and inputs arrive through `env:`, validated fail-closed.
- **The coverage:** Brain `test/playwright/unit/ai/scripts/lint/lintGuides.spec.mjs` (338 lines) carries the rule coverage.

## The Fix

1. **`scripts/check-guides.mjs`, bin `neo-agent-skills-guides`:** the lint, moved here. The Brain copy retires under its own Brain ticket. The root defaults to the working directory. New flags:
   - `--script-root <dir>`, repeatable: the manifests that resolve `ai:*` references, defaulting to the root;
   - `--openapi-root <dir>`, defaulting to the root;
   - `--skip <rule>`, repeatable, accepting known rule ids only.
2. **`reusable-pr-baseline.yml`:** a `Guides` step in `PR baseline`, off by default. Two inputs control it: `guides` (boolean, default `false`) and `guides_skip` (a JSON array of rule ids, default `'[]'`).
3. **Tests:**
   - `scripts/test-check-guides.mjs`, carrying the Brain spec's arms plus the new flags;
   - `test-reusable-pr-baseline.mjs` covers the step: off by default, and its input validated.
4. **Release** 1.2.0, a minor: a new bin and new inputs.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| bin `neo-agent-skills-guides` | this package | Lints the `learn/` guides under `--root` (default: the working directory). Exits 1 on a HARD finding. | none (new) | README bin list | Brain lint at `dev` |
| input `guides` | `reusable-pr-baseline.yml` | Runs the `Guides` step in `PR baseline` over the caller's checkout | `false`: no step runs | workflow header | none yet (new) |
| input `guides_skip` | `reusable-pr-baseline.yml` | Rule ids to skip. An unknown id or malformed JSON fails the step. | `'[]'` | workflow header | none yet (new) |

## Acceptance Criteria

- [ ] AC-1: On one tree (the Engine at `dev`), `neo-agent-skills-guides --root <dir>` reports the same HARD and WARN findings as the Brain's lint.
- [ ] AC-2: `--skip dead-script-ref` drops that rule and no other. An unknown rule id exits non-zero.
- [ ] AC-3: With `guides: true`, `PR baseline` fails on a HARD finding in the caller's tree. With the default, no `Guides` step runs.
- [ ] AC-4 (post-merge): 1.2.0 is published, and its manifest lists the bin.

## Out of Scope

- The Brain switching to the bin, and its check of the Engine's guides: a Brain ticket.
- The Engine adopting it: neomjs/neo#20597.
- What the rules check.

## Avoided Traps

- **A `guards` entry.** The consumer's install would have to carry the bin, which forces a devDependency bump or a second version pin.
- **Default on.** Every consumer's baseline would change at upgrade, including repositories that never asked for the step.

## Related

neomjs/neo#20597 · neomjs/neo#20328 · neomjs/neo#20177 · neomjs/neo-agent-brain#1366 · #319 (the `guards` job) · #14

Live latest-open + closed sweep: latest 20 open, plus a closed search for "guide lint" and a PR search for "guide". Read the same-surface hits #80 and #14: neither covers this. A2A claims and Memory Core also checked, 2026-11-08T18:41Z. No equivalent found.

Origin Session ID: 3041b574-18af-44d7-840f-2baf12f52c3a
Retrieval Hint: "guide lint shared PR baseline step neo-agent-skills-guides lint-guides Brain checkout Engine 20597"

## Timeline

- 2026-11-08T18:42:31Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-08T18:42:32Z @neo-opus-grace added the `enhancement` label
- 2026-11-08T18:42:33Z @neo-opus-grace added the `ai` label
- 2026-11-08T18:42:33Z @neo-opus-grace added the `build` label
- 2026-11-08T18:42:43Z @github-actions added the `needs-design-authority` label
- 2026-11-08T18:42:51Z @neo-opus-grace cross-referenced by #20597
- 2026-11-08T19:06:01Z @neo-opus-grace cross-referenced by PR #321
- 2026-11-08T19:29:00Z @neo-opus-grace referenced in commit `e54f994` - "fix(guides): every rule reads a guide in one linear pass, so a crafted guide cannot stall the baseline (#320)"
- 2026-11-08T20:00:01Z @neo-opus-grace marked this issue as blocking #1518
- 2026-11-08T20:18:46Z @neo-opus-grace referenced in commit `1a79ec6` - "chore(package): npm test keeps a space before the release-tag contract (#320)"
- 2026-11-08T20:30:45Z @tobiu referenced in commit `fa06fca` - "feat(baseline): the guide lint runs in the shared PR baseline, not from a Brain checkout (#320) (#321)

* feat(baseline): the guide lint runs in the shared PR baseline, not from a Brain checkout (#320)

* fix(guides): every rule reads a guide in one linear pass, so a crafted guide cannot stall the baseline (#320)

* chore(package): npm test keeps a space before the release-tag contract (#320)"
- 2026-11-08T20:30:46Z @tobiu closed this issue

