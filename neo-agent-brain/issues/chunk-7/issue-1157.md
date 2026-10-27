---
id: 1157
title: Rewrite ADR-0019's configSnapshot prescription and widen its two guards
state: CLOSED
labels:
  - bug
  - documentation
  - ai
  - testing
  - architecture
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-18T21:59:28Z'
updatedAt: '2026-10-18T22:53:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1157'
author: neo-opus-grace
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
blocking:
  - '[x] 1151 Remove remaining configSnapshot usages, delete helpers, and gate in lint'
  - '[x] 1150 Remove configSnapshot from Memory Core and Knowledge Base services and specs'
closedAt: '2026-10-18T22:53:25Z'
---
# Rewrite ADR-0019's configSnapshot prescription and widen its two guards

## Context

Ticket 1 of the three graduated from Ideation Sandbox Discussion `neomjs/neo#19856` (Option C + Option E), beside `#1150` (Ticket 2) and `#1151` (Ticket 3). After `@tobiu`'s review of `#1115` (`#1135`) and `#1102` (`#1116`), the team traced the `configSnapshot` wave (`#1086`, `#1092`, `#1097`, `#1099`, `#1103`) to its source: ADR-0019 and the B4 guard *prescribed* an injected config snapshot. Authors followed the prescription and built a second configuration authority (`B2`/`B5`), which Tickets 2 and 3 now remove from 39 files.

This ticket corrects the prescription at its source, states the B4 policy with its one real exception, and widens the two guards where the class still escapes. Two of the six sites are mine: `#1096` (PR `#1097`) added the ceiling-breach hint naming `configSnapshot.mjs`, next to the `#508` (PR `#525`) wording.

## The Problem

1. **The guard prescribes an ADR violation.** A ceiling breach ends with *"Isolate by construction instead, through an injected config snapshot (ai/services/shared/configSnapshot.mjs)."* ADR-0019's status line says promoting the broader B4 rule waits on specs *"growing a real injection story"*. Both send authors to a mutable copy of the config.
2. **"By construction" means two things.** ADR-0019 §5.4 uses it for `UNIT_TEST_MODE` resolution. `check-aiconfig-test-mutation.mjs:103` calls `snapshotAiConfig`, a mutate-and-restore of the shared singleton, *"the by-construction answer"*.
3. **The real exception is unnamed.** `snapshotAiConfig` is the sanctioned tool when a service's own `ConfigProvider` read is the unit under test, yet §4 says NEVER without qualification. An unnamed exception reads as a loophole, and that ambiguity is how `configSnapshot` came to look sanctioned.
4. **Two guard blind spots.**
   - `DB_PATH_LEAVES` does not cover Chroma's host/port leaves, and writing those re-points a test's client at another store. Docker isolates a test's filesystem, not the endpoints it reaches: the local plane publishes Chroma on `127.0.0.1:8000` and its ingress on `127.0.0.1:3102`, and a packaged `own`-mode Brain runs on the host (§10.7).
   - `check-aiconfig-antipatterns.mjs` scans `ai/` only, so `src/` is unguarded. `src/evolution/RemDigestion.mjs` already carries `configSnapshot`, and Ticket 3's lint rule needs `src/` in scope before it can gate there.

## The Architectural Reality

Verified on `origin/dev` @ `9cd1528b`.

**Design authority:** ADR-0019 §5.4: *"Tests isolate by construction (`UNIT_TEST_MODE` → the config resolves the test DB). Never mutate the shared singleton (B4)."* §4 Rule: *"a test NEVER mutates the shared `AiConfig`; isolation is by construction."* The prescription contradicts both.

**Prior decision.** `neomjs/neo#12598` and `neomjs/neo#13520` (June 2026) closed unmerged, because a snapshot/restore helper does not isolate a test from a live DB. The ladder keeps that decision: DB-path writes stay gated outright, and rung `(i)` deletes the override. `(iv)` covers only a service's own config-varying read under test, which construction cannot vary per test, and it carries a sunset.

| # | Site | Current text |
|---|---|---|
| 1 | ADR-0019 `:7`, status line (`#508` amendment) | promotion *"waits on … growing a real injection story rather than a per-leaf escape marker"* |
| 2 | `check-aiconfig-test-mutation.mjs:786-789`, ceiling-breach stderr | *"Isolate by construction instead, through an injected config snapshot (ai/services/shared/configSnapshot.mjs)."* |
| 3 | same, `SHARED_CONFIG_MUTATION` block `:297`, `:301-302` | *"no by-construction story yet"*; *"the files whose correct fix is an injected config snapshot"* |
| 4 | same, `findSharedConfigMutations` JSDoc `:344-345` | *"goes through an injected snapshot instead; that is the sanctioned form"* |
| 5 | same, `main()` comment `:718` | *"would red the files whose correct fix is an injected …"* |
| 6 | same, `CLONE_CAPTURE` note `:103` | *"`snapshotAiConfig` is the by-construction answer"* |
| (3a) | same, header `:36-38`, the claim site 3 cites | *"that class has no by-construction story yet"* |

**Guards.**
- `DB_PATH_LEAVES` (`:40`) is `storagePaths|database|collections|logPath`, consumed by `DB_PATH_MUTATION` (`:69-78`). Chroma's endpoint leaves are `engines.chroma.hostProd/hostTest/portProd/portTest` (`ai/configBase.mjs:1189-1192`) and the derived `engines.chroma.host/port` (`:2807-2808`).
- Measured over 931 `test/**` files with the guard's own `SHARED_CONFIG_MUTATION`:
  - **0** writes reach a Chroma host/port leaf.
  - **16** reach other host leaves: 10 in specs (`openAiCompatible.host`, `ollamaConfig.host`, all local stubs) and 6 fixture lines in the guard's own spec.
  - A generic `host|port` widening would gate the 10. The Chroma-scoped one gates nothing today.
- `check-aiconfig-antipatterns.mjs` collects with `find ai …` (`:346`) and keeps only `ai/` paths (`:359`), so an explicit `src/` argument is dropped too. Its current rules report **0** hits over the 13 `src/**/*.mjs` files. As a control, a B3 line and an A5 line run through the same `findAntipatterns` both report, so widening is green today.
- `.github/workflows/config-template-ssot-lint.yml` already runs both guards on `src/**/*.mjs` changes, so no workflow edit is needed.
- `ai/scripts/lint/shared-config-write-ceilings.json` covers 37 files and 342 writes, with 0 overlap with the 21 `configSnapshot` specs; the live guard prints 342. It stays.
- `ConfigProvider.setEnvOverride` (`ai/ConfigProvider.mjs:496`) records into `#runtimeEnvOverrides`, which has no delete or clear. `snapshotAiConfig` lives at `test/playwright/unit/ai/services/memory-core/util.mjs:178`. Worker-scope env is bound in `test/playwright/configTemplateResolver.mjs` (`:133-145`).

## The Fix

1. **ADR-0019 §5.4 states the ladder.** Item 4 becomes: take the first rung that serves.
   - `(i)` An override `UNIT_TEST_MODE` already resolves: delete it.
   - `(ii)` A worker-wide value: harness env bound before the first `ConfigProvider` import, never `setEnvOverride` in a spec.
   - `(iii)` A pure operation's scalar: pass it to that operation, with no config tree (B5).
   - `(iv)` **The one named B4 exception.** When the service's own `ConfigProvider` read is the unit under test, use `snapshotAiConfig` mutate-and-restore inside one `try/finally` that overlaps no other test's reads.

   *By construction* names `(i)`–`(ii)` only.
2. **ADR-0019 §4's Rule names the exception and its boundaries.** It adds the live-resource boundary from Problem 4 (published loopback ports, packaged `own` mode) and the shared-worker / async-overlap boundary: a write persists for every later test in the worker, and un-awaited work can read it after the restore.
3. **ADR-0019 §6 gains OQ2's line:** an amendment to this ADR checks its prescription against §5 before it lands.
4. **ADR-0019 `:7` and the B4 row (`:64`).**
   - The *"real injection story"* wording becomes the §5.4 ladder.
   - The B4 row lists Chroma host/port in its gating subset.
   - The row keeps `[guarded: check-aiconfig-test-mutation]`, its PARTIAL disclosure and the `#1096` ceiling sentence.
5. **`check-aiconfig-test-mutation.mjs` sites 2–6 and 3a** point at ADR-0019 §5.4.
   - Site 6 calls `snapshotAiConfig` rung `(iv)`.
   - The stderr hint names the three non-mutating rungs first and `snapshotAiConfig` last.
   - The ceiling mechanism, its sidecar and its retirement condition (*"when the last ceiling reaches zero"*) are untouched.
6. **Chroma-scoped DB-path leaves.** `DB_PATH_MUTATION` also flags an assignment to a host/port leaf under a `chroma` node, in dotted or string-literal-bracket form. Writes to `openAiCompatible.host` / `ollama*.host` stay report-only under `SHARED_CONFIG_MUTATION`.
7. **`check-aiconfig-antipatterns.mjs` scans `ai/` + `src/`.**
   - `:346` collects both roots, and `:359` keeps both prefixes.
   - Its messages and header name both roots.
   - No new rule: `#1151` adds the `configSnapshot` / whole-root `|| aiConfig` `codeMask` rule once its 39 files are clean.
8. **Byte budget.** ADR-0019 is 53,383 bytes. Rewrites offset the additions where the text allows; the PR states the net delta.

### Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `check-aiconfig-test-mutation` ceiling-breach stderr | ADR-0019 §5.4; `neomjs/neo#19856` OQ1 | Names §5.4's rungs, with `snapshotAiConfig` last; no `configSnapshot` path | Exit code unchanged (1 on breach) | File JSDoc | `configSnapshot` absent from the guard; breach-message arm |
| `DB_PATH_MUTATION` / `findDbPathMutations` | ADR-0019 §4 B4; `#19856` OQ1 AC-3 | Also flags an assignment to a host/port leaf under a `chroma` node (dot or bracket) | Escape marker, as today | B4 row and file header | Red-first arms: Chroma host/port flagged in both forms, `openAiCompatible.host` not flagged; guard run on the PR head reports 0 DB-path violations |
| `check-aiconfig-antipatterns` scan set (default and argv) | `#19856` OQ1 AC-4 | Scans `ai/**/*.mjs` and `src/**/*.mjs`; keeps an explicit `src/` argument | Nothing in scope → exit 0, message names both roots | Lint header | Red-first arm: a `src/` file with a B3 line fails; full run green on the PR head |
| ADR-0019 §4, §5.4, §6, status line, B4 row | `#19856` OQ1 AC-1/AC-2, OQ2 | Ladder, one named B4 exception, both boundaries, the amendment check | — | The ADR itself | ADR-reading lints green; reviewer read against §3/§5 |

**Decision Record:** `Required: ADR 0019`. This ticket is that amendment. Decay-mitigation, if the ladder text does not net-reduce ADR bytes: strike the named `(iv)` exception when its last use goes, and retire the ceiling sentence when the last ceiling reaches zero.
**Decision Record impact:** `amends ADR 0019` (§4 Rule, §5.4, §6, status line, B4 row).

## Signal Ledger
- `claude`: `[GRADUATION_APPROVED]` (family-of-record via §6.4 aggregation)
  - `[GRADUATION_APPROVED by @neo-opus-grace @ 2026-10-18T20:32:39Z]` (`DC_kwDODSospM4BIk2C`)
  - `[GRADUATION_APPROVED by @neo-opus-vega @ 2026-10-18T20:32:39Z]` (`DC_kwDODSospM4BIk2r`)
- `gemini`: `[AUTHOR_SIGNAL by @neo-gemini-pro @ 2026-10-18T20:32:39Z]`
- `gpt`: divergence read completed and folded (`@neo-gpt-sophie` @ `DC_kwDODSospM4BIkQ2`); see `## Unresolved Liveness`

## Unresolved Dissent
None. Point 4 `GRADUATION_DEFERRED` from `DC_kwDODSospM4BIktj` was reconciled in full at `2026-10-18T20:32:39Z` and verified in `DC_kwDODSospM4BIk2C`.

## Unresolved Liveness
- `gpt` (`@neo-gpt-sophie`, `@neo-gpt-emmy`, `@neo-gpt`): `participationStatus: active`. `@neo-gpt-sophie` completed the GPT divergence read (`DC_kwDODSospM4BIkQ2`, folded in full). `STATUS: peer-owned liveness disposition @ DC_kwDODSospM4BIkQ2`, with floor-2 and non-author `APPROVED` satisfied by `gemini` + `claude`.
- `kimi` (`@neo-kimi-phoebe`, `@neo-kimi-iris`): `participationStatus: operator_benched`.

## Discussion Criteria Mapping
- `neomjs/neo#19856` OQ1 AC-1 (the six prescription sites) -> AC-1
- `neomjs/neo#19856` OQ1 AC-2 (B4 policy, `(i)–(iv)` ladder, live-resource and shared-worker boundaries) -> AC-2
- `neomjs/neo#19856` OQ1 AC-3 (Chroma-scoped `DB_PATH_LEAVES`) -> AC-3
- `neomjs/neo#19856` OQ1 AC-4 (`src/` scan root at `:346` and `:359`) -> AC-4
- `neomjs/neo#19856` OQ2 (ADR-0019 §6 amendment check) -> AC-2

## Acceptance Criteria

- [ ] AC-1: The six sites and site 3a no longer prescribe `configSnapshot` or an injection story; each points at ADR-0019 §5.4. `git grep -n -E "configSnapshot|injection story|injected (config )?snapshot"` over `check-aiconfig-test-mutation.mjs` and ADR-0019 returns nothing. In both files, *by construction* refers only to `UNIT_TEST_MODE` or worker-scope resolution.
- [ ] AC-2: ADR-0019 states the B4 policy completely.
  - §5.4 states the `(i)–(iv)` ladder, with `(iv)` as the one named B4 exception (`snapshotAiConfig`, one `try/finally`, non-overlapping).
  - §4's Rule names that exception and both boundaries.
  - §6 carries the amendment check.
  - The B4 row keeps `[guarded: check-aiconfig-test-mutation]`, its PARTIAL disclosure and the `#1096` ceiling sentence.
- [ ] AC-3: `findDbPathMutations` flags a test assignment to a host/port leaf under a `chroma` node in both dotted and bracketed form (red-first arms), and leaves `openAiCompatible.host` to `SHARED_CONFIG_MUTATION`. `node ai/scripts/lint/check-aiconfig-test-mutation.mjs` on the PR head reports 0 gating violations, with `shared-config-write-ceilings.json` unchanged.
- [ ] AC-4: `check-aiconfig-antipatterns.mjs` scans `ai/` and `src/` by default and keeps an explicit `src/` argument (red-first arm with a `src/` B3 line). Its full run on the PR head is green.
- [ ] AC-5: Both guards' unit specs and every lint that reads ADR-0019 (`npm run ai:lint-config-template-ssot`) pass. The PR states ADR-0019's byte delta and, if it is positive, the decay-mitigation clause above.

## Out of Scope

- Removing `configSnapshot` from services and specs, deleting the helper modules, and the `configSnapshot` / whole-root fallback `codeMask` rule with its activation (`#1150`, `#1151`).
- Retiring `shared-config-write-ceilings.json` or changing any ceiling. It gates 342 writes in 37 unrelated files until the last ceiling reaches zero.
- Chroma's `useTestDatabase` selector. A write to it can also re-point a test at a live store and there are 0 test writes today, but it sits outside the signed host/port scope, so it needs its own decision.
- The domain / execution-authority / deployment taxonomy. It lives on `#212` (`IC_kwDOUBzDFM8AAAABdNOTdQ`) and adds no ADR-0019 bytes.
- B4 scanning of `ai/**`. The test-mutation guard stays `test/**`-only, as the B4 row discloses.

## Avoided Traps

- **Generic `host|port` in `DB_PATH_LEAVES`.** It gates 10 live local-stub writes that are not store endpoints and that the ceiling ratchet already owns.
- **Calling `snapshotAiConfig` "by construction".** It writes the shared singleton and restores it. Naming it as the exception keeps the rule honest.
- **`ConfigProvider.setEnvOverride` as rung `(ii)`.** It has no delete or clear, so a per-test override outlives its test.
- **Retiring the ceiling sidecar together with the prescription.** It is a different mechanism; only the prescription was wrong (`DC_kwDODSospM4BIktj` Point 4).

## Related

- Graduated from `neomjs/neo#19856` (`DC_kwDODSospM4BIkDB`, `DC_kwDODSospM4BIkQ2`, `DC_kwDODSospM4BIktj`, `DC_kwDODSospM4BIk2C`, `DC_kwDODSospM4BIk2r`).
- Blocks #1150 and #1151.
- Corrects the prescription that `#508` (PR `#525`) and `#1096` (PR `#1097`) added.
- Coordinated with `#212`.

Live latest-open sweep: checked the latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-18T21:58:03Z` (newest `#1153`), plus an open-issue search for `ADR-0019 OR configSnapshot OR DB_PATH_LEAVES OR check-aiconfig`; no equivalent beyond the siblings `#1150`/`#1151`.
A2A claim sweep: `list_messages({status: 'all', limit: 30})` at `2026-10-18T21:58Z`; no competing claim on this scope.
MC sweep: three `query_raw_memories` calls, on the guard hint and injected snapshot, on `DB_PATH_LEAVES` with Chroma host/port and the scan root, and on the B4 exception. They found one prior decision, cited above (`neomjs/neo#12598`, `neomjs/neo#13520`), and none against this direction.
Own-assignment sweep: 3 open (`#1145`, `#684`, `#54`), none overlapping.
Agent OS structure map: `npm run ai:structure-map -- --files --loc` places both guards in `ai/scripts/lint` (23 files; `check-aiconfig-antipatterns.mjs` 173 code LOC, `check-aiconfig-test-mutation.mjs` 326); no new `.mjs` file.

Origin Session ID: 874e962d-26d5-415d-99e7-582d2b05e15c

Retrieval Hint: `query_raw_memories("D#19856 Ticket 1 ADR-0019 test-isolation ladder named B4 exception snapshotAiConfig configSnapshot prescription")`


## Timeline

- 2026-10-18T21:59:30Z @neo-opus-grace added the `bug` label
- 2026-10-18T21:59:30Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-18T21:59:30Z @neo-opus-grace added the `documentation` label
- 2026-10-18T21:59:30Z @neo-opus-grace added the `ai` label
- 2026-10-18T21:59:30Z @neo-opus-grace added the `testing` label
- 2026-10-18T21:59:30Z @neo-opus-grace added the `architecture` label
- 2026-10-18T21:59:31Z @neo-opus-grace added the `agent-os` label
- 2026-10-18T21:59:45Z @neo-opus-grace marked this issue as blocking #1150
- 2026-10-18T21:59:50Z @neo-opus-grace marked this issue as blocking #1151
- 2026-10-18T22:23:52Z @neo-opus-grace cross-referenced by PR #1160
- 2026-10-18T22:53:25Z @tobiu referenced in commit `cf8b879` - "fix(aiconfig): ADR-0019 and its guards prescribe the test-isolation ladder, not a config snapshot (#1157) (#1160)

* docs(adr): ADR-0019 names its test-isolation ladder and the one B4 exception (#1157)

§5.4 states the (i)-(iv) ladder from D#19856: delete what UNIT_TEST_MODE
resolves, bind worker-wide values in the harness env, pass a pure operation
its input, and only then snapshotAiConfig as the one named B4 exception, with
a sunset. "By construction" now names (i)-(ii) only; three generic uses of
the phrase are reworded.

§4's Rule names that exception and the two boundaries it rests on: Docker
isolates the filesystem, not the stores a test reaches, and a restore does
not stop a standing write from being read. Its lint sentence pointed at a
fail-build check of every write, which the B4 row already corrects to
PARTIAL; it now points at the row.

The status line's "real injection story" becomes the ladder, the B4 row
lists Chroma's endpoint leaves in its gating subset, and §6 asks an
amendment to check its own prescription against §5 (OQ2).

* fix(lint): the B4 guard prescribes the isolation ladder and gates Chroma's endpoint leaves (#1157)

The ceiling-breach hint sent authors to ai/services/shared/configSnapshot.mjs,
and four comments called an injected snapshot the sanctioned fix. They now
name ADR-0019 §5.4's ladder, with snapshotAiConfig last. The CLONE_CAPTURE
note stops calling snapshotAiConfig "the by-construction answer".

DB_PATH_MUTATION also matches a host/port leaf under a chroma node, dotted or
bracketed. A write there re-points the client at another store. Generic host
leaves (openAiCompatible, ollama) stay report-only: they are local stubs, and
the ceiling ratchet already owns those writes. Today the widening gates 0
writes; a generic host|port pattern would have gated 10.

The CLI arm that pinned "configSnapshot" in the hint now pins the ladder and
refuses the old remedy.

* feat(lint): the AiConfig antipattern guard scans src/ beside ai/ (#1157)

Brain domains are moving into src/, and src/evolution/RemDigestion.mjs
already carries configSnapshot, yet the guard collected `find ai` and
dropped every path outside ai/, an explicit src/ argument included.

SCAN_ROOTS names both roots, and collectDefaultFiles lists them. The
argument filter and the messages follow SCAN_ROOTS. No rule changes: its
current rules report 0 hits over src/. The workflow already runs this guard
on src/**/*.mjs changes."
- 2026-10-18T22:53:25Z @tobiu closed this issue
- 2026-10-19T13:38:58Z @neo-opus-vega cross-referenced by PR #1187
- 2026-10-19T16:10:37Z @neo-opus-ada cross-referenced by #1204
- 2026-10-19T17:21:21Z @neo-opus-vega cross-referenced by PR #1210

