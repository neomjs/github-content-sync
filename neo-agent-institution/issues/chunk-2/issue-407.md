---
id: 407
title: 'Accounts: pick the repositories a seat gets clones for'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-01T18:41:34Z'
updatedAt: '2026-10-01T21:08:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/407'
author: neo-opus-ada
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 402 Brain pin 7: the installed FM carries resident placement, repo sets and the seat-home guard'
blocking:
  - '[ ] 408 Accounts shows each repository''s last start outcome'
closedAt: '2026-10-01T21:08:37Z'
---
# Accounts: pick the repositories a seat gets clones for

## Context

The operator, 2026-09-30, on the first FM-launched peer (relayed on #245, comment 5913348117): *"this could be an FM enhancement: picking the repos (multiple ones) that a peer should get clones for."*

The Brain half has shipped. neomjs/neo-agent-brain#683 gives a seat other repositories (`metadata.repos`) and a `setRepos` verb, and clones each one with the seat's PAT before launch. #403 (Brain pin 7) brings it to the Institution. Every current peer works in several repositories: Ada's seat needs five.

## The Problem

The FM can set a seat's working repository (the add form's "Working repository", through `setRepo`), but nothing else. A seat's other repositories can only be set over the Fleet wire, so an operator can't give a seat the clones it needs.

## The Architectural Reality

- **Brain bridge:** `setRepos({id, repos})` (`ai/services/fleet/FleetControlBridge.mjs`, after `setRepo`) validates each entry with `setRepo`'s rules. It refuses a duplicate or the working slug, and `{id, repos: []}` clears the set. `listAgents()` returns the registry's definitions, `metadata` included.
- **Start outcome:** the start status carries `repos: [{repoSlug, state: 'prepared' | 'failed', reason?}]`. A failed entry's `reason` is credential-redacted and bounded.
- **Accounts card** (`AgentOS.view.fleet.detail.AgentConfigComponent`): it renders declared rows and sends edits as `configIntent` through the Panel to the bridge, as the harness and server chips do today.
- **In review:** neomjs/neo-agent-brain#710 / PR #711 adds a forge per repository. GitHub is the default; a GitLab entry carries an explicit clone URL and may name nested groups.

## The Fix

A "Repositories" card under the Accounts config card (placement B, see Intake):
- the working repository, plus the seat's other repositories as removable rows;
- an add field;
- each change sends one `setRepos` with the full list through the existing `configIntent` path. The Brain validates the list, and a refusal shows its reason on the card.
- ~~the last start's per-repository outcome beside each row~~ → #408.

**The pin this needs.** `setRepos` answers its refusals as data only from neomjs/neo-agent-brain#724 on, and #724 also changes `setRepo`'s answer, which the add form reads. So this ticket carries the Brain pin that brings #724 and the add form's adaptation to it (`AddAgentFlow.assignRepo` reads `{status, agent | reason}`), the way #245 carried pin 6. #724's review asked for that pairing to have an owner, and this is it (*revised 2026-10-01 ~19:55Z*).

## Acceptance Criteria

- [ ] AC-1: the card lists the working repository and the seat's other repositories from the definition.
- [ ] AC-2: adding or removing a repository sends one `setRepos` with the whole list. A refusal (invalid slug, duplicate, the working slug) shows the Brain's reason, and the list keeps what the registry holds.
- [ ] ~~AC-3: after a start, each repository shows its outcome, prepared or failed, with the redacted reason.~~ Moved to #408: the Brain returns a start's per-repository outcome only in the start's answer, and the cockpit drops it, so this needs a data path of its own.
- [ ] AC-4: the card fits the detail column at 1000×640 and 1280×800, with no clipped control. The goldens are re-captured at the 0.03 threshold.
- [ ] AC-5: the Brain pin moves to a `dev` commit that carries neomjs/neo-agent-brain#724, in its three places. `AddAgentFlow.assignRepo` reads `setRepo`'s new answer: an accepted working repository confirms as before, and a refused one names the Fleet's reason.

## Contract Ledger

The validation rules belong to the Brain (`FleetManager.setRepo` / `setRepos`, answered as data since neomjs/neo-agent-brain#724). This side only consumes them.

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `AgentDefinition` `metadata.repo` / `metadata.repos` (nested fields) | the registry's public definition (`listAgents`; written by `setRepo` / `setRepos`) | hydrated from `listAgents` and refreshed by every accepted readback (`record.set(agent)`); read as `record['metadata.repos']` | absent → `null` (no repositories) | the model's field comment | `configIntentRoundTrip.spec` (readback refreshes both fields); `reposCard.spec` rows arm |
| Repositories card `configIntent` `{id, repos}` → `ConfigIntentRoundTrip` → wire `setRepos` | neomjs/neo-agent-brain#724: `{status: 'accepted', agent}` / `{status: 'rejected', reason}` | the whole list per change, only `{id, repos}` on the wire; accepted → readback + "Configuration saved."; rejected → the Brain's reason, record untouched | no verb → "Configuration is unavailable in dev-server mode."; transport throw → "Could not save the configuration."; malformed answer → "Configuration response was invalid." (each "Nothing was changed.") | `AgentReposContainer`, `ConfigIntentRoundTrip` JSDoc | round-trip `setRepos` arms; e2e Repositories arm |
| Card owner token + save statuses | `ConfigIntentRoundTrip`'s supersession contract (per-record generations) | the card is its own owner with its own status map: a race with the configuration card paints `superseded`, never a latched `pending` | — | Panel JSDoc | `Accounts.spec` wiring arm |
| Working row; no working repository | `FleetManager.setRepos` refuses others without one | working repository first, tagged, no Remove; without one, the empty line shows and the add row is disabled | — | card JSDoc | `reposCard.spec` arms |
| `AddAgentFlow.assignRepo` reading `setRepo` (AC-5) | #724's `setRepo` answer at Brain pin `e6f0bb0` | accepted → the confirmed definition; rejected → "Agent added, but its working repository is not set: <Fleet reason>" | throw or no verb → "… set <slug> before starting it." | `AddAgentFlow` JSDoc | `addAgentFlow.spec` arms; e2e add-form journey |

## Out of Scope

- Per-harness write access to the extra checkouts, for example Codex's sandbox and OpenCode's `allowedPaths`. That's #682's recorded boundary, to be measured per harness.
- GitLab entries, which follow neomjs/neo-agent-brain#711 in its own leaf.
- Choosing the working repository after creation. `setRepo` stays with the add form; only its answer's reading changes (AC-5).

## Intake (2026-10-01)

Self-authored this session, so only stage 2 runs (the prescription):
- **Prescription checked:** `apps/agentos/view/fleet/detail/AgentConfigComponent.mjs` owns the concern: the card's declared rows and its `configIntent`. `apps/agentos/view/accounts/Panel.mjs#onAgentConfigIntent` owns the bridge round-trip.
- **What the Fix implies on top of that, both measured in code:**
  - The round-trip today is the one `configureAgent` verb, but repositories travel by `setRepos`, so the Panel's intent handling routes a `repos` intent to that second verb.
  - `AgentOS.model.AgentDefinition` has no repository field, and nothing in the Institution maps `metadata.repo` / `metadata.repos` yet, so the definitions read has to carry them into the record.
- **The record shape:** a nested `metadata` field with `repo` and `repos` children. A field `mapping` won't do, because `RecordFactory` applies mappings only to initial values, so the round-trip's `record.set(agent)` readback would keep stale repositories. Nested fields cover both the `listAgents` hydration and the readback.
- **The refusal path — decided: the Brain answers refusals as data (neomjs/neo-agent-brain#721).**
  - Measured at Brain `dev@5041af0`: a duplicate, the working repository and a malformed slug all reach the wire as `operation-failed` ("fleet: 'setRepos' failed"), so the card could never show the reason.
  - The wire contract puts domain outcomes inside `result` (`src/fleet/contract/wire.mjs:48-50`), and `configureAgent` already answers `{status: 'rejected', reason}`. #721 gives `setRepos` that shape, and the round-trip treats both verbs alike.
  - Validating in the Institution instead was rejected: it would copy the Brain's slug rule. The add form's copy has already drifted from `assertRepoSlug` (owner characters, reserved owners).
- **Placement — decided: B.** A sibling "Repositories" card under the config card, at the same 28rem measure (Grace, A2A 2026-10-01 18:52Z). Adding a repository needs a form field, and the config card is a vdom render without one.
- **Blocked by:** #403 (pin 7 carries `setRepos`) and neomjs/neo-agent-brain#721, plus the pin that carries #721.

## Related

#245 · #408 (the start outcome, split from AC-3) · #403 (blocks: pin 7 carries `setRepos`) · neomjs/neo-agent-brain#682 / #683 · neomjs/neo-agent-brain#710 / #711 · neomjs/neo-agent-brain#571

## Sweeps

- Live latest-open sweep: the latest 20 open Institution issues, plus searches for "setRepos", "repo-set picker", "repository set" and "seat repositories". No equivalent.
- Memory Core: the multi-repo design discussion lives in #245's comments (Emmy, 09-30) and #682. There is no picker decision.
- A2A: no claim. Grace's #710 is the Brain forge model, adjacent.
- Own assignments: #402 and #404 (open, not overlapping).

Origin Session ID: 84a3bf84-c9cb-4215-818a-d9640f49669a

Authored by Ada (Claude Opus 5.5, Claude Code).



## Timeline

- 2026-10-01T18:41:34Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-01T18:41:35Z @neo-opus-ada added the `enhancement` label
- 2026-10-01T18:41:36Z @neo-opus-ada added the `agent-os` label
- 2026-10-01T18:41:36Z @neo-opus-ada added the `ai` label
- 2026-10-01T18:41:40Z @neo-opus-ada marked this issue as being blocked by #402
- 2026-10-01T18:54:48Z @neo-opus-ada cross-referenced by #721
- 2026-10-01T19:03:11Z @neo-opus-ada cross-referenced by PR #724
- 2026-10-01T19:08:21Z @neo-opus-ada cross-referenced by #408
- 2026-10-01T19:08:31Z @neo-opus-ada marked this issue as blocking #408
- 2026-10-01T19:23:40Z @neo-opus-ada cross-referenced by #409
- 2026-10-01T20:18:38Z @neo-opus-ada referenced in commit `c734ec3` - "feat(agentos): an Accounts Repositories card picks the repositories a seat gets clones for (#407)

The definition model carries metadata.repo / metadata.repos as nested fields, the configure round-trip routes a repos intent to setRepos, and a Repositories card under the configuration card lists the working repository and adds or removes the others as one new list. Skin and specs follow."
- 2026-10-01T20:18:38Z @neo-opus-ada referenced in commit `f1083ce` - "feat(agentos): the Repositories card's skin and specs, and its Container suffix (#407)

The card extends container.Base, so its class takes the Container suffix the view-topology law asks for. Skin: the configuration card's shell and status vocabulary, rows in RepositoryList.scss, the shared 28rem measure. Specs: the card over real records, the setRepos route of the round-trip, and the Panel scoping both cards with separate save status."
- 2026-10-01T20:18:39Z @neo-opus-ada referenced in commit `20f0526` - "feat(agentos): the Repositories card's rows and action take the cockpit's skin, and the Accounts goldens show it (#407)

The rows zero the engine's list-item fills, so they sit on the card surface. The Add action takes the add form's signal treatment at the cockpit's 32px action height. The sample definitions carry each repository shape (a working repository with two others, one alone, none), the 1000x640 and 1280x800 geometry census measures the new card, and the three Accounts goldens and the baseline stamp are re-taken."
- 2026-10-01T20:18:39Z @neo-opus-ada referenced in commit `2bef5c9` - "feat(agentos): the add form shows the Fleet's reason for a refused working repository, and the Repositories card offers no add without one (#407)

AddAgentFlow.assignRepo reads setRepo's {status, agent | reason} answer (neomjs/neo-agent-brain#724), so a refused working repository names the Fleet's reason. The Repositories card disables its add row while the seat has no working repository, which the Fleet requires first. The Accounts e2e gains a Repositories arm over the real Fleet wire: add, a refused duplicate with the Fleet's reason, remove."
- 2026-10-01T20:18:39Z @neo-opus-ada referenced in commit `1a23a68` - "chore(deps): Brain pin 9 (dev@e6f0bb0) carries setRepo / setRepos answering refusals as data (#407)

The pin moves in its three places to Brain dev@e6f0bb0: neomjs/neo-agent-brain#724 (the Repositories card and AddAgentFlow read its {status, agent | reason} answer), plus #723 (the Wake routes pane's arming reason), #707 (the placement probe) and #726 (a receiver spec flake). Nothing under the Brain's src/ or package.json changed since 92122a0."
- 2026-10-01T20:18:39Z @neo-opus-ada referenced in commit `1ddac1c` - "test(agentos): the visual baseline stamp follows the add flow and the Repositories card (#407)

AddAgentFlow.mjs and AgentReposContainer.mjs are stamped style inputs. The Accounts goldens hold at this head, so only the stamp moves."
- 2026-10-01T20:18:41Z @neo-opus-ada cross-referenced by PR #412
- 2026-10-01T21:08:37Z @tobiu referenced in commit `83e8ca3` - "feat(agentos): Accounts picks the repositories a seat gets clones for (#407) (#412)

* feat(agentos): an Accounts Repositories card picks the repositories a seat gets clones for (#407)

The definition model carries metadata.repo / metadata.repos as nested fields, the configure round-trip routes a repos intent to setRepos, and a Repositories card under the configuration card lists the working repository and adds or removes the others as one new list. Skin and specs follow.

* feat(agentos): the Repositories card's skin and specs, and its Container suffix (#407)

The card extends container.Base, so its class takes the Container suffix the view-topology law asks for. Skin: the configuration card's shell and status vocabulary, rows in RepositoryList.scss, the shared 28rem measure. Specs: the card over real records, the setRepos route of the round-trip, and the Panel scoping both cards with separate save status.

* feat(agentos): the Repositories card's rows and action take the cockpit's skin, and the Accounts goldens show it (#407)

The rows zero the engine's list-item fills, so they sit on the card surface. The Add action takes the add form's signal treatment at the cockpit's 32px action height. The sample definitions carry each repository shape (a working repository with two others, one alone, none), the 1000x640 and 1280x800 geometry census measures the new card, and the three Accounts goldens and the baseline stamp are re-taken.

* feat(agentos): the add form shows the Fleet's reason for a refused working repository, and the Repositories card offers no add without one (#407)

AddAgentFlow.assignRepo reads setRepo's {status, agent | reason} answer (neomjs/neo-agent-brain#724), so a refused working repository names the Fleet's reason. The Repositories card disables its add row while the seat has no working repository, which the Fleet requires first. The Accounts e2e gains a Repositories arm over the real Fleet wire: add, a refused duplicate with the Fleet's reason, remove.

* chore(deps): Brain pin 9 (dev@e6f0bb0) carries setRepo / setRepos answering refusals as data (#407)

The pin moves in its three places to Brain dev@e6f0bb0: neomjs/neo-agent-brain#724 (the Repositories card and AddAgentFlow read its {status, agent | reason} answer), plus #723 (the Wake routes pane's arming reason), #707 (the placement probe) and #726 (a receiver spec flake). Nothing under the Brain's src/ or package.json changed since 92122a0.

* test(agentos): the visual baseline stamp follows the add flow and the Repositories card (#407)

AddAgentFlow.mjs and AgentReposContainer.mjs are stamped style inputs. The Accounts goldens hold at this head, so only the stamp moves."
- 2026-10-01T21:08:37Z @tobiu closed this issue

