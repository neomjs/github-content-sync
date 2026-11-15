---
id: 1772
title: who_is_online names an auto-provisioned identity by its id
state: OPEN
labels:
  - bug
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-11-15T21:33:08Z'
updatedAt: '2026-11-15T21:36:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1772'
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
---
# who_is_online names an auto-provisioned identity by its id

## Context

On 2026-11-15 the Fleet sent `[open-work:unowned]` for neomjs/neo#21048, saying that no seat is named as its author. Its body reads "Authored by Maryam (…)". Six of the last eight unowned wakes to @neo-opus-grace were PRs by @neo-gemini-maryam: neomjs/neo#21040 and #21048, neomjs/create-app#65, neomjs/neo-agent-brain#1701 (three episodes), and two private-repository PRs. @neo-opus-vega posted a defect-note for another one on 2026-11-14.

`who_is_online` (verbose) returns `name: "@neo-gemini-maryam"` for her, while `get_node('@neo-gemini-maryam')` returns `name: "Maryam"`. No other named seat differs.

## The Problem

`_projectAgentLiveness()` (`ai/services/memory-core/WakeSubscriptionService.mjs:1309`) computes `name = node.name || props.displayName || node.id`. Its input comes from `_listAgentIdentityNodes()`, which parses SQLite rows. The store writes only `{id, label, properties}` (`ai/graph/storage/SQLite.mjs:463-467`), and `GraphService.upsertNode()` keeps a node's name at `properties.name` (`ai/services/memory-core/GraphService.mjs:522`). So `node.name` is never set on these rows:
- a seeded identity shows its `displayName`;
- an identity without one, which is every auto-provisioned identity, shows its id.

The Fleet resolves a PR's author line through these names. `seatIdentities().byName` (`ai/services/fleet/wireFleetOpenWorkSource.mjs:43`) matches `row.name`, so "Authored by Maryam" finds no seat. `ownerOf()` (`ai/services/fleet/openWorkReducer.mjs`) then answers `unowned`, and the PR's next action escalates to a rotation seat (#807). An operator without our roster has only auto-provisioned agents, so every agent-authored PR there would be unowned.

## The Architectural Reality

- Maryam's node was auto-provisioned at her first sign-in (2026-10-17T19:32Z). It carries `properties.name: 'Maryam'` and no `displayName`, as #1085 records. `identityRoots.mjs` gives rostered residents a `displayName`, so for our team the bug shows only on a node the roots have not been projected onto.
- #875: runtime identity facts come from the plane's graph. The graph already holds the right name; this reader looks for it in the wrong place.
- The same parsed-row read appears nowhere else: of the modules that `SELECT data FROM Nodes`, only this one reads a top-level `name`.
- The line came with the Agent OS's move into this repository (`11552b08`), and no spec asserts a row's name. `WakeSubscriptionService.spec.mjs` seeds identities with `GraphService.upsertNode({…, name: 'Alice'})`, which stores the name where the projection does not look.

## The Fix

In `_projectAgentLiveness()`, read the stored name after the roots' display name: `props.displayName || props.name || node.id`. Every row that has a `displayName` reads as before. Of the 16 roots, only `@system` has a `displayName` that differs from its `name` ("System" against "System Sender"). No data change and no reseed.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `who_is_online` row `name` (`WakeSubscriptionService._projectAgentLiveness()`) | `GraphService.upsertNode()` keeps a node's name at `properties.name`; #875 (identity facts come from the plane's graph) | A row's `name` is the identity's `displayName`, else its stored name, else its id | An identity with neither keeps its id | No: `openapi.yaml:980` types `name` as a string and names no source | `WakeSubscriptionService.spec.mjs`, the #1772 arm |
| Fleet author resolution (`seatIdentities().byName`, unchanged) | #1485 (`byName` reads the plane's nodes) | "Authored by <name>" resolves a seat whose node has a stored name and no `displayName` | A name no row carries stays `unowned` | No | The same arm, and AC-3 after deployment |

## Acceptance Criteria

- [ ] An identity upserted with a `name` and no `displayName` is listed by `who_is_online` under that name: a `WakeSubscriptionService.spec.mjs` arm, red on the merge base.
- [ ] An identity with neither keeps its id as its name, and a `displayName` still wins over a stored name.
- [ ] [Post-merge, owner Grace] Once the plane runs the fix, `who_is_online` names `@neo-gemini-maryam` "Maryam", and her next PR raises no `unowned` wake.

## Out of Scope

- Falling back to the author login in `ownerOf()`: its "never silently its login" rule stands.
- Reseeding the plane's identity roots.
- Moving other readers off `identityRoots.mjs` (#875).

## Related

#875 · #1485 (`byName` reads the plane's nodes) · #807 (unowned escalation) · #1085 (Maryam's root)

Decision Record impact: none.

Live latest-open + closed sweep: latest 20 open + open and closed search ("unowned author", "Authored by social name seat", "identity name handle who_is_online", "seedAgentIdentities projection", "who_is_online name"; #700, #875, #1085 read) at 2026-11-15T21:28Z, re-read at 21:34Z; 161 open (red band); no equivalent. #875 owns readers that depend on the roots file, and this is a read-path bug inside the plane. A2A claims (latest 30, all read states): none on this scope. Memory Core: Vega's 2026-11-14 defect-note (same symptom, cause unchecked), no ticket. Own assignments: none overlapping. Structure map: ran; the owning file exists, no new file.

Origin Session ID: cd886d2b-b1fb-4f6a-8e9c-e3a14ae48e68

Retrieval Hint: "who_is_online row name reads node.name but SQLite rows keep name at properties.name; Fleet unowned wakes for an auto-provisioned seat"

## Timeline

- 2026-11-15T21:33:08Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-15T21:33:10Z @neo-opus-grace added the `bug` label
- 2026-11-15T21:33:10Z @neo-opus-grace added the `ai` label
- 2026-11-15T21:40:38Z @neo-opus-grace cross-referenced by PR #1773

