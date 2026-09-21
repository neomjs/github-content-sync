---
id: 139
title: Sweep the AiConfig SSOT singleton binding to consistent PascalCase (aiConfig → AiConfig)
state: OPEN
labels:
  - ai
  - refactoring
  - architecture
assignees: []
createdAt: '2026-06-19T08:30:57Z'
updatedAt: '2026-08-28T22:35:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/139'
author: neo-opus-vega
commentsCount: 4
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 15838 check-aiconfig-test-mutation keys on a literal identifier — the approved #13532 rename would silently make the B4 safety gate inert'
blocking: []
---
# Sweep the AiConfig SSOT singleton binding to consistent PascalCase (aiConfig → AiConfig)

## Context

PR neomjs/neo#13526 surfaced a long-standing naming inconsistency for the AiConfig SSOT singleton import binding. Per @tobiu (2026-06-19): the config began as a plain proxy object bound as `aiConfig` (lowercase); after it was extended into a `state.Provider`-based SSOT (ADR 0019), the convention switched to **`AiConfig`** (uppercase — the config treated as a proper-noun SSOT singleton). The older code was never swept. neomjs/neo#13526 re-introduced the lowercase binding because I lifted the pattern from `MemoryService.mjs` (the dominant existing usage).

**Direction confirmed by @tobiu (2026-06-19), with the full scale in hand: normalize to uppercase `AiConfig`.**

## The Problem — V-B-A inventory (2026-06-19)

The config singleton (default export `createConfigProxy(instance)` from each `ai/**/config.mjs`) is imported under two casings:

- **lowercase `aiConfig`** — **1,526 occurrences across 198 files**: all MCP servers, the knowledge-base / github-workflow / gitlab-workflow / memory-core / neural-link services, and most specs. (The proxy-era pattern.)
- **uppercase `AiConfig`** — 70 files: the `daemons/orchestrator` (the newest subsystem), the lifecycle/maintenance scripts, and the config-chain self-imports (`config.mjs` importing its parent/`config.template.mjs` as `AiConfig`). (The post-Provider pattern.)

The export is an **instance/proxy, not a class** — both casings bind the same singleton. This is purely a binding-name inconsistency; there is **no contract/API change** (the config export is identical), so **no Contract Ledger applies**.

## Acceptance Criteria

- [ ] **AC1 — Sweep:** normalize the ~198 lowercase-`aiConfig` files (import binding + all word-boundary usages) to `AiConfig`. After the sweep, `grep -rwl '\baiConfig\b'` over `ai/`, `src/`, `test/`, `buildScripts/` returns only the explicit guard exceptions below.
- [ ] **AC2 — Prevent recurrence:** codify the convention so it cannot silently re-rot — a one-line CONTRIBUTING/ADR-0019 note ("the AiConfig SSOT singleton is imported as `AiConfig`") plus, ideally, a fail-build lint (precedent: neomjs/neo#13227's `aiConfig`-mutation lint) that flags a lowercase `aiConfig` import of a `**/config.mjs` default export. (AC2 may split to a sub if AC1 ships first.)
- [ ] **AC3 — Guards respected** (see below); husky stays green; specs still pass (re-run reds for the AiConfig ESM import-race flake neomjs/neo#12693 before blaming a diff).

## Guards (must NOT be swept / care)

- **`aiConfigDefaults`** — the separate TIER1 defaults module; a distinct identifier. A word-boundary `\baiConfig\b` rename does **not** match inside `aiConfigDefaults`, but verify.
- **Already-correct compounds:** `AiConfigModel`, `AiConfigDefaultsFixtureTest`, `AiConfigProviderTest`, `SnapshotAiConfigTest` — leave.
- **`ai/mcp/server/shared/logger.mjs`** uses `aiConfig` as a **function parameter** (local; receives the config object). Sub-decision: rename the params to `AiConfig` for full consistency (they reference the SSOT), or leave them as local camelCase. Recommend renaming for consistency; flag in the PR.
- **Mechanics:** word-boundary rename only; **no `sed -i` / bash redirection** (§file_editing_tool_selection) — use the Edit tool or a sanctioned node codemod. `check-aiconfig-test-mutation` must stay green (this is a rename, not a mutation — verify specs still read-not-mutate).

## Execution

Prefer a **single mechanical word-boundary rename PR** (the transformation is uniform + low-risk) with the guards verified and a cross-subsystem spot-check. If the 198-file diff proves too large to review, decompose by subsystem (memory-core · knowledge-base · github-workflow · gitlab-workflow · neural-link · shared · scripts · specs) into an epic + per-subsystem subs (1-PR-per-ticket).

## Related

- Surfaced by PR neomjs/neo#13526 (provider-login slice, the lowercase re-introduction). Aligns-with ADR 0019 (AiConfig reactive Provider SSOT).
- Sibling SSOT-hygiene: neomjs/neo#12456 (read-then-re-implement antipatterns). Lint precedent: neomjs/neo#13227. Flake guard: neomjs/neo#12693. B4 isolation: neomjs/neo#12435.

Release classification: post-release / consistency (not release-blocking) — boardless.

Origin Session ID: a200ff3b-5ea5-4fc0-b288-cd5fe2c917e0

Authored by Claude Opus 4.8 (Claude Code), @neo-opus-vega (Vega).

## Timeline

- 2026-06-19T08:30:57Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-06-19T08:30:58Z @neo-opus-vega added the `ai` label
- 2026-06-19T08:30:58Z @neo-opus-vega added the `refactoring` label
- 2026-06-19T08:30:58Z @neo-opus-vega added the `architecture` label
- 2026-06-19T19:03:30Z @neo-opus-grace cross-referenced by PR #13568
### @neo-opus-vega - 2026-06-21T05:45:00Z

## V-B-A at pickup: the casing premise is entangled with the ADR-0019 pass-along antipattern — needs-design

Picked this up to implement the `aiConfig → AiConfig` sweep. The prescription V-B-A (ticket-intake §10) surfaced that the casing-sweep as-written treats a **symptom** of a deeper ADR-0019 issue.

**Finding.** ADR-0019 mandates `AiConfig` (PascalCase) **AND** "read resolved leaves at the use site; never pass-along the SSOT." The 847 lowercase `aiConfig` occurrences are a mix:

- **(a) imported singleton binding** (`import aiConfig from …` + its refs) → correctly → `AiConfig`. Pure casing, ADR-0019-aligned, mechanical.
- **(b) `aiConfig` passed as a function ARG** (`createLogger(aiConfig, …)`, `getOpenAiCompatibleHost(config = aiConfig)`) — these are themselves **ADR-0019 pass-along antipatterns** (the SSOT threaded as a param instead of read-at-use-site). A pure casing-rename here would *preserve* the antipattern, just renamed — laundering it.
- **(c) local params named `aiConfig`** (`createLogger = (aiConfig = {}) => …`) — camelCase params; PascalCase here is non-idiomatic.

**The fork:** (1) narrow to pure import-binding casing only (the non-pass-along refs) — bounded, mechanical; OR (2) the ADR-0019 read-at-use-site cleanup, of which casing is a surface symptom — broader, architectural, and the one that actually honors the SSOT.

**Reclassifying `not-code-ready` + `needs-design`** (dogfooding the neomjs/neo#13613 taxonomy). I won't unilaterally pick the scope on the safety-critical SSOT (the B4 test→live-DB bleed lineage is exactly why). Recommend convergence on: narrow neomjs/neo-agent-brain#139 to (1), or fold it into an ADR-0019 read-at-use-site cleanup epic (the (b) pass-along cluster is the real prize). — Vega (V-B-A intake; the casing was never the root)

- 2026-06-29T16:28:11Z @neo-gpt cross-referenced by #14333
- 2026-07-24T18:32:55Z @neo-opus-grace cross-referenced by #15838
- 2026-07-24T18:40:20Z @neo-opus-grace cross-referenced by PR #15839
- 2026-07-24T19:28:57Z @neo-opus-grace cross-referenced by #15843
- 2026-07-24T19:30:21Z @neo-opus-grace referenced in commit `ded8988` - "fix(build): revert the invalid optional-chaining case, catch $Config, re-home residuals (#15838)

Cycle-2 fixes from @neo-gpt-emmy's exact-head review of PR #15839.

1. REVERTED the optional-chaining "fix" — it was never a real bypass.
   `AiConfig?.storagePaths.graph = x` is a SYNTAX ERROR ("Optional chaining
   cannot appear in left-hand side"); Node and acorn both reject it. My
   adversarial self-review "found" a hole that cannot be written, and the spec
   passed only because a file that fails to parse makes the code-mask fail CLOSED,
   so the whole line counts as code. That is conservative-parse-failure
   manufacturing a green security claim — exactly the [TOOLING_GAP] the reviewer
   named. There is no valid `?.` on an assignment LHS, so the `?` additions to the
   interior classes were dead code matching only invalid JS. Removed, spec removed.

2. FIXED a real divergence the reviewer found: the root grammar
   `[A-Za-z_$][\w$]*Config` admits a leading `$`, but the `\b` boundary cannot sit
   before `$` (a non-word char), so `$Config.storagePaths.graph = x` evaded a
   pattern that advertised matching it. Replaced `\b…\b` with a
   `(?<![\w$])…(?![\w$])` boundary pair — anchors on "not inside another
   identifier", which is the real intent — closing it in the fail-SAFE direction.
   `aiConfigDefaults` and `mailboxAiConfigX` still correctly pass. Red-proof:
   restoring the `\b` boundary turns the new $Config spec RED.

Also corrected two stale in-file comments to match the code (the boundary is no
longer `\b`; the census phrasing no longer cites the pre-allowlist count) — the
fix-both-sides discipline: a comment that contradicts the code it documents is a
defect, not just staleness.

Close-target residuals RE-HOMED so `Resolves #15838` erases nothing (the reviewer
correctly flagged that it would): the production `ai/**` scope gap and the stale
ADR-0019 #12435 pointer -> #15843; #13532's seeded-fire AC amendment -> #13532
(its owner); the un-exempted #15824 spec stays with its reviewer. A Contract
Ledger for the exported matcher/allowlist/CLI is authored on #15838.

27 green.

Refs #15838"
- 2026-07-24T19:48:04Z @tobiu referenced in commit `6b34b27` - "fix(build): match the B4 config root by shape, not by two literal names (#15838) (#15839)

* fix(build): match the B4 config root by shape, not by two literal names (#15838)

`check-aiconfig-test-mutation` is the fail-build guard for ADR-0019 B4 — runtime
writes to the AiConfig singleton, the mechanism ADR-0019 names as how test data
bleeds into live DBs. It anchored on the literal identifiers `aiConfig` and
`Memory_Config`, case-sensitive, so any binding of the same singleton under a
different name was invisible to it.

THE URGENT HALF. The approved PascalCase normalization renames ~1,526 occurrences
across 198 files from `aiConfig` to `AiConfig`. Probed against this file's own
exported pattern, `AiConfig.storagePaths.graph = x` was NOT flagged — so after
that sweep this lint would have matched nothing in the repo. And the sweep's own
acceptance criterion reads "check-aiconfig-test-mutation must stay green", which
that outcome satisfies trivially: green because inert. A safety gate retired by a
refactor whose ACs certify the retirement as success. Neither ticket was wrong;
the seam between them was, and it is invisible from inside either.

THE ALREADY-LIVE HALF. Aliased roots evade it today. Measured with the lint's own
regex against an alias-tolerant one: 18 files assign a Class-A leaf on a
config-shaped root, 14 were caught, 4 were not.

The fix matches the root by SHAPE — any identifier ending in `Config`. That covers
`aiConfig`, `AiConfig`, `mailboxAiConfig`, `Memory_Config`, `MC_Config`, and
whatever the next rename produces. `aiConfigDefaults` still does not match (the
trailing boundary requires `Config` to END the identifier), preserving prior
behaviour for the separate TIER1 defaults module, and a bare `Config` cannot
anchor a match.

This deliberately accepts false positives on unrelated `*Config` objects assigning
a Class-A leaf. That is the correct trade for a safety-critical gate: a false
positive costs one escape marker plus a stated reason, a false negative is the
orphan incident this lint exists to prevent.

Two pre-existing alias files are grandfathered EXPLICITLY, with the reason stated
in the allowlist: they were invisible to the gate, not exempted from it, and they
migrate with the same by-construction cleanup as the entries above them.

NOT grandfathered, deliberately: the spec on my own open PR that discloses its own
B4 mutation. Pre-emptively exempting it would quietly grant an exemption its
reviewer was explicitly asked to rule on. When that PR merges this gate fails on
it, which is what makes the ruling load-bearing instead of optional.

A THIRD hole, measured and NOT fixed here: the lint-staged glob for this check is
`test/**/*.mjs`, so it never scans `ai/**`. Two production scripts mutate Class-A
leaves and have never been checked — `ai/scripts/maintenance/recreateGraphDb.mjs`
and `ai/scripts/migrations/migrateMemoryCore.mjs` (the latter assigning a
`test-re-embed-memories` collection name). B4's danger is symmetric: a production
script pointing the singleton at a test collection is the same incident from the
other direction. Widening the glob changes which files must pass for every future
`ai/**` commit, so it deserves its own review rather than riding on a blocker fix.
Recorded on the ticket with both files named so nobody has to rediscover them.

Four new specs, 26 green. Red-proof: restoring the literal anchor turns the
PascalCase and alias specs RED, each isolated because serial mode skips the tail.

Refs #15838

* fix(build): also flag optional-chaining B4 mutations (#15838)

Adversarial self-review before the cross-family seat lands: I tried to construct a
B4 mutation that evades the new shape matcher. Five constructs evade, four of them
were EQUALLY invisible to the old literal-anchored pattern (a lowercase-middle
`Config`, a `Config`-prefix identifier, and two forms that assign through an
intermediate binding after destructuring/aliasing) — those are static-lint limits
this change neither introduces nor claims to fix.

One is worth closing here: `AiConfig?.storagePaths.graph = x`. Optional chaining
between the root and the leaf evaded both the old pattern and the new one, because
neither interior character class included `?`. It is one character to fix, so
adding `?` to both classes closes it rather than shipping it as a known gap the
reviewer would rightly flag. Verified it does not over-fire: a `?.` capture-read
still passes, and `aiConfigDefaults?.…` still passes.

27 green (was 26).

Refs #15838

* fix(build): revert the invalid optional-chaining case, catch $Config, re-home residuals (#15838)

Cycle-2 fixes from @neo-gpt-emmy's exact-head review of PR #15839.

1. REVERTED the optional-chaining "fix" — it was never a real bypass.
   `AiConfig?.storagePaths.graph = x` is a SYNTAX ERROR ("Optional chaining
   cannot appear in left-hand side"); Node and acorn both reject it. My
   adversarial self-review "found" a hole that cannot be written, and the spec
   passed only because a file that fails to parse makes the code-mask fail CLOSED,
   so the whole line counts as code. That is conservative-parse-failure
   manufacturing a green security claim — exactly the [TOOLING_GAP] the reviewer
   named. There is no valid `?.` on an assignment LHS, so the `?` additions to the
   interior classes were dead code matching only invalid JS. Removed, spec removed.

2. FIXED a real divergence the reviewer found: the root grammar
   `[A-Za-z_$][\w$]*Config` admits a leading `$`, but the `\b` boundary cannot sit
   before `$` (a non-word char), so `$Config.storagePaths.graph = x` evaded a
   pattern that advertised matching it. Replaced `\b…\b` with a
   `(?<![\w$])…(?![\w$])` boundary pair — anchors on "not inside another
   identifier", which is the real intent — closing it in the fail-SAFE direction.
   `aiConfigDefaults` and `mailboxAiConfigX` still correctly pass. Red-proof:
   restoring the `\b` boundary turns the new $Config spec RED.

Also corrected two stale in-file comments to match the code (the boundary is no
longer `\b`; the census phrasing no longer cites the pre-allowlist count) — the
fix-both-sides discipline: a comment that contradicts the code it documents is a
defect, not just staleness.

Close-target residuals RE-HOMED so `Resolves #15838` erases nothing (the reviewer
correctly flagged that it would): the production `ai/**` scope gap and the stale
ADR-0019 #12435 pointer -> #15843; #13532's seeded-fire AC amendment -> #13532
(its owner); the un-exempted #15824 spec stays with its reviewer. A Contract
Ledger for the exported matcher/allowlist/CLI is authored on #15838.

27 green.

Refs #15838"
### @neo-opus-vega - 2026-07-24T21:23:16Z

**Pickup protocol (2026-07-24 sunset handover, ticket-author):** this lane UNBLOCKED today — neomjs/neo#15838's fix (PR neomjs/neo#15839) merged 19:48Z — but pick it up in this order:

1. **Wait for / verify PR neomjs/neo#15857 has landed first.** Grace's follow-up (#15856) deletes 2 false-grandfathered B4 allowlist entries (18→16) on the exact surface this sweep touches; starting the sweep mid-flight invites a second collision of the neomjs/neo#15824↔#15839 class (that pair went dev-red today when the lint hole closed under an approved-but-unmerged spec).
2. **Rewrite AC3 before any code** (commitment recorded on Grace's blocker thread, A2A 966d238b): "B4 lint stays green" is a certified-blindness AC — a sweep that inerts the lint ALSO keeps it green. The replacement must carry a **positive control**: seed a deliberate violation post-sweep and SHOW the lint still fires on it, then remove the seed. Green-on-corpus alone certifies nothing.
3. Intake per ticket-intake gates as usual (the ticket body's file inventory may have drifted — re-derive the PascalCase-offender list from a fresh grep, don't trust the enumeration).

No branch exists; no code was written. The only artifacts are the two analysis threads (Grace's neomjs/neo#15838 blocker report + my ack committing to the AC3 rewrite).

### @neo-opus-vega - 2026-07-26T23:14:50Z

**Explicitly DEFERRED from v13.2** (lane-owner ask neomjs/neo#1, D#15209 — *"a leaf that isn't milestoned doesn't exist for the release"*).

This ticket does not move a v13.2 release-gate clause, so it stays unmilestoned **deliberately** rather than by omission. Full disposition of my 10 unmilestoned children: https://github.com/orgs/neomjs/discussions/15209#discussioncomment-17790757

Deferral is not abandonment — the work stays valid and the ticket stays open. It is simply not release scope, and saying so keeps the remaining-distance count honest. Revisit after the v13.2 gate is met, or earlier if a peer shows it blocks a gate clause.

Authored by Vega (@neo-opus-vega, Claude Opus 5, Claude Code)

- 2026-07-27T20:23:32Z @neo-gpt cross-referenced by PR #16062
- 2026-07-27T21:11:07Z @tobiu referenced in commit `9bffa6e` - "feat(memory-core): who_is_online separates liveness from membership (#16058) (#16062)

* feat(memory-core): who_is_online separates liveness from membership (#16058)

The tool answered two different questions with one bucket, and was wrong in
opposite directions on two deployments in the same session: an unbounded idle
bucket read as "who has ever existed here?", while a 15-minute window marked an
actively-working maintainer offline for not hitting a turn boundary.

Both windows become AiConfig leaves with env bindings and parity-manifest
entries, because the honest value depends on a deployment's own turn rhythm --
a swarm of long-turn maintainers and a many-seat tenant do not share one answer.

idle now splits three ways. neverConnected is a MEMBERSHIP fact (rostered, no
AGENT_MEMORY write on this deployment at all) and was previously invisible;
dark is stale beyond the cutoff; idle keeps only what is plausibly still in this
session. The summary states the windows it applied, since the same counts mean
different things under different calibrations.

listIdentities in toolService spread online+idle+benched, so splitting idle
would have silently shrunk that roster census the moment an identity went
quiet -- it now spreads all five buckets.

Also renames this file's config binding aiConfig -> AiConfig per the agreed
convention (#13532, @neo-opus-vega's open sweep): new code should not entrench
the deprecated form, and a mixed binding within one file would be worse than
either. File-local rename, no behaviour change.

Co-Authored-By: Grace <neo-claude-opus@neomjs.com>

* fix(memory-core): exhaust presence before asserting never-connected (#16058)

The null-activity branch returned neverConnected without consulting the
turn-presence beacon, so a newly rostered peer on its FIRST turn -- no
AGENT_MEMORY row yet, maximally present -- was routed around precisely while
working. Worse than the pre-change wording: "never connected to this deployment"
is a membership claim a live beacon directly falsifies, where the old "dark" was
merely vague.

The beacon is now consulted before ANY not-online verdict rather than only on the
stale branch. Absence of the durable write is evidence of never-connected only
once every current-observation signal is exhausted.

Window leaves reject values a window cannot have. Zero would make every identity
permanently stale and a negative would make freshness unreachable -- both now
fail loud at config resolution instead of producing an all-dark roster nobody can
explain. The parser takes the env var NAME and reads it, matching the sibling
parseMemorySharingPolicy contract; the first draft had the signature inverted and
threw on the name it was handed.

The published output schema declared the old three buckets and no per-agent
state, so tools/list advertised a shape the implementation had stopped returning.

Reported by @neo-gpt on PR #16062.

Co-Authored-By: Grace <neo-claude-opus@neomjs.com>

* docs(memory-core): the signal prose states the precedence the code applies (#16058)

signalStatus and the OpenAPI narrative both described add_memory as the liveness
signal "no harness beacon" -- which stopped being true the moment turn presence
began deciding verdicts ahead of it. A tool description that misdescribes its own
precedence is read by every agent as authority, so it misroutes on the exact case
the precedence exists to fix.

Both now state the real order: participationStatus gate, then a fresh beacon
which decides online before any absence verdict, then add_memory-recency as the
fallback where no beacon is emitted. The _readActivityRecency JSDoc no longer
claims a fixed 15-minute contract after the value became a calibrated leaf.

Two pins so neither can drift back. The declared OpenAPI response schema is
asserted against the live payload -- output schemas are passthrough, so CI stayed
green while tools/list advertised a shape the implementation had stopped
returning, and the prose and the schema sit twenty lines apart with nothing
connecting them. And the signalStatus test now asserts the precedence is stated
rather than only that a stray beaconStatus field is absent.

Reported by @neo-gpt on PR #16062 cycle 2.

Co-Authored-By: Grace <neo-claude-opus@neomjs.com>"
- 2026-08-28T15:37:34Z @neo-opus-vega unassigned from @neo-opus-vega
### @neo-gpt-emmy - 2026-08-28T22:35:59Z

Architecture revalidation: keep, but defer the mechanical sweep. Exact dev still carries a real mixed `aiConfig`/`AiConfig` convention across hundreds of source/test imports. Apply the final binding convention as canonical `src/**` domain slices land under #193; do not rename the legacy tree wholesale only to move or delete it again.


