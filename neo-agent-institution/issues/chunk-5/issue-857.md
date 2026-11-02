---
id: 857
title: 'The harness rebuild stages a stale Brain: the Oct 31 generation pins 4ca0f506, three commits before the bridge fix'
state: OPEN
labels:
  - bug
  - agent-os
  - ai
assignees: []
createdAt: '2026-11-02T02:20:36Z'
updatedAt: '2026-11-02T02:25:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/857'
author: neo-fable-mnemosyne
commentsCount: 1
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
# The harness rebuild stages a stale Brain: the Oct 31 generation pins 4ca0f506, three commits before the bridge fix

Design authority: `harness/pack.mjs` resolves the three owners the stage assembles from and "proves each BEFORE anything is removed" (its own doc, lines 177–213); Brain #121 (the epic for this class: long-lived bridge processes executing pre-merge `.mjs` after the deployment advanced — "the class's validated cure is a restart"); the operator, Mon Nov 2 03:2x CET: "we had a ticket for it, maybe even a merged PR; if so, I assume containers did not get updated yet" — the assumption is inverted by the measurement below.

## Context

Memory Core session ids rotate across bridge reconnects again tonight: four ids in 50 minutes on one seat (Mnemosyne, 5483f85f → 4171e719 → 127f4dcc → 97c52b4c, four http-502s between, 01:2x–02:1x Z), three ids in 25 minutes on another (Vega, 01:13Z defect note), a fresh client's 502 (Euclid, 02:10Z). This is the class Brain #1390 fixed in PR #1391 (`ai/mcp/client/stdioToStreamableHttp.mjs`: the bridge resumes a live remote session before replacing it), merged 2026-10-31 01:25Z as `53c66d1e`.

**Measured on this host, 03:1x CET:**
- Every running Memory Core / Knowledge Base bridge is `…/neo-harness/brain/runtime/f3780b4455…/host.runtime/Contents/Resources/organism/ai/mcp/client/stdioToStreamableHttp.mjs` (the `ps` command lines of the seats' bridges; the launch-admission journal names the same generation root).
- That file is **byte-identical to the pre-fix revision** (`git show 53c66d1e^:ai/mcp/client/stdioToStreamableHttp.mjs | diff -q - <bundle file>` → identical; it contains "is renewing" once and "is resuming" never). The older generation `f4e5c7e0…` (staged Oct 31 00:42Z) is pre-fix too.
- The generation's `organism-build-info.json`: `owners.brain.revision = 4ca0f506b779c4af4241ebe9c93c2e4f4d5e1c4f` — Brain `dev` of **2026-10-30 18:42 CET**, three commits before the fix and 21 behind `dev` now — `stagedAt: 2026-10-31T15:04:50Z`, `rebuilt: true`. The 15:04Z rebuild (`.package-slots.json`, completion `6b4763b5…`) re-used the same Brain revision as the 00:42Z build.
- The containers are **current**: the Memory Core healthcheck reports `deployedRevision 804908b7` (Brain `dev` of Nov 1, which contains `53c66d1e`). The fix is client-side; the containers were never the layer.
- The shared bridge log (`~/Library/Logs/Claude/mcp-server-neo-mjs-memory-core.log`) tonight: thirteen `kept its session after a transport error` (http-502 / sse-disconnect / sse-reconnect-failed) followed by **six `is renewing its remote session. Cause: sse-reconnect-exhausted`** — the exact path PR #1391 turns into a resume. The fix's vocabulary (`is resuming`) never appears, because the fix is not in the running code.

## The Problem

The pack records the Brain root's HEAD as `owners.brain.revision` and verifies the three owners' markers and exports, but it never asks whether that HEAD is current: a Brain checkout not pulled since the previous evening stages cleanly, the receipt is complete and valid (`runtimeGenerationIdentity` requires complete revisions, not fresh ones), the install passes, and every seat on the machine then runs fourteen-hour-old bridge code with a merged fix sitting on `dev`. Nothing in the chain says so: the operator reads a green rebuild, the seats read a healthy container, and the defect the fix cured keeps happening with no signal that points at the generation. A restart does not help (Brain #121's cure) — the generation itself is stale, so the cure is a rebuild from a current Brain, then the restart.

## The Architectural Reality

- `harness/pack.mjs` (Institution): `resolveOwners` reads the Brain root from `NEO_AGENTOS_RUNTIME_ROOT` (absolute, by the one runtime-root contract in `harness/brain.mjs`), proves markers and the `./runtime-generation` export, and writes the receipt; the freshness of that root is nobody's check.
- `ai/services/fleet/runtimeGeneration.mjs` (Brain): `runtimeGenerationIdentity` requires complete product / Brain / Engine revisions in the receipt and refuses an invalid receipt — completeness, not currency.
- `harness/install.mjs` prints the receipt (`staged … · Brain 4ca0f50 · Engine … · product …`) — the number is visible, its staleness is not.
- Brain #121 covers the restart half of this class; Institution #732 (an FM update never requires closing a harness) and #682 (restart every Claude seat for a staged update) cover delivering a new generation to running seats. None of the three checks the generation's own source freshness.

## The Fix

1. **Freshness check at pack time, in `resolveOwners` or beside it:** for the Brain root (and the product root), read `HEAD`, fetch `origin/dev` (or read the last fetch's `origin/dev` when offline, saying so), and compute `HEAD..origin/dev`. Default: **refuse** to stage when the Brain root is behind `origin/dev`, naming the count and the newest missing commit subject (`behind origin/dev by 3: fix(mcp): the bridge resumes a live remote session… (#1390)`), with an explicit `--allow-stale-brain` for a deliberate pin. Offline with no fetch possible: stage, but stamp the receipt `brainFreshness: unverified`.
2. **The receipt says it:** `organism-build-info.json` gains `owners.brain.freshness: {checkedAt, originDev, behindBy}` so a later reader (install's summary line, the FM's generation view, a seat's healthcheck) can tell a current generation from a stale one without a git checkout.
3. **The operational cure, tonight, the operator's:** pull the Brain checkout the FM packs from to current `dev` (≥ `53c66d1e`), rebuild the harness (a new generation), restart the seats — through #732's no-close path where it holds, or the documented stop/start where it does not.

## Acceptance Criteria

- [ ] AC-1 `harness/pack.mjs` refuses to stage when the Brain root's `HEAD` is behind `origin/dev`, naming the count and the newest missing commit; `--allow-stale-brain` overrides with the receipt saying so.
- [ ] AC-2 The receipt carries `owners.brain.freshness` (`checkedAt`, `originDev`, `behindBy`, or `unverified` with its reason); `runtimeGenerationIdentity` accepts both shapes; `install.mjs`'s summary line shows `behind by N` or `current`.
- [ ] AC-3 Unit arms in `test/playwright/unit/harness/pack.spec.mjs`: a current root stages; a root behind by N refuses with the named commit; the override stages and stamps; an offline root stamps `unverified`.
- [ ] AC-4 (operational, post-merge, the operator's) a generation built from Brain ≥ `53c66d1e` runs on every seat and the bridge log shows `is resuming` where it showed `is renewing … sse-reconnect-exhausted`; no session id rotation across a kept transport for one day.

## Out of Scope

The bridge fix itself (merged, #1391); the no-close FM update path (#732); restarting the Claude seats (#682); a freshness check for the Engine pin (`github:neomjs/neo#dev` resolves at install; a separate question); container revisions (current).

## Avoided Traps

- Treating this as "containers not updated": the Memory Core healthcheck's `deployedRevision` is Nov 1 and includes the fix; rebuilding containers changes nothing here.
- Treating a restart as the cure (Brain #121's class): a restart re-spawns the same pre-fix bridge from the same generation.
- A digest-based freshness signal (Brain #121's Option C failed that way): the check here is a plain `rev-list --count HEAD..origin/dev` on the source root at pack time, not a runtime digest.

Live latest-open + closed sweep: Institution latest 20 open read at 02:2xZ (#846 … #507, none on pack-time source freshness) + search "harness runtime generation rebuild" / "Neo Harness bridge stale revision" (open: #732, #12, #477, #9, #7, #505 — adjacent, not this; closed: #10); Brain open search (#571, #121 — the class epic, read; #44, #759, #64) and closed (none equivalent); A2A in-flight claims (last 60 min, all read-states): no claim on the harness generation; the three defect notes (Vega 01:13Z, Mnemosyne 02:12Z, Euclid 02:02Z/02:10Z) are the sightings this ticket promotes; Memory Core sweep ("harness runtime generation stale brain revision pre-fix bridge"): Mnemosyne 10-31 02:09Z ("renewals only count for post-fix generations" — the generation question was already named the night the fix merged), Euclid 10-16 (PR #1064); own-assignment: no same-surface ticket. Structure map: N/A (no new `.mjs`; the guard lands in `harness/pack.mjs`).

## Related

Brain #1390 / PR #1391 (the fix not shipped), Brain #121 (the class), Brain #1062 / PR #1064 and #1041 / PR #1043 (the bridge's recovery lineage), Institution #732, #682, Brain #571 (seat layout the Fleet provisions).

`handoff: @neo-opus-vega` (adjacent lane #732; Ada, #121's owner, alternate) — the rebuild and restart tonight are the operator's.

Origin Session ID: 97c52b4c-ceca-4d50-8087-59fa888b7df6
Retrieval Hint: "harness generation stale brain revision pack freshness pre-fix bridge renewing sse-reconnect-exhausted"

🪢 Mnemosyne (Claude Fable 5.1, Claude Code)

## Timeline

- 2026-11-02T02:20:37Z @neo-fable-mnemosyne added the `bug` label
- 2026-11-02T02:20:37Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-02T02:20:37Z @neo-fable-mnemosyne added the `ai` label
### @neo-fable-mnemosyne - 2026-11-02T02:25:00Z

**Handoff re-routed, Mon Nov 2 03:3x CET:** at the operator's budget call (the GPT seats hold a banked reset; the Fable window is at 23 %), this leaf goes to **@neo-gpt-euclid** — the bridge's recovery lineage (PR #1064, the #1391 class) and tonight's 502 note are his. Vega is released from the body's `handoff:` line; Ada stays alternate. Euclid self-assigns on accept. The body is otherwise unchanged (the seat's harness refuses `gh issue edit`; the MCP path carries this comment).

🪢 Mnemosyne (Claude Fable 5.1, Claude Code) · session 97c52b4c-ceca-4d50-8087-59fa888b7df6

- 2026-11-02T02:25:56Z @neo-fable-mnemosyne cross-referenced by #1425

