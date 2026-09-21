---
id: 337
title: A host daemon can crash-restart for days and reach no surface
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-06T10:20:04Z'
updatedAt: '2026-09-06T12:06:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/337'
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
closedAt: '2026-09-06T12:06:13Z'
---
# A host daemon can crash-restart for days and reach no surface

> Split out of #335 during implementation. #335 was filed over-scoped — "the root owns its closure" and "a dead daemon is visible" are different substrates, and bundling them made #335 not one-PR-resolvable. This is the second half.

## Context

`com.neomjs.agent-os-host-edge` died at 2026-09-04T15:39:53Z and crash-looped under `KeepAlive` + `ThrottleInterval 10` for roughly 42 hours: **274 launchd runs, 4,648 logged crashes, a 5.3 MB `launchd.err.log`**, one process spawned every ten seconds.

Nothing anywhere reported it. Not `healthcheck`, not the orchestrator digest, not a defect-note, not A2A. The outage was discovered only because a human noticed his chat model would not load and asked why.

That daemon is the LM Studio supervisor, and LM Studio carries `"autoStartOnLaunch": false` — so its death was a **queued full-plane outage** for every seat on the host, waiting for the next reboot to land. It landed.

## The Problem

The container plane observes itself well. `healthcheck` reports embedding-provider reachability, WAL drain depth, corpus projection freshness, backup durability, heap starvation. The **host** plane — two LaunchAgents outside every checkout — is observed by nothing that reports.

The asymmetry is easy to miss because the host plane is *supervised*: launchd faithfully restarted the daemon 274 times. Supervision without observability converts a hard failure into an indefinite silent one. A daemon that crashes once and stays down is louder than this was.

Note what a naive threshold would have missed: `KeepAlive` means the label is always *loaded*, and `launchctl list` shows it. The distinguishing facts are `active count = 0` alongside a climbing `runs` and a nonzero `last exit code` — a liveness probe keyed on "is the agent installed" reports healthy throughout.

## The Architectural Reality

- `launchctl print gui/<uid>/<label>` already exposes everything needed: `state`, `runs`, `last exit code`, `active count`. No new bookkeeping is required — this is a read that nobody performs.
- `ai/scripts/diagnostics/check-runtime-root.mjs` (#335) reads the same plists to find each agent's `WorkingDirectory`, so the plist-reading seam exists.
- A defect-note is the zero-ceremony channel (`ticket-create` §1e) and folds per fingerprint, so a crash-looping daemon would produce one standing row rather than 4,648.
- The host plane is explicitly **outside** the container's authority: `orchestrator.log` records `Not running 17 lane(s) this role does not own … This process does not verify that the owning role is live.` Whatever reports this must not pretend to own it.

Structure-map gate: no novel directory. Extends `ai/scripts/diagnostics/` and/or the orchestrator's existing digest path.

## MEASURED 2026-09-06 — one of the three proposed homes is not available

Before designing, I checked whether the container plane can observe host launchd state at all. **It cannot**, and this is a hard constraint rather than a preference:

```
docker exec …orchestrator-1 → command -v launchctl  → absent (Linux container)
docker inspect …orchestrator-1 → the host-edge state dir is NOT mounted
```

Its only host binds are managed Docker volumes, `~/.neo-ai/secrets/mcp-auth-token`, `/var/run/docker.sock`, and one `kb-config.yaml`. Nothing exposes `~/Library/Application Support/Neo/AgentOS`, and no Linux container has `launchctl` regardless.

**So option 1 below — a `hostAgents` facet on the orchestrator healthcheck — cannot be built as stated.** The healthcheck could still be the *presentation* surface, but only if a **host-side producer** writes the observation somewhere the container already reads. That is a second component, not a field.

This also strengthens the case for option 2 on its own merits rather than by preference: the observation has to originate host-side anyway, and a host-side producer that emits a defect-note needs no new mount, no new volume, and keeps working when the container plane is the thing that is degraded — which is the case this ticket is about.

**One shape the ticket did not consider, now the leading candidate:** the two host agents are peers of each other and both are POSIX processes with `launchctl` available. Either can observe the other's `runs` / `last exit code` / `active count` without any new plumbing. The asymmetry is deliberate — a supervisor cannot report its own death, but its sibling can. Whether that coupling is acceptable is the open design question; it is cheaper than every alternative and it is the only one that survives the container plane being down.

**Caveat carried from #335, which applies to any producer chosen:** whatever emits this must not itself live in a tree a seat can mutate. See #335 / PR #338.

## The Fix

Make a repeatedly-exiting host daemon produce a durable signal. The shape is deliberately not prescribed here — three plausible homes, and the choice is the implementer's:

1. a `hostAgents` facet on the orchestrator healthcheck (nearest to where operators already look);
2. an automatic defect-note per label+fingerprint (survives a dead plane, which a healthcheck field does not);
3. a row in the orchestrator digest.

(2) is the only one that still works when the reporting plane is itself degraded, which is the case this ticket is about — but it is a recommendation, not a ruling.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| host-agent exit observability | this ticket | consecutive exits produce a durable signal | none today | local Agent OS README | 4,648 unreported crashes |
| `launchctl print` fields | launchd | `runs` / `last exit code` / `active count` read | — | — | read live during the incident |

## Decision Record impact

`none`.

## Acceptance Criteria

- [ ] A host daemon that exits N consecutive times produces a durable signal a human or agent encounters without knowing to look. The reviewer can name the surface and the threshold.
- [ ] The signal distinguishes crash-looping from cleanly stopped and from never-installed. `active count = 0` with a climbing `runs` is the discriminating state; an "is it loaded" probe reports healthy throughout and does not satisfy this.
- [ ] Test drives the crash-looping state and asserts the signal appears. It must fail when the signal is removed — a spec that passes on a daemon that never crashed proves nothing.
- [ ] One standing signal per label, not one per restart. 4,648 notes is a second outage.
- [ ] Reporting degrades honestly: if the observing plane cannot read host state, it reports unobserved rather than healthy.

## Out of Scope

- Repairing the crash cause or the runtime root — #335.
- Deciding whether the host plane should exist at all — #84 / #90.
- Any change to launchd supervision itself. This ticket observes; it does not restart, throttle, or stop anything.

## Avoided Traps

- **Reporting from the container plane as if it owned the host.** The orchestrator explicitly records that it does not verify the owning role is live. A field implying otherwise is worse than absence.
- **A healthcheck field alone.** During this incident the plane most likely to be degraded is the one being asked to report; a signal that dies with it has no reader when it matters.
- **One note per restart.** Ten-second respawn under `KeepAlive` makes an unfolded signal a flood.
- **Threshold on "is the agent loaded".** `KeepAlive` keeps the label loaded through the entire failure.

## Related

- #335 (parent defect; the crash this ticket would have surfaced)
- #336 (LM Studio supervisor vs. its own update window)
- #54 (an external plane cannot recover itself — the theme this instantiates on the host side)

`unowned-rationale:` parked deliberately. The fix's home is a design choice across three plausible surfaces, and I would rather it be claimed by whoever owns the one chosen than pre-committed by the reporter. I hold the incident evidence and will hand it over.

Origin Session ID: 70502f9a-5b14-4dcf-bcdf-4a29b546df77

Retrieval Hint: `query_raw_memories` — "host-edge crash loop unreported 4648 launchd runs no surface"; `launchctl print` fields `runs` / `last exit code` / `active count`.



## Timeline

- 2026-09-06T10:20:05Z @neo-opus-grace added the `bug` label
- 2026-09-06T10:20:06Z @neo-opus-grace added the `ai` label
- 2026-09-06T10:20:06Z @neo-opus-grace added the `agent-os` label
- 2026-09-06T10:20:41Z @neo-opus-grace cross-referenced by #335
- 2026-09-06T10:24:29Z @neo-opus-grace cross-referenced by PR #338
- 2026-09-06T11:12:31Z @neo-opus-grace assigned to @neo-opus-grace
### @neo-opus-grace - 2026-09-06T12:06:12Z

Closing NOT_PLANNED — and it should never have been a separate ticket, because **#54 already owns it**.

#54 is *"An external plane cannot recover itself: it breaks, we see it, and nothing brings it back."* That is this ticket's whole thesis, filed earlier, assigned to me, and scoped as an epic. I split #337 out of #335 this morning for one-PR-resolvability without checking whether the concern already had a home. It did.

It is also aimed at the host plane #84 deletes and #90 replaces, so the observer would outlive its subject by very little.

**The measurement is the part worth keeping, and it goes to #54:** the container plane cannot observe host launchd state at all — no `launchctl` in a Linux container, and `docker inspect` shows the host-edge state dir is not mounted (only managed volumes, the machine-level auth token, `docker.sock`, one config file). So any "the host plane is dead" signal must originate host-side. The cheapest shape, if the plane survives the cutover: the two host agents are peers and both have `launchctl`, so either can observe the other's `runs` / `last exit code` / `active count` with no new plumbing — a supervisor cannot report its own death, its sibling can.

Empirical anchor for #54: this plane crash-looped **4,648 times over 42 hours** and reached no surface — no healthcheck field, no digest row, no note. It was found because a human noticed a chat model would not load.


- 2026-09-06T12:06:13Z @neo-opus-grace closed this issue
- 2026-09-06T12:06:54Z @neo-opus-grace cross-referenced by #84
- 2026-09-06T12:06:55Z @neo-opus-grace cross-referenced by #54

