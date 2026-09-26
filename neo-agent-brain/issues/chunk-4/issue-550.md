---
id: 550
title: 'The Fleet''s wake-hook env drops NEO_AGENT_IDENTITY, so the hook throws'
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-26T18:45:52Z'
updatedAt: '2026-09-26T18:46:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/550'
author: neo-opus-grace
commentsCount: 0
parentIssue: 503
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
# The Fleet's wake-hook env drops NEO_AGENT_IDENTITY, so the hook throws

## Context

Found while reviewing #548: I executed the generated `write-wake-envelope` hook under the env its only production caller builds. On `dev` `6c65653`, `FleetLifecycleService#bootstrapOpenCodeWakeRoute` (`ai/services/fleet/FleetLifecycleService.mjs` ~:1176) gives the hook:

```js
const hookEnv = {};
for (const key of AMBIENT_ENV_ALLOWLIST) { if (env[key] !== undefined) hookEnv[key] = env[key]; }
hookEnv.OPENCODE_SERVER_USERNAME = env.OPENCODE_SERVER_USERNAME;
hookEnv.OPENCODE_SERVER_PASSWORD = env.OPENCODE_SERVER_PASSWORD;
```

`AMBIENT_ENV_ALLOWLIST` is `HOME, LANG, LC_ALL, LOGNAME, PATH, SHELL, TERM, TMPDIR, USER` (:76). The hook `generateOpenCodeSeatConfig` emits refuses to run without the seat identity:

```js
if (!seatIdentity) throw new Error('write-wake-envelope: NEO_AGENT_IDENTITY must be set in the environment');
```

**Measured** by executing the hook as a child process with exactly that env shape (allowlist + credential pair): it throws `NEO_AGENT_IDENTITY must be set` and writes no envelope. With `NEO_AGENT_IDENTITY` added, it writes the envelope. The child env already carries the identity (`env[AGENT_IDENTITY_ENV_VAR] = agentIdentity`, :520); the hook env drops it. Both sides arrived together in #13 (`11552b0`), so a Fleet-managed OpenCode seat's wake route cannot have reached `ready` through this path since then.

Unmeasured: whether any Fleet-managed OpenCode seat is running today. Hand-launched seats do not go through this bootstrap.

## The Problem

The bootstrap fails closed as designed. The hook throws, the execFile rejects, the stage is `wake-envelope hook`, the envelope is removed and the route degrades. So nothing is silently wrong at runtime, but the route never becomes deliverable, and the reason sits in a stderr nobody reads.

**Why no test caught it:** `test/playwright/unit/ai/FleetLifecycleService.spec.mjs` stubs `openCodeHookExecFileFn` with a fake that writes an envelope, a pre-identity one with no `agentIdentity`, whatever env it is handed. The real generated hook never meets the real hook env anywhere. That is the #508 shape: an instrument that reads as coverage without being coverage. The stub also certifies a schema the reader has refused since #510/#529. And the spec is not on `brain-unit.yml`'s run list (the "move-first Brain smoke"), so CI does not execute it at all.

## The Architectural Reality

- **Hook contract**: `generateOpenCodeSeatConfig.mjs` `renderWakeHook()`. Credentials come from the env, never argv. `NEO_AGENT_IDENTITY` is read from the env "because the seat .env already carries it — no new flag". The stamp exists so the reader can refuse another seat's envelope: the two-seats-one-envelope class, #19.
- **Caller contract**: `bootstrapOpenCodeWakeRoute`'s comment says "The generated hook needs only benign process-runtime vars plus its own server credential pair. Repository/MCP/Bridge credentials belong to the harness child and must not fan out into this auxiliary process." The identity is none of those: it is the definition's canonical GitHub login, already non-secret on the child env and stamped into the envelope file.
- **Launch identity**: `start()` binds `NEO_AGENT_IDENTITY` to the definition's `githubUsername`. Reserved class #4 means `launch.env` can never pre-load it, and a spec pins that. So the value to pass is exactly the one already on `env`.

## The Fix

In `bootstrapOpenCodeWakeRoute`, add the identity to the hook env beside the credential pair: `hookEnv[AGENT_IDENTITY_ENV_VAR] = env[AGENT_IDENTITY_ENV_VAR]`. Update the comment so the hook env reads "benign runtime vars, the seat identity, and the server credential pair". Nothing else crosses: MCP, Bridge and PAT credentials stay out.

Replace, or add beside, the stub arm: an arm that generates the real hook through `generateOpenCodeSeatConfig`, writes it to the instance home, and lets the service run it through the real `execFile` with the env it builds. The route then reaches `ready` with an envelope whose `agentIdentity` is the definition's identity. That arm is red on `dev`: the hook throws and the route degrades.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| The wake hook's process env | `FleetLifecycleService#bootstrapOpenCodeWakeRoute` (`hookEnv`) | carries `NEO_AGENT_IDENTITY` (the child env's value) beside the allowlist and the server credential pair | none: an absent identity keeps failing closed in the hook, as today | the method's env comment names the identity | red-first arm executing the real generated hook |
| The Fleet-managed envelope | `generateOpenCodeSeatConfig` `renderWakeHook()` | unchanged: `agentIdentity` stamped from the env | unchanged | — | the same arm asserts the stamped identity |

## Decision Record impact

`aligned-with` ADR 0034's secret boundary. The identity is not a credential, and no credential class widens. `none` for ADR 0019: no AiConfig read or leaf is touched.

## Acceptance Criteria

- [ ] `bootstrapOpenCodeWakeRoute` passes `NEO_AGENT_IDENTITY` to the hook, and no other non-allowlisted key beyond the existing credential pair.
- [ ] Red-first, measured: an arm executes the real generated hook with the env the service builds. It is red on `dev` (the route degrades at stage `wake-envelope hook`) and green after, with `agentIdentity` equal to the definition's identity.
- [ ] No arm certifies a pre-identity envelope: the existing stub either writes the identity-stamped schema or is replaced.
- [ ] The arm runs in CI: its spec is on `brain-unit.yml`'s run list.
- [ ] Post-merge L3: a Fleet-managed OpenCode seat's `wakeRoute` reaches `ready` with an identity-stamped envelope. Residual-Owner: #503.

## Out of Scope

- **Installing the wake-envelope plant** (#532 / #548). That PR's install path depends on the hook running, but it is its own change.
- **Growing `brain-unit.yml` beyond this spec.** The smoke list's reach is a separate question.
- **Hand-launched seats**, which never run this bootstrap.

## Avoided Traps

- **Making the hook's identity optional.** Rejected: the stamp is the reader's refusal key for another seat's envelope (#19); an optional stamp re-opens that class.
- **Passing the identity as a new `--agent-identity` flag.** Rejected: it changes the hook's documented env-only contract for no gain. The value is already on the child env, and argv is ps-visible.
- **Forwarding the whole child env.** Rejected: that fans the PAT, MCP and Bridge credentials into an auxiliary process, which is exactly what the allowlist exists to stop.

## Related

- #503: parent class, a wake subscription that reports itself deliverable while dispatch fails
- #548 / #532: the plant install, which surfaced this (review 5326941074)
- #19: why the identity stamp exists
- #508: the same false-assurance shape
- #510 / #529: the reader contract the stub's pre-identity envelope predates

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-09-26T18:44:27Z; the wake-class neighbours (#549, #547, #532, #514, #513, #503) cover the plant, the reader path and the envelope schema, and none covers the hook env.
A2A in-flight sweep: the last 30 inbox messages at 18:45Z hold no `[lane-claim]` or `[lane-intent]` on the hook env or the lifecycle bootstrap.
MC sweep: `query_raw_memories("Fleet-managed OpenCode wake route degraded hook throws NEO_AGENT_IDENTITY must be set hookEnv allowlist")`, 6 results, no prior decision on the hook env (nearest: the 09-25 schema-drift finding that became #503/#529, about the plant rather than this caller). Org-wide `gh search issues` for the hook identity: no hits.
Own-assignment sweep: 22 open on me in this repo; the wake-surface bodies (#30, #68, #69, #79, #118) do not mention the hook env or `NEO_AGENT_IDENTITY`.
Structure map: `FleetLifecycleService.mjs` and `generateOpenCodeSeatConfig.mjs` both sit in `ai/services/fleet/`, so there is no placement change.

Origin Session ID: 6408fcd4-3571-4ec2-8009-b4dae5d18917

Retrieval Hint: "write-wake-envelope NEO_AGENT_IDENTITY must be set hookEnv AMBIENT_ENV_ALLOWLIST bootstrapOpenCodeWakeRoute degraded"


## Timeline

- 2026-09-26T18:45:54Z @neo-opus-grace added the `bug` label
- 2026-09-26T18:45:54Z @neo-opus-grace added the `ai` label
- 2026-09-26T18:45:54Z @neo-opus-grace added the `agent-os` label
- 2026-09-26T18:46:02Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-26T18:46:05Z @neo-opus-grace added parent issue #503
- 2026-09-26T18:54:04Z @tobiu referenced in commit `81e7b7c` - "fix(fleet): the wake hook receives the seat identity it stamps (#550)

bootstrapOpenCodeWakeRoute built the hook env from the ambient allowlist
plus the server credential pair, while the generated hook refuses to run
without NEO_AGENT_IDENTITY, so every Fleet-managed OpenCode bootstrap
degraded at stage 'wake-envelope hook'. The identity (non-secret, already
on the child env) now crosses beside the credential pair.

The stub arm fakes the hook and could not see the refusal; it now writes
the identity-stamped schema and pins the hook env's key set. A new arm runs
the real generated hook through the real execFile: red on dev (degraded),
green with agentIdentity '@open-real'. The spec joins brain-unit.yml's run
list. 'Reserved class #4' is reworded, since the archaeology gate reads '#4'
as a ticket reference."
- 2026-09-26T18:55:36Z @neo-opus-grace cross-referenced by PR #551
- 2026-09-26T18:59:46Z @neo-opus-vega cross-referenced by #552

