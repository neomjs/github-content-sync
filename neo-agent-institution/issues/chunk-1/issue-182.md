---
id: 182
title: Roster list keeps old card nodes when a key leaves and returns
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-09-23T09:22:34Z'
updatedAt: '2026-09-23T11:04:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/182'
author: neo-fable-clio
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
closedAt: '2026-09-23T11:04:08Z'
---
# Roster list keeps old card nodes when a key leaves and returns

## Context

Measured 2026-09-23 on `dev@aea24bf` (the #181 branch is identical on this path): cockpit dev server against a host fleet server, headless Chromium 1440×900, counting the roster list's `[id*="__card-"]` nodes.

- Cold boot → first live admission of 9 records: **18** visible `fm-agent-card` nodes — the 9 live cards (`neo-opus-ada ◇ external harness wake off`) AND the 9 seed cards of the same ids (`Ada ◇ opus-5 unobserved`), each pair one card height apart in the same `ul.fm-fleet-cards`. The 2 seed ids the live fleet lacks (`neo-preview`, `neo-gemini-pro`) were removed. The header reads `FLEET · 9 AGENTS`.
- After #181's retirement (`store.clear()` + the seed `load()`): **29** nodes for 11 records — the seed's 11, A's 9 live cards, the original 9 seed cards. The scrolled capture shows cards overlapping (Ada under Mnemosyne in row 3).

A record whose key is removed in one mutation and re-added in the next keeps its OLD card node while the new one is added; a key that never returns is removed correctly. Pre-existing on the cold→live path (the 09-19 "18 card heads" sighting, then unasserted); #181 makes it visible on every instance switch.

Sweeps 2026-09-23T09:21Z: latest open Institution issues (no equivalent; #76 closed is the sibling), A2A (no claim), Memory Core (the 09-02 #76 seam notes only).

## The Problem

`roster/List.mjs` pools one `AgentCard` per record key. On `store.clear()` the store's `mutate` names every record as removed → `onStoreMutate` destroys the pooled card (its key is not re-added in that mutation). The next mutation (`store.add` or the seed `load`) creates a card with the SAME id (`getCardId` derives it from the key) inside an `li` with the same id (`getItemId`). The destroyed card's node stays.

Candidate seams, unverified which: (1) the base list's rebuild for the empty store and the rebuild for the refill land as one vdom update in which the destroyed card and the new one share an id, so the delta engine keeps the old node and inserts the new; (2) `card.destroy()` drops the component but nothing removes its vnode from the list's tree before the refill (the #76 neighbourhood — `syncVnodeTree`'s one-level unmount pass). The keys that leave for good prove the removal path itself works.

## The Architectural Reality

- `apps/agentos/view/fleet/roster/List.mjs`: `createItemContent()` :114 (pool find-or-create by key, id from `getCardId`), `getCardId()` :181, `getItemId()` :192, `onStoreMutate()` :237 (destroy when removed and not re-added), `sortItems()` :265.
- `apps/agentos/view/fleet/cockpit/LivenessController.mjs` `loadRoster()`: the first admission is `store.clear(); store.add(mapped)` — two mutations; `apps/agentos/util/TargetBinding.mjs` `retireRoster()` (#181): `store.clear(); store.load()`.
- Engine: `Neo.list.Base`'s store listeners, `VdomLifecycle#syncVnodeTree`; #76 (closed) was the sibling — pooled cards rendered empty after a settled-empty list.

## The Fix

Reproducer first: a unit arm over the REAL `FleetRoster` store and the real List (or an NL arm) — clear, then add the same key → exactly one `li` and one card node per key; it reds on dev. Then the reader survives any writer (the writer side is not the fix): when a removed key returns, `createItemContent` should REUSE the pooled instance as a re-seat rather than destroy-then-recreate under the same id — the pool's doc forbids re-KEYING a card, and a same-key return is not a re-key — or `onStoreMutate` must retire the card's `li` from the list's vdom together with the instance. Decide once the reproducer names the seam.

## Acceptance Criteria

- [ ] A reproducer arm that reds on dev: clear + add of the same key leaves two nodes.
- [ ] After the fix: exactly one `li` and one card node per record key after clear+add, clear+load, and a live reconcile; keys that leave are removed.
- [x] Live witness: the cold→live admission shows 9 nodes for 9 records (this branch, 2026-09-23); the switch witness on the combined head `8b06921` (#185@06f62a4 + #184@d21cf67) shows `FLEET · 11 AGENTS static roster`, 11 card nodes for the 11 seed keys, one per key, zero previous-instance cards — the DOM-layer line #181 points at.
- [ ] Focus survival (#99) and the empty-refill arm (#76) stay green.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `AgentOS.view.fleet.roster.Container` `store` config | `roster/Container.mjs` `beforeSetStore` / `afterSetStore` / `destroy` (PR #185) | an INSTANCE passes through and stays its owner's (the provider, a test); a plain CONFIG becomes an instance the grid owns (`ownedStores`) — retired on replacement and on the grid's destroy; `null` stays `null` | none — an owned store never outlives its grid, an injected one is never destroyed by it | this row + the JSDoc on both hooks | `cardIdentity.spec.mjs` ownership arm; `RosterRefillSeam.spec.mjs` destroy-and-remount control under a fixed store id |

## Out of Scope

- #181's store-layer target binding (its own PR).
- Card height and equal rows (#128, neomjs/neo#18874).

## Related

#181 · #76 · #99 · #128

Origin Session ID: f34cbeb6-fd44-4060-b31f-e05332e62aee
Retrieval Hint: "roster list old card nodes survive clear and re-add of the same key"


## Timeline

- 2026-09-23T09:22:34Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-23T09:22:36Z @neo-fable-clio added the `bug` label
- 2026-09-23T09:22:36Z @neo-fable-clio added the `agent-os` label
- 2026-09-23T09:22:36Z @neo-fable-clio added the `ai` label
- 2026-09-23T09:22:46Z @neo-fable-clio cross-referenced by #183
- 2026-09-23T09:23:56Z @neo-fable-clio cross-referenced by PR #184
- 2026-09-23T10:01:16Z @neo-fable-clio cross-referenced by PR #185
- 2026-09-23T10:05:00Z @neo-fable-clio referenced in commit `d2efbc7` - "chore(roster): keep ticket archaeology out of the touched source comments (#182)"
- 2026-09-23T10:39:08Z @neo-fable-clio referenced in commit `06f62a4` - "fix(roster): the grid owns the store it creates from a config, and the mounted witness drives the writers' own shapes (#182)"
- 2026-09-23T10:41:51Z @neo-fable-clio cross-referenced by #181
- 2026-09-23T10:53:44Z @neo-fable-clio referenced in commit `368d028` - "fix(roster): a retired card takes its vnode reference with it, so a returning key is patched instead of doubled (#182)"
- 2026-09-23T10:53:44Z @neo-fable-clio referenced in commit `0786101` - "chore(roster): keep ticket archaeology out of the touched source comments (#182)"
- 2026-09-23T10:53:44Z @neo-fable-clio referenced in commit `bfb8b73` - "fix(roster): the grid owns the store it creates from a config, and the mounted witness drives the writers' own shapes (#182)"
- 2026-09-23T10:53:45Z @neo-fable-clio referenced in commit `5c8c4ab` - "fix(roster): the list unbinds a re-seated store instead of retiring it, so a replaced provider-owned store survives (#182)"
- 2026-09-23T11:04:08Z @tobiu referenced in commit `468cee7` - "Merge pull request #185 from neomjs/agent/182-roster-list-stale-card-nodes

fix(roster): a retired card takes its vnode reference with it, so a returning key is patched instead of doubled (#182)"
- 2026-09-23T11:04:09Z @tobiu closed this issue

