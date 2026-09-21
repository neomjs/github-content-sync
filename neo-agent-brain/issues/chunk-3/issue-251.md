---
id: 251
title: 'Disabling kbSync does not stop it: the cascade path consults no gate'
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - architecture
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-08-30T16:24:28Z'
updatedAt: '2026-08-30T17:39:50Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/251'
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
closedAt: '2026-08-30T17:39:50Z'
---
# Disabling kbSync does not stop it: the cascade path consults no gate

## Context

Surfaced during `/peer-role` review of the Brain container-cutover seam (D#17846 / #12) with @neo-gpt-emmy on 2026-08-30, and filed at her request. **The defect is live at current Brain `dev` and is independent of the cutover** — it is filed separately rather than folded into a cutover checklist because a procedure line does not survive the next operator who enables `primary-dev-sync`.

The orchestrator exposes an operator-facing `kbSync` enable. Setting it false does not stop kbSync: a second, ungated call path invokes the same work.

Cross-verified: @neo-gpt-emmy independently confirmed the finding at current Brain `dev`, corrected one of my proposed mitigations (see Avoided Traps), and supplied the control boundary reproduced in the Acceptance Criteria.

## The Problem

`enables.kbSync` gates the **scheduled** task only. The `primary-dev-sync` service invokes `runKbSync()` on its own authority, consulting no enable at any point.

Token census as a positive control, because a bare absence proves nothing:

| file | occurrences of `enables` |
|---|---:|
| `ai/daemons/orchestrator/scheduling/registry.mjs` | **15** |
| `ai/daemons/orchestrator/services/PrimaryRepoSyncService.mjs` | **0** |

Three properties make this worse than a missing check:

1. **Every root-config state reaches the cascade.** `syncPrimaryDev()` skips only on `invalid`; `configured` routes to `syncConfiguredDevRoots()`, and the `unset` fallthrough returns `syncDevRoot({root: primaryRoot, …})` whose signature defaults `runKbSync = true`.
2. **Probe failure fails OPEN.** The class JSDoc states the layer *"falls back to the full KB cascade whenever the revision or changed-path probes fail"* — a failed probe produces **more** cascade, not less.
3. **The default stale strategy is destructive.** `resolveStaleStrategy({staleStrategy, deleteStale = true})` returns `delete-upfront` when no strategy is passed, which removes stale rows **before** embedding. `ai/scripts/lint/lint-openapi-service-parity.mjs:132` records `#16577` as the measurement proving this pipeline reaches the window between the delete and the replacement embed.

So an operator who sets `kbSync = false` has not disabled a read-mostly refresh; they have failed to disable a destructive-by-default full-corpus rebuild.

**Severity bound, stated honestly:** `primaryDevSyncEnabled` is `leaf(null, 'NEO_ORCHESTRATOR_PRIMARY_DEV_SYNC_ENABLED', 'boolean')` resolved through `resolveDeploymentEnabled`, so the effective default is deployment-scoped. @neo-gpt-emmy verified that cloud currently defaults `primary-dev-sync` disabled; I did not independently verify the resolved cloud value. This is therefore **latent under the current cloud default and live on any explicit enable** — not firing in production today.

## The Architectural Reality

| surface | anchor |
|---|---|
| gated path (correct) | `ai/daemons/orchestrator/scheduling/registry.mjs:93` — `if (!enables.kbSync) return null;` |
| ungated cascade | `ai/daemons/orchestrator/services/PrimaryRepoSyncService.mjs` — `syncConfiguredDevRoots()`: `if (completed > 0 && kbSyncRequired) { this.runKbSync(primaryRoot, …) }` |
| unset-roots fallthrough | same file, `syncPrimaryDev()` — returns `this.syncDevRoot({root: primaryRoot, rootKey: 'primaryRoot', …})` |
| per-root cascade default | same file — `syncDevRoot({… runKbSync = true …})` |
| destructive default | `ai/services/knowledge-base/VectorService.mjs` — `resolveStaleStrategy()` → `deleteStale ? 'delete-upfront' : STALE_STRATEGY_SKIP` |
| enable leaf | `ai/configBase.mjs:2030` — `primaryDevSyncEnabled: leaf(null, …)`; resolved at `ai/daemons/orchestrator/Orchestrator.mjs:1301` |
| kbSync enable leaf | `ai/configBase.mjs:2086` — `kbSyncEnabled: leaf(null, 'NEO_ORCHESTRATOR_KB_SYNC_ENABLED', 'boolean')`; resolved at `Orchestrator.mjs:1299` |
| composition seam — scheduled consumer | `ai/daemons/orchestrator/scheduling/pipeline.mjs:153` — `enables: { kbSync: orchestrator.kbSyncEnabled, … }` |
| composition seam — cascade consumer | `ai/daemons/orchestrator/scheduling/pipeline.mjs:456-464` — the `'primary-dev-sync'` service-runner passes `devSyncRootsConfig` and **no kbSync authorization** |

The shape is a guard sitting below one of two convergent paths: both routes reach the same work, only one passes the gate.

The composition seam is where the divergence is actually visible. `pipeline.mjs:153` hands the scheduled path `orchestrator.kbSyncEnabled`; the `'primary-dev-sync'` runner twelve hundred lines later receives roots, task state, health and a logger — and nothing carrying that enable.

Worth noting for whoever threads the authorization: the two enables do not even share a resolver. `kbSyncEnabled` resolves through `resolveCloudOnlyEnabled` (`Orchestrator.mjs:1299`) while `primaryDevSyncEnabled` resolves through `resolveDeploymentEnabled` (`Orchestrator.mjs:1301`). The cascade must consume the **kbSync** enable; consuming the `primary-dev-sync` one would type-check, run, and re-open the same hole under a different name.

No new file is introduced. The fix lands in the existing `ai/daemons/orchestrator/services/` folder, sibling to `TenantRepoSyncService.mjs`, with the gate precedent in `scheduling/registry.mjs`. Structure-map gate: N/A — no file creation or relocation.

## The Fix

`PrimaryRepoSyncService` receives **one explicit kbSync authorization** and refuses every cascade when it is false. Authorization is threaded from the same resolved enable the scheduled registry reads, so the two paths cannot diverge again.

Missing authorization must **fail closed**. An optional-chain or backward-compatible default reintroduces the exact bypass this ticket closes.

Every refusal returns an explicit disabled receipt rather than a silent no-op, so the negative controls below have something to assert against.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| `PrimaryRepoSyncService` cascade entry | the resolved `enables.kbSync` the scheduled registry already reads | one explicit authorization parameter consulted before any `runKbSync()` | absent authorization fails closed, never defaults true | service JSDoc | unit: positive + four negative controls |
| `syncConfiguredDevRoots()` | this ticket | zero `runKbSync()` invocations when unauthorized | `completed > 0` no longer sufficient | service JSDoc | invocation spy |
| `syncPrimaryDev()` unset-roots fallthrough | this ticket | same refusal on the primary-root path | `unset` must not bypass | service JSDoc | invocation spy |
| changed-path-probe failure path | class JSDoc line 36 | refusal outranks the fail-open fallback | probe failure must not escalate to full cascade | service JSDoc | forced-probe-failure spec |
| cascade receipt | this ticket | explicit disabled reason code | silent skip is not acceptable | service JSDoc | receipt assertion |
| `scheduling/registry.mjs` gate | existing | unchanged; retained as sibling control | — | none | existing coverage |
| composition seam (`pipeline.mjs`) | `orchestrator.kbSyncEnabled` | one resolution feeds both `enables.kbSync` and the `'primary-dev-sync'` runner's authorization | wiring `primaryDevSyncEnabled` instead type-checks and re-opens the hole | pipeline JSDoc | composition spec; mutant hardcoding authorization true must red it |

## Decision Record impact

`none` — this restores the intended semantics of an existing enable; it does not change ADR authority.

## Acceptance Criteria

- [ ] `PrimaryRepoSyncService` consults one explicit kbSync authorization before every `runKbSync()` invocation.
- [ ] Missing authorization fails closed; no optional-chain or backward-compat path yields an authorized cascade.
- [ ] Positive control: authorization true + a KB-relevant completed root invokes `runKbSync()` **exactly once**.
- [ ] Negative control: authorization false + completed root → zero invocations, explicit disabled receipt.
- [ ] Negative control: authorization false + changed-path-probe failure → zero invocations (the fail-open fallback does not override refusal).
- [ ] Negative control: authorization false + unset `devSyncRoots` (primary-root fallthrough) → zero invocations.
- [ ] Negative control: authorization false + metadata-reset path → zero invocations.
- [ ] The scheduled-registry gate retains its own coverage as a sibling control, not as the only gate.
- [ ] **Composition coverage** proves the scheduled and cascade paths consume the **same resolved kbSync enable**; service-unit coverage may inject explicit authorization values.
- [ ] Named mutant: hardcoding the cascade authorization to `true` must **red the composition spec**. A mutant that only reds the unit specs has not exercised the divergence class.
- [ ] The cascade consumes the enable resolved from `kbSyncEnabled` (`resolveCloudOnlyEnabled`), not `primaryDevSyncEnabled` (`resolveDeploymentEnabled`). Wiring the wrong one type-checks and re-opens the hole.

## Out of Scope

- **The kbSync identity/disposition** (retire vs Brain identity vs parser-over-tenant-mirror). That is D#17846, high-blast, and requires family-keyed quorum; a two-maintainer A2A does not graduate it.
- **The container cut itself** — #12 (Vega-owned), and #184 / #198 (Emmy-owned).
- **Whether `primary-dev-sync` should be enabled in any given deployment.** This ticket makes the existing enable honest; it does not re-decide the deployment default.
- **The Stop-hook enforcement question** and hook transport — #250 / `neo-agent-skills#21`.

## Avoided Traps

- **"Document it in the cutover checklist."** Rejected: the defect is live independent of the cutover, and a checklist line does not bind the next operator who enables `primary-dev-sync`.
- **"Unset `devSyncRoots` as the suppressor."** This was my own proposed mitigation and it is wrong — @neo-gpt-emmy falsified it, and I verified: the `unset` branch falls through to `syncDevRoot({root: primaryRoot, …})` with `runKbSync` defaulting true. Unsetting the roots reaches the cascade by a different door.
- **Guarding only inside `runKbSync()`.** Rejected: it would leave the decision distributed across call sites and produce no receipt the negative controls can assert against. The authorization belongs at the service boundary.
- **Treating `delete-upfront` as an opt-in branch.** It is the resolved default when no strategy is supplied; framing it as opt-in understates the blast radius.
- **Requiring every new spec to assert against the real resolved enable.** This was my own first formulation of the last AC, and @neo-gpt-emmy corrected it: injecting explicit `true`/`false` into the pure `PrimaryRepoSyncService` units is dependency injection, not a self-confirming fixture, and coupling every unit to `AiConfig` would cost isolation and make the four negative controls harder to state. The anti-self-confirming property belongs at the **composition seam**, where the divergence actually lives — one spec proving both consumers read the same resolution — while the units stay injectable. The rejected version would have bought no extra coverage and spent real isolation.

## Related

- #12 — Receive Agent OS deployment and prove the Brain image (the cutover leaf this was found under)
- #184 · #198 — the source/package cutover sequence
- D#17846 — GitHub content sync as its own repository (owns the durable kbSync identity question)
- `#16577` — the measurement establishing that this pipeline reaches the delete-before-embed window

Origin Session ID: 7ea46d7d-5fba-4402-b328-c22df97494ea

Retrieval Hint: "kbSync cascade authorization primary-dev-sync enables gate fail-open delete-upfront"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5 · Claude Code

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-08-30T16:22:45Z; no equivalent found. A2A in-flight claim sweep over 30 messages (all read-states, ~last 60 min): no competing claim on this scope; the filing request itself is @neo-gpt-emmy's message of 16:18Z.


## Timeline

- 2026-08-30T16:24:30Z @neo-opus-ada added the `bug` label
- 2026-08-30T16:24:30Z @neo-opus-ada added the `ai` label
- 2026-08-30T16:24:30Z @neo-opus-ada added the `testing` label
- 2026-08-30T16:24:30Z @neo-opus-ada added the `architecture` label
- 2026-08-30T16:24:30Z @neo-opus-ada added the `agent-os` label
- 2026-08-30T16:24:48Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-08-30T16:48:51Z @neo-gpt-emmy cross-referenced by #253
- 2026-08-30T17:02:58Z @neo-opus-ada cross-referenced by PR #254
- 2026-08-30T17:16:20Z @neo-opus-ada referenced in commit `aa685ae` - "docs(orchestrator): document kbSyncAuthorized at all six threaded surfaces (#251)

The guard's own helper carried a full contract while the six methods that thread
it carried none — a new option reached `runTask`, `syncPrimaryDev`,
`syncConfiguredDevRoot`, `syncConfiguredDevRoots`, `syncDevRoot` and
`resolveMetaAndPull` undocumented.

Each `@param` now states where the value is resolved from, that only an explicit
`true` authorizes, and the trap specific to that surface: the `unset`-roots
fallthrough is not a suppressor; `syncConfiguredDevRoot` threads it for the
metadata leg though its ff-pull leg is inert; `syncConfiguredDevRoots` records a
receipt rather than skipping silently; `syncDevRoot` keeps authorization
orthogonal to the `runKbSync` ownership flag and outranks the fail-open probe
fallback; `resolveMetaAndPull` is the fourth and least obvious cascade door.

Raised by @neo-gpt-emmy on PR #254."
- 2026-08-30T17:39:51Z @tobiu closed this issue
- 2026-08-30T17:39:51Z @tobiu referenced in commit `90d41ff` - "Merge pull request #254 from neomjs/ada/251-kbsync-cascade-gate

Gate the kbSync cascade on one explicit authorization (#251)"
- 2026-08-30T20:31:05Z @neo-opus-ada cross-referenced by #260
- 2026-08-30T20:55:03Z @neo-opus-ada cross-referenced by #262
- 2026-08-30T22:09:35Z @neo-gpt-emmy cross-referenced by #184
- 2026-08-30T22:36:55Z @neo-opus-ada cross-referenced by #29
- 2026-08-31T01:56:23Z @neo-opus-ada cross-referenced by #263

