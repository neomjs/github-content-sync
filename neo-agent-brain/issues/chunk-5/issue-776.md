---
id: 776
title: The quality floor reads dotted class names as invented
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-10-02T17:32:50Z'
updatedAt: '2026-10-02T18:15:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/776'
author: neo-fable-clio
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
closedAt: '2026-10-02T18:15:29Z'
---
# The quality floor reads dotted class names as invented

Found on 2026-10-02 while measuring the hosted preset (#746 AC-4, #773): `presetQualityFloor.mjs:147` grounds a node name when every word of it occurs in the thread, with the word class `[\p{L}\p{N}_.-]{3,}` — the dot is a word character, so `Neo.main.DomEvents` is one word that no thread contains. Every hosted sample read 2–3 ungrounded names, and five of the six are canonical dotted names whose identity (the last segment) occurs in its thread; the sixth (`Focus Management`) is a genuine invention ("management" is nowhere in the thread). The rule punishes the Neo idiom — a node named by its canonical class — and keeps a model that writes it below the floor by construction.

## Context

Evidence (fixtures digest `f3cd8b71…`, the instrument's own lowercase-includes haystack, re-run offline over the recorded names):

| sample | ungrounded name | current rule | identity in the thread |
|---|---|---|---|
| 3.5-flash s1, 3.8-flash | `Neo.main.DomEvents` | invented | domevents — present (3×) |
| 3.5-flash s1 + s2, 3.8-flash | `Neo.dashboard.dock.Workspace` | invented | workspace — present (10×) |
| 3.8-flash | `Neo.grid.Body.updateMountedAndVisibleRows` | invented | updatemountedandvisiblerows — present (3×) |
| 3.5-flash s2 | `Focus Management` | invented | focus present · management absent — stays invented |

Under identity grounding the 3.8-flash sample and 3.5-flash sample 1 read 0 ungrounded with 4–5 / 3–5 grounded per document (gemma's reference: 3–4), so they would meet the floor; 3.5-flash sample 2 keeps `Focus Management`. The floor is per run — hosted's status is decided by its next measured run, not by this ticket.

## The Problem

A grounding rule that treats `.` as a word character cannot ground a canonical name unless the thread spells it in full; threads say "DomEvents" and "the dock Workspace", never `Neo.main.DomEvents`. The strictness approved in #743's review (0 ungrounded, every axis) stays; what "grounded" means is wrong for canonical paths.

## The Architectural Reality

- `measurePayload` (`ai/scripts/diagnostics/presetQualityFloor.mjs:146–151`): `words = name.match(/[\p{L}\p{N}_.-]{3,}/gu)`, grounded when `words.every(word => haystack.includes(word))`; `LABEL_NODE_TYPES` names are exempt already.
- `GEMMA_FLOOR` (`ai/services/fleet/placementPresets.mjs`) is the reference measured under the current rule; a rule change re-measures the reference under the new rule (same `documentsDigest`) before any comparison means anything.
- `summarizeRuns` / `meetsFloor` are untouched.

## The Fix

A canonical Neo path — a word with the `Neo.` root — counts by its identity, the last segment: the namespace is the model's knowledge of the codebase, which a thread rarely spells, while the identity is what the thread does or does not name (a hallucinated entity hallucinates its identity). Every other name keeps the every-word rule, so `Focus Management` stays invented and a file name (`plugin/Maximize.mjs`) is read as today. Re-measure gemma's reference through the same instrument (three local requests) and record it in `GEMMA_FLOOR` with the unchanged `documentsDigest`; re-measure hosted once (three API requests, within the operator's budget) and record the receipt on #746. Spec: the drift arm gains `Neo.dashboard.dock.Workspace` (grounded — the document says "Workspace", and neither "neo" nor "dashboard" occurs in it), keeps `Neo.dashboard.Main` invented ("Main" absent) and adds `Ratchet Management` (invented: one word of two).

## Acceptance Criteria

- AC-1: `measurePayload` grounds `Neo.dashboard.dock.Workspace` against a document that names only "Workspace", keeps `Neo.dashboard.Main` invented when "Main" is absent and `Ratchet Management` invented when "management" is absent; `presetQualityFloor.spec` pins all three.
- AC-2: `GEMMA_FLOOR.result` is re-measured under the new rule with the unchanged `documentsDigest` (receipt in the PR); `placementPresets.spec` follows.
- AC-3: the hosted preset is re-measured once under the new rule (receipt on #746 AC-4 and in the PR); whether it meets the floor is recorded. Promotion to `supported` is not part of this leaf — a table row of its own, #746's call.

## Out of Scope

- Every-segment grounding (dropping the dot from the word class): it would demand the root "neo" and every namespace word in the thread — words that are the model's knowledge, not the thread's claims — and read `Neo.grid.Body` as invented in a thread that says "grid Body".
- Whole-token matching (today "main" also matches "remaining"): a rule change for every word, with its own evidence.
- Promotion of the hosted preset; the `ungroundedNames` threshold itself.

## Related

#714 / PR #743 (the instrument) · #744 / PR #747 (the hosted lane) · #746 (AC-4 receipts) · #767 / PR #770 (the wire fix) · #773 / PR #775 (the 3.8 pin) · neomjs/neo-agent-institution#351

Decision Record impact: none (instrument-internal; the floor's strictness is unchanged).

Live latest-open sweep: the latest 20 open Brain issues read at 2026-10-02 ~17:33Z (#773 … #521), no equivalent. A2A claim sweep: no claim on the instrument in the mailbox since 14:00Z. Memory Core sweep ("quality floor grounding rule dotted canonical names"): nothing prior. Own-assignment sweep: #773, #51, #53, #50 — none overlap. Structure map: no new file.

Origin Session ID: 1efa16ff-bd83-41e5-87dc-4c186b03b451
Retrieval Hint: "presetQualityFloor grounded canonical Neo path identity last segment Neo.main.DomEvents"


## Timeline

- 2026-10-02T17:32:50Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-02T17:32:51Z @neo-fable-clio added the `bug` label
- 2026-10-02T17:32:52Z @neo-fable-clio added the `ai` label
- 2026-10-02T17:40:26Z @neo-fable-clio cross-referenced by PR #777
- 2026-10-02T17:41:01Z @neo-fable-clio cross-referenced by #746
- 2026-10-02T18:05:34Z @neo-fable-clio referenced in commit `5c39c0f` - "fix(diagnostics): an identity under the three-letter floor keeps its full path (#776)"
- 2026-10-02T18:15:29Z @tobiu referenced in commit `8f79171` - "fix(diagnostics): the quality floor grounds a canonical Neo path by its identity (#776) (#777)

* fix(diagnostics): the quality floor grounds a canonical Neo path by its identity (#776)

* fix(diagnostics): an identity under the three-letter floor keeps its full path (#776)"
- 2026-10-02T18:15:30Z @tobiu closed this issue

