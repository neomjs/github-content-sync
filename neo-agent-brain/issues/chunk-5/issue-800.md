---
id: 800
title: 'Preflight''s Residual-Owner accepts only a same-repo #N'
state: CLOSED
labels:
  - bug
  - ai
  - model-experience
assignees:
  - neo-opus-grace
createdAt: '2026-10-03T08:06:32Z'
updatedAt: '2026-10-03T10:36:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/800'
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
closedAt: '2026-10-03T10:36:25Z'
---
# Preflight's Residual-Owner accepts only a same-repo #N

## Context

There are three independent occurrences of the same friction, each captured but none filed:
- **2026-10-02, @neo-gpt-sophie (defect-note):** #748's hosted residual belonged to neomjs/neo-agent-institution#351.
- **2026-10-03 07:05Z, Grace, PR #791:** the post-merge plane check's honest owner was neomjs/neo-agent-institution#312. Worked around by discharging the check before merge.
- **2026-10-03 07:17Z, @neo-opus-ada (defect-note, #794):** an installed check owned by neomjs/neo-agent-institution#12.

Each time, `agent-preflight` reported the obligation as unowned. The author either names a same-repo stand-in, which is a false owner, or drops the obligation.

## The Problem

The owner parses as `#(\d+)` only, in `RESIDUAL_OWNER_LINE_PATTERN` (`ai/scripts/agent-preflight.mjs:134`) and `RESIDUAL_OWNER_INLINE_PATTERN` (`:140`). These came with the initial import (#13), when one repository held everything. Since the Engine/Brain/Institution split, a Brain fix whose residual is an installed check is the common FM v1 shape. Its honest owner is an Institution issue, and the lint cannot name it.

## The Architectural Reality

- An owner travels as a bare number through three places: `collectDeclaredResidualOwners` (`:670`), the close-target rule (`:787`, number equality with `Resolves #N`), and the injected `resolveOwnerState(number)` (`:797`).
- In production, `resolveOwnerState` is `resolveIssueState` (`:603`, wired at `:1262`). It reads `gh api repos/{owner}/{repo}/issues/N`, with gh's current-repo placeholders.
- CI's `PR body` job (`neo-agent-skills/scripts/check-pr-body.mjs`) checks anchors only and has no Residual-Owner rule. Only the local preflight refuses.

## The Fix

1. Both patterns accept an optional `owner/repo` prefix, so `Residual-Owner: neomjs/neo-agent-institution#12` parses as well as `#12`.
2. A declared owner becomes `{repo, number}`, where `repo` is null for this repository. The close-target rule compares repository and number, so a cross-repo `#N` that shares the close target's number is not the close target.
3. `resolveOwnerState` receives `{repo, number}`, and `resolveIssueState` reads `repos/<repo>/issues/<N>` for a named repository. The closed, missing, PR-versus-issue and could-not-verify outcomes hold per repository.
4. Messages quote the owner as written.
5. *(Added 2026-10-03, from @neo-opus-ada's defect-note on Institution #483.)* **`--pr-repo <owner/repo>`** names the repository a PR body belongs to, for a PR in another repository than the checkout. This tool lives in the Brain, so an Institution PR body is linted from a Brain clone, where `#12` and `Resolves #449` were read as the Brain's. With the flag, same-repo owners and the close target's AC read go to that repository.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `Residual-Owner:` declaration | `agent-preflight.mjs:134`, `:140` | `#N` or `owner/repo#N`, line and inline forms | an unparsable owner still reads as missing | JSDoc | unit |
| Close-target rule | `:787` | repository + number equality | — | JSDoc | unit |
| `resolveOwnerState` / `resolveIssueState` | `:603`, `:797`, `:1262` | `{repo, number}`; a named repository is read as named | could-not-verify stays a warning, never a verdict | JSDoc | unit (injected resolver and exec seam) |
| `--pr-repo` CLI option, `resolveTicketAcs({repo})` | `createProgram`, `runPrBodyGate` | the PR's repository for same-repo owners and the close target | absent: the checkout's repository, as before | `--help` + JSDoc | unit |

## Acceptance Criteria

- [ ] AC-1: `Residual-Owner: neomjs/neo-agent-institution#12` discharges an obligation, both in a Post-Merge Validation section and in the inline `Evidence: … Residual: …, Residual-Owner:` form (unit).
- [ ] AC-2: A cross-repo owner whose number equals the close target's is not refused as the close target, and a same-repo one still is (unit).
- [ ] AC-3: The state read follows the named repository: a closed cross-repo owner fails, an open one passes, and an unreadable one warns (unit).
- [ ] AC-4: Every existing same-repo arm in `agent-preflight.residualOwner.spec.mjs` passes unchanged.
- [ ] AC-5: With `--pr-repo neomjs/neo-agent-institution`, a body's `#N` owner and its close target are read in that repository, and no read uses the checkout's placeholders (unit).

## Out of Scope

- The CI `check-pr-body` job, which has no Residual-Owner rule.
- Deciding which repositories may own a residual: any `owner/repo` shape parses, and the state read decides whether it exists.

## Decision Record impact

None.

## Related

#13 (where the patterns came from) · PR #791 (the workaround) · #794 (Ada's occurrence) · the Evidence Ladder, whose one-line `Residual-Owner:` form this extends.

Live latest-open sweep: latest 20 open Brain issues at 2026-10-03T08:05:07Z; no equivalent. `gh search issues --owner neomjs "Residual-Owner"` (open): no ticket on the parser. A2A sweep: two defect-notes today (Ada 07:17Z, Grace 07:30Z), no claim. MC sweep: "agent-preflight residual owner cross-repo stand-in lint same-repo only", 6 results. Sophie's 10-02 capture was a defect-note with no ticket filed; no prior decision. Own-assignment sweep: none on this surface.

Origin Session ID: 9eba4853-ea86-428a-85f9-e9060002ca22
Retrieval Hint: "agent-preflight Residual-Owner cross-repo owner same-repo pattern close-target"

🖖 Grace (Claude Opus 5.5, Claude Code)


## Timeline

- 2026-10-03T08:06:32Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-03T08:06:34Z @neo-opus-grace added the `bug` label
- 2026-10-03T08:06:34Z @neo-opus-grace added the `ai` label
- 2026-10-03T08:06:34Z @neo-opus-grace added the `model-experience` label
- 2026-10-03T08:13:12Z @neo-opus-grace cross-referenced by PR #801
- 2026-10-03T08:52:37Z @neo-opus-grace referenced in commit `ff970a4` - "feat(preflight): a Residual-Owner may name another repository (#800)

agent-preflight parsed the owner as `#N` only, so a Brain fix whose
residual is an installed check owned by an Institution issue could
not name its honest owner. Both declaration shapes now accept
`owner/repo#N`; an owner travels as {repo, number}, the close-target
rule compares both, the state read goes to the named repository, and
an owner that spells out this repository (from the origin remote) is
still this repository's."
- 2026-10-03T10:14:04Z @neo-opus-grace referenced in commit `e33bda4` - "fix(preflight): an unread origin proves no owner foreign, and a coordinate is one owner/repo (#800)

resolveCurrentRepo reads every clone form GitHub documents, SSH over port 443 included, and anchors the host. An owner is foreign only when this repository is known and differs, so an unreadable origin keeps the close-target guard. Owner tokens and --pr-repo share one owner/repo grammar: no query, fragment, extra or dot segment, and a number ends at a word boundary. --pr-repo names the reads it retargets; the git gates still read the checkout."
- 2026-10-03T10:21:49Z @neo-opus-grace referenced in commit `ae05145` - "fix(preflight): a GitHub origin is read from its authority, not from a substring (#800)

The host must be exactly github.com, or ssh.github.com over ssh:, and the path exactly owner/repo. A subdomain, a look-alike host or github.com inside a path names no repository, so the unknown-context guard holds for them."
- 2026-10-03T10:36:25Z @tobiu referenced in commit `ce710b7` - "feat(preflight): a Residual-Owner may name another repository (#800) (#801)

* feat(preflight): a Residual-Owner may name another repository (#800)

agent-preflight parsed the owner as `#N` only, so a Brain fix whose
residual is an installed check owned by an Institution issue could
not name its honest owner. Both declaration shapes now accept
`owner/repo#N`; an owner travels as {repo, number}, the close-target
rule compares both, the state read goes to the named repository, and
an owner that spells out this repository (from the origin remote) is
still this repository's.

* fix(preflight): an unread origin proves no owner foreign, and a coordinate is one owner/repo (#800)

resolveCurrentRepo reads every clone form GitHub documents, SSH over port 443 included, and anchors the host. An owner is foreign only when this repository is known and differs, so an unreadable origin keeps the close-target guard. Owner tokens and --pr-repo share one owner/repo grammar: no query, fragment, extra or dot segment, and a number ends at a word boundary. --pr-repo names the reads it retargets; the git gates still read the checkout.

* fix(preflight): a GitHub origin is read from its authority, not from a substring (#800)

The host must be exactly github.com, or ssh.github.com over ssh:, and the path exactly owner/repo. A subdomain, a look-alike host or github.com inside a path names no repository, so the unknown-context guard holds for them."
- 2026-10-03T10:36:26Z @tobiu closed this issue

