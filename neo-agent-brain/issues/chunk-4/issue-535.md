---
id: 535
title: The WriteAhead spec's fixture token fails the shared Secrets scan on every PR
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-09-26T07:32:59Z'
updatedAt: '2026-09-26T07:39:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/535'
author: neo-opus-ada
commentsCount: 1
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
closedAt: '2026-09-26T07:39:48Z'
---
# The WriteAhead spec's fixture token fails the shared Secrets scan on every PR

## Context

- **What changed.** #523 / PR #524 (merged 2026-09-26T07:20Z) made every Brain PR call the shared PR baseline. Its `Secrets` job runs `neo-agent-skills-secrets --all`, which scans the whole tree, not just the diff.
- **What it finds on Brain `dev`.** On `dev`@1ac9492 the scan finds `2 credential-shaped literal(s) in 1 file(s)`, so the job is red on every Brain PR from now on. Evidence:
  - the first run after the merge: Brain PR #525, job 108362759972
  - the same scanner (`check-secrets.mjs --all`) run locally on a clean `git archive origin/dev`

## The Problem

Both hits are in `test/playwright/unit/ai/services/memory-core/MemoryService.WriteAhead.spec.mjs`, in the arm "a failed presence terminal carries a sanitized reason, not a bare constant":
- **line 377:** the thrown error's message carries a `ghp_` + 36-character token
- **line 397:** `expect(reason).not.toContain(<the same literal>)`

The token is an obvious fixture: the 36 characters after the prefix are `AAAABBBB…IIII`. But it has the exact shape of a classic GitHub PAT, which is what the scanner flags, and it should.

The cost: a red `Secrets` check on every PR buries the one PR that ever commits a real token.

## The Architectural Reality

- The arm exists to prove that `redactReadFailure` (`ai/services/fleet/redactReadFailure.mjs`) masks the credential family. So the token must stay **token-shaped at runtime**. Nothing requires it to be a literal in source.
- The scanner's documented escape is a `secret-scan-ok: <reason>` marker on each line. Building the value at runtime needs no exemption at all.

## The Fix

Build the fixture once in the arm by joining its two parts (`['ghp', 'AAAABBBB…IIII'].join('_')`), and use the constant at both sites. The runtime value is byte-identical, and the source holds no literal.

## Acceptance Criteria

- [ ] `neo-agent-skills-secrets --all` reports no hits on the branch. The shared `Secrets` job is green on the PR.
- [ ] The arm still passes, with the same runtime token (the `unit` job).

## Out of Scope

- Scanner heuristics, such as ignoring low-entropy tokens. That is a neo-agent-skills question.

## Avoided Traps

- **`secret-scan-ok` markers on both lines.** They work, but they add two exemptions where the value never needed to be a literal.
- **Shortening the token below the PAT shape.** It could stop exercising the redaction the arm exists for.

## Related

- #523 / #524 made the shared baseline run on Brain PRs.
- #525 is the first PR where the red job showed.

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-09-26T07:32Z; no equivalent found. A2A: no claim on the Secrets job.

Origin Session ID: 1b945fcf-1142-475f-8007-ac18d51c069a


## Timeline

- 2026-09-26T07:32:59Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-26T07:33:00Z @neo-opus-ada added the `bug` label
- 2026-09-26T07:33:00Z @neo-opus-ada added the `ai` label
- 2026-09-26T07:33:00Z @neo-opus-ada added the `testing` label
- 2026-09-26T07:34:09Z @neo-opus-ada cross-referenced by PR #536
### @neo-opus-ada - 2026-09-26T07:39:47Z

Closing: invalid premise. The scan behind this ticket read a stale `origin/dev` (1ac9492). By the time I filed it, #524 (09ee6fa) had already marked both fixture lines `secret-scan-ok`. Fresh dev at 4d30321: `check-secrets --all` → `1874 file(s), no credential-shaped literal`. PR #536 is closed.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-09-26T07:39:48Z @neo-opus-ada closed this issue

