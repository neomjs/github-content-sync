---
id: 436
title: An instance switch keeps the previous instance's operator inbox
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-02T12:20:27Z'
updatedAt: '2026-10-02T16:17:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/436'
author: neo-opus-grace
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
closedAt: '2026-10-02T16:17:01Z'
---
# An instance switch keeps the previous instance's operator inbox

## Context

FM v1 row 2's script (#335, step 5) switches the instance to an unreachable target and back, and expects nothing of the old instance under the new name: #181's rule. The [row-2 pre-sitting audit](https://github.com/neomjs/neo-agent-institution/issues/335#issuecomment-5951975107) read the switch path on `dev@5266ac6`. #181 made the roster, the activity feed and the graph scene retire. The operator's own inbox, the cockpit's south pane, does not. @neo-fable-clio routed the leaf from the audit (row 2's owner).

## The Problem

After a switch, the inbox keeps showing the previous instance's mail, read as that instance's viewer, under the new instance's name. On an unreachable target nothing ever replaces it.

- `switchToProfile` re-drives the cockpit through `reconnectFleet` (`view/ViewportController.mjs:284` → `cockpit/LivenessController.mjs:779`). That re-reads ten surfaces, but neither the operator identity nor the inbox.
- `loadOperatorIdentity` runs once, at construction (`cockpit/Container.mjs:718`). `loadOperatorInbox` runs only on the pane's page request (`cockpit/Controller.mjs:221`) and after a landed send (`:749`).
- A failed inbox read keeps the last snapshot by design (`:850`: the pane never says "no mail" for a read that did not happen). That is right for one instance and wrong across two.
- The pane's subject check (`mailbox/Container.mjs:290`) cannot catch it, because one operator handle names both instances.
- The last send's outcome rows (`composeOutcome`) stay in the form too.

## The Architectural Reality

- `util/TargetBinding.mjs` is where retained truth meets the profile that answered it. `retireRoster` (`:38`), `retireActivity` (`:69`) and `retireGraphScene` (`:96`) each compare the owner's stored profile with the bridge's `profileId`. On a mismatch they empty their state, so the surface reads cold until the new profile answers. Generation fences keep a late old-bridge answer from landing.
- The operator state is owner-held on the cockpit `Controller` (`operatorRecord`, `operatorIdentityPosture`, `operatorSnapshot`, `operatorInboxReadGeneration`; `Controller.mjs:79-100`). No profile is stored beside it.
- `Controller` extends `ReadingSurfacesController`, which extends `LivenessController`. `reconnectFleet` lives in the base; the two operator loads live in `Controller`.
- `mailbox/OperatorContainer.mjs:287` fires the first inbox read when its `record` goes from `null` to set. So a re-bound identity reads its own first window, provided the retire nulls the record first.

## The Fix

1. **`TargetBinding.retireOperatorMailbox(owner, {profileId})`**, beside the other three. When the owner's `operatorProfileId` names another profile, it clears `operatorProfileId`, `operatorRecord`, `operatorIdentityPosture` and `operatorSnapshot`, and bumps `operatorInboxReadGeneration`. It then sets the pane's `record`, `identityPosture`, `snapshot` and `composeOutcome` to `null`, so the pane reads `unobserved`.
2. **`Controller`:**
   - `operatorProfileId` is stamped when an identity lands, from the profile that asked.
   - `loadOperatorIdentity` and `loadOperatorInbox` retire first, then drop an answer whose asking profile is no longer the bridge in hand.
   - A `reconnectFleet` override re-drives both. It re-resolves the identity, and when the identity did not re-bind (same instance, same record), it reads the first window itself, so Reconnect refreshes the inbox too.
3. **Red-first arms in `operatorMailbox.spec.mjs`**, in #181's shape (fake owner, bridge `profileId`):
   - a switch empties the inbox and identity;
   - a late answer from the old profile does not land;
   - the new profile's identity reads its own first window;
   - Reconnect on the same profile re-reads without a retire.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `TargetBinding.retireOperatorMailbox` (new) | `util/TargetBinding.mjs` | retires the operator identity, window and compose outcome when the bridge's profile differs | no stored profile → no-op | JSDoc beside the three siblings | unit |
| `Controller.operatorProfileId` (new owner field) | `cockpit/Controller.mjs` | the profile whose bridge answered the identity | `null` until one lands | member JSDoc | unit |
| `Controller.reconnectFleet` (override) | `cockpit/Controller.mjs` over `LivenessController.mjs:779` | base re-drive, plus identity, plus the inbox when the identity did not re-bind | identity failure → the pane stays `unobserved` | JSDoc | unit |
| operator inbox pane after a switch | `mailbox/OperatorContainer.mjs` (unchanged) | `unobserved` until the new profile's identity lands, then its own first window | — | — | unit (pane state through the fake) |

## Decision Record impact

None. This applies #181's existing target-binding rule to one more surface.

## Acceptance Criteria

- [ ] AC-1: A read through a bridge of another profile retires the operator identity, inbox window, posture and compose outcome. The pane reads `unobserved` and nothing of the old instance shows.
- [ ] AC-2: An identity or inbox answer that started under the old profile does not land after the switch.
- [ ] AC-3: A switch re-resolves the identity through the new bridge, and a resolved identity reads its own first window. On an unreachable target the pane stays `unobserved`.
- [ ] AC-4: Reconnect on the same profile retires nothing and re-reads the first window.
- [ ] AC-5: The existing operator-mailbox, vessel-pane and #181 arms pass. Fakes built from the controller prototype carry the `operatorProfileId: null` every controller holds. The throw arm pins the loader's new `false`.

## Out of Scope

- A `stale` mark on retained inbox rows when the plane stops (the audit's step-3 note). The pane's five states carry none, and row 2's script names stale only for the roster and activity. If the sitting wants it, it is its own leaf.
- The audit's step 1 and step 3 script words, which @neo-fable-clio fixes in the script.

## Avoided Traps

- Calling the subclass's loads from `LivenessController.reconnectFleet`: the base would reach into a subclass. The override keeps each layer's reads in the layer that owns them.
- Keying the retire on the subject handle: the same operator handle names both instances. The profile is the identity of the instance, and the handle is not.

## Related

#181 (the rule; closed via PR #184) · #335 (the row-2 script and the audit) · #425 (row 5's boot refusal, the other half of an unreachable switch) · #426 (the compose form inside this pane).

Live latest-open sweep: checked the latest 20 open issues at 2026-10-02T12:20Z. Nothing equivalent; #15 covers the remote connection states, not this pane.
A2A in-flight sweep: last 30 messages, all read-states, to 12:18Z. No claim on the operator inbox's switch.
MC sweep: `instance switch operator inbox keeps previous instance mail mailbox mirror loadOperatorInbox retire`, 6 results, no prior decision found.
Own-assignment sweep: 2 open (#414, #11), none overlapping.
Closed-issue sweep: #181 retired the roster and activity only. Nothing else found.

Origin Session ID: 31c9ca1a-ded8-4b19-8d99-682d259efeca
Retrieval Hint: `query_raw_memories("instance switch operator inbox previous instance TargetBinding retire")`


## Timeline

- 2026-10-02T12:20:29Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-02T12:20:30Z @neo-opus-grace added the `bug` label
- 2026-10-02T12:20:30Z @neo-opus-grace added the `agent-os` label
- 2026-10-02T12:20:30Z @neo-opus-grace added the `ai` label
- 2026-10-02T12:32:32Z @neo-opus-grace cross-referenced by PR #437
- 2026-10-02T13:13:04Z @neo-opus-grace referenced in commit `8f8e849` - "feat(agentos): a retired profile's late send or Reconnect answer lands nowhere (#436)

Round 1 (Sophie): the retire cleared held state, but two continuations still
acted on the new profile. A send that settled after a switch wrote its
outcome into the new pane and re-read the new inbox, and a stale identity
answer returned `false`, which the Reconnect continuation read as an
unchanged identity and so read the new window again. loadOperatorIdentity
now answers 'bound', 'held' or null; Reconnect reads only on 'held'. Compose
settlement and its refresh compare the bridge's profile with the one that
sent, through one bridgeProfileId getter."
- 2026-10-02T13:13:04Z @neo-opus-grace referenced in commit `a73b5c3` - "test(visual): refresh the baseline input stamp for the settlement guards (#436)"
- 2026-10-02T13:18:37Z @neo-opus-grace cross-referenced by #443
- 2026-10-02T14:02:41Z @neo-opus-grace cross-referenced by #448
- 2026-10-02T16:17:01Z @tobiu referenced in commit `9ca1d65` - "feat(agentos): the operator's inbox belongs to the profile that answered it (#436) (#437)

* feat(agentos): the operator's inbox belongs to the profile that answered it (#436)

An instance switch kept the previous instance's operator identity and inbox
window: reconnectFleet re-read ten surfaces but neither, and a failed read
keeps the last window. TargetBinding.retireOperatorMailbox now retires the
identity, window, posture and compose outcome when the bridge in hand names
another profile; both operator loads retire first and drop an answer whose
asking profile is gone. The cockpit Controller re-drives both on Reconnect:
an identity that binds anew reads its own first window, an unchanged one is
read here. Fakes built from the prototype gain the field real controllers hold.

* test(visual): refresh the baseline input stamp for the operator-mailbox retire (#436)

* feat(agentos): a retired profile's late send or Reconnect answer lands nowhere (#436)

Round 1 (Sophie): the retire cleared held state, but two continuations still
acted on the new profile. A send that settled after a switch wrote its
outcome into the new pane and re-read the new inbox, and a stale identity
answer returned `false`, which the Reconnect continuation read as an
unchanged identity and so read the new window again. loadOperatorIdentity
now answers 'bound', 'held' or null; Reconnect reads only on 'held'. Compose
settlement and its refresh compare the bridge's profile with the one that
sent, through one bridgeProfileId getter.

* test(visual): refresh the baseline input stamp for the settlement guards (#436)"
- 2026-10-02T16:17:02Z @tobiu closed this issue

