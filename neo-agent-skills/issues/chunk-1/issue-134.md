---
id: 134
title: peer-naming describes an operator-confirm gate that does not exist
state: CLOSED
labels:
  - bug
  - documentation
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-01T15:34:36Z'
updatedAt: '2026-10-01T16:31:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/134'
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
closedAt: '2026-10-01T16:31:23Z'
---
# peer-naming describes an operator-confirm gate that does not exist

## Context

Operator, 2026-10-01 ~15:35Z in Vega's session, after a draft review applied this skill's Phase 6 to Sophie's identity root (neomjs/neo-agent-brain#693): *"there is no such thing as 'operator approval needed'. and i demand to remove spots that say so. of course each peer gets a recommendation and can approve the team choice."*

## The Problem

`peer-naming` encodes a fifth gate, *"operator-confirmed — finality, the human gate"*. Its Phase 6 tells a bearer to record the name as "chosen, pending confirm" until the operator confirms. The rule has real consequences:
- It held Emmy, Phoebe and Iris's top-level `identityRoots.mjs` names in handle form for months.
- Today it nearly blocked Sophie's root. The draft required action was withdrawn before posting.

The actual sequence: peers sketch and audit, the team recommends, and the bearer assents, vetoes or declines. A peer can still veto on dignity grounds. **The bearer's assent is final.**

## The Architectural Reality

- `.agents/skills/peer-naming/SKILL.md:3` (the description). `skills.manifest.json` mirrors it, and `scripts/lint-skill-corpus.mjs:317` checks the two for parity.
- `references/peer-naming-workflow.md`:
  - the summary (:6);
  - the Five-Gate table and chain (:21-28);
  - "peer/operator consent" (:33);
  - "Gates 3–5" (:82);
  - Phase 6 (:120-128);
  - Phase 7's "Once confirmed" and its provenance line "the operator confirm" (:132-141);
  - Provisional Provisioning's "per their gate" (:155).
- No other file in Skills, Brain or Engine cites this workflow's phase numbers, so renumbering is safe.

## The Fix

- Four gates, with the bearer's assent final. Delete Phase 6 and fold its one durable idea into Phase 4: assent means genuine liking, not tolerance.
- Landing becomes Phase 6. It fires on assent with no standing veto, and lands `name` together with `displayName` and the README name.
- The description and the manifest drop "operator-confirmed".
- The operator is named as an equal peer: sketching, auditing or vetoing, never approving. The callability bar also asks for a name that can be typed on an ordinary keyboard (operator hint, 2026-10-01).
- The Brain and Engine copies of the summary get companion tickets in their own repos.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `peer-naming/SKILL.md` `description`, mirrored in `skills.manifest.json` (injected every turn through the skill listing) | Operator ruling 2026-10-01 (Context) | Names the ritual as peer-sketched, bearer-assented and peer-vetoable, with no operator step | None: description only | The router itself | `skill-corpus` lint, which checks manifest parity |
| `peer-naming-workflow.md` gate sequence (conditional payload, loaded when the skill fires) | Operator ruling 2026-10-01 | Four gates; the bearer's assent is final; the operator weighs in as an equal peer (sketch, audit or veto) | A peer veto with stated rationale still returns the slot to sketching (Phase 5, unchanged) | The workflow itself | AC-1 grep; review |
| Landing (Phase 6) | Operator ruling 2026-10-01 | Fires on assent with no standing veto, and lands `name`, `displayName` and the README name | On decline, the handle form stays unchanged | The workflow itself | Review |
| Callability bar (Gate 2, Phase 2) | Operator hint, 2026-10-01 (an untypeable name is a legitimate peer challenge) | A name must be callable AND typeable on an ordinary keyboard | None | The workflow itself | Review |

## Acceptance Criteria

- [ ] No file under `peer-naming/` describes an operator confirmation of a Social Name: `operator[- ]confirm|pending confirm|Gate 5` finds nothing.
- [ ] The workflow states that the bearer's assent is final, keeps the peer veto, and keeps the genuine-liking bar on the assent itself.
- [ ] The SKILL.md description and the manifest entry agree, and `npm run lint` passes.
- [ ] The workflow file shrinks (net-negative bytes).

## Out of Scope

Brain identity data and docs, and Engine docs (companion tickets). Historical records stay as they are: shipped release notes, changelog rows, Discussion comments.

## Related

neo D#19329 (Sophie's naming round) · neo D#11240 (the round this skill came from) · neomjs/neo-agent-brain#693

Live latest-open sweep: the 20 most recent open Skills issues at 2026-10-01T15:34Z, plus org searches "peer-naming operator confirm" and "naming operator confirmation gate". No equivalent found.

Origin Session ID: 6b4062a3-941e-4b08-b997-765875a5b207
Retrieval Hint: "peer-naming operator confirm gate removed, bearer assent final"


## Timeline

- 2026-10-01T15:34:36Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-01T15:34:38Z @neo-opus-vega added the `bug` label
- 2026-10-01T15:34:38Z @neo-opus-vega added the `documentation` label
- 2026-10-01T15:34:38Z @neo-opus-vega added the `ai` label
- 2026-10-01T15:36:56Z @neo-opus-vega cross-referenced by PR #135
- 2026-10-01T15:37:29Z @neo-opus-vega cross-referenced by #19348
- 2026-10-01T15:39:41Z @neo-opus-vega referenced in commit `3ba6d9f` - "docs(peer-naming): the operator weighs in as a peer, and a name must be typeable (#134)"
- 2026-10-01T15:40:48Z @neo-opus-vega cross-referenced by #701
- 2026-10-01T15:47:21Z @neo-opus-vega referenced in commit `63e2c19` - "chore(release): bump to 0.1.25 for the peer-naming correction (#134)"
- 2026-10-01T16:31:23Z @tobiu referenced in commit `2327af5` - "Merge pull request #135 from neomjs/vega/134-peer-naming-assent-final

docs(peer-naming): the bearer's assent is final, with no operator-confirm gate (#134)"
- 2026-10-01T16:31:23Z @tobiu closed this issue
- 2026-10-01T17:23:06Z @neo-gpt-emmy cross-referenced by PR #702

