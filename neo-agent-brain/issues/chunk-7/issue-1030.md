---
id: 1030
title: Antigravity updates must not require a Fleet build allowlist
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-14T19:31:07Z'
updatedAt: '2026-10-14T20:02:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1030'
author: neo-gpt-emmy
commentsCount: 1
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-14T20:02:28Z'
---
# Antigravity updates must not require a Fleet build allowlist

## Context
On 2026-10-14 an installed Gemini seat was restarted while Antigravity had a downloaded update pending. The vendor update installed 2.23.1; Fleet's probe refused it with `unsupported-app-version` and the original Start was stopped before admission activation. Both the app and language-server hashes changed. Before the restart, direct read-only probes through the seat's actual MC and KB launchers completed initialize, tools/list and healthcheck successfully; native-session attachment remains a separate acceptance step.

## The Problem
`probeAntigravityRuntime.mjs` accepts only a compiled 2.23.0/version/hash tuple. Routine vendor updates therefore require a new Brain build even when the integration still works. Adding 2.23.1 to the tuple only postpones the failure.

## The Architectural Reality
Design authority: Tobi's 2026-10-14 correction: "we get almost daily claude desktop, codex, and probably antigravity updates too. locking in a version backfires." This supersedes the frozen-build policy introduced for the initial native experiment.

The existing Fleet modules already verify the selected app path/resources, Start-owned PID/birth/listener, private profile, registered project ID/name/root, exact renderer route and visible project marker. MCP launch admission independently owns identity and credentials. Those live checks remain required; a recorded version/hash is provenance, not compatibility proof. A changed bundle during one Start is distinct from a new version already installed before that Start.

## The Fix
Remove the historical version/hash allowlist from `ai/services/fleet/probeAntigravityRuntime.mjs`. Keep selected bundle identity/resource checks and measured metadata. Retain the same-Start change fence in `startAgentProvisioned.mjs`, naming it accurately. Preserve the native project binder and its refusal/Stop behavior. Update the existing unit fixtures so they no longer import an alleged supported build from production.

## Contract Ledger
| Surface | Authority | Behavior | Failure | Docs | Evidence |
|---|---|---|---|---|---|
| Runtime probe | selected app and `probeAntigravityRuntime` | Newly installed versions and hashes may proceed to live binding | Wrong app, missing resources or invalid observations refuse | Probe JSDoc | Arbitrary future version/hash positive; wrong identity/resource negatives |
| Original Start binding | `startAgentProvisioned`, `antigravityProjectBinding` | Profile, process, registry, route and visible marker remain mandatory | Specific failed proof; no admission activation | Existing JSDoc | Existing negative matrix plus bundle-change control |
| Installed seat | #571 | Updated vendor client can return through ordinary FM Start | Report actual failed capability | Installed receipt | Post-merge native project and real MC/KB calls |

Decision Record impact: none; no credential, profile, runtime-generation or authorization owner changes.

## Acceptance Criteria
- [ ] No hardcoded Antigravity version or content hash is an admission allowlist; unseen versions and hashes pass the structural probe.
- [ ] Wrong bundle identity, unsupported platform, missing resources and malformed metadata still refuse. A bundle replacement during the same Start is reported distinctly and cannot activate that Start.
- [ ] Process/profile ownership, registry/workspace matching, route/visible-marker verification and canceled/failed Start cleanup remain passing.
- [ ] Native 2.23.1 binding is exercised with the production binder; evidence states its boundary.

## Post-Merge Validation
- [ ] [L4-deferred — installed handoff] Gemini starts in the dedicated profile and invokes real MC and KB from the native session. Residual-Owner: #571.

## Out of Scope
A vendor-version registry, automatic downgrade/update management, credential changes, global Antigravity config edits, other harness rewrites, runtime-generation work (#980), or diagnosis of the earlier MC/KB attachment loss.

## Avoided Traps
Replacing one pinned build with two; treating hashes as behavioral compatibility; removing live ownership guards; counting a dummy MCP fixture as real MC/KB service acceptance.

Related: #571; #1017; #980; neomjs/neo-agent-institution#12.

Live latest-open sweep: latest 20 open issues at 2026-10-14T19:30Z, no equivalent. A2A latest 30 all-state messages: no competing claim. MC problem-noun sweep: 3 unrelated results, no prior decision recovered; direct #1017 body read supplies the initial-boot boundary. KB ticket sweep found no equivalent. Own-assignment sweep: 6 open; #936 and #904 bodies read, distinct wake and HTTP-error work. Structure map: existing `ai/services/fleet` probe/binder/Start modules own this repair; no new module.

Origin Session ID: 05b71d01-be36-41f5-8d86-6c898089c6ec
Retrieval Hint: Antigravity 2.23.1 pending update unsupported-app-version frozen fingerprint.
Authored by Emmy (GPT-6, Codex Desktop).

## Timeline

- 2026-10-14T19:31:07Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-14T19:31:08Z @neo-gpt-emmy added the `bug` label
- 2026-10-14T19:31:09Z @neo-gpt-emmy added the `ai` label
- 2026-10-14T19:31:09Z @neo-gpt-emmy added the `agent-os` label
### @neo-gpt-emmy - 2026-10-14T19:31:43Z

**`[lane-override]` reassignment audit-trail** (#11537 §AC8)

**Previous assignees:** `@neo-gpt-emmy`
**New assignees:** `@me`
**Reason:** This same-session ticket was created already assigned solely to neo-gpt-emmy; confirm that same owner before implementation. No ownership transfer.

*Audit-trail per AGENTS.md §6.5 — `acknowledgedReassign` reason persistence. Graph-ingested via Retrospective daemon comment-scan path.*

- 2026-10-14T19:31:46Z @neo-gpt-emmy added parent issue #571
- 2026-10-14T19:40:14Z @neo-gpt-emmy referenced in commit `1dffa89` - "test(fleet): retain downstream proof failures for updated clients (#1030)"
- 2026-10-14T19:40:31Z @neo-gpt-emmy cross-referenced by PR #1031
- 2026-10-14T19:42:02Z @neo-gpt-emmy cross-referenced by #571
- 2026-10-14T20:02:28Z @tobiu referenced in commit `75581e4` - "fix(fleet): admit Antigravity updates through live checks (#1030) (#1031)

* fix(fleet): admit Antigravity updates through live checks (#1030)

* test(fleet): retain downstream proof failures for updated clients (#1030)"
- 2026-10-14T20:02:28Z @tobiu closed this issue

