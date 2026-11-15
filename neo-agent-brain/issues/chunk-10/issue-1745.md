---
id: 1745
title: A Fleet Codex seat cannot reliably work across its own checkouts
state: OPEN
labels:
  - enhancement
  - ai
  - testing
  - agent-os
assignees: []
createdAt: '2026-11-15T02:58:40Z'
updatedAt: '2026-11-15T02:58:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1745'
author: neo-gpt-emmy
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
# A Fleet Codex seat cannot reliably work across its own checkouts

## Context

Operator escalation, 2026-11-15: a Fleet-started Codex maintainer works across its own explicitly registered repositories during nightshift. In one observed session the primary checkout was writable, but tracked edits in sibling seat checkouts required escalation. Repository membership and a seat-owned checkout do not by themselves establish the harness's effective writable scope.

Two approval-review refusals also occurred; these are separate observations, not a claim that the sandbox caused either decision:

- An Institution push was rejected because the reviewer saw a conflict between issue-based branch `codex/980-native-wake-setup` and PR #981, and cited an earlier publication deferral. The live PR resolves issue #980: the two numbers name different artifacts. Whether later operator authorization reached the reviewer is unproven.
- A Brain test edit was rejected because the reviewer treated the later implementation delegation as untrusted assistant context and relied on the earlier read-only preflight. The requested implementation was explicitly directed later; this record does not assert the vendor's internal cause or guarantee control over its reviewer.

Design authority: the operator's 2026-11-15 request to investigate how a Fleet-started Codex maintainer can work in its own registered checkouts, while keeping publication and production-setting authority separate.

## The Problem

Cross-repository work should be possible within the seat's legitimately registered checkouts without granting the whole agent home, other seats, credentials or runtime configuration. Separately, a remote push needs its own authorization and network/approval decision. A sandbox denial, an approval-review refusal, and a GitHub permission are different facts; fixing one cannot be reported as fixing the others.

## The Architectural Reality

- `ai/services/fleet/deriveHarnessLaunchSpec.mjs` launches Codex Desktop with one provisioned `--open-project` cwd and isolated `CODEX_HOME` / Electron profile. This launch tuple does not itself declare sibling writable roots.
- `ai/services/fleet/prepareManagedAgentWorkspace.mjs` and the repo provisioning path already own the seat's registered repository set. #682 and #950 delivered multi-repository provisioning; neither proves effective Codex sandbox write access.
- OpenAI documents `sandbox_workspace_write.writable_roots` as additional roots: https://learn.chatgpt.com/docs/config-file/config-reference. Whether the installed Desktop and its actual Fleet launch honor a scoped provisioned value is to be tested, not assumed.

## Investigation

Establish the supported vendor and Fleet boundaries first. Test an explicit allowlist derived from canonical registered checkout paths only if the pinned harness supports it; compare that with an operator-selected workspace or another supported mechanism. Determine how additions/removals, symlinks, moved worktrees, stale sessions and process restarts affect effective scope. Return a recommendation with the owning surface and a reversible, separately authorized implementation path; this ticket does not authorize changing installed policy.

## Acceptance Criteria

- [ ] In disposable seat homes and checkouts, record the effective scope for primary plus multiple registered sibling repositories on a real Fleet-derived Codex launch. Separate a local filesystem denial from an approval-review rejection; demonstrate a benign local edit in each admitted checkout and an unregistered sibling control.
- [ ] Test containment and revocation: another seat, traversal/symlink escape, removed repository, credentials, harness/profile/runtime files and protected metadata remain outside any proposed extra checkout grant. Record whether changes require a fresh session or process restart.
- [ ] Independently test remote publication with a disposable repository/branch and explicit authorization: distinguish network/sandbox reachability, authenticated GitHub identity, approval-review decision and repository permission. Include an issue-number/PR-number mismatch control and a delegated instruction following a read-only task; capture any vendor decision as an observation, not a guarantee of control over it.
- [ ] Produce a non-secret receipt and recommendation, or explicitly record that the installed harness lacks a supported seat-scoped mechanism. Installed-seat validation is a separately authorized follow-up; no live policy changes are required to close this investigation.

## Out of Scope

Granting blanket access to a seat home, other seats, credentials or production configuration; changing approval policy, disabling auto-review, publishing or merging a PR, changing a running profile, or assuming maintainer status is authorization for production settings. Do not bypass an explicit auto-review rejection.

## Related

#571 (seat migration and applicable settings), #682 and #950 (multi-repository provisioning), #873 (MCP cwd, explicitly excludes sandbox permissions), Institution #980 / PR #981 (issue-versus-PR mapping), #1731 / PR #1737 (later delegated implementation after read-only investigation). These are context, not a claim that any already fixes this boundary. #571 already has 47 children; this standalone investigation references it rather than adding an oversized parent edge.

Decision Record impact: none proposed; this investigation selects no architecture or configuration change.

Not a good first issue: the evidence spans host sandbox, a vendor harness and operator authority and needs a controlled environment.

Live latest-open + closed sweep: latest 20 open and closed searches for Codex sandbox checkout / writable roots at 2026-11-15 02:58 UTC; 163 open (red). #571 is broader migration ownership; no equivalent precise sandbox-and-publication investigation found. A2A: latest 30 all-state messages, no overlapping claim. Memory Core symptom query returned prior observations of differing profiles, not a current decision. Structure map: `npm run ai:structure-map -- --root ai/services/fleet --files --loc` passed. Unowned rationale: ready for an interested maintainer to investigate; no implementation claimed.

Origin Session ID: 8d6c60e7-2b12-4b16-a202-5b1ad668fd75
Retrieval Hint: "Fleet Codex registered sibling checkouts workspace-write delegated approval remote push"

## Timeline

- 2026-11-15T02:58:42Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-15T02:58:42Z @neo-gpt-emmy added the `ai` label
- 2026-11-15T02:58:42Z @neo-gpt-emmy added the `testing` label
- 2026-11-15T02:58:42Z @neo-gpt-emmy added the `agent-os` label

