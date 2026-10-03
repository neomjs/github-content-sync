---
id: 784
title: 'served-plane never reads ok: /mcp route, no bearer, plane block dropped'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-10-03T06:41:30Z'
updatedAt: '2026-10-03T08:04:14Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/784'
author: neo-fable-clio
commentsCount: 0
parentIssue: 351
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[x] 782 A run-bound verify effect feeds the recipe''s validation and done observers'
closedAt: '2026-10-03T08:04:14Z'
---
# served-plane never reads ok: /mcp route, no bearer, plane block dropped

Found 2026-10-03 while taking #782, whose `validation` / `done` observers gate on `served-plane`. Verified live against the canonical local plane (ingress `127.0.0.1:3102`): `GET /mcp` answers 404, `/mc/mcp` answers 401 without a bearer; with the admission token and `--mcp-path /mc/mcp` the healthcheck answers `healthy` and its payload carries `plane: {id: 'neo-local-canonical', dataRoot: '/app/.neo-ai-data'}` — yet `runHealthcheck`'s RESULT omits that block without expectations (`assertServedPlane` returns `null`; its spec pins "no expectation configured is a no-op"). Both renderers run this observer: the CLI and the vessel's setup broker (`cli.productionObservers({layout, host})`, Institution `harness/setupBroker.mjs:247`). Sub of neomjs/neo-agent-institution#351.

## Context

`firstRunRecipe.mjs` reads `served-plane` as the identity proof every later step hangs on (ADR 0041 §2.4: identity key + corroborating root, read from the served plane, never inferred around). `productionObservers().servedPlane` (`ai/scripts/setup/firstRun.mjs:140-144`) is the only production reader of that identity, and it cannot reach it.

## The Problem

Three independent causes, each sufficient:

1. **Route.** `healthcheck({url: target.endpoint, expectedStatus})` keeps `runHealthcheck`'s default `mcpPath '/mcp'`. The local-agent-os Caddyfile (`deploy/cloud/Caddyfile.local-agent-os`) routes `/mc/*`, `/kb/*` and `/fleet*` and answers 404 elsewhere. The Memory Core is `<endpoint>/mc/mcp` — the derived route every plane client composes (`ai/configBase.mjs:238`, `:345`; `ai/mcp/client/config.mjs:80`; `devFleetServer.mjs:119`; the Institution fixture plane sits behind the same loopback `/mc` ingress).
2. **Credential.** No bearer is sent; the plane admits PATs only. The run already holds the operator's plane credential as a consented file reference (`plane-credential`, admitted by `admitCredentialReference`); `composeCredentialEffects` writes that same value as the admission token secret — so the consent IS the credential the plane accepts.
3. **The plane block.** `runHealthcheck` puts `plane` in its result only when `expectedPlaneId` / `expectedPlaneDataRoot` are passed — and passing them turns a wrong plane into a THROW (`unknown`, exit 2) where the recipe wants a fresh negative read (`failed`, exit 1, "a different plane is answering", `evaluateServedPlane`). Without them the observer returns `null` and the recipe reads "the responder never identified itself".

Consequence: against a real plane `served-plane` reads `unknown` or `failed` forever; `validation`, `done` and #782's `verify` are gated off; the CLI exits 2 on a healthy provisioned plane. The unit specs pass because every served-plane arm injects a fake observer that returns the block; nothing exercises the production observer against a served route.

## The Architectural Reality

- `ai/scripts/diagnostics/mcpHealthcheck.mjs#runHealthcheck` (`:350`) — `mcpPath`, `bearerToken`, `expectedStatus`, the expectations; `assertServedPlane` (`:241`) returns `null` without expectations and fails closed with them.
- `ai/services/fleet/firstRunRecipe.mjs#observe` (`:110-122`) calls `observer(target)`; `evaluateRecipe` already holds `record` and `bound` and hands the record to question and effect evaluation (prior consent is the record's authority).
- `ai/scripts/setup/firstRun.mjs#productionObservers` (`:103-146`) builds observers over `layout` and `host` only; `host.fsModule` is the injected filesystem the CLI and the broker both supply.
- `ai/services/fleet/setupRunRecord.mjs#findConsent` — the consent for `plane-credential` holds the file's absolute path, never its content.

## The Fix

Brain only; no new file; both renderers inherit it through `evaluateRecipe`.

1. **`runHealthcheck({reportServedPlane})`** (default `false`): when set, the result carries `plane: {id, dataRoot}` whenever the payload reports a plane block, with no assertion; an absent block adds no key. `assertServedPlane` and its expectation contract are unchanged.
2. **`firstRunRecipe.mjs#observe`** passes a second argument: `(target, {record})`, the record only when BOUND (`bound ? record : null`), so another target's record never hands an observer a credential path. Existing observers ignore it.
3. **`productionObservers().servedPlane(target, {record})`**: `healthcheck({url: target.endpoint, mcpPath: '/mc/mcp', bearerToken, expectedStatus: 'healthy,degraded', reportServedPlane: true})`, where `bearerToken` is the trimmed content of the bound record's `plane-credential` file read through `host.fsModule`, or `null` before that consent; returns `health.plane ?? null`. The token lives in the call only — never logged, printed or recorded. A refused connection, a 401 or a 404 stay the observer's thrown reason (`unknown`); a wrong plane stays `failed` through the recipe's own comparison; a missing block stays "never identified itself".

## Contract Ledger

| Surface | Authority | Behavior | Edge case | Docs | Evidence |
|---|---|---|---|---|---|
| `runHealthcheck` option `reportServedPlane` | `mcpHealthcheck.mjs` (container probe + the recipe's reader) | result carries `plane: {id, dataRoot}` when the payload reports one; no assertion | absent block → no key; default `false` keeps the pinned exact result shape | JSDoc on the option | spec arms: reported / absent / default unchanged |
| observer call `(target, {record})` | `firstRunRecipe.mjs#observe` | record present only under a bound binding | `no-record`, `target-mismatch`, `version-mismatch` → `{record: null}` | module JSDoc (observer signature) | recipe spec arm pinning both cases |
| `productionObservers().servedPlane` | `firstRun.mjs` | `/mc/mcp`, consented credential as bearer, plane block returned | no consent → no bearer (the plane's refusal is the reason); wrong plane → `failed` via the recipe | observer JSDoc | CLI spec arms with a recording healthcheck double; one live receipt |

Decision Record impact: aligned-with ADR 0041 §2.4 (the served plane answers "is this value set"; nothing inferred around); no ADR text changes.

## Acceptance Criteria

- AC-1: `runHealthcheck({reportServedPlane: true})` returns `plane: {id, dataRoot}` when the payload carries a plane block and no `plane` key when it does not; without the option the pinned exact result shape is unchanged; `assertServedPlane` keeps its expectation contract.
- AC-2: `evaluateRecipe` calls every observation and effect observer with `(target, {record})`, the record only under a bound binding (`null` for `no-record`, `target-mismatch`, `version-mismatch`); one spec arm pins both cases.
- AC-3: `productionObservers().servedPlane` calls the healthcheck seam with `mcpPath '/mc/mcp'`, the consented credential file's trimmed content as `bearerToken` (none before the consent), `reportServedPlane: true` and no expectations, and returns the block; the token appears in no record, stdout or stderr; a thrown probe reads `unknown`, a wrong plane `failed`.
- AC-4: one live receipt in the PR: the production observer, given a record consented to a credential file, returns `{id: 'neo-local-canonical', dataRoot: '/app/.neo-ai-data'}` from the canonical local plane — the observer alone, no effect performed.

## Out of Scope

#782's `validation` / `verify` / `done` (built on this). The vessel's broker (inherits by the Brain pin; no consumer edit). The ingress route policy. Institution #425's attach-time `probePlaneCredential` (a different surface). A tokenless probe mode.

## Avoided Traps

Passing expectations into the healthcheck (a wrong plane would read `unknown` instead of the recipe's `failed`, and a run before create has no id to expect). Reading the admission token from the host layout's `secrets/` (that is what the run WRITES; the consent exists before the write and is the operator's reference). A second MCP client inside the recipe. Logging the bearer anywhere.

## Related

neomjs/neo-agent-institution#351 (parent) · #782 (blocked by this) · #679 (the recipe) · #678 / ADR 0041 · neomjs/neo-agent-institution#12 (the installed cut this unblocks for the wizard's journey)

Live latest-open sweep: latest 20 open Brain issues read at 2026-10-03T06:37Z (#783 … #503), no equivalent. A2A claim sweep (last 15 min): Vega Institution #473 (install leg), Ada Brain #571 (roster), Grace #414 / #760, Emmy's #12 cut, Euclid's read-only census — none on this surface. Memory Core sweep ("served-plane observer runHealthcheck 3102 /mcp 404 ingress bearer"): no prior decision; Euclid's 2026-06-18 deployed-plane probe is the `/mc/mcp` + bearer witness shape. Own-assignment sweep: #50, #51, #53 — none overlap. Structure map: existing `ai/scripts/setup` and `ai/scripts/diagnostics` modules; no new file.

Origin Session ID: fb9561d9-a0dd-4f35-912c-095864afbae4
Retrieval Hint: "served-plane observer /mc/mcp bearer reportServedPlane plane block observer context record"

## Timeline

- 2026-10-03T06:41:30Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-03T06:41:31Z @neo-fable-clio added the `bug` label
- 2026-10-03T06:41:31Z @neo-fable-clio added the `ai` label
- 2026-10-03T06:41:31Z @neo-fable-clio added the `agent-os` label
- 2026-10-03T06:41:49Z @neo-fable-clio added parent issue #351
- 2026-10-03T06:41:54Z @neo-fable-clio marked this issue as blocking #782
- 2026-10-03T06:50:06Z @neo-fable-clio cross-referenced by PR #785
- 2026-10-03T06:52:52Z @neo-fable cross-referenced by #786
- 2026-10-03T07:24:28Z @neo-fable-clio cross-referenced by PR #796
- 2026-10-03T07:45:02Z @neo-fable-clio cross-referenced by #798
- 2026-10-03T08:04:14Z @tobiu referenced in commit `b2cd8be` - "fix(setup): the served-plane observer asks the plane through the Memory Core route with the consented credential and reads the plane block as observed (#784) (#785)

The first run's production served-plane observer could never read ok against a provisioned
plane: it asked the ingress at /mcp (routed nowhere, 404), sent no bearer (401), and
runHealthcheck dropped the plane block without expectations. Both renderers run this observer.

- runHealthcheck gains reportServedPlane: the plane block as observed, asserted against nothing
- the recipe hands every observer (target, {record}), the record under a bound binding only
- productionObservers.servedPlane composes <endpoint>/mc/mcp, reads the consented credential file
  at call time and returns the block; the recipe keeps the verdict (thrown → unknown, wrong → failed)"
- 2026-10-03T08:04:15Z @tobiu closed this issue
- 2026-10-03T08:24:43Z @neo-fable-clio cross-referenced by #480

