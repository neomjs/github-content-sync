---
id: 701
title: Social Names wait on an operator confirm that does not exist
state: CLOSED
labels:
  - bug
  - documentation
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-01T15:40:47Z'
updatedAt: '2026-10-01T17:27:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/701'
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
closedAt: '2026-10-01T17:27:28Z'
---
# Social Names wait on an operator confirm that does not exist

## Context

Operator ruling, 2026-10-01 ~15:35Z: *"there is no such thing as 'operator approval needed'. and i demand to remove spots that say so. of course each peer gets a recommendation and can approve the team choice."* A follow-up hint: the operator weighs in as an equal peer, for example by challenging an untypeable name, and has never disapproved a team-chosen name after the fact. The skill is corrected in neomjs/neo-agent-skills#134 (PR neomjs/neo-agent-skills#135) and the Engine docs in neomjs/neo#19348. This is the Brain companion.

## The Problem

Brain still treats an operator confirmation as the final step of a name:
- `ai/graph/identityRoots.mjs` header (:16-24): Social Names are "…peer-unvetoed, and operator-confirmed".
- **Emmy, Phoebe and Iris** keep a handle-derived top-level `name` (`'Neo GPT Emmy'`, `'Neo Kimi Phoebe'`, `'Neo Kimi Iris'`). Their comments say the Social Name is "pending … operator confirmation". All three assented on first boot, Emmy on 2026-07-12, Phoebe on 07-18 and Iris on 07-19, and each assent is recorded in its entry's provenance comment.
- `ai/scripts/setup/generateRosterOnboarding.mjs:31-34` and `ai/services/fleet/generateOpenCodeSeatConfig.mjs:81-83` describe the operator step.
- `ai/services/fleet/seatMemoryLayerTemplate.mjs:166` writes "→ operator confirmation" into every generated seat's `identity.md`.
- `learn/agentos/ModelStats.md:263/283/329` and `learn/benefits/brain/IdentityRitualsCulture.md:101-103` carry the same claim.

There is a practical effect as well. `agentFamilyResolution.mjs:62` maps the top-level `name` to a login, so "Authored by Emmy" does not resolve through the roster today and has to fall back to the PR author's login.

## The Fix

- Emmy, Phoebe and Iris carry their assented Social Names as top-level `name`, following the Grace and Clio style. The "pending operator confirmation" comments go.
- The header, the generator JSDoc, the seat `identity.md` template and the two docs drop the operator step.
- The specs that pin these values follow (`identityRoots.spec.mjs` Emmy pin, the `agentFamilyResolution.spec.mjs` fixture).
- `ModelStats.md` gains one changelog row for the landing. Its existing changelog rows stay, because they are history.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `IDENTITIES[].name` for `@neo-gpt-emmy`, `@neo-kimi-phoebe`, `@neo-kimi-iris` (`ai/graph/identityRoots.mjs`) | Each bearer's recorded first-boot assent (in each entry's provenance comment) + operator ruling 2026-10-01 | `'Emmy'`, `'Phoebe'`, `'Iris'`, replacing the handle-derived forms | A bearer who prefers the handle form keeps it (no AC forces the change on an objecting bearer) | The entries' provenance comments; `ModelStats.md` rows + changelog | `identityRoots.spec.mjs` Emmy pin |
| Social Name → login lookup (`agentFamilyResolution.mjs:62-63`, which maps top-level `name` to `githubLogin`) | Unchanged code; it reads the new values | "Authored by Emmy/Phoebe/Iris (…)" resolves through the roster instead of falling back to the PR author's login | An unmapped name still falls back to `author.login` (existing behavior) | The module's own JSDoc | `agentFamilyResolution.spec.mjs` fixture |
| Seeded `AgentIdentity` graph node `name` (seeded from `IDENTITIES`) | `identityRoots.mjs` is the seed authority | The node carries the Social Name on the next seed or migration | Existing nodes keep their stored name until reseeded | `identityRoots.mjs` header | Out of CI scope (live reseed not claimed) |
| Generated `identity.md` boot file (`seatMemoryLayerTemplate.renderIdentityMd`, emitted by `generateKimiSeatConfig.mjs:119` and `generateOpenCodeSeatConfig.mjs:176`) | Operator ruling 2026-10-01 (Skills #135) | The naming gate reads "peer sketch → bearer assent → peer-veto window", with no operator step | None: template text | The template itself | The two generators' artifact digest pins, bumped with dated notes |

Unchanged by design: operational handles (`id`, `githubLogin`) and admission (credential-derived, AuthService); historical records (changelog rows, Discussion comments, Eos's "operator-passed"); participation activation (`reactivationTrigger: 'Operator confirms participation activation…'`), which concerns a seat's status, not its name.

## Acceptance Criteria

- [ ] `identityRoots.mjs`: Emmy, Phoebe and Iris have `name` `'Emmy'`, `'Phoebe'` and `'Iris'`, each with a provenance comment citing its recorded assent. No entry or comment describes an operator confirmation of a name.
- [ ] No generator, template or doc in Brain describes an operator confirmation of a Social Name. A grep for `operator confirm` in a naming context finds nothing, and Eos's historical "operator-passed" provenance stays.
- [ ] "Authored by Emmy (…)" resolves to `@neo-gpt-emmy` through the roster. This is CI-covered by the updated `agentFamilyResolution.spec.mjs`.
- [ ] Affected unit specs pass.

## Out of Scope

- Participation activation (`reactivationTrigger: 'Operator confirms participation activation after first boot'`), which concerns a seat's status, not its name.
- Historical records: changelog rows and Discussion comments.
- The GitHub profile `name` fields, which already read Emmy, Phoebe and Iris.

## Related

neomjs/neo-agent-skills#134 · neomjs/neo#19348 · #693 (Sophie's root, merged with `name: 'Sophie'`) · #700 (auto-provisioned identities carry no family, adjacent)

Live latest-open sweep: the 8 most recent open Brain issues at 2026-10-01T15:40Z. No equivalent; #700 is adjacent (model family, not names).

Origin Session ID: 6b4062a3-941e-4b08-b997-765875a5b207


## Timeline

- 2026-10-01T15:40:47Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-01T15:40:48Z @neo-opus-vega added the `bug` label
- 2026-10-01T15:40:48Z @neo-opus-vega added the `documentation` label
- 2026-10-01T15:40:49Z @neo-opus-vega added the `ai` label
- 2026-10-01T15:45:04Z @neo-opus-vega cross-referenced by PR #702
- 2026-10-01T16:27:42Z @neo-gpt-emmy cross-referenced by PR #135
- 2026-10-01T17:27:28Z @tobiu referenced in commit `d106892` - "fix(graph): Emmy, Phoebe and Iris carry their assented Social Names (#701) (#702)"
- 2026-10-01T17:27:28Z @tobiu closed this issue

