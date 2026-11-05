---
id: 1480
title: 'Seat Start reuses checkouts as they are: no fast-forward, no npm install'
state: OPEN
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-11-05T12:20:54Z'
updatedAt: '2026-11-05T12:22:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1480'
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
blocking: []
---
# Seat Start reuses checkouts as they are: no fast-forward, no npm install

## Context

The operator, 2026-11-05: *"automation beats discipline. e.g. fleet manager could call npm i for repos that have a package.json on seat start, stop or both. rationale: peers would not even need to think about it then."* And later: *"i do agree that on seat start is probably smarter than stop. for the neomjs/neo repo (if used => other operators or teams might not), we could also use build-all or the less expensive build-themes."*

The evidence is neomjs/neo-agent-skills#276's census (12:08Z): the installed `neo-agent-skills` in each seat's neo clone. Sophie 0.1.29 · Clio 0.1.39 · Julia/Maryam 0.1.41 · Ada 0.1.73 · Mnemosyne/Vega/Emmy 0.1.78 · Euclid 0.1.82 · Grace 0.1.84. npm is at 0.1.87 and neo pins 0.1.86. The refresh exists only as discipline: `session-sunset` Step 1 pulls and never installs. Nine of ten seats drifted.

## The Problem

`ensureAgentRepo` returns `action: 'reused'` for an existing checkout and does nothing else. Start then launches the harness on whatever that checkout holds. The harness reads its skills (`.claude/skills/*` → `node_modules/neo-agent-skills`) and starts its MCP servers at boot. So the session runs its whole life on a stale checkout, a stale `node_modules`, and stale built assets. Every skills change, engine pin bump and theme change reaches a seat only if the seat remembers to pull, reinstall and rebuild.

## The Architectural Reality

- `ai/services/fleet/startAgentProvisioned.mjs:237`: `startAgentProvisioned` → `provisionAgent({… ensureRepo = ensureAgentRepo …})` ensures the working checkout before launch.
- `ai/services/fleet/ensureAgentRepo.mjs:53`: `ensureAgentRepo` → `{action: 'cloned' | 'reused'}`.
- `ai/services/fleet/ensureAgentRepo.mjs:90`: `ensureAgentRepos({repos, …, via: 'start' | 'prepare', onEnsured})` ensures the declared repos. It already exposes `onEnsured`, "post-clone work before a row is prepared".
- In neo, `npm install` runs `prepare` (`buildScripts/util/prepare.mjs`), which re-materializes the harness skill links. `build-themes` is the cheap rebuild of the engine's themes; `build-all` is the full one.
- The Brain is a multi-tenant product, so Fleet must not name any one tenant's repositories or build steps.
- The discipline twin is `neo-agent-skills` `session-sunset` Step 1: a pull and `--migrate-config`, with no install.
- Design authority: neomjs/neo-agent-skills#140, *"Validation must run against the text a session actually loads, from the recipient's own session"*.
- Decision Record impact: none. Structure map: N/A. This is a step inside the existing fleet Start path, and no new module is required.

## The Fix

On Start (`via: 'start'`), before the harness launches, every ensured checkout whose action is `'reused'` gets:

1. **A fast-forward**, when the tree is clean and on its default branch: `git pull --ff-only`. Otherwise the pull is skipped, so a seat's branch work never moves.
2. **An install**, when the checkout has a `package.json`: `npm install --no-audit --no-fund`.
3. **The repository's own start step:** `npm run --if-present seat:start`. The repository declares what a fresh start needs, and Fleet stays tenant-neutral. Another operator's or team's repository without that script is untouched.
4. **A config reconcile** in the Brain checkout: `ai/scripts/setup/initServerConfigs.mjs --migrate-config`.

All four are best-effort and receipted. Each step's outcome lands on the start row, and a failure never blocks the launch.

**neomjs/neo opts in** with `seat:start` = `build-themes`, run only when the theme sources are newer than the built output. That makes it a no-op on most starts and much cheaper than `build-all`. It ships as its own engine leaf, filed with this PR.

**Not on Stop.** A stop races the closing session's last writes, and Start already guarantees what the next session loads (the operator agreed, 2026-11-05).

**The deletion.** Once this ships, `session-sunset` Step 1's pull and `--migrate-config` retire to one line pointing here, in a skills PR.

## Acceptance Criteria

- [ ] Unit: a reused, clean checkout on its default branch gets the fast-forward, the install and the repository's `seat:start`, in that order, before launch. A dirty or off-branch checkout gets the install and `seat:start` only.
- [ ] Unit: a repository without a `seat:start` script is untouched by step 3.
- [ ] Unit: a failure in any step is recorded on the start row, and the launch proceeds.
- [ ] neomjs/neo declares `seat:start` (`build-themes` when its sources are newer than the build output) in its own engine PR.
- [ ] Installed receipt: after one Fleet restart of a stale seat, its neo clone's `node_modules/neo-agent-skills` matches the clone's pin, and its themes are current.
- [ ] The skills follow-up retires `session-sunset` Step 1's pull (the named deletion).

## Out of Scope

- A refresh on Stop (see above).
- A one-time catch-up broadcast. Once this ships, each seat's next Start is its catch-up.
- `build-all` on Start. It is too expensive for every launch; a repository that needs it can declare it in its own `seat:start`.

## Avoided Traps

- ⛔ **Installing without the fast-forward.** npm would install the clone's old pin.
- ⛔ **Hardcoding neo's build in Fleet.** The Brain is multi-tenant, and the repository declares its own step.
- ⛔ **Shipping before neo pins `neo-agent-skills` ≥ 0.1.88.** Today's pin (0.1.86) carries #251's 250k sunset gate, which neomjs/neo-agent-skills#275 removes. Merge after Dependabot's bump.
- ⛔ **More skill text** (neomjs/neo-agent-skills#276's first prescription). Discipline already failed nine of ten seats.

## Related

#571 (one seat folder layout, which Fleet provisions) · #950 (repositories cloned on the fly) · neomjs/neo-agent-skills#276 (closed into this) · neomjs/neo-agent-skills#140 · neomjs/neo-agent-skills#275

Live latest-open + closed sweep: I checked the latest 20 open issues, plus a search for `npm install`, `refresh checkout`, `ensureAgentRepo`, `stale node_modules` and `seat start pull`, at 2026-11-05T12:20Z. I read #571's scope (layout and provisioning, no refresh). No equivalent ticket exists.

Origin Session ID: 0ec8e189-2090-4833-9e9c-4376ea22f341
Retrieval Hint: "seat start refresh checkout fast-forward npm install seat:start build-themes ensureAgentRepo reused stale node_modules skills drift"


## Timeline

- 2026-11-05T12:20:55Z @neo-opus-grace added the `enhancement` label
- 2026-11-05T12:20:56Z @neo-opus-grace added the `ai` label
- 2026-11-05T12:20:58Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-05T12:21:02Z @neo-opus-grace cross-referenced by #276
- 2026-11-05T13:51:32Z @neo-opus-vega cross-referenced by #277

