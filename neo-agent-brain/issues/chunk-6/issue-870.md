---
id: 870
title: Adding a seat over an unreadable credentials.enc erases the other PATs
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-04T21:00:30Z'
updatedAt: '2026-10-04T21:00:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/870'
author: neo-opus-ada
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
# Adding a seat over an unreadable credentials.enc erases the other PATs

## Context

Found during #815's intake on 2026-10-04 and measured with a throwaway probe: a seat defined while `credentials.enc` cannot be decrypted is added without an error, and every other seat's stored PAT is gone. #571's move re-adds the eight seats defined before #548 through Add Agent, one `defineAgent` each. If the running Fleet cannot read the store during that window, every PAT the operator entered is lost.

Observed at Brain dev `cf376f89`: define `alice`, and the store holds `[alice]`. Overwrite `credentials.enc` with bytes that do not decrypt, then define `bob`: no error, the store holds `[bob]`, and `resolveCredential('alice')` answers `null`.

## The Problem

- `FleetRegistryService.readCredentials()` answers an empty map for a store it cannot decrypt ("Credential store unreadable; failing closed"). That is right for a read: `resolveCredential` answers `null`, and Start refuses.
- `defineAgent` uses the same read as its write snapshot (`previousCredentials`) and publishes `{...previousCredentials, [agentId]: credential}`. Over an unreadable store, this replaces every other seat's ciphertext with a one-entry map. Its rollback writes the same empty snapshot. `storeCredential` (private, no caller today) has the same shape.
- How one machine gets an unreadable store: the key is `NEO_FLEET_SECRET_KEY` when set, else a key file generated in the data folder (`resolveFleetCredentialKey`). Two processes over one data folder with different keys (one with the env key, one with the file) make each other's store undecryptable. So does a data folder copied without its key file, because a fresh key is generated.

## The Architectural Reality

- Design authority: the registry's own read contract, "Credential store unreadable; failing closed" (`readCredentials`), and its sibling's mutation contract, `FleetTenantService.readEncryptedRecord`: "Missing is empty; existing unreadable/wrong-key/non-record ciphertext throws and must remain byte-identical." The registry has the read half and lacks the mutation half.
- `FleetTenantService` splits the two: a lenient `readCredentials()` that delegates, and a strict `readCredentialsForMutation()` that `connectTenant` takes before any write.
- `FleetControlBridge.defineAgent` turns a `FleetRegistryService.defineAgent:` error into `{status: 'rejected', reason}`, which Add Agent renders.
- `removeCredential` writes only when the id is present, so over an unreadable store it already writes nothing.
- Owning folder: `ai/services/fleet` (structure map, 2026-10-04).

## The Fix

1. `FleetRegistryService.readCredentialsForMutation()`: a missing file is an empty map. An existing file that does not decrypt, or does not hold a record, throws and stays byte-identical. `readCredentials()` delegates to it and keeps its fail-closed empty answer for reads.
2. `defineAgent` and `storeCredential` take their snapshot from the strict read before writing anything. A define over an unreadable store refuses with a `FleetRegistryService.defineAgent:` reason: no seat was added, no stored PAT was touched, and the key the store was written with must be restored (the env key or the data folder's key file). The reason names no path.
3. `removeCredential` keeps the lenient read, since it writes only when the id is present; its JSDoc says so.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `defineAgent` (wire) over an unreadable `credentials.enc` | the registry's fail-closed read contract | Refused before anything is written: no registry row, the store's bytes identical | — | `defineAgent` JSDoc | AC-1 |
| `credentials.enc` writers (`defineAgent`, `storeCredential`) | `FleetTenantService.readEncryptedRecord`'s contract | The write snapshot comes from the strict read | a missing store is empty, as today | `readCredentialsForMutation` JSDoc | AC-1, AC-2 |
| `resolveCredential` and Start | unchanged | An unreadable store still resolves `null` | — | — | AC-3 |

## Acceptance Criteria

- [ ] AC-1: defining a seat over a `credentials.enc` that does not decrypt refuses. The store's bytes and the registry are unchanged, and the wire answers `{status: 'rejected', reason}` naming the remedy without a path (unit).
- [ ] AC-2: a missing store still accepts the first define, and a define over a readable store keeps every other seat's PAT (unit).
- [ ] AC-3: `resolveCredential` over an unreadable store still answers `null` (unit).

## Out of Scope

- Recovering PATs already lost this way. None is known, and the store keeps no history.
- Making two processes agree on one key: deployment configuration.
- #815's replacement transaction, which will take the same strict snapshot.

## Related

#571 (the move's re-adds). #815 (found during its intake; its replacement verb needs this strict snapshot). #861 (also edits `defineAgent`; whichever lands second merges).

Sweeps: live latest-open 20 Brain issues at 2026-10-04T20:59Z, no equivalent; A2A, the last 30 in all read states, no claim on this scope; Memory Core keyed on the symptom (an undecryptable store, a wrong key or missing key file, PATs lost after adding a seat), no prior decision; Knowledge Base ticket search, no match; exact search for `credentials.enc` returns #863, #684 and #571, none the same; own assignments, none overlapping (#52 changes what `defineAgent` claims first, not its credential read). Structure map: `ai/services/fleet`.

Decision Record impact: `none`.

Origin Session ID: 6b13f348-5848-47a1-8740-c4a9d1dfaea7
Retrieval Hint: "credentials.enc unreadable defineAgent erases other seats PAT strict mutation read readEncryptedRecord"


## Timeline

- 2026-10-04T21:00:31Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-04T21:00:32Z @neo-opus-ada added the `bug` label
- 2026-10-04T21:00:32Z @neo-opus-ada added the `ai` label
- 2026-10-04T21:00:32Z @neo-opus-ada added the `agent-os` label
- 2026-10-04T21:00:38Z @neo-opus-ada added parent issue #571
- 2026-10-04T21:15:56Z @neo-opus-ada cross-referenced by PR #871

