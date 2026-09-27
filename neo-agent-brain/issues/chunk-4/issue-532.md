---
id: 532
title: 'The wake-envelope plant has no provisioning path, so a merged writer fix reaches no seat'
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-preview
createdAt: '2026-09-26T07:22:10Z'
updatedAt: '2026-09-27T10:38:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/532'
author: neo-preview
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
closedAt: '2026-09-27T10:38:22Z'
---
# The wake-envelope plant has no provisioning path, so a merged writer fix reaches no seat

## Context

`ai/services/fleet/opencodeWakeEnvelopePlugin.mjs` is the OpenCode seat's wake-envelope writer. It lives in this repo, and it is what makes a seat's wake route work at all. **Nothing copies it to a seat.** There is no provisioning path, no installer, no generator arm, no postinstall step.

Measured, on `dev` at `7d7de1c`:

```
$ grep -rn "opencodeWakeEnvelopePlugin\|neo-wake-envelope" ai/ scripts/ .github package.json
ai/daemons/wake/daemon.mjs:1145: * 2. `ai/services/fleet/opencodeWakeEnvelopePlugin.mjs` — the event-driven writer for
```

**One hit, and it is a JSDoc line.** The file is never installed by anything in this repository. The live installation is a manual per-seat copy into `~/.config/opencode/plugins/neo-wake-envelope.mjs`, which no ticket, script, or runbook automates.

Observed consequence, measured on this seat (`@neo-preview`) across 369 digests since 2026-08-23T15:42Z: **132 delivered / 237 failed**, with the two failure classes interleaved all the way through the window — 10 consecutive deliveries between 20:38Z and 22:29Z on 2026-09-25, then a failure at 22:56:55Z. The oscillation is the signature of a hand-managed artifact: the plant was fixed by hand, delivery recovered, and the next seat event stripped the field again.

## The Problem

A substrate fix to this writer cannot reach a single live seat without a human copying a file by hand. Every future fix to this file inherits the same manual step, and **the failure mode of a missed copy is silent** — no seat reports "I am not provisioned"; a seat with an old plant simply stops being delivered to, and the counter that shows it (`consecutiveFailures` on the receiver record) is read by nobody.

That is the same shape as the B4-guard finding in #508: an instrument that reads as coverage without being coverage. Here the gap is not in a guard's detection logic but in the last mile between a merged fix and the process that runs it.

This is also the operational half of #503. #503's two halves were the *contract* (the reader's field list, landed as #510) and the *writer* (landed as #529). Both are merged or approved. Neither reaches a seat by itself.

## The Architectural Reality

`generateOpenCodeSeatConfig.mjs` is the seat-provisioning entry point, and it already returns a `files` array of `{path, content}` — the boot hook `write-wake-envelope.mjs` is emitted through it, and #529's own parity spec consumes that array to run the boot hook's real output through the real reader (`opencodeSeatEnvelopeParity.spec.mjs`, the arm "the boot hook's envelope is admitted by the same reader"). So the seam that would carry the plant is already exercised and already proven by a test.

The plant is a *plugin* rather than a generated script, so the target path differs (`~/.config/opencode/plugins/neo-wake-envelope.mjs` vs. the config directory the generator already writes), and OpenCode loads it at process start — which means an install is only observable after a seat restart, and that boundary must be stated rather than glossed.

## The Fix

Emit the plant through the seat-config generator, beside the boot hook it already emits, into the seat's OpenCode plugins directory. The generator is the only surface that already knows a seat's paths; adding a second installer for one file in the same seat would be the accretion this repo's maintainer test refuses.

**Two of the three questions this ticket originally asked are already answered by `deriveHarnessLaunchSpec.mjs`, and finding that narrowed the scope.** The harness launch spec sets a two-var XDG pair per seat — `XDG_CONFIG_HOME` and `XDG_DATA_HOME` both at the instance home, `XDG_CACHE_HOME` under it — and its own comment records that this "unifies the whole footprint as `<instanceHome>/opencode/` (**the seat-config generator's planting target**)". So the path rule already has exactly one owner and the generator already has a target to plant into. The genuinely missing half is that **nothing writes there**: the only mentions of `~/.config/opencode/plugins/` in this repo are the plant's own JSDoc and one comment.

Decide at the PR, with the generator's sibling precedent read first:

- **Arm shape** — one more entry in the returned `files` array, or a separate plant-specific return. The first is the smaller change; the second is easier to consume for a caller that only wants the plant.
- **Overwrite semantics** — an operator hand-edit of the installed plant must be detected and reported rather than silently clobbered, or silently clobbered with a log line. Pick one deliberately; the current state (no installer, hand edits persist) is not a design, it is an accident.
- **Freshness** — whether the generator rewrites the plant unconditionally, or only when the repo copy's content differs from the installed copy. The second makes a relaunch cheap and makes "the seat is running an old plant" answerable by diff.

### The second, smaller defect: the fallback is honest nowhere

`opencodeWakeEnvelopePlugin.mjs:87` derives its root as `process.env.XDG_DATA_HOME || path.join(os.homedir(), '.local', 'share')`, while its own JSDoc at :54 claims "The envelope root honors `XDG_DATA_HOME`, so per-seat XDG isolation (Fleet launch specs) keeps each seat's envelope on its own path instead of collapsing onto the shared default."

Under the harness that claim holds. **Under a hand launch it does not**, and a hand launch is a normal way to start a seat: `open -n -a <harness> --args --user-data-dir=…` sets that harness's own flag and leaves `XDG_DATA_HOME` unset, so the fallback puts the envelope at a root shared by every seat on the host, while the subscription's registered `envelopePath` points at the per-seat instance home. The reader then refuses or — worse — a *different* seat's bridge answers for this one.

Decide at the PR: whether the unset case **fails loudly** (log and skip the write, so the absence is visible) or keeps writing to a shared root **with the JSDoc corrected** to stop claiming isolation it does not provide. The second is a one-word doc change and leaves a silent cross-seat hazard; the first is safer and makes a mis-provisioned seat self-describing. I lean first, and the argument for the second is that a strict refusal would break every currently-working hand launch — which is an argument about *this week*, not about correctness.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `generateOpenCodeSeatConfig` return value (`files`) | `ai/services/fleet/generateOpenCodeSeatConfig.mjs` | gains a plant entry at the seat's plugins path | none — the boot hook arm is unchanged | generator JSDoc names the plant as a second emitted artifact | red-first spec: a generated seat has the plant at the path OpenCode loads; the parity spec's real-output discipline extends to it |
| Installed plant content | `ai/services/fleet/opencodeWakeEnvelopePlugin.mjs` | byte-equivalent to the repo copy after generation | operator hand-edit is reported, never silently lost | the plant's own JSDoc states it is generated | a drift arm: generate twice, assert the installed file matches the repo source |
| Seat wake route | `ai/daemons/wake/localWakeAdapters.mjs` reader | unchanged — the reader already admits an envelope this plant writes | unchanged | — | post-merge L3: a freshly provisioned seat records `delivered` on its first digest |

## Decision Record impact

`aligned-with` ADR 0034 §2.3's secret-boundary discipline: the plant carries no secret (it reads the identity from the launch env, never a credential), and the PAT/password handling this generator's sibling surfaces carry is untouched. No ADR amendment is proposed; if the arm shape turns out to change what the generator *is* (a config generator vs. a seat provisioner), that is a naming/authority question for a follow-up, not a silent consequence of this ticket.

## Acceptance Criteria

- [ ] `generateOpenCodeSeatConfig` emits the wake-envelope plant into the seat's OpenCode plugins directory, and the emitted file is byte-equivalent to `ai/services/fleet/opencodeWakeEnvelopePlugin.mjs`
- [ ] Red-first: the spec fails on `dev` before the change (a generated seat has no plant installed today) — measured, not asserted
- [ ] The parity spec's real-output discipline covers the generated plant: the generated file, run through the real reader and owner guard, is admitted for its own seat
- [ ] Drift is detectable: generating twice yields an identical installed file, and a hand-edited install is reported rather than silently clobbered or silently lost
- [ ] The generator's JSDoc names the plant as a second emitted artifact, so the "two producers, one contract" statement at `ai/daemons/wake/daemon.mjs:1145` has a real installer behind it
- [ ] `npm run ai:structure-map -- --files --loc` run at intake; the placement is cited or recorded N/A
- [ ] Post-merge L3: a freshly provisioned seat records `delivered` on its first digest without a manual copy. Residual-Owner: an open ticket that is **not** this one.

## Out of Scope

- **Changing what the plant writes.** #529 owns the writer's content. This ticket is distribution only; a content change here would re-litigate a reviewed fix.
- **The `kimi-pull-bridge` adapter's plant**, if one exists. Different envelope, different owner process, different ticket.
- **Retro-fitting existing seats.** Provisioning forward is the deliverable; a fleet-wide backfill of already-installed plants is the operator's call, not this ticket's AC.
- **The reader's field list.** Landed in #510/#529.

## Avoided Traps

- **A standalone installer script** (`install-wake-plant.mjs`). Rejected: it is a second thing that knows a seat's paths, it will drift from the generator, and it re-introduces exactly the "a check that reads as coverage without being coverage" shape (#508) one layer down.
- **A postinstall hook in `package.json`.** Rejected: the Brain's install runs in containers and on CI, where there is no seat to provision. The generator is seat-scoped by construction; a postinstall is not.
- **Documenting the manual copy in a runbook instead.** Rejected as the *whole* fix — it is the right addition to the docs once the generator arms, but a runbook is not a mechanism, and the failure it addresses is silent.

## Related

- #503 — the parent: a wake subscription reports itself deliverable while every dispatch fails
- #528 / #529 — the writer fix this ticket distributes (approved at `7d7de1c`)
- #510 — the reader's delivery projection; the health surface that made the gap visible
- #513, #514 — the two dark seats whose envelopes sat on the pre-identity schema
- #508 — the sibling false-assurance finding (a guard that reads as coverage without being coverage)
- #522 — this seat's fail-closed gates; a different provisioning gap, same family

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-09-26T07:2xZ; no equivalent found. A2A in-flight sweep: `@neo-gpt` holds a live claim on the #510 served-delivery placement follow-up (Brain #530, 07:19:44Z) — adjacent but distinct, that one is about where the delivery read is *served*, this one is how the writer *reaches* a seat. Memory sweep: no prior decision on plant provisioning found; the nearest prior art is the B4-guard gap in #508, which is the same failure shape on a different surface.

Origin Session ID: session-2026-09-25-neo-preview-brain-prs

Retrieval Hint: "wake envelope plant provisioning generator files array seat plugins path" · Commit anchor `7d7de1c` (the approved writer fix whose Post-Merge Validation step 1 is the manual copy this ticket automates).



## Timeline

- 2026-09-26T07:22:12Z @neo-preview added the `enhancement` label
- 2026-09-26T07:22:12Z @neo-preview added the `ai` label
- 2026-09-26T07:22:12Z @neo-preview added the `architecture` label
- 2026-09-26T07:22:12Z @neo-preview added the `agent-os` label
- 2026-09-26T07:22:15Z @neo-preview assigned to @neo-preview
- 2026-09-26T07:24:30Z @neo-preview cross-referenced by PR #529
- 2026-09-26T13:42:21Z @neo-preview cross-referenced by PR #548
- 2026-09-26T13:48:22Z @neo-preview cross-referenced by #549
- 2026-09-26T18:45:54Z @neo-opus-grace cross-referenced by #550
- 2026-09-26T19:51:25Z @neo-preview cross-referenced by PR #551
- 2026-09-26T21:21:33Z @neo-preview cross-referenced by #561
- 2026-09-26T22:05:35Z @neo-opus-ada cross-referenced by #562
- 2026-09-27T08:55:05Z @neo-preview referenced in commit `8d7bc56` - "feat(fleet): the generated wake hook installs the wake-envelope plant (#532)

A provisioned seat published its wake envelope once per hand-copied file and never
again. Nothing in provisioning placed the OpenCode plugin that publishes the envelope on
each new session, so the "two producers, one contract" statement at
`ai/daemons/wake/daemon.mjs` had a runbook step behind it rather than an installer.

The plant is read from disk once at module load and inlined into the generated hook as
base64. Base64 rather than a template literal because the plant is ~19 KB of JavaScript
and embedding source in a generated string literal invites escaping that would break the
byte-equality contract silently. Inlined rather than copied at launch because a hook that
read the plant from the Brain checkout would make a seat's wake lane depend on a
repository the seat may not have.

The install is report-and-skip throughout, which is the part worth arguing for:

- A differing install is reported with both byte counts and left alone. A generator that
  silently overwrote — or deleted — a file it did not place would be a worse failure than
  the drift it exists to surface, and a hand-edited install is a person's decision.
- A SECOND wake-envelope plant in the same directory is named rather than resolved. This
  is not hypothetical: the seat this was measured on carries `neo-wake-envelope.mjs`
  beside an older `neo-wake-envelope.js` from August, and OpenCode loads every plugin it
  finds with nothing recording which one won. The hook reports the ambiguity; deciding
  which file is authoritative is not its call.
- An unset `XDG_CONFIG_HOME` warns loudly rather than skipping quietly, because the
  shared-root fallback would resolve outside the seat and a silent skip is
  indistinguishable from a healthy install.

Measured, not asserted: a freshly generated seat installs a file byte-identical to
`ai/services/fleet/opencodeWakeEnvelopePlugin.mjs` (sha256 equal), a second run reports
"already current", a hand-edited install is reported and its bytes survive, a sibling
plant is reported and left in place, and the unset case warns.

The spec gains four arms that EXECUTE the generated hook in a temp seat home rather than
inspecting the emitted string. A hook can contain every correct instruction and still
install nothing; the existing arms only ever read the emitted text.

Noted, not changed: `ai/scripts/lifecycle/hooks/kimi-code/wakeEnvelopeHook.mjs` is the
Kimi-harness sibling, and the plant is an OpenCode plugin so it does not apply there. Two
asymmetries are real — the Kimi hook is a module while this one is a string literal inside
the generator, and `normalizeAgentIdentity` is duplicated inline across both — but both
belong to a cross-harness question this ticket does not own."
- 2026-09-27T08:55:05Z @neo-preview referenced in commit `a7dd7d2` - "feat(fleet): the wake-envelope plant is a sibling artifact, not a hook payload (#532)

Addresses the round-1 review on #548. The premise is unchanged — a seat's
wake lane needs the plant installed, and the generator is the one surface
that knows a seat's artifact paths. The delivery path was wrong.

RA-1: the plant is now its own `files` entry at a caller-resolved
`wakePlantPath` (<instanceHome>/opencode/plugins/), converged by
prepareOpenCodeArtifacts under its OWN policy row, written before spawn.
The previous shape installed it from inside the boot hook behind an
`XDG_CONFIG_HOME` guard — but the hook runs after the server is listening,
so that install could only ever be loaded by a LATER process, and
`bootstrapOpenCodeWakeRoute` does not pass `XDG_CONFIG_HOME` at all, so on
a real Fleet seat the branch always skipped. Emitted hook: 31,695 ->
3,362 bytes (27.7 KB less substrate in every seat's home).

RA-2: freshness is the existing whole-file posture, now pointed at the
right artifact. A plant update reports divergence on the PLANT, so an
ordinary plant bump no longer reads as corruption of the boot hook.

RA-3: generateOpenCodeSeatConfig.spec.mjs is on the brain-unit.yml run
list for the first time. New arms replace the obsolete install arms: the
plant is emitted byte-equal at the caller-resolved path, the hook carries
no base64 payload, and the caller's real env shape (neither XDG var set)
still lands the plant. Header no longer claims to be fs/env-free.

RA-4: the purity contract now declares its one file read instead of
claiming "no fs access"; renderWakeHook's JSDoc is re-attached to the
function (the new constants had orphaned it).

RA-5: the plant no longer falls back to ~/.local/share when XDG_DATA_HOME
is unset — it refuses. The fallback resolved to the one path that is not
per-seat, and one process can instantiate the plant for several project
directories at once, so the fallback made the route fail by COLLISION
rather than by absence. Arm added; it points HOME at a scratch dir so the
assertion cannot read the developer's real home.

Golden digest bumped, not relaxed, per the spec's own convention: the sole
differing artifact is the boot hook, verified against origin/dev at
bea77518 before rebasing so the bump is attributable to this change and
not to inherited drift.

Measured: 46/46 in the two touched specs, 43/43 in the composer spec.
Four provisioningTemplates arms are red on origin/dev itself (reproduced in
a clean worktree) and receiver.spec:438 is a flake (3/3 in isolation);
neither is this change. Note for the evidence record: neither spec is on
the brain-unit run list, so those receipts are local, not CI."
- 2026-09-27T08:59:21Z @neo-preview referenced in commit `25b2400` - "test(fleet): the digest bump cites behavior, not the ticket, in durable source (#532)

check-ticket-archaeology reds a ticket reference in a durable source comment: provenance belongs in the PR body and the commit subject, not in a comment that has to stay true as the code moves. Same rule my own seat memory records, and I broke it in the very commit whose message says the guard is the gate.

Verified with the published guard locally (neo-agent-skills@0.1.19): 0 violations, exit 0 clean; negative control re-inserting the reference reproduces CI's exact message at exit 1, so the check gates rather than reports."
- 2026-09-27T08:59:56Z @neo-preview referenced in commit `23dc127` - "test(fleet): the digest bump cites behavior, not the ticket, in durable source (#532)

check-ticket-archaeology reds a ticket reference in a durable source comment: provenance belongs in the PR body and the commit subject, not in a comment that has to stay true as the code moves. Same rule my own seat memory records, and I broke it in the very commit whose message says the guard is the gate.

Verified with the published guard locally (neo-agent-skills@0.1.19): 0 violations, exit 0 clean; negative control re-inserting the reference reproduces CI's exact message at exit 1, so the check gates rather than reports."
- 2026-09-27T10:23:21Z @neo-preview referenced in commit `ef6d388` - "feat(fleet): an earlier generation of the plant is replaced, a hand edit is refused (#532)

Discharges the Round-2 review's STILL_OPEN action. The Fleet's general text-artifact
rule is create-or-refuse, so the plant was UNREPLACEABLE: the first plant edit after
provisioning is refused on every seat, because the file on disk is an earlier
generation and a bare byte-compare cannot tell that from a hand edit.

The discriminator is provenance, and a bare marker cannot carry it — anyone can keep
the word and edit underneath it. So the emitted plant carries a marker holding a
sha256 of the body it precedes, which makes "unmodified generation" checkable rather
than promised. Both halves must hold: a file with no marker is a person's, and so is
one whose marker disagrees with its own body.

The plant is withheld from the generic text-artifact pass rather than given a policy
row it cannot satisfy, and converges through the same atomic publish the transport
receipt uses. The receipt distinguishes the two outcomes, which is the point: UPDATED
says the Fleet moved its own file forward, DIVERGENT says a person did.

Arms, red-first: an EARLIER GENERATION on disk is replaced and reported UPDATED;
a hand edit, and a marker kept over an edited body, are both refused and neither
clobbers what the person left. Red-first verified by forcing the provenance check
off, which reproduces the reviewer's exact refusal; green with it on. The birth
census arm now states what the marker changes — the emitted body equals the Brain
copy apart from the marker line, and the marker agrees with that body.

Measured: 23/23 generateOpenCodeSeatConfig, 44/44 prepareManagedAgentWorkspace,
1157 passed across fleet + wake. The 4 provisioningTemplates failures reproduce on a
clean origin/dev worktree and are not this change. The golden digest is unchanged:
PARAMS emits no wakePlantPath, so no hook byte moved."
- 2026-09-27T10:38:22Z @tobiu referenced in commit `d70e2fb` - "Merge pull request #548 from neomjs/agent/532-wake-plant-install

feat(fleet): the generated wake hook installs the wake-envelope plant (#532)"
- 2026-09-27T10:38:22Z @tobiu closed this issue

