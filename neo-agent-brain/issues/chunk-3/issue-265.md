---
id: 265
title: Adopt a resident-and-sufficient LM Studio instance on identifier collision
state: CLOSED
labels:
  - bug
  - ai
  - architecture
  - agent-os
assignees: []
createdAt: '2026-08-30T22:00:31Z'
updatedAt: '2026-08-30T22:35:41Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/265'
author: neo-opus-ada
commentsCount: 0
parentIssue: 29
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-30T22:35:41Z'
---
# Adopt a resident-and-sufficient LM Studio instance on identifier collision

Delivered leaf split out of #29 so a PR can close exactly what it ships. #29 stays open — it carries two further fix points this leaf does not touch.

## Context

#29's corrected mechanism identifies three fix points. This leaf is **fix point 1 only**, minted because #29 is not a one-PR ticket and a PR against the parent would project a close over undelivered work.

Scope split, for the record:

| fix point | disposition |
|---|---|
| FP1 — treat `already exists` as an ADOPT candidate | **this leaf** |
| FP2 — load-before-unload blue/green | **not implementable as written.** It prescribes reordering an LM Studio unload this codebase has never performed — verified absent in the Brain and in the pre-cut Engine tree `467fd122f3` the live plane runs, with positive controls both times. Re-anchor: #29 comment 5471321929 |
| FP3 — identify the JIT discriminator | untouched; `ttlMs` remains consulted nowhere, `parallel` remains structurally inert for the embedding role |

## The Problem

`loadLmsModel()` (`ai/services/graph/providerReadinessHelper.mjs:1195`) rejects **every** `lms` CLI failure through one path, so LM Studio's identifier refusal arrives as a generic load failure:

```js
if (error) { reject(createLmsCliError(`lms load ${model}`, error, stderr)); return }
```

The ensure verify therefore stays unsatisfied, the next round decides to replace again, and it collides again — the retry loop #29 measured at twelve cycles in six minutes, with this repo's own Memory Core embed traffic acting as the JIT re-loader.

A collision is not a failure. It reports that the desired end state may already hold, which is a question about the resident instance's **shape**.

## The Architectural Reality

| surface | anchor |
|---|---|
| uniform CLI rejection | `providerReadinessHelper.mjs:1195` — `loadLmsModel()` |
| error composition (folds stderr in) | `createLmsCliError()` — message becomes `…failed: <cause>; stderr=<text>` |
| the classifier that gates every other readiness decision | `classifyLmsLoadedModelObservation()` |
| the ensure loop and its load call site | `ensureLmsModelsLoadedOnce()` |

## The Fix

On a collision, re-probe residency and let `classifyLmsLoadedModelObservation` decide. Sufficient adopts; insufficient, unreadable, or absent stays a failure.

The adopt belongs in the **ensure layer**, not in `loadLmsModel()`: the latter is a thin CLI shim, and adopting needs residency state plus readiness semantics.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| `isLmsIdentifierCollision(error)` | LM Studio CLI refusal text | recognizes the identifier collision on the **composed** error, stderr included | must not match a failure that merely quotes the model id | function JSDoc | truth-table spec incl. the quotes-the-id discriminator |
| ensure collision path | this leaf | re-probe residency, classify, adopt only on `sufficient` | insufficient / unreadable / absent stays a failure | function JSDoc | positive + three negative specs |
| `adoptedModels` result field | this leaf | adopted instances reported separately from `loadedModels` | never merged into `loadedModels` | result-shape JSDoc | shape assertion |
| `loadLmsModel()` | existing | **unchanged** — stays a thin CLI shim with no readiness semantics | teaching it to classify puts the decision below its owner | none | placement assertion |
| error conduit | `createLmsCliError()` | predicate is exercised against the message the real conduit composes | a hand-composed fixture is not the contract | none | conduit spec + stderr-fold mutant |

## Acceptance Criteria

- [ ] A collision over a **sufficient** resident instance is adopted, not retried, and is reported in `adoptedModels`.
- [ ] A collision over an **insufficient** resident instance stays a failure. **Mutant: adopting on the collision alone, ignoring the classifier, must red.**
- [ ] A collision whose re-probe returns nothing stays a failure — absence of a sufficiency answer is not sufficiency.
- [ ] A non-collision load failure is untouched by the adopt path, proven with a **sufficient** resident row present so a residency-keyed adopt would be caught.
- [ ] `adoptedModels` is distinct from `loadedModels`.
- [ ] The predicate is exercised through the **real conduit** — stubbed `execFileFn` error + stderr → `createLmsCliError()` → predicate — not a pre-composed `Error`. **Mutant: removing the stderr fold must red.**
- [ ] The race fixture reaches the load. The ensure probes residency three times before attempting one; a fixture that makes the model resident earlier skips the load and tests nothing.
- [ ] `loadLmsModel()` gains no readiness semantics.

## Out of Scope

FP2 and FP3 (see the table above) · the underlying race's existence, which this leaf does not claim to have fixed or falsified — it removes the swallowed collision that turned the race into an infinite retry · #131's duration residual, unaffected.

## Avoided Traps

- **Adopting on the collision alone.** Rejected: it greens a verify a wrong-shaped instance cannot honour.
- **Reading an empty or unreadable probe as sufficiency.** Rejected: that is the fail-open shape the guard exists to prevent.
- **Putting the adopt inside `loadLmsModel()`.** Rejected: the shim would acquire readiness semantics it does not own.
- **Testing the predicate against a hand-written message.** Rejected — and it was the first draft. The refusal arrives on **stderr** and is folded in by `createLmsCliError()`; a self-authored string tests the spec's own analogue rather than the production contract.

## Related

#29 (parent; remains open for FP2/FP3) · #131 (duration residual)

Origin Session ID: 3f2c672e-4fb3-41c9-bbd5-e43d4e1f5be5

Retrieval Hint: "LM Studio identifier collision adopt resident sufficient ensure layer classifier not retry"

Live latest-open sweep: 12 latest open Brain issues checked at 2026-08-30T21:59:50Z; duplicate probes on adopt / collision / already-exists return only #29 itself. No equivalent leaf exists.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5 · Claude Code


## Timeline

- 2026-08-30T22:00:32Z @neo-opus-ada added the `bug` label
- 2026-08-30T22:00:32Z @neo-opus-ada added the `ai` label
- 2026-08-30T22:00:32Z @neo-opus-ada added the `architecture` label
- 2026-08-30T22:00:33Z @neo-opus-ada added the `agent-os` label
- 2026-08-30T22:00:50Z @neo-opus-ada referenced in commit `b4f1077` - "test(provider): exercise the collision predicate through the real error conduit (#265)

The first draft asserted against a hand-composed Error whose message I wrote to
contain the refusal text. That tested the spec's own analogue, not the contract:
LM Studio's refusal actually arrives on stderr, and `createLmsCliError()` folds
it in as `; stderr=<text>`. A fixture that skips that composition cannot notice
if the composition changes.

Every collision in these specs is now produced by driving the real
`loadLmsModel()` with a stubbed `execFileFn` that returns a CLI error plus
stderr, so the chain execFileFn → createLmsCliError → predicate → adopt is bound
end to end. Adds a dedicated conduit spec asserting the composed message carries
`stderr=` and that the predicate recognizes it.

Mutant run: removing the stderr fold from `createLmsCliError()` reds two specs.
The pre-composed fixture could not have caught that.

Raised by @neo-gpt-emmy on PR #264 (RA-2)."
- 2026-08-30T22:01:22Z @neo-opus-ada cross-referenced by PR #264
- 2026-08-30T22:35:42Z @tobiu closed this issue
- 2026-08-30T22:36:55Z @neo-opus-ada cross-referenced by #29
- 2026-09-04T10:31:36Z @neo-gpt-emmy cross-referenced by #305
- 2026-09-04T11:53:33Z @tobiu cross-referenced by PR #308

