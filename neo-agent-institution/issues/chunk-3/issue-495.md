---
id: 495
title: 'The install leg''s custody digest hashes the plane record and the fleet root, not the seat homes'
state: CLOSED
labels:
  - bug
  - ai
  - build
assignees:
  - neo-opus-vega
createdAt: '2026-10-03T09:53:02Z'
updatedAt: '2026-10-03T10:45:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/495'
author: neo-opus-vega
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
closedAt: '2026-10-03T10:45:18Z'
---
# The install leg's custody digest hashes the plane record and the fleet root, not the seat homes

## Context

The first real run of `npm run install:mac` (#473 / #474, on the #12 retry at `e1a9dbe`, 2026-10-03 ~09:45Z) replaced the canonical bundle and parked the displaced one in the rollback slot as designed, then **halted before `--open`** on the custody comparison. Emmy's falsifier (A2A `MESSAGE:d43b7af0`, `MESSAGE:5c0f35a5`) names the cause: an independent physical-tree comparison found zero changes across 53,944 regular files and 118 symlinks, and all nine protected files were byte-identical; the digest changed anyway.

## The Problem

`custodyDigest` walks `<userData>/brain/fleet` with `fs.readdirSync(dir, {recursive: true, withFileTypes: true})`. Two defects of mine, each sufficient:

1. **Scope.** `brain/fleet/agents/<seat>/` holds the seats' homes — full repository checkouts (53,945 files on the reference machine). They are the seats' own mutable state, not the custody files the README names (registry, credentials, keys, tenants, the plane record).
2. **Link traversal.** Node's recursive `readdirSync` follows directory symlinks. Each seat home links its `.neo-ai-data/{sqlite,logs,heap-observation,wake-daemon,memory-wal,fleet}` INTO the bundle's organism (`/Applications/Neo Harness.app/Contents/Resources/organism/.neo-ai-data/…`), so replacing the bundle changes what those links resolve to, and the digest hashed the new bundle's data. Controlled reproduction: a `custody/link -> external/` with a changed `external/live.txt` moves the digest (4ee11740… → fc1579d9…) while the physical tree is unchanged.

The guard therefore fires on every successful install. A guard that fires on good is not a guard.

## The Architectural Reality

- The custody set, per the harness README's store-survival section and the install leg's purpose: `<userData>/plane.json`, `plane-bearer.bin`, `seat-root.json`; `<userData>/brain/fleet/{registry.json, credentials.enc, fleet.key, signing.key, tenant-credentials.enc, tenants.json, open-work.json}`; and the **presence** of each seat under `brain/fleet/agents/` ("preserve an existing destination `agents/` directory") — never the homes' contents.
- `harness/install.mjs` — `custodyDigest(dir)`, the `custody` plan steps (`baseline` after quit, `compare` before `open`), `CUSTODY_RELATIVE_DIR`, `DEFAULT_USER_DATA_DIR`.
- Spec: `test/playwright/unit/harness/install.spec.mjs` (the two custody arms).

## The Fix

`custodyDigest(userDataDir)` hashes, in sorted order and without following any link: the three plane-record files when present (name + bytes); every depth-1 entry of `brain/fleet` — a regular file by name + bytes, a symlink by name + its target string (`readlinkSync`), the `agents` directory by name + the sorted names of its entries (`lstat`, no descent), any other directory by name only. `null` when `brain/fleet` is absent. The plan's custody steps carry the userData root; the CLI prints the digest's scope. README: the claim narrows to what is hashed. Arms: a seat-home file change and a change behind a seat-home link leave the digest unchanged; a seat added or removed, a depth-1 link's target string, a plane-record byte, a fleet-root byte each move it; the composed executor arms stay green with the new signature.

## Acceptance Criteria

- [ ] On a userData fixture with a seat home containing a directory symlink to an outside directory, changing a file behind the link and changing a file inside the home leave `custodyDigest` unchanged.
- [ ] Adding a seat directory, changing a depth-1 symlink's target string, changing one byte of `plane.json`, and changing one byte of `registry.json` each change the digest.
- [ ] The executor's two custody arms (shutdown write passes; installer-window write fails before `--open`) hold with the digest keyed on the userData root.
- [ ] README's custody sentence names the hashed set and says seat homes and link targets are not read.
- [ ] Post-merge (Residual-Owner: #12): the next `install:mac --quit` on the installed machine completes its comparison with `unchanged` and relaunches.

## Out of Scope

Which files belong to custody beyond the README's list (a plane-record ADR question if it arises); the seats' `.neo-ai-data` links into the bundle (by design — the organism is the bundle's).

## Avoided Traps

Weakening the comparison to a warning (Emmy's receipt: "I'll send the exact falsifier rather than weaken the guard") — the guard stays fail-closed, its scope is corrected. Hashing symlinks by resolved content — the link string is the custody fact; what it points at belongs to whoever owns the target.

## Related

Parent: #7. #473 / #474 (the install leg). #12 (the cut that found it; the acceptance receipt stays there). Emmy's falsifier messages above.

Live latest-open sweep: the latest 20 open Institution issues read at 2026-10-03T08:47Z and every lane-claim since (#484 #490 #491 #493 #494); no leaf on the custody digest. A2A: Emmy offered the follow-up at 09:48Z; taken by the leg's author with her agreement sought at 09:52Z. Own-assignment: #485, #491.

Origin Session ID: 075e6b2a-b93a-4972-b143-0fca9e7c06d8
Retrieval Hint: "install:mac custodyDigest symlink seat homes false positive plane record fleet root"

## Timeline

- 2026-10-03T09:53:02Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-03T09:53:04Z @neo-opus-vega added the `bug` label
- 2026-10-03T09:53:04Z @neo-opus-vega added the `ai` label
- 2026-10-03T09:53:04Z @neo-opus-vega added the `build` label
- 2026-10-03T09:53:11Z @neo-opus-vega added parent issue #7
- 2026-10-03T09:55:21Z @neo-opus-vega cross-referenced by PR #496
- 2026-10-03T09:58:20Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-03T10:10:27Z @neo-opus-vega referenced in commit `4719290` - "fix(harness): the custody digest classifies every selected path before reading it, and the plane record counts without a fleet root (#495)"
- 2026-10-03T10:45:17Z @tobiu referenced in commit `231a643` - "fix(harness): the install leg's custody digest hashes the plane record, the fleet root and seat presence (#495) (#496)

* fix(harness): the install leg's custody digest hashes the plane record, the fleet root and seat presence, never a seat home or a link's target (#495)

* fix(harness): the custody digest classifies every selected path before reading it, and the plane record counts without a fleet root (#495)"
- 2026-10-03T10:45:18Z @tobiu closed this issue

