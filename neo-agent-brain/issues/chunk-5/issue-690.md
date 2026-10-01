---
id: 690
title: 'The harness catalog groups harness types by product, app or command line'
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-01T14:42:52Z'
updatedAt: '2026-10-01T15:12:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/690'
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
closedAt: '2026-10-01T15:12:15Z'
---
# The harness catalog groups harness types by product, app or command line

## Context

The operator, 2026-09-28, on the installed add-agent form (relayed on neomjs/neo-agent-institution#245, comment 5867878633): the screenshot *"specifically rejects Codex and Codex Desktop as separate product choices … Present the meaningful product choice and preserve required launch mechanics behind it."* The form cannot do that today, because the catalog it renders has no notion of a product.

## The Problem

`src/fleet/contract/harnessTypes.mjs` is a flat list: `codex` "Codex", `codex-desktop` "Codex Desktop", `claude-code` "Claude Code", `claude-desktop` "Claude", `opencode`, `kimi-code`, `antigravity`, `native-neo`. Both Institution add-agent forms (`AddAgentForm` chips, the Accounts panel radios) render one choice per entry. That puts launch mechanics (an app or a command line) at the same level as the product a person is choosing.

The app/command-line split already exists in the Brain, but privately: `FleetLifecycleService` keeps its own `SURVIVING_HARNESS_TYPES = new Set(['antigravity', 'claude-desktop', 'codex-desktop'])` for seats that outlive a Fleet Manager quit (#660). A consumer that needs the same split has to keep a third copy of it.

## The Architectural Reality

- The catalog is the Brain's public contract module: no Fleet services, frozen entries, caller-owned copies (`listHarnessTypes`, `resolveHarnessType`, `resolveHarnessFamily`).
- Its consumers include the Brain's registry, launch spec, workspace plan and identity display, plus the Institution's `AddAgentForm`, Accounts `Panel`, `AgentConfigComponent` and `harness/contentPolicy.mjs`, through the pinned package.
- `native-neo` is registered but cannot be launched (`deriveHarnessLaunchSpec`).

## The Fix

1. Each catalog entry gains two keys:
   - `product`: `codex` for `codex`/`codex-desktop`, `claude` for `claude-code`/`claude-desktop`, and the type's own key otherwise.
   - `runsAs`: `'app'` for `codex-desktop`, `claude-desktop` and `antigravity`; `'cli'` for `codex`, `claude-code`, `opencode` and `kimi-code`; `null` for `native-neo`.
2. A new `listHarnessProducts()` returns `[{product, label, types}]` in catalog order, each product's label once, its types as caller-owned copies.
3. `FleetLifecycleService` derives `SURVIVING_HARNESS_TYPES` from `runsAs === 'app'` instead of listing the three types again.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `HARNESS_TYPES[*].product`, `.runsAs` (new keys) | this ticket; the operator's 09-28 ruling | the product a type belongs to, and whether it runs as an app or a command line | `runsAs: null` for an unlaunchable type | module JSDoc | AC-1 |
| `listHarnessProducts()` (new export) | this ticket | products in catalog order with their types | a single-type product lists one type | JSDoc | AC-2 |
| `FleetLifecycleService` surviving set (existing) | #660 | derived from `runsAs === 'app'`, same three types | — | JSDoc | AC-3 |

## Decision Record impact

`none`.

## Acceptance Criteria

- [ ] AC-1 (unit): every catalog entry carries `product` and `runsAs` as listed, and the existing keys and their order are unchanged.
- [ ] AC-2 (unit): `listHarnessProducts()` returns `codex` (Codex: `codex`, `codex-desktop`), `claude` (Claude: `claude-code`, `claude-desktop`), then the single-type products, in catalog order, as caller-owned copies.
- [ ] AC-3 (unit): `FleetLifecycleService`'s surviving set derives from the catalog and still holds exactly `antigravity`, `claude-desktop` and `codex-desktop`.

## Out of Scope

- The Institution form that renders one choice per product, with the app or command-line choice behind it: neomjs/neo-agent-institution#245, after a Brain pin that carries this.
- Renaming any harness key or label.

## Related

neomjs/neo-agent-institution#245 (the consumer) · #660 (the surviving set) · #656 (`modelFamily`, the last key the catalog gained)

Live latest-open sweep: the latest open Brain issues at 2026-10-01T14:45Z (#687 down), no equivalent; org search "harness product" finds neo#15520 (naming the Fleet Manager product itself, a different question). A2A in-flight sweep: no claim on the catalog. Memory Core sweep: the 09-28 ruling as relayed; no decision against grouping. Own-assignment sweep: Institution #245 (the consumer). Structure map: owning file `src/fleet/contract/harnessTypes.mjs`, no new file.

Origin Session ID: 84a3bf84-c9cb-4215-818a-d9640f49669a
Retrieval Hint: `query_raw_memories("harness catalog product grouping runsAs app cli Codex Desktop separate product choice")`

## Timeline

- 2026-10-01T14:42:53Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-01T14:42:54Z @neo-opus-ada added the `enhancement` label
- 2026-10-01T14:42:55Z @neo-opus-ada added the `ai` label
- 2026-10-01T14:42:55Z @neo-opus-ada added the `agent-os` label
- 2026-10-01T14:45:19Z @neo-opus-ada cross-referenced by PR #691
- 2026-10-01T14:56:35Z @neo-fable-clio cross-referenced by #694
- 2026-10-01T15:12:15Z @tobiu referenced in commit `9f72f91` - "feat(fleet): the harness catalog groups harness types by product, app or command line (#690) (#691)

Each catalog entry names its product (codex, claude, opencode, kimi-code,
antigravity, native-neo) and how it runs (runsAs: 'app' | 'cli' | null), and
listHarnessProducts() lists one choice per product with its types. The add-agent
form can then show "Codex" once with the app or the command line behind it, as
the operator ruled on 2026-09-28. FleetLifecycleService's surviving set now
derives from runsAs === 'app' instead of repeating the three app types."
- 2026-10-01T15:12:16Z @tobiu closed this issue
- 2026-10-01T15:20:12Z @neo-fable-clio cross-referenced by PR #695

