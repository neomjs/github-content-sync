---
id: 123
title: 'Nothing enforces the Design-authority line: a ticket that calls an existing behavior wrong can still ship without it'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-gemini-maryam
createdAt: '2026-09-25T21:35:34Z'
updatedAt: '2026-10-28T13:01:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/123'
author: neo-fable-mnemosyne
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
closedAt: '2026-10-28T12:44:28Z'
---
# Nothing enforces the Design-authority line: a ticket that calls an existing behavior wrong can still ship without it

## Context

PR #122 (#121, merged 2026-09-25) added the gate as prose: `ticket-create/references/design-authority.md` behind a §0 pointer, the `pr-review` §9.0 handle, the template input, the `ticket-intake` step-3 pointer. The reference carries its own sunset: *retire once a body lint enforces the line*. This is that leaf. Ada's round-1 note on #122 (non-blocking, filed here so it does not get lost): the trigger's exclusions — no crash, no red, no contradiction with the surface's own JSDoc — are the author's own assertions. An author who believes a JSDoc contradiction exists skips the line. A lint that only checks "line present when the author agreed it applies" enforces nothing.

## The Problem

Prose gates are discipline-only and measurably not firing (#61). A ticket that calls an existing behavior wrong and carries no line is exactly the specimen the gate came from (neomjs/neo#19231: a designed placement ticketed as a defect, a unit arm, green CI, a reviewer one read away from approving). Nothing mechanical catches it before a reviewer does, and the reviewer is the last line.

## The Architectural Reality

- `scripts/check-pr-body.mjs` lints PR bodies for their anchors (tested by `scripts/test-check-pr-body.mjs`, exercised in `skill-corpus.yml`); `scripts/check-ticket-archaeology.mjs` lints source comments. No ticket-body lint exists in `scripts/` today.
- Tickets are created through the github-workflow MCP `create_issue` (its server validates review bodies via `validate_pr_review_body`; there is no ticket-body validator) or by hand through `gh`.
- The line's grammar, from the reference: `Design authority: <the record sentence, quoted>` or `Design authority: none found (searched: …)`, in The Architectural Reality.
- Design authority: n/a — this ticket adds a check and calls no existing behavior wrong.

## The Fix

A body check with two arms, keyed the way Ada framed it:

1. The line is present → pass (either grammar).
2. The line is absent → the body must cite what excuses it: a crash (a fenced error block), a red (a spec path or a test title in the Problem/Context), or the contradicted JSDoc sentence (a quoted `@`-doc line). None of the three → fail, pointing at `design-authority.md`.

The check does not decide from prose whether a ticket "calls an existing behavior wrong"; arm 2 fires for every ticket without the line, and the three excuses are what a defect ticket carries anyway. Placement is decided in this leaf: a `scripts/check-ticket-body.mjs` shipped in the package for the MCP `create_issue` path to run, or a workflow on issue events; the package tarball allow-list in `skill-corpus.yml` grows by that one script. Then the reference's sunset line is retired.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `neo-agent-skills-ticket-body` (`scripts/check-ticket-body.mjs` CLI / `run(argv, options)`) | `#123` / `.agents/skills/ticket-create/references/design-authority.md` | Reads a ticket body from `--body-file <path>` or `stdin`; exits `0` when `findTicketBodyViolations({body}).ok` is `true` (satisfied by `design-authority` or `defect-excuse`). | Exits `1` and writes diagnostic pointing to `.agents/skills/ticket-create/references/design-authority.md` when neither arm holds or when `--body-file` cannot be read; throws on unknown CLI flags (`strict: true`). | JSDoc in `scripts/check-ticket-body.mjs` + `.agents/skills/ticket-create/references/design-authority.md` | `scripts/test-check-ticket-body.mjs` (unit + packed tarball CLI) |
| `findTicketBodyViolations({body})`, `hasDesignAuthorityLine(body)`, `hasDefectExcuse(body)` (`scripts/check-ticket-body.mjs`) | `#123` / `.agents/skills/ticket-create/references/design-authority.md` | Pure, network-free validator. Arm 1 (`design-authority`) passes when prose outside code fences/tables carries `Design authority: "<record sentence, quoted>"`, `Design authority: none found (searched: <non-placeholder list>)`, or `Design authority: n/a — <reason>`. Arm 2 (`defect-excuse`) passes when the body carries a fenced crash/error/stack block (`crash`), a spec/test path or named failing test title in `## Context` / `## The Problem` (`red`), or a quoted/blockquoted `@`-JSDoc clause (`jsdoc`). | Returns `{ok: false, satisfiedBy: null, excuses, violations: ['Ticket body carries neither a valid `Design authority:` line nor a defect excuse. See .agents/skills/ticket-create/references/design-authority.md.']}` when neither arm holds. | JSDoc in `scripts/check-ticket-body.mjs` + `.agents/skills/ticket-create/references/design-authority.md` | `scripts/test-check-ticket-body.mjs` |
| `.github/workflows/skill-corpus.yml#ticket-body` (`on.issues.types: [opened, edited, reopened]`) | `#123` (AC-2, AC-3) | Scope rule: runs on non-bot, non-`epic` issues authored by `neo-*` or labeled `ai`. Fetches live issue body via `github.rest.issues.get` into a temp file and runs `node scripts/check-ticket-body.mjs --body-file "${BODY_FILE}"`. On pass, removes label `needs-design-authority` if present (`404` ignored). | On validation failure, job fails (`exit 1`) and applies label `needs-design-authority` via `github.rest.issues.addLabels` (with `issues: write` scoped to `ticket-body`) so the failure reaches the ticket. | `.agents/skills/ticket-create/references/design-authority.md` | `scripts/test-check-ticket-body.mjs` (workflow contract) + Post-Merge Validation live test-issue receipt on `dev` |

## Acceptance Criteria

- [x] AC-1 Red-first arms: a body without the line and without any of the three excuses fails; the same body with the line passes (both grammars); the same body with a fenced error, a spec path, or a quoted JSDoc line passes.
- [x] AC-2 The check runs on the path tickets are created through (the MCP `create_issue` validator or a workflow on issue events), not only by hand — the placement named in the PR body with its wiring receipt.
- [x] AC-3 `design-authority.md`'s sunset sentence is retired in the same PR; the reference itself stays (it is what the failure message points at).
- [x] AC-4 The package tarball allow-list and the version bump follow the repository's rules (`skill-corpus.yml`).

## Out of Scope

Deciding from prose semantics whether a ticket calls an existing behavior wrong; the AC-table-vs-body check (#86); PR bodies (`check-pr-body.mjs` already covers them).

## Related

#121 (the gate), #86 (the class Ada named), #61 / #62 (discipline-only triggers), neomjs/neo#19231 (the specimen), neomjs/neo#19225 (the first ticket carrying the line).

Live latest-open sweep: the 20 latest open issues of this repository at 2026-09-25T21:33Z — no ticket-body lint (#86 = AC table vs body; #62 = sweeps vs closed state). A2A in-flight sweep: no claim on it. Unassigned on purpose: a gfi-door leaf for whoever takes the lint; its author (me) stewards the gate.

Origin Session ID: 4c0a5550-17ba-4752-9852-846afa537c86


## Timeline

- 2026-09-25T21:35:36Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-09-25T21:35:36Z @neo-fable-mnemosyne added the `ai` label
- 2026-09-25T21:36:00Z @neo-fable-mnemosyne cross-referenced by #121
- 2026-10-03T18:00:00Z @neo-fable-clio cross-referenced by #138
- 2026-10-09T21:43:51Z @neo-fable-clio cross-referenced by #150
- 2026-10-19T03:13:56Z @neo-fable-mnemosyne cross-referenced by #191
- 2026-10-28T02:01:11Z @neo-gemini-maryam assigned to @neo-gemini-maryam
### @neo-gemini-maryam - 2026-10-28T02:02:20Z

## Contract Ledger (`ticket-intake` §7 & `create-skill` PR-Open Gate)

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `scripts/check-ticket-body.mjs` (`neo-agent-skills-ticket-body`, new) | `neomjs/neo-agent-skills#123`, `.agents/skills/ticket-create/references/design-authority.md` | Pure transport-free validator (`findTicketBodyViolations`, `hasDesignAuthorityLine`, `hasDefectExcuse`, `run`) with two arms: (1) passes when content carries `Design authority:` in either canonical grammar (`Design authority: "<quoted record sentence>"` / `Design authority: n/a — <reason>` or `Design authority: none found (searched: <sources>)`); (2) when absent, passes if the body carries at least one of the three defect excuses: a fenced crash/error block, a spec path or failing test title in Problem/Context, or a quoted `@`-JSDoc line; otherwise exits `1` pointing to `.agents/skills/ticket-create/references/design-authority.md`. | Exits `1` with unreadable `--body-file` or missing design-authority / defect-excuse evidence; rejects unknown CLI flags via strict `parseArgs`. | JSDoc (`@summary`, `@param`, `@returns`) + `.agents/skills/ticket-create/references/design-authority.md` | `node scripts/test-check-ticket-body.mjs` |
| `.github/workflows/skill-corpus.yml` (`ticket-body` job + `corpus` step + tarball allow-list) | `neomjs/neo-agent-skills#123` AC-2 & AC-4, `.github/workflows/skill-corpus.yml` (`pr-body` sibling pattern at lines 99–138, 214–307) | Triggers on `issues: types: [opened, edited, reopened]`, gates `corpus` with `if: ${{ github.event_name != 'issues' }}`, and runs a `ticket-body` job (`if: ${{ github.event_name == 'issues' }}`) that fetches the live issue body via `github.rest.issues.get` and validates non-bot, non-epic `neo-` / `ai`-labeled issues with `scripts/check-ticket-body.mjs`. Adds `scripts/check-ticket-body.mjs` to the packed tarball allow-list and `scripts/test-check-ticket-body.mjs` to the exclusion list. | Bot issues and `epic`-labeled parent issues report unjudged (`exit 0`). | Inline workflow comments | `node scripts/test-check-ticket-body.mjs` + `skill-corpus.yml` CI |
| `.agents/skills/ticket-create/references/design-authority.md` | `neomjs/neo-agent-skills#123` AC-3 (verified at line 7) | Retires the trailing sunset sentence (`*Retire this reference once a body lint enforces the line on behavior-change tickets.*`) while retaining the 6-line reference that `check-ticket-body.mjs` points to on failure. | `lint-skill-corpus.mjs` fails if reference integrity or budget regresses. | `.agents/skills/ticket-create/references/design-authority.md` (`-88 B`) | `node scripts/lint-skill-corpus.mjs --base origin/dev` |
| `package.json` & `package-lock.json` | `neomjs/neo-agent-skills#123` AC-4, `scripts/check-version-bump.mjs` | Registers `"neo-agent-skills-ticket-body": "./scripts/check-ticket-body.mjs"` in `bin`, `"scripts/check-ticket-body.mjs"` in `files`, `node scripts/test-check-ticket-body.mjs` in `scripts.test`, and bumps `version` (`0.1.69` → `0.1.70`). | `check-version-bump.mjs` and tarball checks fail if missing. | `package.json` | `npm test` |

Authored by Maryam (Gemini 4 Argon, Antigravity). Session 58b07b45-1fe9-40ad-80f8-db0f380332b8. 🧭

- 2026-10-28T02:08:21Z @neo-gemini-maryam cross-referenced by PR #228
- 2026-10-28T11:10:45Z @neo-gemini-maryam referenced in commit `fdc799a` - "feat(skills): enforce Design-authority line and defect excuses on ticket bodies (#123)"
- 2026-10-28T12:19:18Z @neo-gemini-maryam referenced in commit `4fda73e` - "fix(skills): align design-authority reference and issue-failure label (#123)"
- 2026-10-28T12:44:28Z @tobiu referenced in commit `88c16f3` - "feat(skills): enforce Design-authority line and defect excuses on ticket bodies (#123) (#228)

* feat(skills): enforce Design-authority line and defect excuses on ticket bodies (#123)

* fix(skills): align design-authority reference and issue-failure label (#123)"
- 2026-10-28T12:44:28Z @tobiu closed this issue
- 2026-10-28T12:47:07Z @neo-gemini-maryam cross-referenced by #230

