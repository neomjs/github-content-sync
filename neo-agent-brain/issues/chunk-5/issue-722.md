---
id: 722
title: 'The Wake routes pane says a seat is unarmed but not why, though the Fleet recorded the reason at start'
state: CLOSED
labels: []
assignees:
  - neo-opus-vega
createdAt: '2026-10-01T18:55:56Z'
updatedAt: '2026-10-01T20:03:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/722'
author: neo-opus-vega
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
closedAt: '2026-10-01T20:03:23Z'
---
# The Wake routes pane says a seat is unarmed but not why, though the Fleet recorded the reason at start

## Context

PR #705 (for #79) makes the Fleet arm a launched GUI seat's wake route at start. When it cannot, the seat's lifecycle `status().wakeRoute` records `unarmed` and the reason. Examples: "no wake receiver is declared", "the seat runs a resident Memory Core", "the seat credential did not prove @…". The FM's Wake routes pane (`apps/agentos/view/fleet/wake/Container.mjs`) renders the decomposed wake-routes source. That source's armed axis comes from the receiver manifest alone (`fleetWakeRoutesSource.mjs` `readArmingAxis`). So a seat whose arming failed shows `armed: none` with `reason: null`, and the operator sees that the seat cannot be woken but not why.

## The Fix

The wake-routes source takes one more injected reader, the seat's lifecycle `wakeRoute`. When the manifest holds no route for a seat (`none`), and the Fleet's last arming of that seat reported `unarmed`, the armed row carries that reason. The manifest stays the authority for *armed*: a seat it carries stays `armed`. An axis that is unobserved or unknown keeps its own reason. `wireFleetWakeRoutesSource` forwards the reader, and `devFleetServer` supplies it from `FleetLifecycleService.status(id).wakeRoute`.

## Acceptance Criteria

- [ ] A seat the manifest does not carry, whose lifecycle `wakeRoute` is `unarmed` with a reason, answers `armed: {state: 'none', reason: <that reason>}`.
- [ ] A seat the manifest carries answers `armed` whatever its lifecycle says. An unobserved or unknown arming axis keeps its own reason. A seat with no lifecycle route answers `none` with `reason: null`, as today.
- [ ] The reason passes the source's existing redaction (`redactReason`), so the bound and redaction rules match the other axes.
- [ ] The source stays Neo-free: the reader is injected, and the source spec covers the three cases above.

## Out of Scope

- The Institution pane already renders `armedReason` beside `armedState` (`Container.mjs`), so it needs no change if that holds. The PR checks this.
- Seats the Fleet did not start in this process carry no lifecycle route. They keep `reason: null`.

## Related

#79 / PR #705 (the lifecycle `wakeRoute`, which this is stacked on) · neomjs/neo-agent-institution#400 (the installed FM passes the receiver) · #571

Live latest-open sweep at 2026-10-01T18:5xZ: the 8 most recent open Brain issues, plus org searches for "armed axis reason", "wake routes unarmed reason" and "seat arming reason pane". No equivalent.

Origin Session ID: 6b4062a3-941e-4b08-b997-765875a5b207


## Timeline

- 2026-10-01T18:55:57Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-01T19:00:30Z @neo-opus-vega cross-referenced by PR #723
- 2026-10-01T19:19:07Z @neo-opus-vega referenced in commit `77ad23a` - "feat(fleet): the Wake routes pane says why a seat is unarmed, from the Fleet's own arming at start (#722)

The armed axis came from the receiver manifest alone, so a seat whose
arming failed showed `none` with no reason, while its lifecycle wakeRoute
held the reason (#705). The wake-routes source now takes readFleetArming.
When the manifest carries no route for a seat and the Fleet's last arming
of it reported unarmed, the row carries that reason through the source's
own redaction. The manifest stays the authority for armed, and an
unobserved or unknown axis keeps its own reason. The wiring forwards the
reader, and devFleetServer supplies it from the lifecycle status."
- 2026-10-01T19:29:44Z @neo-opus-vega referenced in commit `35334be` - "feat(fleet): the Wake routes pane says why a seat is unarmed, from the Fleet's own arming at start (#722)

The armed axis came from the receiver manifest alone, so a seat whose
arming failed showed `none` with no reason, while its lifecycle wakeRoute
held the reason (#705). The wake-routes source now takes readFleetArming.
When the manifest carries no route for a seat and the Fleet's last arming
of it reported unarmed, the row carries that reason through the source's
own redaction. The manifest stays the authority for armed, and an
unobserved or unknown axis keeps its own reason. The wiring forwards the
reader, and devFleetServer supplies it from the lifecycle status."
- 2026-10-01T20:03:23Z @tobiu referenced in commit `8c96f35` - "feat(fleet): the Wake routes pane says why a seat is unarmed, from the Fleet's own arming at start (#722) (#723)

The armed axis came from the receiver manifest alone, so a seat whose
arming failed showed `none` with no reason, while its lifecycle wakeRoute
held the reason (#705). The wake-routes source now takes readFleetArming.
When the manifest carries no route for a seat and the Fleet's last arming
of it reported unarmed, the row carries that reason through the source's
own redaction. The manifest stays the authority for armed, and an
unobserved or unknown axis keeps its own reason. The wiring forwards the
reader, and devFleetServer supplies it from the lifecycle status."
- 2026-10-01T20:03:23Z @tobiu closed this issue

