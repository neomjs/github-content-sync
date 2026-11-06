---
id: 1498
title: The deployment snapshot publishes Docker log frame headers as text
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-06T02:28:04Z'
updatedAt: '2026-11-06T10:20:20Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1498'
author: neo-opus-ada
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
closedAt: '2026-11-06T10:20:20Z'
---
# The deployment snapshot publishes Docker log frame headers as text

## Context

`get_deployment_state_snapshot` serves two log fields per plane service: the bounded tail (`services[].logs.text`) and the startup head (`services[].logs.startup.text`). On the live local plane, both hold Docker's multiplexed log stream instead of log text. The snapshot was read at 2026-11-06T02:2xZ, 26 s old:
- chroma's `text` is 120 frames whose size fields chain to its last byte, with stdout and stderr interleaved;
- fleet-server's `text` is 91 frames, chaining the same way;
- chroma carries 1,028 control characters in 6,109 chars, and the orchestrator 840 in 20,597.

## The Problem

- Every MCP reader of the snapshot gets an 8-byte binary header before each write, decoded as text.
- A size byte ≥ 0x80 is invalid UTF-8 and decodes as U+FFFD. That destroys the frame's length, which is why the orchestrator's and mc-server's chains break after 1 and 2 frames.
- A 10-byte frame's size byte is 0x0A and decodes as a newline. `countLines`, and the line-boundary cut that `boundUtf8Tail` / `boundUtf8Head` guarantee, therefore count header bytes.
- stdout and stderr cannot be told apart.
- Redaction over this text misses a credential split across two frames. That is #27's contract, and it becomes a pinned step there.

## The Architectural Reality

- `DeploymentRuntimeAccessService#readTargetLogs` returns `response.body`.
- `dockerRequest` decodes the whole response as one UTF-8 string (`Buffer.concat(chunks).toString('utf8')`).
- No compose service sets `tty`, so `/containers/{id}/logs` arrives multiplexed. Each frame is one stream byte (1 stdout, 2 stderr), three zero bytes, a 4-byte big-endian payload size, then the payload.
- `DeploymentStateBridgeService` is the only consumer of logs, through the tail (`summarizeLogs`) and the startup head (`readStartupLogHead`). The other `readObserve` callers read `inspect`, `stats` or `system-df`.

## The Fix

Demultiplex in the runtime-read owner, on the bytes, before anything decodes them:
- `readTargetLogs` takes the body as a Buffer.
- It parses frames when the whole body chains as frames from offset 0. A TTY container's raw stream does not chain and passes through as before.
- It decodes each stream with its own `StringDecoder`, so a character split across two writes survives.
- `logs` stays a string: the plain text, in arrival order.

A pure helper beside the service owns the parse, following its siblings' precedent (`memoryPressureDisposition.mjs`, `deploymentDurabilityPosture.mjs`). The owning folder is `ai/daemons/orchestrator/services` (structure map).

## Acceptance Criteria

- [ ] `readTargetLogs` returns demultiplexed text for a multiplexed body, and passes a raw (TTY) body through unchanged.
- [ ] Unit fixtures cover a frame with size ≥ 0x80, a frame of size 10, a UTF-8 character split across two frames, and interleaved stdout/stderr frames.
- [ ] Post-merge, once deployed: the snapshot's `logs.text` and `logs.startup.text` carry no frame headers, measured with the same chain count as above.

## Out of Scope

- #27's serving verb and its redaction.
- Publishing stdout and stderr as separate fields.

## Related

- #27: its log-serving contract needs demux before redaction.
- #83.

Live latest-open + closed sweep: checked the latest 20 open Brain issues, plus all-state searches ("docker logs multiplexed", "log frame header", "snapshot log text control"), at 2026-11-06T02:26Z. No equivalent found. A2A: no claim on this surface. Memory Core: no prior sighting. Own assignments: none on this surface.

Origin Session ID: 89c130ad-8235-41ef-914c-800c028ac868
Retrieval Hint: "deployment snapshot docker log multiplexed frame header readTargetLogs demux"

## Timeline

- 2026-11-06T02:28:04Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-06T02:28:05Z @neo-opus-ada added the `bug` label
- 2026-11-06T02:28:05Z @neo-opus-ada added the `ai` label
- 2026-11-06T02:28:05Z @neo-opus-ada added the `agent-os` label
- 2026-11-06T02:28:49Z @neo-opus-ada cross-referenced by #27
- 2026-11-06T02:34:20Z @neo-opus-ada cross-referenced by PR #1499
- 2026-11-06T03:24:28Z @neo-opus-ada referenced in commit `22451a1` - "fix(orchestrator): the log read honors declared framing and daemon errors (#1498)

A Content-Type of multiplexed-stream, or raw-stream from API 1.42, now decides the framing. Only an undeclared response falls back to the byte chain, so a raw TTY stream whose bytes look framed keeps every byte. A stream-3 frame is a daemon error that Docker sends after HTTP 200. It now fails the read with the daemon's message, instead of passing raw headers through as log text."
- 2026-11-06T10:20:20Z @tobiu referenced in commit `765cbf5` - "fix(orchestrator): a logs read strips Docker's frame headers (#1498) (#1499)

* fix(orchestrator): a logs read strips Docker's frame headers (#1498)

A container without a TTY sends /containers/{id}/logs as a multiplexed stream, with an 8-byte binary header before every write. readTargetLogs decoded the whole body as UTF-8, so the deployment snapshot published those headers as text: U+FFFD for a size byte of 0x80 or more, and a newline for a size of 10. The read now asks for bytes and cuts the frames before decoding, each stream through its own StringDecoder. A body that is not a frame chain, such as a TTY's raw stream, decodes as before.

* fix(orchestrator): the log read honors declared framing and daemon errors (#1498)

A Content-Type of multiplexed-stream, or raw-stream from API 1.42, now decides the framing. Only an undeclared response falls back to the byte chain, so a raw TTY stream whose bytes look framed keeps every byte. A stream-3 frame is a daemon error that Docker sends after HTTP 200. It now fails the read with the daemon's message, instead of passing raw headers through as log text."
- 2026-11-06T10:20:21Z @tobiu closed this issue

