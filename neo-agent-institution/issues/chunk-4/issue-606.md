---
id: 606
title: Carry the merged onboarding fixes in the next Fleet package
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - build
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-08T15:07:20Z'
updatedAt: '2026-10-08T17:09:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/606'
author: neo-gpt-emmy
commentsCount: 1
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
closedAt: '2026-10-08T17:09:26Z'
---
# Carry the merged onboarding fixes in the next Fleet package

## Context

The operator is moving the remaining peers into Fleet Manager. The container plane has the merged onboarding fixes, but the installed app still bundles Brain `4eb08062` and Engine `82bc6158`. The [candidate preparation record](https://github.com/neomjs/neo-agent-institution/issues/12#issuecomment-6059248337) identifies a fresh package as the next step. Emmy owns source/package preparation; Sophie owns independent candidate verification and live FM coordination.

## The Problem

Institution `dev@472e4bb6` still pins Brain `197e659a` and Engine `82bc6158`. Rebuilding those inputs does not deliver verified-repository trust, Stop cancellation, declared Claude Desktop effort, seat-owned imported memory or the merged Claude publication guard. A Docker redeploy cannot change host-side code packaged in the app.

## The Architectural Reality

`package.json` and `package-lock.json` own the installed contracts. The cross-repository job in `.github/workflows/ci.yml` selects its Brain checkout separately. `harness/pack.mjs` stages a product with an explicit `NEO_AGENTOS_RUNTIME_ROOT`; `organism-build-info.json` records Product, Engine and runtime Brain inputs. All of these need to agree on the candidate rather than relying on ambient source or the old default build output.

The current merged inputs are Brain `aab9e2a0722c3032ddd81873b76308e27e4b1dfb` and Engine `e1b8fb0b1ad4631edef9e6a2892373aa11599fdd`. The Brain head includes the four onboarding repairs and the Claude session-ID guard; the Engine head includes the merged VDOM settlement fixes.

## The Fix

Refresh both dependency pins and the lockfile, align CI's Brain checkout, and validate the existing Institution contracts against those exact inputs. Build a fresh explicit artifact with the matching Brain runtime, run the packaged smoke and installer dry-run, and attach the candidate receipt to #12 for the coordinated install. No new packaging subsystem or application feature is proposed.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Dependency pins and lockfile | `package.json`, `package-lock.json` | Resolve the exact Brain and Engine inputs above | Fail resolution; do not silently use another checkout | Existing package workflow unchanged | Lock inspection, reproducible install and unit contracts |
| CI Brain source | `.github/workflows/ci.yml` cross-repo job | Checkout the same Brain revision as the product contract | CI failure is retained | Existing workflow | Exact-head CI |
| Packaged runtime and receipt | `harness/pack.mjs`, `harness/README.md` | Stage explicit matching Brain source and record all three owners | Reject an invalid runtime source; do not reuse a stale artifact | Candidate receipt on #12 | Fresh packaged smoke and installer dry-run |

Decision Record impact: none; updates existing dependency contracts and packaging inputs.

## Acceptance Criteria

- [ ] Package, lockfile and CI resolve the declared inputs consistently.
- [ ] Relevant existing Institution unit/contract checks pass at the candidate head; failures are resolved or explicitly attributed without weakening checks.
- [ ] A fresh artifact has an exact Product/Engine/runtime-Brain receipt, passing isolated packaged smoke and installer dry-run; the artifact is linked on #12.
- [ ] Post-merge, installed: coordinated replacement and Vega's destination-memory, identity, MCP, effort and wake witnesses remain under #12 and neomjs/neo-agent-brain#571. Source/package evidence does not close those installed outcomes.

## Out of Scope

The effort-picker UI (#600), Accounts drag repair (#601), managed Codex memory-source choice (#603), migrations or Starts during source preparation, and modifying the running plane or old seat profiles.

## Avoided Traps

Do not substitute a container update for an app update, select the stale default build artifact, introduce another feature prerequisite, or count a successful subprocess as native seat acceptance.

## Related

#12 · neomjs/neo-agent-brain#571 · neomjs/neo-agent-brain#925 · neomjs/neo-agent-brain#926 · neomjs/neo-agent-brain#927 · neomjs/neo-agent-brain#928 · neomjs/neo-agent-brain#934

Live latest-open sweep: latest 20 Institution issues on 2026-10-08, no equivalent; prior pin leaves are closed. A2A latest 30: Sophie explicitly retains independent candidate/live-UI verification and returns source preparation to Emmy; no competing source claim. MC sweep: installed onboarding/package gap, including memory `5362822e`, confirms this accepted next step. Own-assignment sweep: only #42 is open and concerns a separate view-layer investigation. KB found historical pin precedents, whose stale installed-candidate claim was checked against the actual receipt. Brain structure-map passed; owning surfaces are the existing product package/CI/harness, with no new module.

Origin Session ID: 7cdef292-c073-447b-9afd-4eaab22ecdbf
Retrieval Hint: "next installed candidate onboarding package Brain Engine pin"


## Timeline

- 2026-10-08T15:07:21Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-08T15:07:22Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-08T15:07:23Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-08T15:07:23Z @neo-gpt-emmy added the `ai` label
- 2026-10-08T15:07:23Z @neo-gpt-emmy added the `build` label
### @neo-gpt-emmy - 2026-10-08T15:11:51Z

Prescription checked: `package.json`, `package-lock.json`, `.github/workflows/ci.yml` and `harness/pack.mjs` own this gap. The package stage reads explicit product, Engine and runtime-Brain owners; changing a running service cannot replace that artifact. An independent read-only procedure audit confirmed the existing pipeline can build and dry-run the candidate without new packaging code.

The existing source checkout is now on `codex/606-fm-onboarding-candidate` from `472e4bb6`, with the old Candidate D source revision retained. Package/lock/CI changes are exactly nine inserted and nine removed lines. Lock refresh and `npm ci` succeeded on Node 24.19.0. No installed files, profiles or plane services changed. Next: existing unit contracts with CI's shared-Engine arrangement, then a fresh explicit artifact and packaged smoke.

Origin Session ID: 7cdef292-c073-447b-9afd-4eaab22ecdbf

- 2026-10-08T15:20:35Z @neo-gpt-emmy cross-referenced by PR #607
- 2026-10-08T15:21:05Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-08T17:09:26Z @tobiu referenced in commit `29e8f22` - "feat(fleet): carry merged onboarding repairs (#606) (#607)

* feat(fleet): carry merged onboarding repairs (#606)

* test(fleet): record verified candidate visual baseline (#606)"
- 2026-10-08T17:09:26Z @tobiu closed this issue

