---
id: 635
title: Codex Desktop capability checks fail in the Electron runtime
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-30T13:25:19Z'
updatedAt: '2026-09-30T13:25:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/635'
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
# Codex Desktop capability checks fail in the Electron runtime

## Context

During packaged FM validation on 2026-09-30, the corrected Codex CLI gate passed, but `probeCodexDesktopCapabilities` returned `required-app-bundle-resource-unavailable`. The same installed app passes the same probe under ordinary Node.

Paired Electron 43.5.0 control: `node:fs.statSync(app.asar).isFile()` is false, while `original-fs` reports true. Passing `original-fs` through the existing probe seam makes all capability checks pass.

## The Problem

Electron virtualizes ASAR archives as directories. The physical app-bundle inspector uses ordinary `node:fs` by default, so its regular-file guard rejects the archive before scanning the required markers. The current unit fixtures write plain marker text with an `.asar` suffix; they do not represent a real archive under Electron.

## The Architectural Reality

`ai/services/fleet/manageCodexDesktopRuntime.mjs` owns the physical archive inspection and an injectable `fsImpl` seam. `FleetLifecycleService` calls its default probe before desktop launch. Electron documents `original-fs` as the API for reading whole archives as files: [ASAR archives](https://www.electronjs.org/docs/latest/tutorial/asar-archives#treating-an-asar-archive-as-a-normal-file).

## The Fix

Make this physical-file probe default to Electron's built-in `original-fs` when running in Electron and ordinary Node fs otherwise. Keep the injected seam and every capability predicate. Do not toggle process-global ASAR behavior or weaken the file/marker checks. Make the existing fixture a valid minimal archive and validate the suite in both runtimes.

## Contract Ledger

| Surface | Authority | Behavior | Failure | Docs | Evidence |
|---|---|---|---|---|---|
| Installed bundle probe | `probeCodexDesktopCapabilities` | Read the physical archive in Node and Electron | Missing files/markers still fail closed | Probe JSDoc | Paired runtime fixture and installed-app checks |
| Host runtime | Electron's `original-fs` API | Scoped physical reader; no global mode mutation | Unsupported actual resources remain refused | Official documentation above | Electron's ordinary fs still sees an archive directory after the probe |

## Decision Record impact

None: corrects a platform filesystem primitive in the existing inspector; no AiConfig, launch permission, or storage-policy change. Existing Fleet helper directory is the owner; no new module is needed.

## Acceptance Criteria

- [ ] AC-1: The same valid fixture and installed Codex bundle pass the default capability probe under Node and Electron.
- [ ] AC-2: Missing capabilities, missing updater-disable predicate and invalid executable layouts remain rejected.
- [ ] AC-3: Electron's process-global ASAR handling is unchanged, and Node does not require an Electron-only module.
- [ ] AC-4: The existing regression fixture represents a valid archive, with Node CI and an explicit Electron runtime receipt.

## Out of Scope

CLI location (delivered by #633), provider login, new binary search/fallback policy, app-bundle mutation, global `process.noAsar` changes and complete first-session acceptance.

## Related

#632 · #633 · neomjs/neo-agent-institution#12

Sweeps: latest 20 open issues, recent all-state A2A, exact code/search scan and own assignments show no overlapping repair. #632 body read: it owns the CLI location, not archive inspection. Three memory framings returned no prior mapping of this runtime mismatch. Structure map was run in this onboarding lane; the existing Fleet helper owns the concern.

Origin Session ID: b0dd802b-6451-48ec-b789-d91e29a2b08e.
Retrieval Hint: Electron original-fs Codex capability ASAR virtual directory.

## Timeline

- 2026-09-30T13:25:19Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-30T13:25:21Z @neo-gpt-emmy added the `bug` label
- 2026-09-30T13:25:21Z @neo-gpt-emmy added the `ai` label
- 2026-09-30T13:25:21Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-30T13:34:03Z @neo-gpt-emmy cross-referenced by PR #637

