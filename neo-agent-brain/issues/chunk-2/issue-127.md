---
id: 127
title: 'Contract-Ledger drift: multi-line signature coverage (brace-balanced accumulator) in findShippedSignature'
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
assignees: []
createdAt: '2026-06-27T05:23:24Z'
updatedAt: '2026-08-26T15:17:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/127'
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
---
# Contract-Ledger drift: multi-line signature coverage (brace-balanced accumulator) in findShippedSignature

Fast-follow from @neo-opus-vega's neomjs/neo#14189 review.

`findShippedSignature` (the author-side Contract-Ledger drift pre-flight) is **single-line by construction**: it matches `symbol(params) {|=>` on ONE added diff line. A multi-line definition — params spanning several lines, as large destructured params often are — is a silent **miss** (returns `null`), never a false signal.

**Why non-urgent:** V-B-A of the motivating drifts (#14104 `buildDimensionConsistencyDiagnosis({samples, observedAt, serviceId})`, neomjs/neo#14108 `computeResidueFingerprint({residue, strategyVersion, …})`) confirms they were **single-line** — so the detector catches its own motivating class today. The scope is now documented in the JSDoc (#14189 / d31cccaa5) so authors don't read a non-warn as proof of no drift.

**The work (when triggered):** a brace-balanced accumulator that joins added lines from `+…name(` until the parameter parens close, then matches — extending coverage to multi-line defs without losing the warn-only/no-false-positive posture.

**Sunset / retirement trigger:** implement when a multi-line destructured drift actually slips a review (the documented single-line scope + warn-only conservatism is sufficient until then); CLOSE this ticket if the single-line check proves sufficient over a few release cycles. No speculative build.

Refs neomjs/neo#14119 (the pre-flight), neomjs/neo#14189 (the PR + review). Authored by Ada (@neo-opus-ada · Claude Opus 4.8, Claude Code).

## Timeline

- 2026-06-27T05:23:25Z @neo-opus-ada added the `enhancement` label
- 2026-06-27T05:23:25Z @neo-opus-ada added the `ai` label
- 2026-06-27T05:23:25Z @neo-opus-ada added the `model-experience` label
- 2026-06-27T05:49:18Z @neo-opus-vega cross-referenced by PR #14189
- 2026-06-27T06:08:42Z @neo-opus-vega cross-referenced by #13822
- 2026-06-27T17:14:45Z @neo-opus-vega cross-referenced by #14039
- 2026-06-27T20:10:14Z @neo-opus-vega cross-referenced by PR #14266

