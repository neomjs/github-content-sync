---
id: 400
title: 'The installed FM passes its wake receiver to the Fleet, so launched seats arm'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-01T17:34:51Z'
updatedAt: '2026-10-01T19:20:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/400'
author: neo-opus-vega
commentsCount: 0
parentIssue: 7
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-01T19:20:39Z'
---
# The installed FM passes its wake receiver to the Fleet, so launched seats arm

## Context

Brain PR neomjs/neo-agent-brain#705 (for neomjs/neo-agent-brain#79) makes the Fleet arm the wake route of every Codex Desktop or Claude Desktop seat it starts. It subscribes as the seat and publishes to the host wake receiver. It needs two coordinates, `NEO_WAKE_RECEIVER_MANIFEST` and the new `NEO_WAKE_RECEIVER_BASE` (the receiver's address as the plane sees it). Neither reaches the installed FM's Fleet today: `buildPackagedBrainEnv` (`harness/brain.mjs:543`) passes neither. So on the installed app every launched seat would report `wakeRoute: unarmed — no wake receiver is declared`, and the operator's peers stay unwakeable after they move into the FM.

## The Problem

A Finder-launched app inherits no shell exports, so the runbook's `export NEO_WAKE_RECEIVER_*` lines never reach it. The receiver's real coordinates already exist on the machine. The local-agent-os runbook installs `~/Library/LaunchAgents/com.neomjs.agent-os-wake.plist`, whose program arguments carry `--manifest <path>` and `--port <n>`; on the operator's host these are `…/Neo/AgentOS/wake/routes.json` and port `3199`, and the dockerized plane reaches the receiver at `host.docker.internal:3199`, verified live today by Sophie's repair.

## The Fix

On every packaged launch, `main.mjs` settles the receiver once (`harness/wakeReceiver.mjs`, in the shape of Institution #398's seat root). It settles against the plane base the Brain resolved, and the result joins the Fleet child's env, since the Fleet is its only consumer. The packaged profile (`buildPackagedBrainEnv`) stays unchanged: the plane base is known only after that profile resolves, and the smoke shares the profile. The settle order is:
1. The launch environment, when both variables are present (a shell-started FM).
2. Otherwise, the receiver's own LaunchAgent: `--manifest` gives the manifest path. `--port` gives the base only while the FM attaches to the local dockerized plane (a loopback `planeBase`), as `http://host.docker.internal:<port>`.
3. Otherwise nothing: the Fleet's seats report `unarmed` with the Brain's own reason.

Every launch logs `HARNESS_WAKE_RECEIVER {origin, manifest, base, reason}`. A malformed plist is a named refusal, never a guess.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `NEO_WAKE_RECEIVER_MANIFEST` + `NEO_WAKE_RECEIVER_BASE` in the Fleet child's env | Brain leaves `fleet.wakeReceiverManifestPath` and `fleet.wakeReceiverBase` (neomjs/neo-agent-brain#705) | Both are passed when the receiver is settled. A packaged launch that settles none sets both to `''`, which clears an inherited half, because the child inherits `process.env` beneath its fragment. An unpackaged launch adds neither and inherits. | `''` reads as undeclared in the Brain, so seats stay `unarmed` with its reason | `harness/wakeReceiver.mjs` `wakeReceiverEnv` | `wakeReceiver.spec.mjs`: the fragment arm, plus the child-boundary arm that captures the spawn env for both inherited halves; `pack.spec.mjs` pins the wiring |
| Settle precedence | `harness/wakeReceiver.mjs` `settleWakeReceiver` | The launch environment wins only when it carries both coordinates. Next is the receiver's LaunchAgent, whose `--port` gives `http://host.docker.internal:<port>` only while the Fleet's plane is loopback. Otherwise none. | Origin `none` or `launch-agent`, with the reason | Module JSDoc | The settle-order arms of `wakeReceiver.spec.mjs` |
| Reading the LaunchAgent | The runbook's XML plist `~/Library/LaunchAgents/com.neomjs.agent-os-wake.plist` | A strict property-list parse. Only the declaration and DOCTYPE may precede the root. A comment, processing instruction, truncation, trailing content, duplicate key or unknown entity refuses. `ProgramArguments` must be strings, with an absolute `--manifest` and a `--port` from 1 to 65535. | A named refusal that names the file; no coordinates | Module JSDoc | The malformed and adversarial fixture arms; the real installed LaunchAgent read locally agrees with `plutil` |
| Launch log `HARNESS_WAKE_RECEIVER` | `harness/main.mjs` | One line per packaged launch: `{origin, manifest, base, reason}` | None | None | `pack.spec.mjs` pins the line |

## Acceptance Criteria

*AC-1 and AC-3 were restated 2026-10-01 at implementation, when the receiver moved from `buildPackagedBrainEnv` to the Fleet child's env (see The Fix). AC-1 and AC-2 were sharpened on the same day by the first review: a declined resolution clears inherited halves, and the plist is read strictly.*

- [ ] The Fleet child's env carries `NEO_WAKE_RECEIVER_MANIFEST` and `NEO_WAKE_RECEIVER_BASE` when the receiver is settled. A packaged launch that settles none clears both, so an inherited half never reaches the Fleet. An unpackaged launch inherits. Specs cover `wakeReceiverEnv` and the captured spawn env; the wiring is pinned in `pack.spec.mjs`.
- [ ] The settle order is environment, then LaunchAgent, then none. The base is derived from the plist only in local-plane attach mode. The plist is read strictly, so a construct a reader would skip never names the receiver. Specs run over fixture plists, including malformed and adversarial ones.
- [ ] The launch log `HARNESS_WAKE_RECEIVER` names the origin, and the reason when nothing is settled. The settled result has a spec; the log line is pinned in `pack.spec.mjs`.
- [ ] *(Post-merge, L4, installed FM, after a Brain pin carrying neomjs/neo-agent-brain#705)* A freshly started Codex Desktop seat reports `wakeRoute.state: 'ready'`, and a peer's normal-priority message wakes it.

## Out of Scope

- Installing or configuring the receiver itself (the runbook's job).
- Non-local planes, which cannot reach a host receiver without a tunnel; they stay unarmed with a reason.
- The first-run recipe (Brain #679), which may later record these coordinates. This change reads the receiver's own declaration and does not compete with that recipe.

## Related

neomjs/neo-agent-brain#79 · neomjs/neo-agent-brain#705 (depends on its env names) · #398 (seat-root precedent) · neomjs/neo-agent-brain#571

Live latest-open sweep at 2026-10-01T17:34Z: the 15 most recent open Institution issues plus the org search "wake receiver FM Fleet env manifest". No equivalent found.

Origin Session ID: 6b4062a3-941e-4b08-b997-765875a5b207



## Timeline

- 2026-10-01T17:34:51Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-01T17:34:53Z @neo-opus-vega added the `enhancement` label
- 2026-10-01T17:34:53Z @neo-opus-vega added the `ai` label
- 2026-10-01T17:37:33Z @neo-gpt-emmy cross-referenced by PR #705
- 2026-10-01T18:04:03Z @neo-opus-vega cross-referenced by #79
- 2026-10-01T18:20:57Z @neo-opus-vega added parent issue #7
- 2026-10-01T18:21:49Z @neo-opus-vega cross-referenced by PR #405
- 2026-10-01T18:50:48Z @neo-opus-vega referenced in commit `ca9ffe9` - "fix(harness): a declined wake receiver clears inherited halves, and the LaunchAgent is read as a whole property list (#400)

Two boundaries Sophie's review found.

The Fleet child inherits process.env beneath its fragment, so the empty
fragment for a declined resolution let an inherited half-declaration reach
the Fleet while the log said none. A packaged launch that settles no receiver
now exports both coordinates as '', which reads as undeclared in the Brain.
An unpackaged launch still inherits.

The ProgramArguments pattern could read a pseudo array that a plist reader
skips (inside a processing instruction or comment), and it accepted a
truncated document. The LaunchAgent is now parsed strictly as a property
list: only the declaration and DOCTYPE may precede the root, and comments,
processing instructions, truncation, trailing content, duplicate keys and
unknown entities refuse."
- 2026-10-01T18:55:57Z @neo-opus-vega cross-referenced by #722
- 2026-10-01T19:20:39Z @tobiu referenced in commit `6246a89` - "feat(harness): the installed FM hands its wake receiver to the Fleet, so the seats it launches can arm (#405)

* feat(harness): the installed FM hands its wake receiver to the Fleet, so the seats it launches can arm (#400)

A Finder-launched FM inherits no shell exports, so the Fleet it starts never
learned where the host wake receiver is, and every GUI seat it launched stayed
unarmed. Each packaged launch now settles the receiver once: the launch
environment when it carries both coordinates, else the receiver's own
LaunchAgent (--manifest, and --port as host.docker.internal while the plane is
local), else none. A malformed LaunchAgent is a named refusal. The result is
logged as HARNESS_WAKE_RECEIVER and joins the Fleet child's env, its only
consumer.

* fix(harness): a declined wake receiver clears inherited halves, and the LaunchAgent is read as a whole property list (#400)

Two boundaries Sophie's review found.

The Fleet child inherits process.env beneath its fragment, so the empty
fragment for a declined resolution let an inherited half-declaration reach
the Fleet while the log said none. A packaged launch that settles no receiver
now exports both coordinates as '', which reads as undeclared in the Brain.
An unpackaged launch still inherits.

The ProgramArguments pattern could read a pseudo array that a plist reader
skips (inside a processing instruction or comment), and it accepted a
truncated document. The LaunchAgent is now parsed strictly as a property
list: only the declaration and DOCTYPE may precede the root, and comments,
processing instructions, truncation, trailing content, duplicate keys and
unknown entities refuse."
- 2026-10-01T19:20:40Z @tobiu closed this issue

