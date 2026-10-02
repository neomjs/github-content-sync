---
id: 408
title: Accounts shows each repository's last start outcome
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-01T19:08:19Z'
updatedAt: '2026-10-02T09:43:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/408'
author: neo-opus-ada
commentsCount: 1
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 407 Accounts: pick the repositories a seat gets clones for'
blocking: []
closedAt: '2026-10-02T09:42:40Z'
---
# Accounts shows each repository's last start outcome

## Context

#407 (the Accounts repo picker) carried an AC-3: after a start, each repository shows its outcome, prepared or failed, with the redacted reason. Building #407 showed there is no data path for it, so it moves here and #407 ships the picker without it.

## The Problem

The Brain's start answers `repos: [{repoSlug, state: 'prepared' | 'failed', reason?}]` (neomjs/neo-agent-brain `ai/services/fleet/startAgentProvisioned.mjs`, the loop at :280-291 and the status at :351). It keeps that answer nowhere else: no roster row, runtime status or repo inspection describes a seat's other repositories.

The Institution receives the answer in the cockpit. `FleetLifecycleIntentAdapter.handleFleetLifecycleIntent` clears the roster record's `pendingAction` / `controlReason` and returns the result to `view/fleet/cockpit/Controller.mjs`. Neither caller reads `repos`: one card's Start keeps only `ok`, and Start-all's summary counts outcomes. So the answer is dropped where it arrives.

## The Architectural Reality

- **Roster:** `AgentOS.model.FleetAgent` records in the Viewport provider's `fleetRoster` store. Each roster read reconciles in place (`LivenessController.reconcileRoster`, `record.set(row)`), so a Body-only field survives a read but not an app reload.
- **Accounts:** `AgentOS.model.AgentDefinition` records in `agentDefinitions`. #407 adds `metadata.repo` / `metadata.repos` there, the declared set, not its state on disk.
- **The working repository** already has a durable state source: the roster DTO's `repoStatus`, from the Brain's `inspectFleetRepos`. Nothing equivalent covers the other repositories.

## The Fix (the data path is the decision; recommendation first)

- **A (recommended):** the Brain reports each declared repository beside the working one on the roster's repo source (prepared on disk or not), and keeps the last start's failure reason for a repository that failed. The card then reads durable truth on every roster read and after a reload. The Institution maps it as it maps `repoStatus`.
- **B:** the cockpit writes the start answer's `repos` onto the roster record. This is smaller and survives roster reads, but it is lost on an app reload, and a start from outside this app never shows.

Intake decides between them with a measurement of the roster read path.

*Decided 2026-10-01 ~20:30Z: A, as the launch record.* The Brain half is neomjs/neo-agent-brain#730: `setRepoOutcomes` beside `setWakeRoute`, reported by `status(id)` and carried on the roster row. This ticket then maps the roster row's outcome onto the Repositories card (#412), and is blocked by #730.

*Design note, 2026-10-01 ~19:40Z:* the two halves of A have different sources.
- **The on-disk half** extends `inspectFleetRepos`. Today it inspects only `metadata.repo` (neomjs/neo-agent-brain `ai/services/fleet/inspectFleetRepos.mjs`), and can inspect each `metadata.repos` entry the same way.
- **The failure reason** exists only in the start's answer. neomjs/neo-agent-brain#705 set the precedent for keeping a launch's outcome: `FleetManager.armSeatWake` records the wake route on the lifecycle record with `lifecycle.setWakeRoute(agentId, route, {pid, startedAt})`, bound to that launch. A start's per-repository outcome can ride the same record and reach the roster DTO the same way.

## Contract Ledger

*Added 2026-10-02 with the build, at the Brain pin dev@cbd11cb that carries neomjs/neo-agent-brain#731.*

| Target surface | Source of authority | Behavior | Failure / fallback | Evidence |
|---|---|---|---|---|
| Roster row `repoOutcomes` | Brain `fleetCockpitStatus.mjs`, fed by `FleetLifecycleService.setRepoOutcomes` through `status(id).repos` (neomjs/neo-agent-brain#731) | `[{repoSlug, state: 'prepared' \| 'failed', reason?}]`, reasons redacted where they are written | `null` until a start covers other repositories | Brain unit arms on #731 |
| `FleetAgent.repoOutcomes` | `util/RosterRow.mjs` `mapRosterRow`, `model/FleetAgent.mjs` | The array passes onto the record whole | `null` when the row carries none | the `mapRosterRow` arm, `rosterStore.spec` |
| Repositories card rows | `view/fleet/detail/AgentReposContainer.mjs`, `RepositoryList.mjs` | Each other repository gets a `Prepared` or `Failed` pill titled *At the last start*, and a failed one shows its reason. The working repository keeps `repoStatus` (`Working`). | No pill when the last start's outcome set has no entry for the slug, or for an unknown `state`. A slug removed and re-added before the next start keeps that start's entry. | `reposCard.spec` arms, the two Accounts goldens |
| Card heading | `AgentReposContainer` | `Repositories · declared · last start` once any row has an outcome | `Repositories · declared` otherwise | goldens |
| Lifetime | the Fleet owner process, which holds the launch record | A roster read or a cockpit reload reads it again | A Fleet restart drops it. A seat re-adopted from its lease shows none until its next start. | the "a roster that arrives later fills it" arm |

## Acceptance Criteria

- [x] AC-1: after a start, the Repositories card shows each of the seat's other repositories as prepared or failed, and a failed one shows its redacted reason.
- [x] AC-2: the outcome is still shown after a roster read and after an app reload.
- [x] AC-3: a repository the last start did not cover shows no outcome until the next start. One removed and re-added since keeps that start's outcome, because the outcome records the start, not the current list. *(Narrowed 2026-10-02 at @neo-gpt-sophie's #423 contract read: it first said "added after the last start", which a re-added slug is not.)*

## Post-Merge Validation

- [ ] On an installed Fleet Manager carrying Brain pin 10 or later (#12's next package), start a seat whose other repositories include one that clones and one that cannot.
  - Accounts shows `Prepared` and `Failed`, and the failed row shows its redacted reason.
  - A cockpit reload keeps both.
  - A Fleet restart clears them until the seat's next start.

## Out of Scope

- The picker itself (#407).
- The working repository, which `repoStatus` already covers.

## Related

#407 (blocks: the Repositories card) · #245 · neomjs/neo-agent-brain#682 / #683 (the start's `repos`)

## Sweeps

- Live latest-open sweep: the latest 20 open Institution issues at 2026-10-01T19:07:45Z. No equivalent; #407 is the source of this split.
- Keyword: "start outcome repository" across neomjs. No match.
- A2A: the last 12 messages, all read-states. No claim on start outcomes.
- MC sweep: one query on the problem's nouns (a start's per-repository outcome dropped by the cockpit), 6 results, no prior decision found.
- Own-assignment sweep: 3 open (#402, #404, #407). #407 is the parent of this split; the others don't overlap.

Origin Session ID: 84a3bf84-c9cb-4215-818a-d9640f49669a

Authored by Ada (Claude Opus 5.5, Claude Code).






## Timeline

- 2026-10-01T19:08:21Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-01T19:08:21Z @neo-opus-ada added the `enhancement` label
- 2026-10-01T19:08:22Z @neo-opus-ada added the `agent-os` label
- 2026-10-01T19:08:22Z @neo-opus-ada added the `ai` label
- 2026-10-01T19:08:31Z @neo-opus-ada marked this issue as being blocked by #407
- 2026-10-01T19:08:52Z @neo-opus-ada cross-referenced by #407
- 2026-10-01T19:23:40Z @neo-opus-ada cross-referenced by #409
- 2026-10-01T20:18:41Z @neo-opus-ada cross-referenced by PR #412
- 2026-10-01T20:27:14Z @neo-opus-ada cross-referenced by #730
- 2026-10-01T20:32:22Z @neo-opus-ada cross-referenced by PR #731
### @neo-opus-ada - 2026-10-01T21:05:28Z

**Handover (session sunset, 2026-10-01 ~21:15Z) · owner: @neo-opus-ada**

The Institution half is built and pushed: branch `ada/408-repo-outcomes` at `08f3047`, stacked on #412 (approved, awaiting merge). There is no PR yet.

- `9df1045`: the roster row's `repoOutcomes` maps onto `FleetAgent.repoOutcomes` (`RosterRow.mapRosterRow`). The Repositories card joins it by slug from the provider's `fleetRoster` Store and never writes it. Each other repository shows `Prepared`, or `Failed` with the Fleet's redacted reason on its own line, and one added since the last start shows none (AC-3). The heading reads "Repositories · declared · last start" whenever an outcome shows. The Panel binds `stores.fleetRoster` and hands it to the card, as it hands the tenant Store to the config card. The Viewport's sibling rule now reads "writes or locally maps" instead of "reaches into".
- `08f3047`: the sample roster gives Ada one prepared and one failed repository. `accounts-config-surface` and `accounts-1280x800` are re-captured at 0.03 and the stamp follows. 1000×640 doesn't change, because the card sits below its fold.

Evidence at `08f3047`:
- unit 1162/1162, components 4/4, e2e 11/11;
- `check-visual-baselines` matches, and `check-app-file-sizes` has 0 files over the bar;
- ticket archaeology: 0 across 19 files;
- red-first: without the source, the 7 new or changed arms fail and the other 60 pass.

**Pickup, in order:**
1. After #412 merges: `git -C <institution clone> rebase --onto origin/dev 1ddac1c ada/408-repo-outcomes`, which drops #412's commits. The goldens and the stamp conflict by design: rebuild themes, re-capture the two Accounts goldens, re-stamp. Re-capture again if #413 lands first.
2. After neomjs/neo-agent-brain#731 merges: move the Brain pin to a `dev` commit that carries it, in its three places, with the full pin-move checklist. ⚠️ If neomjs/neo-agent-brain#728 is on Brain `dev` by then, #413 must merge before that pin (Vega's sequencing).
3. Open the PR that resolves this ticket, with a GPT primary reviewer. AC-2's "app reload" rests on the Fleet owner process keeping the record; a Fleet restart drops it (neomjs/neo-agent-brain#730's Contract Ledger).

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-10-02T08:22:02Z @neo-fable cross-referenced by #391
- 2026-10-02T08:52:45Z @neo-fable cross-referenced by #418
- 2026-10-02T08:57:51Z @neo-opus-ada referenced in commit `620f3e5` - "feat(agentos): the Repositories card shows each repository's last start outcome (#408)

The roster row's repoOutcomes (neomjs/neo-agent-brain#731) maps onto FleetAgent. The card joins
it by slug, reading the provider's fleetRoster Store and never writing it: prepared, or failed
with the Fleet's redacted reason. The heading names the last start whenever an outcome shows."
- 2026-10-02T08:57:52Z @neo-opus-ada referenced in commit `3ea5228` - "test(agentos): the fleet sample's seat carries a prepared and a failed repository outcome (#408)"
- 2026-10-02T08:57:52Z @neo-opus-ada referenced in commit `cac53e8` - "chore(deps): Brain pin 10 (dev@cbd11cb) carries the start's per-repository outcome on the roster row (#408)"
- 2026-10-02T08:57:52Z @neo-opus-ada referenced in commit `1b9afbc` - "test(agentos): the Accounts goldens show a prepared and a failed repository, re-captured over #413 (#408)"
- 2026-10-02T09:00:59Z @neo-opus-ada cross-referenced by PR #423
- 2026-10-02T09:05:31Z @neo-opus-ada cross-referenced by #424
- 2026-10-02T09:31:58Z @neo-opus-ada referenced in commit `214d239` - "feat(agentos): the Repositories card shows each repository's last start outcome (#408)

The roster row's repoOutcomes (neomjs/neo-agent-brain#731) maps onto FleetAgent. The card joins
it by slug, reading the provider's fleetRoster Store and never writing it: prepared, or failed
with the Fleet's redacted reason. The heading names the last start whenever an outcome shows."
- 2026-10-02T09:31:58Z @neo-opus-ada referenced in commit `1fee678` - "test(agentos): the fleet sample's seat carries a prepared and a failed repository outcome (#408)"
- 2026-10-02T09:31:58Z @neo-opus-ada referenced in commit `bec7836` - "chore(deps): Brain pin 10 (dev@cbd11cb) carries the start's per-repository outcome on the roster row (#408)"
- 2026-10-02T09:31:59Z @neo-opus-ada referenced in commit `6e7a1e7` - "test(agentos): the Accounts goldens show a prepared and a failed repository, re-captured over #413 (#408)"
- 2026-10-02T09:31:59Z @neo-opus-ada referenced in commit `0341fb8` - "test(agentos): an outcome records the last start, so a repository removed and re-added before the next one keeps it (#408)"
- 2026-10-02T09:36:15Z @neo-gpt-emmy cross-referenced by #430
- 2026-10-02T09:42:40Z @tobiu referenced in commit `7b031c3` - "feat(agentos): the Repositories card shows each repository's last start outcome (#408) (#423)

* feat(agentos): the Repositories card shows each repository's last start outcome (#408)

The roster row's repoOutcomes (neomjs/neo-agent-brain#731) maps onto FleetAgent. The card joins
it by slug, reading the provider's fleetRoster Store and never writing it: prepared, or failed
with the Fleet's redacted reason. The heading names the last start whenever an outcome shows.

* test(agentos): the fleet sample's seat carries a prepared and a failed repository outcome (#408)

* chore(deps): Brain pin 10 (dev@cbd11cb) carries the start's per-repository outcome on the roster row (#408)

* test(agentos): the Accounts goldens show a prepared and a failed repository, re-captured over #413 (#408)

* test(agentos): an outcome records the last start, so a repository removed and re-added before the next one keeps it (#408)"
- 2026-10-02T09:42:40Z @tobiu closed this issue
- 2026-10-02T11:20:41Z @neo-fable-clio cross-referenced by PR #433

