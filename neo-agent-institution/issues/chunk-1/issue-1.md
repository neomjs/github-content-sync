---
id: 1
title: The Fleet Manager arrives with its own test suite
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - build
  - testing
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-26T21:56:08Z'
updatedAt: '2026-08-27T09:42:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1'
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
blocking:
  - '[x] 4 Harness packaging accepts separate product and Brain roots'
  - '[x] 3 Fleet list conversions regain their visual contracts'
  - '[x] 2 Fleet cards expose stale provider validation'
closedAt: '2026-08-27T09:42:30Z'
---
# The Fleet Manager arrives with its own test suite

## Goal

Receive every Fleet Manager-owned test surface here **before** Neo removes the application, styles, harness, or coverage. This is the receive side of [neomjs/neo#17805](https://github.com/neomjs/neo/issues/17805).

Source authority: neomjs/neo `dev@915c1a01e71822574c4b0a68b6394ddcecd5fcda`.

## Exact custody

| Surface | Receive |
|---|---:|
| `test/playwright/unit/apps/agentos/**` | 58 specs |
| `test/playwright/unit/harness/**` | 8 specs |
| `test/playwright/e2e/agentos/**` | 35 specs + 23 support files |
| `test/playwright/component/apps/agentos/**` | 1 spec |
| `test/playwright/visual/FleetCockpitVisual.spec.mjs` | 1 spec + 6 goldens |
| **Product coverage** | **132 paths: 103 specs + 29 support files** |

The fifteen exclusions are deliberate: eight DemoB specs plus `TearOutMatrixRows4To7NL.spec.mjs` are Engine-owned cross-window examples; two product E2Es and three unit specs target the deliberately absent tour / Mission Control source; one screenshot is unreferenced. `FleetManagerNoTourWitnessNL.spec.mjs` stays because the missing tour is intentional product behavior.

## Boundary

- Institution owns discovery, artifacts, ports, server startup, and the component/visual runners.
- Engine imports resolve through the pinned `neo.mjs` package.
- Brain-backed tests require an absolute `NEO_AGENTOS_RUNTIME_ROOT`; no config imports Brain and no cwd/sibling fallback exists.
- The E2E server uses a unique port, `reuseExistingServer: false`, and readiness under this checkout's Agent OS route.
- Darwin goldens move byte-for-byte. This migration does not repaint existing AgentCard, BufferedList, or other design failures.
- A full portable cross-repository E2E repair is not smuggled into the move. Existing reds remain visible for explicit follow-ups.
- The copied stale-validation unit witness is expected-red under #2; list/visual repair is owned by #3.

## Acceptance criteria

- [ ] **AC-1 — Exact custody.** All 132 product-coverage paths above are present; the fifteen exclusions remain absent.
- [ ] **AC-2 — Isolated unit.** The 57 Brain-independent unit files pass without Docker, credentials, or a Brain checkout; #2 remains one explicit expected-red witness.
- [ ] **AC-3 — Component.** The mounted AddAgentForm credential-boundary witness passes from this checkout.
- [ ] **AC-4 — Isolated E2E.** The portable Brain-independent E2E subset starts this checkout and passes; platform goldens remain local-only.
- [ ] **AC-5 — Explicit contract.** With an absolute Brain checkout, all 66 unit files pass and the portable cross-repository E2E set collects without unresolved imports.
- [ ] **AC-6 — CI.** CI enforces isolated unit/component/E2E and a separate explicit-Brain unit/collection job.
- [ ] **AC-7 — Ordering.** Neo removal stays blocked until this receiver is merged and green.

## Out of scope

- Deleting Neo content before this lands.
- Recreating dropped demo / Mission Control source.
- Copying Brain implementation or packaging Brain inside Institution.
- Repairing inherited visual or multi-window E2E failures during custody transfer; #2 and #3 own the known product repairs.
- Reworking packaged Brain assembly; #4 owns the explicit-root pack stage.

## Related

Related: #2

Related: #3

Related: #4


## Timeline

- 2026-08-26T21:56:09Z @neo-opus-vega added the `enhancement` label
- 2026-08-26T21:56:09Z @neo-opus-vega added the `ai` label
- 2026-08-26T21:56:10Z @neo-opus-vega added the `architecture` label
- 2026-08-26T21:56:10Z @neo-opus-vega added the `build` label
- 2026-08-26T21:56:10Z @neo-opus-vega added the `testing` label
- 2026-08-27T07:41:15Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-08-27T08:52:43Z @tobiu cross-referenced by #2
- 2026-08-27T08:53:05Z @neo-gpt-emmy marked this issue as blocking #2
- 2026-08-27T08:54:39Z @tobiu cross-referenced by #3
- 2026-08-27T08:54:48Z @neo-gpt-emmy marked this issue as blocking #3
- 2026-08-27T09:01:29Z @tobiu cross-referenced by #4
- 2026-08-27T09:01:40Z @neo-gpt-emmy marked this issue as blocking #4
- 2026-08-27T09:04:00Z @tobiu cross-referenced by PR #5
- 2026-08-27T09:09:56Z @neo-gpt-emmy cross-referenced by PR #6
- 2026-08-27T09:26:56Z @tobiu referenced in commit `a1075bc` - "fix(agentos): correct childapp workspace paths (#1)"
- 2026-08-27T09:42:30Z @tobiu referenced in commit `48fcc8e` - "Merge pull request #6 from neomjs/codex/1-fleet-tests-emmy

feat: receive Fleet Manager test ownership (#1)"
- 2026-08-27T09:42:30Z @tobiu closed this issue
- 2026-08-27T10:40:27Z @neo-opus-vega cross-referenced by #17805
- 2026-08-27T11:12:44Z @neo-gpt cross-referenced by #25
- 2026-08-28T11:42:18Z @neo-gpt-emmy cross-referenced by PR #32
- 2026-08-28T16:00:36Z @neo-gpt-emmy cross-referenced by PR #35
- 2026-09-01T20:57:22Z @neo-fable-clio cross-referenced by #66
- 2026-09-01T20:58:17Z @neo-fable-clio cross-referenced by PR #65
- 2026-09-02T01:51:22Z @neo-opus-grace cross-referenced by PR #75
- 2026-09-04T18:46:00Z @neo-gpt cross-referenced by PR #109

