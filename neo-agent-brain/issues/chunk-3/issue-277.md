---
id: 277
title: The restoration runbook tells operators to run a script that exists nowhere
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-08-31T07:15:23Z'
updatedAt: '2026-08-31T08:57:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/277'
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
closedAt: '2026-08-31T08:57:02Z'
---
# The restoration runbook tells operators to run a script that exists nowhere

## Context

`learn/agentos/tooling/RestorationRunbook.md:103` hands the operator a copy-pasteable command:

```bash
npm run ai:check-backup-integrity
```

and `:163` points at it again as the way to resolve a bundle whose restorability is unknown. **That script is defined in no `package.json` — not in this repo, and not in the Engine.** An operator following the runbook during a recovery gets `Missing script`.

Surfaced while gathering evidence for #270, whose whole subject is a backup that cannot be trusted. This is the recovery path for exactly that situation.

## The Problem

The Brain unit suite is **red on `dev`** because of it. `test/playwright/unit/ai/scripts/maintenance/backup.spec.mjs:1079` — *"a named npm script invokes the timeline tool"* — asserts:

```js
expect(scripts['ai:check-backup-integrity']).toBeDefined();
expect(scripts['ai:check-backup-integrity']).toContain('backupCorruptionTimeline.mjs');
expect(scripts['ai:check-backup-integrity']).not.toMatch(/^\s*[A-Z_][A-Z0-9_]*=/);
```

Verified by stashing an unrelated branch and re-running against clean `dev`: **1 failed / 47 passed**, this test.

**The pairing is what makes it sharp.** The *sibling* canary — *"the restoration runbook names the script and when to run it"* — **passes**, because the runbook does name it. So the suite currently reports: *the runbook correctly documents the command* ✅ and *the command does not exist* ❌. The green half certifies the documentation of a thing whose absence the red half is reporting.

The tool itself is present and healthy: `ai/scripts/maintenance/backupCorruptionTimeline.mjs` (~15KB), *"Read-only backup-corruption timeline diagnostic — artifact-verified, not manifest-trusted."* Only its entry point is missing.

## The Architectural Reality

- Tool and canary both arrived in `11552b0 feat(brain): receive the Agent OS (#13)` — the repo split.
- The `package.json` entry did **not** come with them, and the Engine does not have it either (`ai:check-backup-integrity` absent from both; the Engine has no backup-related scripts at all).
- So this is not a script that moved to the wrong repo. It is one that **existed only as an assertion and a runbook instruction**, never as a definition — the canary named itself *"reachable by name"* and was the only thing that could have caught that, which it did, into a red nobody dispositioned.
- Sibling convention, from this repo's own `scripts`: `ai:kb-push-client` → `node ./ai/scripts/maintenance/kbPushClient.mjs`. No env prefix, matching the canary's cross-platform clause.

## The Fix

Add the missing entry to `package.json`, following the sibling convention exactly:

```json
"ai:check-backup-integrity": "node ./ai/scripts/maintenance/backupCorruptionTimeline.mjs"
```

Then confirm the tool actually runs under that invocation rather than assuming the canary's three assertions are the whole contract — a script that satisfies the string checks but errors on launch would leave the runbook just as wrong.

## Decision Record impact

`none` — restores an entry point the substrate already documents and tests for.

## Acceptance Criteria

- [ ] `npm run ai:check-backup-integrity` executes `backupCorruptionTimeline.mjs` and exits cleanly on a host with a backup root.
- [ ] `backup.spec.mjs:1079` passes; the Brain unit suite loses this standing red on `dev`.
- [ ] The invocation carries no shell env prefix, per the canary's cross-platform clause and every sibling `ai:` entry.
- [ ] The runbook's `:103` command is executable as written — verified by running it, not by re-reading the runbook.

## Out of Scope

- #270's collapsed-capture refusal (PR #275). Adjacent, deliberately separate: that ticket stops a bad bundle being written, this one restores the tool for inspecting bundles after the fact. I flagged this in #275's body rather than folding it in.
- Any change to `backupCorruptionTimeline.mjs` itself.
- The runbook's prose, which is correct — it is the definition that is missing, not the documentation.

## Avoided Traps

- **Silencing the canary.** Rejected: it is the only thing that detected this, and it named itself *"reachable by name"* for precisely this reason. Deleting it would convert a red into an invisible absence.
- **Adding the script without running it.** Rejected: the assertions are string checks. Satisfying them while the tool errors on launch reproduces the defect one layer in.

## Related

- #270 / PR #275 — the collapsed-capture refusal that surfaced this
- #233 (closed) — *"A guard that prevents a false backup alarm cannot reach…"*, same subsystem

Live latest-open sweep: checked the latest 15 open Brain issues at 2026-08-31T07:14:48Z plus a search for `check-backup-integrity` across all states; no equivalent exists.

Authored by Claude Opus 5 (Claude Code), @neo-opus-ada.
Origin Session ID: 698ab063-0650-4531-a2b4-53ca4268509c
Retrieval Hint: "ai:check-backup-integrity npm script missing restoration runbook backupCorruptionTimeline canary red on dev"

## Timeline

- 2026-08-31T07:15:23Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-08-31T07:15:25Z @neo-opus-ada added the `bug` label
- 2026-08-31T07:15:25Z @neo-opus-ada added the `ai` label
- 2026-08-31T07:15:25Z @neo-opus-ada added the `testing` label
- 2026-08-31T07:15:25Z @neo-opus-ada added the `agent-os` label
- 2026-08-31T07:17:40Z @neo-opus-ada cross-referenced by PR #278
- 2026-08-31T07:18:03Z @neo-opus-ada cross-referenced by #270
- 2026-08-31T08:57:02Z @tobiu closed this issue

