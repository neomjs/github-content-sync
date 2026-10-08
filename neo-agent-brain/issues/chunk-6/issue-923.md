---
id: 923
title: A Claude Desktop seat boots at its declared reasoning effort
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-07T23:36:55Z'
updatedAt: '2026-10-07T23:38:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/923'
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
blocking:
  - '[ ] 600 Agent Detail offers a Claude Desktop seat its effort, not its model'
---
# A Claude Desktop seat boots at its declared reasoning effort

Follow-up of #862 and #867, both closed, both scoped to exclude Claude Desktop. The operator renewed the requirement on 2026-10-07: each Claude Desktop start drops Opus to medium and Fable to high, and Neo's seats must boot at max without a manual adjustment.

## Context

#862 delivered "a seat starts on its declared model and effort" for Claude Code (CLI flags) and Codex (config). It left Claude Desktop out because Desktop has no supported per-session flag. Emmy revalidated that exclusion on 2026-10-07 ([6047195392](https://github.com/neomjs/neo-agent-brain/issues/862#issuecomment-6047195392)):

- The documented persistent route to max is `CLAUDE_CODE_EFFORT_LEVEL=max`. The saved `effortLevel` and `modelSettings` reject max.
- Desktop `2.26454.2` allowlists that variable and keeps a value already present in its process env; it copies from the login shell only when the key is absent. The composed session env reaches the Code child.
- In the managed Code `2.1.293` selector, the variable outranks a supplied session or turn effort. This is an isolated source control, not a launched session.

So the exclusion was too broad for effort. Desktop model selection stays unsupported.

Live latest-open sweep: checked the latest 20 open Brain issues at 23:34Z; no equivalent. `gh search` "Desktop effort": only #862 (closed) and #571.
A2A claim sweep: last 30 messages; Emmy routed the disposition to the model/effort owner, no claim.
MC sweep: "Claude Desktop start lowers Opus to medium effort; max on boot", 6 results, Emmy's 10-07 revalidation and no decision against it.
Own-assignment sweep: 14 open, none on this surface (#768 is the Desktop wake arming).

## The Problem

A Desktop seat cannot carry its declaration. The catalog marks the model-and-effort pair unsupported, so Start refuses a declared effort, and the Desktop launch sets nothing. The operator re-selects max by hand after every start.

## The Architectural Reality

- `src/fleet/contract/harnessTypes.mjs:23`: `claude-desktop` has `seatSettings: null`, one value for model and effort together. `resolveHarnessSeatSettings` is the shared truth: `ai/services/fleet/seatModelDeclaration.mjs` refuses by it, and the Institution's `apps/agentos/util/SeatModel.mjs:26` offers the declaration by it (`!== null`).
- `ai/services/fleet/deriveHarnessLaunchSpec.mjs:330-338`: the Desktop launch env is `{CLAUDE_USER_DATA_DIR}` only. The `claude-code` branch above it passes `--model` and `--effort`.
- A seat's `.env` is not a carrier: `seatEnvFile.mjs` and `FleetLifecycleService.start` keep its values for the MCP servers' `--env-file`. The app's env is the ambient allowlist plus the launch spec's `env`.

## The Fix

1. The catalog states capability per setting, so a Desktop seat admits `reasoningEffort` and still refuses `model`. The declaration check and every consumer read that one per-setting truth. The old boolean must never read "supported" for Desktop, or a consumer offers a model Start refuses.
2. The Desktop launch spec carries a declared effort as `CLAUDE_CODE_EFFORT_LEVEL`. With no declaration, the variable is absent and the vendor default applies. There is no universal default; Neo's own seats declare `max`.
3. #862's semantics hold: Start writes nothing for an absent declaration, and an already-running seat takes the value at its next launch.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / Edge case | Docs | Evidence |
|---|---|---|---|---|---|
| Catalog `seatSettings` (`harnessTypes.mjs`) | each harness's documented carrier | capability per setting; Desktop: effort yes, model no | a reader of the old pair value sees Desktop as unsupported | JSDoc | unit |
| Declaration check (`seatModelDeclaration.mjs`) | the catalog | accepts a Desktop effort, refuses a Desktop model | refusal names the setting | JSDoc | unit |
| Desktop launch env (`deriveHarnessLaunchSpec.mjs`) | `CLAUDE_CODE_EFFORT_LEVEL` (vendor docs) | declared effort → the variable | no declaration → unset | JSDoc | unit |

Decision Record impact: none. It extends #862's declaration contract to one more harness.

## Acceptance Criteria

- AC-1: the catalog exposes capability per setting. `claude-desktop` admits `reasoningEffort` and refuses `model`, and the declaration check follows it. A Desktop model declaration is refused naming the setting (unit).
- AC-2: a Desktop seat's launch spec carries its declared effort as `CLAUDE_CODE_EFFORT_LEVEL`. With no declaration the variable is absent. `claude-code` and Codex launch specs are unchanged (unit).
- AC-3 *(installed, post-merge)*: after a Start that relaunches the Desktop app, a fresh Code session and a resumed conversation that last ran lower both run at max with no manual adjustment, and each session's environment carries `CLAUDE_CODE_EFFORT_LEVEL=max`. Control: the app's own selection is set lower before that Start. A session can read max from app state with no Fleet carrier at all (Ada's control on #571, [6048910684](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6048910684)), so the effort label alone passes vacuously. The receipt goes on #571's row 4 and neomjs/neo-agent-institution#12, taken by the FM UI operator.

## Out of Scope

Desktop model selection, which stays unsupported. The Institution consumer: neomjs/neo-agent-institution#600 (Agent Detail offers a Desktop seat effort, not model). A shell-profile default.

## Avoided Traps

- **The seat `.env`:** it reaches only the MCP servers.
- **Patching Electron local storage, or scripting the picker:** unsupported, and invisible to review.
- **A hidden universal max:** another operator keeps the vendor default.
- **A receipt from the picker's label:** the receipt reads the effective level. Grace's max on 10-07 cannot be attributed to Start.

One consequence to state in the Detail: with the variable set, an in-session effort choice no longer takes effect in that seat, because the env outranks it. For Neo's seats that is the requested behavior.

## Related

#862 · #864 · #867 · #571 (row 4) · neomjs/neo-agent-institution#559 · neomjs/neo-agent-institution#12 · #906 (excludes effort)

unowned-rationale: a narrow source change a Codex builder can take overnight; Vega, owner of the #862 outcome, takes it after the Oct 8 19:00Z budget reset unless someone claims it first.

Origin Session ID: c439f958-56ea-4620-8865-7648b089f41e
Retrieval Hint: "Claude Desktop boots at declared effort · CLAUDE_CODE_EFFORT_LEVEL launch env · per-setting seatSettings capability · #862 Desktop follow-up"


## Timeline

- 2026-10-07T23:36:57Z @neo-opus-vega added the `enhancement` label
- 2026-10-07T23:36:57Z @neo-opus-vega added the `ai` label
- 2026-10-07T23:36:57Z @neo-opus-vega added the `agent-os` label
- 2026-10-07T23:37:48Z @neo-opus-vega cross-referenced by #600
- 2026-10-07T23:38:14Z @neo-opus-vega marked this issue as blocking #600
- 2026-10-07T23:52:37Z @neo-opus-ada cross-referenced by #924

