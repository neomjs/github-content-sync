---
id: 1859
title: Antigravity seats launch with --remote-debugging-port=0 now that 2.33.0 makes it opt-in
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-19T13:44:29Z'
updatedAt: '2026-11-19T16:57:14Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1859'
author: neo-opus-grace
commentsCount: 0
parentIssue: 1531
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-19T16:57:14Z'
---
# Antigravity seats launch with --remote-debugging-port=0 now that 2.33.0 makes it opt-in

## Context

On 2026-11-19 Antigravity 2.33.0 refused every Gemini seat's project binding after 60 s (#1856). Julia diffed the two admitted bundles ([her #1856 comment](https://github.com/neomjs/neo-agent-brain/issues/1856#issuecomment-7010528468)), and Grace confirmed it on the copies (`app.asar`):
- 2.32.2 calls `appendSwitch('remote-debugging-port', '0')` itself, so every launch opened a DevTools endpoint and wrote `DevToolsActivePort`.
- 2.33.0 drops that call and adds `removeStaleActivePortFile`. Without the switch, it deletes `<userData>/DevToolsActivePort` and drops `AGY_BROWSER_ACTIVE_PORT_FILE` from the language server's environment.

The Fleet launches Antigravity with `--user-data-dir=<home>` only. On 2.33.0 no endpoint appears, so `waitForOwnedProfileEndpoint` waits out its budget and refuses with `stale-endpoint`.

## The Problem

On 2.33.0 three Fleet surfaces lose the endpoint: the project binding (`antigravityProjectBinding.mjs`), the neutral MCP invokers that read `AGY_BROWSER_ACTIVE_PORT_FILE` (`generateAntigravityMcpFiles.mjs`, which exit 78 without it), and the native CDP wake (`withOwnedAntigravityCdp`). Today the seats run on the parked fallback 2.32.2 (#1856's mitigation).

## The Architectural Reality

- `ai/services/fleet/deriveHarnessLaunchSpec.mjs`: `HARNESS_LAUNCH_CONTRACTS.antigravity.modeArgs` is `[]`.
- `FleetLifecycleService#runsSeatLaunch` recognises a live seat by its launch-command prefix and `--user-data-dir` word, so an added argument leaves running 2.32.2 seats recognised (Julia's read of `FleetLifecycleService.mjs` 378–393, 2832–2847).
- Design authority: n/a. 2.33.0 changed the vendor's contract; this adopts its supported opt-in.

## The Fix

Set `modeArgs: ['--remote-debugging-port=0']` for Antigravity, with its `deriveHarnessLaunchSpec` spec. On 2.32.2 and earlier, which add the switch themselves only when it is absent, this changes nothing. Optionally, `probeAntigravityRuntime` asserts on the copy that the bundle still reads the switch (`DevToolsActivePort`, `AGY_BROWSER_ACTIVE_PORT_FILE`), so a vendor that removes the opt-in is refused at admission.

## Acceptance Criteria

- [ ] An Antigravity Start passes `--remote-debugging-port=0`; the launch-spec spec pins it.
- [ ] Running 2.32.2 seats stay recognised as Fleet seats across the change.

## Post-Merge Validation

- [ ] [L4-deferred — after the Fleet install] The parked 2.33.0 copy goes back into the store, and a Gemini seat starts and binds on it; #1857's log line shows no refusal. Owner: Grace, with #1856. (Moved here from the ACs on 19 Nov: it can only run on an installed Fleet.)

## Out of Scope

The first-Start proof and fallback (#1856) and the refusal log line (#1857).

## Related

Parent #1531 · #1856 · #1857

handoff: @neo-gemini-julia. She found the change and offered to take a clean slice; it is her family's harness. If she prefers not to, Grace takes it.

Live latest-open + closed sweep: searched "remote-debugging-port" (open and closed) at 2026-11-19T13:4xZ: only #1856 matches. No in-flight claim beyond #1856's.

Origin Session ID: 5a375982-a381-43b6-a753-4fb8e933059d

Retrieval Hint: "Antigravity 2.33.0 remote-debugging-port opt-in, DevToolsActivePort removed, stale-endpoint, launch contract modeArgs"

🖖 Grace (Claude Opus 5.5, Claude Code)

## Timeline

- 2026-11-19T13:44:31Z @neo-opus-grace added the `bug` label
- 2026-11-19T13:44:31Z @neo-opus-grace added the `ai` label
- 2026-11-19T13:44:32Z @neo-opus-grace added the `regression` label
- 2026-11-19T13:44:32Z @neo-opus-grace added the `agent-os` label
- 2026-11-19T13:44:56Z @neo-opus-grace added parent issue #1531
- 2026-11-19T13:45:28Z @neo-opus-grace cross-referenced by #1856
- 2026-11-19T14:07:18Z @neo-gemini-julia cross-referenced by PR #1858
- 2026-11-19T14:10:01Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-19T14:32:47Z @neo-gemini-julia cross-referenced by PR #1860
- 2026-11-19T16:15:21Z @neo-gemini-julia referenced in commit `30b2385` - "test(fleet): move seat-recognition specs to owning suites and align modeArgs comment (#1859)"
- 2026-11-19T16:57:13Z @tobiu referenced in commit `936a363` - "feat(fleet): pass --remote-debugging-port=0 and probe Antigravity markers (#1859) (#1860)

* feat(fleet): pass --remote-debugging-port=0 and probe Antigravity markers (#1859)

* test(fleet): move seat-recognition specs to owning suites and align modeArgs comment (#1859)"
- 2026-11-19T16:57:15Z @tobiu closed this issue

