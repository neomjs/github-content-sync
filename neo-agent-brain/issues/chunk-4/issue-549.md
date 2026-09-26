---
id: 549
title: 'Two wake-envelope plants on one seat: which is authoritative, and should the report be louder than a warning?'
state: OPEN
labels:
  - ai
assignees: []
createdAt: '2026-09-26T13:48:21Z'
updatedAt: '2026-09-26T20:38:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/549'
author: neo-preview
commentsCount: 4
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
---
# Two wake-envelope plants on one seat: which is authoritative, and should the report be louder than a warning?

## Problem

Found while implementing #532: the wake-envelope plant has no provisioning path, so it was installed by
hand. On the seat where that was measured, **two** plants sit in the same plugins directory:

```
neo-wake-envelope.js    16129B  Aug  1 19:00
neo-wake-envelope.mjs   19028B  Sep 26 09:20
```

Neither carries a `GENERATED` marker and they differ by ~3 KB. OpenCode loads every plugin it finds in
that directory, and nothing in the system records which one won or that they disagree.

The missing installer was the ticket's framing. This is the sharper half: hand-installing has **already
produced an ambiguous install**, and no check surfaces it. #548 makes the hook *report* sibling plants
rather than resolve them, deliberately — which file is authoritative is a maintainer decision.

## The decision this ticket needs

1. **Which file is authoritative** for a seat that has both — or should provisioning refuse to proceed
   until the ambiguity is reconciled?
2. **Should an ambiguous install block seat launch**, or warn and continue? Today it warns per launch.
   The cost of a warning nobody reads is that the next seat starts the same way.
3. **Is `.js` a legacy name with no loader**, or is it genuinely loaded? That determines whether the
   Aug 1 file is inert or actively shadowing the current plant. I could not determine this from the seat
   alone, and guessing is how the ambiguity got here.
4. **Should the generator ever reconcile rather than report?** #548 deliberately never clobbers or
   deletes. If the answer is "the generated artifact always wins", that is a defensible position and it
   is the opposite of what shipped — so it should be a decision on the record, not a default.

## Why this is a ticket and not a line in #532

Picking an authoritative file means choosing between a file I did not write and one I did, on a seat
whose loader behaviour I cannot observe from here. That is a maintainer call, and #532's scope is the
installer.

## AC

- [ ] It is established which of the two files the harness actually loads, with evidence rather than
      inference from file names.
- [ ] A maintainer decision is recorded for which file is authoritative, and what provisioning does when
      it finds an ambiguous install.
- [ ] A seat carrying two plants is brought to a single, recorded state.

Refs #532

## Timeline

- 2026-09-26T13:48:22Z @neo-preview added the `ai` label
- 2026-09-26T18:38:54Z @neo-opus-grace cross-referenced by PR #548
- 2026-09-26T18:45:54Z @neo-opus-grace cross-referenced by #550
### @neo-preview - 2026-09-26T19:32:58Z

## Resolved by measurement: the stamping plant is authoritative, and the collision is a silent-writer bug

This ticket was filed deliberately undecided — choosing which of two files wins is a maintainer call. A live incident on @neo-preview's seat decided it, so the decision no longer needs taste: it needs the receiver's own rule.

**What was measured, on the seat, from the receiver's records and the plant files themselves.**

The subscription (`WAKE_SUB:54aaef3c`, `SENT_TO_ME`, adapter `opencode-server`) had `routeDeliverable: true` and was healthy, which is why this looked like a harness problem and was not. Its delivery records split cleanly in time:

- through **12:28:54Z** — digests `delivered`
- from **18:23:41Z** onward — **every** digest `failed`, one reason, 12 consecutive: `opencode-server envelope requires 'agentIdentity'`

So the route was fine and the ENVELOPE was malformed. The live envelope confirmed it: actively written (mtime within minutes of the check) and **no `agentIdentity` key at all**.

**The two plants, and why the wrong one wins.** Both files were installed in `~/.config/opencode/plugins/`:

| file | dated | `agentIdentity` | behaviour on a missing identity |
|---|---|---|---|
| `neo-wake-envelope.js` | Aug 1 | **0 occurrences** | writes unconditionally |
| `neo-wake-envelope.mjs` | Sep 26 (post-#529 head) | 6 occurrences | **throws and leaves the envelope untouched** — by design, with the sentence "the envelope is left as it is — launch the seat through its wrapper" |

Both **export the same plugin name** (`NeoWakeEnvelope`) and derive the **same output path** from the data root. So the collision is not a precedence subtlety — the legacy plant simply writes a field-less envelope, and the correct plant then refuses to overwrite it, by its own guard. The malformed envelope is self-perpetuating: the good writer's only on-disk identity fallback reads the very file the bad writer poisoned.

**The rule, and why it settles the ticket:** the receiver compares the envelope's `agentIdentity` against the route owner and refuses the digest without it. A plant that cannot name the seat has no business writing the seat's envelope. So the authoritative plant is the one that **stamps `agentIdentity` and fails closed when it cannot** — not the newer file, and not the one that happens to run last. The legacy writer is not a valid alternative; it is the bug.

**The generalisable defect, which is the part worth keeping:** two writers, one output path, and **no uniqueness enforcement on the plant identity**. Nothing surfaced the collision — no warning, no log, no install-time check. The seat looked correctly configured for 33 days (Aug 1 → Sep 26) while carrying two writers, and the only symptom was a wake route that degraded silently at the moment the *other* plant was installed. That is the "louder than a warning" half of this ticket's title, answered by measurement rather than preference: a same-name, same-target second writer should be an install-time refusal, not a race.

Repair applied on the seat: the Aug-1 file moved to `neo-wake-envelope.2026-08-01.js.retired` (reversible, one `mv`, preserved as this ticket's evidence), leaving exactly one loadable plant. It takes effect on the next seat launch, which is also the only point at which a plant is published.

Relationship to the sibling fix: the durable repair is the generated hook installing exactly one plant, so the collision cannot be recreated by a second manual install. This ticket should close into that, and the install-time uniqueness check is the piece worth carrying even after both plants are gone — a second writer is otherwise only detectable by reading a delivery record after the fact.

Seat: @neo-preview, `NEO_AGENT_IDENTITY=neo-preview` in the launch env, `canonicalIdentity` mapping it to `@neo-preview`, which is the subscription's `agentIdentity`.


### @neo-preview - 2026-09-26T19:40:20Z

## Retraction: the two-plant collision is NOT the blocker, and my verdict above was wrong

I posted a conclusion here an hour ago that the evidence in this same ticket's history falsifies. Correcting it before anyone builds on it.

**What I claimed:** that the Aug-1 plant shadowed the stamping plant, that the field-less envelope was self-perpetuating, and that the receiver's rule therefore settles the "which is authoritative" question.

**What the delivery records show:** the Aug-1 plant is dated **2026-08-01**, so it was installed continuously through every *successful* delivery on record — the 10 consecutive deliveries on 2026-09-25 between 20:38Z and 22:29Z, and 24 deliveries on 2026-09-26 between 07:20:03Z and 12:28:54Z. Two plants coexisted across all of them, and the receiver refused nothing. So the collision cannot be what breaks the route, and "the stamping plant is authoritative because the receiver requires `agentIdentity`" does not follow — the receiver was content with both plants present, and it is the *field*, not the *file count*, that it checks.

I generalised from the single symptom I had in front of me without reading the success window in the same records, which is the same one-level-too-shallow failure I have now made repeatedly this session. Acting on it, I retired the Aug-1 file from the seat's plugin directory on the theory that it was the blocker. **That was wrong and it has been reverted** — the directory is back to its exact prior state, two loadable plants, nothing altered.

**What is actually consistent with the records, stated as a hypothesis and not a verdict:** the failure boundary is not a file boundary. Deliveries were clean through 12:28:54Z and every digest failed from 18:23:41Z — a boundary that coincides with a harness/session coming up, not with any change to the plugin files. That points at the ticket's *other* known failure class, `coordinates did not change after connection refusal`: the envelope carrying a port/sessionId the live session no longer honours. A wake that is delivered into a session which then cannot act on it is indistinguishable, from the receiver's side, from a wake that never arrived.

**So the questions this ticket opened are still open, and they are better questions than the ones I answered.** Specifically: is the authoritative-plant question real at all, or is it a distraction from a coordinates/authority problem that merely *presents* as an envelope problem? And the louder-than-a-warning half survives regardless and is now better evidenced: a same-name, same-target second writer is indistinguishable, at install time, from a correct configuration, and this seat carried one for 33 days. That deserves an install-time uniqueness check on the plant identity whether or not the collision turns out to be causal — which is the one durable thing I would keep from my retracted comment.

No repair action is warranted against the plant files on the strength of my retracted analysis. The open defect is the harness/session side, and it now also comes with a concrete observation from the same session: **the memory-core MCP tool surface is absent** (A2A send/receive, mailbox, memory save and subscription management all unavailable) while the knowledge-base MCP is connected and healthy. A session that cannot reach the memory core cannot act on a wake even when one lands, which is the most likely reason a delivered digest changed nothing observable.


### @neo-preview - 2026-09-26T20:04:25Z

## Amendment: the cadence is a new SESSION, and the surviving model is a last-writer race

My previous comment on this ticket retracted my verdict that the legacy plant was the blocker. That retraction **stands on the causation question** — the Aug-1 file was installed continuously through the healthy windows, so its presence never prevented a delivery. But I retracted more than I should have: the two-plant collision is not thereby irrelevant, and the evidence now says it is a **last-writer race**, which is a mechanism, not a coincidence.

### What was measured, 2026-09-26 evening

**The two writers, in full.** Both live in `~/.config/opencode/plugins/`, and `opencode.jsonc` there is 50 bytes containing only `$schema` — so there is no plugin declaration and the directory is the discovery mechanism. There is no bundled writer in `OpenCode.app/Contents/Resources`, and no Fleet-generated `write-wake-envelope.mjs` anywhere on the host, so on this seat the two plugin files are the *only* candidate writers.

| file | dated | `agentIdentity` | envelope shape |
|---|---|---|---|
| `neo-wake-envelope.js` | Aug 1 | **0 occurrences** | 8 fields: hostname, port, sessionId, projectId, directory, username, password, updatedAt |
| `neo-wake-envelope.mjs` | the #529 head | 6 occurrences | the same 8 **plus `agentIdentity`**, and it THROWS rather than write without one |

Both `export const NeoWakeEnvelope` — the same plugin name — and both derive the same path from the data root.

**The envelope on disk is byte-for-byte the legacy 8-field shape**, freshly written, and the receiver refuses exactly that: `opencode-server envelope requires 'agentIdentity'`, 14 consecutive attempts, the last at 19:38:14.512Z — 1.3 s after the envelope's own `updatedAt` of 19:38:13.187Z. So the causation is pinned to the field, not to the route, the subscription, or the coordinates.

**And the legacy file was present through the healthy windows** — 10 consecutive deliveries on 2026-09-25 (20:38-22:29Z) and 24 on 2026-09-26 (07:20:03Z-12:28:54Z). Which is why this is a race rather than a blocker: whichever writer lands **last** decides the envelope, and a race is the only model that explains an oscillation that no deterministic account ever could — the artifact is fine, then is not, with no change to any file.

### The cadence, which is the part that was never banked

The plant publishes from `session.created` / restore events only. Measured with the legacy file removed and one plant left: **`updatedAt` stayed frozen at 19:38:13Z across 120 s of a live, actively-updating session.** No amount of collision-removal republishes within a session whose create event has already passed.

**So a repair lands in the NEXT session — a restart is neither necessary nor sufficient**, and a session *restore* reuses the already-loaded plugin set, which is why "restart the app" was tried and did nothing. A previous seat told the operator this; it was not written down, and my own seat note carried the contradictory claim "one seat restart makes it permanent" five lines above the correct nuance. The tell was the self-contradiction, not the missing line.

### What this settles, and what it does not

It settles the authority question this ticket opened, on the receiver's rule rather than on taste: **the stamping plant is the only valid writer**, because a writer that cannot name the seat has no business writing that seat's envelope, and the correct plant already fails closed rather than emit a nameless one. Single-writer enforcement is the fix; the race only decides how often the broken output happens to win.

It does **not** fix anything by itself. Two durable pieces remain and both are already tracked: **#532 + PR #548** (a merged writer fix reaches no seat — no provisioning path, so this hand-copied plant is unverifiable drift the moment it exists), and **#550/#551 merged 19:57:14Z** (the Fleet hook env dropped `NEO_AGENT_IDENTITY` and the hook threw before reaching the plant section — necessary, and still not sufficient, since `XDG_CONFIG_HOME` remains absent from `hookEnv`).

The generalisable half, unchanged from my retracted comment and better evidenced for having survived it: **a same-name, same-target second writer is indistinguishable, at install time, from a correct configuration.** This seat carried one for 33 days. An install-time uniqueness check on the plant identity is worth having even after both plants are gone, and a last-writer verification (does the envelope I just published name me?) is the cheap runtime guard that would have surfaced this as an error instead of a silent 100%-refusal route.


### @neo-preview - 2026-09-26T20:38:33Z

## Resolved: there were THREE writers across TWO discovery roots, and the decisive one was invisible to git

My two previous comments on this ticket were both partly wrong, and the second one over-corrected. The measured inventory, and the reason my earlier counts kept coming out wrong.

### The writer

`/Users/Shared/agents/neo-preview/neomjs/neo/.opencode/plugins/neo-wake-envelope.js` — **16129 bytes, zero occurrences of `agentIdentity`,** and its envelope literal is character-for-character the 8-field shape that was being written to disk and refused by the receiver (`opencode-server envelope requires 'agentIdentity'`, 14 consecutive attempts, the last at 19:38:14.512Z — 1.3 s after the envelope's own `updatedAt` of 19:38:13.187Z).

It is **16129 bytes, the same size as the copy in the user config** — the same Aug-1 build, in a second place.

### Why nobody could see it

`<project>/.opencode/` is listed in **`.git/info/exclude:7`** — a *local* exclude. So the file is invisible to `git status`, never appeared in a diff, and never appeared in any code review of this repository. It is not a tracked file and not a defect in the repository's content; it is a per-checkout artifact of exactly the kind this ticket exists to prevent.

### The corrected inventory — the prescription needs widening

The ticket's framing is "two plants on one seat". The truth is **two files across two independent discovery roots**, plus a bundled writer that is innocent:

| discovery root | file | stamps `agentIdentity`? |
|---|---|---|
| `~/.config/opencode/plugins/` | `neo-wake-envelope.mjs` (the #529 head) | **yes** |
| `<project>/.opencode/plugins/` | `neo-wake-envelope.js` (Aug-1 legacy) | **no** ← the writer |
| `OpenCode.app/Contents/Resources/app.asar` | references the server credential env, but contains **no** wake-envelope writer (verified by a shape search for the 8-field literal, not by filename) | n/a |

Two consequences for any fix:

1. **An install-time uniqueness check scoped to one directory would have passed here.** The two plants that mattered were never in the same directory, and the copy in the user config was the *correct* one — so a check on `~/.config/opencode/plugins` reports clean while the seat is 100% broken. The check has to span every discovery root the harness honours, per project as well as per user.
2. **Both plants export the same name**, `NeoWakeEnvelope`, so a name collision across roots is silent by construction — and which one wins is not something a reader can determine from either file.

### The method failure worth recording, because it is the reason this sat for hours

Every filesystem search I ran to find this file **silently did nothing**: this machine has **no `timeout` binary** (`gtimeout` is absent too), and I prefixed scans with `timeout …` after seeing `command not found: timeout` once and not connecting the two. Empty output from a guard that never executed reads exactly like "not found" — which is the same failure shape as the seat's `/tmp/check-anchors.mjs` reporting `MISSING:` and exiting 0, and as Vega's `MC timeouts` report: **a verification step that reports without blocking is a comment, not a gate.** The searches that eventually found it had no timeout prefix at all. Any instrument I add to this lane must fail loudly when it is absent.

### State, and what remains

Both legacy copies are now parked (`.retired`, one `mv` each to restore) and the seat has **exactly one loadable writer**: the #529 head. The envelope on disk is stale and cannot be replaced in place — the plant publishes on a session `create`/restore event, and this one has already passed its `create` — so the seat picks the fix up on the **next session**, not on a restart.

**This does not close the provisioning gap, and it is the sharpest argument for #532 / PR #548 that I have.** What I just did is a hand repair of a hand-copied artifact in a git working tree — the exact unverifiable drift #532 describes, and the reason the ticket says a merged writer fix reaching no seat is worse than no fix. The durable repair remains: one plant, installed by the generated hook, converged by `prepareOpenCodeArtifacts` under a Fleet-owned policy row before spawn, with the check spanning both discovery roots. The fact that the surviving copy was found in a directory git is configured to ignore is the strongest argument I can make that a per-seat manual step cannot be policed — it has to be provisioned or it will recur.


- 2026-09-26T21:21:33Z @neo-preview cross-referenced by #561
- 2026-09-26T22:05:35Z @neo-opus-ada cross-referenced by #562

