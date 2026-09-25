---
id: 223
title: 'The plane record stores no identity, so a Finder launch cannot attach'
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-09-25T21:32:43Z'
updatedAt: '2026-09-25T21:59:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/223'
author: neo-opus-ada
commentsCount: 0
parentIssue: 7
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-25T21:59:33Z'
---
# The plane record stores no identity, so a Finder launch cannot attach

## Context

#211 gave the packaged shell its own plane record (#212). On the team machine tonight, the record cannot produce a working attach. In plane mode, the fleet child binds its viewer from an identity *claim*, and the plane proves that claim against the bearer. The record hands the child a base and a bearer, but never the claim. So the claim still comes from the launch environment, which #211 set out to stop depending on. There are two outcomes, one per launch path:

- **Finder or Dock launch: refused.** I ran the installed app's own bundled `resolveFleetViewerClaim` (the `organism/node_modules/neo-agent-brain` copy inside the `.app`, on the bundled Electron as Node) under a launchd-like environment. Results:
  - PATH `/usr/bin:/bin:/usr/sbin:/sbin`, no `NEO_AGENT_IDENTITY`: it throws `[fleet] startup refused: no viewer identity resolved`.
  - Control 1, the same run with `/opt/homebrew/bin` on PATH: `@tobiu` via gh-cli.
  - Control 2, with `NEO_AGENT_IDENTITY=tobiu`: `@tobiu` via env-var.

  `launchctl getenv` holds neither `PATH` nor `NEO_AGENT_IDENTITY` on this machine.
- **A shell another process launched: the launcher's identity.** The shell on the operator's screen (started 19:53Z by a peer's session) carries that session's `NEO_AGENT_IDENTITY=neo-fable-clio`. If the operator connects with their own PAT, the child claims `@neo-fable-clio` against a bearer whose subject is theirs. The expected outcome is `plane identity mismatch`, and then exit 1. That half is inferred from the code below, not observed: a run needs the operator's credential. It also assumes the self-relaunch after connect keeps the environment.

Either way, the cockpit shows today's generic `boot-not-ready` / "fleet: Brain is not ready" over the sample roster.

## The Problem

The claim and the bearer are a pair: the plane admits only when the bearer's subject *is* the claim. The record stores one half and leaves the other to whatever the environment holds. That environment is empty from Finder and foreign when an agent launched the app. The connect-time probe cannot notice, because it proves only that the endpoint is a Memory Core that accepts the bearer, not whose bearer it is.

## The Architectural Reality

- **Design authority:** #211, "The packaged shell attaches to a plane from its own first-run config, not from environment variables". [ADR 0034](https://github.com/neomjs/neo/blob/dev/learn/agentos/decisions/0034-electron-shell-architecture.md) as amended by ADR 0038: "identity, registry, credential, and authorization policy are plane-owned". The claim is the one plane-attach input still read from the environment.
- `harness/planeConfig.mjs`:
  - `readPlaneConfig` (:71) returns `{planeBase, bearer}`.
  - `writePlaneConfig` (:104) writes `{planeBase}` to `plane.json`.
  - `planeEnvFragment` (:144) exports `NEO_FLEET_PLANE_BASE` and `NEO_FLEET_PLANE_BEARER` only.
  - `probePlaneCredential` (:200) returns a verdict after `initialize`, then deletes the session.
  - `createPlaneBroker`'s `attachPlane` runs probe → `writePlaneConfig` → relaunch.
- `harness/main.mjs` `bootProductBrain` merges the fragment into the packaged env. Children inherit `process.env` (`brain.mjs`, spawn env), so an inherited `NEO_AGENT_IDENTITY` reaches the fleet child unless the fragment overrides it.
- Brain, on the child side (no change here):
  - `devFleetServer.mjs` plane decision: `resolveFleetViewerClaim()` (env-var → `gh api user`), then `planeClient.init({expectedIdentity})`, then exit 1 on refusal.
  - `planeMailboxClient.mjs` `init`: `list_permissions` → `readMcpToolResultPayload(result).identity` → `'plane identity mismatch'` when it differs.
  - `PermissionService` answers `list_permissions` with `{identity, capabilities, grantedToOthers}`.
- The seam's documented shape is ADR 0034 §2.3 "Transitional seam": base and bearer, "a value already set in the process env wins".

## The Fix

The identity travels with the bearer.

1. **Probe.** After the authenticated `initialize` names `neo-memory-core`, send `notifications/initialized` in the same session, then `tools/call` → `list_permissions`. Read `identity` the way the Brain's client does (`structuredContent`, else the text item's JSON), normalized to the canonical `@login`, and then delete the session.
   - Returns `{verdict: 'accepted', identity}`.
   - An admitted bearer with no canonical identity refuses as `'no-identity'`.
2. **Record.** `writePlaneConfig` stores `identity` in `plane.json` beside the base. It is not a secret and not encrypted. `readPlaneConfig` returns it; a missing or invalid one reads `null`.
3. **Seam.** When `planeEnvFragment` exports the record's bearer, it also exports `NEO_AGENT_IDENTITY` from the record, over an inherited value, because the claim must be the bearer's subject.
   - With a bearer from the environment, the fragment exports no identity, and the environment's pair stands.
   - A record without an identity exports none, which keeps today's behavior.
4. **Card.** `PlaneSetupPanel.reasonText` gains `'no-identity'`: "That plane accepted the credential but named no identity for it. Nothing was stored."

Every boot still re-proves the pair: the child's `init` checks the recorded claim against the stored bearer. So a bearer whose subject changes after connect refuses rather than re-attributing.

## Contract Ledger

| Surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `probePlaneCredential` return (`planeConfig.mjs:200`) | ADR 0034 §2.3 (one authenticated check) | `{verdict, identity}`, plus the new `'no-identity'` verdict | Any failure after an admitted `initialize` → `'no-identity'`, nothing stored | JSDoc | Unit arms on a fake fetch |
| `plane.json` record | ADR 0038 §2.1 client connection profile | `{planeBase, identity}` | Legacy record without `identity` → `identity: null` | `writePlaneConfig` / `readPlaneConfig` JSDoc | Unit arms |
| `planeEnvFragment` output (`:144`) | ADR 0034 §2.3 transitional seam (amended) | Adds `NEO_AGENT_IDENTITY` exactly when it exports the record's bearer, overriding inherited | No record identity, or an env bearer → no identity key | JSDoc; ADR 0034 §2.3 | Red-first arm |
| `attachPlane` refusal reasons → `PlaneSetupPanel.reasonText` | ADR 0034 §2.3 reply shape `{ok, reason, relaunching}` | Gains `'no-identity'` | n/a | Panel JSDoc | Unit arm |

## Decision Record impact

`amends ADR 0034` §2.3 "Transitional seam": the seam carries the identity the plane named for the stored bearer, and that identity follows the bearer, not the environment. The amendment lands in neo as its own leaf, first or alongside.

## Acceptance Criteria

- [ ] AC-1: The probe reads the bearer's identity from `list_permissions` in the admitted session and returns it with `'accepted'`. A plane that names none refuses with `'no-identity'`, and nothing is stored.
- [ ] AC-2: The record stores and returns the identity. A record written before this change reads `identity: null`.
- [ ] AC-3: `planeEnvFragment` exports the record's identity with the record's bearer, over an inherited `NEO_AGENT_IDENTITY`. It exports none with an environment bearer or an identity-less record. This arm is red-first on `dev`.
- [ ] AC-4: Composition receipt in the PR body. Under a launchd-like environment (no `gh` on PATH, no identity), the fragment's env run through the Brain's `resolveFleetViewerClaim` resolves the recorded identity; the same run without the fragment refuses.
- [ ] AC-5 (post-merge, operator's machine): a Finder-launched packaged shell connects through the card and boots `plane-attach`. The cockpit's grid connection reads live, not `boot-not-ready`.

## Out of Scope

- Naming a plane-mode refusal in the banner. The child's "plane mode refused" and "no viewer identity" still read as "Brain is not ready"; that is #15's auth-refused state.
- Showing "connected as" on the card.
- The Brain's claim contract.

## Avoided Traps

- **Loading the login shell's PATH so `gh` resolves.** `gh`'s account is machine-global, not the bearer's subject; a stranger's machine may have no `gh`; and it still hands the claim to the environment.
- **Asking the user to type their handle on the card.** The plane already knows the bearer's subject, so a typed handle can only disagree.
- **Letting the Brain trust the bearer's subject and drop the claim.** That removes the fail-closed re-attribution guard `devFleetServer.mjs` chose on purpose. A recorded claim keeps it meaningful across boots.

## Related

Parent #7. #211 (the attach, closed by #212). #221/#222 (the beside-plane banner, whose Connect leads here). #15 (naming plane refusals). #214 (the fixture-plane smoke, which should exercise this pair). neomjs/neo#17310 / neomjs/neo#17348 (operator-seat conflation, the neighbouring identity-truth leaf).

Live latest-open sweep: checked the latest 20 open issues in neo-agent-institution, neo-agent-brain and neo at 2026-09-25T21:30:16Z, plus `gh search issues --owner neomjs "viewer identity plane"`; no equivalent.
A2A in-flight sweep: last 30 messages, no claim on the plane identity.
MC sweep: "packaged shell Finder launch no viewer identity resolved NEO_AGENT_IDENTITY plane identity mismatch fleet child refused", 6 results. No prior decision found; the neighbours are #211's filing and the conflation leaf.
Own-assignment sweep: 2 open (#219, #221), none overlapping.

Origin Session ID: 0f80515e-7682-4313-8101-b926da48c55c
Retrieval Hint: `query_raw_memories("plane record viewer identity NEO_AGENT_IDENTITY Finder launch list_permissions planeEnvFragment")`


## Timeline

- 2026-09-25T21:32:45Z @neo-opus-ada added the `bug` label
- 2026-09-25T21:32:45Z @neo-opus-ada added the `ai` label
- 2026-09-25T21:32:47Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-25T21:32:56Z @neo-opus-ada added parent issue #7
- 2026-09-25T21:34:02Z @neo-opus-ada cross-referenced by #19233
- 2026-09-25T21:41:44Z @neo-opus-ada cross-referenced by PR #224
- 2026-09-25T21:43:12Z @neo-opus-ada cross-referenced by PR #19234
- 2026-09-25T21:54:44Z @neo-opus-ada referenced in commit `1745324` - "fix(shell): a plane record without its identity is no record, so the card offers to attach again (#223)

A record stored before the identity was recorded (by today's app, e.g.
through the interim NEO_AGENT_IDENTITY launch) exported its bearer with
no identity. From Finder the child then claims nothing and is refused.
status() also called the record configured, so the card never came back.

It now follows the rule for a bearer that no longer decrypts. The
fragment exports nothing for it, so the shell boots on its own and,
beside a running plane, offers Connect. status() reads it as
unconfigured, so the card mounts. One re-attach writes the record whole.
From Eos's review of neo#19234."
- 2026-09-25T21:59:33Z @tobiu referenced in commit `7bbdcd5` - "Merge pull request #224 from neomjs/ada/223-plane-identity

fix(shell): the plane record carries its bearer's identity, so a Finder launch can attach (#223)"
- 2026-09-25T21:59:33Z @tobiu closed this issue
- 2026-09-25T22:08:39Z @neo-opus-ada cross-referenced by #225

