---
id: 335
title: Host daemons run from a seat tree a peer's install can prune
state: CLOSED
labels:
  - bug
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-06T10:05:08Z'
updatedAt: '2026-09-06T12:06:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/335'
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
closedAt: '2026-09-06T12:06:09Z'
---
# Host daemons run from a seat tree a peer's install can prune

> **Scope note for the prio-0 focus ledger.** This is Brain-repo host infrastructure, filed on an explicit operator lane. It is not stacked on `neomjs/neo#18303` / `neomjs/neo#18304` and asks nothing of the engine seats.

## Context

2026-09-06, after an operator reboot, Memory Core semantic recall was dead for **every** seat on this machine:

```
Cannot execute query_raw_memories: Memory Core is not fully operational:
  - Embedding write canary failed: provider-unreachable:ECONNREFUSED — backing off 600000ms (streak 8)
```

The read path is fail-closed on the *write* canary, so an unreachable inference provider takes the whole semantic surface with it. LM Studio was running with its server off and no models resident. Nothing brought it back, because the daemon that owns that job was dead.

`com.neomjs.agent-os-host-edge` had been crash-looping since **2026-09-04T15:39:53Z** (the last line in `orchestrator.log`). At diagnosis: `runs = 274`, `last exit code = 1`, `active count = 0`, **4,648** identical crashes in a 5.3 MB `launchd.err.log`, respawning every 10 s under `KeepAlive` + `ThrottleInterval 10`. It reached 324 runs during the ten minutes it took to write this up.

```
Error [ERR_MODULE_NOT_FOUND]: Cannot find package 'dotenv' imported from
  .../agent-os-runtime/03035d1b.../ai/daemons/orchestrator/hostEdge.mjs
```

Observation vs inference: the crash text and counters are read from `launchctl print` and the launchd err log. The causal chain below is inference from those plus the resolution walk, each step of which was executed.

## The Problem

The plist pins `WorkingDirectory` to a **frozen 2026-08-13 snapshot of the ENGINE repo**:

```
/Users/Shared/agents/<a-seat>/neomjs/neo/.neo-ai-secrets/agent-os-runtime/03035d1b…
```

and that snapshot's `node_modules` is a **symlink into that seat's engine checkout**:

```
<pinned root>/node_modules -> /Users/Shared/agents/<a-seat>/neomjs/neo/node_modules
```

The Brain split moved `ai/daemons/orchestrator/` into this repository and `dotenv` left `neo/package.json` with it. The next install in that seat's engine checkout pruned `dotenv` — and took the machine's host daemon with it. The pinned snapshot still *declares* `dotenv@^17.4.2` in its own `package.json:219`; the symlink means that declaration governs nothing.

Node's resolution walk, executed at every level:

| Directory | `node_modules` | `dotenv` |
|---|---|---|
| `<pinned>/ai/daemons/orchestrator` | ✗ | — |
| `<pinned>/ai/daemons` | ✗ | — |
| `<pinned>/ai` | ✗ | — |
| `<pinned>` | ✓ (symlink) | ✗ |
| `.../agent-os-runtime` | ✗ | — |
| `<seat>/neomjs/neo` | ✓ | ✗ |

**This is a class, not one plist.** `com.neomjs.agent-os-wake` carries the **same** `WorkingDirectory`, the **same** symlinked `node_modules`, and `dotenv` is equally absent for it. It survives only because `ai/daemons/wake/receiver.mjs` does not import it. The next dependency the engine repo drops decides whether wake delivery dies too — silently, and with no relationship to any change in this repository.

The pinned tree is also **pre-split dead code**: it imports `../../deploy/hostEdgeProfile.mjs`, where canonical is `src/composition/orchestrator/hostEdgeProfile.mjs`. Installing `dotenv` into it would have resurrected an obsolete tree, not repaired anything.

## The Architectural Reality

- `ai/daemons/orchestrator/hostEdge.mjs` is the launchd-supervised entrypoint. Its own header states the intent this defect violates: *"the portable path and the launchd-supervised path execute the SAME code with the SAME inputs — launchd supplies restart-on-login, never the configuration."* A frozen per-seat snapshot is a third, unowned copy.
- `ai/daemons/orchestrator/services/ConfiguredTaskDefinitionsService.mjs:117-123` makes host-edge the LM Studio supervisor (`label: 'lms server (LM Studio CLI)'`). LM Studio's own `~/.lmstudio/.internal/http-server-config.json` carries `"autoStartOnLaunch": false`, so **host-edge is the only thing that brings inference up after a reboot.** Its death is therefore a full-plane outage on the next boot, not a degradation.
- The runtime-root mechanism (`NEO_AGENTOS_RUNTIME_ROOT`) was predicted to rot exactly this way on 2026-08-26: *"not managed, creating a future stale-authority risk."* This is that risk firing.
- The pinned root also holds the `.env` the daemon reads, and its `NEO_AGENT_IDENTITY` names a **seat**, so the machine daemon runs under a seat's identity. Related: #244.

### The prescribing substrate — added 2026-09-06 after reading what governs the plists

`ai/scripts/lifecycle/local-agent-os/README.md:216-224` is where these plists come from. It already anticipates a version of this defect:

> *"Run this guide from the Brain checkout. Both plists invoke `ai/daemons/**` RELATIVE to this root, and the Engine repo no longer carries an `ai/` tree — so an Engine clone here produces a plist that installs cleanly and never launches."*

Two things follow, and both make the ticket sharper rather than smaller:

**1. This machine is a worse case than the doc predicts.** The Engine-rooted plists did not "install cleanly and never launch" — they launched correctly for weeks, and only died when the split's `dotenv` removal reached that seat's `node_modules`. A failure the doc expects to be immediate and self-evident was instead silent and deferred by roughly three weeks. Any guard keyed on "an Engine root never starts" would have passed this machine every day until it didn't.

**2. The prescription does not close the class.** The install block resolves the root as:

```sh
export AGENTOS_RUNTIME_ROOT="$(pwd -P)"
```

There is no *the* Brain checkout. This machine has **eight**, several on feature branches. Running that line from a seat's Brain tree produces a seat-coupled `agentosRuntimeRoot` with the identical prune vector — and it would be *harder* to diagnose than what happened here, because the tree would look correct: right repo, right file, right relative paths. The root cause is not "an Engine clone was used"; it is that the root is allowed to be a tree someone else edits.

Structure-map gate: the diagnostic in AC-3 lands in `ai/scripts/diagnostics/` (siblings: `lmStudioEmbeddingInstances.mjs`, `defectObservations.mjs`, `check-retired-primitives.mjs`). Host layout plus the two LaunchAgent plists otherwise; owning folder for daemon-side work is `ai/daemons/orchestrator/`.

## The Fix

A machine-level daemon must not resolve its dependencies through a tree any seat can mutate.

1. **A seat-neutral runtime root.** A Brain checkout owned by no seat, pinned to `dev`, with its own real `node_modules`. Both LaunchAgents point there.
2. **Secrets stay single-homed.** `DOTENV_CONFIG_PATH` in the plist's `EnvironmentVariables` addresses the existing `.env` in place. No credential is copied to a second location.
3. **Never a symlinked `node_modules` under a runtime root.** A root whose dependency closure is another tree's is not pinned.
4. **The failure must be loud** — **now #337.** 4,648 silent crashes over ~42 h reached no surface: no healthcheck field, no A2A note, no digest row. Split out because it is a different substrate from root soundness and deserves its own reviewer.

### Interim state already applied on this machine (2026-09-06, operator-approved)

Not a substitute for the ACs — recorded so the next reader knows what they are looking at.

- Cloned `neomjs/neo-agent-brain@dev` to `/Users/Shared/agent-os/neo-agent-brain`, `npm ci`.
- `WorkingDirectory` → that path; added `DOTENV_CONFIG_PATH` → the existing `.env`. Prior plist preserved as `.before-grace-neutral-root-20260906`.
- `bootout` + `bootstrap`. Result: `state = running`, `runs = 1`, `last exit code = (never exited)`, and `[ProcessSupervisor] lms server (LM Studio CLI) readiness hook` active again.
- **`com.neomjs.agent-os-wake` was deliberately left on the doomed root** — it is currently working, and swapping a live wake receiver mid-day is not warranted by this ticket. That latent half is AC-2.

### Evidence

Two-arm probe, same entrypoint, same contradictory-role guard, only the tree differs:

| Arm | Result |
|---|---|
| canonical Brain tree | resolves `dotenv` + `hostEdgeProfile.mjs`, reaches the role guard, refuses cleanly |
| pinned 08-13 tree | `ERR_MODULE_NOT_FOUND: dotenv` |

`DOTENV_CONFIG_PATH` verified with a negative control: set → four expected keys present; unset → absent.

**Falsified along the way, recorded so nobody re-derives it:** I claimed LM Studio's `"networkInterface": "127.0.0.1"` made it unreachable from the containers. Probing from inside `mc-server` and `orchestrator` returned HTTP 200 with all four models. Loopback is fine; `host.docker.internal` reaches it. Retracted.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `com.neomjs.agent-os-host-edge` `WorkingDirectory` | this ticket | seat-neutral Brain root | prior plist backup | local Agent OS README | `launchctl print` runs/exit-code |
| `com.neomjs.agent-os-wake` `WorkingDirectory` | this ticket | same root | leave as-is until AC-2 | same | resolution walk table |
| `DOTENV_CONFIG_PATH` (plist env) | `dotenv` v17 `dotenv/config` | address `.env` in place | cwd `.env` | same | two-arm probe above |
| consecutive-exit observability | **moved to #337** | — | — | — | 4,648 unsurfaced crashes |

## Decision Record impact

`aligned-with ADR 0040` (§2.5 *Two root authorities, never one*).

**Corrected 2026-09-06T10:1xZ — this line originally read `none`, "no ADR asserts the runtime root's ownership". That was wrong**, and it is left visible rather than silently swapped. ADR 0040 §2.5 names the authority directly:

> **`agentosRuntimeRoot`** — where the Agent OS itself is installed and runs.
> **`targetRepoRoot`** — the checkout the Agent OS operates ON.

I found it only when I opened `ai/scripts/lifecycle/local-agent-os/README.md:217` — the block that installs these very plists cites the ADR in its first line. I had read the plists, the launchd state, and the daemon source, but not the substrate that *prescribes* them. This work restores §2.5's separation on a machine that had drifted from it; it does not amend or challenge the ADR.

## Acceptance Criteria

- [ ] `ai/scripts/lifecycle/local-agent-os/README.md` prescribes an `agentosRuntimeRoot` that no agent seat owns. `AGENTOS_RUNTIME_ROOT="$(pwd -P)"` from "the Brain checkout" is insufficient — with eight Brain checkouts on this machine it still permits a seat-coupled root, and the reviewer must be able to point at the line that now forbids it.
- [ ] A seat-neutral runtime root exists with a real (non-symlinked) `node_modules` and a documented refresh procedure. The procedure states that the root is load-bearing for a running daemon, so it is not a place to do development.
- [ ] **Both** LaunchAgents resolve every dependency inside that root. Verified by executing each entrypoint from its configured `WorkingDirectory` — resolution reached, not merely "the file exists".
- [ ] A diagnostic REDs on an unsound runtime root — a root inside a seat tree, **or** a symlinked `node_modules`. It must fail on the actual 08-13 root as a specimen; a check that cannot go red on the configuration that caused this outage does not close the AC. A doc alone repeats the failure — the doc claiming a guard is not a guard.
- [ ] Secrets are referenced, not duplicated: exactly one `.env` on the machine backs both agents.
- [ ] ~~A host daemon that has exited N consecutive times is observable from inside the plane.~~ **Split to #337** during implementation — "the root owns its closure" and "a dead daemon is visible" are different substrates, and bundling them made this ticket not one-PR-resolvable. My over-scoping at filing time; recorded rather than quietly dropped.
- [ ] Reboot receipt: cold boot → both agents running, LM Studio server up, both models resident, `query_raw_memories` answering. Post-merge-only.

## Out of Scope

- Deleting the host-edge LaunchAgent — that is #84's cutover.
- Moving `.env` out of the seat tree (#244).
- The machine daemon running under a seat's `NEO_AGENT_IDENTITY` — noted, not fixed here.
- LM Studio's `autoStartOnLaunch`. Deliberate, on operator direction: our own logic should own inference lifecycle rather than our guides asking a small macOS+LM-Studio audience to change OS-level settings.

## Avoided Traps

- **`npm install dotenv` into the seat's `node_modules`.** Green in a minute; pruned again by the next `npm ci`, and it resurrects a pre-split tree.
- **Repointing at another seat's Brain checkout.** Eight exist, several on feature branches; a rebase would swap the machine's daemon out from under it. Same class as the original bug.
- **Copying `.env` next to the neutral clone.** A second credential home to keep in sync and to leak. `DOTENV_CONFIG_PATH` costs one plist key.
- **Reading "the plist is broken".** It parses, launchd loads it, the `WorkingDirectory` and script both exist. Only the dependency closure is gone — which is why 4,648 crashes never looked like a config error.

## Related

- #84 (host-edge is inside its eventual deletion scope; this is the interim repair, currently blocked there by `neomjs/neo#16180`)
- #90 · #253 (local runtime parity / Brain-built images)
- #244 (seat credential routing in an untracked dotfile — the `.env` residual)
- #337 (split from this ticket: a crash-looping host daemon reaches no surface)
- #54 (an external plane cannot recover itself — same theme, different measured symptoms)
- Sibling filed from the same session: the LM Studio supervisor vs. its own update window.

Origin Session ID: 70502f9a-5b14-4dcf-bcdf-4a29b546df77

Retrieval Hint: `query_raw_memories` — "host-edge crash loop dotenv pinned runtime root seat node_modules symlink"; `launchd.err.log` `ERR_MODULE_NOT_FOUND` at `agent-os-runtime/03035d1b`.



## Timeline

- 2026-09-06T10:05:09Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-06T10:05:10Z @neo-opus-grace added the `bug` label
- 2026-09-06T10:05:10Z @neo-opus-grace added the `ai` label
- 2026-09-06T10:05:10Z @neo-opus-grace added the `architecture` label
- 2026-09-06T10:05:10Z @neo-opus-grace added the `agent-os` label
- 2026-09-06T10:06:09Z @neo-opus-grace cross-referenced by #336
- 2026-09-06T10:20:05Z @neo-opus-grace cross-referenced by #337
- 2026-09-06T10:24:29Z @neo-opus-grace cross-referenced by PR #338
- 2026-09-06T10:32:12Z @neo-opus-grace referenced in commit `af21dca` - "docs(agent-os): a chained bootout+bootstrap leaves the agent stopped (#335)

`bootout` returns before launchd releases the label, so an immediately chained
`bootstrap` fails 5: Input/output error. The `bootout` half has already
succeeded, so the agent is left stopped rather than restarted — for the wake
receiver that is a silent wake-delivery outage on port 3199.

Measured on this host: the repoint command in this PR's first revision did
exactly that. The host-edge repoint earlier in the same session had a settle
delay between the two calls; it was dropped when the command was transcribed
for someone else to run, which is the asymmetry a guide exists to remove.

Adds the settle delay, a verification line, and the repair (run bootstrap again
once the label is gone) next to the refresh procedure."
- 2026-09-06T11:44:50Z @neo-opus-grace referenced in commit `9bf9a71` - "docs(agent-os): a machine can need more than one runtime root (#335)

ADR 0040 §2.5 names `agentosRuntimeRoot` in the singular, which reads as one per
machine. §2.3's cross-repository whitebox tier needs a second, separate one: an
externally provisioned Agent OS runtime for Engine-owned Neural Link e2e. This
repository has no e2e tier at all — only unit, integration and
integration-parity — so that run originates in the Engine repo and reaches for
a Brain root from outside.

The two roots differ only by intent, and the supervised one is the more
discoverable: it is the path an operator hands you. Pointing a whitebox run at
it means npm ci or a branch switch under two live daemons, which is this
ticket's own mechanism with a person's hand on it instead of a three-week
delay.

Written after a near-miss today: a maintainer was correctly following the ADR,
had been pointed at the supervised root, and nothing in this guide said a second
root should exist. The omission was mine — the 'installed software, not a
workspace' rule landed in this same section one commit earlier and still did not
say where the other root goes."
- 2026-09-06T11:51:05Z @neo-opus-grace referenced in commit `edddbb8` - "chore(agent-os): drop the unwired runtime-root diagnostic (#335)

Operator: unwired diagnostics are close to a hard no in this repo. The team
produced enough never-run diagnostic scripts that a client demanded the repo
split and then refused to use it. Measured before acting: 20 of 34 files in
ai/scripts/diagnostics are referenced by nothing — not package.json, CI, hooks,
daemons, services or src. 59 percent dead. check-runtime-root.mjs registered
one reference, and that reference was its own npm alias.

It also could not prevent the failure it was written for. A pruned dotenv kills
hostEdge.mjs before any of its code runs, so the check can only ever observe a
root that is already fine, or one nobody is asking about. It was a one-off
audit I ran by hand, wearing a guard's costume; its findings are already in this
guide and in the repaired host.

What survives is the half that needs no runtime surface: the seat-neutral root
prescription, the two-root distinction for ADR 0040 §2.3 provisioning, the
bootout/bootstrap restart trap, and now two literal shell commands in place of
the npm alias. The second command is the one that matters — it resolves both
daemon entrypoints, which is the axis a directory listing cannot see, and both
are import-safe because they gate boot on argv[1]. Verified against the live
root: node_modules real, both entrypoints resolve.

Net for the repo: minus two files, minus one package script, minus one CI smoke
entry."
- 2026-09-06T12:01:27Z @neo-opus-grace referenced in commit `e21eedd` - "docs(agent-os): the guide stops naming one deployment's paths (#335)

Operator: "agent os => dockerized => KB and MC for OTHER teams, ingesting OTHER
repos." This repository is a multi-tenant product and our machine is one tenant,
so an operator guide must not hand the next deployment our directory layout to
edit out.

My additions carried seven occurrences of this host's literal install path.
Replaced with a placeholder the reader sets once, plus the variable the section
already exports. Nothing about the procedure depends on the location — only on
who can write to it — and saying that outright is more useful than an example
path that is right on exactly one machine.

Also fixes an ordering defect I introduced: the verification block referenced
AGENTOS_RUNTIME_ROOT, which this document does not export until eighteen lines
later. It now uses the variable defined above it."
### @neo-opus-grace - 2026-09-06T12:06:08Z

Closing NOT_PLANNED. The outage was real and **the host is already repaired** — both LaunchAgents run from a seat-neutral root, inference is back, wake serves `:3199`. That fix needed no ticket and no merge.

What this ticket additionally asked for was documentation and a diagnostic for the local host-edge procedure. Two open epics forbid both:

- **#86**: *"starting earlier would re-describe a surface we are about to delete"* — #84 hard-cuts this plane, #90 replaces it with the container topology.
- **#191**: *"A retained script or command must have a repeated current use case, a real consumer, and a domain owner. Historical existence and test volume are not owners."*

I filed this at 10:0x without reading either. `ticket-create` §0 — *understand the intent, does it fit the current architecture and goals* — is the first gate in that skill and I started at §1a.

**Preserved on #84/#90 rather than here:** a runtime root must own its dependency closure (no `node_modules` symlink into a tree any seat can reinstall — that is what killed this plane for 42 h), and `bootout` chained to `bootstrap` leaves the agent stopped rather than restarted. Both matter to whoever re-provisions the plane; neither needs a ticket of its own.

PR #338 closed alongside.


- 2026-09-06T12:06:09Z @neo-opus-grace closed this issue
- 2026-09-06T12:06:54Z @neo-opus-grace cross-referenced by #84

