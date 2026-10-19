---
id: 1167
title: A Codex Desktop seat's project tool approvals refuse its Start as a launcher override
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-19T02:35:53Z'
updatedAt: '2026-10-19T03:06:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1167'
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
closedAt: '2026-10-19T03:06:59Z'
---
# A Codex Desktop seat's project tool approvals refuse its Start as a launcher override

## Context

On 2026-10-19 the first Start of `neo-gpt-euclid` on the new FM (Brain e4886d2a) refused twice: `prepareManagedAgentWorkspace: host apply effect failed · Launch profile is unverified: workspace-mismatch`. Every Codex Desktop seat starts Neural Link through Fleet's launcher since #1156 (`launchedRows` is unconditional for `codex-desktop`, `startAgentProvisioned.mjs:397`), so Sophie and Emmy are blocked the same way.

## The Problem

The launcher table moved into the seat's `CODEX_HOME` config, but the workspace's `.codex/config.toml` keeps resident-owned per-tool approvals, `[mcp_servers."neo-mjs-neural-link".tools.<tool>]` with `approval_mode`, seeded from the template. On Euclid's real file `codexProfileProjection` returns a `neo-mjs-neural-link` row holding only `tools`, and the recovery check (`launchProfileCustody.mjs` ≈:303) refuses whenever the project names a bound table at all.

Codex merges config layers recursively (`codex-rs/config/src/merge.rs`, `merge_toml_values`). A project table overrides home only through the keys it names, so approvals cannot displace the launcher.

## The Fix

The project check refuses a bound server's table only when that table names a key other than `tools`: `command`, `args`, `env`, `env_vars`, `url` and the rest would override the grant. `enabled` is already outside the projection as the seat's own switch.

## Acceptance Criteria

- [ ] A project table carrying only tool approvals for the bound server keeps custody (Euclid's shape).
- [ ] A project table naming any launcher key for the bound server still refuses `workspace-mismatch`.
- [ ] [L3-deferred — operator handoff needed] Installed: Euclid starts on the next FM candidate. Residual-Owner: neomjs/neo-agent-institution#736.

## Out of Scope

- Emmy's stuck `cleanupUnresolved` record (separate leaf).

Related: #1145, #1156, neomjs/neo-agent-institution#732.



## Timeline

- 2026-10-19T02:35:54Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-19T02:35:55Z @neo-opus-vega added the `bug` label
- 2026-10-19T02:35:55Z @neo-opus-vega added the `ai` label
- 2026-10-19T02:38:28Z @neo-opus-vega cross-referenced by PR #1168
- 2026-10-19T02:42:48Z @neo-opus-vega cross-referenced by #1169
- 2026-10-19T02:46:43Z @neo-opus-vega referenced in commit `829721a` - "fix(fleet): a Codex table's tool approvals stay outside the projection, like its switch (#1167)

Tool approvals are the seat's own settings in either layer, and Codex merges
layers key by key, so they never displace a launcher. The projection now drops
`tools` as it drops `enabled`: an approvals-only table names no launcher, and
home-layer approvals no longer break the binding or Fleet's receipt."
- 2026-10-19T02:57:23Z @neo-opus-vega cross-referenced by #1170
- 2026-10-19T03:06:59Z @tobiu referenced in commit `07fcdcc` - "fix(fleet): a Codex seat's project tool approvals no longer read as a launcher override (#1167) (#1168)

* fix(fleet): a Codex seat's project tool approvals no longer read as a launcher override (#1167)

Codex merges config layers key by key, so a project table overrides home only
through the keys it names. Recovery refused any project table that named the
bound server, including the resident's tool approvals seeded from the template,
which blocked every Codex Desktop seat's first Start on the new FM. It now
refuses only a table that names a key other than tools.

* fix(fleet): a Codex table's tool approvals stay outside the projection, like its switch (#1167)

Tool approvals are the seat's own settings in either layer, and Codex merges
layers key by key, so they never displace a launcher. The projection now drops
`tools` as it drops `enabled`: an approvals-only table names no launcher, and
home-layer approvals no longer break the binding or Fleet's receipt."
- 2026-10-19T03:06:59Z @tobiu closed this issue
- 2026-10-19T03:13:28Z @neo-fable-mnemosyne cross-referenced by PR #1171
- 2026-10-19T03:29:53Z @neo-fable-mnemosyne cross-referenced by #1173

