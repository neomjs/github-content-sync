---
id: 390
title: Possessive and unmatched quotes hide defect-note delimiters
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - regression
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-19T21:24:36Z'
updatedAt: '2026-09-19T21:46:20Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/390'
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
closedAt: '2026-09-19T21:46:20Z'
---
# Possessive and unmatched quotes hide defect-note delimiters

## Context

Ada's [review of the merged parser change](https://github.com/neomjs/neo-agent-brain/pull/389#pullrequestreview-5257729073) found a real edge case. I reproduced it against the merged implementation: `defect-note: \`Foo.mjs\`'s header broke parsing` and `defect-note: (Foo)'s header is wrong about parsing` both remain unparsed; an equivalent plain surface parses.

## The Problem

The delimiter scanner opens single quotes after any character other than a letter, digit or underscore. A closing backtick or parenthesis therefore turns a possessive apostrophe into an unterminated quote. A lone double quote similarly hides the rest of the note. Capture remains admitted and retains raw text, but an otherwise documented verb cannot produce the intended surface/symptom split.

## The Architectural Reality

`findDefectNoteDelimiter` in `ai/services/memory-core/helpers/defectObservationFold.mjs` owns quote handling. `parseDefectNote` feeds both receipt classification and the subject-only fold. Fingerprints hash the original normalized subject independently. Its existing sibling unit spec already runs in Brain Unit CI; no new file or storage is needed.

## The Fix

Make quote protection apply only to complete quoted spans. Treat possessive apostrophes after closed code/parenthesized surfaces as punctuation. An unmatched quote remains literal text so scanning can continue while still protecting later complete quoted spans. Keep the documented verb set and whole-subject identity.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `parseDefectNote` surface/symptom | Existing helper contract and Skills defect-note grammar | Parse documented delimiters after quoted-symbol possessives and unmatched quotes; skip delimiters in complete quoted spans | Whole raw note when no structural delimiter exists | Helper JSDoc | Possessive/unmatched controls plus closed/escaped-quote counterexamples |
| Capture and identity | Existing `inspectDefectNoteCapture` / fingerprint contract | Admission, raw retention, fingerprint and recovery semantics unchanged | Existing receipt reason | Existing module contract | Receipt/fold controls and historical census |

## Acceptance Criteria

- [ ] AC-1 Backticked and parenthesized possessive surfaces parse both documented verbs with correct arms; malformed/unmatched quote text cannot hide a later structural delimiter.
- [ ] AC-2 Closed quoted delimiters remain literal, including escaped quote characters and a complete quoted span following an unmatched quote. The original quoted-separator false positive remains rejected.
- [ ] AC-3 Admission, fingerprints, recovery and the nine-note census remain unchanged. Undocumented `are wrong` remains a raw note, without expanding the grammar.

## Out of Scope

New verbs, changing the Skills grammar, body-derived capture, storage, receipt schema, and broader graduation-marker feedback.

## Avoided Traps

A global unquoted retry would resurrect false positives inside genuinely closed spans. Fix the scanner's span classification instead. This repair changes parsing only; it must not rewrite the subject before fingerprinting.

Decision Record impact: none — bounded restoration inside the existing pure parser.

Related: #304, #389

Origin Session ID: f18d3aa0-4065-41ba-9e2f-04c6bc109d5f
Retrieval Hint: defect-note quoted symbol possessive apostrophe unmatched quote hides delimiter

Creation checks: current latest-20 open Brain issues and all-read-state A2A inspected immediately before filing; no competing parser repair. Memory queries recovered Ada's exact finding and the earlier whole-subject recovery decision. Own-assignment sweep found only #48, unrelated. Existing memory-core/helpers structure owns the repair. I am taking the fix.

Prescription checked: `ai/services/memory-core/helpers/defectObservationFold.mjs` — owns lexical delimiter/quote classification. Receipt/schema changes would only conceal the wrong parse result, so they are not the repair layer.

## Timeline

- 2026-09-19T21:24:37Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-19T21:24:38Z @neo-gpt-emmy added the `bug` label
- 2026-09-19T21:24:38Z @neo-gpt-emmy added the `ai` label
- 2026-09-19T21:24:39Z @neo-gpt-emmy added the `testing` label
- 2026-09-19T21:24:39Z @neo-gpt-emmy added the `regression` label
- 2026-09-19T21:24:39Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-19T21:26:53Z @neo-gpt-emmy cross-referenced by PR #389
- 2026-09-19T21:28:28Z @neo-gpt-emmy cross-referenced by PR #391
- 2026-09-19T21:46:20Z @tobiu referenced in commit `2ed3873` - "Merge pull request #391 from neomjs/codex/390-defect-note-quote-spans

fix(memory): preserve delimiters after unmatched quotes (#390)"
- 2026-09-19T21:46:20Z @tobiu closed this issue

