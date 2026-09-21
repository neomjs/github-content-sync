---
number: 5
title: 'feat: receive Fleet Manager test ownership (#1)'
author: tobiu
state: CLOSED
createdAt: '2026-08-27T09:03:58Z'
updatedAt: '2026-08-27T09:10:05Z'
closedAt: '2026-08-27T09:10:05Z'
mergedAt: null
head: codex/1-fleet-tests
base: dev
url: 'https://github.com/neomjs/neo-agent-institution/pull/5'
contentTrust:
  projected: true
  quarantined: 0
  signals: []
---
Resolves #1

Receives Fleet Manager test ownership in Institution before Neo removes the product: 132 product-coverage paths (103 specs + 29 support files), local unit/component/E2E/visual runners, a pinned Engine dependency, and an explicit absolute Brain binding. The move preserves all 28 goldens byte-for-byte and leaves product repairs in #2 and #3.

Evidence: L3 (live local Node and Chrome execution plus explicit-Brain contract probes) → L3 required (AC-2 through AC-5 runtime surfaces). No residuals.

## AC Evidence

| AC | Evidence |
|---|---|
| AC-1 | Exact source census: 58 app unit + 8 harness unit + 35 E2E + 1 component + 1 visual specs; 29 owning support files; all 15 exclusions absent. |
| AC-2 | `npm run test-unit`: 702/702; the missing stale-validation behavior remains one explicit expected-red witness owned by #2. |
| AC-3 | `npm run test-components`: 1/1 mounted credential-boundary witness green. |
| AC-4 | `NEO_E2E_SKIP_PLATFORM_VISUAL=1 npm run test-e2e`: 7/7 portable isolated cases green on this checkout. |
| AC-5 | Absolute Brain binding: 812/812 unit cases green; CI-mode cross-repository E2E collection resolves 37 cases in 31 files. |
| AC-6 | `.github/workflows/ci.yml` separates Institution-only execution from the explicit Brain checkout and shares one physical Engine install. |
| AC-7 | neomjs/neo#17805 remains open and untouched; removal begins only after this receiver lands green. |

## Deltas from ticket

- The final census also found and received the AgentOS-owned component witness plus Fleet visual suite and six goldens; leaving them in Neo would have created dangling coverage at deletion time.
- Darwin visual baselines are copied exactly, not refreshed. AgentCard/BufferedList design repairs are deferred to #3.
- One copied card assertion exposed missing Institution view/SCSS behavior. It stays expected-red under #2 rather than importing that fix.
- `harness/pack.mjs` still needs a deliberate two-root packaging design; #4 owns it. This receiver does not recreate `ai/**`, `package.brain.json`, or a bundled Brain.

## Test Evidence

All required coverage runs in CI. Outside CI, all 28 PNG artifacts were blob-compared against Neo `origin/dev`; zero differed. The six Fleet visual cases collect locally but are intentionally not refreshed or promoted to green evidence in this custody PR.

## Post-Merge Validation

- [x] None — Engine removal remains a separate neomjs/neo#17805 workflow after the human merge gate.

Authored by Emmy (GPT-5.6 Sol Ultra, Codex). Session d39e8182-295f-418a-82cd-a96be9c08e4f.


## Comments

### `@neo-gpt-emmy` commented on 2026-08-27T09:10:04Z

Superseded by #6. The first PR was opened through the machine-default tobiu identity from an unmapped temp checkout; #6 has identical tree content, corrected Emmy commit authorship, and Emmy PR authorship.

---

