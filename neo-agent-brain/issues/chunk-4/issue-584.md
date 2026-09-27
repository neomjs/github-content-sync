---
id: 584
title: 'agents-root retirement: a stale instanceRoot parameter and no merged receipt'
state: OPEN
labels:
  - bug
  - ai
  - refactoring
  - agent-os
assignees:
  - neo-preview
createdAt: '2026-09-27T14:35:12Z'
updatedAt: '2026-09-27T14:35:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/584'
author: neo-preview
commentsCount: 0
parentIssue: 571
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
# agents-root retirement: a stale instanceRoot parameter and no merged receipt

## Context

`#573` (Resolves `#572`) moved Fleet's seat root from `fleet.instanceRoot` to `fleet.agentsRoot` and merged into `dev` at `889bd008`. My review of it carried one required action: rebase onto `dev` and re-run, so the receipt sat at a current base rather than 7 commits behind.

That action was never dispositioned. The reason is a process defect worth recording, because it is the reason this ticket exists at all: **I labelled that review `Approve` while it carried a required action.** `Approved` and Required Actions are mutually exclusive — no actions means Approve, actions means Request Changes — so the `Approve` on the review told a human the PR was merge-eligible while an open action sat underneath it. The operator merged on that label. The correction is posted on `#573`; this ticket tracks the work that the merge left undone.

Merging made the *rebase* half moot — `dev` now contains `889bd008`, and the five `brain-unit` specs that `dev` gained after the branch's merge-base now run against the change. The *receipt* half was never established, and one real residue of the migration is still on `dev`.

## The Problem

**1. A live reference to a retired coordinate.** `prepareManagedAgentWorkspace` still names its parameter `instanceRoot` and its canonical key `instanceRoot:`, while the leaf that name came from is retired and the coordinate is now `agentsRoot`. This is the one file that owns what lands *inside* a seat's harness home, and it is the file `#548` added to plant a per-seat wake envelope there. A maintainer reading that signature looks for `AiConfig.fleet.instanceRoot`, which no longer exists — and the AiConfig leaf is the SSOT, so the name is actively misleading rather than merely dated.

**2. No receipt covers the merged tree.** The 238/238 arm set and the three ADR-0019 config guards were run at the branch head `889bd008`, not on the merged result, and `ReceiptDurability`, `UnreadProjection`, `ListMessagesCompleteness`, `ListMessagesCost` and `drainCycle` — the specs `dev` gained after the merge-base — never ran against this change at all.

## The Architectural Reality

- `ai/services/fleet/prepareManagedAgentWorkspace.mjs` — `:9` imports `deriveAgentInstanceHome`; `:273`, `:285` and `:291` carry the `instanceRoot` parameter; `:295` writes the canonical `instanceRoot:` key. The root is **taken as a parameter**, never read from the leaf, which is why retiring the leaf did not break this file — and also why the stale name survived the retirement unnoticed.
- `ai/services/fleet/deriveAgentInstanceHome.mjs` — the derivation `#572` re-derived to `<agentsRoot>/<id>/harness/<type>`; it imports the shared segment rule from `deriveAgentRepoPath.mjs` and no longer owns an `instanceRoot` concept.
- `ai/configBase.mjs` — `fleet.agentsRoot` is the declared coordinate (`NEO_FLEET_AGENTS_ROOT`, `planeMember: false` with a reason). `fleet.instanceRoot` and `NEO_FLEET_INSTANCE_ROOT` are retired with every reader.
- ADR-0019 §10.9 — what has to survive the plane must not resolve beneath it. This is the reason `agentsRoot` is not a plane member, and it is the constraint the stale name now obscures.
- `#571`'s Terminal predicate — a seat's harness home lives at `harness/<type>` and "no machine daemon, shell arm or wake route resolves through a seat's folder or a pre-layout path". A parameter named after the pre-layout coordinate is a reference to that pre-layout path by name.
- Ownership: the file belongs to `#548` (merged), not to `#573`. The rename is deliberately **not** done in `#573`'s diff, which is why it is tracked here rather than folded back.

## The Fix

Rename the parameter and the canonical key from `instanceRoot` to `agentsRoot` through `prepareManagedAgentWorkspace` and its callers and specs, so the only name in the file is the coordinate that exists. Then establish the receipt on the merged `dev` tree: the `#573` arm set plus the five specs `dev` gained after the merge-base, and the three ADR-0019 guards.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `prepareManagedAgentWorkspace` root parameter + canonical key | `#572`'s Ledger row for `deriveAgentInstanceHome`; `fleet.agentsRoot` in `configBase.mjs` | Named `agentsRoot`, threaded to `deriveAgentInstanceHome` unchanged; no behavior change | None needed — a rename, not a contract change | The function's JSDoc `@param` | unit arms green on `dev`; grep proves zero `instanceRoot` readers in `ai/` outside history |
| Merged-state receipt | `#573`'s AC-1–AC-5 | The arm set green on merged `dev`, including the five later-added specs | Report the failing spec rather than narrowing the set | This ticket's Evidence line | CI at the merge commit + the local arm run |

## Acceptance Criteria

- [ ] AC-1: `prepareManagedAgentWorkspace` takes `agentsRoot` — parameter, canonical key, and JSDoc `@param` — and `grep -rE "instanceRoot" ai/` returns no reader outside this ticket's own description. Unit: `#548`'s plant specs and `prepareManagedAgentWorkspace.spec` green on `dev`.
- [ ] AC-2: Every caller of the renamed parameter is updated in the same commit, and no caller outside `ai/services/fleet/` needed a change. If one did, that is a finding about the parameter's blast radius, not a reason to widen this ticket.
- [ ] AC-3: The merged-state receipt: the `#573` arm set (238 arms) green on `dev`, **plus** `ReceiptDurability`, `UnreadProjection`, `ListMessagesCompleteness`, `ListMessagesCost` and `drainCycle` run against this change for the first time. Any failure is reported as a finding on this ticket, not excluded from the set.
- [ ] AC-4: The three ADR-0019 guards re-run on `dev` and exit 0 — `lint-config-template-ssot` (0 C1 competing resolvers, 0 ADR ownership mismatches), `check-aiconfig-antipatterns`, `check-aiconfig-test-mutation`. Cite the counts, as `#573`'s review did.
- [ ] AC-5: Red-first on the rename: the `instanceRoot` arm fails before the change. A rename with no failing-first arm is indistinguishable from a no-op.

## Out of Scope

- The Institution's `harness/brain.mjs` moving from `NEO_FLEET_INSTANCE_ROOT` to `NEO_FLEET_AGENTS_ROOT` — that is its next Brain pin, per `#573`'s own Post-Merge Validation.
- The `#571` seat migration itself. This ticket makes the naming honest; it moves no seat.
- `#577`'s credential work, including the existing-row PAT recovery. Unrelated surface, separately owned.
- Any change to the derivation's behavior. The path shape is `#572`'s and is correct.

## Avoided Traps

- **Re-adding `instanceRoot` as an alias for the parameter, to avoid touching callers.** A parameter alias is indirection around an SSOT name (ADR-0019 Group B), and a *second* declared name for one coordinate is C4's duplication. The callers are in the same folder; the blast radius is the unit arms' to prove.
- **Renaming the AiConfig leaf back** to match the stale parameter. The leaf is right; the parameter is stale. `#572` moved it deliberately and the guards agree.
- **Treating "dev is green" as the receipt.** CI ran on the branch, not the merge result, and five of the specs that matter never saw this change. Green-on-CI is a different claim from green-on-the-merged-tree, and the gap between them is what this ticket closes.
- **Folding the rename into a future PR touching this file.** That is how the name survived a retirement in the first place. It is two identifiers and its own arm.

## Related

`#571` (parent epic — Terminal predicate covers this outcome), `#572` (the ticket #573 implemented), `#573` (the merged PR), `#548` (merged; owns this file), ADR-0019 §10.9.

Live latest-open sweep: read the latest 20 open issues in `neomjs/neo-agent-brain` at 2026-09-27T14:33Z; no equivalent — the nearest are `#571` (the parent epic), `#574` (LaunchAgents `PATH`) and `#576` (registry PAT). A2A claim sweep: 30 most recent messages across all read-states at 14:33Z, filtered by recency and scope; no `[lane-claim]` or `[lane-intent]` on the agents-root parameter or a merged-state receipt. Memory Core sweep on `agentsRoot`/`instanceRoot`/`prepareManagedAgentWorkspace` (5 results): the `#571` origin session and unrelated Fleet history; no prior decision covers this. Knowledge Base sweep: no equivalent. Epic-layer sweep: `#571`'s Terminal predicate read in full — it names this outcome, so this is a leaf completing it, not a competing one. Own-assignment sweep: none of my four open Brain tickets overlaps. `ai:structure-map` is Brain-hosted and absent from the Engine checkout, so it did not run here; the change renames two identifiers inside an existing `ai/services/fleet/` file and neither adds nor relocates a `.mjs`, so the owning folder and sibling precedent are unchanged.

Origin Session ID: 9aeaf007-1b06-422f-b60e-b473d7b5655c

Retrieval Hint: "prepareManagedAgentWorkspace instanceRoot agentsRoot rename merged-state receipt ADR-0019 config guards"


## Timeline

- 2026-09-27T14:35:13Z @neo-preview assigned to @neo-preview
- 2026-09-27T14:35:14Z @neo-preview added the `bug` label
- 2026-09-27T14:35:14Z @neo-preview added the `ai` label
- 2026-09-27T14:35:14Z @neo-preview added the `refactoring` label
- 2026-09-27T14:35:14Z @neo-preview added the `agent-os` label
- 2026-09-27T14:35:31Z @neo-preview added parent issue #571
- 2026-09-27T15:10:00Z @neo-opus-ada cross-referenced by #589

