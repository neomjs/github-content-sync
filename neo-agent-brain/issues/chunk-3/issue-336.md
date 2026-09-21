---
id: 336
title: The LM Studio supervisor relaunches the app inside its update window
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees: []
createdAt: '2026-09-06T10:06:08Z'
updatedAt: '2026-09-06T12:06:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/336'
author: neo-opus-grace
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
closedAt: '2026-09-06T12:06:11Z'
---
# The LM Studio supervisor relaunches the app inside its update window

> **Scope note for the prio-0 focus ledger.** Brain-repo host infrastructure on an explicit operator lane; not stacked on `neomjs/neo#18303` / `neomjs/neo#18304`.

`unowned-rationale:` the fix lands in the same supervisor/residency surface as #29 (LM Studio residency hook), which @neo-opus-ada owns. Parking it rather than forking that surface in parallel — fold it into #29's lane or claim it standalone. I hold the full diagnosis and will hand it over on request.

## Context

Operator account, 2026-09-05: LM Studio offered a **"Restart to Update"** button. They clicked it, LM Studio quit — and it came back **almost instantly**, fast enough that the update never completed. It only finished after a full machine reboot.

The supervisor and the thing it supervises disagree about what "the process exited" means. LM Studio exits *in order to* be replaced; the supervisor reads the exit as death and undoes the replacement. Neither side is wrong locally.

## The Problem

`ai/daemons/orchestrator/services/ConfiguredTaskDefinitionsService.mjs:117-123` defines the lane:

```js
command        : 'lms',
args           : ['server', 'start', '--port', String(AiConfig.orchestrator.lms.port)],
expectedCommand: 'lms server',
// `lms server start` is fire-and-exit: it wakes the LM Studio service and returns, so
```

That comment is the defect in one line: **`lms server start` wakes the LM Studio application.** The orchestrator polls at `poll=3000ms` (observed in `orchestrator.log` at boot). So while host-edge is healthy, quitting LM Studio for any reason returns it within roughly three seconds — matching "almost instantly" exactly. An updater that needs to swap a bundle or an extension pack does not get that long.

There is a supported opt-out — `NEO_ORCHESTRATOR_LMS_ENABLED=false` (`src/composition/orchestrator/hostEdgeProfile.mjs:99`, and its JSDoc names the no-LM-Studio case) — but it is a **plist edit plus a launchd reload**, i.e. an OS-level operation, to perform a routine application update. There is no runtime hold.

## Observation vs inference — the 09-05 instance was NOT host-edge

Stated plainly because the convenient reading is wrong, and the ticket is worth more without it:

| Fact | Value |
|---|---|
| LM Studio app bundle mtime | **2026-08-28 16:54**, v0.4.23+1 — the *app* never updated |
| `~/.lmstudio/extensions/backends/vendor/_amphibian` mtime | **2026-09-05 13:09** |
| host-edge last healthy | **2026-09-04T15:39:53Z**, then crash-looping (#335) |

host-edge was **already dead** on 09-05, so it cannot have been what relaunched LM Studio that day. What updated was a **runtime-extension pack**, not the application. The mechanism is nonetheless real, is confirmed from source rather than inferred, and is **live again as of 2026-09-06T10:02Z** now that host-edge has been restored — so the trap is armed today even though it did not spring on 09-05.

## The damage a half-applied backend update leaves

Independently measured the same morning, and the reason this matters beyond inconvenience.

LM Studio's registry selected `mlx-llm-mac-arm64-apple-metal-nax-advsimd@1.11.0`, which launches:

```
~/.lmstudio/extensions/backends/vendor/_amphibian/app-mlx-generate-mac26-arm64@33/bin/python
```

**That directory does not exist.** Installed are `@28`, `@30`, `@31`, `@32`. The registry advanced; the payload never landed. Every MLX model load then dies in the embedded interpreter:

```
libc++abi: terminating due to uncaught exception of type std::runtime_error:
  failed to get the Python codec of the filesystem encoding
sys.prefix = '/install'   sys.path = ['/install/lib/python311.zip', ...]
```

Surfaced to the operator only as `Error loading model. (Exit code: null).`

Controlled experiment — same model, same machine, **only the selected engine changed**:

| Selected MLX engine | `lms load google/gemma-4-26b-a4b` |
|---|---|
| `@1.11.0` | fails, exit code null |
| `@1.10.1` | **loaded in 7.57 s** |

GGUF/llama.cpp models were never affected, which is exactly why the operator observed the embedding model working while the chat model did not — the two use different backends, and only one was broken.

## The Architectural Reality

`ai/services/graph/providerReadinessHelper.mjs` already observes runtime state — `fetchLmsSelectedRuntimeRows` (`:1284`) shells `lms runtime ls`, and `createLmsLoadFailureGuard` (`:1330`) bounds repeated same-fingerprint load failures and re-arms on an observed selected-row change. Its JSDoc states the deliberate limit:

> *"The vendor table cannot identify which selected runtime owns one missing model"*

Correct as written — the table cannot. But **the filesystem can**: the selected engine names a backend directory, and that directory's presence is a cheap local check. Today the guard converts an unloadable engine into bounded silence rather than an attributable cause, and during this incident nothing ran it at all, because host-edge was down.

Structure-map gate: no new `.mjs` proposed. Owning folders are `ai/daemons/orchestrator/services/` (hold) and `ai/services/graph/` (attribution).

## The Fix

Two independent changes; either is useful alone.

1. **A maintenance hold on the supervised-process lane.** A bounded, self-expiring pause an operator can take without touching launchd — so "I am updating LM Studio" is expressible in our logic. Self-expiry is the safety property: a hold that outlives the update becomes a silent inference outage of exactly the kind #335 describes.
2. **Attribute an unloadable selected runtime.** When an MLX load fails and the selected engine's backend directory is absent, say so — the operator should read *"selected runtime `@1.11.0` expects backend `…@33`, which is not installed"*, not `Exit code: null`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| supervised-process lane | AC-1 | operator-takeable, self-expiring hold | `NEO_ORCHESTRATOR_LMS_ENABLED=false` + reload | local Agent OS README | `poll=3000ms`; the fire-and-exit comment |
| `NEO_ORCHESTRATOR_LMS_ENABLED` | `hostEdgeProfile.mjs:99` | unchanged; documented as the coarse opt-out | — | same | JSDoc at that leaf |
| LMS load-failure diagnosis | AC-2 | name the missing backend path | current bounded silence | — | `@1.11.0` vs `@1.10.1` table |

## Decision Record impact

`none`.

## Acceptance Criteria

- [ ] An operator can pause the LM Studio lane without editing a plist or invoking `launchctl`, and the pause is bounded and self-expiring.
- [ ] With the hold taken, LM Studio stays down for its full duration — asserted by an arm that **fails without the hold**, not only by one that passes with it.
- [ ] An MLX load failure whose selected engine names an absent backend directory reports that path. Test drives the absent-directory case; a green that cannot go red on it does not count.
- [ ] The coarse opt-out and the new hold are documented together, so a reader can tell which is for "no LM Studio on this machine" and which is for "updating right now".
- [ ] Reboot/update receipt: apply a real LM Studio update with host-edge running and the hold taken; the update completes. Post-merge-only, needs a real pending update.

## Out of Scope

- The `already exists` residency race — #29 owns it.
- Repairing LM Studio's extension installer. Third-party; we detect and report, we do not fix their updater.
- Recommending OS-level settings changes to operators. Explicit operator direction: our logic owns this, rather than our guides asking a small macOS+LM-Studio audience to change host settings for something we can handle.
- The host-edge crash-loop and its runtime root — #335.

## Avoided Traps

- **A longer poll interval.** Turns a certainty into a race and degrades genuine crash recovery. The supervisor needs to know an exit was *intended*, not to notice it later.
- **Blaming host-edge for the 09-05 half-update.** Tidy, and false — host-edge was dead. Filed with the contradiction visible; a future reader who assumes the causal link will mis-scope the fix.
- **An unbounded hold.** Reproduces #335's outage with a friendlier name.
- **Trusting the vendor table alone.** Its own JSDoc says it cannot attribute a missing model to a runtime. The filesystem check is what adds the information, not more parsing.

## Related

- #335 (host-edge crash-loop / seat-coupled runtime root — the sibling from this session)
- #29 (LM Studio residency hook; nearest owner surface)
- #84 · #90 (eventual Docker cutover)

Origin Session ID: 70502f9a-5b14-4dcf-bcdf-4a29b546df77

Retrieval Hint: `query_raw_memories` — "LM Studio restart to update supervisor relaunch MLX backend missing payload amphibian"; strings `app-mlx-generate-mac26-arm64@33`, `failed to get the Python codec of the filesystem encoding`.


## Timeline

- 2026-09-06T10:06:09Z @neo-opus-grace added the `bug` label
- 2026-09-06T10:06:10Z @neo-opus-grace added the `ai` label
- 2026-09-06T10:06:10Z @neo-opus-grace added the `agent-os` label
- 2026-09-06T10:20:05Z @neo-opus-grace cross-referenced by #337
### @neo-opus-grace - 2026-09-06T12:06:10Z

Closing NOT_PLANNED. The mechanism is real — I reproduced it — but it lives entirely in a plane that is being deleted.

The LM Studio supervisor is the host-edge `ProcessSupervisor`. **#84** hard-cuts this machine to the canonical Docker Agent OS and deletes the legacy host ownership; **#90** replaces it with the cloud container topology. Building a maintenance-hold verb into a supervisor scheduled for removal is work whose only consumer is a surface with a deletion ticket, which is what **#191** rules against.

**Worth keeping, and it needs no ticket:** LM Studio's registry can advance to a backend build whose payload never installed (`app-mlx-generate-mac26-arm64@33` absent while @28/@30/@31/@32 exist), and every MLX load then dies with `failed to get the Python codec of the filesystem encoding`, surfaced only as `Exit code: null`. The repair is `lms runtime select <previous MLX engine>`; verified today, `@1.11.0` → fails, `@1.10.1` → loads in 7.57 s. That is an operator note, not a backlog item.

Adjacent and still open on its own merits: **#29** (LM Studio residency hook).


- 2026-09-06T12:06:11Z @neo-opus-grace closed this issue
- 2026-09-06T12:06:54Z @neo-opus-grace cross-referenced by #84

