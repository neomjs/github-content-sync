---
id: 1408
title: 'security: resolve 12 CodeQL code-scanning alerts across workflows, FleetManager, deployment prescriptions, and specs'
state: CLOSED
labels:
  - bug
  - ai
  - security
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-31T21:16:07Z'
updatedAt: '2026-10-31T21:51:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1408'
author: neo-gemini-julia
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
closedAt: '2026-10-31T21:51:03Z'
---
# security: resolve 12 CodeQL code-scanning alerts across workflows, FleetManager, deployment prescriptions, and specs

## Context

GitHub CodeQL code-scanning on [`neomjs/neo-agent-brain`](https://github.com/neomjs/neo-agent-brain/security/code-scanning) currently reports **12 open alerts** (`#1`–`#10`, `#12`–`#13`) across our own workflows, production modules, and unit specs. Seven of `.github/workflows/*.yml` already declare explicit least-privilege `permissions:`, while the remaining seven lint/SLA workflows omit `permissions:` (`actions/missing-workflow-permissions`), and four JavaScript/TypeScript sites trigger CodeQL high/medium dataflow or sanitization rules (`js/incomplete-sanitization`, `js/clear-text-logging`, `js/bad-code-sanitization`, `js/incomplete-url-substring-sanitization`).

## The Problem

The 12 open CodeQL alerts break down into five concrete findings:

| Alert(s) | Rule & Severity | Target Path & Line | Finding |
|---|---|---|---|
| `#1`–`#7` | `actions/missing-workflow-permissions` (`medium`) | `.github/workflows/check-retired-primitives.yml:23`, `config-template-ssot-lint.yml:77`, `detection-retention-sla.yml:41`, `identity-vocabulary-lint.yml:35`, `mcp-test-location-lint.yml:29`, `openapi-service-parity-lint.yml:60`, `script-plane-lint.yml:41` | Each read-only lint/SLA workflow omits an explicit `permissions: contents: read` block, leaving `GITHUB_TOKEN` unscoped at the workflow level. |
| `#8` | `js/incomplete-sanitization` (`high`) | `ai/services/fleet/FleetManager.mjs:114` | `repoSlug.replace(/\./g, '\\.')` escapes only `.` before interpolating into `new RegExp(...)`, without escaping `\\` or other regex metacharacters. |
| `#9` | `js/bad-code-sanitization` (`medium`) | `test/playwright/unit/ai/services/fleet/installAgentRepoDependencies.spec.mjs:168` | Interpolates `${JSON.stringify(pidFile)}` and `${JSON.stringify(heartbeat)}` inside a string passed to `spawn(process.execPath, ['-e', ...])`. |
| `#10` | `js/clear-text-logging` (`high`) | `ai/scripts/maintenance/materializeDeploymentPrescriptions.mjs:877` | `main()` in `materialize` mode calls `console.log(JSON.stringify(result))`, logging `result.statePath` which flows from `DEFAULT_ROOT = process.env.NEO_HOST_DEPLOYMENT_PRESCRIPTION_ROOT` (`codeFlows`: `:46` → `:812` → `:868` → `:563` → `:670` → `:877`), whereas `append` (`:849`) and `receipt` (`:894`) log explicit bounded summary fields without host paths. |
| `#12`, `#13` | `js/incomplete-url-substring-sanitization` (`high`) | `test/playwright/unit/ai/services/github-workflow/GitHubCommunityContentService.spec.mjs:57-58` | Uses `.includes('https://hostile.example')` on `result.content.body` and `result.content.title`, which CodeQL flags as an unanchored URL host substring check. |

## The Architectural Reality

- All seven sibling workflows in `.github/workflows/` (`brain-unit.yml`, `brain-integration.yml`, `shared-pr-baseline.yml`, `dependabot-automerge.yml`, `substrate-sync.yml`, `publish-mcp-registry.yml`, `claim-responder.yml`) already declare explicit top-level or job-level `permissions:`.
- In `ai/services/fleet/FleetManager.mjs` (`repoCoordinates`), `assertRepoSlug` validates the slug before line 114, and escaping the full regex metacharacter class (`repoSlug.replace(/[\\^$.*+?()[\]{}|]/g, '\\$&')`) satisfies `js/incomplete-sanitization` while hardening regex construction if `assertRepoSlug` ever widens.
- In `ai/scripts/maintenance/materializeDeploymentPrescriptions.mjs`, `materializeDeploymentPrescriptions()` returns `{status, activeCount, benignHistoryCount, deploymentRunId, materializedDigest, statePath}` to programmatic callers, while the CLI `main()` only needs to emit `{status, activeCount, benignHistoryCount, deploymentRunId, materializedDigest}` on stdout (matching `append` and `receipt` modes, and matching `deploy-pipeline.sh` which ignores stdout).
- In `installAgentRepoDependencies.spec.mjs`, `runToExit` already executes with `{cwd: root}`, so static `-e` scripts reading `'pid'` and `'heartbeat'` in `cwd` (with the child code passed via `process.argv[1]`) eliminate dynamic `-e` code interpolation altogether.
- In `GitHubCommunityContentService.spec.mjs`, asserting exact equality (`expect(result.content.body).toBe(trusted ? externalBody : expectedRedacted)`) is both stronger than `.includes('https://hostile.example')` and free of `js/incomplete-url-substring-sanitization`.

## The Fix

1. Add explicit top-level `permissions:\n  contents: read` to the 7 read-only GitHub Actions workflows (`#1`–`#7`).
2. Escape all regex metacharacters (`repoSlug.replace(/[\\^$.*+?()[\]{}|]/g, '\\$&')`) in `ai/services/fleet/FleetManager.mjs:114` (`#8`).
3. Replace dynamic `JSON.stringify` `-e` code construction in `test/playwright/unit/ai/services/fleet/installAgentRepoDependencies.spec.mjs` with static `-e` script strings using `cwd`-relative `'pid'` / `'heartbeat'` and `process.argv[1]` (`#9`).
4. Log the explicit bounded summary object `{status, activeCount, benignHistoryCount, deploymentRunId, materializedDigest}` in `materializeDeploymentPrescriptions.mjs` `main()` `materialize` mode (`#10`), keeping `materializeDeploymentPrescriptions()`'s programmatic return shape (`statePath`) intact.
5. Replace `.includes('https://hostile.example')` in `test/playwright/unit/ai/services/github-workflow/GitHubCommunityContentService.spec.mjs:57-58` with exact string equality against `externalBody` vs. `'Please see [QUARANTINED_URL: hostile.example].'` (`#12`, `#13`).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Refusal | Docs | Evidence |
|---|---|---|---|---|---|
| `.github/workflows/{check-retired-primitives,config-template-ssot-lint,detection-retention-sla,identity-vocabulary-lint,mcp-test-location-lint,openapi-service-parity-lint,script-plane-lint}.yml` | CodeQL `actions/missing-workflow-permissions` (`#1`–`#7`) | Declare top-level `permissions: contents: read` on all 7 read-only lint/SLA workflows | Any workflow needing write access declares it explicitly (`dependabot-automerge.yml`, `claim-responder.yml`) | Inline workflow YAML | `OpenApiServiceParityTriggerReachability.spec.mjs` + CI CodeQL Actions scan |
| `ai/services/fleet/FleetManager.mjs` (`repoCoordinates`) | CodeQL `js/incomplete-sanitization` (`#8`) | Escape all regex metacharacters (`/[\\^$.*+?()[\]{}|]/g`) before interpolating `repoSlug` into `new RegExp(...)` | Invalid slugs continue to throw in `assertRepoSlug` / `remote.test` | Existing JSDoc preserved | `FleetManager.spec.mjs` |
| `ai/scripts/maintenance/materializeDeploymentPrescriptions.mjs` (`main`) | CodeQL `js/clear-text-logging` (`#10`) | CLI `materialize` mode logs bounded summary `{status, activeCount, benignHistoryCount, deploymentRunId, materializedDigest}` without `statePath` | Programmatic `materializeDeploymentPrescriptions()` still returns `statePath` | Existing JSDoc preserved | `materializeDeploymentPrescriptions.spec.mjs` |
| `test/playwright/unit/ai/services/fleet/installAgentRepoDependencies.spec.mjs` & `GitHubCommunityContentService.spec.mjs` | CodeQL `js/bad-code-sanitization` (`#9`) & `js/incomplete-url-substring-sanitization` (`#12`, `#13`) | Static `-e` child script strings (`argv[1]`) and exact `toBe(trusted ? externalBody : expectedRedacted)` assertions | Test failure on process leak or redaction mismatch | Spec descriptions preserved | Both unit specs green |

Decision Record impact: none (`AiConfig` is untouched; ADR-0019 is not engaged).

## Acceptance Criteria

- [ ] AC-1: All 7 flagged GitHub Actions workflows declare `permissions: contents: read`, and workflow reachability/lint specs pass.
- [ ] AC-2: `FleetManager.mjs:114` escapes backslashes and regex metacharacters (`/[\\^$.*+?()[\]{}|]/g`), and `materializeDeploymentPrescriptions.mjs:877` logs the bounded CLI summary without `statePath` taint from `process.env`.
- [ ] AC-3: `installAgentRepoDependencies.spec.mjs` and `GitHubCommunityContentService.spec.mjs` eliminate `js/bad-code-sanitization` and `js/incomplete-url-substring-sanitization` while passing all unit assertions.
- [ ] AC-4: All affected unit specs (`FleetManager.spec.mjs`, `materializeDeploymentPrescriptions.spec.mjs`, `installAgentRepoDependencies.spec.mjs`, `GitHubCommunityContentService.spec.mjs`, `OpenApiServiceParityTriggerReachability.spec.mjs`) pass, and CodeQL PR scanning reports 0 new or remaining open alerts on the touched surfaces.

## Out of Scope

- Dismissed Alert `#11` (`ai/services/graph/providerReadinessHelper.mjs:735`, already dismissed).
- Changing programmatic return fields of `materializeDeploymentPrescriptions()`.

## Related

- Code scanning dashboard: `https://github.com/neomjs/neo-agent-brain/security/code-scanning` (`#1`–`#10`, `#12`–`#13`)

---
*Sweep Attestations (`ticket-create §1a`):*
- *Live latest-open sweep:* Checked latest 20 open issues in `neomjs/neo-agent-brain` (`#1407`, `#1403`, `#1401`, `#1383`, etc.) plus `gh issue list --repo neomjs/neo-agent-brain --state all --search "code-scanning OR CodeQL"` at `2026-10-31T21:15Z`; no existing ticket tracks these 12 CodeQL alerts.
- *A2A in-flight claim sweep:* Checked `list_messages({status: 'unread'})` and recent `AGENT:*` broadcasts at `2026-10-31T21:15Z`; no claim on `code-scanning` or CodeQL alerts.
- *Memory Core rationale sweep:* Checked `query_raw_memories(query: "neo-agent-brain code-scanning CodeQL alerts FleetManager materializeDeploymentPrescriptions missing-workflow-permissions")` at `2026-10-31T21:15Z`; no prior decision against resolving these 12 alerts.
- *Own-assignment sweep:* Checked `gh issue list --repo neomjs/neo-agent-brain --state open --assignee @me` (`#1248` offline migration only, zero overlap).
- *Structure map:* Ran `npm run ai:structure-map -- --files --loc` at `2026-10-31T21:15Z`; edits are in-place on existing files (`ai/services/fleet/FleetManager.mjs`, `ai/scripts/maintenance/materializeDeploymentPrescriptions.mjs`, 2 unit specs, and 7 `.github/workflows/*.yml` files) with no new `.mjs` files.

Origin Session ID: 7d7b8e30-8cfd-4ac9-91fc-3fbeb3e49a99
Retrieval Hint: "code-scanning CodeQL 12 alerts missing-workflow-permissions FleetManager incomplete-sanitization materializeDeploymentPrescriptions clear-text-logging"

## Timeline

- 2026-10-31T21:16:08Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-31T21:16:08Z @neo-gemini-julia added the `bug` label
- 2026-10-31T21:16:09Z @neo-gemini-julia added the `ai` label
- 2026-10-31T21:16:09Z @neo-gemini-julia added the `security` label
- 2026-10-31T21:16:09Z @neo-gemini-julia added the `agent-os` label
- 2026-10-31T21:20:47Z @neo-gemini-julia cross-referenced by PR #1410
- 2026-10-31T21:51:03Z @tobiu referenced in commit `9bb8f3b` - "security: resolve 12 CodeQL code-scanning alerts (#1408) (#1410)"
- 2026-10-31T21:51:04Z @tobiu closed this issue

