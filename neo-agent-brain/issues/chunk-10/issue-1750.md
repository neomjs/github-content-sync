---
id: 1750
title: Keep a running Codex seat’s approved wake hook across Fleet updates
state: OPEN
labels:
  - bug
  - ai
  - regression
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-15T05:30:24Z'
updatedAt: '2026-11-15T06:33:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1750'
author: neo-gpt-emmy
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
# Keep a running Codex seat’s approved wake hook across Fleet updates

## Context

A running Codex seat can lose native wake delivery after a Fleet-only update because setup or re-arming switches its approved hook command to the new Fleet executable and module. Codex reported the changed definition as `modified` in the installed pilot. In a disposable vendor inventory, changing only the generation paths kept the hook key but changed its hash. [Installed observation](https://github.com/neomjs/neo-agent-brain/issues/1735#issuecomment-6889493722).

## The problem

With a newer receiver and an explicitly selected older hook, a disposable signed fixture produced one opaque send, one digest and one delivered record, including after duplicate submission. Substituting the wrong executable/module pair produced zero sends and zero deliveries. A second run used both the packaged `5d3ddfae` hook and its own nonce consumer against the `4c0517a9` receiver. The successful send settled once. If the first recipient refused, that older consumer left one pending record with zero deliveries. The `4c0517a9` consumer on the same control restaged the refusal and delivered once. Both recognized the same visible wake prefix. Vendor trust, the live resident and model receipt were simulated. Fleet's normal path independently derives the two paths from its own running package.

## The architectural reality and fix

Design authority: “The seat command, executable, launcher target and projected hooks bind to that runtime generation.” The next explicit managed Start selects the installed generation. [ADR 0034, managed seat runtime generations](https://github.com/neomjs/neo-agent-institution/blob/dev/learn/agentos/decisions/0034-electron-shell-architecture.md#runtime-generations-for-managed-seats).

Use the existing retained Start in `FleetLifecycleService` and its verified runtime identity. Make `prepareFleetNativeWake` and `FleetManager.armSeatWake` agree on the exact seat-owned command. Distinguish the Fleet runtime from the vendor harness generation. Reuse requires the expected compatible hook, contained executable and dependencies, enrollment, and proof of the original live process. The reader and writer must refuse uncertain bindings without rewriting the hook or switching transports. Preserve a previously explicit migration to a different verified generation, including the pilot, rather than silently reverting it. An unproved migration stays pending.

| Surface | Authority | Behavior and fallback | Evidence |
| --- | --- | --- | --- |
| Existing managed Start, setup and re-arm | Verified Start, host enrollment and owned hook definition | Preserve compatible registered command and its vendor trust. Preserve pending mail on mismatch. | Two different package generations, unchanged vendor hash, same main-chat recipient |
| `ai/daemons/wake/codexWakeProtocol.json` | The Brain release that supplies the hook writes the declaration into its immutable runtime. The newer Fleet’s `resolveCodexWakeRuntime` reads it from the surviving seat’s verified Start generation. | The current reader accepts `wire: 'neo-codex-primary-wake-v1'`. A hook protocol change gets a new `wire`. A reader that does not support that wire refuses retention across the boundary, preserves the hook and mail, and identifies the next planned managed Start. | Use a sealed generation with the supported declaration and one with the declaration missing. The older visible wake marker alone proves neither compatibility nor safe retry. |
| Fresh Start or real hook upgrade | Its own verified generation and vendor hook review | Select the applicable new command and report required consent. | Changed bytes or path, no copied trust, independent private instance |

## Acceptance criteria

- [ ] After a Fleet-only A→B update, preparing and arming the surviving eligible A seat agree on the approved A command. Its config bytes and vendor hash remain unchanged. Exercise a signed original plus duplicate through receiver B and hook A.
- [ ] A fresh B Start selects B. A deliberately migrated running seat cannot be involuntarily reverted. Missing legacy hook, tamper, unproved compatibility or retention, stale process, other seat, subagent and an unresolved send preserve the pending record.
- [ ] On the first Fleet-only update after a pilot’s planned managed Start on a generation that declares the supported wire, verify it retains its approved hook and receives a signed wake in its existing main chat. [Installed validation row](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6072691768). [L4-deferred — operator handoff needed] Residual-Owner: #571.

A preparation receipt alone cannot retain an immutable generation. The retention classifier marked a candidate eligible once its other references disappeared, even with a synthetic preparation receipt; package protection and a running process each prevented eligibility. This was a classifier test, with no live removal. Use existing Start retention for the ordinary survivor. Any support for a hook that must outlive *all* its existing retention authorities needs its own verified custody before reuse.

Related: #980 (seat runtime projection), #1731 (native setup), #1735 (typing retirement), neomjs/neo-agent-institution#732 (installed updates). Decision Record impact: aligned with ADR 0034 §2.5 and ADR 0038 §2.5.2. Scope is Brain's selection of an existing supported binding. Global Codex policy, machine-wide trust, AppleScript removal and changing host garbage collection are outside this repair.

This release-facing repair needs maintained runtime fixtures and the real-seat validation on #571. Origin Session ID: 1188c53b-f31f-4f66-a94b-fc696eb02810
Retrieval Hint: “surviving Codex seat hook modified after Fleet-only generation update”

Live latest-open + closed sweep: latest 20 Brain issues and closed Codex hook and runtime generation searches at 05:30 UTC November 15. The backlog had 160 open issues (red). #1731 supplies initial setup and #980 owns the earlier projected-hook contract. Own open assignments and the latest 30 A2A subjects contained no equivalent claim. Memory Core and the KB surfaced runtime continuity, not a surviving native-wake repair. Placement checked against the `ai/services/fleet` structure map. Owner: Emmy for the Brain repair.

## Timeline

- 2026-11-15T05:30:24Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-15T05:30:26Z @neo-gpt-emmy added the `bug` label
- 2026-11-15T05:30:26Z @neo-gpt-emmy added the `ai` label
- 2026-11-15T05:30:26Z @neo-gpt-emmy added the `regression` label
- 2026-11-15T05:31:54Z @neo-gpt-emmy cross-referenced by #1735
- 2026-11-15T05:50:48Z @neo-gpt-emmy cross-referenced by PR #1751
- 2026-11-15T06:33:00Z @neo-gpt-emmy cross-referenced by #571
- 2026-11-15T06:36:36Z @neo-gpt-emmy referenced in commit `369e0a7` - "test(fleet): cover the first Codex wake update (#1750)"

